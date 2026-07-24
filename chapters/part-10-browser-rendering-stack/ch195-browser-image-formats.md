# Chapter 195: Browser Image Formats — Decode Pipelines, Compression Mechanisms, and HDR

> **Part**: Part X — The Browser Rendering Stack
> **Audience**: Browser and web platform engineers who need to understand how image bytes from the network become GPU textures; web application developers choosing between JPEG, WebP, AVIF, and JPEG XL; systems engineers debugging image decode performance or HDR colour fidelity in the browser
> **Status**: First draft — 2026-06-24

## Table of Contents

- [Overview](#overview)
- [1. The `<img>` Decode Pipeline in Chrome](#1-the-img-decode-pipeline-in-chrome)
  - [1.6 What is SkBitmap?](#16-what-is-skbitmap)
  - [1.7 What is SkCodec?](#17-what-is-skcodec)
  - [1.8 What is gpu::SharedImage?](#18-what-is-gpusharedimage)
- [2. Format Negotiation: Accept Headers and `<picture>`](#2-format-negotiation-accept-headers-and-picture)
- [3. JPEG: DCT, Quantisation, and libjpeg-turbo](#3-jpeg-dct-quantisation-and-libjpeg-turbo)
- [4. PNG: DEFLATE Compression and libpng](#4-png-deflate-compression-and-libpng)
- [5. WebP: VP8, VP8L, and Extended Features](#5-webp-vp8-vp8l-and-extended-features)
- [6. AVIF: AV1 Intra-Frame Coding and HDR](#6-avif-av1-intra-frame-coding-and-hdr)
- [7. JPEG XL: Next-Generation Compression and Lossless JPEG Re-encoding](#7-jpeg-xl-next-generation-compression-and-lossless-jpeg-re-encoding)
- [8. GIF and Animated Images](#8-gif-and-animated-images)
- [9. SVG: XML-Based Vector Graphics](#9-svg-xml-based-vector-graphics)
- [10. ICC Profiles and Colour Management Through the Browser Pipeline](#10-icc-profiles-and-colour-management-through-the-browser-pipeline)
- [11. GPU Upload Path: CPU Decode to Compositor Texture](#11-gpu-upload-path-cpu-decode-to-compositor-texture)
- [12. The ImageDecoder API (WebCodecs)](#12-the-imagedecoder-api-webcodecs)
- [13. Format Comparison](#13-format-comparison)
- [14. Roadmap](#14-roadmap)
- [15. Integrations](#15-integrations)

---

## Overview

Every `<img>` tag, CSS `background-image`, `<picture>` element, and `createImageBitmap()` call initiates an image decode pipeline that spans network, CPU, and GPU. The format of the image determines which codec library decodes it, what colour metadata is preserved, and whether HDR content survives the trip to the compositor.

The Linux browser's image pipeline is entirely CPU-side for still images — no VA-API acceleration for JPEG, AVIF, or PNG (unlike video; see Ch147). The CPU decode produces a raster in a canonical format (`SkBitmap`, typically RGBA8 or RGBA16F for HDR), which is then uploaded to the GPU as a texture and made available to the compositor via `gpu::SharedImage`. The format choice affects decode time, file size, colour fidelity, and HDR capability, but is invisible to the GPU rasterisation pipeline downstream.

---

## 1. The `<img>` Decode Pipeline in Chrome

Understanding where image decoding happens and which components are involved is prerequisite to reasoning about decode performance and colour management.

### 1.1 Initiation: Resource Loader

When Blink encounters `<img src="photo.avif">`, the HTML parser creates an `HTMLImageElement` and calls `ImageLoader::updateFromElement()`. The resource fetcher (`blink::ResourceFetcher`) issues a network request through Chrome's URL loader (`content::URLLoaderFactory`), which routes through the browser process's network service (`network::NetworkService` → `network::URLLoader` → Cronet / BoringSSL).

The response body arrives as a sequence of data chunks delivered to `blink::ImageResource`. As chunks arrive, Blink begins **incremental decode**: the codec is selected, the image header is parsed (width, height, colour space, ICC profile), and — for progressive formats — partial renders may be produced before the full file is received.

### 1.2 Codec Selection

Blink selects a codec by examining the HTTP `Content-Type` response header first; if absent or unreliable, it sniffs the first bytes of the file:

```
Content-Type: image/avif       → SkCodec wraps libavif
Content-Type: image/webp       → SkCodec wraps libwebp
Content-Type: image/jpeg       → SkCodec wraps libjpeg-turbo
Content-Type: image/png        → SkCodec wraps libpng
Content-Type: image/jxl        → SkCodec wraps libjxl
Content-Type: image/gif        → Skia GIF decoder
Content-Type: image/svg+xml    → Blink SVG parser (not a pixel codec)
```

Chrome's codec integration point is `SkCodec` (the Skia image codec abstraction, `src/codec/SkCodec.cpp`). All raster image formats arrive as `SkCodec` subclasses:
- `SkJpegCodec` — wraps libjpeg-turbo
- `SkPngCodec` — wraps libpng
- `SkWebpCodec` — wraps libwebp
- `SkAvifCodec` — wraps libavif (which uses dav1d or libaom for AV1 decode)
- `SkJxlCodec` — wraps libjxl

[Source: https://skia.googlesource.com/skia/+/refs/heads/main/src/codec/]

### 1.3 Decode and SkBitmap Production

The `SkCodec::getPixels()` call decodes the compressed image data into an `SkBitmap` with a specified `SkColorType` and `SkAlphaType`. For standard 8-bit images the target is `kN32_SkColorType` (BGRA8888 on little-endian); for HDR images (10-bit AVIF, JPEG XL) the target is `kRGBA_F16_SkColorType` (half-float RGBA).

The decode happens on a worker thread pool in the renderer process (`blink::ImageDecodingStore` / `blink::DeferredImageDecoder`). Blink caches partially-decoded frames and decoded pixel data to avoid redundant decode on repaint.

### 1.4 Colour Space Conversion

After decode, if the image embeds an ICC profile or declares a colour space (AVIF `colr` box, PNG `iCCP` / `sRGB` / `cICP` chunks, JPEG APP2 ICC), Blink performs colour space conversion to the compositor's working colour space:

- **For SDR images on standard displays**: conversion to sRGB is performed by Skia's colour management layer (`skcms`, a compact ICC profile library embedded in Skia) on the CPU during decode.
- **For HDR images (wide gamut or PQ/HLG transfer function)**: the `SkColorSpace` is propagated through the pipeline and converted by the compositor's GPU colour management path (via `wp_color_management_v1` or the Skia Graphite colour space transform pass).

[Source: https://skia.googlesource.com/skia/+/refs/heads/main/modules/skcms/]

### 1.5 GPU Upload: SkBitmap → SharedImage

After decode, the `SkBitmap` is uploaded to the GPU as a `gpu::SharedImage`. In Chrome:

```
SkBitmap (CPU RGBA8 / F16 pixel data)
  │
  SkImage::MakeFromBitmap() → SkImage backed by SkBitmap
  │
  gpu::ClientSharedImageInterface::CreateSharedImage()
  │  Mojo IPC → GPU process
  │
  GPU process: SharedImageFactory::CreateSharedImage()
  │  vkCreateImage (VK_FORMAT_R8G8B8A8_UNORM or VK_FORMAT_R16G16B16A16_SFLOAT)
  │  vkCmdCopyBufferToImage (staging buffer upload)
  │  vkQueueSubmit → fence
  │
  gpu::Mailbox (cross-process texture token)
  │
  Viz: TextureDrawQuad references Mailbox
  → viz::SkiaRenderer samples texture at composite time
```

The upload uses a staging buffer (`VkBuffer` with `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`) into which the CPU pixel data is written, then a `vkCmdCopyBufferToImage` transfers it to a device-local `VkImage`. This is the standard Vulkan texture upload path.

### 1.6 What is SkBitmap?

`SkBitmap` is Skia's primary CPU-side pixel buffer type, defined in `include/core/SkBitmap.h`. It represents a rectangular array of pixels in a specified colour type (`SkColorType`) and alpha type (`SkAlphaType`), backed by a reference-counted `SkPixelRef` that owns the underlying memory allocation. In the Chrome image decode pipeline, `SkBitmap` is the canonical intermediate representation between the compressed byte stream arriving from the network and the GPU texture consumed by the compositor.

When a codec decodes an image, it writes pixel data into an `SkBitmap` with a target colour type determined by the source format: standard 8-bit images use `kN32_SkColorType` (BGRA8888 on little-endian platforms, matching the native GPU upload format), while HDR images from AVIF or JPEG XL use `kRGBA_F16_SkColorType` (16-bit half-float per channel, preserving the extended dynamic range). The `SkBitmap` carries an associated `SkColorSpace` describing the colour gamut and transfer function of the decoded pixel data; this metadata propagates through the pipeline and informs subsequent colour management decisions.

Once decoded, the `SkBitmap` exists in renderer-process memory. It is transferred to the GPU by copying its pixel data into a host-visible staging buffer and issuing a Vulkan transfer command. At that point the `SkBitmap` is typically released; the decoded pixels live only in GPU memory, referenced by a `gpu::SharedImage` mailbox that the compositor resolves at composite time. Understanding `SkBitmap` is therefore prerequisite to following both the decode path (§1.3) and the GPU upload path (§1.5 and §11).

### 1.7 What is SkCodec?

`SkCodec` is Skia's abstract base class for image decoders, defined in `include/codec/SkCodec.h`. It provides a unified decode interface over format-specific libraries — libjpeg-turbo for JPEG, libpng for PNG, libwebp for WebP, libavif for AVIF, and libjxl for JPEG XL — so that Chrome's image infrastructure interacts with a single API regardless of the underlying codec library in use. The format is identified either from the HTTP `Content-Type` response header or by sniffing the first bytes of the compressed file, and the matching `SkCodec` subclass is instantiated and returned to Blink's image loading machinery.

The core API has two primary decode modes. `SkCodec::getPixels()` performs a complete synchronous decode into a caller-supplied buffer described by an `SkImageInfo` (dimensions, colour type, colour space). `SkCodec::startIncrementalDecode()` combined with repeated calls to `SkCodec::incrementalDecode()` supports streaming decode, allowing partial images to be rendered as compressed data arrives over the network — this is the path used for progressive JPEG and interlaced PNG. Each `SkCodec` subclass is also responsible for extracting embedded colour metadata: it reads ICC profiles from JPEG APP2 segments, AVIF `colr` boxes, PNG `iCCP` and `cICP` chunks, and exposes them as an `SkColorSpace` associated with the decoded output.

All `SkCodec` subclasses live under `src/codec/` in the Skia source tree. Chrome bundles Skia under `third_party/skia/`. [Source: https://skia.googlesource.com/skia/+/refs/heads/main/include/codec/SkCodec.h]

### 1.8 What is gpu::SharedImage?

`gpu::SharedImage` is Chrome's mechanism for sharing GPU texture resources across the renderer and GPU process boundary. In Chrome's multi-process architecture, image decode runs in the renderer process but GPU memory is owned by the GPU process; `SharedImage` provides the cross-process handle that ties them together. A `SharedImage` is identified by a `gpu::Mailbox` — a 128-bit opaque token passed over Mojo IPC — that lets different processes reference the same underlying `VkImage` (or `GLTexture` when the GL backend is active) without duplicating the pixel data.

When a decoded `SkBitmap` is ready for composite, the renderer calls `gpu::ClientSharedImageInterface::CreateSharedImage()` over Mojo to the GPU process. The GPU process allocates a `VkImage` in the appropriate format (`VK_FORMAT_R8G8B8A8_UNORM` for SDR, `VK_FORMAT_R16G16B16A16_SFLOAT` for HDR), performs the staging-buffer upload via `vkCmdCopyBufferToImage`, and returns a `gpu::Mailbox` to the renderer. The renderer embeds this mailbox in a `TextureDrawQuad` sent to the Viz compositor. The compositor resolves the mailbox back to the `VkImage` and samples from it during the compositing pass.

`SharedImage` is also the token used for video frames (Ch147), WebGL textures, and canvas surfaces — it is the universal GPU resource handle in Chrome's multi-process graphics model. The client interface is in `gpu/command_buffer/client/shared_image_interface.h`; the GPU-process factory is in `gpu/ipc/service/shared_image_factory.h`.

---

## 2. Format Negotiation: Accept Headers and `<picture>`

### 2.1 HTTP Accept Header

Chrome sends a prioritised `Accept` header that advertisees which formats it can decode, allowing the server to return the most efficient format available:

```
Accept: image/avif,image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
```

A server capable of serving AVIF will respond with `Content-Type: image/avif`; a server that only has JPEG falls back to `image/jpeg`. This passive negotiation requires no page author involvement and is the primary mechanism by which CDNs serve modern formats transparently.

### 2.2 `<picture>` and `srcset`

For explicit control, the `<picture>` element lets authors list format alternatives in preference order:

```html
<picture>
  <!-- JPEG XL: best compression, HDR, not yet universally supported -->
  <source srcset="photo.jxl" type="image/jxl">
  <!-- AVIF: AV1-based, HDR capable, wide browser support -->
  <source srcset="photo.avif" type="image/avif">
  <!-- WebP: near-universal support, no HDR -->
  <source srcset="photo.webp" type="image/webp">
  <!-- JPEG: universal fallback -->
  <img src="photo.jpg" alt="Photo">
</picture>
```

The browser selects the first `<source>` whose `type` it supports. Blink's `<picture>` implementation in `third_party/blink/renderer/core/html/media/html_picture_element.cc` iterates the `<source>` elements, checks each `type` against the registered codec list, and issues the network request for the first match.

`srcset` with `w` descriptors additionally enables responsive images — selecting a different resolution based on the viewport and device pixel ratio — orthogonal to format selection.

---

## 3. JPEG: DCT, Quantisation, and libjpeg-turbo

### 3.1 Compression Mechanism

JPEG (ISO/IEC 10918-1) compresses images using the **Discrete Cosine Transform** (DCT). The compression pipeline:

1. **Colour space conversion**: RGB → YCbCr (luminance/chrominance separation). Human vision is more sensitive to luminance than colour, so chrominance channels are optionally subsampled (4:2:0 halves chrominance resolution horizontally and vertically).
2. **Block DCT**: The image is divided into 8×8 pixel blocks. Each block is transformed by the 2D DCT, producing 64 frequency coefficients. Low-frequency coefficients (top-left of the 8×8 block) carry most of the image energy.
3. **Quantisation**: Each DCT coefficient is divided by a quality-dependent quantisation matrix value and rounded to an integer. Higher quantisation → more rounding → more loss → smaller file. The quantisation table is the primary quality control.
4. **Entropy coding**: Quantised coefficients are Huffman-coded (baseline JPEG) or arithmetic-coded (JPEG arithmetic, rarely used). The DC coefficient (top-left, average luminance of the block) is delta-coded from the previous block's DC; AC coefficients use run-length encoding of zeros.

**Progressive JPEG** encodes the image in multiple scans: first a low-quality scan covering the full image, then refinement scans adding higher-frequency detail. This allows browsers to show a rough image immediately and refine as data arrives — the default for most web images.

### 3.2 libjpeg-turbo

Chrome uses **libjpeg-turbo** (`third_party/libjpeg_turbo/`) — a fork of libjpeg with SIMD-optimised IDCT, colour conversion, and Huffman decoding using SSE2/AVX2 (x86) and NEON (ARM):

```c
/* libjpeg-turbo API (simplified) */
tjhandle tj = tjInitDecompress();
tjDecompressHeader3(tj, jpeg_buf, jpeg_size, &width, &height, &subsamp, &colorspace);
tjDecompress2(tj, jpeg_buf, jpeg_size, rgb_buf,
              width, 0 /* pitch */, height, TJPF_RGBA, TJFLAG_FASTDCT);
tjDestroy(tj);
```

On x86 with AVX2, libjpeg-turbo's IDCT is approximately 3–5× faster than the reference libjpeg implementation. Chrome's decode throughput for a 12 MP JPEG is typically 80–150 ms on a single CPU core, and 20–40 ms with parallelism.

### 3.3 Limitations

- No transparency (alpha channel) — use PNG or WebP for transparent images
- No HDR — 8 bits per channel; JPEG 2000 supported HDR but is not used in browsers
- Blocking artefacts at low quality due to independent block quantisation
- No animation support
- No lossless mode (JPEG lossless exists but is incompatible with the lossy standard)

---

## 4. PNG: DEFLATE Compression and libpng

### 4.1 Compression Mechanism

PNG (ISO/IEC 15948) is a **lossless** format. Its compression pipeline:

1. **Filter prediction**: Each row of pixels is transformed by a prediction filter (None, Sub, Up, Average, Paeth) chosen to minimise the variance of the residuals. The filter choice is made per-row and stored in the data stream.
2. **DEFLATE compression**: The filtered pixel data is compressed using DEFLATE (LZ77 + Huffman coding, the same algorithm as zlib and gzip). PNG files are essentially a series of IDAT chunks containing DEFLATE-compressed filtered pixel rows.

PNG supports:
- **Bit depths**: 1, 2, 4, 8, 16 bits per channel
- **Colour types**: greyscale, RGB, RGBA (with alpha), indexed-colour
- **16-bit per channel**: enables HDR storage, but browser support for 16-bit PNG display is limited
- **Interlaced (Adam7)**: like progressive JPEG but for PNG; rarely used

### 4.2 libpng

Chrome uses **libpng** (`third_party/libpng/`) with SIMD-optimised filter application and DEFLATE via zlib:

```c
/* libpng decode (simplified) */
png_structp png = png_create_read_struct(PNG_LIBPNG_VER_STRING, NULL, NULL, NULL);
png_infop info = png_create_info_struct(png);
png_set_read_fn(png, &io_state, my_read_fn);
png_read_info(png, info);

int width  = png_get_image_width(png, info);
int height = png_get_image_height(png, info);
/* Request RGBA8 output regardless of source format */
png_set_expand(png);
png_set_strip_16(png);
png_set_add_alpha(png, 0xFF, PNG_FILLER_AFTER);
png_read_update_info(png, info);

for (int y = 0; y < height; y++)
    png_read_row(png, row_pointers[y], NULL);
```

PNG decode is typically slower than JPEG for equivalent image sizes because DEFLATE decompression (while fast) must process the entire compressed stream serially. A 12 MP PNG (uncompressed ~36 MB) takes 150–400 ms to decode depending on compression level and CPU.

### 4.3 Use Cases

PNG is the correct choice when lossless quality is required (icons, screenshots, UI assets, medical imaging). For photographic content, PNG files are 5–10× larger than JPEG at comparable visual quality, making it unsuitable for web photos. AVIF lossless or WebP lossless are better alternatives to PNG for photographic content.

---

## 5. WebP: VP8, VP8L, and Extended Features

### 5.1 Compression Mechanisms

**WebP** uses two distinct compression algorithms depending on whether lossy or lossless mode is selected:

**Lossy WebP (VP8)**: Derived from the VP8 video codec's intra-frame coding:
1. The image is divided into 16×16 macroblocks.
2. Each macroblock is predicted from neighbouring decoded macroblocks (intra prediction modes: horizontal, vertical, DC, TrueMotion).
3. The prediction residual is transformed by the Walsh–Hadamard transform (4×4) and quantised.
4. Entropy coding uses a modified VP8 boolean arithmetic coder.

Lossy WebP produces 25–35% smaller files than JPEG at equivalent visual quality (SSIM), particularly for images with large flat regions and smooth gradients.

**Lossless WebP (VP8L)**:
1. Colour space transform (green channel as predictor for R and B channels).
2. Spatial predictor: 2D predictor chosen per 4×4 block from 14 predictor modes.
3. Entropy coding: 2D Huffman codes with locality-sensitive prefix coding.

Lossless WebP is typically 20–30% smaller than PNG for photographic content; for non-photographic content (flat colour, UI graphics) PNG may be smaller.

**Extended WebP** (RIFF container wrapping VP8 or VP8L): adds:
- **Alpha channel** (separate VP8L-compressed plane)
- **Animation** (`ANIM`/`ANMF` chunks; multiple frames with inter-frame delta encoding)
- **ICC profile** (`ICCP` chunk)
- **Metadata** (EXIF, XMP via `EXIF`/`XMP ` chunks)

### 5.2 libwebp

Chrome uses **libwebp** (`third_party/libwebp/`) — Google's reference implementation:

```c
/* libwebp decode */
WebPDecoderConfig config;
WebPInitDecoderConfig(&config);
config.output.colorspace = MODE_RGBA;
config.options.use_threads = 1;  /* parallel decode of horizontal strips */
WebPDecode(webp_data, webp_size, &config);
/* config.output.u.RGBA.rgba contains decoded pixels */
```

libwebp's VP8 decoder uses SSE2/AVX2/NEON SIMD for the IDCT and loop filter steps. Decode performance is similar to JPEG; animation frames are decoded incrementally.

### 5.3 Limitations

WebP has no HDR support — it is limited to 8 bits per channel. For HDR photographic content, AVIF or JPEG XL are the correct choices.

---

## 6. AVIF: AV1 Intra-Frame Coding and HDR

### 6.1 Compression Mechanism

**AVIF** (AV1 Image File Format, specified by the Alliance for Open Media) stores still images as single intra frames of the AV1 video codec, wrapped in an ISOBMFF container (`.avif` / `.heif`).

AV1 intra-frame coding is significantly more complex than JPEG's 8×8 DCT:

- **Block partitioning**: Recursive quad-tree partitioning from 128×128 superblocks down to 4×4 sub-blocks, allowing large flat regions to use large blocks and detail regions to use small blocks.
- **Transform types**: 16 2D transform types (DCT, ADST, identity, and their transposes) chosen per block, compared to JPEG's single 8×8 DCT.
- **Intra prediction**: 56 angular prediction modes plus 7 non-directional modes (DC, smooth, smooth-H, smooth-V, Paeth-equivalent); JPEG has none.
- **Loop filters**: deblocking filter, constrained directional enhancement filter (CDEF), and loop restoration filter applied post-decode to reduce blocking artefacts.
- **Entropy coding**: CABAC-derived multi-symbol range coder.

The result is 50–60% better compression than JPEG at equivalent visual quality, and 20–30% better than lossy WebP.

**AVIF HDR**: AVIF natively supports:
- 10-bit and 12-bit per channel
- BT.2020 colour primaries (wide gamut)
- PQ (ST 2084) and HLG transfer functions
- `clli` box: `MaxCLL` and `MaxFALL` HDR luminance metadata
- `mdcv` box: SMPTE ST 2086 mastering display colour volume

This makes AVIF the first widely-supported web image format capable of carrying HDR content with complete metadata.

### 6.2 libavif and dav1d

Chrome uses **libavif** (`third_party/libavif/`) which wraps **dav1d** (the VideoLAN Foundation's fast AV1 decoder) for decode:

```c
avifDecoder *decoder = avifDecoderCreate();
decoder->maxThreads = 4;  /* parallel tile decode */
avifDecoderSetIOMemory(decoder, avif_data, avif_size);
avifDecoderParse(decoder);  /* reads container, selects codec */
avifDecoderNextImage(decoder);  /* decodes first frame → decoder->image */

avifImage *img = decoder->image;
/* img->width, img->height, img->depth (8/10/12), img->colorPrimaries,
   img->transferCharacteristics, img->matrixCoefficients */
/* img->yuvPlanes[AVIF_CHAN_Y/U/V]: pixel data (YUV) */

/* Convert to RGBA for GPU upload */
avifRGBImage rgb;
avifRGBImageSetDefaults(&rgb, img);
rgb.depth = (img->depth > 8) ? 16 : 8;
rgb.format = AVIF_RGB_FORMAT_RGBA;
avifImageYUVToRGB(img, &rgb);  /* YUV → RGBA with colour space transform */
```

dav1d uses heavy SIMD (AVX2/AVX-512 on x86, NEON/SVE on ARM) and multi-threaded tile decoding. A 12 MP AVIF (lossy) decodes in 150–400 ms on a single core; with 4 threads this drops to 60–120 ms. AVIF decode is consistently slower than JPEG at equivalent resolution, which is the primary adoption barrier for large-image workloads.

### 6.3 AVIF Animation

AVIF supports animation via the ISOBMFF `moov`/`mdat` structure: each frame is an AV1 intra frame (no inter-frame prediction in standard AVIF; inter-frame AVIF "sequences" are a separate profile not widely used). Chrome's `SkAvifCodec` decodes animated AVIF frame by frame, with frame durations from the `trak` box.

---

## 7. JPEG XL: Next-Generation Compression and Lossless JPEG Re-encoding

### 7.1 Compression Mechanism

**JPEG XL** (ISO/IEC 18181, `.jxl`) was finalised in 2022 as a next-generation general-purpose image codec. It uses two distinct internal encoding modes:

**VarDCT mode** (lossy, JPEG-like): A generalisation of JPEG's DCT approach:
- Adaptive quantisation with frequency-band grouping (`LfQuant`, `HfQuant`)
- Prediction using a 77-coefficient reference frame (`adaptive predictor`)
- XYB colour space (a perceptual space derived from human visual response, unlike JPEG's YCbCr)
- Context-adaptive entropy coding using ANS (Asymmetric Numeral Systems) with Brotli dictionary compression

**Modular mode** (lossless or near-lossless):
- Prediction using MA (Meta-Adaptive) trees with 6 predictor modes
- Delta palette and squeeze transforms for flat-colour images
- ANS entropy coding
- Optional Brotli post-compression

VarDCT lossy JPEG XL typically achieves 30–60% better compression than JPEG at equivalent visual quality (SSIM); lossless JPEG XL is 20–35% smaller than PNG.

**JPEG lossless recompression**: JPEG XL's most unusual feature. Any existing JPEG file can be losslessly transcoded to JPEG XL (and back) with a 20–30% size reduction, because JPEG XL's container can represent JPEG's DCT coefficients directly in a more efficient entropy coding. This is not a re-encode; the original JPEG pixels are mathematically recoverable bit-for-bit. This enables CDNs to serve JPEG XL to supporting browsers with smaller file sizes while retaining the original JPEG for non-supporting clients — at zero quality cost.

### 7.2 libjxl

Chrome uses **libjxl** (`third_party/libjxl/`) — the reference implementation from the JPEG XL working group:

```c
JxlDecoder *dec = JxlDecoderCreate(NULL);
JxlDecoderSubscribeEvents(dec,
    JXL_DEC_BASIC_INFO | JXL_DEC_COLOR_ENCODING | JXL_DEC_FULL_IMAGE);

JxlDecoderSetInput(dec, jxl_data, jxl_size);
JxlDecoderProcessInput(dec);

JxlBasicInfo info;
JxlDecoderGetBasicInfo(dec, &info);
/* info.xsize, info.ysize, info.bits_per_sample, info.alpha_bits */

/* Get ICC profile */
size_t icc_size;
JxlDecoderGetICCProfileSize(dec, JXL_COLOR_PROFILE_TARGET_DATA, &icc_size);
uint8_t *icc = malloc(icc_size);
JxlDecoderGetColorAsICCProfile(dec, JXL_COLOR_PROFILE_TARGET_DATA, icc, icc_size);

/* Decode pixels */
JxlPixelFormat format = { 4, JXL_TYPE_UINT8, JXL_NATIVE_ENDIAN, 0 };
JxlDecoderSetImageOutBuffer(dec, &format, rgba_buf, width * height * 4);
JxlDecoderProcessInput(dec);  /* completes decode */
```

libjxl uses AVX2/AVX-512 SIMD and multi-threaded decode for large images. Decode speed is comparable to or slightly slower than AVIF at equivalent quality levels.

### 7.3 Browser Status

As of mid-2026, JPEG XL is supported in Chrome (enabled by default since Chrome 110) and Safari (since Safari 17.0). Firefox has JPEG XL support behind a flag (`image.jxl.enabled`), with full enablement planned. The `image/jxl` MIME type is in Chrome's `Accept` header.

---

## 8. GIF and Animated Images

**GIF** (Graphics Interchange Format, 1987/1989) uses LZW compression over an indexed-colour palette of up to 256 colours. GIF images are inherently limited to 8-bit colour with no alpha blending (transparency is single-bit: fully transparent or fully opaque per-pixel via the GCE transparent colour index). GIF animation is a sequence of frames with per-frame delay, disposal method, and optional local palettes.

Chrome's GIF decoder (`third_party/skia/src/codec/SkGifCodec.cpp`) wraps a custom streaming LZW decoder. GIF decodes are fast (LZW is simple) but produce low-quality output for photographic content due to the 256-colour limit and absence of dithering in most encoders.

**GIF should be replaced by:**
- **Animated WebP**: same container, better compression, 8-bit colour plus full alpha
- **Animated AVIF**: AV1 intra frames, HDR capable, best compression
- **`<video>` with `loop autoplay muted`**: H.264 or AV1 video in a `<video>` element, which can use hardware-accelerated decode (Ch147) and is far more efficient than animated GIF for long animations

Chrome and Firefox display animated GIF natively; no codec selection is needed. The animation loop is driven by the Blink `ImageAnimationController` which schedules frame advances via a `TaskHandle` timer.

---

## 9. SVG: XML-Based Vector Graphics

**SVG** (Scalable Vector Graphics) is structurally different from all raster formats: it is not a pixel codec but an XML document describing geometric shapes, text, and embedded raster images. Blink parses SVG as a DOM subtree (`SVGDocument`, `SVGElement` hierarchy) and rasterises it on demand via Skia.

When an `<img src="icon.svg">` is loaded, Blink:
1. Parses the SVG XML into an `SVGDocument`.
2. Applies CSS styling and layout to the SVG element tree.
3. Paints the SVG to an `SkCanvas` via the Blink paint system, which traverses the SVG element tree and emits `SkPath`, `SkPaint`, and `SkTextBlob` calls.
4. The resulting raster (at the requested display resolution) is cached as a `gpu::SharedImage` texture.

SVG rasterisation happens at the display resolution, so SVG images remain sharp at any zoom level. The cost is re-rasterisation on zoom change (the cached texture is at a fixed resolution and is discarded when the size changes significantly). Chrome caches the raster at the element's layout size; very large SVGs may be rasterised in tiles.

Inline SVG (SVG embedded in HTML) is handled differently: it is part of the main document DOM and is painted by the normal Blink paint pipeline rather than as an image resource.

---

## 10. ICC Profiles and Colour Management Through the Browser Pipeline

Every raster image format supports embedding an ICC profile describing the colour space of the encoded pixel values. Chrome's colour management pipeline:

```
Image file (JPEG/PNG/WebP/AVIF/JXL)
  │  ICC profile embedded in file (JPEG APP2, PNG iCCP, AVIF colr/colr, JXL color_encoding)
  │
SkCodec / skcms
  │  Parse ICC profile → skcms_ICCProfile
  │  Convert pixel values from image colour space to sRGB (or compositor working space)
  │  (For HDR: convert to scRGB / BT.2100 PQ if display supports it)
  │
SkBitmap (sRGB RGBA8 for SDR, or scRGB RGBA F16 for HDR)
  │
GPU upload → gpu::SharedImage (VkImage)
  │
Viz compositor
  │  For SDR output: compositor works in sRGB; no further colour transform
  │  For HDR output (wp_color_management_v1): compositor applies output colour space
  │  transform before handing to KMS
  │
KMS: drm_hdr_output_metadata or VCGT gamma LUT
  │
Display
```

**skcms** (`modules/skcms/`) is Skia's compact, fast ICC profile library. It handles:
- Parsing ICC v2 and v4 profiles
- Matrix+TRC (tone response curve) profiles: 3×3 matrix + per-channel gamma curve; covers sRGB, Display P3, BT.2020
- Table-based profiles: arbitrary 3D LUT for advanced colour spaces
- Conversion between arbitrary profiles via the PCS (Profile Connection Space, D50 XYZ)

For AVIF and JPEG XL, colour space information may be expressed as **NCLX** (Numeric Coded Colour Space) metadata — primaries code, transfer function code, matrix coefficients — rather than an ICC profile. Blink maps these codes to `SkColorSpace` objects directly, bypassing ICC profile parsing.

**HDR image display**: AVIF images with PQ or HLG transfer functions and BT.2020 primaries are treated as HDR content. Chrome maps these to `SkColorSpace::MakeRGB(SkNamedTransferFn::kPQ, SkNamedGamut::kRec2020)` and, when the display and compositor support it (via `wp_color_management_v1`, Ch194), routes them through the HDR compositor path rather than tonemapping to SDR. On SDR displays, Chrome applies a simple Reinhard tonemap in skcms during decode.

---

## 11. GPU Upload Path: CPU Decode to Compositor Texture

The canonical upload path from a decoded `SkBitmap` to a `VkImage` usable by the Viz compositor:

```
SkBitmap (CPU, RGBA8 or F16)
  │
  CreateSharedImage (Mojo IPC to GPU process)
    │
    GPU process: SharedImageFactory
      │
      vkCreateBuffer (VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | HOST_COHERENT)
        memcpy(mapped_ptr, bitmap.getPixels(), size)
        vkUnmapMemory
      │
      vkCreateImage (VK_FORMAT_R8G8B8A8_UNORM or VK_FORMAT_R16G16B16A16_SFLOAT)
        VK_IMAGE_USAGE_SAMPLED_BIT | TRANSFER_DST_BIT
        VK_IMAGE_LAYOUT_UNDEFINED → TRANSFER_DST_OPTIMAL (pipeline barrier)
      │
      vkCmdCopyBufferToImage
      │
      VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL → SHADER_READ_ONLY_OPTIMAL
      │
      vkQueueSubmit (fence)
  │
  gpu::Mailbox token returned to renderer process
  │
  Blink: ImageResource holds Mailbox via gpu::ClientSharedImageInterface
  │
  Viz: TextureDrawQuad references Mailbox
  → viz::SkiaRenderer: vkCmdBindDescriptorSets (sampler + VkImageView)
                         vkCmdDraw (textured quad)
```

**Image tiling**: Very large images (e.g., 64 MP from a camera) are decoded and uploaded in tiles to avoid allocating a single large `VkImage`. Blink's `DeferredImageDecoder` supports partial decode; the compositor's `TileManager` allocates tiles and triggers decode of the visible region first.

**Decoded image cache**: Blink maintains a decoded image cache (`blink::ImageDecodingStore`) keyed by `(ImageResource, decode_options)`. Cache hits skip the CPU decode entirely and return the cached `SkBitmap`. The cache is size-limited (default 128 MB on desktop) with LRU eviction.

---

## 12. The ImageDecoder API (WebCodecs)

The `ImageDecoder` interface (part of the WebCodecs API, Ch146) exposes image format decoding directly to JavaScript, with explicit control over decode timing, frame selection, and colour space handling:

```js
const response = await fetch('animation.avif');
const contentType = response.headers.get('Content-Type');  // 'image/avif'
const imageData = await response.arrayBuffer();

const decoder = new ImageDecoder({
    data: imageData,
    type: contentType,
    colorSpaceConversion: 'none',    // preserve original colour space, don't convert to sRGB
    desiredWidth: 512, desiredHeight: 512,  // request downscale during decode (not all codecs support)
    preferAnimation: true,           // prefer animated track if ambiguous
});

await decoder.tracks.ready;
const track = decoder.tracks.selectedTrack;
console.log(`${track.frameCount} frames, ${decoder.type}`);

// Decode individual frames
for (let i = 0; i < track.frameCount; i++) {
    const result = await decoder.decode({ frameIndex: i });
    const frame = result.image;  // VideoFrame
    console.log(frame.codedWidth, frame.colorSpace);
    // Draw to canvas, process with WebGPU, send to VideoEncoder, etc.
    frame.close();
}
decoder.close();
```

`ImageDecoder` decodes to `VideoFrame` objects (the same type used by `VideoDecoder`), which carry pixel data as either CPU-accessible `ArrayBuffer` (via `frame.copyTo()`) or as a GPU texture importable into WebGPU via `device.importExternalTexture({ source: frame })`.

Supported types (as of Chrome 94+): `image/jpeg`, `image/png`, `image/webp`, `image/avif`, `image/gif`. JPEG XL support via `ImageDecoder` is in progress.

**ImageDecoder advantages over `<img>`**:
- Access to individual animation frames
- Explicit decode timing (decode on demand, not on load)
- `VideoFrame` output: direct WebGPU import without CPU copy
- `colorSpaceConversion: 'none'` preserves original colour space data
- Works in Web Workers

---

## 13. Format Comparison

| Format | Lossy | Lossless | Alpha | Animation | HDR | Bit depth | Relative size (photo) | Decode speed | Browser support |
|---|---|---|---|---|---|---|---|---|---|
| JPEG | ✓ | — | — | — | — | 8-bit | 1× (baseline) | Fast | Universal |
| PNG | — | ✓ | ✓ | — | ✓ (16-bit) | 1–16-bit | 3–5× | Medium | Universal |
| WebP | ✓ | ✓ | ✓ | ✓ | — | 8-bit | 0.65× | Fast | Universal |
| AVIF | ✓ | ✓ | ✓ | ✓ | ✓ | 8/10/12-bit | 0.45× | Slow | Chrome/Firefox/Safari |
| JPEG XL | ✓ | ✓ | ✓ | ✓ | ✓ | 8/16/32-bit | 0.40× | Medium | Chrome/Safari (FF flag) |
| GIF | — | ✓ | 1-bit | ✓ | — | 8-bit (palette) | 2–4× | Fast | Universal |

**Decision guide:**
- **Photos without transparency, wide compatibility**: JPEG (legacy) or WebP (modern)
- **Photos, best compression, HDR support**: AVIF
- **Photos, future-proof, lossless JPEG recompression**: JPEG XL
- **Logos, icons, UI assets (transparency required)**: WebP lossless or PNG
- **Vector graphics, icons at multiple sizes**: SVG
- **Short animations, GIF replacement**: Animated WebP or `<video autoplay loop muted>`
- **HDR photography**: AVIF (now) or JPEG XL (where supported)

---

## 14. Roadmap

### Near-term (6–12 months)

- **AVIF hardware decode**: Intel Gen12+ and Apple Silicon have hardware AV1 decode units that VA-API can expose. Browser AVIF decode does not yet route through VA-API (it uses dav1d in software). Integrating AVIF decode through the VA-API / `VK_KHR_video_decode_av1` path would reduce decode time by 5–10× and GPU memory bus pressure for large or animated AVIF files. Chrome's Media team tracks this under the WebCodecs hardware acceleration initiative.
- **JPEG XL default in Firefox**: Firefox ships JPEG XL behind `image.jxl.enabled`; full enablement is expected once the libjxl integration completes performance validation on Linux ARM (Raspberry Pi, mobile SoCs).
- **`ImageDecoder` JPEG XL support**: The WebCodecs `ImageDecoder` API currently omits JPEG XL; adding `image/jxl` support requires integrating libjxl's incremental decode path into Chrome's `ImageDecoder` implementation.
- **AVIF HDR display pipeline**: Chrome on Linux does not yet propagate AVIF `clli`/`mdcv` HDR metadata through the compositor to KMS `drm_hdr_output_metadata`. Completing this path requires integrating AVIF colour space metadata with `wp_color_management_v1` surface colour descriptions (Ch194 §5).

### Medium-term (1–3 years)

- **PNG `cICP` chunk support (PNG HDR)**: The PNG 3rd edition (ISO/IEC 15948:2024) added the `cICP` chunk for NCLX-style colour space signalling — enabling 16-bit HDR PNG without embedding a full ICC profile. Browser implementations are in early stages.
- **WebP 2**: Google has been developing WebP 2, an experimental successor with better compression than WebP. It is currently in research and has not been announced for browser inclusion. Note: needs verification.
- **AV2 / EVC for AVIF sequences**: Next-generation video codecs (AV2 from AOM, EVC from MPEG) may eventually enable AVIF-like still image formats with even better compression. Timeline is speculative (2028+).
- **`ImageBitmap` colour space preservation**: Currently `createImageBitmap()` always converts to sRGB. A proposal to add `colorSpaceConversion: 'none'` and return the original colour space metadata would allow WebGPU to perform HDR image processing with full fidelity.

### Long-term

- **Universal hardware image decode**: As AV1 hardware decode becomes ubiquitous (already in most 2022+ SoCs), AVIF decode may move to hardware paths in all browsers, eliminating the decode-speed disadvantage vs. JPEG.
- **JPEG XL lossless JPEG recompression as a CDN standard**: If JPEG XL adoption reaches critical mass, CDNs may transparently re-encode existing JPEG images to JPEG XL, saving 20–30% bandwidth with zero quality loss and zero client-side change.

---

## 15. Integrations

- **Chapter 36 (Chromium Compositor — CC and Viz)**: Image textures produced by the decode pipeline are referenced by `viz::TextureDrawQuad`s in the Viz compositor. The SharedImage mailbox system (§1.5) is described in Ch36. The Viz `SkiaRenderer` that samples these textures to produce the final composited frame is also covered there.

- **Chapter 37 (Skia and 2D Rendering)**: All raster image decode in Chrome passes through `SkCodec` (Skia's codec abstraction). The `SkBitmap` produced by `SkCodec::getPixels()` is the same type that `CanvasRenderingContext2D::drawImage()` consumes. The `createImageBitmap()` path (§1.3, §12.7 of Ch37) is the async entry point shared between image loading and canvas drawing.

- **Chapter 146 (WebCodecs and Browser Hardware Acceleration)**: `ImageDecoder` (§12 of this chapter) is part of the WebCodecs API covered in Ch146. The `VideoFrame` output of `ImageDecoder` is the same type produced by `VideoDecoder`; both can be imported into WebGPU via `device.importExternalTexture()`.

- **Chapter 147 (Chrome/Firefox VA-API Video Decode)**: AVIF hardware decode (§14 roadmap) would use the same VA-API / `VK_KHR_video_decode_av1` infrastructure that Ch147 covers for video. The absence of AVIF in the current hardware decode path is a gap between the video and image decode stacks.

- **Chapter 74 (HDR and Wide Color Gamut on Linux)**: AVIF and JPEG XL HDR images carry `MaxCLL`/`MaxFALL` and mastering display metadata (§6, §10) that maps to the KMS `drm_hdr_output_metadata` structures described in Ch74.

- **Chapter 194 (Cross-Stack Integration)**: The `wp_color_management_v1` surface colour description (§10 of this chapter) is the Wayland protocol mechanism described in Ch194 §5 that allows the compositor to tonemap or pass through HDR image content correctly.

- **Chapter 35 (Dawn and WebGPU)**: `ImageDecoder` output as `VideoFrame` can be imported into WebGPU via `device.importExternalTexture()`. The `wgpu::SharedTextureMemory` DMA-BUF import path (Ch35 §13) is the mechanism that allows zero-copy GPU access to decoded image data.

---

## References

- WebP specification: [https://developers.google.com/speed/webp/docs/riff_container](https://developers.google.com/speed/webp/docs/riff_container)
- AVIF specification (AOM): [https://aomediacodec.github.io/av1-avif/](https://aomediacodec.github.io/av1-avif/)
- JPEG XL specification (ISO 18181): [https://jpeg.org/jpegxl/](https://jpeg.org/jpegxl/)
- libjxl reference implementation: [https://github.com/libjxl/libjxl](https://github.com/libjxl/libjxl)
- libavif: [https://github.com/AOMediaCodec/libavif](https://github.com/AOMediaCodec/libavif)
- libwebp: [https://chromium.googlesource.com/webm/libwebp](https://chromium.googlesource.com/webm/libwebp)
- libjpeg-turbo: [https://libjpeg-turbo.org/](https://libjpeg-turbo.org/)
- Skia SkCodec: [https://skia.googlesource.com/skia/+/refs/heads/main/src/codec/](https://skia.googlesource.com/skia/+/refs/heads/main/src/codec/)
- skcms (Skia colour management): [https://skia.googlesource.com/skia/+/refs/heads/main/modules/skcms/](https://skia.googlesource.com/skia/+/refs/heads/main/modules/skcms/)
- WebCodecs ImageDecoder API: [https://www.w3.org/TR/webcodecs/#imagedecoder-interface](https://www.w3.org/TR/webcodecs/#imagedecoder-interface)
- PNG 3rd edition (cICP chunk): [https://www.w3.org/TR/png-3/](https://www.w3.org/TR/png-3/)
- Chrome image format support (MDN): [https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
