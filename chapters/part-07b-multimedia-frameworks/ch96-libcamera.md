# Chapter 96: libcamera and the Linux Camera Stack

**Part VII — Application APIs and Middleware**

**Audiences targeted**: Embedded Linux developers building camera-enabled products; video conferencing application developers who need to understand the path from sensor to WebRTC; robotics and computer vision engineers consuming camera frames in real time; IoT developers integrating ISP-equipped SoCs.

The Linux camera subsystem has historically been one of the most fragmented areas of the platform. Unlike audio — where ALSA provided a single kernel ABI and PipeWire a unified userspace session layer — cameras on Linux accumulated a decade of SoC-specific proprietary HALs, divergent V4L2 extension usage, and undocumented ISP firmware blobs. libcamera, started in 2019 by the kernel community, solved this by moving complex ISP control from firmware into auditable userspace, providing a single C++ API above a per-platform *PipelineHandler* layer and a sandboxed *Image Processing Algorithms* (IPA) module system. Today libcamera underpins camera access on Raspberry Pi 4 and 5, Intel laptops with CIO2-attached sensors, Rockchip RK3399/RK3588 SBCs, and a wide range of USB webcams — all exposed through PipeWire's camera portal so that WebRTC, GStreamer, and OBS acquire frames through a single security-conscious gate. [Source](https://libcamera.org/)

---

## Table of Contents

1. [Introduction: The Camera Fragmentation Problem](#1-introduction-the-camera-fragmentation-problem)
2. [V4L2 Media Controller Framework](#2-v4l2-media-controller-framework)
3. [libcamera Architecture](#3-libcamera-architecture)
4. [PipelineHandler: Per-Platform Subclasses](#4-pipelinehandler-per-platform-subclasses)
5. [IPA: Image Processing Algorithms](#5-ipa-image-processing-algorithms)
6. [FrameBuffer and DMA-BUF Integration](#6-framebuffer-and-dma-buf-integration)
7. [libcamera-apps and rpicam](#7-libcamera-apps-and-rpicam)
8. [PipeWire Camera Integration](#8-pipewire-camera-integration)
9. [GStreamer Integration](#9-gstreamer-integration)
10. [Platform Coverage and Hardware](#10-platform-coverage-and-hardware)
11. [Integrations](#integrations)

---

## 1. Introduction: The Camera Fragmentation Problem

### 1.1 The Pre-libcamera Landscape

Before libcamera, camera support on Linux fragmented across three incompatible strategies. **USB webcams** used the UVC class driver (`uvcvideo`) and spoke simple V4L2 video node ioctls: one `open("/dev/video0")`, `VIDIOC_S_FMT`, `VIDIOC_REQBUFS`, `VIDIOC_STREAMON`, done. The ISP was inside the camera's firmware and Linux saw only processed YUYV or MJPEG output. **Android-derived SoC cameras** relied on proprietary camera HALs — closed-source shared libraries compiled against Android's camera HAL3 interface, running GPU blobs and firmware co-processors in ways utterly opaque to the host kernel. Porting these to desktop Linux required bridging the Android HAL to a V4L2 device node, which most vendors never attempted. **Embedded Linux BSPs** exposed raw sensor data via V4L2 but required a sequence of `media-ctl` topology manipulation and `v4l2-ctl` subdevice format settings that varied per SoC and were almost always scripted by hand. Any application wanting to drive the ISP (auto-exposure, white balance, lens shading correction) had to integrate the vendor's proprietary tuning library.

V4L2 alone was structurally insufficient for this problem. V4L2 video nodes model a simple capture device: set format, allocate buffers, stream. But modern SoC camera pipelines are *graphs* — a sensor subdevice feeds a CSI-2 DPHY block, which feeds a receiver (MIPI CSI-2 receiver, CIO2, Unicam), which feeds an ISP with multiple input and output pads, which outputs to memory. Configuring every edge in that graph, negotiating pixel formats and crop rectangles at every pad, and translating ISP statistics into exposure and gain adjustments requires the Media Controller API and a software ISP control layer that V4L2 video nodes never provided.

### 1.2 libcamera as the Unified Solution

libcamera addresses the fragmentation at three levels. [Source](https://libcamera.org/faq.html)

**Kernel level** — the existing V4L2 and Media Controller APIs are used unchanged. No new kernel ABIs are required. libcamera consumes `MEDIA_IOC_ENUM_ENTITIES`, `MEDIA_IOC_SETUP_LINK`, `VIDIOC_SUBDEV_S_FMT`, `VIDIOC_SUBDEV_S_SELECTION`, and the V4L2 buffer streaming ioctls directly.

**Userspace PipelineHandler layer** — each SoC platform has a `PipelineHandler` subclass that knows how to discover the media graph, configure all subdevices in the right order, and translate a libcamera `CameraConfiguration` into the correct V4L2 calls. PipelineHandlers ship in the libcamera tree and are maintained upstream.

**IPA (Image Processing Algorithm) sandbox** — 3A algorithms (AEC/AGC, AWB, LSC, CCM, gamma) run in a sandboxed subprocess or shared library that receives ISP statistics from the pipeline handler and returns control parameters. IPA modules may be open-source (loaded in-process) or proprietary (forcibly isolated in a separate process via Unix socket IPC). [Source](https://libcamera.org/api-html/classlibcamera_1_1IPAManager.html)

The full architecture stack looks like this:

```
Application (C++, Python, GStreamer, PipeWire)
        │
        ▼
libcamera C++ API (CameraManager, Camera, Request)
        │
   PipelineHandler (RkISP1, IPU3, RPi/PiSP, Simple, …)
        │                        │
        ▼                        ▼
  Media Controller         IPA subprocess / lib
  V4L2 subdev API          (AGC, AWB, LSC, CCM)
        │
        ▼
  Kernel: rkisp1, ipu3-cio2/imgu, rp1-cfe, uvcvideo, vimc
        │
        ▼
  Camera Sensor (OV5647, IMX219, IMX477, IMX708, …)
```

---

## 2. V4L2 Media Controller Framework

### 2.1 The Media Graph Model

The Media Controller API, introduced in Linux 2.6.38 ([Documentation](https://docs.kernel.org/userspace-api/media/mediactl/media-controller.html)), represents a camera pipeline as a directed graph of **entities**, **pads**, and **links**. An entity (`struct media_entity`) is any hardware block: an image sensor, a CSI-2 receiver, an ISP input stage, an ISP output, or a memory-mapped video node. Entities expose **pads** — directional endpoints typed as `MEDIA_PAD_FL_SOURCE` or `MEDIA_PAD_FL_SINK`. A **link** connects a source pad on one entity to a sink pad on another.

```c
/* include/uapi/linux/media.h */
struct media_entity_desc {
    __u32        id;
    char         name[32];
    __u32        type;        /* MEDIA_ENT_T_V4L2_SUBDEV, MEDIA_ENT_T_DEVNODE… */
    __u32        flags;
    union {
        struct { __u32 major; __u32 minor; } dev; /* character device */
        struct { __u32 index; }               v4l;
    };
};

struct media_link_desc {
    struct media_pad_desc source;
    struct media_pad_desc sink;
    __u32                 flags;  /* MEDIA_LNK_FL_ENABLED, MEDIA_LNK_FL_IMMUTABLE */
};
```

Applications enumerate the graph via `MEDIA_IOC_ENUM_ENTITIES` and `MEDIA_IOC_ENUM_LINKS`, or use the newer atomic `MEDIA_IOC_G_TOPOLOGY` which returns the full graph in one call. Links are enabled or disabled with `MEDIA_IOC_SETUP_LINK`:

```c
int fd = open("/dev/media0", O_RDWR);

/* Enumerate all entities */
struct media_entity_desc desc = { .id = MEDIA_ENT_ID_FLAG_NEXT };
while (ioctl(fd, MEDIA_IOC_ENUM_ENTITIES, &desc) == 0) {
    printf("Entity %u: %s (type 0x%x)\n", desc.id, desc.name, desc.type);
    desc.id |= MEDIA_ENT_ID_FLAG_NEXT;
}

/* Enable a link: sensor pad 0 → CSI-2 receiver pad 0 */
struct media_link_desc link = {
    .source = { .entity = sensor_id,  .index = 0, .flags = MEDIA_PAD_FL_SOURCE },
    .sink   = { .entity = csi2rx_id,  .index = 0, .flags = MEDIA_PAD_FL_SINK   },
    .flags  = MEDIA_LNK_FL_ENABLED,
};
ioctl(fd, MEDIA_IOC_SETUP_LINK, &link);
```

### 2.2 V4L2 Subdevice API

Once links are enabled, each entity's format and geometry is configured through the V4L2 subdevice API on the character device at `/dev/v4l-subdev*`. The key ioctls are:

- `VIDIOC_SUBDEV_S_FMT` / `VIDIOC_SUBDEV_G_FMT` — set or get the media bus pixel format, width, height, and field order on a specific pad.
- `VIDIOC_SUBDEV_S_SELECTION` / `VIDIOC_SUBDEV_G_SELECTION` — set crop (`MEDIA_BUS_FMT_*`) or compose rectangles on a pad, used to select a region of interest from the sensor.
- `VIDIOC_SUBDEV_ENUM_MBUS_CODE` — enumerate supported media bus codes on a pad.
- `VIDIOC_SUBDEV_ENUM_FRAME_SIZE` — enumerate discrete or stepwise frame sizes per bus code.

Format must be propagated from source to sink, matching at every pad. A sensor outputting `MEDIA_BUS_FMT_SRGGB10_1X10` at 4056×3040 must find a downstream receiver that accepts that bus code; the receiver then outputs to the ISP's input pad, and the ISP negotiates its own output format independently.

```c
/* Configure sensor pad 0: RGGB10, 1920×1080 */
struct v4l2_subdev_format fmt = {
    .which = V4L2_SUBDEV_FORMAT_ACTIVE,
    .pad   = 0,
    .format = {
        .code   = MEDIA_BUS_FMT_SRGGB10_1X10,
        .width  = 1920,
        .height = 1080,
        .field  = V4L2_FIELD_NONE,
    },
};
ioctl(subdev_fd, VIDIOC_SUBDEV_S_FMT, &fmt);
```

### 2.3 Video Nodes vs. Subdevices

A crucial distinction: **V4L2 video nodes** (`/dev/video*`) handle streaming — `VIDIOC_REQBUFS`, `VIDIOC_QBUF`, `VIDIOC_DQBUF`, `VIDIOC_STREAMON`. They carry frame data. **V4L2 subdevice nodes** (`/dev/v4l-subdev*`) handle *configuration* — formats, selections, controls. They carry no data themselves. A complete pipeline setup always involves subdevice configuration first, then video node streaming second.

The `media-ctl` utility (from `v4l-utils`) provides a command-line interface to the Media Controller API:

```bash
# Print full topology
media-ctl --device /dev/media0 --print-topology

# Set format on sensor pad 0 and ISP sink pad
media-ctl -d /dev/media0 \
    --set-v4l2 '"ov5647 1-0036":0 [fmt:SBGGR10_1X10/1920x1080]' \
    --set-v4l2 '"rkisp1_isp":0 [fmt:SBGGR10_1X10/1920x1080 crop:(0,0)/1920x1080]'
```

The kernel's `media_graph_walk` API provides an iterator over the graph for in-kernel users; libcamera implements an equivalent in userspace using `MediaGraph` and `MediaLink` objects in `src/libcamera/media_device.cpp`. [Source](https://github.com/libcamera-org/libcamera)

---

## 3. libcamera Architecture

### 3.1 CameraManager

`CameraManager` is a singleton that bootstraps the entire library. It enumerates all available camera devices by loading and running every registered `PipelineHandler::match()` in turn, and it exposes the set of discovered `Camera` instances. Hot-plug events from the kernel (via udev) trigger `cameraAdded` and `cameraRemoved` signals.

```cpp
#include <libcamera/libcamera.h>
using namespace libcamera;

CameraManager *cm = new CameraManager();
cm->start();   // Loads pipeline handlers, enumerates devices

// List cameras
for (auto const &camera : cm->cameras())
    std::cout << camera->id() << "\n";

// Acquire a camera
std::shared_ptr<Camera> cam = cm->get(cm->cameras()[0]->id());
cam->acquire();
```

[Source](https://libcamera.org/api-html/classlibcamera_1_1CameraManager.html)

### 3.2 Camera and CameraConfiguration

A `Camera` object represents one logical camera device (which may internally be a multi-component pipeline). Applications generate a `CameraConfiguration` by specifying one or more `StreamRole` hints:

```cpp
std::unique_ptr<CameraConfiguration> config =
    cam->generateConfiguration({ StreamRole::Viewfinder,
                                  StreamRole::StillCapture });

/* Adjust stream 0: viewfinder at 1280×720, NV12 */
StreamConfiguration &vfConfig = config->at(0);
vfConfig.pixelFormat = formats::NV12;
vfConfig.size        = { 1280, 720 };
vfConfig.bufferCount = 4;

/* Validate: pipeline handler may adjust parameters */
CameraConfiguration::Status status = config->validate();
if (status == CameraConfiguration::Adjusted)
    std::cerr << "Configuration adjusted by pipeline\n";

cam->configure(config.get());
```

`StreamRole` values include `Viewfinder` (continuous preview, may be lower resolution), `StillCapture` (full-resolution single shot), `VideoRecording` (continuous high-quality capture), and `Raw` (unprocessed Bayer data). The pipeline handler translates roles into specific hardware stream paths.

### 3.3 Stream, Request, and FrameBufferAllocator

A `Stream` is an output of the configured pipeline — one logical video stream with a fixed pixel format and size. Multiple streams may be active simultaneously if the hardware supports it (e.g., both a preview and a high-res still path on the RPi ISP).

A `Request` is the unit of capture. Each request carries one `FrameBuffer` per enabled `Stream`. The application creates requests, populates them with pre-allocated buffers, queues them, and receives completion via a signal:

```cpp
FrameBufferAllocator *allocator = new FrameBufferAllocator(cam);

for (StreamConfiguration &cfg : *config) {
    int ret = allocator->allocate(cfg.stream());
    assert(ret > 0);
}

Stream *stream = config->at(0).stream();
const std::vector<std::unique_ptr<FrameBuffer>> &buffers =
    allocator->buffers(stream);

/* Create one Request per buffer */
std::vector<std::unique_ptr<Request>> requests;
for (const auto &buf : buffers) {
    std::unique_ptr<Request> req = cam->createRequest();
    req->addBuffer(stream, buf.get());
    requests.push_back(std::move(req));
}

/* Connect completion signal */
cam->requestCompleted.connect(handleRequestCompleted);

/* Start streaming and queue requests */
cam->start();
for (auto &req : requests)
    cam->queueRequest(req.get());
```

The `requestCompleted` signal fires in a thread context managed by libcamera's internal `EventDispatcher`. The callback receives the completed `Request`, extracts metadata from `Request::metadata()` (exposure time, analogue gain, colour correction matrices as returned by the IPA), and re-queues the request for the next frame.

### 3.4 Event Model and Controls

libcamera uses a **signal/slot** event model (similar to Qt's, but with no Qt dependency). Signals are typed function objects; `connect()` binds a slot. The `ControlList` type carries key-value pairs of controls (exposure time, analogue gain, AWB mode, etc.) that the application attaches to a request:

```cpp
ControlList controls;
controls.set(controls::AeEnable, false);          /* disable auto-exposure */
controls.set(controls::ExposureTime, 20000);       /* 20 ms, in microseconds */
controls.set(controls::AnalogueGain, 2.0f);

std::unique_ptr<Request> req = cam->createRequest();
req->controls() = controls;
req->addBuffer(stream, buf.get());
cam->queueRequest(req.get());
```

Control IDs are defined in `include/libcamera/control_ids.yaml` and generated into `libcamera/control_ids.h` at build time. Vendor-specific controls appear in `vendor/` subdirectories and follow the same generation path.

---

## 4. PipelineHandler: Per-Platform Subclasses

### 4.1 Architecture

Every `PipelineHandler` subclass embodies the knowledge of one hardware platform. The base class provides common infrastructure: device acquisition via `DeviceEnumerator`, `MediaDevice` ownership, `Camera` registration, request queuing logic, and the `completeBuffer()` / `completeRequest()` chain. Subclasses implement a small set of virtual methods.

The pipeline handlers shipping in libcamera 0.5.x (mid-2025) are: [Source](https://github.com/libcamera-org/libcamera/tree/master/src/libcamera/pipeline)

| Handler | Hardware | Notes |
|---------|----------|-------|
| `ipu3` | Intel IPU3 (Skylake, Kaby Lake, Apollo Lake) | CIO2 + ImgU dual-component |
| `ipu6` | Intel IPU6 (Tiger Lake+) | Distinct ISP architecture |
| `rkisp1` | Rockchip RK3288, RK3399, RK3568, i.MX8MP | Upstream rkisp1 kernel driver |
| `rpi/vc4` | Raspberry Pi 0–4 (BCM283x) | Unicam + VC4 ISP |
| `rpi/pisp` | Raspberry Pi 5 (RP1 SoC) | CFE + PiSP BackEnd |
| `simple` | Sensors with no ISP, software ISP path | USB UVC, simple SBCs |
| `uvcvideo` | USB UVC cameras (class driver) | Minimal configuration |
| `vimc` | Virtual Media Controller | CI testing |
| `virtual` | Software-generated frames | Unit testing |
| `imx8-isi` | NXP i.MX8MP ISI | Separate from rkisp1 |
| `mali-c55` | Arm Mali-C55 ISP | |

### 4.2 match(): Device Discovery

`match()` is called once per handler during `CameraManager::start()`. It constructs a `DeviceMatch` specifying media device names and entity names that uniquely identify the hardware:

```cpp
/* src/libcamera/pipeline/ipu3/ipu3.cpp */
bool PipelineHandlerIPU3::match(DeviceEnumerator *enumerator)
{
    DeviceMatch cio2_dm("ipu3-cio2");
    cio2_dm.add("ipu3-csi2 0");
    cio2_dm.add("ipu3-cio2 0");

    DeviceMatch imgu_dm("ipu3-imgu");
    imgu_dm.add("ipu3-imgu 0");

    cio2MediaDev_ = acquireMediaDevice(enumerator, cio2_dm);
    imguMediaDev_ = acquireMediaDevice(enumerator, imgu_dm);

    if (!cio2MediaDev_ || !imguMediaDev_)
        return false;

    cio2MediaDev_->disableLinks();   /* start with a clean graph */

    return registerCameras() == 0;
}
```

`acquireMediaDevice()` opens `/dev/mediaX`, validates that all required entity names are present, and takes exclusive ownership. The enumerator is queried again on udev hotplug events.

### 4.3 configure(): Media Graph Setup

`configure()` receives a validated `CameraConfiguration` and must:

1. Select and enable Media Controller links.
2. Call `VIDIOC_SUBDEV_S_FMT` on every subdevice pad from source to sink.
3. Call `V4L2VideoDevice::setFormat()` on the capture video node.
4. Negotiate crop/compose rectangles for scaling.
5. Initialize the IPA via `ipa_->configure()`.

For the RkISP1 handler, configuration propagates formats along the chain: sensor → ISP input pad → ISP main/self output path → capture node:

```cpp
/* Simplified from src/libcamera/pipeline/rkisp1/rkisp1.cpp */
int PipelineHandlerRkISP1::configure(Camera *camera,
                                     CameraConfiguration *config)
{
    RkISP1CameraData *data = cameraData(camera);

    /* 1. Set sensor format */
    V4L2SubdeviceFormat sensorFmt;
    sensorFmt.code   = data->sensor_->mbusCodes()[0];
    sensorFmt.size   = sensorResolution;
    data->sensor_->setFormat(&sensorFmt);

    /* 2. Propagate to ISP input pad */
    isp_->setFormat(0 /* sink pad */, &sensorFmt);

    /* 3. Set ISP output pad and video node */
    V4L2DeviceFormat outFmt;
    outFmt.fourcc    = V4L2PixelFormat(formats::NV12);
    outFmt.size      = streamConfig.size;
    mainPath_->setFormat(&outFmt);

    /* 4. Configure IPA */
    ipa_->configure(data->sensorInfo_, streamConfig.pixelFormat,
                    data->parameters_);
    return 0;
}
```

### 4.4 queueRequest(): Buffer Enqueue

`queueRequestDevice()` maps each `FrameBuffer` in the `Request` to the corresponding V4L2 video device buffer and enqueues it:

```cpp
int PipelineHandlerRkISP1::queueRequestDevice(Camera *camera, Request *request)
{
    RkISP1CameraData *data = cameraData(camera);

    FrameBuffer *buf = request->findBuffer(data->mainStream_);
    if (buf) {
        /* Apply per-request controls */
        ControlList ctrls = data->isp_->controls();
        processControls(data, request, &ctrls);
        mainPath_->queueBuffer(buf);
    }

    /* Queue ISP params buffer (IPA-filled) */
    paramsBuffer_->queueBuffer(data->paramsBuffers_[buf]);
    return 0;
}
```

When the kernel DMA-completes a buffer, the V4L2 video device emits a `bufferReady` signal. The pipeline handler's slot calls `completeBuffer(request, buffer)` for each stream, then `completeRequest(request)` when all buffers in the request are done, which triggers the `Camera::requestCompleted` signal back to the application.

### 4.5 start() and stop()

`start()` imports pre-allocated `FrameBuffer` objects into the V4L2 device (using `V4L2_MEMORY_DMABUF` or `V4L2_MEMORY_MMAP`), then calls `streamOn()`. `stop()` calls `streamOff()` and releases buffer imports. The IPA is also notified of start/stop transitions via `ipa_->start()` and `ipa_->stop()`.

---

## 5. IPA: Image Processing Algorithms

### 5.1 Architecture and Isolation

IPA modules implement the algorithms that translate ISP statistics into control parameters. They are dynamically loaded shared libraries (`.so` files) installed to `/usr/lib/libcamera/ipa/` (or `/usr/local/lib/libcamera/ipa/`). The `IPAManager` discovers them at startup and validates their digital signatures.

The isolation policy is: [Source](https://libcamera.org/api-html/classlibcamera_1_1IPAManager.html)

- **Open-source IPA modules** (verified by their licence field and an optional upstream signature) are loaded directly into the libcamera process via `dlopen()`. They run in the same process but a separate thread.
- **Closed-source IPA modules** are *forcibly* run in an isolated subprocess (`ipa_proxy`). The isolation ensures no closed-source code executes in the libcamera address space. The subprocess communicates via an `IPCUnixSocket` pair — unnamed Unix datagram sockets. [Source](https://libcamera.org/api-html/classlibcamera_1_1IPCUnixSocket.html)

The proxy layer (`IPAProxyThread` for in-process, `IPAProxyProcess` for isolated) is generated automatically from a Mojo IDL interface definition (`.mojom` file), so the IPA implementation writes to a typed C++ interface and the serialisation/deserialisation and thread-safety concerns are handled by generated code.

### 5.2 IPA Interface Definition

Each platform defines its IPA interface in `include/libcamera/ipa/<platform>.mojom`. For example `include/libcamera/ipa/rkisp1.mojom` declares:

```
// Mandatory synchronous calls
interface IPARkISP1Interface {
    init(IPASettings settings, uint32 hwRevision,
         IPACameraSensorInfo sensorInfo,
         map<uint32, ControlInfoMap> entityControls)
        => (int32 ret, map<uint32, ControlInfoMap> ipaControls);

    start() => (int32 ret);
    stop();

    configure(IPARkISP1Config config, uint32 op_flags)
        => (int32 ret, map<uint32, ControlInfoMap> ipaControls);

    // Asynchronous: returns nothing, fires events
    [async] processStats(uint32 frame, uint32 bufferId,
                          ControlList sensorControls);
};

interface IPARkISP1EventInterface {
    [async] setSensorControls(uint32 frame,
                               ControlList sensorControls,
                               ControlList lensControls);
    [async] paramsBufferReady(uint32 bufferId);
    [async] metadataReady(uint32 frame, ControlList metadata);
};
```

The `[async]` attribute marks calls that cross the thread or process boundary without blocking. `processStats()` is called every frame by the pipeline handler with the ISP statistics buffer; the IPA runs its algorithms and fires `setSensorControls()` and `paramsBufferReady()` asynchronously.

### 5.3 3A and ISP Algorithms

IPA modules implement a suite of algorithms, each operating on statistics structures provided by the platform ISP. The canonical algorithm set is:

**AEC/AGC (Auto Exposure / Auto Gain Control)**: A histogram of the green channel (chosen for Bayer sensitivity balance) estimates whether the image is over- or under-exposed. The algorithm computes a target exposure product (shutter time × analogue gain) and partitions it according to the sensor's gain range and a configurable preference for gain-vs.-shutter-time. For the IPU3 IPA: [Source](https://libcamera.org/api-html/classlibcamera_1_1ipa_1_1ipu3_1_1algorithms_1_1Agc.html)

```cpp
/* src/ipa/ipu3/algorithms/agc.cpp — simplified */
void Agc::process(IPAContext &context,
                  const ipu3_uapi_stats_3a *stats)
{
    uint32_t hist[IPU3_UAPI_AWB_MAX_STRIPES] = {};
    parseHistogram(stats->awb_raw_buffer.meta_data, hist);

    double yMean = computeMeanLuminance(hist);
    double targetExposure = context.configuration.agc.exposureTarget;
    double correction = targetExposure / std::max(yMean, 0.001);

    /* Clamp to sensor limits */
    uint32_t shutter = context.activeState.agc.automatic.exposure;
    double   gain    = context.activeState.agc.automatic.gain;
    applyExposureCorrection(correction, &shutter, &gain,
                            context.configuration.agc.maxShutterSpeed,
                            context.configuration.agc.maxAnalogueGain);

    context.activeState.agc.automatic.exposure = shutter;
    context.activeState.agc.automatic.gain     = gain;
}
```

**AWB (Auto White Balance)**: The Greyworld algorithm assumes a scene's average colour under neutral lighting is grey. It computes per-pixel R/G and B/G ratios from the ISP's AWB statistics grid (excluding saturated pixels), averages them, and derives red/blue gains: `r_gain = 1/mean(R/G)`, `b_gain = 1/mean(B/G)`. [Source](https://libcamera.org/api-html/classlibcamera_1_1ipa_1_1ipu3_1_1algorithms_1_1Awb.html)

**LSC (Lens Shading Correction)**: Corrects the natural vignetting of lenses — the centre of the image is brighter than the corners. A 2D gain mesh table (typically 17×13 nodes) is computed from a flat-field calibration capture. The ISP applies the gain mesh to every incoming frame.

**CCM (Colour Correction Matrix)**: Transforms from camera RGB (sensor-specific) to sRGB. A 3×3 matrix is parameterized by colour temperature; the IPA selects or interpolates between matrices based on the AWB-estimated colour temperature.

**Gamma / Tone Mapping**: A gamma lookup table (typically 256 entries) linearises or applies a display gamma. Some platforms support tone-mapping curves for HDR.

**Noise Reduction / Debayering**: On platforms with software ISP (the Simple pipeline handler), debayering (Bayer → RGB) and temporal/spatial noise reduction runs in a CPU SIMD kernel (`debayer_cpu.cpp`). On hardware ISP platforms, these are implemented in silicon.

### 5.4 ControlList Data Flow

Statistics flow from ISP → IPA; control parameters flow from IPA → pipeline handler → ISP:

```
Frame N captured:
  ISP hardware fills stats buffer
  Pipeline handler: ipa_->processStats(frame, statsBufferId, sensorControls)
  IPA algorithms run (AGC, AWB, LSC, CCM, …)
  IPA fires: setSensorControls(frame+2, {ExposureTime, AnalogueGain, …})
                paramsBufferReady(paramsBufferId)
  Pipeline handler: sensor_->setControls(sensorControls)  ← applied to frame N+2
                    queue params buffer for ISP
Frame N+1: ISP processes with updated params
Frame N+2: Sensor captures with updated exposure/gain
```

The two-frame latency (statistics from N, controls applied at N+2) is inherent in pipelined hardware and is tracked in `RkISP1Frames` / `IPU3Frames` helper objects.

---

## 6. FrameBuffer and DMA-BUF Integration

### 6.1 FrameBuffer and Plane Layout

`FrameBuffer` is libcamera's abstraction for one captured frame. It contains a vector of `Plane` structs: [Source](https://libcamera.org/api-html/classlibcamera_1_1FrameBuffer.html)

```cpp
struct Plane {
    SharedFD fd;      /* DMA-BUF file descriptor (shared, reference-counted) */
    off_t    offset;  /* byte offset within the DMA-BUF */
    size_t   length;  /* byte length of this plane */
};
```

For packed formats like YUYV, there is one plane. For semi-planar NV12, there are two planes (luma Y, interleaved CbCr) — they may share a single DMA-BUF fd with different offsets, or use separate fds. The `FrameBufferAllocator` creates DMA-BUF-backed buffers via `V4L2_MEMORY_DMABUF` allocation from the V4L2 device, returning file descriptors suitable for cross-process sharing.

```cpp
/* Access plane file descriptors after request completion */
void handleRequestCompleted(Request *request)
{
    const Request::BufferMap &bufs = request->buffers();
    for (auto &[stream, buf] : bufs) {
        for (size_t i = 0; i < buf->planes().size(); i++) {
            const FrameBuffer::Plane &plane = buf->planes()[i];
            int fd = plane.fd.get();
            /* fd is a DMA-BUF — can be passed to GPU, encoder, display */
        }
        /* Frame metadata from IPA */
        const ControlList &meta = request->metadata();
        int64_t ts = meta.get(controls::SensorTimestamp).value_or(0);
    }
}
```

### 6.2 Zero-Copy Path to Display

The DMA-BUF file descriptor enables a zero-copy path from the ISP output to the display compositor. On a Wayland compositor supporting `linux-dmabuf-unstable-v1` (now promoted to the stable `linux-dmabuf` protocol):

```c
/* 1. Create wl_buffer from DMA-BUF */
struct zwp_linux_buffer_params_v1 *params =
    zwp_linux_dmabuf_v1_create_params(dmabuf_interface);

zwp_linux_buffer_params_v1_add(params,
    plane_fd,       /* DMA-BUF fd from libcamera plane */
    0,              /* plane index */
    0,              /* offset */
    stride,
    DRM_FORMAT_NV12,
    modifier >> 32,
    (uint32_t)modifier);

struct wl_buffer *wl_buf =
    zwp_linux_buffer_params_v1_create_immed(params,
        width, height, DRM_FORMAT_NV12, 0);

/* 2. Attach to Wayland surface */
wl_surface_attach(surface, wl_buf, 0, 0);
wl_surface_commit(surface);
```

The compositor imports the DMA-BUF via `drmPrimeFDToHandle()` into the DRM device's GEM space and scans it out without any CPU copy.

### 6.3 Zero-Copy Path to GPU (OpenGL / EGL)

For GPU processing — e.g., applying a GLSL shader to the camera feed before display — the DMA-BUF is imported as an `EGLImage` using `EGL_EXT_image_dma_buf_import`:

```c
/* Attributes for NV12: plane 0 (luma) */
EGLint attribs[] = {
    EGL_WIDTH,                  (EGLint)width,
    EGL_HEIGHT,                 (EGLint)height,
    EGL_LINUX_DMA_BUF_EXT,     EGL_NONE,
    EGL_LINUX_DRM_FOURCC_EXT,  DRM_FORMAT_NV12,
    EGL_DMA_BUF_PLANE0_FD_EXT,     plane0_fd,
    EGL_DMA_BUF_PLANE0_OFFSET_EXT, 0,
    EGL_DMA_BUF_PLANE0_PITCH_EXT,  stride,
    EGL_DMA_BUF_PLANE1_FD_EXT,     plane1_fd,
    EGL_DMA_BUF_PLANE1_OFFSET_EXT, plane1_offset,
    EGL_DMA_BUF_PLANE1_PITCH_EXT,  stride,
    EGL_NONE
};

EGLImage img = eglCreateImageKHR(egl_dpy, EGL_NO_CONTEXT,
                                  EGL_LINUX_DMA_BUF_EXT,
                                  NULL, attribs);

/* Bind to GL texture */
GLuint tex;
glGenTextures(1, &tex);
glBindTexture(GL_TEXTURE_EXTERNAL_OES, tex);
glEGLImageTargetTexture2DOES(GL_TEXTURE_EXTERNAL_OES, img);
```

The GPU imports the DMA-BUF handle and accesses the frame directly from the ISP output memory — no CPU copy, no format conversion unless the GPU shader performs it explicitly. [Source](https://docs.kernel.org/userspace-api/dma-buf-alloc-exchange.html)

### 6.4 V4L2 M2M Hardware Encode

For hardware video encoding, the DMA-BUF can be handed directly to a V4L2 Memory-to-Memory (M2M) encoder (e.g., the `bcm2835-codec` H.264 encoder on Raspberry Pi 4, or `hantro-vpu` on i.MX8):

```c
/* Set OUTPUT queue to consume DMA-BUF (camera frame) */
struct v4l2_buffer m2m_buf = {
    .type   = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE,
    .memory = V4L2_MEMORY_DMABUF,
    .index  = 0,
};
m2m_buf.m.planes[0].m.fd     = plane0_fd;
m2m_buf.m.planes[0].length   = plane0_length;
m2m_buf.m.planes[0].bytesused = plane0_length;
ioctl(encoder_fd, VIDIOC_QBUF, &m2m_buf);
```

This is the basis of the hardware encode path in `libcamera-vid` and `rpicam-vid` on Raspberry Pi 4 — the ISP output flows via DMA-BUF into the hardware H.264 encoder with no CPU involvement for frame data.

### 6.5 YUV Format Handling

libcamera surfaces pixel formats through `libcamera::formats::` constants that map 1:1 to V4L2 fourcc codes and DRM fourcc codes:

| libcamera format | V4L2 fourcc | DRM fourcc | Notes |
|-----------------|-------------|------------|-------|
| `NV12` | `V4L2_PIX_FMT_NV12` | `DRM_FORMAT_NV12` | Semi-planar YUV 4:2:0, most common ISP output |
| `YUYV` | `V4L2_PIX_FMT_YUYV` | `DRM_FORMAT_YUYV` | Packed YUV 4:2:2, UVC cameras |
| `MJPEG` | `V4L2_PIX_FMT_MJPEG` | — | Compressed, needs CPU/HW decode |
| `SBGGR10` | `V4L2_PIX_FMT_SBGGR10` | — | Raw Bayer 10-bit |
| `BGR888` | `V4L2_PIX_FMT_BGR32` | `DRM_FORMAT_BGR888` | RGB output from ISP |

MJPEG frames from USB webcams require decoding before GPU import. This is typically handled by `libjpeg-turbo` in a CPU step, or by a hardware JPEG decoder on SoCs that include one.

---

## 7. libcamera-apps and rpicam

### 7.1 libcamera-apps

`libcamera-apps` is the reference application suite bundled with libcamera. It provides command-line tools built on the libcamera C++ API, with preview support via DRM (direct KMS without a compositor) or EGL (inside a Wayland/X11 window):

- **`libcamera-hello`** — Display a live camera preview for a configurable timeout.
- **`libcamera-still`** — Capture one or a burst of still images (JPEG, PNG, BMP, raw DNG).
- **`libcamera-vid`** — Record a video stream. Encoding options include software H.264 (`libav`), hardware H.264 via V4L2 M2M, and MJPEG.
- **`libcamera-raw`** — Capture unprocessed Bayer frames, bypassing the ISP.
- **`libcamera-jpeg`** — Combined preview-then-capture workflow.

All tools share an `Options` class that parses a common set of arguments: `--width`, `--height`, `--framerate`, `--codec`, `--output`, `--timeout`, `--tuning-file`, and so on. The preview path is abstracted through a `Preview` interface with backends `EglPreview`, `DrmPreview`, and `NullPreview` (for headless encode pipelines).

### 7.2 rpicam-apps

The Raspberry Pi fork of libcamera-apps, now named **rpicam-apps**, ships as part of Raspberry Pi OS. It replaces the legacy `raspivid`/`raspistill`/`raspicam` tools and uses the `PipelineHandlerRPi` (Pi 4, via VC4 ISP) and `PipelineHandlerPiSP` (Pi 5, via PiSP BackEnd). [Source](https://www.raspberrypi.com/documentation/computers/camera_software.html)

```bash
# Preview for 5 seconds
rpicam-hello --timeout 5000

# Capture a still image
rpicam-still --output image.jpg --width 4056 --height 3040

# Record 10s of H.264 video
rpicam-vid --timeout 10000 --output video.h264 --codec h264

# Record raw Bayer (for custom ISP processing)
rpicam-raw --timeout 5000 --output frame.dng

# Network stream via UDP
rpicam-vid --timeout 0 --output udp://192.168.1.100:5000 --codec h264
```

rpicam-apps extend the shared Options framework with Raspberry Pi-specific controls (HDR mode, autofocus lens position, IMX500 AI camera tensor output).

### 7.3 Python Bindings: picamera2

Picamera2 is the official Python API for Raspberry Pi cameras, built on top of libcamera's Python bindings (generated via pybind11). It provides a high-level interface suitable for educational and prototyping use cases: [Source](https://datasheets.raspberrypi.com/camera/picamera2-manual.pdf)

```python
from picamera2 import Picamera2, Preview
import time

picam2 = Picamera2()

# Configure: main stream for capture, lores for preview
config = picam2.create_preview_configuration(
    main={"size": (1920, 1080), "format": "BGR888"},
    lores={"size": (640, 480), "format": "YUV420"},
)
picam2.configure(config)

# Start with a Qt/GL preview window
picam2.start_preview(Preview.QTGL)
picam2.start()
time.sleep(2)

# Capture a NumPy array (zero-copy via DMA-BUF mmap)
frame = picam2.capture_array("main")   # returns np.ndarray (H, W, 3)
print(f"Captured frame: shape={frame.shape}, dtype={frame.dtype}")

# Capture a still image to JPEG
picam2.capture_file("output.jpg")

picam2.stop()
```

`capture_array()` maps the DMA-BUF-backed frame buffer into the Python process's address space using `mmap()` and wraps it in a `numpy.ndarray` — a zero-copy-from-GPU operation, though the CPU must access the memory to construct the NumPy view. For truly zero-copy GPU workflows, `capture_buffer()` returns the raw `FrameBuffer` with its DMA-BUF fd.

---

## 8. PipeWire Camera Integration

### 8.1 Architecture

PipeWire integrates libcamera as a camera *source node*. The `pipewire-camera` plugin (part of the `pipewire` package on most distributions) discovers cameras through `CameraManager`, creates a `pw_node` for each, and streams frames as `spa_buffer` objects carrying DMA-BUF file descriptors. [Source](https://mind.be/eoss2023-camera-applications-with-libcamera-and-pipewire-daniel-scally-ideas-on-board-oy/)

The camera access path for applications under a modern Linux desktop is:

```
Application (Firefox, OBS, GNOME Camera, …)
      │  navigator.mediaDevices.getUserMedia() / PipeWire API
      ▼
xdg-desktop-portal (D-Bus: org.freedesktop.portal.Camera)
      │  Grants permission, creates PipeWire session
      ▼
PipeWire daemon (pw_node: libcamera source)
      │  spa_buffer with SPA_DATA_DmaBuf
      ▼
libcamera (PipelineHandler, IPA)
      │  V4L2 + Media Controller
      ▼
Camera hardware (ISP, sensor)
```

### 8.2 The xdg-desktop-portal Camera Permission Model

`org.freedesktop.portal.Camera` is the D-Bus interface through which sandboxed or permission-controlled applications request camera access. The portal broker — running as the user's desktop environment backend (GNOME Shell's `xdg-desktop-portal-gnome`, KDE's `xdg-desktop-portal-kde`) — presents a permission dialog. Once granted:

1. The portal creates a PipeWire session and returns a PipeWire **stream node ID** to the application.
2. The application connects to PipeWire and subscribes to the stream.
3. PipeWire buffers carry DMA-BUF fds, or fall back to shared memory if the GPU/ISP doesn't support DMA-BUF export.

Firefox 116+ supports this path via the `media.webrtc.camera.allow-pipewire` preference (enabled by default on recent distributions). Chrome/Chromium implements the same portal path. [Source](https://jgrulich.cz/2024/01/30/how-to-use-pipewire-camera-in-firefox/)

```
// Firefox (about:config):
media.webrtc.camera.allow-pipewire = true
```

If `xdg-desktop-portal-wlr` is the active backend (common on sway/wlroots desktops), note that it does *not* implement the Camera portal — only GNOME and KDE backends do. Applications must fall back to direct V4L2 or an alternative portal backend.

### 8.3 PipeWire pw_stream for Camera

Applications that use PipeWire's native API to receive camera frames work with `pw_stream` and SPA format negotiation:

```c
#include <pipewire/pipewire.h>
#include <spa/param/video/format-utils.h>

static void on_process(void *userdata)
{
    struct pw_stream *stream = userdata;
    struct pw_buffer *buf = pw_stream_dequeue_buffer(stream);
    if (!buf) return;

    struct spa_buffer *spa_buf = buf->buffer;
    if (spa_buf->datas[0].type == SPA_DATA_DmaBuf) {
        int fd = spa_buf->datas[0].fd;  /* DMA-BUF from libcamera */
        /* Process frame: pass to GL, encoder, OpenCV, … */
    }
    pw_stream_queue_buffer(stream, buf);
}

/* Create a consuming stream subscribing to camera */
struct pw_stream *stream = pw_stream_new(core, "my-camera-consumer",
    pw_properties_new(PW_KEY_MEDIA_TYPE, "Video",
                      PW_KEY_MEDIA_CATEGORY, "Capture",
                      PW_KEY_MEDIA_ROLE, "Camera", NULL));

uint8_t buf[1024];
struct spa_pod_builder b = SPA_POD_BUILDER_INIT(buf, sizeof(buf));
const struct spa_pod *params[1];
params[0] = spa_format_video_raw_build(&b, SPA_PARAM_EnumFormat,
    &SPA_VIDEO_INFO_RAW_INIT(.format = SPA_VIDEO_FORMAT_NV12,
                              .size   = SPA_RECTANGLE(1280, 720),
                              .framerate = SPA_FRACTION(30, 1)));

pw_stream_connect(stream, PW_DIRECTION_INPUT, node_id,
    PW_STREAM_FLAG_AUTOCONNECT | PW_STREAM_FLAG_MAP_BUFFERS, params, 1);
```

WirePlumber (the PipeWire session manager) handles linking the camera source node to the consuming stream based on the node's media properties and any active portal session policy.

---

## 9. GStreamer Integration

### 9.1 libcamerasrc Element

The `libcamerasrc` GStreamer element (`gst-plugin-libcamera`, part of `gst-plugins-good` since 1.20 or as a standalone plugin) exposes libcamera cameras as GStreamer sources. It creates a `CameraManager`, selects a camera, and negotiates formats through normal GStreamer caps negotiation.

```bash
# Preview NV12 1080p @ 30fps
gst-launch-1.0 libcamerasrc ! \
    video/x-raw,format=NV12,width=1920,height=1080,framerate=30/1 ! \
    videoconvert ! autovideosink

# Specify a camera by ID
gst-launch-1.0 libcamerasrc camera-name="imx219 4-0010" ! \
    video/x-raw,format=NV12,width=1280,height=720 ! \
    videoconvert ! autovideosink

# Raw Bayer capture
gst-launch-1.0 libcamerasrc ! \
    video/x-bayer,format=rggb,width=3280,height=2464 ! \
    filesink location=raw.bayer
```

### 9.2 Hardware H.264 Encode Pipeline

On Raspberry Pi 4 (which has a hardware H.264 encoder via `bcm2835-codec`):

```bash
# Hardware H.264 encode and save to file
gst-launch-1.0 libcamerasrc ! \
    video/x-raw,format=NV12,width=1920,height=1080,framerate=30/1 ! \
    v4l2h264enc extra-controls="controls,video_bitrate=4000000" ! \
    video/x-h264,profile=high ! \
    h264parse ! \
    mp4mux ! \
    filesink location=output.mp4

# RTSP streaming via mediamtx or similar
gst-launch-1.0 libcamerasrc ! \
    video/x-raw,colorimetry=bt709,format=NV12,width=1920,height=1080,framerate=30/1 ! \
    v4l2h264enc extra-controls="controls,video_bitrate=600000,h264_i_frame_period=15" ! \
    video/x-h264,level=(string)4 ! \
    h264parse ! \
    rtph264pay pt=96 ! \
    udpsink host=192.168.1.100 port=5000
```

Note that on Raspberry Pi 5, `v4l2h264enc` is unavailable (no hardware H.264 encoder on RP1); use `openh264enc` or `x264enc` for software encoding, or the HEVC encoder (`v4l2h265enc`) on platforms that support it.

### 9.3 appsink for OpenCV / Computer Vision

For computer vision pipelines consuming frames in C++:

```cpp
#include <gst/gst.h>
#include <gst/app/gstappsink.h>
#include <opencv2/opencv.hpp>

GstElement *pipeline = gst_parse_launch(
    "libcamerasrc ! "
    "video/x-raw,format=BGR,width=640,height=480,framerate=30/1 ! "
    "videoconvert ! "
    "appsink name=sink max-buffers=2 drop=true sync=false", nullptr);

GstElement *sink = gst_bin_get_by_name(GST_BIN(pipeline), "sink");
gst_element_set_state(pipeline, GST_STATE_PLAYING);

while (running) {
    GstSample *sample = gst_app_sink_try_pull_sample(
        GST_APP_SINK(sink), GST_SECOND);
    if (!sample) continue;

    GstBuffer *buf = gst_sample_get_buffer(sample);
    GstMapInfo map;
    gst_buffer_map(buf, &map, GST_MAP_READ);

    cv::Mat frame(480, 640, CV_8UC3, map.data);
    cv::imshow("Camera", frame);

    gst_buffer_unmap(buf, &map);
    gst_sample_unref(sample);
}
```

For zero-copy OpenCV integration, use `appsink` with DMA-BUF plane access and `cv::Mat` wrapping the mmap'd DMA-BUF, avoiding the `gst_buffer_map()` copy.

### 9.4 Caps Negotiation

`libcamerasrc` negotiates with downstream elements through GStreamer caps. The element queries the camera's supported formats and sizes, generates a `GstCaps` structure from `PixelFormat` and `Size` pairs, and lets GStreamer's pad negotiation machinery find a compatible format. Adding a `v4l2convert` element between `libcamerasrc` and `v4l2h264enc` is often necessary when the encoder's accepted pixel format (e.g., `NV12`) differs from libcamera's negotiated output.

---

## 10. Platform Coverage and Hardware

### 10.1 Raspberry Pi 4: rpi/vc4 Pipeline

On Raspberry Pi 0 through 4, the camera pipeline consists of:

- **Unicam** — the BCM283x CSI-2 receiver driver (`drivers/media/platform/broadcom/bcm2835-unicam.c`). Unicam receives MIPI CSI-2 data from the sensor and deposits it into memory as Bayer data or embedded sensor metadata. It appears in the media graph as the entity `unicam`.
- **VC4 ISP** — the VideoCore IV ISP, controlled by firmware running on the VideoCore. libcamera's `rpi/vc4` pipeline handler communicates with the ISP via the `bcm2835-isp` V4L2 M2M driver, passing Bayer input and receiving processed output (NV12, etc.) plus statistics.
- **IPA**: `src/ipa/rpi/vc4/` — implements AGC, AWB, LSC, CCM, gamma, noise reduction, and focus algorithms tuned for the BCM283x ISP capabilities.

Supported sensors (with upstream `drivers/media/i2c/` kernel drivers and IPA tuning files): OV5647 (Pi Camera V1), IMX219 (Pi Camera V2), IMX477 (Pi HQ Camera), IMX708 (Pi Camera V3), IMX296 (Pi Global Shutter), IMX500 (Pi AI Camera).

### 10.2 Raspberry Pi 5: rpi/pisp Pipeline

Raspberry Pi 5 uses the RP1 south bridge SoC, which includes:

- **CFE (Camera Front End)** — the CSI-2 DPHY + receiver on RP1, exposed as the kernel driver `rp1-cfe`. Media entities: `rp1-cfe-fe-image0`, `rp1-cfe-fe-stats`, `rp1-cfe-fe-config`.
- **PiSP BackEnd** — a programmable ISP BackEnd (`pispbe` kernel driver) that accepts Bayer input and produces multiple processed output streams simultaneously. Unlike the VC4 ISP, PiSP supports fully programmable tone-mapping, noise reduction, and colour processing.
- **IPA**: `src/ipa/rpi/pisp/` — extends the VC4 IPA with PiSP-specific configuration structures using `libpisp`.

The `rpi/pisp` pipeline handler's `match()` function searches for `rp1-cfe` media devices and acquires both the CFE and PiSP BackEnd media devices. [Source: patch series, libcamera mailing list](https://lists.libcamera.org/pipermail/libcamera-devel/2025-January/048118.html)

### 10.3 Intel IPU3: ipu3 Pipeline

The Intel IPU3 (Image Processing Unit version 3) is the camera subsystem found in Intel Skylake, Kaby Lake, and Apollo Lake SoCs. It consists of two separate media devices: [Source](https://deepwiki.com/kbingham/libcamera/3.1-ipu3-pipeline-handler)

- **ipu3-cio2** — the CIO2 (Camera Input/Output 2) CSI-2 receiver. Media entities: up to 4 `ipu3-csi2 N` entities and `ipu3-cio2 N` capture nodes. The CIO2 accepts raw sensor data and deposits it into memory.
- **ipu3-imgu** — the ImgU (Image Processing Unit). Two ImgU instances (`ipu3-imgu 0`, `ipu3-imgu 1`) process CIO2 output through Bayer down-scaling, geometric distortion correction, and 3A statistics generation.

The IPU3 `match()` function acquires both media devices and calls `registerCameras()` which walks the CIO2 media graph to find attached sensors. Configuration calculates ImgU input/output sizes respecting alignment constraints (width multiple of 64, height multiple of 32). The IPA algorithms in `src/ipa/ipu3/` consume ImgU statistics buffers (`ipu3_uapi_stats_3a`) and produce parameter buffers (`ipu3_uapi_params`).

### 10.4 Intel IPU6: ipu6 Pipeline

Intel IPU6 (Tiger Lake, Alder Lake, Raptor Lake) is architecturally distinct from IPU3. The `ipu6` pipeline handler supports it through a separate code path; the CSI-2 frontend and ISP communicate via a different media topology and the IPA algorithms target IPU6-specific statistics structures.

### 10.5 Rockchip RkISP1: rkisp1 Pipeline

The Rockchip ISP1 is present in RK3288, RK3399, RK3568, RK3588 (via a compatible ISP version), and NXP i.MX8MP. The upstream kernel driver at `drivers/media/platform/rockchip/rkisp1/` exposes: [Source](https://docs.kernel.org/admin-guide/media/rkisp1.html)

- `rkisp1_isp` — the ISP entity with input pads (from DPHY/CSI) and output pads (main path, self path).
- `rkisp1_resizer_mainpath` / `rkisp1_resizer_selfpath` — resizer blocks on each output path.
- `rkisp1_mainpath` / `rkisp1_selfpath` — V4L2 capture nodes.
- `rkisp1_params` / `rkisp1_stats` — V4L2 nodes for ISP parameter input and statistics output.

RK3399 has two ISP instances (`rkisp1_0`, `rkisp1_1`) supporting dual-camera configurations. The `rkisp1` pipeline handler (in `src/libcamera/pipeline/rkisp1/`) configures the ISP via the params node and reads 3A statistics from the stats node, delegating algorithm computation to `src/ipa/rkisp1/`.

### 10.6 Simple Pipeline Handler

The `simple` pipeline handler supports cameras that either have no ISP (raw capture only) or use a very simple ISP that doesn't require full graph control — notably most USB UVC cameras and some embedded sensors on I²C + parallel interface. [Source](https://libcamera.org/open-projects.html)

For cameras without hardware ISP, the Simple handler incorporates a **software ISP** path (`src/libcamera/software_isp/`) that performs Bayer demosaicing, white balance, and basic colour correction on the CPU using SIMD-optimised kernels. This path runs in a separate thread and adds latency.

### 10.7 VIMC: Virtual Media Controller

VIMC (`drivers/media/test-drivers/vimc/`) is a kernel driver that creates a virtual media graph — sensor entity → debayer → scaler → capture node — without real hardware. The `vimc` pipeline handler in libcamera uses it for continuous integration testing: libcamera CI runs the full pipeline stack against VIMC on standard kernel VMs, catching regressions without camera hardware. [Source: patch](https://patchwork.libcamera.org/patch/106/)

```bash
# Load VIMC kernel module
sudo modprobe vimc

# Run libcamera against VIMC
cam --list-cameras   # should show "VIMC Sensor B"
cam --camera 0 --stream role=viewfinder,width=640,height=480 --capture=10
```

### 10.8 V4L2 Compatibility Layer

For applications that use direct V4L2 ioctls on `/dev/video*` and cannot be modified, libcamera provides a V4L2 compatibility layer (`src/v4l2/v4l2_compat.cpp`) loaded via `LD_PRELOAD`:

```bash
LD_PRELOAD=/usr/lib/libcamera/v4l2-compat.so my_v4l2_application
```

The layer intercepts `open()`, `ioctl()`, `mmap()`, and `close()` calls on paths matching `/dev/video*` and routes them through the libcamera API, transparently presenting complex ISP pipelines as simple V4L2 devices. This enables applications like `ffmpeg -f v4l2 -i /dev/video0` to benefit from ISP processing on platforms where the raw sensor output would otherwise require direct media controller manipulation.

---

## Roadmap

### Near-term (6–12 months)

- **GPU-accelerated software ISP (GPUISP):** A 35-patch series introducing a GLES 2.0 GPU ISP backend for libcamera's `software_isp` path has been posted to the mailing list, aiming to offload Bayer demosaicing and 3A processing to the GPU for platforms without a dedicated hardware ISP (notably Intel IPU6 laptops). [Source](https://patchwork.libcamera.org/cover/23506/)
- **Intel IPU6 image quality and power improvements:** Current IPU6 support via the Simple pipeline handler delivers adequate video-conferencing quality but degrades battery life relative to hardware ISP operation. Near-term work targets per-sensor tuning data and improved AEC/AWB algorithms in the software IPA to close the quality gap. [Source](https://fosdem.org/2026/schedule/event/TKSK3G-libcamera-softisp/)
- **Multi-threaded software ISP debayering:** libcamera 0.7.1 introduced multi-threaded Bayer demosaicing in the SoftISP path; follow-on patches are expected to further parallelise the colour-correction and tone-mapping stages. [Source](https://www.phoronix.com/news/libcamera-0.7.1-Released)
- **Camera synchronisation layer:** A synchronisation layer that aligns captured frames across heterogeneous cameras using `linux-ptp` is in active prototyping. Accuracy is currently around one image-line period; sub-line precision via horizontal-blanking adjustment is a stated near-term target. [Source](https://libcamera.org/entries/2025-08-25.html)
- **Camshark debug/tuning toolchain:** A new developer tool — demonstrated at ELCE 2025 — for pipeline introspection and IPA tuning parameter visualisation is being upstreamed alongside the Layers plugin infrastructure. [Source](https://libcamera.org/entries/2025-08-25.html)

### Medium-term (1–3 years)

- **libcamera Layers plugin architecture:** The Layers design introduces a per-request middleware stack between the application API and the PipelineHandler, enabling features such as Zero Shutter Lag, burst HDR merging, and vendor post-processing extensions to be bolted in without forking core pipeline code. The architecture was presented at the 2025 libcamera workshop. [Source](https://lwn.net/Articles/1022874/)
- **Renesas R-Car V4H and Dream Chip RPP-X1 ISP support:** Full pipeline handler and IPA support for the Renesas R-Car V4H (automotive SoC) with the Dream Chip RPP-X1 ISP companion chip was demonstrated at ELCE 2025; upstream submission of the pipeline handler is expected in the 1–2 year timeframe. [Source](https://libcamera.org/entries/2025-08-25.html)
- **Intel IPU6 hardware PSYS pipeline handler:** The PSYS (Processing System) block of IPU6, which performs hardware ISP operations analogous to IPU3's ImgU, lacks a mainline kernel driver. A hardware PSYS pipeline handler is a medium-term goal that would eliminate the software ISP workaround on modern Intel laptops. Note: needs verification of upstream timeline.
- **Android Camera HAL3 adapter stabilisation:** The `src/android/` HAL3 adapter that enables ChromeOS and Android-on-Linux to use libcamera is functional but not officially supported as a stable ABI; medium-term work aims to close the feature gap with vendor HALs and add CTS compliance testing. [Source](https://libcamera.org/faq.html)
- **PipeWire camera portal v2 integration:** Deeper integration with the xdg-desktop-portal Camera interface is planned to support richer camera controls (zoom, focus, exposure) negotiated between PipeWire nodes and applications, aligning with the Flatpak sandboxing roadmap. Note: needs verification of specific portal API version timeline.

### Long-term

- **Networked / distributed camera synchronisation:** The `linux-ptp`-based synchronisation layer is designed to scale beyond dual-camera stereo rigs to larger arrays — surveillance, volumetric capture, multi-drone swarms — requiring robust clock discipline, fault-tolerance in the sync protocol, and network-transparent libcamera API extensions. [Source](https://libcamera.org/entries/2025-08-25.html)
- **Machine-learning IPA modules:** Replacing hand-tuned 3A algorithms with on-device ML inference (NPU-offloaded AE/AF/AWB networks) is a research-stage direction discussed in the libcamera community; the IPA sandbox's process-isolation model is architecturally suited to hosting NPU runtimes. Note: needs verification.
- **Unified kernel camera graph (MC next-gen):** Long-running discussions on the Linux media mailing list explore replacing the current per-driver Media Controller entity model with a more declarative firmware-described graph (Device Tree camera graph bindings, ACPI `_CRS` camera topology) that would reduce platform-specific PipelineHandler complexity. Note: needs verification of formal RFC status.
- **libcamera as the universal V4L2 abstraction layer:** The V4L2 compatibility shim (`LD_PRELOAD=v4l2-compat.so`) is currently best-effort; a long-term goal is to make it reliable enough that unmodified FFmpeg, OpenCV, and legacy V4L2 applications transparently benefit from libcamera ISP processing on all supported platforms. [Source](https://libcamera.org/open-projects.html)

---

## 11. Integrations

**Ch26 (Hardware Video Decode/Encode)** — The libcamera DMA-BUF output is the natural input to V4L2 M2M hardware encoders. On Raspberry Pi 4, the `bcm2835-codec` encoder receives NV12 DMA-BUFs from the VC4 ISP output. VA-API encode on Intel platforms can similarly consume DMA-BUF surfaces via `vaExportSurfaceHandle()`. The `libcamera-vid` and `rpicam-vid` tools exercise this path directly.

**Ch38 (PipeWire)** — libcamera is the camera backend behind PipeWire's camera source nodes. WirePlumber links libcamera PipeWire nodes to application streams based on policy. The xdg-desktop-portal Camera interface (`org.freedesktop.portal.Camera`) brokers access. All modern WebRTC applications (Firefox, Chrome, GNOME Calls) reach libcamera via this PipeWire/portal path rather than directly.

**Ch57 (FFmpeg)** — `ffmpeg -f v4l2 -i /dev/video0` reaches libcamera on complex-pipeline platforms through the V4L2 compatibility layer (`LD_PRELOAD=v4l2-compat.so`). On simpler platforms (USB webcams), V4L2 and libcamera both work directly. The `libcamera` FFmpeg input device (`-f libcamera`) has been prototyped but is not in mainline as of mid-2026.

**Ch85 (Android Camera HAL)** — Android's Camera HAL3 is the direct counterpart to libcamera in the Android ecosystem. libcamera includes a Camera HAL3 adapter (`src/android/`) that implements the Android HAL3 interface on top of libcamera, enabling ChromeOS and Android-on-Linux platforms to use libcamera's pipeline handlers and IPAs rather than vendor HAL blobs.

**Ch92 (Raspberry Pi Platform)** — The `rpi/vc4` and `rpi/pisp` pipeline handlers are the Raspberry Pi-specific implementations of `PipelineHandler`. They encapsulate deep knowledge of the BCM283x VC4 ISP and the RP1 PiSP architectures respectively, including the ISP parameter and statistics buffer formats that the corresponding IPA modules consume.

---

### References

- libcamera project home: [https://libcamera.org/](https://libcamera.org/)
- libcamera source repository: [https://github.com/libcamera-org/libcamera](https://github.com/libcamera-org/libcamera)
- libcamera API reference: [https://libcamera.org/api-html/](https://libcamera.org/api-html/)
- Pipeline Handler Writer's Guide: [https://libcamera.org/guides/pipeline-handler.html](https://libcamera.org/guides/pipeline-handler.html)
- IPA Writer's Guide: [https://github.com/libcamera-org/libcamera/blob/master/Documentation/guides/ipa.rst](https://github.com/libcamera-org/libcamera/blob/master/Documentation/guides/ipa.rst)
- Linux Kernel Media Controller documentation: [https://docs.kernel.org/userspace-api/media/mediactl/media-controller.html](https://docs.kernel.org/userspace-api/media/mediactl/media-controller.html)
- Linux Kernel V4L2 subdev API: [https://docs.kernel.org/driver-api/media/v4l2-subdev.html](https://docs.kernel.org/driver-api/media/v4l2-subdev.html)
- Linux Kernel RkISP1 documentation: [https://docs.kernel.org/admin-guide/media/rkisp1.html](https://docs.kernel.org/admin-guide/media/rkisp1.html)
- DMA-BUF allocation and exchange: [https://docs.kernel.org/userspace-api/dma-buf-alloc-exchange.html](https://docs.kernel.org/userspace-api/dma-buf-alloc-exchange.html)
- Picamera2 manual: [https://datasheets.raspberrypi.com/camera/picamera2-manual.pdf](https://datasheets.raspberrypi.com/camera/picamera2-manual.pdf)
- Raspberry Pi camera software documentation: [https://www.raspberrypi.com/documentation/computers/camera_software.html](https://www.raspberrypi.com/documentation/computers/camera_software.html)
- IPCUnixSocket API reference: [https://libcamera.org/api-html/classlibcamera_1_1IPCUnixSocket.html](https://libcamera.org/api-html/classlibcamera_1_1IPCUnixSocket.html)
- IPU3 Pipeline Handler deep-dive: [https://deepwiki.com/kbingham/libcamera/3.1-ipu3-pipeline-handler](https://deepwiki.com/kbingham/libcamera/3.1-ipu3-pipeline-handler)
- Wikipedia: libcamera: [https://en.wikipedia.org/wiki/Libcamera](https://en.wikipedia.org/wiki/Libcamera)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
