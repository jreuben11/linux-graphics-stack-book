# Chapter 142: V4L2 and the Linux Media Subsystem

> **Part**: Part XIII — Video Streaming on Linux
>
> **Audience**: Systems and driver developers building camera, ISP, or codec drivers; application developers constructing V4L2 capture, decode, and zero-copy GPU pipelines.
>
> **Status**: First draft — 2026-06-19

---

## Table of Contents

1. [Overview](#1-overview)
2. [V4L2 Architecture: Device Nodes and the Media Controller](#2-v4l2-architecture-device-nodes-and-the-media-controller)
   - [Device Node Taxonomy](#21-device-node-taxonomy)
   - [The Media Controller Framework](#22-the-media-controller-framework)
   - [Entity, Pad, and Link Graph](#23-entity-pad-and-link-graph)
3. [V4L2 Device Types](#3-v4l2-device-types)
   - [Capture Devices](#31-capture-devices)
   - [Output Devices](#32-output-devices)
   - [Memory-to-Memory (M2M) Devices](#33-memory-to-memory-m2m-devices)
   - [Overlay Devices](#34-overlay-devices)
4. [The V4L2 ioctl API](#4-the-v4l2-ioctl-api)
   - [Capability Query](#41-capability-query)
   - [Format Enumeration and Negotiation](#42-format-enumeration-and-negotiation)
   - [Pixel Format FourCC Taxonomy](#43-pixel-format-fourcc-taxonomy)
5. [Memory Models and Buffer Management](#5-memory-models-and-buffer-management)
   - [MMAP Buffers](#51-mmap-buffers)
   - [USERPTR Buffers](#52-userptr-buffers)
   - [DMABUF: Zero-Copy Import and Export](#53-dmabuf-zero-copy-import-and-export)
6. [Streaming and the Buffer Lifecycle State Machine](#6-streaming-and-the-buffer-lifecycle-state-machine)
   - [select() and poll()](#61-select-and-poll)
   - [The vb2_buffer_state Machine](#62-the-vb2_buffer_state-machine)
7. [V4L2 Subdev API and Sensor Architecture](#7-v4l2-subdev-api-and-sensor-architecture)
   - [MEDIA_IOC_SETUP_LINK and Pipeline Routing](#71-media_ioc_setup_link-and-pipeline-routing)
   - [Pad-Level Format Negotiation](#72-pad-level-format-negotiation)
   - [Sensor Driver Architecture: IMX219 Example](#73-sensor-driver-architecture-imx219-example)
8. [libcamera: C++ Abstraction over V4L2](#8-libcamera-c-abstraction-over-v4l2)
   - [Architecture and Pipeline Handlers](#81-architecture-and-pipeline-handlers)
   - [IPA Modules and Sandboxing](#82-ipa-modules-and-sandboxing)
   - [V4L2-Compat Library for Legacy Applications](#83-v4l2-compat-library-for-legacy-applications)
9. [DMA-BUF Bridge to the GPU](#9-dma-buf-bridge-to-the-gpu)
   - [EGL Import](#91-egl-import)
   - [Vulkan Import](#92-vulkan-import)
   - [OpenCL Import](#93-opencl-import)
10. [V4L2 Codec Devices: Stateful vs. Stateless](#10-v4l2-codec-devices-stateful-vs-stateless)
    - [Stateful Codecs](#101-stateful-codecs)
    - [Stateless Codecs and the Request API](#102-stateless-codecs-and-the-request-api)
11. [The Request API In Depth](#11-the-request-api-in-depth)
    - [Allocation, Queuing, and Polling](#111-allocation-queuing-and-polling)
    - [Synchronisation: Fences vs. Request API](#112-synchronisation-fences-vs-request-api)
12. [GStreamer V4L2 Elements](#12-gstreamer-v4l2-elements)
    - [v4l2src, v4l2sink, v4l2convert](#121-v4l2src-v4l2sink-v4l2convert)
    - [Zero-Copy DMA-BUF Pipeline](#122-zero-copy-dma-buf-pipeline)
13. [Raspberry Pi CSI Pipeline](#13-raspberry-pi-csi-pipeline)
    - [Pi 4: bcm2835-unicam and the VC4 ISP](#131-pi-4-bcm2835-unicam-and-the-vc4-isp)
    - [Pi 5: rp1-cfe and the PiSP Back End](#132-pi-5-rp1-cfe-and-the-pisp-back-end)
    - [libcamera Pipeline Handlers for Raspberry Pi](#133-libcamera-pipeline-handlers-for-raspberry-pi)
14. [Integrations](#14-integrations)
15. [References](#15-references)

---

## 1. Overview

Video4Linux version 2 (V4L2) is the Linux kernel subsystem that exposes camera sensors, video capture hardware, hardware video codecs, image signal processors (ISPs), and a growing class of compute-on-frame devices to userspace. It is not a single monolithic API but a layered system: a generic capture/output ioctl API sitting on top of a buffer-management framework (videobuf2, vb2), unified by the media controller graph that describes how physical hardware blocks connect.

This chapter targets two audiences. **Systems and driver developers** will find architectural walkthroughs of the key kernel subsystems — the media controller entity-pad-link model, the vb2 buffer state machine, the subdev format negotiation protocol, and the request API for per-frame parameter delivery. **Application developers** will find concrete ioctl sequences, the DMABUF zero-copy patterns that bridge V4L2 frames to EGL, Vulkan, and GStreamer, and the stateless codec programming model required for Raspberry Pi, Allwinner Cedrus, and Rockchip RKVDEC devices.

Where the kernel subsystem is well-documented but companion APIs are in flux (notably fence-based per-buffer synchronisation), this chapter explicitly distinguishes what is upstream and stable from what is a long-standing out-of-tree proposal.

---

## 2. V4L2 Architecture: Device Nodes and the Media Controller

### 2.1 Device Node Taxonomy

V4L2 presents three classes of character device nodes:

- `/dev/videoN` — the primary userspace interface for capture, output, memory-to-memory (M2M), and overlay devices. A single physical pipeline may expose multiple video nodes: for example, a CSI-2 receiver can expose one `/dev/videoN` per DMA channel.
- `/dev/mediaM` — the media controller node. It exposes the entity/pad/link graph of a media device and hosts the request API (`MEDIA_IOC_REQUEST_ALLOC`). Each logical media device has exactly one `/dev/mediaM`.
- `/dev/v4l-subdevX` — raw subdevice access. Opened directly by applications (or libcamera) that need pad-level format negotiation (`VIDIOC_SUBDEV_S_FMT`) or direct sensor control beyond what the video node exposes.

A single physical camera module typically exposes all three: a `/dev/v4l-subdevX` for the sensor, one or more `/dev/videoN` for the capture output queues, and a `/dev/mediaM` for the graph connecting them through a CSI-2 receiver subdev.

The capability flag `V4L2_CAP_IO_MC` (added in kernel 5.11) indicates a media-controller-centric device. On such devices `VIDIOC_ENUMINPUT` and `VIDIOC_ENUMOUTPUT` are handled automatically, and all pipeline routing must be established via the media controller before streaming starts. [Source](https://docs.kernel.org/userspace-api/media/v4l/vidioc-querycap.html)

### 2.2 The Media Controller Framework

The media controller framework, implemented in `drivers/media/mc/media-device.c` and `drivers/media/mc/media-request.c`, manages a directed graph of hardware blocks. It was introduced to handle the complex topologies of embedded SoC camera pipelines where a single sensor output may be routed to different ISP paths depending on the use case, and where per-frame configuration must be atomic across multiple drivers.

Userspace interfaces to the media controller through `/dev/mediaM`:

```c
/* drivers/media/mc/media-device.c (conceptual ioctl dispatch) */
MEDIA_IOC_DEVICE_INFO   /* query topology version and driver info */
MEDIA_IOC_ENUM_ENTITIES /* enumerate all hardware blocks */
MEDIA_IOC_ENUM_LINKS    /* enumerate pads and links for an entity */
MEDIA_IOC_SETUP_LINK    /* enable or disable a link */
MEDIA_IOC_G_TOPOLOGY    /* get the full graph in one call (newer API) */
MEDIA_IOC_REQUEST_ALLOC /* allocate a request fd (request API) */
```

[Source](https://docs.kernel.org/driver-api/media/mc-core.html)

The `media-ctl` tool from the `v4l-utils` package provides a human-readable interface:

```bash
media-ctl -d /dev/media0 --print-topology
```

### 2.3 Entity, Pad, and Link Graph

Every hardware block exposed by the media controller is a **media entity** (struct `media_entity`). Entities have **pads** — directional ports that are either sinks (data enters) or sources (data leaves). **Links** connect a source pad of one entity to the sink pad of another.

```
[imx219 sensor] ---source pad 0---> [rp1-cfe CSI-2] ---source pad ch0---> [/dev/video0]
                                                      ---source pad ch1---> [/dev/video1]
```

Links are typed:
- `MEDIA_LNK_FL_ENABLED` — active link, data flows
- `MEDIA_LNK_FL_IMMUTABLE` — cannot be changed at runtime (e.g., internal hardware connections)
- `MEDIA_LNK_FL_DYNAMIC` — can be changed while streaming (driver-declared capability)

Configuring a pipeline consists of: (1) enabling the correct links via `MEDIA_IOC_SETUP_LINK`; (2) negotiating the format on each subdev pad with `VIDIOC_SUBDEV_S_FMT`; (3) setting the video node format with `VIDIOC_S_FMT`; (4) allocating buffers; (5) starting streaming. The order of steps 1–3 is mandatory because format negotiation must propagate from the sensor through each intermediate entity to the memory node.

---

## 3. V4L2 Device Types

### 3.1 Capture Devices

Capture devices read data from an external source (camera, tuner, HDMI input) into system memory. The buffer type is `V4L2_BUF_TYPE_VIDEO_CAPTURE` (single-plane) or `V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE` (multi-planar, added in kernel 2.6.39 for separate luma/chroma planes). The device produces data; userspace dequeues filled frames.

### 3.2 Output Devices

Output devices send frames from system memory to a display or encoder. Buffer type `V4L2_BUF_TYPE_VIDEO_OUTPUT` or `V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE`. Userspace queues frames; the device consumes them.

### 3.3 Memory-to-Memory (M2M) Devices

M2M devices process data from an input queue (OUTPUT type, confusingly named) to an output queue (CAPTURE type). This covers hardware video encoders, decoders, scalers, and ISPs. The device has two independent vb2 queues: userspace queues a source buffer to the OUTPUT queue and dequeues the result from the CAPTURE queue.

The M2M framework is implemented in `drivers/media/v4l2-core/v4l2-mem2mem.c`. Drivers register with `v4l2_m2m_init()` and expose the `vb2_ops` for each queue separately. Userspace must call `VIDIOC_REQBUFS` independently for each buffer type.

### 3.4 Overlay Devices

Overlay devices (buffer type `V4L2_BUF_TYPE_VIDEO_OVERLAY`) perform direct DMA into a display framebuffer region without kernel-managed buffer queues. They are legacy on modern systems and seldom implemented in new drivers. This chapter does not cover them further.

---

## 4. The V4L2 ioctl API

### 4.1 Capability Query

```c
/* include/uapi/linux/videodev2.h */
struct v4l2_capability {
    __u8    driver[16];     /* driver name, e.g. "rp1-cfe" */
    __u8    card[32];       /* device name */
    __u8    bus_info[32];   /* PCI or platform bus identifier */
    __u32   version;        /* kernel version (KERNEL_VERSION macro) */
    __u32   capabilities;   /* all device capabilities (V4L2_CAP_*) */
    __u32   device_caps;    /* capabilities of this specific /dev/videoN */
    __u32   reserved[3];
};

ioctl(fd, VIDIOC_QUERYCAP, &cap);
```

Key `V4L2_CAP_*` flags: `V4L2_CAP_VIDEO_CAPTURE`, `V4L2_CAP_VIDEO_M2M`, `V4L2_CAP_STREAMING` (required for MMAP/DMABUF), `V4L2_CAP_IO_MC` (MC-centric pipeline, kernel 5.11+). Always test `device_caps`, not `capabilities`, as the latter aggregates capabilities across all nodes of the same driver. [Source](https://docs.kernel.org/userspace-api/media/v4l/vidioc-querycap.html)

### 4.2 Format Enumeration and Negotiation

```c
/* Enumerate supported pixel formats */
struct v4l2_fmtdesc fmtdesc = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE };
for (fmtdesc.index = 0; ioctl(fd, VIDIOC_ENUM_FMT, &fmtdesc) == 0; fmtdesc.index++) {
    printf("  %s (%.4s)\n", fmtdesc.description, (char *)&fmtdesc.pixelformat);
}

/* Negotiate format: driver adjusts to nearest supported */
struct v4l2_format fmt = {
    .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .fmt.pix = {
        .width       = 1920,
        .height      = 1080,
        .pixelformat = V4L2_PIX_FMT_NV12,
        .field       = V4L2_FIELD_NONE,
    },
};
ioctl(fd, VIDIOC_S_FMT, &fmt);
/* fmt.fmt.pix.{width,height,bytesperline,sizeimage} updated by driver */
```

`VIDIOC_TRY_FMT` probes without committing. `VIDIOC_G_FMT` reads the current format. [Source](https://docs.kernel.org/userspace-api/media/v4l/vidioc-g-fmt.html)

### 4.3 Pixel Format FourCC Taxonomy

V4L2 uses 32-bit FourCC codes defined in `include/uapi/linux/videodev2.h`. Major categories:

| FourCC Constant | Value | Description |
|---|---|---|
| `V4L2_PIX_FMT_NV12` | `'N','V','1','2'` | YUV 4:2:0, semi-planar (Y plane, interleaved UV) |
| `V4L2_PIX_FMT_NV21` | `'N','V','2','1'` | YUV 4:2:0, semi-planar (Y plane, interleaved VU) |
| `V4L2_PIX_FMT_YUV420M` | multi-planar | YUV 4:2:0, three separate planes |
| `V4L2_PIX_FMT_YUYV` | `'Y','U','Y','V'` | YUV 4:2:2 packed |
| `V4L2_PIX_FMT_MJPEG` | `'M','J','P','G'` | Motion JPEG |
| `V4L2_PIX_FMT_H264` | `'H','2','6','4'` | H.264 Annex B (stateful decoder input) |
| `V4L2_PIX_FMT_H264_SLICE` | `'S','2','6','4'` | H.264 slice data (stateless decoder input) |
| `V4L2_PIX_FMT_HEVC` | `'H','E','V','C'` | H.265/HEVC (stateful) |
| `V4L2_PIX_FMT_VP9_FRAME` | stateless VP9 | VP9 compressed frame (stateless) |
| `V4L2_PIX_FMT_AV1_FRAME` | stateless AV1 | AV1 OBU frame |
| `V4L2_PIX_FMT_SRGGB10` | Bayer 10-bit | Raw Bayer RGGB 10-bit packed |
| `V4L2_PIX_FMT_SRGGB12` | Bayer 12-bit | Raw Bayer RGGB 12-bit packed |

The distinction between `V4L2_PIX_FMT_H264` and `V4L2_PIX_FMT_H264_SLICE` is architecturally significant: the former is the input to a **stateful** decoder that parses the bitstream itself; the latter is the input to a **stateless** decoder that requires the application to supply parsed SPS/PPS and slice headers via the request API controls (Section 10). [Source](https://docs.kernel.org/userspace-api/media/v4l/pixfmt-compressed.html)

---

## 5. Memory Models and Buffer Management

### 5.1 MMAP Buffers

MMAP is the most common memory model. The kernel allocates DMA-coherent memory; userspace maps it.

```c
/* include/uapi/linux/videodev2.h */
struct v4l2_requestbuffers req = {
    .count  = 4,
    .type   = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .memory = V4L2_MEMORY_MMAP,
};
ioctl(fd, VIDIOC_REQBUFS, &req);
/* req.count may be adjusted by driver */

struct v4l2_buffer buf = { .type = req.type, .memory = V4L2_MEMORY_MMAP, .index = i };
ioctl(fd, VIDIOC_QUERYBUF, &buf);
void *ptr = mmap(NULL, buf.length, PROT_READ|PROT_WRITE, MAP_SHARED, fd, buf.m.offset);

/* Queue all buffers, then start streaming */
ioctl(fd, VIDIOC_QBUF, &buf);
int type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
ioctl(fd, VIDIOC_STREAMON, &type);

/* Capture loop */
struct v4l2_buffer dqbuf = { .type = req.type, .memory = V4L2_MEMORY_MMAP };
ioctl(fd, VIDIOC_DQBUF, &dqbuf);
/* process frame at ptr[dqbuf.index] ... */
ioctl(fd, VIDIOC_QBUF, &dqbuf);  /* re-queue */
```

The vb2 allocator backing MMAP buffers is selected by the driver: `videobuf2-dma-contig.c` for contiguous DMA (typical for embedded SoCs), `videobuf2-dma-sg.c` for scatter-gather (x86 capture cards), or `videobuf2-vmalloc.c` for drivers not requiring DMA-coherent memory (USB webcams). [Source](https://www.kernel.org/doc/html/v6.2/driver-api/media/v4l2-videobuf2.html)

### 5.2 USERPTR Buffers

`V4L2_MEMORY_USERPTR` allows userspace to provide pre-allocated memory. The buffer `m.userptr` field holds a virtual address. This is rarely used for performance in modern kernels because the kernel must pin and DMA-map each page on every queue/dequeue cycle. It is primarily useful for legacy systems. DMABUF (Section 5.3) is the correct zero-copy mechanism for inter-subsystem sharing.

### 5.3 DMABUF: Zero-Copy Import and Export

V4L2 supports two complementary DMABUF directions:

**Export (VIDIOC_EXPBUF):** Export a MMAP buffer as a DMABUF fd. Available since approximately kernel 3.8 [Source](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/vidioc-expbuf.html), restricted to `V4L2_MEMORY_MMAP` allocations only.

```c
/* Allocate MMAP buffers first */
struct v4l2_requestbuffers req = {
    .count  = 4,
    .type   = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .memory = V4L2_MEMORY_MMAP,
};
ioctl(fd, VIDIOC_REQBUFS, &req);

/* Export each buffer as a DMABUF fd */
struct v4l2_exportbuffer expbuf = {
    .type  = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .index = i,
    .plane = 0,             /* plane index for multi-planar */
    .flags = O_CLOEXEC | O_RDWR,
};
ioctl(fd, VIDIOC_EXPBUF, &expbuf);
int dmabuf_fd = expbuf.fd;
```

**Import (V4L2_MEMORY_DMABUF):** Use DMABUF fds allocated externally (by GBM, another V4L2 device, or a GPU driver) as the backing memory for V4L2 buffers.

```c
struct v4l2_requestbuffers req = {
    .count  = 4,
    .type   = V4L2_BUF_TYPE_VIDEO_OUTPUT,
    .memory = V4L2_MEMORY_DMABUF,
};
ioctl(encoder_fd, VIDIOC_REQBUFS, &req);

struct v4l2_buffer buf = {
    .type   = V4L2_BUF_TYPE_VIDEO_OUTPUT,
    .memory = V4L2_MEMORY_DMABUF,
    .index  = i,
    .m.fd   = dmabuf_fd,   /* fd from camera or GPU allocator */
};
ioctl(encoder_fd, VIDIOC_QBUF, &buf);
```

For multi-planar buffers, each plane carries its own fd in `planes[i].m.fd`. DMABUF import was integrated into vb2 in approximately kernel 3.5. [Source](https://docs.kernel.org/driver-api/media/v4l2-videobuf2.html)

The zero-copy producer→consumer pattern:

1. Camera device allocates via `VIDIOC_REQBUFS(MMAP)`.
2. Camera calls `VIDIOC_EXPBUF` per buffer to obtain DMABUF fds.
3. Encoder device receives those fds via `VIDIOC_REQBUFS(DMABUF)` and `VIDIOC_QBUF(m.fd=dmabuf_fd)`.
4. No CPU copy occurs; the DMA engine of the encoder reads directly from camera-allocated memory.

---

## 6. Streaming and the Buffer Lifecycle State Machine

### 6.1 select() and poll()

V4L2 video device nodes are pollable. For capture devices:

```c
struct pollfd pfd = { .fd = fd, .events = POLLIN };
poll(&pfd, 1, -1);
/* POLLIN set: VIDIOC_DQBUF will not block */
```

For output devices, `POLLOUT` indicates a buffer slot is available for `VIDIOC_QBUF`. `POLLPRI` signals an event (see `VIDIOC_DQEVENT`, used for resolution-change events from stateful decoders). [Source](https://docs.kernel.org/userspace-api/media/v4l/func-poll.html)

The request API (Section 11) uses a separate request fd for polling: `poll` on the request fd's `POLLIN` indicates the request has completed and all associated CAPTURE buffers are ready.

### 6.2 The vb2_buffer_state Machine

The videobuf2 framework (`drivers/media/common/videobuf2/videobuf2-core.c`) enforces a seven-state lifecycle for every buffer: [Source](https://www.kernel.org/doc/html/v6.2/driver-api/media/v4l2-videobuf2.html)

| State | Meaning |
|---|---|
| `VB2_BUF_STATE_DEQUEUED` | Under userspace control; returned by `VIDIOC_DQBUF` or freshly created |
| `VB2_BUF_STATE_IN_REQUEST` | Queued to a media request; waiting for `MEDIA_REQUEST_IOC_QUEUE` |
| `VB2_BUF_STATE_PREPARING` | Inside the driver's `buf_prepare` callback |
| `VB2_BUF_STATE_QUEUED` | In the vb2 queue; not yet passed to the hardware driver |
| `VB2_BUF_STATE_ACTIVE` | Hardware DMA engine owns the buffer |
| `VB2_BUF_STATE_DONE` | Hardware done; awaiting `VIDIOC_DQBUF` |
| `VB2_BUF_STATE_ERROR` | Done with error; `VIDIOC_DQBUF` returns it with `V4L2_BUF_FLAG_ERROR` set |

The `VB2_BUF_STATE_IN_REQUEST` state is specific to the stateless codec path. Buffers in this state do not enter the hardware queue until `MEDIA_REQUEST_IOC_QUEUE` is called on their associated request fd. This is the vb2-level mechanism that enables atomic per-frame parameter delivery (Section 11). [Source: `include/media/videobuf2-core.h`]

---

## 7. V4L2 Subdev API and Sensor Architecture

### 7.1 MEDIA_IOC_SETUP_LINK and Pipeline Routing

Before streaming from an MC-centric pipeline, links must be configured. The default state of links is driver-defined (often disabled for mux-select paths, enabled for fixed connections):

```c
struct media_link_desc link_desc = {
    .source = { .entity = sensor_entity_id, .index = 0, .flags = MEDIA_PAD_FL_SOURCE },
    .sink   = { .entity = csi_rx_entity_id, .index = 0, .flags = MEDIA_PAD_FL_SINK   },
    .flags  = MEDIA_LNK_FL_ENABLED,
};
ioctl(media_fd, MEDIA_IOC_SETUP_LINK, &link_desc);
```

[Source](https://docs.kernel.org/userspace-api/media/mediactl/media-ioc-setup-link.html)

In practice, userspace libraries such as libcamera or `media-ctl` handle link configuration. Direct ioctl use is required for custom pipeline implementations or drivers not yet covered by libcamera.

### 7.2 Pad-Level Format Negotiation

Subdev format negotiation uses a two-phase protocol: `V4L2_SUBDEV_FORMAT_TRY` probes the driver without committing; `V4L2_SUBDEV_FORMAT_ACTIVE` commits the format. The driver adjusts the format to the nearest supported configuration and returns the result.

```c
/* include/uapi/linux/v4l2-subdev.h */
struct v4l2_subdev_format fmt = {
    .which  = V4L2_SUBDEV_FORMAT_TRY,
    .pad    = 0,
    .format = {
        .width  = 3280,
        .height = 2464,
        .code   = MEDIA_BUS_FMT_SRGGB10_1X10,   /* 10-bit Bayer RGGB */
        .field  = V4L2_FIELD_NONE,
    },
};
ioctl(subdev_fd, VIDIOC_SUBDEV_S_FMT, &fmt);
/* fmt.format adjusted to nearest supported mode */

fmt.which = V4L2_SUBDEV_FORMAT_ACTIVE;
ioctl(subdev_fd, VIDIOC_SUBDEV_S_FMT, &fmt);
```

Format negotiation must propagate through the pipeline sink-to-source: sensor output pad → CSI-2 receiver sink pad → CSI-2 receiver source pad → video node. Each hop applies the `V4L2_SUBDEV_FORMAT_TRY` / `V4L2_SUBDEV_FORMAT_ACTIVE` cycle, in order. [Source](https://docs.kernel.org/userspace-api/media/v4l/vidioc-subdev-g-fmt.html)

### 7.3 Sensor Driver Architecture: IMX219 Example

The IMX219 (Sony 8 MP sensor, used in the Raspberry Pi Camera Module v2) is one of the best-maintained sensor drivers in the mainline kernel, located at `drivers/media/i2c/imx219.c`. It illustrates the standard subdev driver structure:

```c
/* drivers/media/i2c/imx219.c — key driver structures */
struct imx219 {
    struct v4l2_subdev      sd;
    struct media_pad        pad;          /* single source pad */
    struct regmap          *regmap;
    struct clk             *xclk;
    u32                     xclk_freq;
    struct gpio_desc       *reset_gpio;
    struct regulator_bulk_data supplies[IMX219_NUM_SUPPLIES];

    /* Control handler for exposure, gain, flip, blanking */
    struct v4l2_ctrl_handler ctrl_handler;
    struct v4l2_ctrl        *pixel_rate;
    struct v4l2_ctrl        *link_freq;
    struct v4l2_ctrl        *exposure;
    struct v4l2_ctrl        *vflip;
    struct v4l2_ctrl        *hflip;
    struct v4l2_ctrl        *vblank;
    struct v4l2_ctrl        *hblank;
    u8                       lanes;
};

static const struct v4l2_subdev_pad_ops imx219_pad_ops = {
    .enum_mbus_code  = imx219_enum_mbus_code,
    .get_fmt         = v4l2_subdev_get_fmt,
    .set_fmt         = imx219_set_pad_format,
    .get_selection   = imx219_get_selection,
    .enum_frame_size = imx219_enum_frame_size,
    .enable_streams  = imx219_enable_streams,   /* streams API, kernel 6.x+ */
    .disable_streams = imx219_disable_streams,
};

static const struct v4l2_subdev_video_ops imx219_video_ops = {
    .s_stream = v4l2_subdev_s_stream_helper,    /* delegates to enable/disable_streams */
};
```

The sensor driver registers its `v4l2_subdev` with `v4l2_async_register_subdev()` and waits for the platform driver (CSI-2 receiver) to claim it via the async notifier mechanism. The async notifier (`v4l2_async_notifier`, `drivers/media/v4l2-core/v4l2-async.c`) decouples sensor driver probe from CSI receiver driver probe: the CSI driver registers a notifier with a list of fwnodes (DT node handles) it expects; when each subdev driver registers itself, the core matches it to a pending notifier and fires the `.bound` callback. This is essential for correct probe ordering on embedded SoCs where sensor and CSI drivers may be initialised in any order. [Source](https://elixir.bootlin.com/linux/latest/source/drivers/media/i2c/imx219.c)

The control handler exposes AE/AWB parameters (`V4L2_CID_EXPOSURE`, `V4L2_CID_ANALOGUE_GAIN`) that libcamera's IPA module writes per-frame to implement auto-exposure algorithms. Standard control IDs are defined in `include/uapi/linux/v4l2-controls.h`; drivers use `v4l2_ctrl_new_std()` and `v4l2_ctrl_new_custom()` to register controls.

The newer `.enable_streams` / `.disable_streams` pad ops (introduced to replace `.s_stream` for multi-pad subdevs) allow per-stream start/stop on multi-stream sensors, which is required for the Pi 5's rp1-cfe driver that handles up to four simultaneous CSI-2 data streams.

#### Writing a Minimal Sensor Subdev Driver

The key entry points a sensor driver must implement are:

1. `probe()` — parse DT bindings (clock, regulators, reset GPIO, I2C address); initialise `v4l2_subdev` with `v4l2_i2c_subdev_init()`; populate `v4l2_ctrl_handler`; register async.
2. `pad_ops.set_fmt` — validate and store the requested mode; populate `v4l2_subdev_format.format` with the driver's nearest supported mode.
3. `pad_ops.enum_frame_size` — enumerate discrete or stepwise frame sizes.
4. `pad_ops.enable_streams` / `video_ops.s_stream` — write register tables to sensor over I2C/SPI; enable/disable MIPI DPHY lanes.

The IMX219 driver uses `cci_write()` (Camera Control Interface, a regmap abstraction for camera sensor registers introduced to simplify common read-modify-write patterns) from `drivers/media/v4l2-core/v4l2-cci.c`, available since approximately kernel 6.6. [Source](https://elixir.bootlin.com/linux/latest/source/drivers/media/v4l2-core/v4l2-cci.c)

---

## 8. libcamera: C++ Abstraction over V4L2

### 8.1 Architecture and Pipeline Handlers

libcamera ([https://libcamera.org/](https://libcamera.org/)) was created because the raw V4L2 media controller API, while sufficient for driver-level access, requires platform-specific knowledge to configure a production camera pipeline: which links to enable, what media bus format to negotiate on each subdev pad, how to configure the ISP, and how to implement auto-exposure and auto-white-balance against that ISP's statistics output. These tasks are different for every SoC vendor.

libcamera introduces **pipeline handlers** — platform-specific C++ classes that know the topology of a particular camera system and manage all the V4L2 media controller operations transparently:

- `src/libcamera/pipeline/rpi/vc4/vc4.cpp` — Raspberry Pi 4 (BCM2835/VC4 ISP)
- `src/libcamera/pipeline/rpi/pisp/pisp.cpp` — Raspberry Pi 5 (PiSP ISP)
- `src/libcamera/pipeline/uvcvideo/uvcvideo.cpp` — USB UVC cameras (simple, no ISP)
- `src/libcamera/pipeline/ipu3/ipu3.cpp` — Intel IPU3 on Skylake/Kaby Lake
- `src/libcamera/pipeline/simple/simple.cpp` — Generic simple capture devices

[Source](https://libcamera.org/api-html/)

The public libcamera API is intentionally simple: `CameraManager`, `Camera`, `CameraConfiguration`, `Request`, `FrameBuffer`, `Stream`. Applications acquire a camera, configure streams, allocate buffers from the camera allocator (which returns DMABUF-backed `FrameBuffer` objects), and queue `Request` objects. The pipeline handler translates each `Request` into V4L2 ioctls, media controller link configuration, and ISP control writes.

A minimal libcamera capture application:

```cpp
#include <libcamera/libcamera.h>
using namespace libcamera;

auto manager = std::make_unique<CameraManager>();
manager->start();

auto camera = manager->get(manager->cameras()[0]->id());
camera->acquire();

auto config = camera->generateConfiguration({ StreamRole::Viewfinder });
auto &stream_cfg = config->at(0);
stream_cfg.size = { 1920, 1080 };
stream_cfg.pixelFormat = formats::NV12;
camera->configure(config.get());

FrameBufferAllocator allocator(camera);
allocator.allocate(config->at(0).stream());

/* Each FrameBuffer is backed by DMABUF fds */
for (auto &fb : allocator.buffers(config->at(0).stream())) {
    auto request = camera->createRequest();
    request->addBuffer(config->at(0).stream(), fb.get());
    camera->queueRequest(request.get());
}

camera->start();
/* requestCompleted signal fires per frame */
```

The `FrameBuffer::planes()` vector exposes each plane's DMABUF fd, offset, and length, which can be passed directly to `eglCreateImageKHR` for zero-copy GPU display. [Source](https://libcamera.org/api-html/classlibcamera_1_1FrameBuffer.html)

### 8.2 IPA Modules and Sandboxing

Image Processing Algorithms (IPAs) implement the 3A (auto-exposure, auto-focus, auto-white-balance) and other per-frame computations. They receive ISP statistics (histogram, AWB stats, focus stats) from the ISP hardware and return per-frame control values (exposure time, analogue gain, colour correction matrix) back to the pipeline handler.

For security isolation, IPAs run **out-of-process** with Google Mojo IPC [Source](https://gitlab.freedesktop.org/libcamera/libcamera/-/tree/main/src/ipa). The IPA interface is defined in `.mojom` IDL files, for example `include/libcamera/ipa/raspberrypi.mojom`, which generates both in-process and IPC stubs. The IPA receives three buffer categories:

1. **Bayer (raw sensor data)** — passed by reference (DMABUF fd), not copied
2. **Embedded data** — sensor metadata stream (exposure/gain register readback)
3. **Statistics** — ISP-generated per-frame stats (AWB, histogram, face detection boxes)

The IPA returns control updates via Mojo callbacks (`setIspControls()`, `metadataReady()`), which the pipeline handler applies via `VIDIOC_S_EXT_CTRLS` to the ISP subdev before streaming the next frame.

The Raspberry Pi 5 IPA is at `src/ipa/rpi/pisp/pisp.cpp`. [Source](https://gitlab.freedesktop.org/libcamera/libcamera/-/blob/main/src/ipa/rpi/pisp/pisp.cpp)

### 8.3 V4L2-Compat Library for Legacy Applications

libcamera ships `src/v4l2/v4l2_camera_proxy.cpp`, an LD_PRELOAD shim that intercepts V4L2 ioctls from legacy applications and redirects them through libcamera. This allows applications written for simple `/dev/videoN` UVC cameras to transparently use complex ISP-equipped camera pipelines:

```bash
LD_PRELOAD=/usr/lib/libcamera/v4l2-compat.so ffmpeg -f v4l2 -i /dev/video0 out.mp4
```

The shim translates `VIDIOC_QUERYCAP`, `VIDIOC_S_FMT`, `VIDIOC_REQBUFS`, `VIDIOC_QBUF`, and `VIDIOC_DQBUF` into the libcamera Request lifecycle. [Source](https://gitlab.freedesktop.org/libcamera/libcamera/-/blob/main/src/v4l2/v4l2_camera_proxy.cpp)

---

## 9. DMA-BUF Bridge to the GPU

A V4L2 capture device produces frames in device memory. Moving those frames to the GPU for display, post-processing, or ML inference without a CPU copy requires importing the V4L2 DMABUF fd into the GPU API.

### 9.1 EGL Import

The `EGL_EXT_image_dma_buf_import` extension (required by all Mesa EGL implementations) allows creating an `EGLImage` directly from a DMABUF fd:

```c
/* After VIDIOC_EXPBUF yields dmabuf_fd for an NV12 frame */
EGLint attribs[] = {
    EGL_LINUX_DMA_BUF_EXT,          (EGLint)NULL,
    EGL_WIDTH,                       width,
    EGL_HEIGHT,                      height,
    EGL_LINUX_DRM_FOURCC_EXT,       DRM_FORMAT_NV12,
    /* Plane 0 (luma) */
    EGL_DMA_BUF_PLANE0_FD_EXT,     dmabuf_fd,
    EGL_DMA_BUF_PLANE0_OFFSET_EXT, 0,
    EGL_DMA_BUF_PLANE0_PITCH_EXT,  stride,
    /* Plane 1 (interleaved chroma) */
    EGL_DMA_BUF_PLANE1_FD_EXT,     dmabuf_fd,
    EGL_DMA_BUF_PLANE1_OFFSET_EXT, y_size,
    EGL_DMA_BUF_PLANE1_PITCH_EXT,  stride,
    EGL_NONE,
};
EGLImageKHR image = eglCreateImageKHR(dpy, EGL_NO_CONTEXT,
                                       EGL_LINUX_DMA_BUF_EXT, NULL, attribs);
/* EGL does NOT take ownership of dmabuf_fd — caller closes it */
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, image);
```

For tiled or hardware-compressed frame formats (common on mobile SoCs and the Raspberry Pi 5's PiSP compressed Bayer modes), `EGL_EXT_image_dma_buf_import_modifiers` is additionally required. It adds `EGL_DMA_BUF_PLANE0_MODIFIER_LO_EXT` / `_HI_EXT` attributes that pass the DRM format modifier to the EGL driver, which programs the GPU's texture sampler to handle the compression. [Source](https://registry.khronos.org/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import_modifiers.txt)

### 9.2 Vulkan Import

Vulkan imports DMABUF fds via `VK_EXT_external_memory_dma_buf` combined with `VK_KHR_external_memory_fd`:

```c
/* Chain VkExternalMemoryImageCreateInfo into VkImageCreateInfo.pNext */
VkExternalMemoryImageCreateInfo ext_img_info = {
    .sType       = VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
VkImageCreateInfo img_info = {
    .sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .pNext = &ext_img_info,
    /* format, extent, usage as normal */
};
vkCreateImage(device, &img_info, NULL, &image);

/* Import the DMABUF fd when allocating memory */
VkImportMemoryFdInfoKHR import_info = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    .fd         = dmabuf_fd,
};
VkMemoryAllocateInfo alloc_info = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
    .pNext = &import_info,
    .allocationSize  = required_size,
    .memoryTypeIndex = memory_type_index,
};
vkAllocateMemory(device, &alloc_info, NULL, &device_memory);
vkBindImageMemory(device, image, device_memory, 0);
```

For tiled/compressed layouts, `VK_EXT_image_drm_format_modifier` and `VkImageDrmFormatModifierExplicitCreateInfoEXT` are required, analogously to the EGL extension. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_external_memory_dma_buf.html)

### 9.3 OpenCL Import

OpenCL 3.0 with the `cl_khr_external_memory_dma_buf` extension (KHR provisional, vendor extensions exist earlier) allows importing a DMABUF fd as a `cl_mem` object. The import uses `clImportMemoryARM` (Mali) or the standard `clCreateBufferWithProperties` / `clCreateImageWithProperties` with `CL_IMPORT_TYPE_DMA_BUF_ARM`. Vendor support varies; on systems where both V4L2 and OpenCL run on the same SoC (common in automotive), this path avoids CPU copies in computer vision pipelines.

Note: the `cl_khr_external_memory_dma_buf` standard extension is still being ratified across all implementations as of kernel 6.x. Verify extension string availability at runtime with `clGetPlatformInfo(CL_PLATFORM_EXTENSIONS)` before use.

---

## 10. V4L2 Codec Devices: Stateful vs. Stateless

V4L2 codec devices are M2M devices: compressed data enters the OUTPUT queue, decoded frames exit the CAPTURE queue (or vice versa for encoders). Two distinct programming models exist, reflecting different hardware architectures.

### 10.1 Stateful Codecs

Stateful codecs (e.g., Hantro G1/G2, Rockchip CODA, many SoC vendor codecs) contain an embedded bitstream parser. The driver maintains decoder state (SPS, PPS, DPB) internally:

**Pixel format on OUTPUT queue:** `V4L2_PIX_FMT_H264` (complete Annex B stream fragments)

**Resolution change sequence:**
1. Driver detects a new SPS/PPS in the stream and fires `V4L2_EVENT_SOURCE_CHANGE` with `V4L2_EVENT_SRC_CH_RESOLUTION`.
2. Userspace receives the event via `VIDIOC_DQEVENT` after polling with `POLLPRI`.
3. Userspace stops CAPTURE streaming (`VIDIOC_STREAMOFF(CAPTURE)`).
4. Userspace queries the new format via `VIDIOC_G_FMT(CAPTURE)`.
5. Userspace queries minimum DPB size via `VIDIOC_G_CTRL(V4L2_CID_MIN_BUFFERS_FOR_CAPTURE)`.
6. Userspace reallocates CAPTURE buffers and restarts streaming.

This sequence is driver-managed and does not require application-level bitstream parsing. [Source](https://docs.kernel.org/userspace-api/media/v4l/dev-decoder.html)

### 10.2 Stateless Codecs and the Request API

Stateless codecs (Allwinner Cedrus, Rockchip RKVDEC, Mediatek VDEC, Raspberry Pi PiSP) offload parsing to userspace. The application must supply all decode parameters per-frame via V4L2 extended controls attached to a media request.

**Pixel format on OUTPUT queue:** `V4L2_PIX_FMT_H264_SLICE` (raw slice NAL units, no stream headers)

**Required capability:** `V4L2_BUF_CAP_SUPPORTS_REQUESTS` must be set on the OUTPUT queue (queried via `VIDIOC_QUERY_EXT_CTRL`).

Key stateless H.264 control IDs (in `include/uapi/linux/v4l2-controls.h`, stabilised ~kernel 5.10 and moved from staging):

```c
V4L2_CID_STATELESS_H264_SPS          /* struct v4l2_ctrl_h264_sps */
V4L2_CID_STATELESS_H264_PPS          /* struct v4l2_ctrl_h264_pps */
V4L2_CID_STATELESS_H264_SLICE_PARAMS /* struct v4l2_ctrl_h264_slice_params */
V4L2_CID_STATELESS_H264_DECODE_PARAMS/* struct v4l2_ctrl_h264_decode_params (DPB) */
V4L2_CID_STATELESS_H264_PRED_WEIGHTS /* struct v4l2_ctrl_h264_pred_weights */
```

These use the control class `V4L2_CTRL_CLASS_CODEC_STATELESS` (0x00a40000). Analogous constants exist for HEVC (`V4L2_CID_STATELESS_HEVC_*`), VP9 (`V4L2_CID_STATELESS_VP9_FRAME`), and AV1 (`V4L2_CID_STATELESS_AV1_SEQUENCE`, `V4L2_CID_STATELESS_AV1_TILE_GROUP_ENTRY`). [Source](https://docs.kernel.org/userspace-api/media/v4l/ext-ctrls-codec-stateless.html)

---

## 11. The Request API In Depth

### 11.1 Allocation, Queuing, and Polling

The request API, merged in kernel 5.2, provides a mechanism for delivering a group of V4L2 controls and buffer metadata atomically to the hardware for a single frame decode. [Source](https://docs.kernel.org/userspace-api/media/mediactl/request-api.html)

```c
/* 1. Allocate a request — returns a file descriptor */
int req_fd;
ioctl(media_fd, MEDIA_IOC_REQUEST_ALLOC, &req_fd);

/* 2. Queue the compressed slice to the OUTPUT queue, attached to this request */
struct v4l2_buffer buf = {
    .type       = V4L2_BUF_TYPE_VIDEO_OUTPUT,
    .memory     = V4L2_MEMORY_MMAP,
    .index      = output_buf_index,
    .request_fd = req_fd,
    .flags      = V4L2_BUF_FLAG_REQUEST_FD,
};
/* Fill buf.m.offset with compressed slice data first */
ioctl(v4l2_fd, VIDIOC_QBUF, &buf);
/* Buffer is now in VB2_BUF_STATE_IN_REQUEST — not yet queued to hardware */

/* 3. Set per-frame decode parameters, attached to the same request */
struct v4l2_ext_controls ctrls = {
    .which      = V4L2_CTRL_WHICH_REQUEST_VAL,
    .request_fd = req_fd,
    .count      = num_ctrls,
    .controls   = ctrl_array,   /* V4L2_CID_STATELESS_H264_SPS etc. */
};
ioctl(v4l2_fd, VIDIOC_S_EXT_CTRLS, &ctrls);

/* 4. Queue a CAPTURE buffer (not request-attached — just needs to be available) */
struct v4l2_buffer cap_buf = {
    .type   = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .memory = V4L2_MEMORY_MMAP,
    .index  = capture_buf_index,
};
ioctl(v4l2_fd, VIDIOC_QBUF, &cap_buf);

/* 5. Submit the request to hardware */
ioctl(req_fd, MEDIA_REQUEST_IOC_QUEUE);
/* Both the OUTPUT buffer and controls are now committed atomically */

/* 6. Poll the request fd for completion */
struct pollfd pfd = { .fd = req_fd, .events = POLLIN };
poll(&pfd, 1, -1);

/* 7. Dequeue the decoded frame */
struct v4l2_buffer dqbuf = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
                              .memory = V4L2_MEMORY_MMAP };
ioctl(v4l2_fd, VIDIOC_DQBUF, &dqbuf);

/* 8. Reinitialise request for reuse, or close it */
ioctl(req_fd, MEDIA_REQUEST_IOC_REINIT);   /* or close(req_fd) */
```

Multi-slice frames (common in H.264) use `V4L2_BUF_FLAG_M2M_HOLD_CAPTURE_BUF` on the OUTPUT buffer for all slices except the last, signalling the driver to accumulate slices before releasing the CAPTURE buffer. [Source](https://docs.kernel.org/userspace-api/media/v4l/buffer.html)

### 11.2 Synchronisation: Fences vs. Request API

A patch series proposing explicit per-buffer IN_FENCE/OUT_FENCE support for V4L2 (using `sync_file` fds, similar to DRM's fence mechanism) was proposed by Collabora in approximately 2018 (revision v7, author Gustavo Padovan). This series would have added `V4L2_BUF_FLAG_IN_FENCE` and `V4L2_BUF_FLAG_OUT_FENCE` fields to `struct v4l2_buffer`. As of the current kernel (6.x), **this patch series was not accepted into mainline**. There is no upstream V4L2 per-buffer fence mechanism.

The request API (Section 11.1) is the only stable per-frame synchronisation mechanism. It provides atomicity across controls and buffers for a single decode unit, and request-fd polling provides completion notification. Applications requiring tighter integration with DRM/Wayland fence timelines must manage the boundary between V4L2 completion (via request-fd poll) and GPU work submission manually — for example, by using a semaphore or by staging through a CPU-signalled fence.

Note: Some downstream kernels (Android, automotive SoC vendors) carry variants of the fence patch series as out-of-tree modifications. These are not part of the upstream V4L2 ABI.

---

## 12. GStreamer V4L2 Elements

### 12.1 v4l2src, v4l2sink, v4l2convert

GStreamer's `gst-plugins-good` provides `v4l2src` (capture), `v4l2sink` (output/display), and `v4l2video0convert` (M2M format/scale conversion). `gst-plugins-bad` provides codec-specific M2M elements: `v4l2h264dec`, `v4l2h264enc`, `v4l2hevcenc`, etc.

Key `v4l2src` properties:
- `device` — path to `/dev/videoN`
- `io-mode` — buffer sharing mode: `mmap` (default), `dmabuf` (export MMAP as DMABUF downstream), `dmabuf-import` (expect DMABUF from upstream)
- `num-buffers` — capture buffer count

The `io-mode=dmabuf` property causes `v4l2src` to call `VIDIOC_EXPBUF` on its MMAP buffers and advertise them downstream with `memory:DMABuf` caps. [Source](https://gstreamer.freedesktop.org/documentation/video4linux2/v4l2src.html)

### 12.2 Zero-Copy DMA-BUF Pipeline

A complete zero-copy encode pipeline using V4L2 M2M:

```bash
# Zero-copy: camera → V4L2 H.264 encoder → file
gst-launch-1.0 \
  v4l2src io-mode=dmabuf ! \
  video/x-raw,format=NV12,width=1920,height=1080 ! \
  v4l2h264enc output-io-mode=dmabuf-import ! \
  video/x-h264,level='(string)4' ! \
  h264parse ! \
  mp4mux ! \
  filesink location=output.mp4
```

The `io-mode=dmabuf` on the source and `output-io-mode=dmabuf-import` on the encoder ensure buffers flow from the camera driver to the encoder driver without a CPU copy. The caps filter `video/x-raw,format=NV12` ensures the source and encoder agree on the frame format without inserting a software converter.

**Performance caveat:** Inserting a `v4l2convert` element between source and encoder to perform colour space or resolution conversion invokes the M2M ISP device, which has per-frame submission overhead. Depending on the pipeline, this can reduce throughput significantly (measured drops from ~50 fps to ~24 fps on some embedded SoCs). Prefer hardware-accelerated conversion inside the encoder or ISP where possible.

#### Stateless Decode via GStreamer

GStreamer's `gst-plugins-bad` provides `v4l2slh264dec` for stateless H.264 decode:

```bash
# Stateless H.264 decode using V4L2 request API (Raspberry Pi 4/5)
gst-launch-1.0 \
  filesrc location=input.h264 ! \
  h264parse ! \
  v4l2slh264dec ! \
  video/x-raw,format=NV12 ! \
  kmssink
```

The `v4l2slh264dec` element handles the request API lifecycle internally — it parses SPS/PPS from the bytestream, allocates a request per frame, populates `V4L2_CID_STATELESS_H264_*` controls, and calls `MEDIA_REQUEST_IOC_QUEUE`. The `kmssink` element performs direct KMS scanout from the NV12 DMABUF frames without any CPU copy. [Source](https://gstreamer.freedesktop.org/documentation/video4linux2/v4l2slh264dec.html)

For GPU-accelerated display of V4L2 camera output, the DMABUF flow continues into a Wayland compositor:

```bash
gst-launch-1.0 \
  v4l2src io-mode=dmabuf ! \
  video/x-raw,format=NV12,width=1920,height=1080 ! \
  glimagesink
```

`glimagesink` uses `eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)` internally (Section 9.1) to import the V4L2 DMABUF into an OpenGL texture for display. [Source](https://gstreamer.freedesktop.org/documentation/opengl/glimagesink.html)

---

## 13. Raspberry Pi CSI Pipeline

The Raspberry Pi family provides the most complete open-source reference for the full V4L2/libcamera/ISP pipeline, with all components in mainline Linux and the libcamera project.

### 13.1 Pi 4: bcm2835-unicam and the VC4 ISP

On the Raspberry Pi 4 (BCM2835 SoC):

- **CSI-2 receiver:** `drivers/media/platform/bcm2835/bcm2835-unicam.c` — a simple capture-only V4L2 driver. Unicam receives the raw Bayer or YUV data from the sensor and writes it to system memory via DMA. It does not perform any image processing.
- **ISP:** The BCM2835 VideoCore IV firmware ISP is exposed as a V4L2 M2M device (`/dev/video12` and `/dev/video13` for the ISP statistics/parameters, plus `/dev/video13` for the ISP output, depending on the distribution). This device is managed by the proprietary VC4 firmware via the MMAL interface in the downstream kernel, but libcamera's vc4 pipeline handler accesses it through a V4L2 M2M interface provided by `bcm2835-isp` driver — also in the downstream RPi kernel tree, but increasingly in mainline as of recent 6.x releases.
- **libcamera pipeline:** `src/libcamera/pipeline/rpi/vc4/vc4.cpp` handles link setup, format negotiation, and ISP M2M buffer chaining.

The Pi 4 pipeline with libcamera:

```
[imx219 subdev] --Bayer10--> [unicam capture] --DMABUF--> [bcm2835-isp M2M] --NV12--> [display]
                              raw /dev/video0              /dev/video13
```

### 13.2 Pi 5: rp1-cfe and the PiSP Back End

The Raspberry Pi 5 uses the RP1 south bridge chip, which contains the PiSP (Raspberry Pi Image Signal Processor), a two-stage ISP:

**Stage 1: rp1-cfe (CSI-2 Front End + simple processing)**
- Driver: `drivers/media/platform/raspberrypi/rp1-cfe/cfe.c`
- Merged approximately kernel 6.12–6.13 [Source](https://docs.kernel.org/admin-guide/media/raspberrypi-rp1-cfe.html)
- Exposes 8 video nodes:
  - `csi2-ch0` through `csi2-ch3` — raw CSI-2 lane DMA outputs
  - `fe-image0`, `fe-image1` — processed image outputs from the front-end
  - `fe-stats` — AWB/AE statistics output (consumed by the IPA)
  - `fe-config` — front-end configuration input (written by libcamera/IPA)
- Supports up to four simultaneous CSI-2 data stream channels

**Stage 2: PiSP Back End (M2M ISP)**
- Driver merged in **kernel 6.11** [Source: `drivers/media/platform/raspberrypi/pisp/`]
- A V4L2 M2M device capable of 4K@60 fps processing
- Implements: demosaicing (Bayer → RGB), lens shading correction, colour correction matrix, tone mapping, noise reduction, resizing, format conversion
- Note: the exact in-tree path (`drivers/media/platform/raspberrypi/pisp/`) should be verified against the 6.11 kernel source, as naming may vary.

The complete Pi 5 pipeline:

```
[imx219] --MIPI CSI-2--> [rp1-cfe front-end]
                                |
              +----------+------+--------+
              |          |              |
          fe-image0   fe-stats      fe-config
              |          |
              v          v (to IPA via libcamera)
     [PiSP back-end M2M ISP]
              |
           NV12/RGB output
              |
          [display / encoder / DMABUF consumer]
```

libcamera manages this entire chain: it configures the rp1-cfe links via `MEDIA_IOC_SETUP_LINK`, negotiates formats on each subdev pad, feeds `fe-config` from the IPA, routes `fe-stats` to the IPA for 3A computation, and drives the PiSP back-end M2M queue. [Source](https://gitlab.freedesktop.org/libcamera/libcamera/-/blob/main/src/libcamera/pipeline/rpi/pisp/pisp.cpp)

### 13.3 libcamera Pipeline Handlers for Raspberry Pi

The Pi-specific libcamera code lives in `src/libcamera/pipeline/rpi/`. A common base layer in `src/libcamera/pipeline/rpi/common/pipeline_base.cpp` handles shared RPi logic; `vc4/vc4.cpp` and `pisp/pisp.cpp` add SoC-specific ISP driving. The IPA interface uses the Mojo IDL at `include/libcamera/ipa/raspberrypi.mojom`.

Basic libcamera capture from the command line (Pi 5):

```bash
# List cameras
libcamera-hello --list-cameras

# Capture a still at full resolution
libcamera-still -o image.jpg --width 4608 --height 2592

# Low-latency preview (routes through rp1-cfe + PiSP back-end)
libcamera-vid -t 0 --codec yuv420 -o - | ffplay -
```

DMA-BUF chain from sensor to display: libcamera's `FrameBuffer` objects are backed by DMABUF fds (allocated by the pipeline handler from the V4L2 MMAP pool of each device and exported via `VIDIOC_EXPBUF`). When the application calls `Request::addBuffer()`, it provides a `FrameBuffer` whose DMABUF fd the pipeline handler queues to the ISP CAPTURE queue. The display path (KMS direct scanout or Wayland compositor) imports those same DMABUF fds via `linux-dmabuf-unstable-v1` — no CPU copy occurs anywhere in the chain.

---

## 14. Integrations

**Chapter 26 (Hardware Video: VA-API, V4L2, GStreamer):** Chapter 26 surveys VA-API alongside V4L2 at an API overview level. The two subsystems share the DMABUF fd as the zero-copy interop primitive. VA-API surfaces are exported as DMABUF fds via `vaExportSurfaceHandle(dpy, surface, VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2, ...)`. V4L2 surfaces are exported via `VIDIOC_EXPBUF`. Both fds can be imported into the same `eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)` call (Section 9.1). A hardware decode pipeline can combine the two: a V4L2 stateless decoder (Ch142) produces NV12 DMABUF frames that a VA-API post-processor (Ch26) ingests for deinterlacing or colour conversion without crossing a CPU boundary.

**Chapter 38 (PipeWire and the Video Session Layer):** PipeWire acts as the session-level camera broker on modern Linux desktops. Two paths exist for V4L2 devices:
1. **libcamera source node** — PipeWire has a native libcamera integration that routes libcamera-managed cameras as PipeWire graph nodes. Applications that open the PipeWire camera portal receive DMABUF-backed `SPA_DATA_DmaBuf` buffers originating from the libcamera pipeline handler described in Section 8.
2. **pw-v4l2 shim** — The `libpw-v4l2.so` LD_PRELOAD library translates V4L2 ioctls to PipeWire API calls, enabling legacy V4L2 applications to work through PipeWire's routing and permission layer without modification:
   ```bash
   pw-v4l2 -- ffmpeg -f v4l2 -i /dev/video0 output.mp4
   ```
   [Source](https://docs.pipewire.org/page_v4l2.html)

**Chapter 57 (FFmpeg Architecture and Programming):** FFmpeg exposes V4L2 M2M codecs as `AVCodec` entries with names like `h264_v4l2m2m`, `hevc_v4l2m2m`, `vp8_v4l2m2m`, and `vp9_v4l2m2m`, implemented in `libavcodec/v4l2_m2m.c`. For zero-copy decode output, FFmpeg uses `AV_PIX_FMT_DRM_PRIME` (`AVHWFramesContext` with the DRM PRIME hardware device type) to expose decoded frames as DMABUF fds directly to downstream filters or muxers, exactly matching the V4L2 EXPBUF pattern described in Section 5.3. CLI usage:
   ```bash
   ffmpeg -hwaccel drm -hwaccel_device /dev/dri/renderD128 \
          -c:v h264_v4l2m2m -i input.h264 output.mkv
   ```

**Chapter 99 (Automotive and Embedded Linux Graphics):** V4L2 is the standard camera capture interface on Linux-based automotive SoCs (TI TDA4VM, NXP i.MX 8, Qualcomm SA8195P). Multi-camera ADAS sensor fusion pipelines use the media controller topology to model the physical hardware: camera sensors connect through GMSL or FPD-Link deserializers (modelled as subdevs) to a CSI-2 receiver, then to an on-chip ISP, then to memory. The same DMABUF chain described in this chapter flows from V4L2 capture into OpenCL or Vulkan compute for lane detection, object recognition, and occupancy grid updates. The Vulkan SC profile (discussed in Chapter 99) additionally constrains how driver memory allocation and pipeline management must behave in safety-critical deployments, but the V4L2 capture path and DMABUF import into Vulkan remain architecturally identical to the development-focused flows described here.

---

## 15. References

- [V4L2 API Reference](https://docs.kernel.org/userspace-api/media/v4l/) — Primary UAPI documentation
- [Media Controller Core API](https://docs.kernel.org/driver-api/media/mc-core.html) — Entity, pad, link model
- [Request API Documentation](https://docs.kernel.org/userspace-api/media/mediactl/request-api.html) — Stateless codec request lifecycle
- [Stateless Codec Controls](https://docs.kernel.org/userspace-api/media/v4l/ext-ctrls-codec-stateless.html) — H.264/HEVC/VP9/AV1 control IDs
- [Stateful Decoder API](https://docs.kernel.org/userspace-api/media/v4l/dev-decoder.html) — Resolution change sequence and DPB management
- [V4L2 Buffer Documentation](https://docs.kernel.org/userspace-api/media/v4l/buffer.html) — `v4l2_buffer` flags including REQUEST_FD and M2M_HOLD_CAPTURE_BUF
- [videobuf2 Framework](https://www.kernel.org/doc/html/v6.2/driver-api/media/v4l2-videobuf2.html) — vb2 state machine and allocator backends
- [rp1-cfe Driver](https://docs.kernel.org/admin-guide/media/raspberrypi-rp1-cfe.html) — Raspberry Pi 5 CSI-2 front-end
- [IMX219 driver source](https://elixir.bootlin.com/linux/latest/source/drivers/media/i2c/imx219.c) — Reference sensor subdev implementation
- [libcamera project](https://libcamera.org/) — Pipeline handlers, IPA modules, V4L2-compat
- [GStreamer V4L2 elements](https://gstreamer.freedesktop.org/documentation/video4linux2/) — v4l2src, v4l2sink, io-mode documentation
- [EGL_EXT_image_dma_buf_import_modifiers](https://registry.khronos.org/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import_modifiers.txt) — DRM format modifier support for EGL image import
- [VK_EXT_external_memory_dma_buf](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_external_memory_dma_buf.html) — Vulkan DMABUF import extension

## Roadmap

### Near-term (6–12 months)
- Stabilisation of AV1 stateless decode controls (`V4L2_CID_STATELESS_AV1_*`) across drivers including Hantro and Rockchip RKVDEC2, following the AV1 control series merged in kernel 6.8–6.9 that moved the controls out of staging.
- The V4L2 subdev streams API (multi-stream multiplexing over a single CSI-2 link, introduced in kernel 6.3) is gaining adoption in new sensor and bridge drivers; expect the streams API to become the required implementation path for multi-pad subdevs replacing the legacy `.s_stream` callback.
- libcamera is expanding its pipeline handler coverage for Intel IPU6 (Meteor Lake) and Qualcomm Camss platforms, with the IPU6 handler expected to reach production quality in the 0.4–0.5 release cycle.
- The rp1-cfe driver (merged in kernel 6.12) and the PiSP back-end M2M ISP are receiving ongoing bug fixes and performance work upstream, with DMA-BUF fence integration between the two pipeline stages under active review.

### Medium-term (1–3 years)
- A V4L2 per-buffer explicit synchronisation mechanism using `sync_file` fds (the long-pending "fences for V4L2" series) is expected to be revived and merged now that the request API is stable; this would enable V4L2 buffers to carry in-fences from GPU work and out-fences consumed by KMS or Wayland compositors, eliminating the current polling gap.
- libcamera's IPA sandboxing model is evolving toward Seccomp-based process isolation with a defined IPA ABI, aiming to allow vendor-supplied binary IPA modules to run safely alongside open-source pipeline handlers without requiring kernel changes.
- The `V4L2_MEMORY_DMABUF` import path is expected to gain support for heap-allocated DMABUF fds from the DMA-heap framework (`/dev/dma_heap/`) as the preferred allocator over ION, enabling a unified allocation path shared with display and GPU subsystems.
- GStreamer's `v4l2codecs` plugin (implementing stateless H.264, HEVC, VP9, and AV1 decode via the request API) is expected to graduate from `gst-plugins-bad` to `gst-plugins-good` as driver and API coverage matures.

### Long-term
- The media controller topology model may be extended to express compute and ML inference nodes (ISP-attached NPUs on automotive SoCs) as first-class media entities, enabling pipeline graphs that mix capture, ISP, and on-chip inference without leaving the V4L2/media controller programming model.
- MIPI CSI-3 and next-generation camera serialiser standards (GMSL3, FPD-Link IV) will require corresponding kernel driver infrastructure; the V4L2 async notifier and subdev routing models are the designated extension points, but multi-hop serialiser topologies may require new link type extensions to the media controller graph.
- Long-term unification of V4L2 stateless codec controls with Vulkan Video decode parameters (which carry structurally identical SPS/PPS/slice metadata) could enable a common userspace parsing layer that feeds either V4L2 or Vulkan Video hardware paths without codec-level duplication.
