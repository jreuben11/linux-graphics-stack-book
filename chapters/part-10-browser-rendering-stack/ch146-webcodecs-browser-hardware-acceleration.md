# Chapter 146: WebCodecs and Browser Hardware Acceleration

**Part**: X — The Browser Rendering Stack

**Primary audiences**: Browser and web platform engineers building real-time media pipelines — live streaming, video conferencing, custom codec transcoders, and frame-level WebGPU compositing. Application developers who need to understand the performance boundaries between the web platform and the Linux hardware stack beneath it: VA-API, DMA-BUF, SharedImage, and the GPU process sandbox. Systems engineers debugging hardware video acceleration on Linux Chrome who need to trace the path from a JavaScript `VideoDecoder.decode()` call down to a libva `vaEndPicture()` invocation in a kernel DRM driver.

---

## Table of Contents

1. [WebCodecs Overview: The Web Platform's First Low-Latency Codec API](#1-webcodecs-overview-the-web-platforms-first-low-latency-codec-api)
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

### Why WebCodecs Instead of MSE or WebRTC?

Media Source Extensions feeds segments to the browser's internal decoder, but frames are never exposed to JavaScript — they go directly from the internal decoder to the compositor. WebRTC's `RTCPeerConnection` similarly hides the codec layer, exposing only rendered frames via `<video>` elements. Neither gives frame-level access or allows the application to choose per-frame processing.

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

## 12. Integrations

### Chapter 26: VA-API Architecture

The WebCodecs hardware acceleration path on Linux desktop is built entirely on VA-API. [Chapter 26](../part-07-application-apis-middleware/) covers the full VA-API architecture: `vaInitialize()`, `vaCreateConfig()`, `vaCreateContext()`, surface management, profile enumeration, and the libva driver model. This chapter focuses on the Chromium-specific wrapping in `VaapiWrapper` and `VaapiVideoDecoder` — the layer between Chrome's media stack and the libva API described in Chapter 26. For `vaExportSurfaceHandle()` internals, the VA-API 1.1.0 specification, and the iHD vs i965 driver distinction at the libva level, see Chapter 26.

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
