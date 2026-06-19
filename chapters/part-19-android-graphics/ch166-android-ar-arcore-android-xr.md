# Chapter 166: Android AR: ARCore Architecture, Camera HAL Integration, and Android XR

> **Audiences:** Android graphics and AR application developers targeting the ARCore C API and Camera2 pipeline; platform engineers working on Camera HAL3 integration and sensor fusion; developers building cross-platform XR applications targeting both Android XR (Samsung Galaxy XR) and Linux (Monado/SteamVR).

---

## Table of Contents

1. [Introduction: ARCore in the Android Graphics Stack](#1-introduction-arcore-in-the-android-graphics-stack)
2. [ARCore Architecture Overview](#2-arcore-architecture-overview)
3. [Android Camera HAL3 Architecture](#3-android-camera-hal3-architecture)
4. [ARCore ↔ Camera HAL Data Flow](#4-arcore--camera-hal-data-flow)
5. [Sensor Fusion and Tracking](#5-sensor-fusion-and-tracking)
6. [ARCore GPU Pipeline](#6-arcore-gpu-pipeline)
7. [Android XR Platform](#7-android-xr-platform)
8. [OpenXR on Android XR](#8-openxr-on-android-xr)
9. [Snapdragon Spaces: Qualcomm's AR SDK](#9-snapdragon-spaces-qualcomms-ar-sdk)
10. [Linux AR: Monado, SteamVR, and the Open Stack](#10-linux-ar-monado-steamvr-and-the-open-stack)
11. [Integrations](#11-integrations)

---

## 1. Introduction: ARCore in the Android Graphics Stack

**ARCore** is Google's augmented reality SDK for Android — an application-layer platform that sits above the operating system, consuming camera frames from the Android Camera HAL and inertial measurements from the SensorManager, and producing world-understanding primitives (device pose, detected planes, depth maps, light estimates, and cloud anchors) that renderers consume to overlay virtual content on a live camera feed. Unlike Vulkan drivers or the DRM subsystem, ARCore is not a kernel module or a hardware abstraction layer component: it is a Play Services component (`com.google.ar.core`) that runs partly in the app process and partly as a background ARCore service, updated independently of Android OS version through the Play Store.

ARCore's graphics integration is deep. The camera preview stream arrives as an `AHardwareBuffer` — a zero-copy handle backed by the same gralloc buffer that the Camera ISP wrote into — which ARCore exposes to the app as an OpenGL ES external texture (`GL_TEXTURE_EXTERNAL_OES`) or, on the Vulkan path, as a `VkImage` imported via `VK_ANDROID_external_memory_android_hardware_buffer`. The 4×4 view and projection matrices that ARCore's Visual-Inertial Odometry produces are consumed directly by the app's render loop to position virtual geometry in the AR scene.

**Android XR** is the evolution of this stack to headset form factors. Announced at Google I/O 2024 and first shipping on the Samsung Galaxy XR headset (formerly Project Moohan) in October 2025, Android XR is a standard Android 15+ base with spatial-computing extensions: an OpenXR 1.0 runtime, passthrough compositing of the real-world camera feed, a Jetpack XR SDK (`androidx.xr.*`), and support for 6DoF tracking on Snapdragon XR2+ Gen 2 hardware. Where ARCore sessions (`ArSession`) map to 6DoF phone AR, Android XR maps the same tracking primitives to the `XrSession` / `XrSpace` model of OpenXR, enabling code reuse between phone AR and headset XR.

This chapter differs from Ch87 (which covers ARCore's core session lifecycle, plane detection, depth API, and light estimation in detail) by focusing specifically on the lower-level interfaces: Camera HAL3 architecture, the sensor fusion pipeline, the GPU buffer path, and the Android XR / OpenXR runtime. Developers should read Ch87 for the AR application API surface and this chapter for the platform and graphics integration internals.

---

## 2. ARCore Architecture Overview

### Components

ARCore's feature set is built from five subsystems:

1. **Motion Tracking (Visual-Inertial Odometry).** Continuous 6DoF pose estimation fusing camera feature tracks with accelerometer and gyroscope data via an Extended Kalman Filter (EKF). The output is `ArPose`: a 7-element `{qx, qy, qz, qw, tx, ty, tz}` quaternion-translation tuple. Exposed as `ArCamera_getPose()` and `ArCamera_getDisplayOrientedPose()`.

2. **Plane Detection and Environmental Understanding.** The SLAM subsystem maintains a keyframe graph and incrementally grows detected horizontal and vertical planes (`ArPlane`) and feature point clouds (`ArPointCloud`) as the user moves. Scene Semantics (Android 12+, `AR_SEMANTIC_MODE_ENABLED`) per-pixel labels via an on-device ML model.

3. **Light Estimation.** Analysis of the camera image to produce ambient intensity scalars or a full spherical HDR environment map (`ArImageCubemap` in `AIMAGE_FORMAT_RGBA_FP16` plus L2 spherical harmonics coefficients) for PBR image-based lighting.

4. **Depth API.** Per-pixel depth from structured-light / time-of-flight sensors (`ArFrame_acquireDepthImage16Bits()`), or from ARCore's ML-based MotionStereo algorithm (`AR_DEPTH_MODE_AUTOMATIC`) which works on depth-sensor-less devices by computing disparity between successive frames.

5. **Cloud Anchors.** Persisted world anchors with a `cloud_anchor_id` string, hosted and resolved via Google's Visual Positioning Service (VPS). Multi-user AR experiences anchor virtual content at known WGS84 coordinates accessible via `ArSession_hostCloudAnchorAsync()`.

### Process model

ARCore ships as two components. The **ARCore app library** (`libarcore_sdk_c.so` for NDK or the Kotlin/Java `com.google.ar.core` Gradle dependency) links into the application process. It communicates via Binder IPC with the **ARCore service** (`com.google.ar.core`), a privileged background process that is updated via Play Store independently of the device OS image. The ARCore service holds the camera and sensor subscriptions; the app-process library exposes the `ArSession` handle that marshals frame data back.

This separation enables ARCore to be updated — improving SLAM accuracy, adding new device support — without an OEM firmware update or system image flash. The tradeoff is an extra Binder hop on each `ArSession_update()` call, though in practice the per-frame overhead is dominated by camera ISP and GPU work.

### Platform support and native API

ARCore supports Android 7.0 (API 24) and above on a curated hardware list exceeding one billion devices as of 2025 ([ARCore supported devices](https://developers.google.com/ar/devices)). The requirement is not merely an Android version: each device must pass hardware validation by the OEM for accurate gyroscope calibration, camera–IMU timestamp alignment, and ISP quality.

The **C API** (`libarcore_sdk_c.so`) is the lowest-level interface:

```c
// arcore_c_api.h — representative declarations
// https://github.com/google-ar/arcore-android-sdk/blob/main/libraries/include/arcore_c_api.h

typedef struct ArSession_ ArSession;
typedef struct ArFrame_  ArFrame;
typedef struct ArCamera_ ArCamera;
typedef struct ArPose_   ArPose;

typedef enum ArStatus {
    AR_SUCCESS                   = 0,
    AR_ERROR_NOT_TRACKING        = 2,
    AR_ERROR_SESSION_PAUSED      = 3,
    AR_ERROR_FATAL               = 1001,
    /* ... */
} ArStatus;

typedef enum ArTrackingState {
    AR_TRACKING_STATE_TRACKING = 0,
    AR_TRACKING_STATE_PAUSED   = 1,
    AR_TRACKING_STATE_STOPPED  = 2,
} ArTrackingState;
```

The Java/Kotlin API (`com.google.ar.core.Session`, `Frame`, `Camera`) wraps the C API and is the recommended entry point for pure-Java apps, but NDK-based renderers (OpenGL ES or Vulkan) typically use the C API directly for lower latency and simpler buffer management.

---

## 3. Android Camera HAL3 Architecture

### Interface hierarchy

Android Camera HAL3 defines an AIDL-based interface hierarchy (HIDL in Android ≤12; migrated to AIDL in Android 13+). The three primary interfaces are:

- **`ICameraProvider`** — Discovers available camera devices, reports camera status changes (hot-plug), and returns `ICameraDevice` handles. AIDL source: `hardware/interfaces/camera/provider/aidl/android/hardware/camera/provider/ICameraProvider.aidl` ([AOSP](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/camera/provider/aidl/)).
- **`ICameraDevice`** — Represents a single camera logical unit (front, rear, logical multi-camera). Opens a session via `open()`, returning an `ICameraDeviceSession`.
- **`ICameraDeviceSession`** — The active session; receives `processCaptureRequest()` calls from the camera service (`cameraserver`) and delivers results via callbacks.

The legacy HIDL `camera3_device_ops_t` C struct (still present in `hardware/libhardware/include/hardware/camera3.h`) defines the HAL entry points at the C ABI level:

```c
/* hardware/libhardware/include/hardware/camera3.h */
typedef struct camera3_device_ops {
    /* Initialize the HAL device. */
    int (*initialize)(const struct camera3_device *,
                      const camera3_callback_ops_t *callback_ops);

    /* Configure stream set for upcoming capture requests. */
    int (*configure_streams)(const struct camera3_device *,
                             camera3_stream_configuration_t *stream_list);

    /* Submit a single capture request to the HAL. Non-blocking. */
    int (*process_capture_request)(const struct camera3_device *,
                                   camera3_capture_request_t *request);

    void (*dump)(const struct camera3_device *, int fd);
    int  (*flush)(const struct camera3_device *);
} camera3_device_ops_t;
```

[Source: AOSP Camera HAL3 documentation](https://source.android.com/docs/core/camera/camera3)

### Stream configuration and capture requests

Streams are described by `camera3_stream_t`:

```c
typedef struct camera3_stream {
    int      stream_type;      /* CAMERA3_STREAM_OUTPUT / INPUT / BIDIRECTIONAL */
    uint32_t width;
    uint32_t height;
    int      format;           /* HAL_PIXEL_FORMAT_YCbCr_420_888 for VIO; */
                               /* HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED for preview */
    uint32_t usage;            /* gralloc usage flags: GRALLOC_USAGE_HW_TEXTURE etc. */
    uint32_t max_buffers;      /* max inflight buffers the HAL may hold */
    void    *priv;             /* HAL-private state */
    android_dataspace_t data_space;
    camera3_stream_rotation_t rotation;
} camera3_stream_t;
```

A `CaptureRequest` submitted to `process_capture_request()` carries:
- **Settings metadata** — 3A parameters (AE, AF, AWB), noise reduction, edge enhancement, crop region.
- **Output buffer set** — one `camera3_stream_buffer_t` per configured output surface, each wrapping an `AHardwareBuffer` (gralloc handle).
- **Input buffer** (optional) — for reprocessing flows.

Results arrive asynchronously via `camera3_callback_ops_t::process_capture_result()`, providing the metadata result and filled output buffers with synchronisation fences.

### AHardwareBuffer and buffer management

`AHardwareBuffer` is the Android NDK opaque handle for a GPU-accessible, cross-process buffer backed by the kernel's DMA-BUF mechanism (or ION on older kernels). In the camera context:

```c
/* Acquire the native buffer handle for a camera image (NDK AImage API) */
AHardwareBuffer *hardwareBuffer;
media_status_t status = AImage_getHardwareBuffer(image, &hardwareBuffer);
/* hardwareBuffer is now importable into EGL or Vulkan */
```

For the YUV_420_888 format used by ARCore's motion tracking stream, the Y plane, U plane, and V plane are accessible via `AImage_getPlaneData()` for CPU-side feature extraction. The GPU preview surface receives the buffer via a `Surface` / `ANativeWindow` backed by the same gralloc allocator that the Camera ISP writes into — zero additional copies from ISP to GPU.

### Synchronisation fences

Every camera output buffer carries a release fence: a `sync_fence` fd that becomes signalled when the ISP finishes writing. ARCore's GPU pipeline waits on this fence before sampling the camera texture:

```c
/* Framework-level fence flow (simplified) */
camera3_stream_buffer_t buffer;
buffer.acquire_fence = -1;       /* Framework: no acquire constraint */
buffer.release_fence = -1;       /* HAL fills this on result callback */

/* After result callback: release_fence is valid fd.
   ARCore's EGL path waits on it before using EGLImageKHR. */
```

On the Vulkan path (Section 6), the fence becomes a `VkSemaphore` via `VkImportSemaphoreFdInfoKHR` using `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT`, establishing GPU–GPU ordering between the ISP completion and the vertex/fragment stage that samples the camera image.

---

## 4. ARCore ↔ Camera HAL Data Flow

### Camera2 session open

When `ArSession_resume()` is called from the app, the ARCore library (via the ARCore service) executes the following Camera2 sequence in the framework layer:

```kotlin
// Conceptual — ARCore executes this internally via the ARCore service process

val manager = context.getSystemService(Context.CAMERA_SERVICE) as CameraManager

// 1. Open the camera (typically the rear-facing camera)
manager.openCamera(cameraId, object : CameraDevice.StateCallback() {
    override fun onOpened(device: CameraDevice) {
        cameraDevice = device
        createCaptureSession(device)
    }
    override fun onDisconnected(device: CameraDevice) { device.close() }
    override fun onError(device: CameraDevice, error: Int) { device.close() }
}, cameraHandler)

// 2. Configure the capture session with output surfaces
fun createCaptureSession(device: CameraDevice) {
    val surfaces = listOf(
        previewSurface,        // GL_TEXTURE_EXTERNAL_OES target from ArSession
        imageReaderSurface     // CPU-accessible YUV for depth/semantics (optional)
    )
    device.createCaptureSession(surfaces,
        object : CameraCaptureSession.StateCallback() {
            override fun onConfigured(session: CameraCaptureSession) {
                // 3. Submit repeating request with TEMPLATE_RECORD
                val builder = device.createCaptureRequest(
                    CameraDevice.TEMPLATE_RECORD)
                builder.addTarget(previewSurface)
                builder.set(CaptureRequest.CONTROL_AF_MODE,
                    CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_VIDEO)
                session.setRepeatingRequest(builder.build(), null, cameraHandler)
            }
            override fun onConfigureFailed(session: CameraCaptureSession) {}
        }, cameraHandler)
}
```

[Source: Android Camera2 API](https://developer.android.com/media/camera/camera2)

`TEMPLATE_RECORD` is chosen over `TEMPLATE_PREVIEW` because it requests a stable, low-jitter framerate (typically 30 fps) with minimal ISP latency variation — essential for tight IMU-to-image timestamp alignment.

### Preview stream format and resolution

ARCore's primary camera stream uses `HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED`, letting the HAL choose the optimal format (typically NV12 on Snapdragon; NV21 or P010 on other platforms). The resolution is typically 640×480 or 1280×720 for the VIO pipeline — higher resolutions increase feature-point count but also CPU cost. On depth-capable devices, a second stream in `HAL_PIXEL_FORMAT_Y16` or a proprietary depth format provides the raw depth sensor output.

### Depth stream and ARCore Depth API

On devices with ToF (time-of-flight) or structured-light depth sensors (e.g., Pixel 7 Pro with the Samsung 3D ToF module), ARCore configures an additional depth stream. The 16-bit depth image is accessible via:

```c
// C API — acquire depth image from the current frame
ArImage *depth_image = nullptr;
ArStatus status = ArFrame_acquireDepthImage16Bits(
    ar_session,
    ar_frame,
    &depth_image);

if (status == AR_SUCCESS) {
    int32_t width, height;
    ArImage_getWidth(ar_session, depth_image, &width);
    ArImage_getHeight(ar_session, depth_image, &height);

    const uint8_t *data;
    int32_t data_length;
    ArImage_getPlaneData(ar_session, depth_image,
        /*plane_index=*/0, &data, &data_length);
    // data is an array of uint16_t values in millimetres
    // 0 = invalid/no-depth; 1–65535 = depth in mm

    ArImage_release(depth_image);
}
```

[Source: ARCore C API — ArFrame](https://developers.google.com/ar/reference/c/group/ar-frame)

On devices without a hardware depth sensor, ARCore's MotionStereo algorithm (`AR_DEPTH_MODE_AUTOMATIC`) estimates dense depth from the monocular camera stream using a network running on the Hexagon DSP or NNAPI.

### IMU timestamp synchronisation

Accurate IMU-to-camera timestamp alignment is the single most critical factor for VIO quality. ARCore registers gyroscope and accelerometer listeners at `SENSOR_DELAY_FASTEST` (typically 200–400 Hz):

```java
// Camera: timestamps come from CaptureResult.SENSOR_TIMESTAMP (nanoseconds,
// CLOCK_BOOTTIME domain on Android 7+)
Long cameraTimestamp = captureResult.get(CaptureResult.SENSOR_TIMESTAMP);

// IMU: SensorEvent.timestamp is also CLOCK_BOOTTIME nanoseconds
public void onSensorChanged(SensorEvent event) {
    long imuTimestamp = event.timestamp; // CLOCK_BOOTTIME ns
    float[] gyroValues = event.values;   // [rad/s x, y, z]
}
```

Both sources use `CLOCK_BOOTTIME` since Android 7, enabling direct timestamp comparison without clock domain conversion. ARCore's EKF preintegrates IMU measurements between camera frames to produce a high-rate pose estimate, corrected at each camera frame boundary by the visual feature track update.

---

## 5. Sensor Fusion and Tracking

### IMU characteristics and sensor noise model

The accelerometer and gyroscope on a modern Android device (e.g., ICM-42688-P or LSM6DSO) operate at 400–1000 Hz. Their noise model has four dominant terms:

- **Additive white noise** (ARW / VRW) — gyroscope angle random walk in rad/s/√Hz; accelerometer velocity random walk in m/s/√Hz.
- **Bias instability** — slow-varying offset; modelled as a random-walk process in the EKF state.
- **Temperature drift** — mitigated by factory calibration tables stored in device firmware.
- **Mechanical vibration** — camera OIS can couple into the IMU; some devices carry the IMU on the camera module specifically to measure OIS motion directly.

ARCore's internal calibration routine (run during `ArSession_resume()` if the device has not been previously calibrated in the ARCore service) estimates the camera–IMU extrinsic transform (rotation + translation between camera optical centre and IMU sensor origin) and the IMU-to-camera time offset `Δt`. This calibration is stored in the ARCore service and reused across sessions.

### Visual-Inertial Odometry pipeline

ARCore's VIO pipeline follows a tightly-coupled formulation:

1. **IMU preintegration** — Between camera frames at time `t_{k-1}` and `t_k`, IMU measurements are preintegrated to produce a relative pose delta `ΔR, Δv, Δp` and preintegrated covariance, following the on-manifold preintegration of Forster et al. (2017).

2. **Feature extraction** — On each camera frame, a CPU-side thread extracts corner-like features (FAST corners followed by a descriptor-free tracking step using KLT optical flow between consecutive frames). Typically 100–300 features are tracked per frame.

3. **EKF update** — The preintegrated IMU delta provides the process model; the feature reprojection residuals (observed 2D position vs. projected 3D landmark) provide the measurement model. The EKF state vector is `[R, v, p, b_g, b_a]` — rotation, velocity, position, gyroscope bias, accelerometer bias.

4. **Keyframe selection and loop closure** — When the device moves sufficiently (translation or rotation above a threshold), a new keyframe is inserted into the keyframe graph. A bag-of-words place recognition step (similar to DBoW2) detects loop closure candidates; on positive detection, a non-linear pose graph optimisation corrects accumulated drift.

5. **Tracking state machine** — The `ArTrackingState` enum reflects VIO health:

```c
ArTrackingState tracking_state;
ArCamera_getTrackingState(ar_session, ar_camera, &tracking_state);

switch (tracking_state) {
case AR_TRACKING_STATE_TRACKING:
    // Pose is valid; render virtual content.
    break;
case AR_TRACKING_STATE_PAUSED:
    // Tracking temporarily lost (occlusion, fast motion, low texture).
    // Continue calling ArSession_update(); do not render virtual content.
    break;
case AR_TRACKING_STATE_STOPPED:
    // Session has been explicitly paused or ARCore service disconnected.
    break;
}
```

### Obtaining view and projection matrices

Every AR render frame begins with acquiring the camera matrices from the current `ArCamera`:

```c
void ar_session_update(ArSession *ar_session, ArFrame *ar_frame)
{
    ArStatus status = ArSession_update(ar_session, ar_frame);
    if (status != AR_SUCCESS) return;

    ArCamera *ar_camera = nullptr;
    ArFrame_acquireCamera(ar_session, ar_frame, &ar_camera);

    /* 4x4 column-major view matrix (world-to-camera).
       Compatible with OpenGL convention: right-hand, -Z forward. */
    float view_mat[16];
    ArCamera_getViewMatrix(ar_session, ar_camera,
        /*out_col_major_4x4=*/view_mat);

    /* 4x4 column-major projection matrix.
       near/far clip planes in metres. */
    float proj_mat[16];
    ArCamera_getProjectionMatrix(ar_session, ar_camera,
        /*near=*/0.1f, /*far=*/100.0f,
        /*dest_col_major_4x4=*/proj_mat);

    /* Acquire camera pose in world space. */
    ArPose *camera_pose = nullptr;
    ArPose_create(ar_session, nullptr, &camera_pose);
    ArCamera_getPose(ar_session, ar_camera, camera_pose);

    float pose_raw[7]; /* [qx, qy, qz, qw, tx, ty, tz] */
    ArPose_getPoseRaw(ar_session, camera_pose, pose_raw);

    ArPose_destroy(camera_pose);
    ArCamera_release(ar_camera);
}
```

[Source: ARCore C API reference — ArCamera](https://developers.google.com/ar/reference/c/group/ar-camera)

`ArCamera_getViewMatrix()` returns the standard OpenGL column-major view matrix with a right-handed coordinate system (+Y up, -Z into screen). `ArCamera_getProjectionMatrix()` accounts for the physical sensor aspect ratio and is pre-corrected for the display orientation set via `ArSession_setDisplayGeometry()`.

---

## 6. ARCore GPU Pipeline

### Camera texture: GL_TEXTURE_EXTERNAL_OES

The zero-copy camera background path uses the OpenGL ES external texture extension. Before the first frame:

```c
/* Create and bind an external texture for the camera stream. */
GLuint camera_texture_id;
glGenTextures(1, &camera_texture_id);
glBindTexture(GL_TEXTURE_EXTERNAL_OES, camera_texture_id);
glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

/* Register the texture with ARCore. From this point, ArSession_update()
   will update the EGLImageKHR bound to this texture with each new camera frame. */
ArSession_setCameraTextureName(ar_session, camera_texture_id);
```

[Source: ARCore C API — ArSession_setCameraTextureName](https://developers.google.com/ar/reference/c/group/ar-session#arsession_setcameratexturename)

Internally, ARCore wraps the gralloc buffer from the Camera HAL into an `EGLImageKHR` via `EGL_ANDROID_image_native_buffer`, then calls `glEGLImageTargetTexture2DOES()` to bind that image to the external texture. The texture update occurs inside `ArSession_update()` — the CPU does not touch the pixel data; the ISP-filled DMA-BUF is transferred directly to the GPU texture unit.

### Camera background fragment shader

The external texture requires a non-standard GLSL sampler:

```glsl
/* Camera background fragment shader (GLSL ES 3.00) */
#version 300 es
#extension GL_OES_EGL_image_external_essl3 : require

precision mediump float;

uniform samplerExternalOES u_CameraColorTexture;
in vec2 v_TexCoord;
out vec4 o_FragColor;

void main()
{
    /* The external texture handles YUV→RGB conversion on the GPU.
       No explicit colour space conversion needed. */
    o_FragColor = texture(u_CameraColorTexture, v_TexCoord);
}
```

### Full OpenGL ES AR render loop

```c
void OnDrawFrame(ArSession *ar_session, ArFrame *ar_frame,
                 GLuint camera_texture_id,
                 GLuint shader_program)
{
    /* Step 1: Update ARCore session. This posts the latest camera buffer
               to camera_texture_id and advances the tracking state. */
    if (ArSession_update(ar_session, ar_frame) != AR_SUCCESS)
        return;

    /* Step 2: Retrieve camera matrices. */
    ArCamera *ar_camera = nullptr;
    ArFrame_acquireCamera(ar_session, ar_frame, &ar_camera);

    float view[16], proj[16];
    ArCamera_getViewMatrix(ar_session, ar_camera, view);
    ArCamera_getProjectionMatrix(ar_session, ar_camera,
        0.1f, 100.0f, proj);

    ArTrackingState tracking_state;
    ArCamera_getTrackingState(ar_session, ar_camera, &tracking_state);

    /* Step 3: Draw camera background (full-screen quad). */
    glDepthMask(GL_FALSE);
    glUseProgram(background_shader_program);
    glBindTexture(GL_TEXTURE_EXTERNAL_OES, camera_texture_id);
    glUniform1i(glGetUniformLocation(background_shader_program,
        "u_CameraColorTexture"), 0);
    DrawFullscreenQuad();
    glDepthMask(GL_TRUE);

    /* Step 4: If tracking, draw virtual objects. */
    if (tracking_state == AR_TRACKING_STATE_TRACKING) {
        /* Build MVP: model matrix from anchor pose × view × proj */
        float model[16];
        ArPose *anchor_pose = GetAnchorPose(ar_session);
        ArPose_getMatrix(ar_session, anchor_pose, model);

        float mv[16], mvp[16];
        MatrixMultiply(view, model, mv);
        MatrixMultiply(proj, mv, mvp);

        glUseProgram(object_shader_program);
        glUniformMatrix4fv(
            glGetUniformLocation(object_shader_program, "u_MVP"),
            1, GL_FALSE, mvp);
        DrawVirtualObject();
    }

    ArCamera_release(ar_camera);
}
```

### Vulkan camera path via AHardwareBuffer

On the Vulkan path, the camera frame arrives as an `AHardwareBuffer` exposed through ARCore's `Frame.getHardwareBuffer()` (Java) or by enabling `TextureUpdateMode.EXPOSE_HARDWARE_BUFFER`. The import sequence:

```c
/* Import AHardwareBuffer into Vulkan as a YUV image. */

AHardwareBuffer *hw_buffer = GetCameraHardwareBuffer();

/* Step 1: Query Vulkan memory requirements for this AHardwareBuffer. */
VkAndroidHardwareBufferPropertiesANDROID buffer_props = {
    .sType = VK_STRUCTURE_TYPE_ANDROID_HARDWARE_BUFFER_PROPERTIES_ANDROID,
};
vkGetAndroidHardwareBufferPropertiesANDROID(device, hw_buffer, &buffer_props);

/* Step 2: Allocate VkDeviceMemory backed by the AHardwareBuffer. */
VkImportAndroidHardwareBufferInfoANDROID import_info = {
    .sType  = VK_STRUCTURE_TYPE_IMPORT_ANDROID_HARDWARE_BUFFER_INFO_ANDROID,
    .buffer = hw_buffer,
};
VkMemoryAllocateInfo alloc_info = {
    .sType           = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
    .pNext           = &import_info,
    .allocationSize  = buffer_props.allocationSize,
    .memoryTypeIndex = FindMemoryType(buffer_props.memoryTypeBits,
                                      VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT),
};
VkDeviceMemory camera_memory;
vkAllocateMemory(device, &alloc_info, nullptr, &camera_memory);

/* Step 3: Create VkImage for the camera buffer's format (YCbCr_420_888). */
VkExternalFormatANDROID external_format = {
    .sType              = VK_STRUCTURE_TYPE_EXTERNAL_FORMAT_ANDROID,
    .externalFormat     = buffer_props.format.externalFormat,
};
VkImageCreateInfo image_ci = {
    .sType     = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .pNext     = &external_format,
    .imageType = VK_IMAGE_TYPE_2D,
    .format    = VK_FORMAT_UNDEFINED, /* Required for external format */
    .extent    = { camera_width, camera_height, 1 },
    .mipLevels = 1, .arrayLayers = 1, .samples = VK_SAMPLE_COUNT_1_BIT,
    .tiling    = VK_IMAGE_TILING_OPTIMAL,
    .usage     = VK_IMAGE_USAGE_SAMPLED_BIT,
};
VkImage camera_image;
vkCreateImage(device, &image_ci, nullptr, &camera_image);
vkBindImageMemory(device, camera_image, camera_memory, 0);
```

[Source: `VK_ANDROID_external_memory_android_hardware_buffer` extension spec](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ANDROID_external_memory_android_hardware_buffer.html)

Sampling a YCbCr external image in Vulkan requires a `VkSamplerYcbcrConversion` object that converts the planar YUV data to RGB in the sampler unit, matching the format's `VkAndroidHardwareBufferFormatPropertiesANDROID.suggestedYcbcrModel` and `suggestedYcbcrRange`. See Ch86 for the complete YCbCr sampler setup.

---

## 7. Android XR Platform

### What Android XR is

Android XR is Google's spatial computing operating system layer, built on Android 15 and above, targeting head-mounted devices (headsets and glasses). It was announced at Google I/O 2024, entered developer preview in late 2024, and shipped on its first device — the Samsung Galaxy XR (formerly Samsung Project Moohan) — in October 2025.

Android XR is not a fork of Android but a set of extensions on top of standard Android: an integrated OpenXR 1.0 runtime, a display compositor that can blend passthrough camera feeds with virtual 3D content, a system-level 6DoF tracking service, and the Jetpack XR SDK (`androidx.xr.*`) for Kotlin/Java app development. Standard Android apps (phone-form-factor APKs) run in a compatibility layer, displayed on a virtual 2D flat panel floating in the 3D scene.

### Samsung Galaxy XR hardware (Project Moohan)

The Samsung Galaxy XR, shipping October 2025 at US$1,799, is the reference hardware for Android XR:

| Component | Specification |
|---|---|
| SoC | Qualcomm Snapdragon XR2+ Gen 2 |
| RAM | 16 GB LPDDR5X |
| Storage | 256 GB UFS |
| Displays | Dual 3552×3840 Micro-OLED per eye |
| Tracking | 6DoF inside-out, hand tracking, eye tracking |
| Passthrough | Full-colour stereo camera passthrough |
| Weight | 545 g headset + 302 g external battery pack |

The Snapdragon XR2+ Gen 2 pairs the Adreno 740 GPU (supporting Vulkan 1.3, OpenXR via proprietary extensions, and VK_EXT_fragment_density_map for foveated rendering) with dedicated XR coprocessors for pose prediction and sensor fusion.

[Source: Road to VR — Samsung Galaxy XR specs](https://roadtovr.com/samsung-galaxy-xr-headset-price-specs-release-date/)

### Jetpack XR SDK

The Jetpack XR SDK (`androidx.xr.*`) provides Kotlin/Java APIs for building spatial apps without writing OpenXR C code directly:

```kotlin
import androidx.xr.runtime.Session
import androidx.xr.runtime.SessionCreateSuccess
import androidx.xr.scenecore.GltfModel
import androidx.xr.scenecore.GltfModelEntity
import androidx.xr.scenecore.SpatialCapabilities

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Create the XR session
        when (val result = Session.create(this)) {
            is SessionCreateSuccess -> {
                val xrSession = result.session
                setupSpatialContent(xrSession)
            }
            else -> {
                // Device does not support Android XR
                fallbackTo2DUI()
            }
        }
    }

    private fun setupSpatialContent(session: Session) {
        val scene = session.scene

        // Check spatial capabilities at runtime
        val caps: SpatialCapabilities = scene.spatialCapabilities
        if (caps.hasCapability(SpatialCapabilities.SPATIAL_3D_CONTENT)) {
            // Load and place a glTF 3D model
            val modelFuture = GltfModel.create(session, "models/robot.glb")
            modelFuture.thenAccept { model ->
                val entity = GltfModelEntity.create(session, model)
                entity.setParent(scene.activitySpace)
                entity.setPose(Pose(Vector3(0f, 0f, -2f), Quaternion.Identity))
                entity.startAnimation(loop = true)
            }
        }
    }
}
```

[Source: Android Developers — Access a session](https://developer.android.com/develop/xr/jetpack-xr-sdk/add-session)

`SpatialCapabilities` capabilities include `SPATIAL_3D_CONTENT`, `APP_ENVIRONMENT`, `EMBED_ACTIVITY`, `PASSTHROUGH_CONTROL`, `SPATIAL_AUDIO`, and `SPATIAL_UI`. Apps should check capabilities dynamically because the same APK may run on a phone (no spatial capabilities) or on an Android XR headset.

### Jetpack Compose for XR

For declarative UI, `androidx.xr.compose` extends Jetpack Compose with spatial composables:

```kotlin
import androidx.xr.compose.subspace.SpatialPanel
import androidx.xr.compose.subspace.Volume
import androidx.xr.compose.spatial.Orbiter

@Composable
fun SpatialUI() {
    Subspace {
        // A 2D panel floating in 3D space
        SpatialPanel(
            modifier = SubspaceModifier.width(800.dp).height(600.dp)
        ) {
            MyApp2DContent()
        }

        // A 3D volume for glTF content placed relative to the panel
        Volume(
            modifier = SubspaceModifier.offset(z = (-0.5f).dp)
        ) {
            // SceneCore entities placed here
        }
    }
}
```

[Source: Android Developers — Spatial UI with Compose for XR](https://developer.android.com/develop/xr/jetpack-xr-sdk/ui-compose)

---

## 8. OpenXR on Android XR

Android XR exposes a full OpenXR 1.0 runtime for native C/C++ XR apps. The runtime ships as part of the Android XR system image and is accessed via the standard OpenXR loader, initialized with Android-specific structs.

### Instance creation

On Android, the OpenXR loader requires Android VM context before `xrCreateInstance()`. The `XR_KHR_android_create_instance` extension provides the required struct:

```c
#include <openxr/openxr.h>
#include <openxr/openxr_platform.h>

/* Step 1: Initialize the OpenXR loader with Android JVM context.
   Must be called before any other OpenXR function. */
PFN_xrInitializeLoaderKHR xrInitializeLoaderKHR = nullptr;
xrGetInstanceProcAddr(XR_NULL_HANDLE, "xrInitializeLoaderKHR",
    (PFN_xrVoidFunction*)&xrInitializeLoaderKHR);

XrLoaderInitInfoAndroidKHR loader_info = {
    .type               = XR_TYPE_LOADER_INIT_INFO_ANDROID_KHR,
    .next               = nullptr,
    .applicationVM      = app->activity->vm,       /* JavaVM* */
    .applicationContext = app->activity->clazz,    /* jobject Activity */
};
xrInitializeLoaderKHR((XrLoaderInitInfoBaseHeaderKHR*)&loader_info);

/* Step 2: Chain XrInstanceCreateInfoAndroidKHR into XrInstanceCreateInfo. */
XrInstanceCreateInfoAndroidKHR instance_android = {
    .type                = XR_TYPE_INSTANCE_CREATE_INFO_ANDROID_KHR,
    .next                = nullptr,
    .applicationVM       = app->activity->vm,
    .applicationActivity = app->activity->clazz,
};

const char *enabled_extensions[] = {
    XR_KHR_ANDROID_CREATE_INSTANCE_EXTENSION_NAME,  /* "XR_KHR_android_create_instance" */
    XR_KHR_VULKAN_ENABLE2_EXTENSION_NAME,
    XR_EXT_HAND_TRACKING_EXTENSION_NAME,
    XR_ANDROID_EYE_TRACKING_EXTENSION_NAME,         /* "XR_ANDROID_eye_tracking" */
    XR_FB_PASSTHROUGH_EXTENSION_NAME,
    XR_FB_FOVEATION_EXTENSION_NAME,
    XR_FB_FOVEATION_VULKAN_EXTENSION_NAME,
};

XrInstanceCreateInfo create_info = {
    .type                      = XR_TYPE_INSTANCE_CREATE_INFO,
    .next                      = &instance_android,  /* chain the android struct */
    .createFlags               = 0,
    .applicationInfo           = {
        .applicationName    = "MyXRApp",
        .applicationVersion = 1,
        .engineName         = "MyEngine",
        .engineVersion      = 1,
        .apiVersion         = XR_CURRENT_API_VERSION,
    },
    .enabledApiLayerCount      = 0,
    .enabledExtensionCount     = ARRAY_SIZE(enabled_extensions),
    .enabledExtensionNames     = enabled_extensions,
};

XrInstance xr_instance;
xrCreateInstance(&create_info, &xr_instance);
```

[Source: XrInstanceCreateInfoAndroidKHR specification](https://registry.khronos.org/OpenXR/specs/0.90/man/html/XrInstanceCreateInfoAndroidKHR.html)

Note: the field is named `applicationActivity` (not `applicationContext`) in the OpenXR spec. The struct chains into `XrInstanceCreateInfo.next` using the standard OpenXR next-chain mechanism.

### Vulkan graphics binding

```c
/* Step 3: Create XrSession with Vulkan graphics binding. */

/* Obtain Vulkan requirements from the OpenXR runtime. */
XrGraphicsRequirementsVulkan2KHR vk_reqs = {
    .type = XR_TYPE_GRAPHICS_REQUIREMENTS_VULKAN2_KHR
};
PFN_xrGetVulkanGraphicsRequirements2KHR getReqs;
xrGetInstanceProcAddr(xr_instance, "xrGetVulkanGraphicsRequirements2KHR",
    (PFN_xrVoidFunction*)&getReqs);
getReqs(xr_instance, xr_system_id, &vk_reqs);
/* vk_reqs.minApiVersionSupported and maxApiVersionSupported bound Vulkan version. */

/* Build Vulkan device (requires extensions reported by OpenXR runtime). */
/* ... VkInstance, VkPhysicalDevice, VkDevice creation ... */

/* Bind the Vulkan device to the OpenXR session. */
XrGraphicsBindingVulkan2KHR vk_binding = {
    .type               = XR_TYPE_GRAPHICS_BINDING_VULKAN2_KHR,
    .next               = nullptr,
    .instance           = vk_instance,
    .physicalDevice     = vk_physical_device,
    .device             = vk_device,
    .queueFamilyIndex   = graphics_queue_family,
    .queueIndex         = 0,
};

XrSessionCreateInfo session_info = {
    .type        = XR_TYPE_SESSION_CREATE_INFO,
    .next        = &vk_binding,
    .createFlags = 0,
    .systemId    = xr_system_id,
};

XrSession xr_session;
xrCreateSession(xr_instance, &session_info, &xr_session);
```

### Render loop

The canonical OpenXR frame loop on Android XR follows the standard OpenXR pattern:

```c
void RenderLoop(XrSession xr_session, XrSwapchain swapchain)
{
    bool exit_render_loop = false;

    while (!exit_render_loop) {
        /* Step 1: Process OpenXR events (session state changes). */
        XrEventDataBuffer event = { .type = XR_TYPE_EVENT_DATA_BUFFER };
        while (xrPollEvent(xr_instance, &event) == XR_SUCCESS) {
            if (event.type == XR_TYPE_EVENT_DATA_SESSION_STATE_CHANGED) {
                auto *state_event = (XrEventDataSessionStateChanged*)&event;
                if (state_event->state == XR_SESSION_STATE_READY) {
                    XrSessionBeginInfo begin_info = {
                        .type                         = XR_TYPE_SESSION_BEGIN_INFO,
                        .primaryViewConfigurationType =
                            XR_VIEW_CONFIGURATION_TYPE_PRIMARY_STEREO,
                    };
                    xrBeginSession(xr_session, &begin_info);
                } else if (state_event->state == XR_SESSION_STATE_STOPPING) {
                    xrEndSession(xr_session);
                    exit_render_loop = true;
                }
            }
        }

        /* Step 2: Wait for the next frame (blocks until optimal render time). */
        XrFrameWaitInfo wait_info = { .type = XR_TYPE_FRAME_WAIT_INFO };
        XrFrameState frame_state = { .type = XR_TYPE_FRAME_STATE };
        xrWaitFrame(xr_session, &wait_info, &frame_state);

        /* Step 3: Begin the frame. */
        XrFrameBeginInfo begin_frame = { .type = XR_TYPE_FRAME_BEGIN_INFO };
        xrBeginFrame(xr_session, &begin_frame);

        /* Step 4: Locate views (per-eye pose and FOV). */
        XrView views[2] = {{ .type = XR_TYPE_VIEW }, { .type = XR_TYPE_VIEW }};
        XrViewLocateInfo locate_info = {
            .type                  = XR_TYPE_VIEW_LOCATE_INFO,
            .viewConfigurationType = XR_VIEW_CONFIGURATION_TYPE_PRIMARY_STEREO,
            .displayTime           = frame_state.predictedDisplayTime,
            .space                 = xr_stage_space,
        };
        XrViewState view_state = { .type = XR_TYPE_VIEW_STATE };
        uint32_t view_count = 0;
        xrLocateViews(xr_session, &locate_info, &view_state,
            2, &view_count, views);

        /* Step 5: Render each eye into the swapchain. */
        XrSwapchainImageAcquireInfo acquire_info = {
            .type = XR_TYPE_SWAPCHAIN_IMAGE_ACQUIRE_INFO
        };
        uint32_t image_index;
        xrAcquireSwapchainImage(swapchain, &acquire_info, &image_index);

        XrSwapchainImageWaitInfo wait_image = {
            .type    = XR_TYPE_SWAPCHAIN_IMAGE_WAIT_INFO,
            .timeout = XR_INFINITE_DURATION,
        };
        xrWaitSwapchainImage(swapchain, &wait_image);

        RenderStereoEyes(views, image_index); /* app render function */

        XrSwapchainImageReleaseInfo release_info = {
            .type = XR_TYPE_SWAPCHAIN_IMAGE_RELEASE_INFO
        };
        xrReleaseSwapchainImage(swapchain, &release_info);

        /* Step 6: End the frame, submitting the rendered layers. */
        XrCompositionLayerProjectionView proj_views[2] = { /* ... */ };
        XrCompositionLayerProjection layer = {
            .type      = XR_TYPE_COMPOSITION_LAYER_PROJECTION,
            .space     = xr_stage_space,
            .viewCount = 2,
            .views     = proj_views,
        };
        const XrCompositionLayerBaseHeader *layers[] = {
            (XrCompositionLayerBaseHeader*)&layer
        };
        XrFrameEndInfo end_info = {
            .type                = XR_TYPE_FRAME_END_INFO,
            .displayTime         = frame_state.predictedDisplayTime,
            .environmentBlendMode= XR_ENVIRONMENT_BLEND_MODE_ALPHA_BLEND,
            .layerCount          = 1,
            .layers              = layers,
        };
        xrEndFrame(xr_session, &end_info);
    }
}
```

### Foveated rendering

Android XR supports foveated rendering via `XR_FB_foveation` and `XR_FB_foveation_vulkan`:

```c
/* Create swapchain with foveated rendering hints. */
XrSwapchainCreateInfoFoveationFB foveation_swapchain_info = {
    .type  = XR_TYPE_SWAPCHAIN_CREATE_INFO_FOVEATION_FB,
    .flags = XR_SWAPCHAIN_CREATE_FOVEATION_SCALED_BIN_BIT_FB,
};
XrSwapchainCreateInfo swapchain_info = {
    .type   = XR_TYPE_SWAPCHAIN_CREATE_INFO,
    .next   = &foveation_swapchain_info,
    /* ... width, height, format, etc. */
};
xrCreateSwapchain(xr_session, &swapchain_info, &swapchain);

/* Apply foveation profile with gaze-tracked density. */
XrFoveationLevelProfileCreateInfoFB level_profile = {
    .type        = XR_TYPE_FOVEATION_LEVEL_PROFILE_CREATE_INFO_FB,
    .level       = XR_FOVEATION_LEVEL_HIGH_FB,
    .verticalOffset = 0.0f,
    .dynamic     = XR_FOVEATION_DYNAMIC_LEVEL_ENABLED_FB,
};
XrFoveationProfileFB foveation_profile;
PFN_xrCreateFoveationProfileFB createProfile;
xrGetInstanceProcAddr(xr_instance, "xrCreateFoveationProfileFB",
    (PFN_xrVoidFunction*)&createProfile);
createProfile(xr_session, &level_profile, nullptr, &foveation_profile);
```

On the Adreno 740 GPU in the Galaxy XR, the underlying mechanism is `VK_EXT_fragment_density_map` (the Vulkan API for subsampled shading rates), which reduces pixel-shader invocations in the peripheral visual field based on the gaze vector from `XR_ANDROID_eye_tracking`. The older OpenGL ES `GL_QCOM_texture_foveated` extension serves a similar purpose on GLES pipelines and predates the Vulkan standardised path.

### Hand tracking and passthrough

```c
/* Hand tracking via XR_EXT_hand_tracking */
XrHandTrackerCreateInfoEXT hand_create = {
    .type = XR_TYPE_HAND_TRACKER_CREATE_INFO_EXT,
    .hand = XR_HAND_LEFT_EXT,
    .handJointSet = XR_HAND_JOINT_SET_DEFAULT_EXT,
};
XrHandTrackerEXT left_hand_tracker;
PFN_xrCreateHandTrackerEXT createHandTracker;
xrGetInstanceProcAddr(xr_instance, "xrCreateHandTrackerEXT",
    (PFN_xrVoidFunction*)&createHandTracker);
createHandTracker(xr_session, &hand_create, &left_hand_tracker);

/* Locate joint poses each frame */
XrHandJointLocationEXT joint_locations[XR_HAND_JOINT_COUNT_EXT];
XrHandJointLocationsEXT locations = {
    .type          = XR_TYPE_HAND_JOINT_LOCATIONS_EXT,
    .jointCount    = XR_HAND_JOINT_COUNT_EXT,
    .jointLocations = joint_locations,
};
XrHandJointsLocateInfoEXT locate_hand = {
    .type        = XR_TYPE_HAND_JOINTS_LOCATE_INFO_EXT,
    .baseSpace   = xr_stage_space,
    .time        = frame_state.predictedDisplayTime,
};
PFN_xrLocateHandJointsEXT locateHands;
xrGetInstanceProcAddr(xr_instance, "xrLocateHandJointsEXT",
    (PFN_xrVoidFunction*)&locateHands);
locateHands(left_hand_tracker, &locate_hand, &locations);
```

Passthrough — compositing the real-world camera feed as the environment blend behind virtual content — is configured via `XR_FB_passthrough` (using `XR_ENVIRONMENT_BLEND_MODE_ALPHA_BLEND` in `xrEndFrame`), or via the Android XR-specific `XR_ANDROID_passthrough_camera_state` extension for finer passthrough control.

[Source: Android Developers — Supported OpenXR extensions](https://developer.android.com/develop/xr/openxr/extensions)

---

## 9. Snapdragon Spaces: Qualcomm's AR SDK

### Overview

Snapdragon Spaces is Qualcomm's Adreno-optimised AR SDK targeting Snapdragon XR platforms (standalone headsets and tethered glasses). It wraps OpenXR with a Unity and Unreal Engine integration layer and provides depth, hand tracking, eye tracking, plane detection, and image recognition on validated Snapdragon XR hardware. Supported targets include the Lenovo ThinkReality A3, Motorola Edge+ AR glasses, and third-party headsets using Snapdragon XR2+ and XR2 Gen 2.

[Source: Snapdragon Spaces developer portal](https://spaces.qualcomm.com/developer/ar-sdk/)

### Feature comparison with ARCore

| Feature | ARCore (phone) | Snapdragon Spaces (headset/glasses) |
|---|---|---|
| Tracking | 6DoF VIO | 6DoF + dedicated XR tracking coprocessor |
| Depth | MotionStereo or ToF | ToF point cloud via `SpacesDepthProvider` |
| Plane detection | ARCore SLAM | OpenXR `XR_EXT_plane_detection` |
| Hand tracking | None (phone) | `XR_EXT_hand_tracking` |
| Eye tracking | None (phone) | `XR_EXT_eye_gaze_interaction` |
| API surface | `ArSession` C API + Camera2 | OpenXR 1.0 extensions via Unity/Unreal plugin |
| Target form factor | Android smartphones | XR headsets and AR glasses |

### Depth provider

Snapdragon Spaces exposes depth sensor data via the `SpacesDepthProvider` Unity component, which wraps an OpenXR depth extension. The raw output is a point cloud in device space, updated at the ToF sensor framerate (typically 5–30 Hz). On the Unity side:

```csharp
// Snapdragon Spaces — Unity C# depth provider usage (representative pattern;
// consult the official Snapdragon Spaces Unity SDK API reference for exact
// member names, event signatures, and NativeArray element types)
using Qualcomm.Snapdragon.Spaces;

public class DepthVisualizer : MonoBehaviour
{
    private SpacesDepthProvider _depthProvider;

    void Start()
    {
        _depthProvider = GetComponent<SpacesDepthProvider>();
        _depthProvider.onDepthFrameReceived.AddListener(OnDepthFrame);
    }

    void OnDepthFrame(SpacesDepthFrame frame)
    {
        // Point cloud in device local space (metres) and per-point confidence
        VisualizePointCloud(frame.Points, frame.Confidence);
    }
}
```

> **Note:** The exact property names (`Points`, `Confidence`) and the Unity event API surface (`onDepthFrameReceived`) for `SpacesDepthProvider` should be verified against the installed Snapdragon Spaces SDK package. The public docs.spaces.qualcomm.com API reference pages for this class were unavailable at time of writing.

### Foveated rendering on Adreno

The Adreno GPU's foveated rendering capability exposes two paths:

- **OpenGL ES**: `GL_QCOM_texture_foveated` — an extension that designates specific texture attachments as foveated, reducing texel resolution in peripheral screen regions. This is a GLES-only extension predating Vulkan standardisation ([Khronos OpenGL registry](https://registry.khronos.org/OpenGL/extensions/QCOM/QCOM_texture_foveated.txt)).
- **Vulkan**: `VK_EXT_fragment_density_map` — the cross-vendor standard Vulkan extension for variable shading rate. The Adreno 740 driver exposes this extension, enabling foveated rendering in OpenXR Vulkan swapchains via the `XR_FB_foveation_vulkan` extension layer.

Eye-tracked foveated rendering (ETF) combines `XR_ANDROID_eye_tracking` gaze data with `XR_FB_foveation` density maps: the compositor updates the fragment density map attachment each frame with the current gaze fixation point, directing full-resolution shading to the fovea (typically 5–8° visual angle) and reducing it toward the periphery.

### Snapdragon Spaces vs ARCore

Snapdragon Spaces targets a different hardware tier than phone ARCore. ARCore runs on 1 billion+ phone SKUs with conservative sensor requirements; Snapdragon Spaces assumes dedicated XR sensor suites (ToF, IR cameras for hand tracking, eye tracking cameras) available only on validated Snapdragon XR headset designs. The two SDKs are not directly interchangeable, though both expose OpenXR extensions for the tracking primitives they share.

---

## 10. Linux AR: Monado, SteamVR, and the Open Stack

### Monado OpenXR runtime

**Monado** (`https://monado.freedesktop.org`) is the open-source cross-platform OpenXR runtime, developed under the FreeDesktop umbrella and maintained by Collabora. It runs on Linux, Windows, and Android, and implements the OpenXR 1.0 core plus a growing set of extensions.

On Linux, Monado integrates with:
- **libcamera** (Ch96) — V4L2-backed camera pipeline for SLAM tracker input on devices without ARKit-style sensors.
- **libsurvive** — SteamVR Lighthouse base-station tracking.
- **realsense** (Intel RealSense ToF cameras) — depth and RGB streams for 6DoF SLAM.
- **Northstar** and **ALVR** — open headset platforms for which Monado is the sole XR runtime.

Monado's driver architecture is modular: each XR device (headset, controller, tracker) implements a `xrt_device` interface, and each input source (IMU, camera, USB HID) implements an `xrt_prober` driver. The AR/SLAM tracker subsystem (`t_imu.h`, `t_camera_slam.h`) wraps Basalt SLAM or OpenVINS as a pluggable VIO backend, mirroring ARCore's EKF/factor-graph approach on open hardware.

[Source: Monado GitLab](https://gitlab.freedesktop.org/monado/monado)

### SteamVR on Linux and DRM leases

SteamVR for Linux (via Proton's VR path) requires direct display access to the headset's display to achieve the low-latency, tearing-free rendering that VR demands. This is achieved via **DRM leases** (`wp_drm_lease_device_v1`, covered in Ch121): the compositor leases the headset's DRM connector and CRTC directly to the SteamVR compositor process, bypassing SurfaceFlinger/Wayland composition for the headset display while the desktop compositor continues managing the host monitor. [Source: Ch121 — DRM Leases for VR Headset Direct Mode]

### ALVR and WiVRn: wireless VR streaming

**ALVR** (Air Light VR) and **WiVRn** stream SteamVR content from a Linux PC to a standalone headset (Meta Quest, Pico) over WiFi. The Linux-side component encodes the rendered framebuffer with VA-API (H.264/HEVC/AV1 hardware encode; Ch26) and streams over UDP, while the headset-side client decodes and presents on the display. The result is SteamVR-compatible 6DoF tracking and rendering without a wired tether, using Monado as the OpenXR runtime on the Linux host.

### libcamera as a Linux Camera HAL equivalent

Linux has no Camera HAL3 equivalent in the Android sense, but **libcamera** (Ch96) fills the same architectural role: it abstracts camera hardware (V4L2 sensors, ISPs) behind a unified C++ API (`libcamera::Camera`, `libcamera::Stream`, `libcamera::Request`), handles ISP parameter tuning via IPA (Image Processing Algorithm) plugins, and delivers frames as DMA-BUF `libcamera::FrameBuffer` objects. Monado's `libcamera` driver passes these frames to its SLAM tracker exactly as ARCore's tracking thread would consume Camera2 frames.

An ARCore-equivalent AR experience on Linux would require:

1. **libcamera** for camera access and frame delivery (replacing Camera2).
2. An IMU input driver via `iio-sensor-proxy` or direct `/dev/iio:deviceN` reads (replacing Android `SensorManager`).
3. **Monado** as the OpenXR runtime hosting a VIO SLAM tracker (replacing ARCore's proprietary VIO).
4. The app's GPU renderer using standard OpenXR swapchain images (same as Android XR).

This stack exists and works on devices like the Raspberry Pi with a connected IMU and CSI camera, but is far from the polished OEM-validated experience of ARCore on Android.

### Asahi Linux and the hardware constraint

The Asahi Linux project (Ch73) demonstrates that new ARM SoC support on Linux requires extensive reverse engineering of proprietary firmware interfaces. The same constraint applies to AR: Apple Neural Engine acceleration for ARKit's depth and semantics models, Samsung Hexagon DSP code for ARCore's MotionStereo, and Qualcomm's XR tracking coprocessor firmware are all proprietary. An open-source AR stack on the same hardware is possible at a functional level (CPU-side VIO, software depth estimation) but cannot match the power efficiency or throughput of hardware-accelerated counterparts without open firmware or vendor cooperation.

The path forward for Linux AR follows two tracks: open hardware (Project North Star, ALVR-connected Quest) where Monado has full access, and vendor cooperation (libcamera IPA plugins from Sony, NXP) that exposes ISP tuning without requiring full firmware reversal.

---

## 11. Integrations

**Ch27 — OpenXR Runtime Architecture on Linux**: ARCore's `ArSession` lifecycle (create → resume → update loop → pause → destroy) maps directly to the OpenXR `XrSession` state machine (IDLE → READY → SYNCHRONIZED → VISIBLE → FOCUSED → STOPPING → EXITING). Android XR makes this correspondence explicit: the Jetpack XR `Session` wraps an `XrSession`, and every ARCore tracking primitive (planes, anchors, depth) has an OpenXR extension analogue (`XR_ANDROID_trackables`, `XR_ANDROID_depth_texture`). Monado on Linux implements the same `XrSession` state machine for open hardware.

**Ch85 — SurfaceFlinger and Android Buffer Pipeline**: Camera buffers follow the SurfaceFlinger buffer queue path before reaching ARCore: the Camera HAL writes into gralloc-backed `AHardwareBuffer` objects managed by `BufferQueue`. The `Surface` that ARCore registers as a camera output target is backed by a `BufferQueue` consumer. SurfaceFlinger does not composite AR overlay frames — the AR app renders into its own `Surface` presented via `ANativeWindow`/`EGLSurface` — but the camera feed's buffer lifecycle is tied to SurfaceFlinger's buffer queue machinery.

**Ch86 — Vulkan on Android**: The `VK_ANDROID_external_memory_android_hardware_buffer` extension used in Section 6 of this chapter for camera frame import is the same extension used for all cross-process buffer sharing on Android (Ch86, Section 6). The YCbCr sampler conversion (`VkSamplerYcbcrConversion`) required for camera images is described in detail in Ch86 Section 6, including the `VkAndroidHardwareBufferFormatPropertiesANDROID.suggestedYcbcrModel` query.

**Ch73 — Asahi Linux**: The firmware-constrained situation for Linux AR on Apple Silicon mirrors the challenge Asahi faces for GPU acceleration: both require either open firmware specifications or binary blob extraction. The Asahi project's AGX driver work demonstrates that full GPU stack reversal is achievable over years of effort; an equivalent "ARKit on Linux" effort would require the same approach for the Neural Engine and ISP coprocessors.

**Ch96 — libcamera and the Linux Camera Stack**: libcamera's role as the Linux equivalent of Android Camera HAL3 is direct: both manage camera pipelines via request/result pairs, buffer queues, and pluggable ISP parameter tuning. Monado's libcamera driver for Linux AR uses exactly the same `libcamera::Request` → `FrameBuffer` → DMA-BUF path that Android's Camera HAL → gralloc → `AHardwareBuffer` path implements. The key gap is synchronisation with an IMU: Android's `SensorManager` guarantees `CLOCK_BOOTTIME` timestamp alignment between camera and IMU; Linux requires manual timestamp reconciliation between `iio-sensor-proxy` and V4L2 frame timestamps.

**Ch121 — DRM Leases for VR Headset Direct Mode**: Monado's direct-display support for VR headsets on Linux uses `wp_drm_lease_device_v1` to request a DRM lease from the Wayland compositor, acquiring exclusive `DRM_MASTER` access to the headset connector. This is the Linux equivalent of Android XR's system compositor handing exclusive display access to the OpenXR runtime for the headset's Micro-OLED panels. Both mechanisms solve the same problem: giving the XR runtime sub-millisecond control over the display refresh timing that VR/AR requires for comfort.

---

*Sources and upstream references used in this chapter:*

- [ARCore C API reference](https://developers.google.com/ar/reference/c)
- [ARCore hello_ar_c sample](https://github.com/google-ar/arcore-android-sdk/blob/main/samples/hello_ar_c/app/src/main/cpp/hello_ar_application.cc)
- [Android Camera HAL3 documentation](https://source.android.com/docs/core/camera/camera3)
- [Android Camera HAL3 buffer management](https://source.android.com/docs/core/camera/buffer-management-api)
- [AOSP Camera HAL AIDL interfaces](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/camera/)
- [VK_ANDROID_external_memory_android_hardware_buffer](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ANDROID_external_memory_android_hardware_buffer.html)
- [Android XR for OpenXR — supported extensions](https://developer.android.com/develop/xr/openxr/extensions)
- [Jetpack XR SDK — Session](https://developer.android.com/develop/xr/jetpack-xr-sdk/add-session)
- [Jetpack XR SDK — Spatial Capabilities](https://developer.android.com/develop/xr/jetpack-xr-sdk/check-spatial-capabilities)
- [Samsung Galaxy XR specs](https://roadtovr.com/samsung-galaxy-xr-headset-price-specs-release-date/)
- [Snapdragon Spaces developer portal](https://spaces.qualcomm.com/developer/ar-sdk/)
- [GL_QCOM_texture_foveated OpenGL extension](https://registry.khronos.org/OpenGL/extensions/QCOM/QCOM_texture_foveated.txt)
- [XrInstanceCreateInfoAndroidKHR specification](https://registry.khronos.org/OpenXR/specs/0.90/man/html/XrInstanceCreateInfoAndroidKHR.html)
- [XR_KHR_android_surface_swapchain](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_KHR_android_surface_swapchain.html)
- [Monado OpenXR runtime](https://gitlab.freedesktop.org/monado/monado)
- [ARCore Vulkan rendering guide](https://developers.google.com/ar/develop/java/vulkan)
