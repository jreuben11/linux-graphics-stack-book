# Chapter 60e: Media Source Extensions and Encrypted Media Extensions

*Audience: Browser and web platform engineers implementing or debugging playback pipelines; graphics application developers who need to understand why premium video is capped or blocked on Linux desktops.*

## Table of Contents

1. [MSE Core API](#1-mse-core-api)
   - 1.1 [The MediaSource Object Model](#11-the-mediasource-object-model)
   - 1.2 [SourceBuffer: Append, Remove, Buffered](#12-sourcebuffer-append-remove-buffered)
   - 1.3 [Codec Gating and Quota Eviction](#13-codec-gating-and-quota-eviction)
   - 1.4 [The ISO BMFF Byte Stream Format](#14-the-iso-bmff-byte-stream-format)
   - 1.5 [The npm ISOBMFF Ecosystem](#15-the-npm-isobmff-ecosystem)
2. [MSE in Player Libraries](#2-mse-in-player-libraries)
   - 2.1 [Shaka Player, hls.js, dash.js](#21-shaka-player-hlsjs-dashjs)
   - 2.2 [The Fetch-Append-Evict Loop](#22-the-fetch-append-evict-loop)
   - 2.3 [segments vs. sequence Mode](#23-segments-vs-sequence-mode)
   - 2.4 [The npm Ecosystem for HTTP Adaptive Streaming](#24-the-npm-ecosystem-for-http-adaptive-streaming)
3. [Recent MSE Evolution](#3-recent-mse-evolution)
   - 3.1 [MSE-in-Workers and MediaSourceHandle](#31-mse-in-workers-and-mediasourcehandle)
   - 3.2 [ManagedMediaSource](#32-managedmediasource)
4. [The EME Handshake](#4-the-eme-handshake)
   - 4.1 [Capability Negotiation](#41-capability-negotiation)
   - 4.2 [Sessions, Licenses, and Key Status](#42-sessions-licenses-and-key-status)
   - 4.3 [Clear Key: The Reference CDM](#43-clear-key-the-reference-cdm)
5. [Common Encryption (CENC)](#5-common-encryption-cenc)
   - 5.1 [The Four Schemes, Two in Practice](#51-the-four-schemes-two-in-practice)
   - 5.2 [pssh, tenc, senc](#52-pssh-tenc-senc)
   - 5.3 [Multi-DRM Packaging](#53-multi-drm-packaging)
6. [CDM Architecture and Linux Realities](#6-cdm-architecture-and-linux-realities)
   - 6.1 [Widevine Security Levels L1/L2/L3](#61-widevine-security-levels-l1l2l3)
   - 6.2 [Why Desktop Linux Is Capped at L3](#62-why-desktop-linux-is-capped-at-l3)
   - 6.3 [Chromium's Bundled CDM](#63-chromiums-bundled-cdm)
   - 6.4 [Firefox: Widevine as a Gecko Media Plugin](#64-firefox-widevine-as-a-gecko-media-plugin)
   - 6.5 [WebKitGTK/WPE: CDMProxy and Thunder/OpenCDM](#65-webkitgtkwpe-cdmproxy-and-thunderopencdm)
   - 6.6 [Out of Scope](#66-out-of-scope)
7. [MSE/DASH-HLS vs. WebRTC: When to Use Which](#7-msedash-hls-vs-webrtc-when-to-use-which)
8. [Building a Video App with Shaka Player](#8-building-a-video-app-with-shaka-player)
   - 8.1 [Project Setup and Basic Playback](#81-project-setup-and-basic-playback)
   - 8.2 [Configuration: ABR, Buffering, and Network Retry](#82-configuration-abr-buffering-and-network-retry)
   - 8.3 [Error Handling](#83-error-handling)
   - 8.4 [Wiring Up DRM: License Servers, Authentication, and Wrapping](#84-wiring-up-drm-license-servers-authentication-and-wrapping)
   - 8.5 [Widevine Service Certificates](#85-widevine-service-certificates)
   - 8.6 [The UI Library](#86-the-ui-library)
   - 8.7 [MoQ: Media over QUIC (Experimental)](#87-moq-media-over-quic-experimental)
9. [Integrations](#9-integrations)

---

## 1. MSE Core API

### 1.1 The MediaSource Object Model

Media Source Extensions (MSE) let JavaScript hand a browser's media pipeline compressed bytes instead of a URL, which is what makes adaptive-bitrate players possible on the open web: the player, not the browser's network stack, decides which segment to fetch next. The `MediaSource` object is a `MediaProvider` — historically attached via `URL.createObjectURL(mediaSource)` assigned to `<video>.src`, and in newer engines attached directly through `HTMLMediaElement.srcObject = mediaSource` (the same attachment point used for `MediaStream` from `getUserMedia()`, see [Chapter 60b](ch60b-video-streaming-protocols.md)). [Source: MSE Recommendation §2 MediaSource Object](https://www.w3.org/TR/media-source/#mediasource)

A `MediaSource` moves through a small state machine exposed as `readyState`:

- `closed` — not attached to a media element, or detached.
- `open` — attached, ready to accept `addSourceBuffer()` calls and appends.
- `ended` — the app called `endOfStream()`; no further segments will arrive unless the app reopens it.

Transitions fire `sourceopen`, `sourceended`, and `sourceclose` events on the `MediaSource` object itself — the events a player's buffering controller listens for before creating its first `SourceBuffer`. [Source: MSE Recommendation §2.4 Attributes and §2.5 Event Summary](https://www.w3.org/TR/media-source/#idl-def-mediasource)

```javascript
const mediaSource = new MediaSource();
video.src = URL.createObjectURL(mediaSource);

mediaSource.addEventListener('sourceopen', () => {
  const sourceBuffer = mediaSource.addSourceBuffer(
    'video/mp4; codecs="avc1.640028"'
  );
  fetch('init-segment.mp4')
    .then(r => r.arrayBuffer())
    .then(buf => sourceBuffer.appendBuffer(buf));
});
```

### 1.2 SourceBuffer: Append, Remove, Buffered

`MediaSource.addSourceBuffer(mimeType)` creates one `SourceBuffer` per elementary stream the app intends to feed — typically one for video, one for audio, though a single muxed buffer is also legal. [Source: MSE Recommendation §3 SourceBuffer Object](https://www.w3.org/TR/media-source/#sourcebuffer) Three operations dominate the API surface a player actually drives at runtime:

- **`appendBuffer(ArrayBuffer | ArrayBufferView)`** — asynchronously parses and queues the passed bytes (an init segment or a media segment) for decode. It is illegal to call again while `updating === true`; players serialize appends through an `updateend` listener or an internal queue.
- **`remove(start, end)`** — evicts a time range from the buffer, used by ABR controllers to discard already-played or stale-quality segments and reclaim memory.
- **`buffered`** — a `TimeRanges` object describing exactly what spans of media are currently appended and decodable; this is the ground truth an ABR loop consults before deciding whether to fetch the next segment, not `HTMLMediaElement.buffered` in isolation.

[Source: MSE Recommendation §3.1 Attributes, §3.2 Methods](https://www.w3.org/TR/media-source/#sourcebuffer-attributes)

### 1.3 Codec Gating and Quota Eviction

Before committing to a codec, a player calls the static `MediaSource.isTypeSupported(mimeType)` — the same codec-string mechanism used elsewhere in the platform (`canPlayType()`, WebCodecs' `VideoDecoder.isConfigSupported()`, see [Chapter 146](../part-10-browser-rendering-stack/ch146-webcodecs-browser-hardware-acceleration.md)) — to decide whether, say, `video/mp4; codecs="hev1.2.4.L153.B0"` (HEVC) is decodable before wasting a network round-trip on a segment it cannot play. [Source: MSE Recommendation §2.4.1 isTypeSupported()](https://www.w3.org/TR/media-source/#dom-mediasource-istypesupported)

Buffer memory is finite and the spec deliberately does not pin down how much: eviction policy, total quota, and per-tab limits are left to the implementation. A player that appends without ever calling `remove()` will eventually receive a `QuotaExceededError` from `appendBuffer()`, at which point it is expected to evict old ranges itself and retry — MSE gives no automatic garbage collection guarantee, only the tools to implement one. [Source: MSE Recommendation §2.4.3 Sourcebuffer Monitoring, quota discussion](https://www.w3.org/TR/media-source/#quota)

### 1.4 The ISO BMFF Byte Stream Format

MSE itself is container-agnostic — the `MediaSource` and `SourceBuffer` interfaces in §1.1–1.2 say nothing about box layout. The actual bytes a `SourceBuffer` will accept for `video/mp4`/`audio/mp4` MIME types are pinned down in a companion spec, the **ISO BMFF Byte Stream Format**, which layers MSE-specific constraints on top of the generic container format defined by ISO/IEC 14496-12 ("ISO base media file format"). Understanding this layering matters in practice: a file can be perfectly valid ISO/IEC 14496-12 and still be rejected by `appendBuffer()`, because the byte stream format spec adds requirements the base standard leaves optional. [Source: ISO BMFF Byte Stream Format spec §1 Introduction](https://www.w3.org/TR/mse-byte-stream-format-isobmff/#introduction)

**Initialization segment structure.** Every `SourceBuffer` must receive exactly one initialization segment before any media segment, an `ftyp` + `moov` box pair carrying codec configuration and zero samples:

- **`ftyp`** — the File Type box, declaring the ISOBMFF "brand" (e.g. `iso5`, `mp42`) the segment conforms to; the same box type CMAF packaging already writes for CDN delivery.
- **`moov`** (Movie box) → **`mvhd`** (overall movie header; largely vestigial for fragmented content, since real duration comes from the segment stream, not this box) → one **`trak`** per elementary stream → `tkhd`/`mdia`/`minf`/**`stbl`** (sample table), whose **`stsd`** (Sample Description box) carries the codec configuration record — an `avcC`, `hvcC`, or `av1C` box nested inside an `avc1`/`hev1`/`av01` sample entry — that must exactly match the `codecs` parameter string passed to `addSourceBuffer()` (§1.2); a mismatch is a decode error, not a silent fallback.
- **`mvex`** (Movie Extends box), inside `moov` — the box whose mere presence marks the file as *fragmented* rather than a conventional flat MP4; it holds one **`trex`** per track supplying default sample duration/size/flags that a `trun` box (below) may omit to save bytes.

**Media segment structure.** Every segment after the first is a self-contained, independently appendable fragment:

- **`styp`** (Segment Type box, optional) — `ftyp`'s counterpart for a segment rather than a whole file; commonly present in DASH/CMAF output, tolerated but not required by MSE.
- **`sidx`** (Segment Index box, optional) — a byte-offset/duration index into the segments that follow; not consumed by MSE's parser directly, but ubiquitous in fMP4-packaged HLS and DASH because it is what lets a CDN or player do byte-range addressing into a single large segment file (see [Chapter 60b §9](ch60b-video-streaming-protocols.md)).
- **`moof`** (Movie Fragment box) → **`mfhd`** (fragment sequence number) + one **`traf`** per track → **`tfhd`** (Track Fragment Header — track ID plus optional per-fragment overrides of the `trex` defaults) + **`tfdt`** (Track Fragment Decode Time — the fragment's absolute decode timestamp) + **`trun`** (Track Run — per-sample size, duration, flags, and composition-time offset for every sample in the fragment).
- **`mdat`** (Media Data box) — the raw encoded sample bytes that `trun`'s per-sample table describes.

The byte stream format spec adds one binding constraint here that base ISOBMFF leaves optional: **`tfdt` is mandatory in every `traf`.** MSE's `"segments"` mode (§2.3) places each media segment on the presentation timeline using exactly this decode timestamp, so a fragment without `tfdt` has no way to declare where it belongs — a producer targeting MSE cannot omit it the way a producer targeting only local file playback sometimes can. [Source: ISO BMFF Byte Stream Format spec §3.3 Segments](https://www.w3.org/TR/mse-byte-stream-format-isobmff/#iso-segments) Random-access points (the fragments a player may start decoding from after a seek) are signaled per-sample via `trun`'s sample-flags field rather than as a separate box, which is why a player must inspect `trun` — not just fragment boundaries — before it can seek safely mid-stream.

**Inspecting segments.** Two command-line tools dump the box tree directly rather than requiring a hex editor: GPAC's `MP4Box -info init.mp4` and `MP4Box -info segment.m4s` print a human-readable box hierarchy with field values (track IDs, timescales, `tfdt` baseMediaDecodeTime, sample counts per `trun`); Bento4's `mp4dump segment.m4s` produces an equivalent tree from an independent implementation, useful for cross-checking a packager's output against two parsers before assuming a playback failure is a browser bug rather than a malformed segment. [Source: GPAC MP4Box documentation](https://gpac.io/tools/mp4box/) · [Bento4 mp4dump](https://www.bento4.com/documentation/mp4dump/)

### 1.5 The npm ISOBMFF Ecosystem

Because the byte stream format in §1.4 is just a well-defined box layout, a small set of JavaScript packages exist purely to read or write it, independent of any particular ABR player. These sit one layer below the player libraries in §2.1 — Shaka Player, hls.js, and dash.js all either depend on one of these or reimplement an equivalent internally:

- **`mp4box`** ([npm](https://www.npmjs.com/package/mp4box), source [gpac/mp4box.js](https://github.com/gpac/mp4box.js)) — the JavaScript port of GPAC's MP4Box tool referenced in §1.4's tooling note: a full ISOBMFF **parser and segmenter** capable of both reading an existing fMP4 file's box tree in-browser and re-fragmenting a flat MP4 into MSE-appendable init/media segments client-side. It is one of the most-downloaded packages in this space and is what several browser-based video-editing and DASH-conformance tools use to avoid a server round-trip for repackaging.
- **`codem-isoboxer`** ([npm](https://www.npmjs.com/package/codem-isoboxer), source [madebyhiro/codem-isoboxer](https://github.com/madebyhiro/codem-isoboxer)) — a lightweight, **parse-only** box reader with no muxing side. It is a direct runtime dependency of **dash.js** itself, used internally to walk `sidx`/`moof`/`traf` boxes and extract `pssh` (§5.2) key-IDs during CENC handling — confirmed in dash.js's own `package.json` dependency list. [Source: dash.js package.json](https://github.com/Dash-Industry-Forum/dash.js/blob/development/package.json)
- **`mux.js`** ([npm](https://www.npmjs.com/package/mux.js), source [videojs/mux.js](https://github.com/videojs/mux.js)) — Brightcove/video.js's **MPEG-TS-to-fMP4 remuxer**. Legacy and many live HLS origins still serve MPEG-TS segments, which `SourceBuffer` cannot accept directly under the ISOBMFF byte stream format (§1.4) — `mux.js` demuxes the TS elementary streams and re-muxes them into spec-conformant `ftyp`/`moov`/`moof`/`mdat` boxes entirely in the browser. It is a pinned direct dependency of `@videojs/http-streaming`, the HLS/DASH engine behind video.js. [Source: videojs/http-streaming package.json](https://github.com/videojs/http-streaming/blob/main/package.json)
- **`mp4-muxer`** ([npm](https://www.npmjs.com/package/mp4-muxer), source [Vanilagy/mp4-muxer](https://github.com/Vanilagy/mp4-muxer)) — a modern, pure-TypeScript **muxer** purpose-built to sit downstream of the WebCodecs API (see [Chapter 146](../part-10-browser-rendering-stack/ch146-webcodecs-browser-hardware-acceleration.md)): raw `EncodedVideoChunk`/`EncodedAudioChunk` output from `VideoEncoder`/`AudioEncoder` in, spec-conformant fragmented ISOBMFF out, ready either to feed `SourceBuffer.appendBuffer()` or to download as a standalone `.mp4`. Its sibling package `webm-muxer`, from the same author, covers the WebM/Matroska container for the codec/container combinations ISOBMFF does not serve.

**hls.js is the outlier worth noting**: it ships with zero runtime dependencies and instead implements its own minimal ISOBMFF box-writer inline (`mp4-generator` in its source tree) rather than depending on `mux.js` or any of the above — a deliberate bundle-size tradeoff distinct from video.js's choice to depend on the shared `mux.js` package. [Source: video-dev/hls.js package.json](https://github.com/video-dev/hls.js/blob/master/package.json)

---

## 2. MSE in Player Libraries

### 2.1 Shaka Player, hls.js, dash.js

MSE is a low-level primitive; virtually no production site drives `SourceBuffer` directly. Three open-source libraries do the ABR logic, manifest parsing, and buffer bookkeeping on top of it:

- **Shaka Player** (Google, TypeScript) — supports both DASH and HLS manifests against one internal pipeline; its buffer-management layer is `shaka.media.MediaSourceEngine`, which wraps `SourceBuffer` operations in promises and serializes them against `updating`.
- **hls.js** — HLS-only, TypeScript; its `BufferController` owns the `SourceBuffer` lifecycle and reacts to level-switch events from the ABR controller by adjusting which segments get appended next.
- **dash.js** — the DASH Industry Forum's reference player, JavaScript; similarly centralizes append/evict decisions behind its own buffer controller.

All three exist because a real player needs far more than MSE provides: manifest parsing (MPD/M3U8), ABR heuristics, DRM session orchestration (§4), gap-jumping over minor segment discontinuities, and codec-switch handling — MSE supplies only the mechanism to get bytes into the decode pipeline. [Source: Shaka Player GitHub](https://github.com/shaka-project/shaka-player) · [hls.js GitHub](https://github.com/video-dev/hls.js) · [dash.js GitHub](https://github.com/Dash-Industry-Forum/dash.js)

**Comparing the three.** The libraries differ enough in scope and maintainership that "just pick one" is rarely the right framing:

| | hls.js | dash.js | Shaka Player |
|---|---|---|---|
| Manifest formats | HLS only | DASH only | DASH **and** HLS, one internal pipeline |
| Maintainer | Community (video-dev, originating at Dailymotion) | DASH Industry Forum (spec consortium) | Google |
| Runtime dependencies | Zero — own inline ISOBMFF box-writer (§1.5) | `codem-isoboxer` for box parsing (§1.5) | Self-contained |
| Primary role | The de facto HLS engine — embedded inside video.js's `@videojs/http-streaming` and countless HLS-only players | Reference/conformance implementation — the player DASH spec authors point to when demonstrating correct behavior | General-purpose production player |
| DRM/multi-DRM breadth | EME support exists but narrower in practice, mainly FairPlay-oriented deployments | CENC-oriented (§5), since DASH is the format multi-DRM packaging chiefly targets | Broadest of the three — Widevine, PlayReady, and FairPlay all first-class, plus offline storage via IndexedDB |
| Typical fit | Sites that only need HLS, or want the smallest footprint | Reference testing and DASH-only conformance work rather than general production use | Production OTT/premium video needing both formats and full DRM coverage from one codebase |

The practical split: **hls.js** wins on minimalism for HLS-only delivery; **dash.js** is reached for chiefly to validate DASH-spec conformance rather than to ship a general-purpose player; **Shaka Player** is what production teams pick when they need both manifest formats and full DRM coverage maintained by a single, actively-developed vendor codebase — which is also why it is the heaviest of the three to bundle. [Source: hls.js package.json](https://github.com/video-dev/hls.js/blob/master/package.json) · [dash.js package.json](https://github.com/Dash-Industry-Forum/dash.js/blob/development/package.json) · [Shaka Player GitHub](https://github.com/shaka-project/shaka-player)

### 2.2 The Fetch-Append-Evict Loop

Every one of these players runs the same core loop: the ABR controller picks a bitrate rendition for the next segment based on measured throughput and current `buffered` depth; the player fetches that segment over HTTP; on arrival, it calls `appendBuffer()`; once `updateend` fires, it checks `buffered` again to decide whether to fetch further ahead or pause fetching because the buffer is already deep enough; periodically it calls `remove()` on ranges behind the playhead to keep memory bounded. The loop is entirely userland JavaScript — MSE contributes only the four primitives in §1.2, not the scheduling policy around them.

### 2.3 segments vs. sequence Mode

`SourceBuffer.mode` controls how appended media segments are placed on the timeline. In `"segments"` mode (the default), each segment's own presentation timestamps determine where it lands — required for DASH, where segments carry absolute or well-defined relative timestamps. In `"sequence"` mode, segments are placed back-to-back in append order regardless of their embedded timestamps, which is what makes MSE tolerate certain live-HLS discontinuities (ad insertion, encoder restarts) without the player having to rewrite timestamps itself. [Source: MSE Recommendation §3.1 mode attribute](https://www.w3.org/TR/media-source/#dom-sourcebuffer-mode)

### 2.4 The npm Ecosystem for HTTP Adaptive Streaming

§1.5 covered the packages that only read or write ISOBMFF boxes. This section covers the layer above: npm packages that implement HTTP adaptive streaming end to end — DASH/HLS manifest handling, ABR, and MSE buffer management on the **client**, and DASH/HLS segmentation and packaging on the **server**.

**Client: full players.** §2.1 already introduced Shaka Player, hls.js, and dash.js as libraries; their npm distributions are worth naming explicitly because the package name does not always match the project name:

- **`dashjs`** ([npm](https://www.npmjs.com/package/dashjs), source [Dash-Industry-Forum/dash.js](https://github.com/Dash-Industry-Forum/dash.js)) — note the npm package name drops the dot that the GitHub repository name ("dash.js") uses. Published by the DASH Industry Forum as "a reference client implementation for the playback of MPEG DASH via Javascript and compliant browsers."
- **`shaka-player`** ([npm](https://www.npmjs.com/package/shaka-player), source [shaka-project/shaka-player](https://github.com/shaka-project/shaka-player)) — Google's DASH/HLS/EME player library, the same engine referenced in §2.1.
- **`rx-player`** ([npm](https://www.npmjs.com/package/rx-player), source [canalplus/rx-player](https://github.com/canalplus/rx-player)) — Canal+'s open-source player, supporting DASH, Smooth Streaming, and (since v4) HLS against a shared MSE-driven core; less widely embedded than the other two but notable as a second production-grade European broadcaster implementation independent of Shaka's and dash.js's codebases.
- **`hls.js`** — already covered in §2.1 and §1.5; the npm package name matches the project name.

All four expose broadly the same shape of API to an integrating app: point the player at a manifest URL, attach it to a `<video>` element, and it owns the `MediaSource`/`SourceBuffer` lifecycle from §1–§2.3 internally. Framework wrapper packages (`hls-video-element`, `dash-video-element`) exist on top of several of these to expose the player as a custom element rather than an imperative API, but they are thin adapters, not independent implementations.

**Server: packaging and segmentation.** Producing conformant DASH/HLS output — the manifests and fMP4/TS segments the clients above consume — is generally native-binary work (ffmpeg, GPAC, Shaka Packager itself); npm's role here is almost entirely as a thin wrapper or companion utility around those binaries rather than a pure-JavaScript reimplementation:

- **`shaka-packager`** ([npm](https://www.npmjs.com/package/shaka-packager), source [shaka-project/shaka-packager](https://github.com/shaka-project/shaka-packager)) — an npm-distributed wrapper around Google's Shaka Packager binary. Verified directly from its README: "a media packaging tool and SDK... for DASH and HLS packaging and encryption," producing fragmented ISO-BMFF (§1.4) output "compatible with Media Source Extensions" and supporting CENC (§5) "alongside SAMPLE-AES, supporting multiple DRM systems including Widevine, PlayReady, FairPlay, and Marlin." This is the closest thing in the npm registry to a general-purpose, standards-conformant DASH+HLS origin packager. [Source: shaka-project/shaka-packager README](https://github.com/shaka-project/shaka-packager)
- **`fluent-ffmpeg`** ([npm](https://www.npmjs.com/package/fluent-ffmpeg), source [fluent-ffmpeg/node-fluent-ffmpeg](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg)) — a generic fluent API over the `ffmpeg` CLI, not DASH/HLS-specific itself. It is nevertheless the most common way Node backends script HTTP adaptive streaming segmentation, by driving ffmpeg's own native `-f hls` and `-f dash` muxers (segment duration, playlist/manifest writing, variant-stream layout) rather than reimplementing packaging logic in JavaScript.
- **`m3u8-parser`** ([npm](https://www.npmjs.com/package/m3u8-parser), source [videojs/m3u8-parser](https://github.com/videojs/m3u8-parser)) and **`mpd-parser`** ([npm](https://www.npmjs.com/package/mpd-parser), source [videojs/mpd-parser](https://github.com/videojs/mpd-parser)) — video.js's HLS and DASH manifest parsers, used internally by `@videojs/http-streaming` (§1.5) on the client, but also common server-side for manifest validation, origin-side inspection, or transforming a manifest between requests (e.g. stitching in ad breaks) without a full player attached.

**A popular package that does *not* belong here.** `node-media-server`, one of the most widely used Node.js media servers, is deliberately excluded from this list: verification against its current README and source (no `hls`-matching code found via repository code search as of this writing) shows it implements RTMP/FLV ingest and playback — push/pull over RTMP, FLV over HTTP/WebSocket — with no DASH or HLS packaging in its current codebase, despite its name suggesting broader scope. Treating it as an HLS/DASH tool would be an overclaim; sites needing RTMP ingest transcoded to HLS/DASH pair a server like this with `fluent-ffmpeg` or Shaka Packager downstream instead of expecting it natively.

---

## 3. Recent MSE Evolution

MSE v1 reached W3C Recommendation status on 2016-11-17 and has been stable since. A second edition, **MSE v2** (formally "Media Source Extensions™"), remains a W3C Working Draft — last published 2026-08-07 — that normatively folds in two features that shipped in browsers years before the spec caught up: MSE-in-Workers and `ManagedMediaSource`. [Source: Media Source Extensions™ Level 2 Working Draft](https://www.w3.org/TR/media-source-2/)

### 3.1 MSE-in-Workers and MediaSourceHandle

Originally, `MediaSource` could only be constructed on the main thread, which meant every `appendBuffer()` call — parsing box headers, running eviction logic — competed with layout, style, and script execution for main-thread time, a real source of jank during aggressive ABR churn. MSE-in-Workers lets a dedicated worker construct the `MediaSource` and do all of that off-thread; `MediaSource.handle` returns a transferable `MediaSourceHandle`, which the worker posts back to the main thread via `postMessage()`, where it is assigned to `HTMLMediaElement.srcObject` for attachment.

```javascript
// inside a DedicatedWorker
const mediaSource = new MediaSource();
const handle = mediaSource.handle;
postMessage({ handle }, [handle]);

mediaSource.addEventListener('sourceopen', () => {
  // addSourceBuffer / appendBuffer all run off the main thread here
});
```

```javascript
// main thread
const worker = new Worker('mse-worker.js');
worker.onmessage = (e) => {
  video.srcObject = e.data.handle;
};
```

Chrome originally scheduled this for Chrome 105 but reverted it after compatibility issues with existing media sites; it shipped behind `--enable-blink-features=MediaSourceInWorkers,MediaSourceInWorkersUsingHandle` through Chrome 105.0.5180.0–108.0.5333.0, then became enabled by default starting in **Chrome 108.0.5334.0**. [Source: Chrome blink-dev "Intent to Ship: MSE in Workers"](https://groups.google.com/a/chromium.org/g/blink-dev/c/FRY3F1v6Two) · [MDN Media Source Extensions API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API)

### 3.2 ManagedMediaSource

WebKit shipped a memory-pressure-aware MSE variant, `ManagedMediaSource`, first on iPad and Mac in Safari 17.0, then on **iPhone in Safari 17.1** (released 2023-10-26). Unlike a plain `MediaSource`, a `ManagedMediaSource` participates in the OS's memory-pressure signaling: it fires `startstreaming`/`endstreaming` events so the page can start or pause fetching, and iOS itself can request buffer eviction under pressure rather than leaving eviction policy entirely to the page. This mattered concretely: before `ManagedMediaSource`, MSE was effectively unusable on iOS Safari because Apple did not expose plain `MediaSource` there at all, which meant hls.js and Shaka Player — both MSE-dependent — could not run on iOS Safari. `ManagedMediaSource` is what made that possible for the first time. [Source: WebKit Features in Safari 17.1](https://webkit.org/blog/14735/webkit-features-in-safari-17-1/)

The feature has since been proposed to the W3C Media Working Group as a cross-browser addition, tracked as `ManagedMediaSource` in the `w3c/media-source` issue tracker, and is one of the two headline additions in the MSE v2 Working Draft referenced above.

---

## 4. The EME Handshake

Encrypted Media Extensions (EME) is a separate W3C specification that layers key-acquisition and decryption on top of MSE (or, in principle, any `HTMLMediaElement` source); it reached Recommendation status on 2017-09-18 as a controversial but ultimately adopted addition to the open web platform. [Source: W3C Encrypted Media Extensions](https://www.w3.org/TR/encrypted-media/)

### 4.1 Capability Negotiation

Before a player commits to a key system, it asks the browser whether that key system can satisfy a given configuration:

```javascript
navigator.requestMediaKeySystemAccess('com.widevine.alpha', [{
  initDataTypes: ['cenc'],
  videoCapabilities: [{
    contentType: 'video/mp4; codecs="avc1.640028"',
    robustness: 'SW_SECURE_DECODE'
  }],
  audioCapabilities: [{
    contentType: 'audio/mp4; codecs="mp4a.40.2"'
  }]
}]).then(mediaKeySystemAccess => {
  return mediaKeySystemAccess.createMediaKeys();
}).then(mediaKeys => {
  return video.setMediaKeys(mediaKeys);
});
```

`requestMediaKeySystemAccess()` resolves only if the browser has (or can obtain) a Content Decryption Module for the requested key system string, and if at least one candidate `MediaKeySystemConfiguration` — codec plus `robustness` string — is satisfiable. Robustness strings are what let a player ask specifically for hardware-backed decryption (`HW_SECURE_DECODE`) versus accept software fallback (`SW_SECURE_DECODE`), which is the negotiation point where Linux's L3-only Widevine CDM (§6.2) forces every request down the software path. [Source: EME Recommendation §5 Session Types and Key Systems, §9 MediaKeySystemConfiguration](https://www.w3.org/TR/encrypted-media/#mediakeysystemconfiguration-dictionary)

### 4.2 Sessions, Licenses, and Key Status

Once `MediaKeys` is attached via `setMediaKeys()`, the browser fires an `encrypted` event on the media element the moment it parses a `pssh` box (§5.2) out of an appended segment, carrying `initDataType` and the raw `initData`:

```javascript
video.addEventListener('encrypted', (event) => {
  const session = mediaKeys.createSession();
  session.addEventListener('message', (msgEvent) => {
    fetch('/license-server', { method: 'POST', body: msgEvent.message })
      .then(r => r.arrayBuffer())
      .then(license => session.update(license));
  });
  session.generateRequest(event.initDataType, event.initData);
});
```

`generateRequest()` asks the CDM to build a license request from the init data; the CDM emits it asynchronously as a `message` event, which the app forwards, unmodified as far as the app is concerned, to whatever license server the DRM operator runs. The server's response is handed back via `session.update(response)`, which loads decryption keys into the CDM. From then on, the session's `keyStatuses` — a `MediaKeyStatusMap` — reports per-key state such as `usable`, `expired`, `output-restricted`, or `internal-error`, which a player surfaces as playback stalls or quality restrictions. [Source: EME Recommendation §6 MediaKeySession, §11 Key Status](https://www.w3.org/TR/encrypted-media/#mediakeysession-interface)

### 4.3 Clear Key: The Reference CDM

Every conformant EME implementation is required to support one key system that needs no proprietary CDM at all: **Clear Key** (`org.w3.clearkey`), where the "license" is simply the raw AES key encoded as JSON, sent back through `session.update()` with no real DRM server involved. Clear Key exists purely for interoperability testing and specification conformance — content protected only by Clear Key offers no actual content protection, since the key travels in the open — but it is what makes it possible to write and run EME conformance tests, and what browser engines use internally to validate their EME plumbing before a real CDM (Widevine, PlayReady, FairPlay) is wired in. [Source: EME Recommendation §9.1 Clear Key](https://www.w3.org/TR/encrypted-media/#clear-key)

---

## 5. Common Encryption (CENC)

### 5.1 The Four Schemes, Two in Practice

Common Encryption — standardized as **ISO/IEC 23001-7** (3rd edition, 2023), "Information technology — MPEG systems technologies — Part 7: Common encryption in ISO base media file format files" — defines the encryption layer that lets one encrypted file be decrypted by more than one DRM system without re-encoding. The standard defines four protection schemes, but only two see real deployment:

| Scheme | Cipher / mode | Subsample pattern | Deployed by |
|---|---|---|---|
| `cenc` | AES-128, CTR mode | Full-sample or full-subsample | Widevine (legacy), PlayReady |
| `cbc1` | AES-128, CBC mode | Full-subsample | Rare, largely superseded |
| `cens` | AES-128, CTR mode | Pattern (1-of-10 over NAL subsamples) | Rare |
| `cbcs` | AES-128, CBC mode | Pattern (1-of-10 over NAL subsamples) | FairPlay, modern Widevine, PlayReady |

`cenc` (§4.2a of the standard) full-sample-encrypts with AES-CTR, which is cheap to seek into at arbitrary byte offsets — a property that matters for byte-range HTTP delivery. `cbcs` (§4.2d) instead encrypts only 1 block in every 10 (the "1-of-10" pattern) of each NAL subsample with AES-CBC, trading some seek granularity for compatibility with Apple's FairPlay Streaming, which mandated CBCS from the start; modern multi-DRM packaging has largely converged on `cbcs` as the single scheme that satisfies Widevine, PlayReady, and FairPlay simultaneously, avoiding the need to package two separate encrypted copies of the same content. [Source: ISO/IEC 23001-7:2023 standard record](https://www.iso.org/standard/68042.html) · [W3C "cenc" Initialization Data Format Note](https://www.w3.org/TR/2016/NOTE-eme-initdata-cenc-20160915/) · [W3C ISO Common Encryption Protection Scheme for ISOBMFF Stream Format](https://www.w3.org/TR/eme-stream-mp4/)

### 5.2 pssh, tenc, senc

CENC's on-disk footprint inside a fragmented MP4/CMAF file is a small set of boxes layered on top of the ordinary ISO/IEC 14496-12 (MP4) box structure the rest of the stack already produces (see [Chapter 60b §9](ch60b-video-streaming-protocols.md) on CMAF packaging):

- **`pssh`** (Protection System Specific Header) — one box per DRM system, keyed by a `SystemID` UUID (Widevine, PlayReady, and FairPlay each register their own), carrying opaque, DRM-specific data such as key IDs and license-acquisition hints. A single CMAF file can carry multiple `pssh` boxes side by side — the mechanism that makes one encrypted file servable to three different DRM ecosystems.
- **`tenc`** (Track Encryption Box) — sits in the init segment's `moov`, declaring the default key ID and IV size for the track.
- **`senc`** (Sample Encryption Box) — sits per media segment, carrying the actual per-sample IVs and, for pattern schemes, the subsample byte-range layout the decryptor needs to know which bytes are encrypted versus in the clear.

[Source: W3C ISO Common Encryption Protection Scheme for ISOBMFF Stream Format](https://www.w3.org/TR/eme-stream-mp4/)

### 5.3 Multi-DRM Packaging

The practical consequence of §5.1–5.2 is that a packager needs to encode media exactly once into CMAF segments, encrypt with `cbcs`, and attach multiple `pssh` boxes — one per DRM system the operator wants to support — to produce a single set of segments servable to Widevine-only Chrome/Firefox/Android clients, FairPlay-only Safari/tvOS clients, and PlayReady-only Edge/Xbox clients alike, all fetching the identical encrypted bytes over the identical DASH or HLS manifest, differing only in which `pssh` box their local CDM consumes and which license server URL the player is configured to call for that key system. This is the point where §1–§4 of this chapter meet: MSE delivers the CMAF segments into the browser, the `encrypted` event fires off the `pssh` box MSE just appended, and EME (§4) takes over from there.

---

## 6. CDM Architecture and Linux Realities

### 6.1 Widevine Security Levels L1/L2/L3

Google's Widevine — the key system string `com.widevine.alpha` — defines three security levels that determine where in the hardware/software stack decrypted, decoded video ever exists in the clear:

- **L1** — decryption and video decode both happen inside a hardware Trusted Execution Environment (TEE); decrypted pixels never touch general-purpose CPU memory in the clear. Required by major studios for 1080p/4K/HDR delivery.
- **L2** — decryption keys are handled inside the TEE, but decoded video processing happens on the host CPU/GPU outside it.
- **L3** — everything, including decryption, runs in ordinary software with no TEE involvement; keys are obfuscated in software rather than hardware-isolated.

[Source: Widevine security levels overview, bunny.net documentation](https://docs.bunny.net/stream/widevine-security-levels) · [Bitmovin: Widevine Security Levels in Depth](https://developer.bitmovin.com/playback/docs/widevine-security-levels-in-web-video-playback)

`MediaKeySystemConfiguration.videoCapabilities[].robustness` (§4.1) is the API surface a player uses to request a specific level — strings such as `SW_SECURE_CRYPTO`, `SW_SECURE_DECODE`, `HW_SECURE_CRYPTO`, `HW_SECURE_DECODE`, and `HW_SECURE_ALL` map to points along the L3→L1 spectrum, and `requestMediaKeySystemAccess()` simply fails to resolve a configuration the platform's CDM cannot satisfy.

### 6.2 Why Desktop Linux Is Capped at L3

There is no widely deployed hardware TEE path exposed to Widevine on generic Linux desktops — no equivalent of the Android TrustZone integration, ChromeOS's verified-boot-backed TEE, or Windows' hardware-DRM path that lets Widevine attest to and use a TEE. Consequently every Widevine CDM instance running in desktop Linux Chrome, Chromium, or Firefox negotiates down to **L3**, software-only. This is the direct, mechanical cause of a familiar symptom: services like Netflix or Amazon Prime Video that gate 1080p/4K/HDR behind an L1 robustness requirement fall back to 480p or 720p on Linux, or refuse playback outright, even though the exact same browser engine plays the content at full quality on Windows or ChromeOS. This is a platform/TEE-availability limitation, not a browser-implementation bug — Chromium's Widevine CDM binary is identical across desktop OSes; only the security level the host platform can attest to differs.

### 6.3 Chromium's Bundled CDM

Chromium ships Widevine as a proprietary binary component (`libwidevinecdm.so` on Linux) distributed through Chromium's component updater — the same mechanism used for other frequently updated, non-open-source-compatible pieces such as certain codecs — rather than being compiled into the open-source Chromium tree. The CDM runs inside a sandboxed CDM host process, isolated from the renderer process that hosts the page's JavaScript, following the same multi-process isolation philosophy the rest of Chromium's GPU and media pipeline uses (see [Chapter 33](../part-VIII-browser-graphics/ch33-chromium-multiprocess-gpu.md)). Because the component is closed-source, purely open-source Chromium builds (as opposed to Google Chrome or a distro build that opts into proprietary components) may ship without it, which is a separate reason "Chromium" and "Google Chrome" can differ in DRM playback support on the same Linux distribution.

### 6.4 Firefox: Widevine as a Gecko Media Plugin

Firefox uses its **Gecko Media Plugin (GMP)** system — an extension point originally built for sandboxed third-party codecs — to host Widevine. Because Widevine's shared library does not conform to the GMP ABI natively, Firefox ships an adapter layer between the two. On desktop Linux, Firefox downloads the Widevine CDM at runtime (not bundled in the initial installer) and enables it by default so DRM-gated sites work without manual configuration; **only 64-bit Linux** is supported — there is no 32-bit Linux Widevine build. As on Chromium, the CDM negotiates down to L3 on Linux for the same TEE-availability reasons described in §6.2. [Source: MozillaWiki GeckoMediaPlugins](https://wiki.mozilla.org/GeckoMediaPlugins) · [Mozilla Support: Watch DRM content on Firefox](https://support.mozilla.org/en-US/kb/enable-drm) · [Mozilla Bugzilla #1265235 — Add Linux support for Widevine](https://bugzilla.mozilla.org/show_bug.cgi?id=1265235)

### 6.5 WebKitGTK/WPE: CDMProxy and Thunder/OpenCDM

WebKit's EME implementation splits into two layers: a cross-platform layer implementing the W3C-facing types (`MediaKeys`, `MediaKeySession`) plus internal `CDM`/`CDMPrivate` abstractions, and a platform-specific layer (`CDMInstance`, `CDMInstanceSession`) that each port fills in. For the GStreamer-based ports — WebKitGTK and WPE WebKit, the engines used by Linux embedded browsers and frameworks such as Tauri (see [Chapter 193](../part-10-browser-rendering-stack/ch193-tauri-webkitgtk-desktop.md)) — that platform layer is implemented via `CDMProxy`, `CDMInstanceProxy`, and `CDMInstanceSessionProxy` classes, plus a GStreamer decryptor element that sits in the playback pipeline and defers actual key handling to an external framework: **Thunder**, using a plugin built on **OpenCDM** (Open Content Decryption Module). Thunder/OpenCDM is not a WebKit-specific invention — it is the same DRM plumbing framework deployed broadly across set-top-box firmware, which WebKitGTK/WPE reuses rather than reimplementing CDM integration from scratch. The GStreamer-specific glue — wiring `CDMProxy` to Thunder and writing the decryptor element — is a self-contained, relatively small amount of platform code (on the order of ~1,200 lines) precisely because the heavy lifting (actual license/key management) is delegated to Thunder. This path is gated behind WebKit's `--opencdm` build option; a WebKitGTK build without it has no EME support at all. [Source: Igalia — "Serious Encrypted Media Extensions on GStreamer based WebKit ports"](https://blogs.igalia.com/xrcalvar/2020/09/02/serious-encrypted-media-extensions-on-gstreamer-based-webkit-ports/)

### 6.6 Out of Scope

FairPlay Streaming (Apple-only, Safari/tvOS/iOS) and PlayReady (Microsoft-only, Windows/Xbox/Edge) are noted here for completeness but are outside this book's Linux scope, since neither ships a Linux CDM path. This chapter also deliberately does not cover DRM circumvention or key-extraction tooling of any kind — that is out of scope for a technical reference on the legitimate playback stack.

---

## 7. MSE/DASH-HLS vs. WebRTC: When to Use Which

The delivery stack this chapter covers (MSE-consumed DASH/HLS, §1–§3, full protocol detail in [Chapter 60b](ch60b-video-streaming-protocols.md)) and the WebRTC SFU-mediated stack in [Chapter 60c](ch60c-webrtc-server-infrastructure.md) are not interchangeable implementations of the same idea — they trade latency against fan-out economics in opposite directions, and the choice between them is almost never about codec support.

**HTTP adaptive streaming (MSE + DASH/HLS)** treats every segment as a cacheable HTTP object: an origin serves each segment once, and a commodity CDN absorbs however many million times it gets requested afterward, at ordinary object-storage egress pricing. Conventional HLS/DASH segment durations (2–6s) plus client-side buffering put glass-to-glass latency at roughly 6–30s. Low-Latency HLS narrows that considerably — 1–2s segments split into ~200–400ms partial segments delivered over HTTP/2 chunked transfer, with delta playlists and preload hints letting the player fetch the next partial before the segment finishes writing — typically landing around 2–4s glass-to-glass, provided the packager, origin, every CDN point-of-presence, and the player all support partial-segment fetching. [Source: AWS — Demystifying Apple Low-Latency HTTP Live Streaming](https://aws.amazon.com/blogs/media/alhls-apple-low-latency-http-live-streaming-explained/)

**WebRTC**, by contrast, has no cacheable byte stream to hand to a CDN at all: an SFU holds a stateful, encrypted peer connection per viewer, parses every RTP packet to decide what to forward, and reacts continuously to each viewer's measured bandwidth (see [Ch60c](ch60c-webrtc-server-infrastructure.md) §"SFU internals") — CPU- and memory-heavy work that scales only by adding more SFU capacity, not by caching. That buys genuinely sub-second latency, but at infrastructure cost that does not fall with scale the way CDN egress does; production deployments commonly cite the crossover point somewhere between a few hundred and a few thousand concurrent viewers, past which teams either keep paying for SFU capacity because interactivity is the product, or shift the broadcast leg to HTTP delivery. [Source: bloggeek.me — WebRTC SFU Explained](https://bloggeek.me/webrtcglossary/sfu/)

**Hybrid topologies** split the difference for the common town-hall shape: a small interactive pool (presenter, panelists, live Q&A) stays on WebRTC for sub-second back-and-forth, while everyone else watches an LL-HLS/LL-DASH egress of that same session through a CDN — the shape [Ch60c](ch60c-webrtc-server-infrastructure.md)'s LiveKit Egress and Janus RecordPlay-to-CDN patterns exist to support, and structurally the same ingest/egress split YouTube uses for its own "Go Live" feature (noted in [Ch60b](ch60b-video-streaming-protocols.md) §4.5).

The practical rule: reach for MSE/DASH-HLS whenever delivery is one-to-many and a few seconds of latency is acceptable — it is almost always the cheaper and more robust choice at scale, and pairs directly with the EME/CDM machinery in §4–§6 of this chapter for protected VOD and live content. Reach for WebRTC specifically when the latency floor itself is the product requirement — calls, interactive live production, cloud gaming, remote control — and budget for SFU infrastructure that behaves nothing like a CDN.

## 8. Building a Video App with Shaka Player

§1–§7 covered the MSE/EME/CENC/CDM machinery from the specification side. This section grounds that machinery in one concrete, working application, built entirely on Shaka Player's own public API and documented tutorials — the reference implementation already introduced in §2.1 as the one player library that speaks both DASH and HLS through a single internal pipeline with the broadest DRM coverage of the three libraries compared there. [Source: Shaka Player tutorials](https://shaka-project.github.io/shaka-player/docs/api/tutorial-basic-usage.html)

### 8.1 Project Setup and Basic Playback

Shaka's own tutorials build the compiled bundle from source and load it as a global `<script>` — a workflow aimed at contributors to the library itself. Most application code instead consumes Shaka Player as an ordinary npm dependency:

```bash
npm install shaka-player
```

The published package's `main` entry points at the same closure-compiled UMD bundle the from-source build produces (`dist/shaka-player.compiled.js`), so it has no native ESM export — it detects a CommonJS `exports` object and assigns each top-level symbol (`Player`, `polyfill`, `util`, ...) onto it. A bundler (webpack, Vite, Rollup, Parcel) resolves the bare `'shaka-player'` specifier against `node_modules` and CJS-interops that into a default import; a bare browser `<script type="module">` with no bundler cannot load it. [Source: npm registry — shaka-player `package.json`](https://registry.npmjs.org/shaka-player/latest)

```html
<!DOCTYPE html>
<html>
  <head>
    <script type="module" src="myapp.js"></script>
  </head>
  <body>
    <video id="video" width="640" controls autoplay></video>
  </body>
</html>
```

```javascript
// myapp.js — bundled with webpack/Vite/Rollup, per the mux media-elements
// shaka-video-element and dozens of other real-world consumers of the
// npm package.
import shaka from 'shaka-player';

const manifestUri =
    'https://storage.googleapis.com/shaka-demo-assets/angel-one/dash.mpd';

function initApp() {
  // shaka.polyfill.installAll() patches browser-specific gaps in the MSE/EME
  // surface described in §1 and §4 before any player code touches them.
  shaka.polyfill.installAll();
  if (shaka.Player.isBrowserSupported()) {
    initPlayer();
  } else {
    console.error('Browser not supported!');
  }
}

async function initPlayer() {
  const video = document.getElementById('video');
  const player = new shaka.Player();
  await player.attach(video);
  window.player = player; // for debugging in the console

  player.addEventListener('error', onErrorEvent);

  try {
    await player.load(manifestUri);
    console.log('The video has now been loaded!');
  } catch (e) {
    onError(e);
  }
}

function onErrorEvent(event) {
  onError(event.detail);
}

function onError(error) {
  console.error('Error code', error.code, 'object', error);
}

document.addEventListener('DOMContentLoaded', initApp);
```

`player.attach(video)` is the same attachment point EME uses internally (`video.setMediaKeys()`, §4.1) once a manifest turns out to be encrypted — the app never calls that directly; Shaka does it as part of `load()` after the capability negotiation in §4.1 succeeds. [Source: Shaka Player — Basic Usage tutorial](https://shaka-project.github.io/shaka-player/docs/api/tutorial-basic-usage.html)

### 8.2 Configuration: ABR, Buffering, and Network Retry

Shaka centralizes all tunables — ABR, buffering, DRM, manifest parsing, language selection — behind one hierarchical `player.configure()` / `player.getConfiguration()` pair. Configuration accepts either a partial object (unset fields keep their current value; fields explicitly set to `undefined` revert to the default) or a single dotted path string plus a value, for one-off changes:

```javascript
// Object form
player.configure({
  streaming: {
    bufferingGoal: 30,
    rebufferingGoal: 15,
    bufferBehind: 30,
  },
});

// Path-string form — equivalent single-field update
player.configure('streaming.bufferingGoal', 30);
```

`bufferingGoal` is the target amount of buffered content in seconds — the player keeps fetching segments until it reaches this; `rebufferingGoal` is the minimum buffered content required before playback starts or resumes after a stall, and must always be set lower than `bufferingGoal`; `bufferBehind` is how much already-played content stays buffered behind the playhead before eviction (§1.3's quota eviction, exposed as a tunable rather than left to the browser's implicit MSE quota heuristics). Not every change is immediate: networking and buffering settings apply right away, while DRM and manifest-parsing settings (§8.4) only take effect on the *next* `load()` call.

Network resilience is split across three independent retry blocks — `drm.retryParameters` (license requests), `manifest.retryParameters`, and `streaming.retryParameters` (segment requests) — all sharing the same shape:

```javascript
player.configure({
  streaming: {
    retryParameters: {
      timeout: 30000,        // ms, 0 = no timeout
      stallTimeout: 5000,
      connectionTimeout: 10000,
      maxAttempts: 2,
      baseDelay: 1000,       // ms before the first retry
      backoffFactor: 2,      // each retry multiplies the delay by this
      fuzzFactor: 0.5,       // randomize each delay by ±50%
    },
  },
});
```

With a 1000ms base delay and a backoff factor of 2, successive retries wait roughly 1s, 2s, 4s, 8s before the fuzz factor randomizes each one by ±50% — the randomization specifically exists to stop many simultaneously-failing clients from re-hammering the same origin in lockstep. `streaming.lowLatencyMode` flips several of these and the buffering goals at once for LL-DASH/LL-HLS content (§7); `player.configurationForLowLatency()` exposes the individual values it sets, for apps that want to start from that baseline and adjust further. [Source: Shaka Player — Configuration tutorial](https://shaka-project.github.io/shaka-player/docs/api/tutorial-config.html) · [Network and Buffering Configuration tutorial](https://shaka-project.github.io/shaka-player/docs/api/tutorial-network-and-buffering-config.html)

### 8.3 Error Handling

Shaka errors carry a `severity` field distinguishing **critical** errors (`shaka.util.Error.Severity.CRITICAL` — playback cannot continue) from non-fatal ones the player can often recover from on its own. Three places to catch them cover the full lifecycle:

```javascript
// 1. Errors after a successful load — via the player's own event
player.addEventListener('error', (event) => onError(event.detail));

// 2. Errors during the initial load — via the load() promise
try {
  await player.load(manifestUri);
} catch (error) {
  onError(error);
}

function onError(error) {
  if (error.severity === shaka.util.Error.Severity.CRITICAL) {
    // stop, show a fatal error UI
  } else {
    // log and let Shaka's own recovery continue
  }
}
```

`streaming.failureCallback` lets an app intercept and convert streaming-layer errors before Shaka's default handling runs, and the `NetworkingEngine`'s `retry` event can be used to stop retries outright for errors that retrying can never fix — a 404 on a VOD manifest that will never appear, for instance — by calling `event.preventDefault()` inside the listener. [Source: Shaka Player — Error Handling tutorial](https://shaka-project.github.io/shaka-player/docs/api/tutorial-errors.html)

### 8.4 Wiring Up DRM: License Servers, Authentication, and Wrapping

§4.2 showed the EME handshake with a bare `fetch()` call standing in for "whatever license server the DRM operator runs." In a real Shaka app, that plumbing is `drm.servers` plus, when needed, request/response filters:

```javascript
player.configure({
  drm: {
    servers: {
      'com.widevine.alpha': 'https://foo.bar/drm/widevine',
      'com.microsoft.playready': 'https://foo.bar/drm/playready',
    },
  },
});
```

`drm.advanced` carries per-key-system settings, most importantly the `robustness` strings from §4.1/§6.1 as actual configuration rather than abstract API surface:

```javascript
player.configure({
  drm: {
    advanced: {
      'com.widevine.alpha': {
        videoRobustness: ['HW_SECURE_ALL'],
        audioRobustness: ['HW_SECURE_ALL'],
        headers: { 'customHeader1': 'value1' },
      },
    },
  },
});
```

Widevine accepts `SW_SECURE_CRYPTO`, `SW_SECURE_DECODE`, `HW_SECURE_CRYPTO`, `HW_SECURE_DECODE`, `HW_SECURE_ALL` — the same L3→L1 spectrum from §6.1, and the reason requesting `HW_SECURE_ALL` on desktop Linux fails to resolve per §6.2. PlayReady accepts `3000`/`2000`/`150` but, per Shaka's own documentation, effectively ignores whatever is configured and defaults to `2000` regardless. FairPlay (§6.6, noted there as out of this book's Linux scope) takes an empty-string robustness value. `drm.clearKeys` configures the Clear Key reference CDM from §4.3 directly, without a license server round-trip at all — useful for exactly the conformance-testing role §4.3 describes:

```javascript
player.configure({
  drm: {
    clearKeys: {
      'deadbeefdeadbeefdeadbeefdeadbeef': '18675309186753091867530918675309',
    },
  },
});
```

**Authenticating license requests.** EME itself has no notion of application-level authentication — the CDM just POSTs opaque bytes wherever `drm.servers` points. Shaka's `NetworkingEngine` exposes request filters to inject whatever an operator's license server actually requires: a header, a cookie, or an asynchronously-fetched bearer token.

```javascript
player.getNetworkingEngine().registerRequestFilter((type, request, context) => {
  if (type === shaka.net.NetworkingEngine.RequestType.LICENSE) {
    request.headers['CWIP-Auth-Header'] = 'VGhpc0lzQVRlc3QK';
  }
});
```

Cookie-based auth needs an explicit opt-in, since cross-origin credentials are disabled by default: `request.allowCrossSiteCredentials = true;` inside the same filter. A filter that returns a `Promise` is treated as asynchronous, which is the documented pattern for fetching a short-lived auth token before the license request goes out, rather than caching a stale one; combined with a `retry` listener watching for HTTP 401 responses, an app can refresh credentials and let Shaka's retry logic re-attempt the request. [Source: Shaka Player — License Server Authentication tutorial](https://shaka-project.github.io/shaka-player/docs/api/tutorial-license-server-auth.html)

**License wrapping.** Some license servers need to exchange application metadata alongside the raw CDM-generated license request — again something EME has no channel for. The same request/response filter pair used for authentication is what makes that possible: a request filter base64-encodes the platform's raw license request into a JSON envelope carrying whatever extra fields the server expects, and a matching response filter unwraps the server's JSON response back down to the raw license bytes the CDM actually needs.

```javascript
player.getNetworkingEngine().registerRequestFilter((type, request, context) => {
  if (type === shaka.net.NetworkingEngine.RequestType.LICENSE) {
    const wrapped = {
      rawLicenseRequestBase64:
          shaka.util.Uint8ArrayUtils.toBase64(new Uint8Array(request.body)),
      favoriteColor: 'blue', // arbitrary app-specific metadata
    };
    request.body = shaka.util.StringUtils.toUTF8(JSON.stringify(wrapped));
  }
});

player.getNetworkingEngine().registerResponseFilter((type, response, context) => {
  if (type === shaka.net.NetworkingEngine.RequestType.LICENSE) {
    const wrapped = JSON.parse(shaka.util.StringUtils.fromUTF8(response.data));
    response.data =
        shaka.util.Uint8ArrayUtils.fromBase64(wrapped.rawLicenseBase64);
  }
});
```

This is the mechanism that turns §4.2's `session.update(license)` step from a direct pass-through into whatever a real operator's license-proxy contract actually requires, without the CDM or the EME spec needing to know or care. [Source: Shaka Player — License Wrapping tutorial](https://shaka-project.github.io/shaka-player/docs/api/tutorial-license-wrapping.html)

### 8.5 Widevine Service Certificates

A Widevine service certificate lets a player skip one network round-trip per session: without it, the CDM's first license request has to go fetch the certificate from the license server before it can generate a real request; preloading the certificate removes that round-trip. It can be provided directly or as a URI Shaka fetches itself, and once configured it persists across every subsequent `load()` on the same `Player` instance:

```javascript
player.configure({
  drm: {
    advanced: {
      'com.widevine.alpha': {
        serverCertificateUri: 'https://example.com/service.cert',
      },
    },
  },
});
```

This is a session-setup optimization independent of the L1/L2/L3 security-level negotiation in §6.1 — it affects how quickly the first license request goes out, not which security level the CDM ultimately gets. [Source: Shaka Player — Widevine Service Certificates tutorial](https://shaka-project.github.io/shaka-player/docs/api/tutorial-widevine-service-certs.html)

### 8.6 The UI Library

Everything in §8.1–§8.5 works against a bare `<video>` element with no on-screen controls beyond the browser default. Shaka's separate, optional UI library (`dist/shaka-player.ui.js` + `dist/controls.css`) adds an accessible, localized, skinnable control surface declaratively:

```html
<div data-shaka-player-container style="max-width:40em">
  <video autoplay data-shaka-player id="video"
         style="width:100%;height:100%"></video>
</div>
```

```javascript
document.addEventListener('shaka-ui-loaded', () => {
  const video = document.getElementById('video');
  const ui = video['ui'];
  const controls = ui.getControls();
  const player = controls.getPlayer();

  player.addEventListener('error', onPlayerErrorEvent);
  controls.addEventListener('error', onUIErrorEvent);
});
document.addEventListener('shaka-ui-load-failed', initFailed);
```

A manifest URI can be supplied declaratively too (`<video data-shaka-player src="...dash.mpd">`, with `<source>` fallbacks for multi-format manifests) instead of calling `load()` from application code. For apps that build the player/video DOM dynamically rather than declaring it in markup up front, `new shaka.ui.Overlay(player, container, video)` attaches the same UI programmatically. Optional attributes add Chromecast support (`data-shaka-player-cast-receiver-id`) and VR playback mode. [Source: Shaka Player — UI Library tutorial](https://shaka-project.github.io/shaka-player/docs/api/tutorial-ui.html)

### 8.7 MoQ: Media over QUIC (Experimental)

Shaka Player carries experimental support for a fourth delivery mode alongside the DASH/HLS-over-HTTP model this chapter otherwise covers: **Media over QUIC Transport (MoQT)**, consumed through Shaka's own **MoQ Streaming Format (MSF)** manifest parser. It is built entirely on the browser `WebTransport` API and tracks IETF drafts that are still under active development — not a finished RFC. Loading it only requires signaling the MIME type explicitly, since there is no file extension or manifest structure for Shaka to sniff:

```javascript
await player.load(manifestUri, null, 'application/msf');
```

Internally this opens a `WebTransport` connection to the given URI, performs MoQT session setup and version negotiation, subscribes to (or auto-discovers) a catalog track namespace, parses the resulting catalog JSON to find the available media tracks, and subscribes to those tracks into the same playback pipeline every other MSE-backed source in this chapter uses. Content protection works the same way as DASH/HLS — protection info in the catalog populates `drm.servers` (§8.4) automatically. Configuration lives under `manifest.msf`: `namespaces` for an explicit MoQT path (auto-discovered if omitted), `authorizationToken` for a bearer token, `fingerprintUri` for validating a self-signed relay certificate, `useFetchCatalog` to choose a one-shot catalog fetch over an ongoing subscription, and `version` to pin a specific draft (`DRAFT_14`, `DRAFT_16`) instead of auto-negotiating. The current, load-bearing limitation: **only live content is supported — a VOD MSF manifest fails to load.** [Source: Shaka Player — MoQ tutorial](https://shaka-project.github.io/shaka-player/docs/api/tutorial-moq.html)

MoQT is worth noting against §7's latency/scale framing specifically because it targets the gap between the two poles that section describes: QUIC/WebTransport's per-object subscription model aims at WebRTC-adjacent latency while staying on a pub/sub relay architecture that, unlike an SFU, is designed to fan out through cacheable relay infrastructure rather than one stateful connection per viewer. Whether that holds up in production is still an open question — the relevant IETF working group's specifications remain drafts, and Shaka's own implementation is explicitly experimental.

---

## 9. Integrations

- **Ch33 (Chromium's Multi-Process GPU Architecture)**: the Widevine CDM host process (§6.3) follows the same sandboxed multi-process isolation model documented there for the GPU process.
- **Ch36 (The Chromium Compositor — CC and Viz)**: decoded and decrypted frames handed back by the CDM/decoder pipeline enter the compositor exactly like any other video frame once past the protected-content boundary.
- **Ch57 (FFmpeg Architecture and Programming)**: FFmpeg's own decryption filters and `crypto`/`decrypt` protocol handlers solve a related but distinct problem outside the browser sandbox — no shared code path with browser EME/CDM.
- **Ch58 (GStreamer: Pipeline-Based Multimedia)**: the `decryptor` element pattern and GStreamer's Clear Key plugin are the same conceptual shape as §4.3's Clear Key CDM, and are literally what WebKitGTK's GStreamer ports (§6.5) build their decryptor element against.
- **Ch60b (Video Streaming Protocols and Adaptive Bitrate Delivery)**: MSE (§1–3) is the browser-side consumption half of the DASH/HLS + CMAF segmented-delivery pipeline documented there in full; §5's CENC/CMAF packaging is the encryption layer over the same segments.
- **Ch60c (WebRTC Server Infrastructure on Linux)**: §7's latency-vs-scale tradeoff is the decision this chapter's SFU/TURN infrastructure exists to make possible; its LiveKit Egress and Janus RecordPlay-to-CDN patterns are the hybrid-topology mechanism §7 describes.
- **Ch146 (WebCodecs and Browser Hardware Acceleration)**: WebCodecs deliberately does not expose decrypted samples for protected content — a scope boundary that exists specifically because doing so would break the CDM isolation guarantees this chapter describes.
- **Ch193 (Tauri — Rust-Native Desktop Applications via WebKitGTK)**: any Tauri app relying on WPE/WebKitGTK for protected playback goes through the exact Thunder/OpenCDM path described in §6.5.

---

## Roadmap

### Near-term (6–12 months)
- **MSE v2 remains in active Working Draft revision**, with the working group continuing to publish successive drafts and explicitly cautioning implementors that behavior may still change before Candidate Recommendation status; `changeType()`, MSE-in-Workers, and `ManagedMediaSource` (§3) are the substantive additions since the 2016 Recommendation, none of which has reached Candidate Recommendation status yet. [Source: Media Source Extensions™ Level 2 Working Draft](https://www.w3.org/TR/media-source-2/)
- **`ManagedMediaSource` cross-browser adoption is the near-term watch item.** As of this writing it is still WebKit/Safari-only and explicitly "not Baseline" per MDN's compatibility data; whether Chromium and Firefox implement it now that it is normatively specified in the MSE v2 draft determines whether hls.js/Shaka Player gain a single cross-browser memory-pressure API or continue treating it as an iOS-only special case. [Source: MDN ManagedMediaSource](https://developer.mozilla.org/en-US/docs/Web/API/ManagedMediaSource)
- **EME's CENC-version fragmentation gap remains an open, acknowledged spec limitation.** The EME spec still normatively references only the original edition of the ISO/IEC 23001-7 CENC standard, while CENC itself has since changed subsample-encryption rules (video slice headers must stay unencrypted from CENCv3 on) and added content-sensitive encryption schemes and multi-key/IV support (§5.1) that EME has no negotiation mechanism for; the working group has repeatedly discussed the gap without shipping a resolution. [Source: W3C Encrypted Media Extensions issue tracker](https://github.com/w3c/encrypted-media/issues/563)

### Medium-term (1–3 years)
- **A standardized Screen Capture Protection signal may join `MediaKeySystemConfiguration`.** An open EME proposal would let a page request screen-capture/recording protection through the EME configuration itself rather than relying on each DRM vendor's CDM to implement it inconsistently — collapsing a currently vendor-specific behavior into one negotiable capability alongside the `robustness` strings of §4.1. Note: needs verification — the proposal has no browser-vendor commitment attached as of this writing, so adoption timeline is uncertain. [Source: W3C Encrypted Media Extensions issue tracker](https://github.com/w3c/encrypted-media/issues/564)
- **CMAF packaging continues consolidating on `cbcs`** (§5.1) as the single scheme serving Widevine/PlayReady/FairPlay simultaneously; if the CENC-version negotiation gap above is resolved, expect packagers to begin adopting the 2023 edition's newer subsample and multi-key features without needing to maintain parallel CENCv1-compatible streams for legacy hardware CDMs.
- **MSE-in-Workers usage broadens beyond ABR players** as `MediaSource.handle` (§3.1) moves past its Chromium-only origin; player libraries (Shaka, hls.js, dash.js — §2.1) are the natural next adopters now that the API is Baseline in Chromium, since the underlying motivation — keeping segment-append work off the main thread — applies equally to every MSE-based player.

### Long-term
- **The desktop-Linux L3 ceiling (§6.2) is structural, not a roadmap item with a fix date.** It requires a widely deployed, attestable hardware TEE on generic Linux desktops — something no vendor roadmap currently commits to — so absent a platform-level change (e.g., a consumer-grade confidential-computing TEE becoming ubiquitous on Linux desktop hardware), 4K/HDR premium-content gating on Linux is likely to persist indefinitely as a platform limitation rather than something any browser update resolves. Note: needs verification if hardware/OS vendors announce a concrete TEE attestation path for Linux desktops.
- **EME and WebCodecs' protected-content boundary (Ch146) is likely to stay a hard line**, not a temporary gap: exposing decrypted samples to WebCodecs would defeat the CDM isolation guarantee that is the entire reason studios accept L3 software playback at all, so relaxing it would require a fundamentally different trust model, not just a spec extension.
