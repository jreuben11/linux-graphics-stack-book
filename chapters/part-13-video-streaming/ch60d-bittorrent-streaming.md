# Chapter 60d: BitTorrent Adaptive Streaming on Linux — libtorrent, WebTorrent, and the GPU Decode Pipeline

*Audience: Graphics application developers integrating torrent-backed media delivery pipelines; backend engineers building zero-copy ingest paths from network to GPU.*

## Table of Contents

1. [The Streaming Seam](#1-the-streaming-seam)
2. [BitTorrent Protocol Fundamentals](#2-bittorrent-protocol-fundamentals)
   - 2.1 [Bencoding and the .torrent Metainfo File (BEP 3)](#21-bencoding-and-the-torrent-metainfo-file-bep-3)
   - 2.2 [The Peer Wire Protocol (BEP 3)](#22-the-peer-wire-protocol-bep-3)
   - 2.3 [Tit-for-Tat Choking Algorithm](#23-tit-for-tat-choking-algorithm)
   - 2.4 [Extension Protocol, PEX, and DHT Bootstrapping](#24-extension-protocol-pex-and-dht-bootstrapping)
   - 2.5 [uTP — Micro Transport Protocol (BEP 29)](#25-utp--micro-transport-protocol-bep-29)
   - 2.6 [BitTorrent v2 (BEP 52)](#26-bittorrent-v2-bep-52)
3. [Introduction to libtorrent-rasterbar](#3-introduction-to-libtorrent-rasterbar)
4. [Magnet URIs and .torrent Metadata](#4-magnet-uris-and-torrent-metadata)
   - 4.1 [The Magnet URI Scheme (BEP 9)](#41-the-magnet-uri-scheme-bep-9)
   - 4.2 [BEP 9 — ut_metadata: Fetching Torrent Metadata from Peers](#42-bep-9--ut_metadata-fetching-torrent-metadata-from-peers)
   - 4.3 [libtorrent Magnet URI Helpers](#43-libtorrent-magnet-uri-helpers)
5. [libtorrent Programming Model](#5-libtorrent-programming-model)
   - 5.1 [Session Construction](#51-session-construction)
   - 5.2 [Adding Torrents and Magnet Links](#52-adding-torrents-and-magnet-links)
   - 5.3 [torrent_info and file_storage](#53-torrent_info-and-file_storage)
   - 5.4 [Torrent Status](#54-torrent-status)
   - 5.5 [Alert Polling Loop](#55-alert-polling-loop)
   - 5.6 [Resume Data and Session Persistence](#56-resume-data-and-session-persistence)
6. [libtorrent-rasterbar 2.x Streaming API](#6-libtorrent-rasterbar-2x-streaming-api)
   - 6.1 [Sequential Download Flag](#61-sequential-download-flag)
   - 6.2 [set_piece_deadline — The Streaming-Recommended API](#62-set_piece_deadline--the-streaming-recommended-api)
   - 6.3 [Alert Subscription and Piece Data Delivery](#63-alert-subscription-and-piece-data-delivery)
   - 6.4 [Partially Downloaded Files on Disk](#64-partially-downloaded-files-on-disk)
7. [The HTTP Bridge Pattern](#7-the-http-bridge-pattern)
   - 7.1 [WebTorrent createServer()](#71-webtorrent-createserver)
   - 7.2 [peerflix and torrent-stream](#72-peerflix-and-torrent-stream)
   - 7.3 [htorrent: HTTP-to-BitTorrent Gateway](#73-htorrent-http-to-bittorrent-gateway)
   - 7.4 [Stremio: Proprietary Bridge at Port 11470](#74-stremio-proprietary-bridge-at-port-11470)
8. [WebTorrent over WebRTC Data Channels](#8-webtorrent-over-webrtc-data-channels)
   - 8.1 [WebSocket Tracker Signaling](#81-websocket-tracker-signaling)
   - 8.2 [Browser ServiceWorker Server](#82-browser-serviceworker-server)
9. [VLC BitTorrent Plugin and MPV Integration](#9-vlc-bittorrent-plugin-and-mpv-integration)
   - 9.1 [vlc-bittorrent Plugin Architecture](#91-vlc-bittorrent-plugin-architecture)
   - 9.2 [Seek Implementation via Piece Priorities](#92-seek-implementation-via-piece-priorities)
   - 9.3 [MPV HTTP Bridge Hooks](#93-mpv-http-bridge-hooks)
10. [Post-Assembly Decode Pipeline](#10-post-assembly-decode-pipeline)
    - 10.1 [Compressed Bitstream to VA-API Decode](#101-compressed-bitstream-to-va-api-decode)
    - 10.2 [DMA-BUF Export and Zero-Copy Display](#102-dma-buf-export-and-zero-copy-display)
    - 10.3 [GStreamer souphttpsrc Pipeline](#103-gstreamer-souphttpsrc-pipeline)
11. [Zero-Copy Network Ingest: The Kernel Horizon](#11-zero-copy-network-ingest-the-kernel-horizon)
    - 11.1 [The Network Receive Bottleneck](#111-the-network-receive-bottleneck)
    - 11.2 [Device Memory TCP (Linux 6.12/6.16)](#112-device-memory-tcp-linux-612616)
    - 11.3 [io_uring ZC RX with DMA-BUF (Linux 6.15/6.16)](#113-io_uring-zc-rx-with-dma-buf-linux-615616)
    - 11.4 [GPUDirect RDMA for PCIe Video Capture](#114-gpudirect-rdma-for-pcie-video-capture)
12. [Integrations](#12-integrations)

---

## 1. The Streaming Seam

BitTorrent is a peer-to-peer file distribution protocol. By itself it has no GPU involvement. The point where it becomes relevant to the Linux graphics stack is the **streaming seam**: the moment a torrent engine has assembled enough contiguous bitstream data to hand off to a hardware video decoder. From that boundary onward, the pipeline is identical to any other media source — VA-API, Vulkan Video, DMA-BUF export, EGL import, Wayland compositor presentation — all documented in Chs 26, 50, and 57–58.

Two distinct paths connect torrent delivery to that seam:

1. **The HTTP bridge** — a local HTTP server exposes the in-progress torrent download as a seekable byte stream. Any standard media player or FFmpeg/GStreamer pipeline consumes it via `http://localhost/...` with HTTP Range requests.

2. **Native plugin integration** — `vlc-bittorrent` implements a VLC access module that calls the libtorrent API directly, bypassing HTTP entirely.

A third, forward-looking path concerns kernel infrastructure: Linux 6.12 introduced Device Memory TCP, which can receive TCP payloads directly into GPU-mapped memory without a CPU copy. No BitTorrent client implements this today, but the underlying kernel plumbing is the same mechanism that makes all zero-copy network-to-GPU ingest possible. §11 covers this path explicitly.

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

## 3. Introduction to libtorrent-rasterbar

[libtorrent-rasterbar](https://github.com/arvidn/libtorrent) is a C++ library (BSD license) implementing the BitTorrent protocol suite. It covers the full BEP stack: core wire protocol (BEP 3), DHT (BEP 5), Extension Protocol (BEP 10), PEX (BEP 11), uTP/LEDBAT (BEP 29), magnet link bootstrapping via `ut_metadata` (BEP 9), HTTP web seeds (BEP 19), BitTorrent v2 (BEP 52), protocol encryption, and more.

Applications built on libtorrent: qBittorrent, Deluge, and the `vlc-bittorrent` plugin (§9). The Stremio community replacement `perpetus/stream-server` (§7.4) also uses it.

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

## 4. Magnet URIs and .torrent Metadata

### 4.1 The Magnet URI Scheme (BEP 9)

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

### 4.2 BEP 9 — ut_metadata: Fetching Torrent Metadata from Peers

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

### 4.3 libtorrent Magnet URI Helpers

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

## 5. libtorrent Programming Model

### 5.1 Session Construction

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

### 5.2 Adding Torrents and Magnet Links

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

### 5.3 torrent_info and file_storage

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

### 5.4 Torrent Status

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

### 5.5 Alert Polling Loop

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

### 5.6 Resume Data and Session Persistence

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

## 6. libtorrent-rasterbar 2.x Streaming API

libtorrent-rasterbar (introduced in §3) is the dominant BitTorrent C++ library on Linux. Latest releases: v2.0.13 (2026-06-08, RC_2_0 branch) and v2.1.0 (2026-07-09, RC_2_1 branch). This section covers the streaming-specific API layered on top of the general programming model in §5.

### 6.1 Sequential Download Flag

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

### 6.2 set_piece_deadline — The Streaming-Recommended API

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

The vlc-bittorrent plugin (§9.2) uses priority 7/6/5 rather than deadlines.

### 6.3 Alert Subscription and Piece Data Delivery

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

### 6.4 Partially Downloaded Files on Disk

libtorrent writes each verified piece to `save_path/<torrent-name>/<file-path>` at the correct byte offset. Files are sparse: unverified piece ranges are holes. As soon as `piece_finished_alert` fires for a given piece, an external reader can `open()` and `pread()` at the correct offset without waiting for the full download.

Byte-offset computation from a piece index:

```cpp
// file_storage maps piece byte ranges to file offsets
auto slices = ti->map_block(piece_idx, 0, ti->piece_size(piece_idx));
// each file_slice: .file_index, .offset (bytes from file start), .size
```

The v2.0.x default disk backend is `mmap_disk_io` (kernel mmap, OS page cache manages flushing). v2.1.0 switches the default to `pread_disk_io` (preadv/pwritev on dedicated threads). Both write completed pieces immediately. [Source: libtorrent upgrade_to_2.1-ref.html](https://www.libtorrent.org/upgrade_to_2.1-ref.html)

---

## 7. The HTTP Bridge Pattern

The HTTP bridge is the universal integration mechanism: a local HTTP server exposes the in-progress torrent as a byte-seekable HTTP resource. Any tool that can open an HTTP URL — FFmpeg, GStreamer, VLC, mpv — consumes the torrent stream without BitTorrent awareness. The minimum contract is `Accept-Ranges: bytes` in the response and correct `206 Partial Content` + `Content-Range` headers for byte-range requests.

### 7.1 WebTorrent createServer()

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

### 7.2 peerflix and torrent-stream

[peerflix](https://github.com/mafintosh/peerflix) is a Node.js CLI that starts a local HTTP server on port 8888 and launches a player when the file is ready. It predates WebTorrent and uses the `torrent-stream` library as its download engine.

```bash
npm install -g peerflix
peerflix "magnet:?xt=..." --mpv         # launch mpv automatically
peerflix "magnet:?xt=..." -p 9000       # custom port
peerflix "video.torrent" --list         # list files in torrent
```

peerflix is community-maintained legacy software with no significant upstream commits in recent years. The underlying `torrent-stream` library does not use WebRTC. [Source: github.com/mafintosh/peerflix](https://github.com/mafintosh/peerflix)

### 7.3 htorrent: HTTP-to-BitTorrent Gateway

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

### 7.4 Stremio: Proprietary Bridge at Port 11470

[Stremio](https://www.stremio.com/) splits into an open-source UI layer and a proprietary streaming server:

- **`stremio-core`** (Rust, MIT): addon system, stream selection, UI models. [github.com/Stremio/stremio-core](https://github.com/Stremio/stremio-core)
- **`stremio-web`** (JS, GPL-2.0): browser/Electron UI. [github.com/Stremio/stremio-web](https://github.com/Stremio/stremio-web)
- **`server.js`** (proprietary): BitTorrent download engine, HTTP stream proxying, optional FFmpeg transcoding, subtitle fetching. Runs at `http://127.0.0.1:11470/`. Accepts `infoHash` + `fileIdx` parameters. Serves HLS endpoints (`master.m3u8`) for Chromecast-compatible output.
- **`stremio-shell`** (Qt/C++): embeds mpv via `mpv_render_context_create()` with `gpu-hwdec-interop=auto` and a 15,000 KB playback cache. Passes `http://127.0.0.1:11470/...` URLs to the embedded mpv. [Source: stremio-shell/mpv.cpp](https://github.com/Stremio/stremio-shell/blob/master/mpv.cpp)

An open-source Rust replacement for `server.js` — [perpetus/stream-server](https://github.com/perpetus/stream-server) — implements the same API endpoints using libtorrent, consuming roughly 50 MB versus `server.js`'s 200 MB+. A Python/FastAPI alternative — [andrewhack/stremio-libtorrent-server](https://github.com/andrewhack/stremio-libtorrent-server) — adds playhead-first piece picking.

---

## 8. WebTorrent over WebRTC Data Channels

WebTorrent extends BitTorrent to run over `RTCDataChannel` instead of TCP/uTP, enabling browser-native peer exchange. The BitTorrent wire protocol (handshake + piece exchange) is unchanged; only the transport layer differs. In Node.js, WebTorrent uses both TCP/uTP and WebRTC. In the browser, only WebRTC is available. [Source: WebTorrent FAQ](https://webtorrent.io/faq)

**Interoperability constraint:** Browser peers can only connect to seeders that support WebRTC — WebTorrent Desktop, BiglyBT with WebTorrent plugin, Vuze with WebTorrent plugin. Standard TCP/uTP clients cannot serve browser peers without WebRTC support.

### 8.1 WebSocket Tracker Signaling

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

### 8.2 Browser ServiceWorker Server

In the browser, `createServer({ controller: swRegistration })` installs a ServiceWorker that intercepts `fetch()` calls to the `/webtorrent/<infoHash>/` path and serves file bytes from the in-memory torrent buffer. This enables standard `<video src="...">` playback of in-progress downloads without a Node.js backend. Supported video containers and codecs are subject to the browser's native codec support: `mp4`, `webm`, `mkv`, `ogv`, `mov` with AV1, H.264, HEVC, VP8, VP9. [Source: WebTorrent API docs](https://webtorrent.io/docs)

---

## 9. VLC BitTorrent Plugin and MPV Integration

### 9.1 vlc-bittorrent Plugin Architecture

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

### 9.2 Seek Implementation via Piece Priorities

When VLC's demuxer seeks to a byte offset, the plugin's seek callback (in `download.cpp`) performs:

1. Maps the file byte offset to torrent piece indices via `ti->map_file(file_index, file_offset, length)`.
2. Sets graduated libtorrent piece priorities:
   - Priority 7 (highest): pieces covering the requested data range.
   - Priority 6: first/last 0.1% or 128 KB of the file (container headers/trailers, needed by the demuxer).
   - Priority 5: next 5% or 32 MB around the seek position.
3. Calls `m_th.read_piece(piece_index)` for async piece retrieval.
4. Subscribes to `piece_finished_alert` and `read_piece_alert` via an internal `AlertSubscriber` to deliver bytes back to VLC's demuxer without polling.

This architecture keeps seeking latency proportional to swarm availability at the seek target, not to overall download progress.

### 9.3 MPV HTTP Bridge Hooks

mpv has no native BitTorrent support. It consumes torrent content via the HTTP bridge, issuing `Range: bytes=0-` on the first request to probe seeking capability — if the server returns `206 Partial Content`, mpv uses byte-range seeks throughout playback. [Source: mpv issue #655](https://github.com/mpv-player/mpv/issues/655), [mpv issue #8145](https://github.com/mpv-player/mpv/issues/8145)

Two mpv hook scripts automate the bridge:

- **mpv-peerflix-hook** ([noctuid/mpv-peerflix-hook](https://github.com/noctuid/mpv-peerflix-hook)): shell/Lua script that intercepts magnet URIs passed to mpv, forks peerflix in the background, extracts the HTTP URL from its output, and redirects mpv to that URL. Handles port conflict detection and kills the peerflix process when mpv exits.

- **webtorrent-mpv-hook** ([mrxdst/webtorrent-mpv-hook](https://github.com/mrxdst/webtorrent-mpv-hook)): TypeScript npm package installed as an mpv script. Accepts magnet links, info-hashes, or `.torrent` files directly as mpv arguments. Displays a download-progress overlay; `p` key toggles it. Uses WebTorrent internally.

---

## 10. Post-Assembly Decode Pipeline

Once the HTTP bridge or native plugin delivers a byte-range-seekable bitstream, the rest of the pipeline is identical to any other media source. This section cross-references the relevant chapters rather than repeating their content.

### 10.1 Compressed Bitstream to VA-API Decode

With peerflix or WebTorrent's `createServer()` running on port 8000, FFmpeg reads and hardware-decodes directly:

```bash
ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -hwaccel_output_format vaapi \
       -i "http://localhost:8000/webtorrent/<infoHash>/video.mkv" \
       -f null -
```

The `AVHWFramesContext` for VAAPI causes decoded frames to remain in VA surfaces without any CPU memcpy. See Ch57 §3 for the `AVHWDeviceContext` lifecycle and Ch26 §4 for the VA-API decode surface model.

### 10.2 DMA-BUF Export and Zero-Copy Display

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

### 10.3 GStreamer souphttpsrc Pipeline

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

## 11. Zero-Copy Network Ingest: The Kernel Horizon

### 11.1 The Network Receive Bottleneck

The zero-copy chain described in §10 — VA-API decode → DMA-BUF → EGL/Vulkan → compositor — is deployed and zero-copy today. The remaining copy is at the other end of the pipeline: network receive. libtorrent and all current BitTorrent clients receive TCP packet payloads via `recv()` into system RAM (kernel `sk_buff` → userspace buffer). The compressed bitstream then passes CPU-side to the hardware decoder. Eliminating this copy requires kernel infrastructure that landed in Linux 6.12–6.16.

### 11.2 Device Memory TCP (Linux 6.12/6.16)

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

### 11.3 io_uring ZC RX with DMA-BUF (Linux 6.15/6.16)

**Linux 6.0** introduced `IORING_OP_SEND_ZC` — zero-copy socket *send* from a registered user buffer.

**Linux 6.15** (April 2025) introduced `IORING_OP_RECV_ZC` (io_uring zcrx) — zero-copy socket *receive*. Incoming packet payloads DMA directly into a userspace-registered memory region (a page pool provisioned from user pages). Headers and payloads are split at L4; completions arrive via `io_uring_zcrx_cqe` entries with buffer offsets. Saturates a 200 Gbit/s link on a single CPU core. Requires `IORING_SETUP_CQE32`. Currently supports **host memory only**. [Source: kernel iou-zcrx docs](https://docs.kernel.org/networking/iou-zcrx.html), [Phoronix: Linux 6.15 io_uring](https://www.phoronix.com/news/Linux-6.15-IO_uring)

**Linux 6.16 target:** DMA-BUF support for io_uring zcrx, allowing a DMA-BUF fd (including a CUDA-exported GPU allocation) to serve as the receive region. This is the io_uring analog to Device Memory TCP's `netdev_bind_rx()`. [Source: Phoronix: io_uring ZCRX DMA-BUF](https://www.phoronix.com/news/IO_uring-ZCRX-DMA-BUF)

No BitTorrent client implements `IORING_OP_RECV_ZC` as of mid-2026.

### 11.4 GPUDirect RDMA for PCIe Video Capture

GPUDirect RDMA allows a third-party PCIe device (InfiniBand or RoCE HCA, capture card, FPGA) to DMA directly into GPU VRAM via `nvidia-peermem.ko`. The nv-p2p kernel API (`nvidia_p2p_get_pages`, `nvidia_p2p_dma_map_pages`) used by third-party drivers is deprecated in CUDA 13.0 and will be removed in CUDA 14.0; replacements are standard upstream `pin_user_pages()` + `dma_map_sgtable()`. [Source: GPUDirect RDMA docs](https://docs.nvidia.com/cuda/gpudirect-rdma/index.html)

GPUDirect RDMA for video ingest is deployed in two contexts:

- **NVIDIA Holoscan SDK + AJA NTV2**: The AJA NTV2 SDK 16.1 added GPUDirect support. A 4K UHD RGBA capture card DMA-writes frames directly into CUDA GPU memory over PCIe. Measured latency drops from 23.7 ms (CPU path, exceeds 16.7 ms frame budget at 60 fps) to 12.4 ms (GPUDirect PCIe path). [Source: NVIDIA Holoscan blog](https://developer.nvidia.com/blog/supporting-low-latency-streaming-video-for-ai-powered-medical-devices-with-clara-holoscan/)
- **Deltacast VideoMaster SDK 6.20+**: GPUDirect RDMA on Linux x86_64 and ARM64 for broadcast I/O cards. [Source: Deltacast GPUDirect page](https://www.deltacast.tv/technologies/gpudirect-rdma/)

In both cases the "RDMA" is PCIe peer-to-peer between a capture card and the GPU — not an InfiniBand or RoCE network path. GPUDirect RDMA over InfiniBand/RoCE is used at scale for HPC distributed training (NCCL, NVSHMEM), not for media streaming. No BitTorrent client or torrent-backed media pipeline uses GPUDirect RDMA. The connection raised earlier in this chapter — RDMA as grounds for torrent streaming's relevance to this book — is accurate at the infrastructure level: the same kernel p2pdma and DMA-BUF mechanisms that enable PCIe capture-to-GPU paths are the foundation of Device Memory TCP and io_uring ZC RX, which are the paths that could eventually connect network delivery (including BitTorrent) to GPU decode without a CPU bounce, as covered in §11.2–11.3.

GPUDirect Storage (GDS, via `cuFile`) is a storage-only technology covering NVMe local, NVMe-oF, NFS, Lustre, and WekaFS. It has no socket or TCP path and is unrelated to BitTorrent. [Source: GDS Overview Guide](https://docs.nvidia.com/gpudirect-storage/overview-guide/index.html)

---

## 12. Integrations

- **Ch26 (VA-API)**: Hardware video decode of the bitstream assembled by libtorrent or delivered via HTTP bridge. `vaExportSurfaceHandle()` DMA-BUF export is the zero-copy path from decoder to display.
- **Ch50 (Vulkan Video)**: Alternative to VA-API for decode; Vulkan Video decode surfaces can be imported as DMA-BUF fds via `VK_EXT_external_memory_dma_buf`, same chain as §10.2.
- **Ch57 (FFmpeg)**: `AVHWDeviceContext` for VAAPI consumes the HTTP bridge via `-i http://localhost:...`; the `AVVkFrame` path (Vulkan hwaccel) applies equally.
- **Ch58 (GStreamer)**: `souphttpsrc` consumes the HTTP bridge; `vah264dec`/`vaav1dec` with DMABuf caps negotiation provides the zero-copy decode-to-display chain.
- **Ch60b (Streaming Protocols and ABR)**: WebTorrent's WebRTC transport uses the same RTCDataChannel / ICE / DTLS-SRTP infrastructure documented there. WebSocket tracker signaling is the BitTorrent-specific signaling mechanism layered on top of that transport.
- **Ch60c (WebRTC Server Infrastructure)**: Public WebSocket trackers (`wss://tracker.openwebtorrent.com`) serve the same role for WebTorrent peers that STUN/TURN servers serve for WebRTC media connections.
- **Ch189 (VLC)**: `vlc-bittorrent` is a VLC access module that integrates libtorrent-rasterbar directly into VLC's demuxer callback chain.
- **Ch4 (DRM Architecture / DMA-BUF)**: The DMA-BUF fd produced by `vaExportSurfaceHandle()` and consumed by EGL/Vulkan/Wayland is the same kernel DMA-BUF primitive. Device Memory TCP's `netdev_bind_rx()` uses the same fd type.
- **Ch49 (Multi-GPU and p2pdma)**: GPUDirect RDMA's PCIe peer-to-peer DMA path (capture card → GPU) uses the same `p2pdma` kernel framework. The Linux 6.2+ PCI P2PDMA path for GDS also uses this infrastructure.
