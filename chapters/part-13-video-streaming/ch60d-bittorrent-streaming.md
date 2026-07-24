# Chapter 60d: BitTorrent Adaptive Streaming on Linux — libtorrent, WebTorrent, and the GPU Decode Pipeline

*Audience: Graphics application developers integrating torrent-backed media delivery pipelines; backend engineers building zero-copy ingest paths from network to GPU.*

## Table of Contents

1. [The Streaming Seam](#1-the-streaming-seam)
   - [1.1 What is BitTorrent?](#11-what-is-bittorrent)
   - [1.2 What is libtorrent-rasterbar?](#12-what-is-libtorrent-rasterbar)
   - [1.3 What is WebTorrent?](#13-what-is-webtorrent)
2. [BitTorrent Protocol Fundamentals](#2-bittorrent-protocol-fundamentals)
   - 2.1 [Bencoding and the .torrent Metainfo File (BEP 3)](#21-bencoding-and-the-torrent-metainfo-file-bep-3)
   - 2.2 [The Peer Wire Protocol (BEP 3)](#22-the-peer-wire-protocol-bep-3)
   - 2.3 [Tit-for-Tat Choking Algorithm](#23-tit-for-tat-choking-algorithm)
   - 2.4 [Extension Protocol, PEX, and DHT Bootstrapping](#24-extension-protocol-pex-and-dht-bootstrapping)
   - 2.5 [uTP — Micro Transport Protocol (BEP 29)](#25-utp--micro-transport-protocol-bep-29)
   - 2.6 [BitTorrent v2 (BEP 52)](#26-bittorrent-v2-bep-52)
3. [Release Naming Conventions and Codec Taxonomy](#3-release-naming-conventions-and-codec-taxonomy)
   - 3.1 [Filename Anatomy](#31-filename-anatomy)
   - 3.2 [Source Tags](#32-source-tags)
   - 3.3 [Video Codec Tags](#33-video-codec-tags)
   - 3.4 [HDR and Colour-Depth Tags](#34-hdr-and-colour-depth-tags)
   - 3.5 [Audio Codec Tags](#35-audio-codec-tags)
   - 3.6 [Streaming-Service Origin Tags](#36-streaming-service-origin-tags)
   - 3.7 [Release-Quality and Re-Release Tags](#37-release-quality-and-re-release-tags)
   - 3.8 [Container, Packaging, and Integrity Files](#38-container-packaging-and-integrity-files)
   - 3.9 [Codec Tags → Hardware Decode Path](#39-codec-tags--hardware-decode-path)
4. [Introduction to libtorrent-rasterbar](#4-introduction-to-libtorrent-rasterbar)
5. [Magnet URIs and .torrent Metadata](#5-magnet-uris-and-torrent-metadata)
   - 5.1 [The Magnet URI Scheme (BEP 9)](#51-the-magnet-uri-scheme-bep-9)
   - 5.2 [BEP 9 — ut_metadata: Fetching Torrent Metadata from Peers](#52-bep-9--ut_metadata-fetching-torrent-metadata-from-peers)
   - 5.3 [libtorrent Magnet URI Helpers](#53-libtorrent-magnet-uri-helpers)
6. [libtorrent Programming Model](#6-libtorrent-programming-model)
   - 6.1 [Session Construction](#61-session-construction)
   - 6.2 [Adding Torrents and Magnet Links](#62-adding-torrents-and-magnet-links)
   - 6.3 [torrent_info and file_storage](#63-torrent_info-and-file_storage)
   - 6.4 [Torrent Status](#64-torrent-status)
   - 6.5 [Alert Polling Loop](#65-alert-polling-loop)
   - 6.6 [Resume Data and Session Persistence](#66-resume-data-and-session-persistence)
7. [libtorrent-rasterbar 2.x Streaming API](#7-libtorrent-rasterbar-2x-streaming-api)
   - 7.1 [Sequential Download Flag](#71-sequential-download-flag)
   - 7.2 [set_piece_deadline — The Streaming-Recommended API](#72-set_piece_deadline--the-streaming-recommended-api)
   - 7.3 [Alert Subscription and Piece Data Delivery](#73-alert-subscription-and-piece-data-delivery)
   - 7.4 [Partially Downloaded Files on Disk](#74-partially-downloaded-files-on-disk)
8. [The HTTP Bridge Pattern](#8-the-http-bridge-pattern)
   - 8.1 [WebTorrent createServer()](#81-webtorrent-createserver)
   - 8.2 [peerflix and torrent-stream](#82-peerflix-and-torrent-stream)
   - 8.3 [htorrent: HTTP-to-BitTorrent Gateway](#83-htorrent-http-to-bittorrent-gateway)
   - 8.4 [Stremio: Proprietary Bridge at Port 11470](#84-stremio-proprietary-bridge-at-port-11470)
9. [WebTorrent over WebRTC Data Channels](#9-webtorrent-over-webrtc-data-channels)
   - 9.1 [WebSocket Tracker Signaling](#91-websocket-tracker-signaling)
   - 9.2 [Browser ServiceWorker Server](#92-browser-serviceworker-server)
10. [VLC BitTorrent Plugin and MPV Integration](#10-vlc-bittorrent-plugin-and-mpv-integration)
    - 10.1 [vlc-bittorrent Plugin Architecture](#101-vlc-bittorrent-plugin-architecture)
    - 10.2 [Seek Implementation via Piece Priorities](#102-seek-implementation-via-piece-priorities)
    - 10.3 [MPV HTTP Bridge Hooks](#103-mpv-http-bridge-hooks)
11. [Post-Assembly Decode Pipeline](#11-post-assembly-decode-pipeline)
    - 11.1 [Compressed Bitstream to VA-API Decode](#111-compressed-bitstream-to-va-api-decode)
    - 11.2 [DMA-BUF Export and Zero-Copy Display](#112-dma-buf-export-and-zero-copy-display)
    - 11.3 [GStreamer souphttpsrc Pipeline](#113-gstreamer-souphttpsrc-pipeline)
12. [Zero-Copy Network Ingest: The Kernel Horizon](#12-zero-copy-network-ingest-the-kernel-horizon)
    - 12.1 [The Network Receive Bottleneck](#121-the-network-receive-bottleneck)
    - 12.2 [Device Memory TCP (Linux 6.12/6.16)](#122-device-memory-tcp-linux-612616)
    - 12.3 [io_uring ZC RX with DMA-BUF (Linux 6.15/6.16)](#123-io_uring-zc-rx-with-dma-buf-linux-615616)
    - 12.4 [GPUDirect RDMA for PCIe Video Capture](#124-gpudirect-rdma-for-pcie-video-capture)
13. [Integrations](#13-integrations)

---

## 1. The Streaming Seam

BitTorrent is a peer-to-peer file distribution protocol. By itself it has no GPU involvement. The point where it becomes relevant to the Linux graphics stack is the **streaming seam**: the moment a torrent engine has assembled enough contiguous bitstream data to hand off to a hardware video decoder. From that boundary onward, the pipeline is identical to any other media source — VA-API, Vulkan Video, DMA-BUF export, EGL import, Wayland compositor presentation — all documented in Chs 26, 50, and 57–58.

Two distinct paths connect torrent delivery to that seam:

1. **The HTTP bridge** — a local HTTP server exposes the in-progress torrent download as a seekable byte stream. Any standard media player or FFmpeg/GStreamer pipeline consumes it via `http://localhost/...` with HTTP Range requests.

2. **Native plugin integration** — `vlc-bittorrent` implements a VLC access module that calls the libtorrent API directly, bypassing HTTP entirely.

A third, forward-looking path concerns kernel infrastructure: Linux 6.12 introduced Device Memory TCP, which can receive TCP payloads directly into GPU-mapped memory without a CPU copy. No BitTorrent client implements this today, but the underlying kernel plumbing is the same mechanism that makes all zero-copy network-to-GPU ingest possible. §12 covers this path explicitly.

### 1.1 What is BitTorrent?

BitTorrent is a peer-to-peer file distribution protocol designed to distribute large files efficiently across many participants without relying on a central server for data transfer. Rather than downloading from a single origin, a BitTorrent client splits a file into fixed-size pieces and downloads pieces simultaneously from many peers who already hold them, while simultaneously uploading pieces to others. This mutual exchange — governed by the tit-for-tat choking algorithm — lets aggregate download bandwidth scale with the number of participants rather than with the capacity of any one server.

In the context of this chapter, BitTorrent serves as the network delivery layer for media content. The protocol itself is indifferent to file type: it treats video files as sequences of equal-sized pieces identified by cryptographic hashes. The challenge for adaptive streaming is that standard BitTorrent downloads pieces in whatever order maximizes throughput, while video playback requires pieces in sequential order — or at least in decode-order proximity. Bridging this mismatch is the central engineering problem addressed here.

The protocol specification lives in BEP 3, with subsequent BEPs extending it for DHT peer discovery, peer exchange, version 2 merkle-based hashing, and the uTP UDP transport. The relevant Linux kernel involvement appears only downstream of the protocol: once the client has assembled pieces into a decodable bitstream, the VA-API and DMA-BUF subsystems take over. [Source: BEP 3](https://www.bittorrent.org/beps/bep_0003.html)

### 1.2 What is libtorrent-rasterbar?

libtorrent-rasterbar (commonly called libtorrent) is a C++ library implementing the BitTorrent client protocol. It provides session management, peer connection handling, piece scheduling, disk I/O, DHT, and extension protocol machinery that most Linux BitTorrent applications are built on. The library exposes a C++ API centered on `lt::session`, `lt::torrent_handle`, and an alert-based event system through which applications receive notifications about piece availability, peer connections, and download progress.

From a streaming standpoint, libtorrent-rasterbar's most important facility is the piece deadline API (`set_piece_deadline`), which lets an application assign a deadline in milliseconds to individual pieces. The scheduler prioritizes deadline-constrained pieces over the standard bandwidth-maximizing order, enabling video players to request imminent playback pieces first. Combined with the sequential download flag, this gives the HTTP bridge and native plugin patterns described in this chapter their ability to present an in-progress download as a seekable byte stream.

The library is available in most Linux distributions as `libtorrent-rasterbar-dev`. Version 2.x introduced significant API changes and improved DHT v2 and BEP 52 (v2 torrent) support. Source is at [https://github.com/arvidn/libtorrent](https://github.com/arvidn/libtorrent); upstream API documentation is at [https://libtorrent.org/reference.html](https://libtorrent.org/reference.html).

### 1.3 What is WebTorrent?

WebTorrent is a JavaScript implementation of the BitTorrent protocol that runs in both Node.js and web browsers. Its defining characteristic is the use of WebRTC data channels as the transport layer when running in browser contexts, because browsers cannot open raw TCP or UDP sockets. WebTorrent clients in browsers communicate with other WebTorrent browser clients via WebRTC, using a WebSocket-based tracker for the initial WebRTC signaling. When running in Node.js, WebTorrent falls back to standard TCP and UDP transports and can interoperate with the broader BitTorrent swarm.

For Linux graphics stack purposes, WebTorrent's primary contribution is the `createServer()` method, which launches a local HTTP server that exposes a torrent's files as HTTP endpoints supporting Range requests. Any media pipeline that can consume an HTTP byte stream — MPV, VLC, GStreamer's `souphttpsrc`, or a Chromium-based player — can point at this server and receive a progressively delivered video stream without knowing it is backed by BitTorrent. This HTTP bridge pattern is the simplest integration path and requires no BitTorrent-specific code in the media pipeline itself. Source is at [https://github.com/webtorrent/webtorrent](https://github.com/webtorrent/webtorrent).

---

## 2. BitTorrent Protocol Fundamentals

BitTorrent protocol specifications are published as BEPs (BitTorrent Enhancement Proposals) at [bittorrent.org/beps/](https://www.bittorrent.org/beps/).

### 2.1 Bencoding and the .torrent Metainfo File (BEP 3)

Bencoding is the serialization format for all BitTorrent data structures. Four types:

- **Strings**: `<length>:<bytes>` — e.g., `4:spam`
- **Integers**: `i<base10>e` — e.g., `i42e`, `i-3e`
- **Lists**: `l<items>e`
- **Dictionaries**: `d<key><value>...e` — keys must be bytestrings in lexicographic order

A `.torrent` metainfo file is a bencoded dictionary. Top-level keys:

| Key | BEP | Description |
|-----|-----|-------------|
| `announce` | 3 | Tracker HTTP/HTTPS URL |
| `info` | 3 | The info dictionary (below); SHA-1 of its bencoded form is the v1 infohash |
| `announce-list` | 12 | List of lists of tracker URLs in tier order |
| `url-list` | 19 | HTTP/FTP web seed URLs |
| `comment`, `created by`, `creation date` | convention | Informal metadata |
| `piece layers` | 52 | Per-file SHA-256 merkle tree hashes (v2 only) |

The `info` dictionary (single-file mode):

| Key | Description |
|-----|-------------|
| `name` | Suggested file/directory name (UTF-8) |
| `piece length` | Piece size in bytes; power of two, most commonly 2^18 = 256 KiB |
| `pieces` | Concatenated 20-byte SHA-1 hashes, one per piece |
| `length` | Total file size in bytes (single-file) |
| `files` | List of `{length, path[]}` dicts (multi-file; replaces `length`) |

The **v1 infohash** is the SHA-1 of the bencoded `info` dict. [Source: BEP 3](https://www.bittorrent.org/beps/bep_0003.html)

### 2.2 The Peer Wire Protocol (BEP 3)

Connections start with a handshake:

1. `pstrlen` = `\x13` (19)
2. `pstr` = `"BitTorrent protocol"` (19 bytes)
3. `reserved` = 8 zero bytes (extension capability bits; BEP 10 sets bit 20 from right)
4. `info_hash` = 20-byte SHA-1 infohash
5. `peer_id` = 20-byte peer identifier

All subsequent integers are 4-byte big-endian. Message format: `<4-byte length><1-byte msg_id><payload>`. Length zero is a keepalive (sent every ~120 seconds).

| ID | Message | Payload |
|----|---------|---------|
| 0 | choke | — |
| 1 | unchoke | — |
| 2 | interested | — |
| 3 | not interested | — |
| 4 | have | 4-byte piece index |
| 5 | bitfield | one bit per piece, MSB first; sent right after handshake |
| 6 | request | index (4B) + begin (4B byte offset within piece) + length (4B) |
| 7 | piece | index (4B) + begin (4B) + data |
| 8 | cancel | same three fields as request |

The standard block size in `request` messages is **16,384 bytes (2^14, 16 KiB)**. Connections start in the choked+not-interested state on both sides.

**Endgame mode:** once all sub-pieces of outstanding pieces are requested, the client sends requests for remaining blocks to all connected peers simultaneously and cancels on first receipt, minimizing tail latency. [Source: BEP 3](https://www.bittorrent.org/beps/bep_0003.html)

### 2.3 Tit-for-Tat Choking Algorithm

The choking algorithm enforces reciprocal upload:

- **Regular unchoke** (every 10 seconds): unchoke the 4 interested peers with the highest measured download rates to the local node.
- **Optimistic unchoke** (rotates every 30 seconds): one randomly selected interested peer is unchoked regardless of upload contribution, allowing discovery of better peers.
- **Snubbing**: if no data has been received from a peer for 60 seconds, upload to that peer is withheld (except via optimistic unchoke).

Standard listening port range: **6881–6889**. [Source: BEP 3](https://www.bittorrent.org/beps/bep_0003.html)

### 2.4 Extension Protocol, PEX, and DHT Bootstrapping

**BEP 10 — Extension Protocol:** bit 20 from right in the reserved handshake bytes signals BEP 10 support. After the handshake, peers exchange an extension handshake (message ID 20, extension ID 0) — a bencoded dict with `m: {ext_name: local_id, ...}` negotiating which extensions are active and the local message IDs to use for each. [Source: BEP 10](https://www.bittorrent.org/beps/bep_0010.html)

**BEP 11 — Peer Exchange (PEX):** the `ut_pex` extension lets peers share their peer lists. PEX messages carry `added`/`dropped` compact IPv4/IPv6 peer lists with per-peer flags (0x01 = prefers encryption, 0x02 = seed-only, 0x04 = supports uTP). Maximum 50 peers per update, one message per minute. PEX reduces tracker dependency in established swarms. [Source: BEP 11](https://www.bittorrent.org/beps/bep_0011.html)

**BEP 5 — Kademlia DHT:** a distributed hash table for tracker-less peer discovery. Each node has a random 160-bit node ID; distance is XOR. Routing tables organize nodes in K=8 capacity buckets covering ranges of the ID space. Four KRPC operations over UDP:

- `ping` — verify node liveness
- `find_node(target)` — returns K closest known nodes to target
- `get_peers(info_hash)` — returns peer contacts (`values`) or closer nodes (`nodes`); includes anti-spam `token`
- `announce_peer(info_hash, port, token)` — registers the caller; `implied_port=1` uses the UDP source port (NAT traversal)

Tokens are HMAC of the requester's IP and a rotating secret; valid for 10 minutes (secret rotates every 5 minutes). [Source: BEP 5](https://www.bittorrent.org/beps/bep_0005.html)

### 2.5 uTP — Micro Transport Protocol (BEP 29)

uTP runs over UDP and implements **LEDBAT** (Low Extra Delay Background Transport): delay-based congestion control that yields to TCP traffic by targeting one-way queuing delay ≤ 100 ms above base latency. uTP and TCP coexist on port 6881.

Five packet types: `ST_SYN` (4), `ST_DATA` (0), `ST_STATE` (2, ACK-only), `ST_FIN` (1), `ST_RESET` (3). The 20-byte header carries `connection_id`, `timestamp_microseconds`, `timestamp_difference_microseconds` (one-way delay), receiver `wnd_size`, and `seq_nr`/`ack_nr` (counting packets, not bytes).

LEDBAT window adjustment per RTT:

```
off_target    = 100ms - our_delay
delay_factor  = off_target / 100ms
window_factor = outstanding_bytes / max_window
max_window   += MAX_CWND_INCREASE_PER_RTT × delay_factor × window_factor
```

On packet loss: `max_window *= 0.5`. Minimum timeout: 500 ms. Base delay is a 2-minute sliding minimum. [Source: BEP 29](https://www.bittorrent.org/beps/bep_0029.html)

### 2.6 BitTorrent v2 (BEP 52)

v2 replaces SHA-1 with SHA2-256. Key `info` dict changes:

| | v1 | v2 |
|--|----|----|
| Hash algorithm | SHA-1, 20 bytes | SHA2-256, 32 bytes |
| File layout | `files` list | `file tree` nested dict |
| Per-piece hashes | `pieces` concatenation | per-file `pieces root` (merkle root) |
| Version marker | absent | `meta version: 2` |

The `file tree` value for each file contains a `pieces root` — the root of a SHA2-256 merkle tree with 16 KiB leaves. The top-level `piece layers` key carries concatenated hashes at the piece-level layer of each merkle tree. The v2 infohash is SHA2-256 of the bencoded v2 `info` dict.

**Hybrid torrents** include both v1 (`pieces`, `files`/`length`) and v2 (`file tree`, `piece layers`) structures, yielding two separate infohashes and enabling participation in both v1 and v2 swarms. [Source: BEP 52](https://www.bittorrent.org/beps/bep_0052.html)

---

## 3. Release Naming Conventions and Codec Taxonomy

Torrent filenames encode the technical properties of the enclosed media in a structured tag syntax established by the scene release community and widely adopted by P2P groups. Understanding this taxonomy is essential for two reasons: it determines which hardware decode path the player must use, and it predicts the range of container/codec/HDR combinations a torrent-backed application must handle.

### 3.1 Filename Anatomy

Tags are separated by dots or dashes. A typical 4K release:

```
Movie.Title.2024.2160p.HMAX.WEB-DL.DDP5.1.Atmos.DV.HDR10.HEVC-GroupName.mkv
```

Decoded left to right: title, year, **resolution**, **streaming-service source**, **transfer method**, **audio codec + channels + object format**, **HDR formats**, **video codec**, release group, container. Tags may be reordered; the codec and resolution tags are always present. Season packs take the form `S01.2160p...` or `S01E01-E06...` for episode ranges.

### 3.2 Source Tags

The source tag identifies where the encoded bitstream originated.

| Tag | Full name | Quality notes |
|-----|-----------|---------------|
| `WEB-DL` | Web download | Closest to streaming master; losslessly muxed from service streams |
| `WEBRip` | Web rip | Screen-captured or stream-intercepted; slightly softer than WEB-DL |
| `REMUX` / `BDRemux` | Blu-ray remux | Lossless Blu-ray extract, no re-encoding; largest files |
| `BluRay` / `BDRip` | Blu-ray encode | Re-encoded from disc |
| `BDAV` | Blu-ray AV | Japanese Blu-ray disc format; often MPEG-2/H.264 |
| `HDTV` | HDTV capture | Broadcast capture; may include logo bugs, ad cuts |
| `DVDRip` | DVD rip | Standard-definition, interlaced source |
| `CAM` | Camcorder | In-theatre recording; lowest quality |
| `TS` | Telesync | Slightly better camcorder (projected from separate audio) |
| `TC` | Telecine | Film-to-digital transfer, often from workprint |

The quality ordering is: `REMUX` > `WEB-DL` > `WEBRip` > `BluRay` > `BDRip` > `HDTV` > `DVDRip` > `CAM/TS/TC`.

**WEB-DL vs WEBRip:** WEB-DL captures the MPEG-DASH or HLS stream segments directly from the CDN, then demuxes them into a standard container without re-encoding. The video data is the service's own master encode. WEBRip historically used screen capture or partial stream interception and introduced an additional lossy encode step; modern WEBRip captures are often indistinguishable from WEB-DL in practice, but the tag still signals a non-lossless capture path.

**REMUX vs BluRay encode:** A REMUX is a bit-exact copy of the Blu-ray disc video stream in a new container (MKV). Peak bitrate on a 4K UHD Blu-ray reaches 80–100 Mbps, with the average encode around 40–50 Mbps. A `BluRay` or `BDRip` release re-encodes that source at 15–25 Mbps (x265) or 8–15 Mbps (x264), discarding detail that is genuinely invisible at normal viewing distance. For a 1080p encode with libx265 at CRF 18, the visual difference from REMUX is negligible; at CRF 22+, fine grain and dark-scene detail degrade measurably. File size: 4K REMUX ≈ 50–80 GB; 4K x265 BluRay encode ≈ 15–25 GB; 4K WEB-DL ≈ 12–30 GB (varies by service bitrate).

**WEB-DL vs REMUX for streaming:** WEB-DL is the preferred source for torrent streaming because file sizes are practical (10–30 GB) and quality is already limited by the service encode, not the container. REMUX files are so large that sequential-download or `set_piece_deadline()` can stall playback while waiting for enough verified pieces; a 60 GB REMUX at 10 MB/s requires 100 minutes to buffer a 5-minute initial segment.

**HDTV:** broadcast captures add a transmission-compression pass on top of the original studio master, introducing mosquito noise around hard edges and block artefacts in dark scenes. HDTV remains the only timely source for sports, live events, and series that are not yet on streaming services.

### 3.3 Video Codec Tags

| Tag | Codec | Encoder / notes |
|-----|-------|-----------------|
| `x264` | H.264/AVC | libx264 software encoder |
| `H.264` / `AVC` | H.264/AVC | Generic or hardware-encoded (NVENC, QSV, AMF) |
| `x265` / `HEVC` | H.265/HEVC | libx265; High 10 profile for 10-bit content |
| `H.265` | H.265/HEVC | Generic |
| `AV1` | AV1 | libaom, SVT-AV1, rav1e; increasingly common for 4K WEB-DL |
| `VP9` | VP9 | Rare in scene releases; common in YouTube/Android streams |
| `MPEG2` | MPEG-2 | Legacy; still appears in HDTV/BDAV |
| `NVENC` | Any | NVIDIA GPU encoder; faster encode, slightly lower quality than software |
| `QSV` | Any | Intel Quick Sync Video hardware encoder |
| `AMF` | Any | AMD hardware encoder (AMD Media Framework) |

**Profile and level** are embedded in the bitstream SPS NAL (H.264/H.265) or sequence header (AV1). For H.264: High 4.0 for 1080p, High 5.1 for 4K; High 10 for 10-bit. For H.265: Main 10 for 10-bit HDR. mpv and FFmpeg log the profile on open; GStreamer surfaces it via `video/x-h265, profile=(string)main-10`.

**x264 vs x265 vs AV1 — compression efficiency:** at equivalent perceptual quality (measured via VMAF), x265 produces files roughly 40–50% smaller than x264 at the same resolution. AV1 (SVT-AV1) is a further 20–30% smaller than x265. Quantitatively: a 1080p SDR film at VMAF 95 might encode to 8 GB with x264 (CRF 18), 4.5 GB with x265 (CRF 20), and 3 GB with SVT-AV1 (CRF 22). These ratios shift at higher resolutions: the gain from x265→AV1 is smaller for 4K content because display-referred sharpness is harder to preserve at any bitrate.

**Encode time:** x264 ≈ 80–120 fps at 1080p on a modern CPU; x265 ≈ 10–20 fps; libaom-AV1 ≈ 0.5–2 fps; SVT-AV1 ≈ 20–60 fps at medium presets; rav1e ≈ 5–15 fps. This is why HEVC dominates scene encodes (time-to-release matters) and AV1 is only practical for streaming services with dedicated encoder fleets or for P2P groups willing to spend GPU time.

**Hardware vs software encode quality:** NVENC, QSV, and AMF trade ≈5–15% quality (VMAF) for 10–100× encode speed versus their software equivalents at the same bitrate. They are the correct choice for real-time or near-real-time use cases (Shadowplay, OBS, WHIP publishing) but produce inferior quality compared to libx264/libx265/SVT-AV1 at the same file size, which is why hardware-encoded content is rare in scene releases.

**Hardware decode support on Linux** — this directly determines which GStreamer/FFmpeg hwaccel element a player must use:

| Codec | VA-API decoder | Min Intel GPU | Min AMD GPU | Min NVIDIA GPU |
|-------|---------------|--------------|------------|----------------|
| H.264 High/High10 | `vah264dec` | Broadwell (Gen8) | GCN 1st gen | Kepler (600 series) |
| H.265/HEVC Main10 | `vah265dec` | Skylake (Gen9) | GCN 4th gen (Polaris) | Maxwell (900 series) |
| VP9 Profile 2 (10-bit) | `vavp9dec` | Kaby Lake (Gen9.5) | Vega+ | Maxwell+ |
| AV1 | `vaav1dec` | Tiger Lake (Gen12/Xe) | RDNA 2 (RX 6000) | Ampere (RTX 30 series) |

On hardware without AV1 decode, an `AV1` release requires software decode (`av1dec` in GStreamer, `libdav1d` via `-c:v libdav1d` in FFmpeg), which consumes ≈3–8 CPU cores for 4K at 24 fps.

**8-bit vs 10-bit:** 10-bit encodes (`10bit`, `yuv420p10le` in ffprobe) suppress banding artefacts in smooth gradients — skies, dark scenes, skin tones — even when the display is 8-bit SDR, because the encoder has more headroom to represent subtle tonal steps. File size overhead is ≈10–15%. All HDR content is 10-bit minimum; SDR releases are typically 8-bit. For VA-API, 10-bit surfaces are P010 (2-plane NV12 with 16-bit samples); `vainfo` lists `VAProfileH265Main10` separately from `VAProfileH265Main`.

### 3.4 HDR and Colour-Depth Tags

| Tag | Standard | Transfer curve | Metadata type |
|-----|----------|----------------|---------------|
| `HDR` / `HDR10` | SMPTE ST.2084 PQ | Perceptual Quantizer | Static — mastering display (ST.2086) + MaxCLL/MaxFALL |
| `HDR10+` | SMPTE ST.2094-40 | PQ | Dynamic per-frame (Samsung) |
| `DV` / `DoVi` | Dolby Vision | PQ (base) | Dynamic per-frame RPU; up to 12-bit |
| `HLG` | ARIB STD-B67 | Hybrid Log-Gamma | Scene-referred; backwards-compatible with SDR displays |
| `SDR` | BT.709 | BT.709 / sRGB | None |
| `10bit` | — | any | 10-bit luma/chroma sample depth |
| `12bit` | — | any | 12-bit; rare; associated with Dolby Vision dual-layer |

**Dolby Vision profiles** you will encounter in release `.nfo` files and in the bitstream:

| Profile | Description | Fallback |
|---------|-------------|---------|
| 5 | Streaming single-layer, pure DV | None (requires DV display) |
| 7 | Dual-layer (FEL), Blu-ray | HDR10 base layer |
| 8 | Single-layer with HDR10 base (most common in `WEB-DL`) | HDR10 |
| 9 | Single-layer with SDR base | SDR |

Profile 8 is the dominant format in `DV.HDR10` releases from streaming services. The `RPU` (Reference Processing Unit) side-data stream carries per-frame tone-mapping curves and colour transforms; it is carried as HEVC SEI NAL units or prepended to AV1 OBUs.

**HYBRID tag** (distinct from DV hybrid torrents in §5.1): indicates a custom encode that combines a DV RPU extracted from one source with a re-encoded HDR10 base layer from another, typically to improve bitrate efficiency. Not a standard format tag; meaning is group-specific.

**HDR10 vs HDR10+:** HDR10 applies a single static tone-map curve to the entire film based on the peak and average brightness of the grade. HDR10+ adds per-scene or per-frame dynamic metadata (SMPTE ST.2094-40) so that a dim scene is not crushed by the same peak-brightness assumption as an outdoor day scene. In practice the visual improvement is most apparent on displays with peak brightness below the mastering peak (e.g. a 600-nit panel playing a 1000-nit master). On Linux, no open-source display stack currently applies HDR10+ dynamic metadata; the HDR10 static base is used as a fallback. A `HDR10+` release is therefore visually identical to its `HDR10` counterpart on Linux until libplacebo or KWin gains ST.2094-40 support.

**HDR10 vs Dolby Vision:** both use PQ transfer and BT.2020 primaries. DV adds a per-frame RPU layer that can specify independent tone-mapping curves for highlights and shadows, and supports up to 12-bit precision in dual-layer (Profile 7) form. The visual advantage of DV over HDR10 is most evident on OLED displays capable of near-zero black levels, where the per-frame minimum black setting in the RPU suppresses global dimming artefacts. On LCD panels with local dimming, the difference is smaller. For Linux playback: mpv 0.37+ with `libplacebo` can read and apply DV RPU side-data for software tone-mapping; VA-API has no RPU-aware decode path, so hardware-decoded DV content falls back to HDR10 (Profile 8) or SDR (Profile 9) base layer.

**HDR10 vs HLG:** HLG (Hybrid Log-Gamma) is scene-referred rather than display-referred. It carries no absolute nit values; brightness is interpreted relative to the display's SDR white point. On an SDR display, HLG simply looks like a slightly bright SDR signal — no tone-mapping is required. This makes it ideal for live broadcast where the receiver display is unknown. HDR10 requires an active tone-map for SDR output. For streaming torrent content (films, series), `HDR10` is the standard; `HLG` appears in broadcast sports captures.

**10-bit vs 8-bit for SDR content:** even a nominally SDR encode benefits from 10-bit because the extra precision eliminates banding in dark scenes. The cost is a ≈15% file size increase and a higher GPU clock requirement for software decode. For hardware decode, the codec profile changes (`VAProfileH264High` → `VAProfileH264High10`), so the player must explicitly check the profile before selecting a hwaccel path.

**Linux HDR display pipeline (2026):** explicit HDR output to a connected HDR display requires a compositor with HDR support. KDE Plasma 6.1+ on Wayland supports HDR with `--hdr-enabled` via `VK_EXT_hdr_metadata` / `VK_EXT_swapchain_colorspace`. Gamescope supports `--hdr-enabled` for gaming use. GNOME 47+ gained HDR support. X11 has no composited HDR path; mpv on X11 with `vo=gpu-next` and `target-colorspace-hint=yes` can do direct-scan-out HDR on some drivers via the atomic KMS HDR metadata property (`HDR_OUTPUT_METADATA`).

### 3.5 Audio Codec Tags

| Tag | Full name | Channels | Lossless? |
|-----|-----------|----------|-----------|
| `AAC2.0` / `AAC5.1` | Advanced Audio Codec | 2.0 / 5.1 | No |
| `AC3` / `DD5.1` | Dolby Digital (AC-3) | 5.1 | No |
| `DDP5.1` / `EAC3` | Dolby Digital Plus (E-AC-3) | 5.1 | No |
| `DDP7.1` | Dolby Digital Plus | 7.1 | No |
| `Atmos` | Dolby Atmos | Object-based | No (lossy wrappers) |
| `TrueHD` | Dolby TrueHD | Up to 7.1 | Yes |
| `TrueHD.Atmos` | TrueHD + Atmos | Object-based | Yes |
| `DTS` | DTS core | 5.1 | No |
| `DTS-HD.MA` | DTS-HD Master Audio | Up to 7.1 | Yes |
| `DTS:X` | DTS:X | Object-based | No |
| `FLAC` | FLAC | Any | Yes |
| `MP3` | MPEG-1 Layer III | Stereo | No |
| `OPUS` | Opus | Any | No |

Channel count suffixes: `2.0` (stereo), `5.1` (3 front, 2 surround, 1 LFE), `7.1` (adds 2 rear surround). Atmos adds object-based metadata on top of a bed channel layout — `TrueHD.Atmos` on Blu-ray, `DDP5.1.Atmos` or `DDP7.1.Atmos` on streaming.

In an MKV container, multiple audio tracks are common: a lossless `TrueHD.Atmos` primary track and a compatibility `AC3` (Dolby Digital) core track embedded in the same MKV. FFmpeg exposes these as stream indices `0:a:0`, `0:a:1`.

**Lossless vs lossy — practical differences:** for critical listening on a high-quality system, TrueHD and DTS-HD MA are bit-identical to the studio master and are perceptually indistinguishable from each other. For typical home theatre and laptop playback, DDP5.1 at 640 kbps and DTS at 1510 kbps are transparent — ABX testing reliably fails to differentiate them from lossless at these rates. Lossy tracks are ≈2–10× smaller than their lossless equivalents (a 2-hour film: TrueHD.Atmos ≈ 4–8 GB, DDP5.1 ≈ 0.4–0.6 GB).

**AC3 vs DDP:** AC3 (Dolby Digital) is capped at 640 kbps and 5.1 channels — the format of legacy DVD and broadcast. DDP (E-AC-3) extends to 1024 kbps on streaming platforms and supports 7.1 beds plus Atmos object metadata. DDP is fully backwards-compatible: an E-AC-3 decoder falls back gracefully to the AC-3 core for devices that only speak AC-3. In MKV releases, a separate low-bitrate AC3 track (`0:a:1`) serves as a universal compatibility track for devices that cannot decode DDP.

**AAC vs DDP vs Opus:** AAC achieves better quality than DDP at the same bitrate for stereo (2.0) content — Apple TV+ and Crunchyroll exploit this. At 5.1 multichannel, DDP is more efficient than AAC-LC. Opus (not yet common in releases, widespread in WebRTC and YouTube) achieves the best quality-per-bit of all lossy formats at both stereo and multichannel.

**Atmos vs DTS:X — format and Linux support:** both are object-based spatial audio formats. Dolby Atmos carries its object metadata inside a TrueHD or DDP container; DTS:X carries it inside a DTS-HD MA container as an extension substream. On Linux, PipeWire 1.0+ can decode Dolby Atmos (DDP bed + object metadata) and DTS:X in software and down-mix to stereo or 5.1; HDMI passthrough of the bitstream to an AV receiver for lossless spatial decoding requires a player that supports `AUDIO_NONINTERLEAVED` or `IEC61937` output — mpv with `ao=alsa,oss` and `audio-spdif=truehd,dts-hd`.

**Linux decoder availability:** FFmpeg and GStreamer (`avdec_*`, `decodebin`) support all formats in software. Hardware audio decode (DSP offload) is not exposed through ALSA, PipeWire, or VA-API on Linux — all decoding is CPU-side, which is negligible overhead for audio compared to video.

### 3.6 Streaming-Service Origin Tags

These appear in `WEB-DL` and `WEBRip` releases to indicate which platform's stream was captured. Relevant because different services use different codec stacks and DRM:

| Tag | Service | Common codec | DRM |
|-----|---------|-------------|-----|
| `NF` | Netflix | H.264 / HEVC / AV1 | Widevine L1 |
| `AMZN` | Amazon Prime Video | H.264 / HEVC / AV1 | Widevine L1 / PlayReady |
| `DSNP` | Disney+ | H.264 / HEVC / AV1 | Widevine L1 |
| `HMAX` | Max (HBO Max) | H.264 / HEVC | Widevine L1 |
| `ATVP` | Apple TV+ | HEVC / AV1 | FairPlay |
| `PCOK` | Peacock | H.264 / HEVC | Widevine L1 |
| `HULU` | Hulu | H.264 / HEVC | Widevine L1 |
| `CRKL` | Crunchyroll | H.264 / HEVC | Widevine L1 |
| `STAN` | Stan (AU) | H.264 / HEVC | Widevine L1 |
| `MA` | Movies Anywhere | (distribution, not encode) | Varies |

`WEB-DL` releases from these services are already demuxed bitstreams — the DRM has been bypassed during capture; the released file is a standard MKV or MP4.

**Encode quality comparisons:** streaming services do not publish encode specifications, but independent analysis of `ffprobe` output and VMAF measurements from released files reveals consistent patterns. Netflix encodes with per-title optimisation (different bitrate ladders per title based on complexity) and has used AV1 for 4K since 2023; typical 4K AV1 average bitrate is 8–16 Mbps, VMAF ≈ 93–95. Amazon Prime Video uses HEVC for most 4K content with average bitrates of 12–20 Mbps; some titles use `AMZN AV1`. Disney+ targets HEVC at 15–20 Mbps for 4K with aggressive quality levels, scoring VMAF ≈ 93–95. Max (HBO Max) is noted in the release community for higher average bitrates (15–25 Mbps HEVC) and conservative CRF equivalents, resulting in WEB-DL files often being larger than Netflix or Disney+ equivalents for the same title. Apple TV+ uses aggressive HEVC/AV1 encodes with `FairPlay` DRM, making lossless capture more difficult; released `ATVP` files are less frequent.

**DRM hierarchy:** Widevine L1 (hardware-enforced, requires Trusted Execution Environment) is used by NF, AMZN, DSNP, HMAX, PCOK, CRKL; Widevine L3 (software, running in CDM) is used for lower-resolution fallback streams. FairPlay (Apple's DRM) is entirely separate and requires Apple hardware or Safari. PlayReady (Microsoft) appears on AMZN on Edge/Windows and on some smart TV platforms. None of these DRM systems is present in released WEB-DL files — the file is a standard MKV or MP4 once captured.

**Codec availability by service over time:** the shift from H.264 to HEVC to AV1 has not been uniform. `NF 1080p` releases are often still H.264 (the service uses H.264 as a fallback for devices without HEVC decode); `NF 2160p` is HEVC or AV1. `AMZN` 1080p is almost entirely HEVC. A `NF.WEB-DL.H264` filename signals a 1080p or lower capture; a `NF.WEB-DL.HEVC` or `NF.WEB-DL.AV1` signals a 4K capture.

### 3.7 Release-Quality and Re-Release Tags

| Tag | Meaning |
|-----|---------|
| `REPACK` | Re-release by the **same group** correcting a technical flaw (bad audio sync, wrong audio track, corrupt piece, aspect ratio error) |
| `REPACK2` | Third attempt from the same group |
| `PROPER` | Re-release by a **different group** fixing a flaw in a prior release; named `PROPER` rather than REPACK |
| `INTERNAL` | Distributed within the scene group only; not intended for wide torrent circulation |
| `EXTENDED` | Extended cut (additional footage vs theatrical release) |
| `DC` | Director's Cut |
| `THEATRICAL` | The original theatrical version (distinguished when DC or extended also exists) |
| `DUBBED` | Foreign-language dub audio track (replaces or supplements original) |
| `SUBBED` | Subtitle tracks included |
| `HARDSUB` | Subtitles burned into the video frame (cannot be disabled) |
| `SOFTSUB` / `MULTI-SUB` | Subtitles as separate MKV/MP4 tracks (can be toggled) |
| `MULTI` | Multiple audio language tracks |
| `COMPLETE` / `SEASON.PACK` | Entire season or series in one torrent |
| `SAMPLE` | A short 50–100 MB preview clip from the same encode, released alongside the main torrent |

`REPACK` beats any earlier release of the same title at the same quality tier — seeders switch, indexers replace.

**REPACK vs PROPER:** a REPACK can be anticipated by the releasing group (they caught their own mistake before widespread seeding); a PROPER is reactive (a second group noticed and publicly named the flaw). PROPER releases historically signal a more serious defect — a misidentified audio track, wrong aspect ratio, interlaced source passed as progressive — because the second group had to invest effort in re-ripping. From a download perspective, prefer PROPER when one exists; prefer REPACK2 over REPACK if the REPACK itself was found to have issues.

**Cut comparisons (EXTENDED vs DC vs THEATRICAL):** extended cuts add deleted scenes reinstated by the studio after theatrical release, often increasing runtime by 10–30 minutes. A Director's Cut (DC) reflects creative intent and can be shorter than theatrical (unnecessary studio-mandated scenes removed) or longer (the director's preferred pacing). Where all three exist, the theatrical cut has the most peer-reviewed viewing time and widest subtitle/dub availability; extended cuts are often subtitled-only releases. For automated media server ingestion, identifying the cut is important because audio/subtitle track timestamps differ across cuts.

**HARDSUB vs SOFTSUB for torrent streaming applications:** hardsub video has subtitles composited into the video frame as part of the encode — they cannot be suppressed, restyled, or processed separately. For a GPU pipeline that passes video frames as DMA-BUF fds, this is actually simpler: the subtitle texture is already present in the decoded frame, no separate subtitle renderer is needed. Softsub (separate ASS/SSA or SRT tracks in MKV) requires a subtitle renderer (libass, FFmpeg `subtitles` filter) to composite the text onto the video surface before display, which means either CPU-side compositing into a software frame or a GPU-side `cairo`/`harfbuzz`/`freetype` text render that writes into the final compositor surface.

### 3.8 Container, Packaging, and Integrity Files

**Containers:**

| Extension | Format | Notes |
|-----------|--------|-------|
| `.mkv` | Matroska | Universal scene/P2P container; supports all codec/subtitle/attachment types |
| `.mp4` | MPEG-4 | Common for mobile-targeted releases; limited subtitle track support |
| `.ts` | MPEG Transport Stream | HDTV captures; often demuxed to MKV by release groups |
| `.m2ts` | Blu-ray Transport Stream | Blu-ray disc raw format before remux |

**Scene packaging (RAR + PAR2):** older scene releases split the MKV into multi-volume RAR archives (`movie.rar`, `movie.r00`, `movie.r01`, …) with PAR2 parity recovery files. PAR2 can reconstruct damaged or missing parts without re-downloading if enough parity data is present. Modern P2P releases (YTS-pattern, torrent-native) ship single uncompressed files; RAR packaging is a scene convention with no benefit for torrent distribution.

**Integrity files:**

- `.nfo` — ASCII/ANSI info file containing encoder settings, release notes, CRC32 checksums per file, and group signature. Standard parse target for torrent indexers.
- `.sfv` (Simple File Verification) — text file listing filenames with CRC32 checksums; used to verify archive integrity.
- `.md5` / `.sha256` — hash files; less common than SFV.

**MKV vs MP4 — technical comparison:**

| Feature | MKV (Matroska) | MP4 (MPEG-4 Part 14) |
|---------|---------------|----------------------|
| Codec support | Any codec (no restrictions) | H.264, H.265, AV1, AAC, MP3 (others via vendor boxes) |
| Subtitle formats | SRT, ASS/SSA, PGS, VobSub, HDMV-TextST | TTML, WebVTT (SRT via workaround) |
| Attachment tracks | Yes (fonts for ASS rendering) | No |
| Chapter metadata | Yes (MatroskaChapters) | Yes (QuickTime chapters) |
| Seekability | Cluster-based (random access requires Cues element) | `moov` atom (fast seek when placed at file start with `faststart`) |
| Streaming compatibility | Requires player MKV support | Universal (iOS, Android, smart TV, browser MSE) |
| HDR metadata | Carried in track headers (ColourSpace, MaxFall, etc.) | Carried in `mdcv`/`clli` boxes |
| DV RPU side-data | As codec private data | As `dovi` box |

For torrent streaming applications: MKV's codec and subtitle flexibility makes it the preferred container for releases intended for PC/media-centre playback with mpv, VLC, or Kodi. MP4 is preferred when the downstream target includes iOS, Android smart TVs, or browser-based players (MSE API only handles fragmented MP4 / CMAF natively; MKV requires a JavaScript demuxer). The HTTP bridge pattern works equally well with both containers as long as the server returns `Content-Type: video/x-matroska` or `video/mp4` respectively, and the player handles byte-range seeking.

**RAR/PAR2 vs single-file for torrent distribution:** scene RAR splits offer no advantage for torrent-native distribution. BitTorrent already provides piece-level integrity via SHA-1/SHA-256 hash verification; PAR2 parity is redundant. RAR split archives require full assembly before the file can be played, adding a large disk-space overhead and breaking sequential-access streaming via `set_piece_deadline()`. Single-file MKV/MP4 releases downloaded via libtorrent allow partial-file reading as soon as individual pieces are verified — the direct streaming advantage described in §7.4.

**ffprobe** is the most direct tool for reading what a release actually contains:

```bash
ffprobe -v quiet -print_format json -show_streams \
    "Movie.2024.2160p.HMAX.WEB-DL.DDP5.1.Atmos.DV.HDR10.HEVC-Group.mkv"
```

Key fields: `codec_name`, `profile`, `level`, `pix_fmt` (e.g. `yuv420p10le`), `color_transfer` (e.g. `smpte2084`), `color_primaries` (e.g. `bt2020`), `color_space` (e.g. `bt2020nc`), `side_data_list` (HDR10 mastering metadata, Dolby Vision RPU).

### 3.9 Codec Tags → Hardware Decode Path

Each release's codec/profile/HDR combination maps to a specific VA-API, NVDEC, or V4L2 codec profile, and determines whether tone-mapping is needed on the decode side:

| Release tags | VA-API element | GStreamer caps | Tone-mapping needed? |
|---|---|---|---|
| `x264` / `H.264` 8-bit | `vah264dec` | `video/x-h264, profile=high` | No (SDR) |
| `x265` / `HEVC` 10-bit `HDR10` | `vah265dec` | `video/x-h265, profile=main-10` | Yes (HDR→SDR or pass-through to HDR display) |
| `AV1` 10-bit `HDR10` | `vaav1dec` | `video/x-av1, profile=main` | Yes |
| `HEVC` 10-bit `DV.HDR10` | `vah265dec` + RPU | `video/x-h265, profile=main-10` | Yes (DV tone-map or HDR10 fallback) |
| `AV1` 10-bit `DV` | `vaav1dec` + RPU | `video/x-av1` | Yes |
| `VP9` 10-bit | `vavp9dec` | `video/x-vp9` | Profile-dependent |

`pix_fmt=yuv420p10le` in ffprobe output confirms a 10-bit file; `color_transfer=smpte2084` confirms PQ HDR. VA-API decode always produces NV12 (8-bit) or P010 (10-bit) surfaces; tone-mapping from HDR to SDR happens downstream in the display pipeline (see Ch26 §5 for `VADisplayAttribute` tone-mapping configuration, Ch57 §4 for FFmpeg `tonemap_vaapi` filter).

Dolby Vision RPU side-data requires application-level handling: VA-API provides no RPU-aware decode path on Linux; the RPU must be extracted and passed to a display-side tone-mapper (e.g., `libplacebo`'s Dolby Vision support in mpv 0.37+). [Source: mpv Dolby Vision tracking](https://github.com/mpv-player/mpv/issues/10190)

---

## 4. Introduction to libtorrent-rasterbar

[libtorrent-rasterbar](https://github.com/arvidn/libtorrent) is a C++ library (BSD license) implementing the BitTorrent protocol suite. It covers the full BEP stack: core wire protocol (BEP 3), DHT (BEP 5), Extension Protocol (BEP 10), PEX (BEP 11), uTP/LEDBAT (BEP 29), magnet link bootstrapping via `ut_metadata` (BEP 9), HTTP web seeds (BEP 19), BitTorrent v2 (BEP 52), protocol encryption, and more.

Applications built on libtorrent: qBittorrent, Deluge, and the `vlc-bittorrent` plugin (§10). The Stremio community replacement `perpetus/stream-server` (§8.4) also uses it.

**Version timeline:**
- **v2.0.x** (branch RC_2_0, latest v2.0.13, 2026-06-08): requires C++14; default disk backend is `mmap_disk_io`
- **v2.1.0** (branch RC_2_1, 2026-07-09): requires C++17 and OpenSSL ≥ 1.1.0; default disk backend switches to `pread_disk_io`; adds `set_sequential_range()`

**Build** (CMake):
```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release \
      -Dcmake_deprecated_functions=ON   # keeps ABI v2 compatibility layer
make -C build -j$(nproc)
```

Link with `-ltorrent-rasterbar -lssl -lcrypto -lpthread`. pkg-config name: `libtorrent-rasterbar`. [Source: github.com/arvidn/libtorrent](https://github.com/arvidn/libtorrent)

---

## 5. Magnet URIs and .torrent Metadata

### 5.1 The Magnet URI Scheme (BEP 9)

A magnet URI identifies content by hash rather than location. The BitTorrent v1 format:

```
magnet:?xt=urn:btih:<infohash>[&dn=<name>][&tr=<tracker>][&tr=<tracker2>][&x.pe=<host:port>]
```

`xt=urn:btih:` carries the 40-character hex encoding (or 32-character base32) of the SHA-1 info dict hash. Multiple `tr=` parameters are allowed; each must be percent-encoded. `x.pe=` provides bootstrap peer addresses.

Additional parameters from the generic Magnet URI scheme (not defined in any BEP, but widely implemented):

| Parameter | libtorrent field | Description |
|-----------|-----------------|-------------|
| `ws=<url>` | `add_torrent_params::url_seeds` | Web seed (BEP 19 HTTP seed) |
| `xs=<url>` | — | Exact source |
| `as=<url>` | — | Acceptable source (fallback) |
| `so=<indices>` | `file_priorities` | Select-only file indices (BEP 53); comma-separated, ranges allowed (e.g. `so=0,2,4,6-8`) |

For BitTorrent **v2**, the `xt=` uses the SHA2-256 infohash as a multihash with prefix `1220` (function code 0x12 = SHA2-256, length 0x20 = 32 bytes):

```
magnet:?xt=urn:btmh:1220<64-hex-sha256>
```

**Hybrid torrents** carry both parameters:

```
magnet:?xt=urn:btih:<40-hex-sha1>&xt=urn:btmh:1220<64-hex-sha256>&dn=Example&tr=...
```

[Source: BEP 9](https://www.bittorrent.org/beps/bep_0009.html), [BEP 52](https://www.bittorrent.org/beps/bep_0052.html), [multiformats/multihash](https://github.com/multiformats/multihash)

### 5.2 BEP 9 — ut_metadata: Fetching Torrent Metadata from Peers

When only a magnet URI is available, the client fetches the `info` dictionary from peers via the `ut_metadata` extension (negotiated over BEP 10). A peer that has the metadata advertises it in its BEP 10 extension handshake:

```
{'m': {'ut_metadata': 3}, 'metadata_size': 31235}
```

`metadata_size` is mandatory and tells the requesting peer how many 16 KiB pieces to expect. Three message types sent over the negotiated `ut_metadata` channel:

| `msg_type` | Name | Payload |
|------------|------|---------|
| 0 | request | `{'msg_type': 0, 'piece': N}` |
| 1 | data | `{'msg_type': 1, 'piece': N, 'total_size': S}` + raw 16 KiB chunk |
| 2 | reject | `{'msg_type': 2, 'piece': N}` |

The receiving peer reassembles pieces and validates SHA-1 of the assembled `info` dict against the magnet URI's infohash before using it. In libtorrent, `torrent_status::state == downloading_metadata` (state 2) indicates the fetch is in progress; `metadata_received_alert` (type 45) signals completion. [Source: BEP 9](https://www.bittorrent.org/beps/bep_0009.html)

### 5.3 libtorrent Magnet URI Helpers

```cpp
#include <libtorrent/magnet_uri.hpp>

// Parse a magnet URI → add_torrent_params:
lt::add_torrent_params atp = lt::parse_magnet_uri(
    "magnet:?xt=urn:btih:da39a3ee...&dn=Example&tr=http%3A%2F%2Ftracker.example.org");

// Non-throwing overload:
lt::error_code ec;
lt::add_torrent_params atp2 = lt::parse_magnet_uri(uri, ec);
if (ec) { /* handle parse error */ }

// Generate a magnet URI from params:
std::string uri = lt::make_magnet_uri(atp);
```

`parse_magnet_uri` populates `atp.info_hashes` (v1 and/or v2), `atp.name`, `atp.trackers`, `atp.url_seeds`, and `atp.peers`. [Source: libtorrent magnet_uri.hpp RC_2_1](https://github.com/arvidn/libtorrent/blob/RC_2_1/include/libtorrent/magnet_uri.hpp)

---

## 6. libtorrent Programming Model

### 6.1 Session Construction

`lt::session` owns all torrent state, peer connections, and the DHT node. Construct via `lt::session_params`:

```cpp
#include <libtorrent/session.hpp>
#include <libtorrent/settings_pack.hpp>

lt::settings_pack sp;
sp.set_int(lt::settings_pack::connections_limit, 500);
sp.set_int(lt::settings_pack::download_rate_limit, 0);    // 0 = unlimited
sp.set_str(lt::settings_pack::listen_interfaces, "0.0.0.0:6881,[::]:6881");
sp.set_bool(lt::settings_pack::enable_upnp, true);
sp.set_bool(lt::settings_pack::enable_dht, true);
sp.set_int(lt::settings_pack::alert_queue_size, 2000);

lt::session_params params(sp);
// ut_metadata is required for magnet link bootstrapping:
params.extensions.push_back(lt::create_ut_metadata_plugin());
params.extensions.push_back(lt::create_ut_pex_plugin());

lt::session ses(std::move(params));
```

Key setting defaults: `connections_limit=200`, `alert_queue_size=2000`, listen on `0.0.0.0:6881,[::]:6881`. Settings can be applied post-construction via `ses.apply_settings(sp)`. [Source: libtorrent settings_pack.hpp RC_2_1](https://github.com/arvidn/libtorrent/blob/RC_2_1/include/libtorrent/settings_pack.hpp)

### 6.2 Adding Torrents and Magnet Links

**From a .torrent file:**

```cpp
auto ti = std::make_shared<lt::torrent_info>("video.torrent");

lt::add_torrent_params atp;
atp.ti = ti;
atp.save_path = "/media/downloads";
// flags: seed_mode, upload_mode, paused, auto_managed, sequential_download, etc.
atp.flags &= ~lt::torrent_flags::paused;

lt::torrent_handle h = ses.add_torrent(std::move(atp));
```

**From a magnet URI** (metadata fetched from peers via ut_metadata):

```cpp
lt::add_torrent_params atp = lt::parse_magnet_uri(
    "magnet:?xt=urn:btih:...&dn=Movie&tr=...");
atp.save_path = "/media/downloads";
lt::torrent_handle h = ses.add_torrent(std::move(atp));
// Status: downloading_metadata until peers deliver the info dict.
// metadata_received_alert (type 45) fires on completion.
```

### 6.3 torrent_info and file_storage

```cpp
auto ti = std::make_shared<lt::torrent_info>("bundle.torrent");

std::cout << "name:   " << ti->name() << "\n"
          << "pieces: " << ti->num_pieces()
          << " × " << ti->piece_length() << " B\n"
          << "total:  " << ti->total_size() << " B\n";

// info_hashes() returns an info_hash_t with both v1 (SHA-1) and v2 (SHA-256)
lt::info_hash_t hashes = ti->info_hashes();
if (hashes.has_v1())
    std::cout << "v1: " << hashes.v1 << "\n";
if (hashes.has_v2())
    std::cout << "v2: " << hashes.v2 << "\n";

// Enumerate files:
const lt::file_storage& fs = ti->files();
for (lt::file_index_t i(0); i < lt::file_index_t(fs.num_files()); ++i) {
    std::cout << fs.file_path(i)   // relative path
              << "  " << fs.file_size(i) << " B\n";
}
```

`file_storage::file_path(idx, save_path)` returns the absolute on-disk path. `file_storage::file_offset(idx)` is the byte offset within the concatenated piece space. [Source: libtorrent torrent_info.hpp RC_2_1](https://github.com/arvidn/libtorrent/blob/RC_2_1/include/libtorrent/torrent_info.hpp)

### 6.4 Torrent Status

```cpp
lt::torrent_status st = h.status();

// state_t enum values (from torrent_status.hpp):
//   checking_files (1), downloading_metadata (2), downloading (3),
//   finished (4), seeding (5), checking_resume_data (7)
if (st.state == lt::torrent_status::downloading) {
    printf("%.1f%%  %.1f kB/s  %d peers  %d seeds\n",
        st.progress * 100.0f,
        st.download_rate / 1024.0f,
        st.num_peers,
        st.num_seeds);
}
```

Key `torrent_status` fields: `progress` [0.0, 1.0], `download_rate`/`upload_rate` (bytes/sec), `num_peers`, `num_seeds`, `total_done`/`total_wanted`, `pieces` (downloaded-piece bitfield), `name`, `save_path`, `current_tracker`. [Source: libtorrent torrent_status.hpp RC_2_1](https://github.com/arvidn/libtorrent/blob/RC_2_1/include/libtorrent/torrent_status.hpp)

### 6.5 Alert Polling Loop

libtorrent delivers all asynchronous events as `lt::alert` subclasses. The standard dispatch loop (ABI=2, default RC_2_1 build; `wait_for_alert` returns `alert*`):

```cpp
ses.set_alert_mask(lt::alert_category::status
                 | lt::alert_category::storage
                 | lt::alert_category::piece_progress);

std::vector<lt::alert*> alerts;
for (bool running = true; running; ) {
    lt::alert* a = ses.wait_for_alert(std::chrono::milliseconds(250));
    if (!a) continue;
    ses.pop_alerts(&alerts);

    for (lt::alert* al : alerts) {
        switch (al->type()) {
        case lt::add_torrent_alert::alert_type:           // 67
            break;
        case lt::metadata_received_alert::alert_type:     // 45 — magnet ready
            break;
        case lt::state_changed_alert::alert_type: {       // 10
            auto* sc = lt::alert_cast<lt::state_changed_alert>(al);
            // sc->state, sc->prev_state (state_t enum)
            break;
        }
        case lt::read_piece_alert::alert_type: {          // 5
            auto* rp = lt::alert_cast<lt::read_piece_alert>(al);
            // rp->buffer (shared_array<char>), rp->size, rp->piece
            break;
        }
        case lt::torrent_finished_alert::alert_type:      // 26
            h.save_resume_data(lt::torrent_handle::save_info_dict);
            break;
        case lt::save_resume_data_alert::alert_type: {    // 37
            auto* srd = lt::alert_cast<lt::save_resume_data_alert>(al);
            auto buf = lt::write_resume_data_buf(srd->params);
            // write buf to <infohash>.resume
            break;
        }
        }
    }
}
```

[Source: libtorrent session_handle.hpp RC_2_1](https://github.com/arvidn/libtorrent/blob/RC_2_1/include/libtorrent/session_handle.hpp)

### 6.6 Resume Data and Session Persistence

Resume data lets a restarted client skip re-hashing already-verified pieces:

```cpp
// Save (async) — typically on torrent_finished_alert or clean shutdown:
h.save_resume_data(lt::torrent_handle::save_info_dict
                 | lt::torrent_handle::flush_disk_cache);
// → save_resume_data_alert fires; .params is a populated add_torrent_params
auto buf = lt::write_resume_data_buf(srd->params);
// write buf to e.g. ~/.local/share/myapp/<infohash>.resume

// Restore at next startup:
auto buf = /* read file */;
lt::add_torrent_params atp = lt::read_resume_data(buf);
atp.save_path = "/media/downloads";
ses.add_torrent(std::move(atp));
```

DHT routing table persistence (preserves peer discovery state across restarts):

```cpp
// At shutdown:
auto sp = ses.session_state(lt::session_params::save_dht_state);
auto buf = lt::write_session_params_buf(sp, lt::session_params::save_dht_state);
// write buf to ~/.local/share/myapp/session.state

// At startup:
auto buf = /* read file */;
auto sp = lt::read_session_params(buf, lt::session_params::save_dht_state);
lt::session ses(std::move(sp));
```

[Source: libtorrent reference-Resume_Data](https://www.libtorrent.org/reference-Resume_Data.html), [reference-Session](https://www.libtorrent.org/reference-Session.html)

---

## 7. libtorrent-rasterbar 2.x Streaming API

libtorrent-rasterbar (introduced in §4) is the dominant BitTorrent C++ library on Linux. Latest releases: v2.0.13 (2026-06-08, RC_2_0 branch) and v2.1.0 (2026-07-09, RC_2_1 branch). This section covers the streaming-specific API layered on top of the general programming model in §6.

### 7.1 Sequential Download Flag

```cpp
// include/libtorrent/torrent_flags.hpp (RC_2_1)
constexpr torrent_flags_t sequential_download = 9_bit;
```

Enable at add time:

```cpp
lt::add_torrent_params p;
p.save_path = "/media/store";
p.flags |= lt::torrent_flags::sequential_download;
lt::torrent_handle h = ses.add_torrent(std::move(p));
```

Or post-add: `h.set_flags(lt::torrent_flags::sequential_download)`.

The libtorrent header comment is explicit: *"Sequential mode is not ideal for streaming media. For that, see `set_piece_deadline()` instead."* Sequential mode downloads pieces from index 0 upward regardless of peer speed and does not prioritize the playback head. It is appropriate for archive extraction, not adaptive media delivery. [Source: libtorrent torrent_flags.hpp RC_2_1](https://github.com/arvidn/libtorrent/blob/RC_2_1/include/libtorrent/torrent_flags.hpp)

v2.1.0 adds `set_sequential_range()`, absent in v2.0.x:

```cpp
// RC_2_1 only
void set_sequential_range(piece_index_t first_piece, piece_index_t last_piece) const;
void set_sequential_range(piece_index_t first_piece) const; // range to end of file
```

[Source: libtorrent torrent_handle.hpp RC_2_1](https://github.com/arvidn/libtorrent/blob/RC_2_1/include/libtorrent/torrent_handle.hpp)

### 7.2 set_piece_deadline — The Streaming-Recommended API

```cpp
void set_piece_deadline(piece_index_t index, int deadline, deadline_flags_t flags = {}) const;
void reset_piece_deadline(piece_index_t index) const;
void clear_piece_deadlines() const;
```

`deadline` is milliseconds from the call time by which libtorrent attempts to deliver the piece. Pieces with nearer deadlines outprioritize pieces with farther deadlines. When `deadline_flags_t::alert_when_available` is set, a `read_piece_alert` fires once the piece is available, delivering the raw bytes to the application.

The scheduler maintains two sorted structures: **peers** sorted by estimated download queue time (outstanding bytes / estimated rate), and **time-critical pieces** sorted by deadline. Each tick it iterates pieces by deadline and requests remaining blocks from the fastest available peer. Pieces that exceed `avg_piece_time + 0.5 × deviation` are timed out and a redundant block request is sent to a second peer, escalating with each repeated timeout. Slow (10th-percentile), choked, and snubbed peers are excluded from the assignment pool. [Source: libtorrent streaming docs](http://libtorrent.org/streaming.html)

A typical streaming client drives this API with a sliding window: as the player advances, set deadlines for the next N pieces proportional to expected playback time, and `reset_piece_deadline()` for pieces behind the playhead.

Per-piece priority is a coarser alternative:

```cpp
// Priority scale: 0 (don't download) through 7 (top priority); default 4
void piece_priority(piece_index_t index, download_priority_t priority) const;
void prioritize_pieces(std::vector<download_priority_t> const& pieces) const;
void prioritize_pieces(std::vector<std::pair<piece_index_t, download_priority_t>> const& pieces) const;
```

The vlc-bittorrent plugin (§10.2) uses priority 7/6/5 rather than deadlines.

### 7.3 Alert Subscription and Piece Data Delivery

```cpp
// alert_types.hpp (RC_2_1)
struct read_piece_alert final : torrent_alert {
    static int const alert_type = 5;
    // category: alert_category::storage
    error_code const error;
    boost::shared_array<char> const buffer; // null on failure
    piece_index_t const piece;
    int const size;
};

struct piece_finished_alert final : torrent_alert {
    static int const alert_type = 27;
    // category: alert_category::piece_progress (subscribe explicitly)
    piece_index_t const piece_index;
};

struct file_completed_alert final : torrent_alert {
    static int const alert_type = 6;
    // category: alert_category::file_progress
    file_index_t const index;
};
```

Subscribe in `settings_pack::alert_mask`:

```cpp
sp.set_int(lt::settings_pack::alert_mask,
           lt::alert_category::storage |
           lt::alert_category::piece_progress |
           lt::alert_category::file_progress);
lt::session ses(sp);
```

`read_piece_alert` delivers the raw piece bytes. `piece_finished_alert` fires when hash verification passes (before disk flush). `file_completed_alert` fires when all pieces covering a file index are verified. [Source: libtorrent alert_types.hpp RC_2_1](https://github.com/arvidn/libtorrent/blob/RC_2_1/include/libtorrent/alert_types.hpp)

### 7.4 Partially Downloaded Files on Disk

libtorrent writes each verified piece to `save_path/<torrent-name>/<file-path>` at the correct byte offset. Files are sparse: unverified piece ranges are holes. As soon as `piece_finished_alert` fires for a given piece, an external reader can `open()` and `pread()` at the correct offset without waiting for the full download.

Byte-offset computation from a piece index:

```cpp
// file_storage maps piece byte ranges to file offsets
auto slices = ti->map_block(piece_idx, 0, ti->piece_size(piece_idx));
// each file_slice: .file_index, .offset (bytes from file start), .size
```

The v2.0.x default disk backend is `mmap_disk_io` (kernel mmap, OS page cache manages flushing). v2.1.0 switches the default to `pread_disk_io` (preadv/pwritev on dedicated threads). Both write completed pieces immediately. [Source: libtorrent upgrade_to_2.1-ref.html](https://www.libtorrent.org/upgrade_to_2.1-ref.html)

---

## 8. The HTTP Bridge Pattern

The HTTP bridge is the universal integration mechanism: a local HTTP server exposes the in-progress torrent as a byte-seekable HTTP resource. Any tool that can open an HTTP URL — FFmpeg, GStreamer, VLC, mpv — consumes the torrent stream without BitTorrent awareness. The minimum contract is `Accept-Ranges: bytes` in the response and correct `206 Partial Content` + `Content-Range` headers for byte-range requests.

### 8.1 WebTorrent createServer()

[WebTorrent](https://github.com/webtorrent/webtorrent) v2.3.0 (June 2024) is the canonical Node.js/browser BitTorrent implementation. As of v2.3.0, native WebRTC support is built in; the former `webtorrent-hybrid` package is deprecated and archived.

```javascript
import WebTorrent from 'webtorrent'
const client = new WebTorrent()

client.add(magnetURI, torrent => {
  const server = client.createServer()
  server.listen(8000, () => {
    // URL: http://localhost:8000/webtorrent/<infoHash>/<filepath>
    const file = torrent.files.find(f => f.name.endsWith('.mkv'))
    console.log(file.streamURL) // full URL for this file
  })
})
```

The server uses the `range-parser` npm package to parse `Range: bytes=<start>-<end>` headers and responds with `HTTP 206 Partial Content`. It always sets `Accept-Ranges: bytes` and `Cache-Control: no-cache, no-store`. File bytes are streamed via the file's async iterator over the requested byte window, causing only the covering torrent pieces to be fetched on demand:

```javascript
const iterator = file[Symbol.asyncIterator](range)
const stream = Readable.from(iterator)
```

For lower-level access: `file.createReadStream({ start, end })` returns a Node.js `Readable` stream covering exactly those bytes. [Source: webtorrent/lib/server.js](https://raw.githubusercontent.com/webtorrent/webtorrent/master/lib/server.js), [WebTorrent API docs](https://webtorrent.io/docs)

### 8.2 peerflix and torrent-stream

[peerflix](https://github.com/mafintosh/peerflix) is a Node.js CLI that starts a local HTTP server on port 8888 and launches a player when the file is ready. It predates WebTorrent and uses the `torrent-stream` library as its download engine.

```bash
npm install -g peerflix
peerflix "magnet:?xt=..." --mpv         # launch mpv automatically
peerflix "magnet:?xt=..." -p 9000       # custom port
peerflix "video.torrent" --list         # list files in torrent
```

peerflix is community-maintained legacy software with no significant upstream commits in recent years. The underlying `torrent-stream` library does not use WebRTC. [Source: github.com/mafintosh/peerflix](https://github.com/mafintosh/peerflix)

### 8.3 htorrent: HTTP-to-BitTorrent Gateway

[htorrent](https://github.com/pojntfx/htorrent) is a purpose-built HTTP gateway with explicit seeking support, designed as a proxy between media players and torrent content:

```
/info?magnet=<url>                        → JSON torrent metadata and file list
/stream?magnet=<url>&path=<filepath>      → byte-range-seekable file stream
/metrics                                  → download progress and peer stats
```

Default listen port: 1337. Supports HTTP Basic Auth and OIDC. mpv integration:

```bash
mpv --http-header-fields="Authorization: Basic <token>" \
    "http://localhost:1337/stream?magnet=<magnet>&path=video.mkv"
```

### 8.4 Stremio: Proprietary Bridge at Port 11470

[Stremio](https://www.stremio.com/) splits into an open-source UI layer and a proprietary streaming server:

- **`stremio-core`** (Rust, MIT): addon system, stream selection, UI models. [github.com/Stremio/stremio-core](https://github.com/Stremio/stremio-core)
- **`stremio-web`** (JS, GPL-2.0): browser/Electron UI. [github.com/Stremio/stremio-web](https://github.com/Stremio/stremio-web)
- **`server.js`** (proprietary): BitTorrent download engine, HTTP stream proxying, optional FFmpeg transcoding, subtitle fetching. Runs at `http://127.0.0.1:11470/`. Accepts `infoHash` + `fileIdx` parameters. Serves HLS endpoints (`master.m3u8`) for Chromecast-compatible output.
- **`stremio-shell`** (Qt/C++): embeds mpv via `mpv_render_context_create()` with `gpu-hwdec-interop=auto` and a 15,000 KB playback cache. Passes `http://127.0.0.1:11470/...` URLs to the embedded mpv. [Source: stremio-shell/mpv.cpp](https://github.com/Stremio/stremio-shell/blob/master/mpv.cpp)

An open-source Rust replacement for `server.js` — [perpetus/stream-server](https://github.com/perpetus/stream-server) — implements the same API endpoints using libtorrent, consuming roughly 50 MB versus `server.js`'s 200 MB+. A Python/FastAPI alternative — [andrewhack/stremio-libtorrent-server](https://github.com/andrewhack/stremio-libtorrent-server) — adds playhead-first piece picking.

---

## 9. WebTorrent over WebRTC Data Channels

WebTorrent extends BitTorrent to run over `RTCDataChannel` instead of TCP/uTP, enabling browser-native peer exchange. The BitTorrent wire protocol (handshake + piece exchange) is unchanged; only the transport layer differs. In Node.js, WebTorrent uses both TCP/uTP and WebRTC. In the browser, only WebRTC is available. [Source: WebTorrent FAQ](https://webtorrent.io/faq)

**Interoperability constraint:** Browser peers can only connect to seeders that support WebRTC — WebTorrent Desktop, BiglyBT with WebTorrent plugin, Vuze with WebTorrent plugin. Standard TCP/uTP clients cannot serve browser peers without WebRTC support.

### 9.1 WebSocket Tracker Signaling

WebRTC requires SDP offer/answer exchange before an `RTCDataChannel` can open. WebTorrent repurposes BitTorrent tracker announces over WebSocket (`wss://`) as the signaling channel.

When a client calls `client.add(magnetURI)`, it connects to WebSocket tracker URLs from the magnet (`tr=wss://...`) and sends an `announce` message embedding SDP offers:

```json
{
  "action": "announce",
  "info_hash": "<hex info hash>",
  "peer_id": "<20-byte peer id>",
  "numwant": 5,
  "offers": [
    { "offer_id": "<random 20-byte id>", "offer": { /* RTCSessionDescription */ } }
  ]
}
```

The tracker relays each offer to another peer announcing the same info-hash. That peer generates an SDP answer and the tracker forwards it back. Both ends complete the ICE/DTLS handshake and open an `RTCDataChannel` for BitTorrent piece exchange. [Source: bittorrent-tracker](https://github.com/webtorrent/bittorrent-tracker)

Public WebSocket trackers include `wss://tracker.openwebtorrent.com` and `wss://tracker.btorrent.xyz`. The high-performance open-source tracker [wt-tracker](https://github.com/Novage/wt-tracker) handles approximately 30,000 WSS peers on 2 GiB RAM / 1 vCPU.

### 9.2 Browser ServiceWorker Server

In the browser, `createServer({ controller: swRegistration })` installs a ServiceWorker that intercepts `fetch()` calls to the `/webtorrent/<infoHash>/` path and serves file bytes from the in-memory torrent buffer. This enables standard `<video src="...">` playback of in-progress downloads without a Node.js backend. Supported video containers and codecs are subject to the browser's native codec support: `mp4`, `webm`, `mkv`, `ogv`, `mov` with AV1, H.264, HEVC, VP8, VP9. [Source: WebTorrent API docs](https://webtorrent.io/docs)

---

## 10. VLC BitTorrent Plugin and MPV Integration

### 10.1 vlc-bittorrent Plugin Architecture

VLC's official source tree (`modules/access/`) contains no BitTorrent module — the capability was never merged upstream. The active third-party plugin is [johang/vlc-bittorrent](https://github.com/johang/vlc-bittorrent) (last commit July 7 2026), packaged as:

- Debian/Ubuntu: `vlc-plugin-bittorrent` (official repositories)
- Fedora/EPEL: `vlc-plugin-bittorrent` (EPEL 10)
- Arch: AUR package `vlc-bittorrent`

The plugin compiles to `libaccess_bittorrent_plugin.so` using libtorrent-rasterbar and registers three VLC module capabilities via `module.cpp`:

| Capability | Priority | Purpose |
|---|---|---|
| `stream_directory` | 99 | Metadata access for `.torrent` files (`MetadataOpen`) |
| `stream_extractor` | 99 | Data extraction from torrents (`DataOpen/DataClose`) |
| `access` | 60 | Handles `magnet:` and `file:` URL schemes |

### 10.2 Seek Implementation via Piece Priorities

When VLC's demuxer seeks to a byte offset, the plugin's seek callback (in `download.cpp`) performs:

1. Maps the file byte offset to torrent piece indices via `ti->map_file(file_index, file_offset, length)`.
2. Sets graduated libtorrent piece priorities:
   - Priority 7 (highest): pieces covering the requested data range.
   - Priority 6: first/last 0.1% or 128 KB of the file (container headers/trailers, needed by the demuxer).
   - Priority 5: next 5% or 32 MB around the seek position.
3. Calls `m_th.read_piece(piece_index)` for async piece retrieval.
4. Subscribes to `piece_finished_alert` and `read_piece_alert` via an internal `AlertSubscriber` to deliver bytes back to VLC's demuxer without polling.

This architecture keeps seeking latency proportional to swarm availability at the seek target, not to overall download progress.

### 10.3 MPV HTTP Bridge Hooks

mpv has no native BitTorrent support. It consumes torrent content via the HTTP bridge, issuing `Range: bytes=0-` on the first request to probe seeking capability — if the server returns `206 Partial Content`, mpv uses byte-range seeks throughout playback. [Source: mpv issue #655](https://github.com/mpv-player/mpv/issues/655), [mpv issue #8145](https://github.com/mpv-player/mpv/issues/8145)

Two mpv hook scripts automate the bridge:

- **mpv-peerflix-hook** ([noctuid/mpv-peerflix-hook](https://github.com/noctuid/mpv-peerflix-hook)): shell/Lua script that intercepts magnet URIs passed to mpv, forks peerflix in the background, extracts the HTTP URL from its output, and redirects mpv to that URL. Handles port conflict detection and kills the peerflix process when mpv exits.

- **webtorrent-mpv-hook** ([mrxdst/webtorrent-mpv-hook](https://github.com/mrxdst/webtorrent-mpv-hook)): TypeScript npm package installed as an mpv script. Accepts magnet links, info-hashes, or `.torrent` files directly as mpv arguments. Displays a download-progress overlay; `p` key toggles it. Uses WebTorrent internally.

---

## 11. Post-Assembly Decode Pipeline

Once the HTTP bridge or native plugin delivers a byte-range-seekable bitstream, the rest of the pipeline is identical to any other media source. This section cross-references the relevant chapters rather than repeating their content.

### 11.1 Compressed Bitstream to VA-API Decode

With peerflix or WebTorrent's `createServer()` running on port 8000, FFmpeg reads and hardware-decodes directly:

```bash
ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -hwaccel_output_format vaapi \
       -i "http://localhost:8000/webtorrent/<infoHash>/video.mkv" \
       -f null -
```

The `AVHWFramesContext` for VAAPI causes decoded frames to remain in VA surfaces without any CPU memcpy. See Ch57 §3 for the `AVHWDeviceContext` lifecycle and Ch26 §4 for the VA-API decode surface model.

### 11.2 DMA-BUF Export and Zero-Copy Display

After VA-API decode, the zero-copy path to the display is:

```c
// Export the decoded VA surface as a DMA-BUF fd
VADRMPRIMESurfaceDescriptor desc;
vaExportSurfaceHandle(va_dpy, surface,
    VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2,
    VA_EXPORT_SURFACE_READ_ONLY, &desc);

// desc.objects[0].fd is a DMA-BUF fd carrying the decoded NV12 frame
// Import into EGL:
EGLImageKHR img = eglCreateImageKHR(dpy, EGL_NO_CONTEXT,
    EGL_LINUX_DMA_BUF_EXT, NULL, attrs);  // attrs reference desc.fd

// Or import into Vulkan:
VkImportMemoryFdInfoKHR import_info = {
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    .fd = desc.objects[0].fd
};
```

No pixel data copy occurs between decode and GPU-side compositing. The DMA-BUF fd carries the GEM handle across driver boundaries and into the Wayland compositor via `linux-dmabuf-unstable-v1`. See Ch26 §8 for `vaExportSurfaceHandle()` details and Ch4 for DMA-BUF architecture.

### 11.3 GStreamer souphttpsrc Pipeline

GStreamer's `souphttpsrc` element (libsoup-based HTTP source) implements byte-range seeking: when the server returns `206 Partial Content` with `Content-Range`, `souphttpsrc` marks itself seekable. A full zero-copy decode and display pipeline:

```bash
gst-launch-1.0 \
  souphttpsrc location="http://localhost:8000/webtorrent/<infoHash>/video.mp4" \
  ! qtdemux \
  ! vah264dec \
  ! 'video/x-raw(memory:DMABuf)' \
  ! waylandsink
```

`vah264dec` (from `gst-plugins-bad`) outputs NV12 VA-API surfaces exported as DMA-BUF fds with DRM format modifiers. `waylandsink` imports via `linux-dmabuf-unstable-v1`. See Ch58 §4 for VA-API element caps negotiation. [Source: souphttpsrc docs](https://gstreamer.freedesktop.org/documentation/soup/souphttpsrc.html)

---

## 12. Zero-Copy Network Ingest: The Kernel Horizon

### 12.1 The Network Receive Bottleneck

The zero-copy chain described in §11 — VA-API decode → DMA-BUF → EGL/Vulkan → compositor — is deployed and zero-copy today. The remaining copy is at the other end of the pipeline: network receive. libtorrent and all current BitTorrent clients receive TCP packet payloads via `recv()` into system RAM (kernel `sk_buff` → userspace buffer). The compressed bitstream then passes CPU-side to the hardware decoder. Eliminating this copy requires kernel infrastructure that landed in Linux 6.12–6.16.

### 12.2 Device Memory TCP (Linux 6.12/6.16)

Device Memory TCP creates a direct DMA path from NIC to device-memory (including GPU VRAM):

**RX path (Linux 6.12, November 2024):** TCP payload bytes land in a dmabuf-backed memory region, bypassing the CPU entirely. Headers land in normal host-memory `sk_buff` buffers (requires hardware header-split on the NIC). The socket API:

```c
// Bind a dmabuf to a NIC RX queue via netlink
netdev_bind_rx(ifindex, dmabuf_fd, queue_id);

// Receive: payload bytes are in the dmabuf, not in a user buffer
struct msghdr msg = {};
recvmsg(sock, &msg, MSG_SOCK_DEVMEM);
// SCM_DEVMEM_DMABUF cmsg carries the buffer offset
```

**TX path (Linux 6.16, mid-2025):** Zero-copy send from device memory, commit `bd61848900bf`. [Source: Linux kernel devmem docs](https://docs.kernel.org/networking/devmem.html), [Phoronix: Device Memory TCP TX](https://www.phoronix.com/news/Device-Memory-TCP-TX-Linux-6.16)

**GPU memory integration:** CUDA 13.0 introduced `cuMemGetHandleForAddressRange()` with `CU_MEM_RANGE_HANDLE_TYPE_DMA_BUF_FD`, enabling a CUDA allocation to be exported as a DMA-BUF fd. This fd can be passed to `netdev_bind_rx()` as the receive region, routing TCP payload bytes directly into GPU VRAM. The full NIC→GPU→VA-API decode chain is technically possible with Linux 6.16 + CUDA 13.0, but has not been assembled into any shipping product as of mid-2026.

**NIC requirements:** Hardware header-split (separates L4 header from payload), flow steering or RSS to direct target flows to the dmabuf-bound queue. The initial reference NIC is the Google Virtual Ethernet (GVE) with DQO support in GKE. No standard consumer or datacenter NIC driver exposes this path broadly yet. Notably, TCPdump, eBPF, and software checksum offload do not work on devmem TCP payloads. [Source: LWN Device Memory TCP](https://lwn.net/Articles/937882/)

### 12.3 io_uring ZC RX with DMA-BUF (Linux 6.15/6.16)

**Linux 6.0** introduced `IORING_OP_SEND_ZC` — zero-copy socket *send* from a registered user buffer.

**Linux 6.15** (April 2025) introduced `IORING_OP_RECV_ZC` (io_uring zcrx) — zero-copy socket *receive*. Incoming packet payloads DMA directly into a userspace-registered memory region (a page pool provisioned from user pages). Headers and payloads are split at L4; completions arrive via `io_uring_zcrx_cqe` entries with buffer offsets. Saturates a 200 Gbit/s link on a single CPU core. Requires `IORING_SETUP_CQE32`. Currently supports **host memory only**. [Source: kernel iou-zcrx docs](https://docs.kernel.org/networking/iou-zcrx.html), [Phoronix: Linux 6.15 io_uring](https://www.phoronix.com/news/Linux-6.15-IO_uring)

**Linux 6.16 target:** DMA-BUF support for io_uring zcrx, allowing a DMA-BUF fd (including a CUDA-exported GPU allocation) to serve as the receive region. This is the io_uring analog to Device Memory TCP's `netdev_bind_rx()`. [Source: Phoronix: io_uring ZCRX DMA-BUF](https://www.phoronix.com/news/IO_uring-ZCRX-DMA-BUF)

No BitTorrent client implements `IORING_OP_RECV_ZC` as of mid-2026.

### 12.4 GPUDirect RDMA for PCIe Video Capture

GPUDirect RDMA allows a third-party PCIe device (InfiniBand or RoCE HCA, capture card, FPGA) to DMA directly into GPU VRAM via `nvidia-peermem.ko`. The nv-p2p kernel API (`nvidia_p2p_get_pages`, `nvidia_p2p_dma_map_pages`) used by third-party drivers is deprecated in CUDA 13.0 and will be removed in CUDA 14.0; replacements are standard upstream `pin_user_pages()` + `dma_map_sgtable()`. [Source: GPUDirect RDMA docs](https://docs.nvidia.com/cuda/gpudirect-rdma/index.html)

GPUDirect RDMA for video ingest is deployed in two contexts:

- **NVIDIA Holoscan SDK + AJA NTV2**: The AJA NTV2 SDK 16.1 added GPUDirect support. A 4K UHD RGBA capture card DMA-writes frames directly into CUDA GPU memory over PCIe. Measured latency drops from 23.7 ms (CPU path, exceeds 16.7 ms frame budget at 60 fps) to 12.4 ms (GPUDirect PCIe path). [Source: NVIDIA Holoscan blog](https://developer.nvidia.com/blog/supporting-low-latency-streaming-video-for-ai-powered-medical-devices-with-clara-holoscan/)
- **Deltacast VideoMaster SDK 6.20+**: GPUDirect RDMA on Linux x86_64 and ARM64 for broadcast I/O cards. [Source: Deltacast GPUDirect page](https://www.deltacast.tv/technologies/gpudirect-rdma/)

In both cases the "RDMA" is PCIe peer-to-peer between a capture card and the GPU — not an InfiniBand or RoCE network path. GPUDirect RDMA over InfiniBand/RoCE is used at scale for HPC distributed training (NCCL, NVSHMEM), not for media streaming. No BitTorrent client or torrent-backed media pipeline uses GPUDirect RDMA. The connection raised earlier in this chapter — RDMA as grounds for torrent streaming's relevance to this book — is accurate at the infrastructure level: the same kernel p2pdma and DMA-BUF mechanisms that enable PCIe capture-to-GPU paths are the foundation of Device Memory TCP and io_uring ZC RX, which are the paths that could eventually connect network delivery (including BitTorrent) to GPU decode without a CPU bounce, as covered in §12.2–12.3.

GPUDirect Storage (GDS, via `cuFile`) is a storage-only technology covering NVMe local, NVMe-oF, NFS, Lustre, and WekaFS. It has no socket or TCP path and is unrelated to BitTorrent. [Source: GDS Overview Guide](https://docs.nvidia.com/gpudirect-storage/overview-guide/index.html)

---

## 13. Integrations

- **Ch26 (VA-API)**: Hardware video decode of the bitstream assembled by libtorrent or delivered via HTTP bridge. `vaExportSurfaceHandle()` DMA-BUF export is the zero-copy path from decoder to display.
- **Ch50 (Vulkan Video)**: Alternative to VA-API for decode; Vulkan Video decode surfaces can be imported as DMA-BUF fds via `VK_EXT_external_memory_dma_buf`, same chain as §11.2.
- **Ch57 (FFmpeg)**: `AVHWDeviceContext` for VAAPI consumes the HTTP bridge via `-i http://localhost:...`; the `AVVkFrame` path (Vulkan hwaccel) applies equally.
- **Ch58 (GStreamer)**: `souphttpsrc` consumes the HTTP bridge; `vah264dec`/`vaav1dec` with DMABuf caps negotiation provides the zero-copy decode-to-display chain.
- **Ch60b (Streaming Protocols and ABR)**: WebTorrent's WebRTC transport uses the same RTCDataChannel / ICE / DTLS-SRTP infrastructure documented there. WebSocket tracker signaling is the BitTorrent-specific signaling mechanism layered on top of that transport.
- **Ch60c (WebRTC Server Infrastructure)**: Public WebSocket trackers (`wss://tracker.openwebtorrent.com`) serve the same role for WebTorrent peers that STUN/TURN servers serve for WebRTC media connections.
- **Ch189 (VLC)**: `vlc-bittorrent` is a VLC access module that integrates libtorrent-rasterbar directly into VLC's demuxer callback chain.
- **Ch4 (DRM Architecture / DMA-BUF)**: The DMA-BUF fd produced by `vaExportSurfaceHandle()` and consumed by EGL/Vulkan/Wayland is the same kernel DMA-BUF primitive. Device Memory TCP's `netdev_bind_rx()` uses the same fd type.
- **Ch49 (Multi-GPU and p2pdma)**: GPUDirect RDMA's PCIe peer-to-peer DMA path (capture card → GPU) uses the same `p2pdma` kernel framework. The Linux 6.2+ PCI P2PDMA path for GDS also uses this infrastructure.
