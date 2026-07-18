# Chapter 203: WebXR — Browser-Based Immersive Experiences on Linux

**Audiences**: Browser and web platform engineers building or debugging WebXR applications; systems developers understanding how a browser maps the WebXR Device API onto OpenXR/Monado and the Linux graphics stack.

---

## Table of Contents

1. [What WebXR Is and Why It Exists](#1-what-webxr-is-and-why-it-exists)
2. [Session Types and the Permission Model](#2-session-types-and-the-permission-model)
3. [The WebXR Frame Loop](#3-the-webxr-frame-loop)
4. [Reference Spaces and Spatial Reasoning](#4-reference-spaces-and-spatial-reasoning)
5. [Input Sources: Controllers and Hand Tracking](#5-input-sources-controllers-and-hand-tracking)
6. [Rendering: XRWebGLLayer and the WebXR Layers API](#6-rendering-xrwebgllayer-and-the-webxr-layers-api)
7. [Chromium's WebXR Implementation: Process Model and Mojo](#7-chromiums-webxr-implementation-process-model-and-mojo)
8. [The OpenXR Backend: OpenXrDevice to XrSession](#8-the-openxr-backend-openxrdevice-to-xrsession)
9. [High-Level WebXR Frameworks: A-Frame and Babylon.js](#9-high-level-webxr-frameworks-a-frame-and-babylonjs)
    - [A-Frame: Declarative WebXR with an Entity-Component System](#a-frame-declarative-webxr-with-an-entity-component-system)
    - [A-Frame's Rendering Pipeline and the Linux Stack](#a-frames-rendering-pipeline-and-the-linux-stack)
    - [Babylon.js XR: Imperative High-Performance WebXR](#babylonjs-xr-imperative-high-performance-webxr)
    - [Three.js VRButton / ARButton](#threejs-vrbutton--arbutton)
    - [Framework Comparison and Linux Considerations](#framework-comparison-and-linux-considerations)
10. [WebXR on Linux: Status, Gaps, and the Community Path](#10-webxr-on-linux-status-gaps-and-the-community-path)
11. [Firefox WebXR via wgpu](#11-firefox-webxr-via-wgpu)
12. [WebXR vs. Native OpenXR: Tradeoffs](#12-webxr-vs-native-openxr-tradeoffs)
13. [Roadmap: WebXR Specification and Implementation Trajectory](#13-roadmap-webxr-specification-and-implementation-trajectory)
14. [Integrations](#14-integrations)
15. [References](#15-references)

---

## 1. What WebXR Is and Why It Exists

The **WebXR Device API** is a W3C Living Standard that allows web applications running in a browser to initiate immersive VR and AR sessions, access head and hand tracking data, and render stereo frames to an XR device — all without a native app installation. The user activates an experience by clicking a button on a web page; the browser negotiates with the underlying XR runtime (OpenXR, ARCore, or a platform-specific SDK) and delivers tracking data and rendered frames at the device's native refresh rate (typically 72–120 Hz). [Source: W3C WebXR Device API](https://www.w3.org/TR/webxr/)

WebXR's design contract is deliberately minimal at the JS layer: it provides spatial poses, per-eye projection matrices, and a framebuffer to render into. The browser handles all negotiation with the XR runtime, display presentation, and the security model that prevents a web page from accessing camera pixels or physical room geometry without explicit grants. This contrasts with native OpenXR, where the application directly creates `XrInstance`, drives the full render loop, and has unrestricted access to extension APIs.

On Linux, WebXR sits above the same Monado/SteamVR OpenXR runtimes covered in Chapter 27. The browser translates the JS WebXR surface into native `XrSession` calls through a multi-process architecture using Mojo IPC. As of mid-2026, this path is officially complete only on Windows and Android; Linux requires either a community-patched Chromium build or Firefox's wgpu-backed path. Understanding why illuminates how tightly the rendering path is coupled to the platform's native graphics API.

---

## 2. Session Types and the Permission Model

### Session Modes

WebXR defines three session modes:

| Mode | Display | Tracking | Hardware required |
|---|---|---|---|
| `inline` | Browser canvas | Optional 3DoF/6DoF | None — works without headset |
| `immersive-vr` | HMD, full-screen | 6DoF (required) | VR headset |
| `immersive-ar` | HMD with passthrough | 6DoF + depth (optional) | AR headset or phone camera |

**Inline sessions** render into a canvas element and are available without user activation or headset hardware. They enable the "Magic Window" pattern: tilting a phone or looking around a 3D scene on a desktop monitor using orientation tracking. On Linux, inline sessions work in Chrome (DeviceOrientation API feeds the tracking) even where immersive sessions do not.

**Immersive sessions** take exclusive control of the HMD display, equivalent to Monado's direct-mode path (Chapter 27, Section 4). Only one immersive session per XR device is allowed at a time across the entire browser, and the session can only be requested inside a user gesture handler.

### Feature Negotiation

Session creation accepts required and optional features, decoupling discovery from instantiation:

```javascript
const session = await navigator.xr.requestSession('immersive-vr', {
  requiredFeatures: ['local-floor'],
  optionalFeatures: ['hand-tracking', 'bounded-floor', 'depth-sensing'],
});
```

The browser rejects the promise if any `requiredFeatures` entry is unavailable. `optionalFeatures` that fail are silently dropped — the application must check at runtime whether a feature was granted. This negotiation maps onto OpenXR's extension enumeration: Chromium's `OpenXrApiWrapper` checks which `XrInstance` extensions are present before declaring which WebXR features can be satisfied. `hand-tracking` requires `XR_EXT_hand_tracking`; `depth-sensing` requires `XR_EXT_depth_sensor` or equivalent; `plane-detection` requires `XR_EXT_plane_detection`.

### Availability Check

Before requesting a session, applications check availability:

```javascript
// Check immersive-vr availability
if (navigator.xr) {
  const supported = await navigator.xr.isSessionSupported('immersive-vr');
  if (supported) {
    document.getElementById('vr-button').disabled = false;
  }
}
```

`isSessionSupported` calls into the browser's VR device service, which in turn calls `xrEnumerateSystems` and `xrCreateInstance` to determine whether a compatible OpenXR runtime is present. On Linux without a runtime, this returns `false` synchronously after the device service fails to find a valid `active_runtime.json`.

---

## 3. The WebXR Frame Loop

### XRSession.requestAnimationFrame

WebXR replaces `window.requestAnimationFrame` with `XRSession.requestAnimationFrame` for immersive content. The session drives the callback at the HMD's native refresh rate rather than the monitor refresh rate:

```javascript
function onXRFrame(time, frame) {
  const session = frame.session;

  // Schedule next frame immediately — must happen at top of callback
  session.requestAnimationFrame(onXRFrame);

  // Obtain viewer pose in the reference space
  const pose = frame.getViewerPose(referenceSpace);
  if (!pose) return;  // pose may be null if tracking is lost

  // Bind the XRWebGLLayer framebuffer
  const glLayer = session.renderState.baseLayer;
  gl.bindFramebuffer(gl.FRAMEBUFFER, glLayer.framebuffer);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

  // Render one pass per eye
  for (const view of pose.views) {
    const viewport = glLayer.getViewport(view);
    gl.viewport(viewport.x, viewport.y, viewport.width, viewport.height);

    // view.projectionMatrix: column-major 4x4, left-handed, WebGL-compatible
    // view.transform.matrix: 4x4 camera-to-reference-space transform
    renderEye(view.projectionMatrix, view.transform.inverse.matrix);
  }
}

session.requestAnimationFrame(onXRFrame);
```

**`XRFrame` lifetime**: the frame object is only valid inside the callback. Attempting to call `frame.getViewerPose()` after the callback returns throws a `InvalidStateError`. This enforces that pose data is consumed at the correct timestamp — the predicted display time corresponding to when photons will leave the display.

### XRViewerPose and XRView

`XRViewerPose` wraps the observer's pose in the reference space:
- `pose.transform` — `XRRigidTransform` with `position` (DOMPointReadOnly) and `orientation` (quaternion DOMPointReadOnly)
- `pose.views` — array of `XRView` objects, one per rendered eye for immersive-vr (always two), one for inline

Each `XRView`:
- `view.eye` — `"left"`, `"right"`, or `"none"` for mono
- `view.projectionMatrix` — 4×4 column-major projection matrix. **Must not be modified.** The runtime adjusts this for IPD, lens distortion, and display geometry.
- `view.transform` — `XRRigidTransform` placing the eye in reference space; invert to get the view matrix

The application provides no IPD or field-of-view values — these come from the headset calibration via the OpenXR `XrView` structs populated by `xrLocateViews`, which Chromium's `OpenXrApiWrapper` queries each frame and transcribes into the `XRFrameData` Mojo struct.

---

## 4. Reference Spaces and Spatial Reasoning

Reference spaces are coordinate frames in which poses are meaningful. They map directly onto OpenXR's `XrReferenceSpaceType`:

| WebXR | OpenXR equivalent | Origin |
|---|---|---|
| `viewer` | `XR_REFERENCE_SPACE_TYPE_VIEW` | Headset itself |
| `local` | `XR_REFERENCE_SPACE_TYPE_LOCAL` | Head position at session start |
| `local-floor` | `XR_REFERENCE_SPACE_TYPE_LOCAL_FLOOR` | Floor level below head at start |
| `bounded-floor` | `XR_REFERENCE_SPACE_TYPE_STAGE` | Centre of guardian boundary |
| `unbounded` | `XR_REFERENCE_SPACE_TYPE_UNBOUNDED_MSFT` | Large-space tracking |

```javascript
// Request a floor-relative space; fall back to local if unavailable
let referenceSpace;
try {
  referenceSpace = await session.requestReferenceSpace('local-floor');
} catch {
  referenceSpace = await session.requestReferenceSpace('local');
}
```

`XRReferenceSpace.getOffsetReferenceSpace(originOffset)` creates a child space with a fixed offset — useful for teleportation: moving the reference space origin to the target position rather than translating all scene objects.

---

## 5. Input Sources: Controllers and Hand Tracking

### XRInputSource

The `session.inputSources` array contains one `XRInputSource` per detected device (controller, hand, gaze). Each input source:
- `targetRayMode`: `"gaze"` (headset direction), `"tracked-pointer"` (controller ray), or `"screen"` (touch/mouse for inline)
- `targetRaySpace`: an `XRSpace` representing the pointer ray origin and direction
- `gripSpace`: optional space anchored to the physical grip of a held controller
- `gamepad`: a `Gamepad` object for button/axis state, using the standard Gamepad API

```javascript
session.addEventListener('inputsourceschange', (event) => {
  for (const source of event.added) {
    console.log(`Added: ${source.profiles[0]}, mode: ${source.targetRayMode}`);
    // source.profiles[0] is e.g. "valve-index" or "oculus-touch-v3"
    // Maps to OpenXR interaction profile paths via Chromium's GetOpenXrInputProfilesMap()
  }
});
```

In Chromium's OpenXR backend (`openxr_controller.cc`), each `XRInputSource` corresponds to an OpenXR interaction profile. The `GetOpenXrInputProfilesMap()` function (`openxr_interaction_profiles.cc`) maps WebXR profile strings to OpenXR interaction profile paths like `/interaction_profiles/valve/index_controller`. Button state is read via `xrGetActionStateBoolean` and `xrGetActionStateFloat`; grip and pointer poses via `xrLocateSpace`.

### XRHand

When `hand-tracking` is granted, `XRInputSource.hand` provides an `XRHand` object — an iterable of 25 named `XRJointSpace` objects matching the 26 joints of `XR_EXT_hand_tracking` (wrist + 5 × 5 finger joints):

```javascript
for (const inputSource of frame.session.inputSources) {
  if (!inputSource.hand) continue;

  // Get all joint poses in one call
  const jointPoses = frame.fillJointRadii(
    [...inputSource.hand.values()],  // XRJointSpace[]
    radiiFloat32Array                 // output radii, one per joint
  );
  frame.fillPoses(
    [...inputSource.hand.values()],
    referenceSpace,
    poseMatricesFloat32Array         // output: 16 floats per joint (4x4 matrix)
  );
}
```

`frame.fillJointRadii` and `frame.fillPoses` are batch APIs that retrieve all joint data in a single Mojo round-trip, minimising IPC overhead. In Chromium's OpenXR backend, these call `xrLocateHandJointsEXT` (from `XR_EXT_hand_tracking`) via `openxr_hand_tracker.cc`, which caches the `XrHandJointLocationsEXT` result per-frame and transcribes it into the `XRFrameData` Mojo struct.

---

## 6. Rendering: XRWebGLLayer and the WebXR Layers API

### XRWebGLLayer

`XRWebGLLayer` is the original rendering substrate: a UA-managed WebGL framebuffer that the runtime uses to carry rendered pixels to the OpenXR swapchain. The application does not create the `VkImage` or `EGLImage` behind the framebuffer — the browser allocates and manages the OpenXR swapchain internally.

```javascript
// Make the WebGL context XR-compatible before session creation
await gl.makeXRCompatible();

// Attach to session
session.updateRenderState({
  baseLayer: new XRWebGLLayer(session, gl, {
    framebufferScaleFactor: 1.0,  // 1.0 = native headset resolution
    antialias: true,
    depth: true,
    stencil: false,
    alpha: false,
  }),
});
```

`gl.makeXRCompatible()` is a critical synchronisation point: on Chrome, it ensures the WebGL context and the XR device use the same GPU and the same underlying EGL/Vulkan device. If the contexts live on different adapters (multi-GPU laptop), the call migrates the WebGL context. On Linux, this maps onto `xrGetVulkanGraphicsDeviceKHR` — the same constraint described in Chapter 27, Section 1.

Inside Chrome, `XRWebGLLayer` maps onto an ANGLE WebGL framebuffer whose backing texture is a `gpu::SharedImage` mailbox. On Windows, this `SharedImage` is a D3D11 texture shared with the OpenXR swapchain via `XR_KHR_D3D11_enable`. The absence of a Vulkan equivalent in the official codebase is the root cause of the Linux gap (Section 9).

### The WebXR Layers API (XRProjectionLayer)

The WebXR Layers API (`XRWebGLBinding`, `XRProjectionLayer`) is a more efficient rendering path available when the `layers` optional feature is granted. Unlike `XRWebGLLayer`, which renders into a single compositor-owned framebuffer and then copies to the swapchain, `XRProjectionLayer` gives the application direct access to the swapchain images, eliminating a copy:

```javascript
const session = await navigator.xr.requestSession('immersive-vr', {
  optionalFeatures: ['layers'],
});

const glBinding = new XRWebGLBinding(session, gl);
const projLayer = glBinding.createProjectionLayer({
  textureType: 'texture',   // or 'texture-array' for single-pass stereo
  colorFormat: gl.RGBA8,
  depthFormat: gl.DEPTH_COMPONENT24,
});

session.updateRenderState({ layers: [projLayer] });

function onXRFrame(time, frame) {
  session.requestAnimationFrame(onXRFrame);
  const pose = frame.getViewerPose(referenceSpace);
  if (!pose) return;

  for (const view of pose.views) {
    // getViewSubImage() returns the correct texture slice and viewport
    const subImage = glBinding.getViewSubImage(projLayer, view);
    gl.bindFramebuffer(gl.FRAMEBUFFER, subImage.framebuffer);
    const vp = subImage.viewport;
    gl.viewport(vp.x, vp.y, vp.width, vp.height);
    renderEye(view.projectionMatrix, view.transform.inverse.matrix);
  }
}
```

`getViewSubImage` returns a different `framebuffer` for each eye when `textureType: 'texture'`, or a different `imageIndex` into a texture array layer when `textureType: 'texture-array'` (single-pass stereo using `OVR_multiview2`). The `framebuffer` here directly wraps the acquired OpenXR swapchain image, so `xrReleaseSwapchainImage` is called automatically when the frame callback returns.

---

## 7. Chromium's WebXR Implementation: Process Model and Mojo

### Multi-Process Architecture

Chromium's WebXR implementation spans three processes:

```
Renderer process                 Browser process               VR Device process
─────────────────────            ────────────────────          ──────────────────────
Blink (JS WebXR API)             XRDeviceImpl                  isolated_xr_device
      │                                │                       (Windows only)
      │  VRService.RequestSession      │  XRRuntime             OpenXrDevice
      ├──────────────────────────────► │ ─────────────────────► OpenXrRenderLoop
      │                                │                        OpenXrApiWrapper
      │◄── XRSession (Mojo struct) ────┤◄── XRFrameData ────────│
      │                                │                        (XrSession loop)
      │  XRFrameDataProvider.GetFrameData()
      ├──────────────────────────────────────────────────────►  │
      │◄── XRFrameData ──────────────────────────────────────── │
```

On **Windows**, `isolated_xr_device` is a sandboxed utility process that holds the OpenXR session. The browser acts as a broker, passing Mojo pipe endpoints across the process boundary. The utility process isolation contains crashes in the XR runtime (SteamVR, WMR, OpenXR loader) from the browser.

On **Android**, the VR device code runs directly in the browser process to minimise startup latency and process overhead.

On **Linux**, the architecture is the same as Windows, but the `isolated_xr_device` process is not built or launched because the D3D11 graphics binding — the only `OpenXrGraphicsBinding` implementation in the official codebase — is Windows-specific.

### Key Mojo Interfaces

All WebXR cross-process communication uses Mojo interfaces defined in `//device/vr/public/mojom/vr_service.mojom`:

**`VRService`** (browser process, consumed by renderer): single interface for session initiation and runtime availability queries.
```
interface VRService {
  RequestSession(XRSessionOptions options, XRSessionRequestTokenPtr token)
      => (XRSession? session, string? error_message);
  SupportsSession(XRSessionMode mode) => (bool supports);
  ExitPresent() => ();
};
```

**`XRFrameDataProvider`** (VR device process, consumed by renderer directly on the XR thread): delivers per-frame tracking data at native refresh rate.
```
interface XRFrameDataProvider {
  GetFrameData(XRFrameDataRequestOptions options)
      => (XRFrameData? frame_data);
};
```

The **`XRFrameData`** struct bundles everything the renderer needs for one frame:
- `XRRenderInfo render_info` — viewer pose + per-eye `XRView` (position, orientation, projectionMatrix, viewport)
- `mojo_base.mojom.TimeDelta time_delta` — predicted display time
- `XRInputSourceStatePtr[]` — all input sources with poses and button states
- `XRHandTrackingDataPtr? hand_tracking_data` — 26-joint arrays per hand (when `hand-tracking` granted)
- `XRPlaneDetectionDataPtr? detected_planes` — plane polygon arrays (when `plane-detection` granted)
- `XRAnchorDataPtr[] updated_anchors` — anchor pose updates (when `anchors` granted)
- `XRDepthDataPtr? depth_data` — depth image reference (when `depth-sensing` granted)

**`XRPresentationProvider`** (VR device process): receives the rendered framebuffer for presentation to the OpenXR swapchain.
```
interface XRPresentationProvider {
  SubmitFrameMissing(int16 frame_index, gpu.mojom.SyncToken sync_token);
  SubmitFrame(int16 frame_index, gpu.mojom.MailboxHolder mailbox,
              mojo_base.mojom.TimeDelta time_waited);
};
```

`SubmitFrame` passes a `gpu::MailboxHolder` referencing the `SharedImage` that contains the rendered eye texture. The VR device process resolves this mailbox, synchronises via the `SyncToken` (a GPU fence), and copies/imports the texture into the OpenXR swapchain image before calling `xrReleaseSwapchainImage` and `xrEndFrame`.

---

## 8. The OpenXR Backend: OpenXrDevice to XrSession

### Class Hierarchy

```
OpenXrDevice          implements XRRuntime
  └── OpenXrRenderLoop    drives the XR thread + Mojo interfaces
        └── OpenXrApiWrapper    owns XrSession and XrSwapchain
              ├── OpenXrExtensionHelper   extension enumeration and function pointers
              ├── OpenXrInputHelper       action sets, controller poses
              │     └── OpenXrHandTracker  XR_EXT_hand_tracking per hand
              ├── OpenXrHitTestManager    XR_MSFT_raycast or XR_EXT_plane_detection
              ├── OpenXrPlaneManager      XR_EXT_plane_detection
              ├── OpenXrAnchorManager     XR_EXT_spatial_anchor
              └── OpenXrLightEstimator    XR_EXT_light_estimation
```

Source: `//device/vr/openxr/` ([chromium.googlesource.com](https://chromium.googlesource.com/chromium/src/+/HEAD/device/vr/openxr/))

### Session Creation and the Render Loop

`OpenXrRenderLoop` inherits from `XRThread` (a `base::Thread` wrapper) and owns the Mojo `XRFrameDataProvider` and `XRPresentationProvider` receiver endpoints. All frame-level operations run on this dedicated thread to achieve low-latency data delivery without blocking the browser's main thread.

The session creation sequence mirrors native OpenXR (Chapter 27, Section 1), but with the constraint that the graphics binding must match what the OpenXR runtime expects:

1. `xrCreateInstance` with the required extensions enumerated by `OpenXrExtensionHelper`
2. `xrGetSystem` for `XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY`
3. `OpenXrPlatformHelper::GetRequiredGraphicsApiExts()` — returns `["XR_KHR_D3D11_enable"]` on Windows, `["XR_KHR_opengl_es_enable"]` on Android
4. `xrCreateSession` with the platform's `XrGraphicsBinding*` in the `next` chain
5. `xrCreateSwapchain` with `XR_USAGE_COLOR_ATTACHMENT` — the image array is shared with the `gpu::SharedImage` system via the platform graphics binding

### Input Action Sets

`OpenXrInputHelper` creates a single `XrActionSet` at session start. For each supported interaction profile (Valve Index, Oculus Touch, WMR, etc.), it creates `XrAction` entries for:
- Grip pose (`XR_ACTION_TYPE_POSE_INPUT`)
- Aim/pointer pose (`XR_ACTION_TYPE_POSE_INPUT`)
- Primary trigger (`XR_ACTION_TYPE_FLOAT_INPUT`)
- Squeeze / grip button (`XR_ACTION_TYPE_FLOAT_INPUT`)
- Thumbstick X/Y (`XR_ACTION_TYPE_FLOAT_INPUT`)

Each frame, `xrSyncActions` processes all bound actions, and `xrGetActionStatePose` + `xrLocateSpace` yields the grip and aim poses that populate `XRInputSourceState` in the `XRFrameData` Mojo response.

---

## 9. High-Level WebXR Frameworks: A-Frame and Babylon.js

The raw WebXR Device API provides spatial poses, projection matrices, and a framebuffer — it intentionally says nothing about scene graphs, asset loading, physics, or animation. In practice, the vast majority of WebXR content is built on one of three JavaScript frameworks that absorb this complexity: **A-Frame** (declarative, entity-component, Mozilla-originated), **Babylon.js** (imperative, full engine, Microsoft-backed), and **Three.js** (minimal scene graph, the substrate both A-Frame and many Babylon.js projects also rely on). Understanding how these frameworks sit on the WebXR Device API — and how that chain maps onto the Linux graphics stack — is essential context for diagnosing performance and compatibility issues on Linux.

### A-Frame: Declarative WebXR with an Entity-Component System

**A-Frame** ([https://aframe.io](https://aframe.io), source at [https://github.com/aframevr/aframe](https://github.com/aframevr/aframe)) is a web framework for building VR and AR experiences using HTML-like custom elements. An entire VR scene can be authored as a single HTML file:

```html
<!DOCTYPE html>
<html>
  <head><script src="https://aframe.io/releases/1.6.0/aframe.min.js"></script></head>
  <body>
    <a-scene>
      <a-box position="-1 0.5 -3" rotation="0 45 0" color="#4CC3D9" shadow></a-box>
      <a-sphere position="0 1.25 -5" radius="1.25" color="#EF2D5E" shadow></a-sphere>
      <a-plane position="0 0 -4" rotation="-90 0 0" width="4" height="4" color="#7BC8A4" shadow></a-plane>
      <a-sky color="#ECECEC"></a-sky>
    </a-scene>
  </body>
</html>
```

Clicking the VR headset icon in the A-Frame UI calls `navigator.xr.requestSession('immersive-vr')` and transitions the scene into an `XRSession`. A-Frame handles all WebXR session management, reference space acquisition, and per-frame pose retrieval internally.

**Entity-Component System (ECS).** A-Frame's architecture is a browser-native ECS:

- **Entity** (`<a-entity>`) — a DOM element with no inherent behaviour; the unit of composition
- **Component** — a reusable JavaScript object attached via HTML attribute (e.g. `position`, `rotation`, `material`, `geometry`, `shadow`, `laser-controls`); holds data and per-frame `tick()` logic
- **System** — a scene-level singleton that manages shared state across all instances of a component type

```javascript
// Registering a custom A-Frame component
AFRAME.registerComponent('spin', {
  schema: { speed: { type: 'number', default: 1 } },
  tick(time, delta) {
    this.el.object3D.rotation.y += this.data.speed * delta * 0.001;
  }
});
```

This ECS maps cleanly onto Three.js's `Object3D` hierarchy: each `<a-entity>`'s underlying `object3D` is a `THREE.Object3D` node in the scene graph, and A-Frame delegates all rendering to Three.js's `WebGLRenderer`. [Source: A-Frame architecture docs](https://aframe.io/docs/1.6.0/introduction/entity-component-system.html)

**Input handling.** A-Frame wraps WebXR input sources via the `tracked-controls`, `laser-controls`, and `hand-controls` components. These poll `XRFrame.getInputSourceArray()` each tick and synthesise A-Frame events (`triggerdown`, `thumbstickmoved`, `pinchstarted`) from `XRInputSource.gamepad` state. On Linux with Monado, this chain works identically to Windows provided the OpenXR input action bindings are present.

### A-Frame's Rendering Pipeline and the Linux Stack

The full rendering call chain from A-Frame JavaScript down to the Linux kernel is:

```
A-Frame scene.tick()
  └─ THREE.WebGLRenderer.render(scene, camera)          // Three.js r168+
       └─ WebGL2RenderingContext.drawElements()          // Browser WebGL2 API
            └─ ANGLE (Vulkan backend, libEGL/libGLESv2)  // Ch34: OpenGL ES → VkCmdDraw
                 └─ Mesa Vulkan driver (RADV / ANV / NVK) // Ch18: vkQueueSubmit
                      └─ amdgpu / i915 / nouveau DRM KMS  // Ch2: mode-set, scanout
```

When in an immersive WebXR session, Three.js's `WebGLRenderer` uses its `WebXRManager` to:
1. Call `session.requestAnimationFrame()` instead of `window.requestAnimationFrame()`
2. Retrieve `XRFrame.getViewerPose(referenceSpace)` to get per-eye `XRView` objects
3. Set `gl.bindFramebuffer(gl.FRAMEBUFFER, session.renderState.baseLayer.framebuffer)` to render into the XR compositor's framebuffer
4. Issue two viewport-separated draw calls (one per eye) using `XRView.projectionMatrix` and `XRView.transform.inverse.matrix`

The framebuffer at step 3 is backed by an ANGLE `FramebufferVk` whose `VkImage` is shared with the OpenXR swapchain (on the community-patched Chromium path) or with Firefox's `wgpu` device (on the Firefox path). The pixel data flows from Mesa's Vulkan driver into the OpenXR runtime's compositor without a CPU copy when `XR_KHR_vulkan_enable2` is used. [Source: Three.js WebXRManager](https://github.com/mrdoob/three.js/blob/dev/src/renderers/webxr/WebXRManager.js)

**A-Frame version compatibility.** A-Frame 1.5+ requires Three.js r155+, which in turn requires `WebGL2RenderingContext` (no WebGL 1 fallback). On Linux, ANGLE's Vulkan backend provides WebGL2; the path through ANGLE → Mesa Vulkan → RADV/ANV is the same as described in Chapter 34. A-Frame's `renderer` component allows setting `antialias`, `colorManagement`, and `logarithmicDepthBuffer` properties that map to corresponding Three.js `WebGLRenderer` options and ultimately to MSAA sample counts and depth format selection in ANGLE's `FramebufferVk`.

### Babylon.js XR: Imperative High-Performance WebXR

**Babylon.js** ([https://www.babylonjs.com](https://www.babylonjs.com), source at [https://github.com/BabylonJS/Babylon.js](https://github.com/BabylonJS/Babylon.js)) is a full 3D engine with a more imperative API than A-Frame. Its WebXR support is encapsulated in `WebXRExperienceHelper`, which manages session lifecycle, and `WebXRDefaultExperience`, a higher-level helper that wires up teleportation locomotion, controller models, and UI:

```typescript
// Babylon.js WebXR session init
const xr = await scene.createDefaultXRExperienceAsync({
  floorMeshes: [ground],
  optionalFeatures: ['hand-tracking', 'layers'],
});
// xr.baseExperience.sessionManager holds the XRSession
// xr.baseExperience.camera is a WebXRCamera (stereo)
```

Babylon.js renders via its own abstraction over WebGL2 (`ThinEngine` → `WebGLHardwareTexture`, `WebGLRenderTargetWrapper`), not Three.js. The path through ANGLE to Mesa Vulkan is identical — Babylon.js calls the same `WebGL2RenderingContext` methods — but Babylon.js manages its own material system, PBR shader generation, and occlusion culling independently of Three.js. For WebXR the key class is `WebXRCamera`, which overrides `getTransformationMatrix()` with per-eye `XRView.projectionMatrix` values each frame. [Source: Babylon.js WebXR docs](https://doc.babylonjs.com/features/featuresDeepDive/webXR/introToWebXR)

Babylon.js also supports WebGPU via `WebGPUEngine` (backed by Dawn on Chrome), which replaces the WebGL2 backend while keeping the same scene graph API. On Linux, `WebGPUEngine` follows the Dawn → Mesa Vulkan path described in Chapter 35 rather than the ANGLE path. The dual-backend design (`WebGLEngine` / `WebGPUEngine`) mirrors `wgpu`'s multi-backend model (Chapter 40) but at the framework level rather than the driver abstraction level.

### Three.js VRButton / ARButton

**Three.js** itself provides minimal WebXR scaffolding in its `examples/jsm/webxr/` directory — `VRButton.js` and `ARButton.js` — which are thin wrappers that call `navigator.xr.requestSession()` and wire the session into a `THREE.WebGLRenderer`:

```javascript
import { VRButton } from 'three/addons/webxr/VRButton.js';

renderer.xr.enabled = true;
document.body.appendChild(VRButton.createButton(renderer));
// VRButton calls renderer.xr.setSession(session) on user gesture
```

`renderer.xr` is a `WebXRManager` instance that handles `requestAnimationFrame` delegation to `XRSession`, viewer pose retrieval, and per-eye viewport setup. A-Frame's internal rendering path is a superset of this — A-Frame creates a `THREE.WebGLRenderer` with `xr.enabled = true` and drives it through its ECS tick loop rather than a hand-written animation loop.

### Framework Comparison and Linux Considerations

| Dimension | A-Frame 1.6 | Babylon.js 7.x | Three.js r168 (raw) |
|---|---|---|---|
| **API style** | Declarative HTML / ECS | Imperative TypeScript engine | Minimal scene graph; user drives loop |
| **WebXR integration** | `XRSession` managed internally; `laser-controls`, `hand-controls` components | `WebXRExperienceHelper`; feature modules (locomotion, UI, hand-tracking) | `WebXRManager`; user wires session into renderer |
| **Renderer** | Three.js `WebGLRenderer` | Babylon.js `ThinEngine` (WebGL2 or WebGPU) | `WebGLRenderer` (WebGL2) or `WebGPURenderer` (experimental) |
| **WebGPU support** | Experimental (Three.js `WebGPURenderer` opt-in) | Stable `WebGPUEngine` | Experimental `WebGPURenderer` |
| **Linux path (WebGL)** | A-Frame → Three.js → ANGLE Vulkan → Mesa → DRM | Babylon.js → ANGLE Vulkan → Mesa → DRM | Three.js → ANGLE Vulkan → Mesa → DRM |
| **Linux path (WebGPU)** | Three.js `WebGPURenderer` → Dawn Vulkan → Mesa → DRM | `WebGPUEngine` → Dawn Vulkan → Mesa → DRM | `WebGPURenderer` → Dawn Vulkan → Mesa → DRM |
| **Hand tracking** | `hand-controls` + `XR_EXT_hand_tracking` | `WebXRHandTracking` feature module | Manual; `XRHand` polling |
| **Asset loading** | `<a-asset-item>`, glTF via Three.js `GLTFLoader` | Babylon.js `SceneLoader` (glTF, OBJ, etc.) | User-managed loaders |
| **Physics** | `aframe-physics-system` (Cannon.js / Rapier) | Havok Physics (WASM), Cannon.js, Ammo.js | User-supplied |
| **Typical bundle size** | ~1 MB minified (A-Frame includes Three.js) | ~2–4 MB (full engine) | ~600 KB (core only) |

**Linux-specific considerations.** All three frameworks share the same WebXR session availability constraint: on stock Chromium, `isSessionSupported('immersive-vr')` returns `false` on Linux (§10). A-Frame's `<a-scene>` will gracefully degrade to flat rendering in this case. Framework-level debugging on Linux is most productive via `chrome://webxr-internals` (Chromium) or `about:debugging` → WebXR (Firefox), which show whether `XrSession` creation succeeded and which OpenXR features were negotiated.

For performance profiling of A-Frame or Babylon.js scenes on Linux, the relevant measurement points are:
- **JavaScript frame budget**: Chrome DevTools Performance panel; look for `tick()` / `render()` cost
- **ANGLE draw call overhead**: `chrome://tracing` with `gpu` category; look for `ANGLE::drawElements` and `CommandBufferProxy::Flush`
- **Mesa Vulkan submission**: `VK_INSTANCE_LAYERS=VK_LAYER_MESA_overlay` renders a GPU utilisation HUD
- **OpenXR runtime timing**: Monado's `XRT_PRINT_PERFORMANCE=1` env var prints per-frame compositor latency

---

## 10. WebXR on Linux: Status, Gaps, and the Community Path

### Official Status

As of mid-2026, **Chromium does not officially support `immersive-vr` or `immersive-ar` WebXR sessions on Linux**. Inline sessions (`session.requestAnimationFrame` in a canvas) work via the DeviceOrientation API. The root cause is the graphics binding:

The only `OpenXrGraphicsBinding` implementations in the official Chromium codebase are:
- `OpenXrGraphicsBindingD3D11` (Windows) — `XR_KHR_D3D11_enable`
- `OpenXrGraphicsBindingEGLES` (Android) — `XR_KHR_opengl_es_enable`

Linux needs `XR_KHR_vulkan_enable` (or `XR_KHR_vulkan_enable2`), which would bind Chromium's Vulkan device to the OpenXR swapchain — exactly the `XrGraphicsBindingVulkanKHR` that native OpenXR applications on Linux use (Chapter 27, Section 1). This binding has not been added to the official codebase. [Source: Chromium device/vr README](https://chromium.googlesource.com/chromium/src/+/HEAD/device/vr/README.md)

### The Community Patch (webxr-linux)

The `mrxz/webxr-linux` project provides a Chromium patch that adds a Vulkan graphics binding for Linux:

- Adds `OpenXrGraphicsBindingVulkan` implementing `OpenXrGraphicsBinding` using `XR_KHR_vulkan_enable2`
- Calls `xrGetVulkanGraphicsDevice2KHR` and `xrGetVulkanGraphicsRequirements2KHR` to select the correct `VkPhysicalDevice` — the same device the OpenXR runtime (Monado or SteamVR) expects
- Shares swapchain images between the OpenXR `XrSwapchain` and Chromium's `gpu::SharedImage` via `VkExternalMemoryImageCreateInfo` and `VkImportMemoryFdInfoKHR` (the DMA-BUF path, identical to what Chapter 25 describes for CUDA-Vulkan interop)
- Builds with `ENABLE_VR=true` and `enable_openxr=true` GN args

[Source: webxr-linux on GitHub](https://github.com/mrxz/webxr-linux)

### Enabling WebXR on a Patched Build

With the community patch applied and a Monado or SteamVR runtime present:

```bash
# Point Chrome at the Monado OpenXR runtime
export XR_RUNTIME_JSON=~/.config/openxr/1/active_runtime.json
# Or for SteamVR:
export XR_RUNTIME_JSON=/usr/share/steam/steamvr/steamvr.json

# Launch with VR flags
chromium --enable-features=WebXR,OpenXR \
         --use-vulkan \
         --enable-unsafe-webgpu
```

The `chrome://flags/#webxr-runtime` flag (available on Windows builds) selects between OpenXR and other runtimes; on Linux it must be driven by GN build configuration. `chrome://webxr-internals` shows session state, tracking status, and the active OpenXR runtime version — useful for diagnosing whether the runtime JSON was found and whether `XrInstance` creation succeeded.

### What Works and What Does Not (mid-2026)

| Feature | Status on Linux |
|---|---|
| `inline` session | Works in stock Chrome |
| `isSessionSupported('immersive-vr')` | Returns `false` in stock Chrome |
| `immersive-vr` with community patch + Monado | Works (RADV/ANV Vulkan) |
| `immersive-vr` with community patch + SteamVR | Works on supported headsets |
| `hand-tracking` via `XR_EXT_hand_tracking` | Works when Monado driver implements it |
| `plane-detection`, `anchors` | Depends on runtime; Monado in progress |
| `depth-sensing` | Monado roadmap; not stable |
| WebXR Layers API (`XRProjectionLayer`) | Not tested on Linux path |

---

## 11. Firefox WebXR via wgpu

Firefox's WebXR implementation uses a different architecture. Rather than an isolated XR device process, Firefox runs the OpenXR session within the compositor process (`RDD` — Remote Data Decoder, or the GPU process) using the `wgpu` Rust library as the GPU abstraction.

### wgpu on Linux

`wgpu`'s Vulkan backend (`wgpu-hal/src/vulkan/`) directly creates `VkInstance`, `VkDevice`, and `VkSwapchain` on Linux without a D3D11 or EGL intermediate. For WebXR, Firefox creates an `XrGraphicsBindingVulkanKHR` from `wgpu`'s `VkDevice`, making the Linux graphics binding a natural consequence of `wgpu`'s cross-platform Vulkan support.

As of Firefox 126 (mid-2026):
- `immersive-vr` is supported on Linux with SteamVR (via the system OpenXR loader)
- Monado support is functional but less tested
- `navigator.xr.isSessionSupported('immersive-vr')` returns `true` when an OpenXR runtime is present
- Hand tracking is behind `dom.webxr.hands.enabled` in `about:config`

Firefox discovers the OpenXR runtime via the standard `XR_RUNTIME_JSON` environment variable or the XDG path `$XDG_CONFIG_HOME/openxr/1/active_runtime.json`, identical to native applications.

### Firefox WebXR Frame Path

```
Gecko JS engine (SpiderMonkey)
  → WebXR IDL bindings (dom/xr/)
  → XRSession::RequestAnimationFrame()
  → wgpu WebXR integration (gfx/webrender_bindings/ + gfx/vr/)
  → OpenXR runtime (SteamVR / Monado)
    xrWaitFrame / xrBeginFrame / xrEndFrame
  → wgpu Vulkan swapchain image
  → xrReleaseSwapchainImage
```

WebRender's GPU pipeline (Chapter 52) and the WebXR frame loop share the same `wgpu` device; the `WebRenderBridgeParent` and the XR compositor thread coordinate via `wgpu`'s `Queue::submit()` fences. [Source: Firefox wgpu WebXR](https://firefox-source-docs.mozilla.org/gfx/webrender.html)

---

## 12. WebXR vs. Native OpenXR: Tradeoffs

| Dimension | WebXR | Native OpenXR (Monado) |
|---|---|---|
| **Distribution** | URL — zero install | Native binary or package |
| **Runtime access** | JS only; browser mediates all XR calls | Full `XrInstance` API, all extensions |
| **Extension access** | Runtime decides which features to expose | Any `XrInstance` extension available |
| **Latency** | +0.5–2 ms Mojo IPC overhead per frame | Direct in-process OpenXR calls |
| **Raw camera** | Not available (privacy model) | Available via `XR_EXT_camera_access` |
| **Passthrough** | `immersive-ar` + `XRCompositionLayerPassthrough` (limited) | Full `XrCompositionLayerPassthroughFB` |
| **GPU API** | WebGL (ANGLE) or WebGPU (Dawn) | Vulkan, OpenGL — any Mesa backend |
| **Performance ceiling** | Lower: ANGLE/WebGL overhead, Mojo copy | Higher: direct Mesa Vulkan, no copy |
| **Platform** | Chrome (Win/Android officially, Linux via patch); Firefox (Win/Linux/Android) | Linux, Windows, Android — same binary |
| **Scene understanding** | `plane-detection`, `anchors` (runtime-dependent) | Full `XR_EXT_plane_detection`, `XR_EXT_spatial_anchor`, mesh reconstruction |

For end users and web developers, WebXR offers zero-install portability and sandbox security at the cost of ~1–2 ms additional latency and a constrained API surface. For VR application developers targeting Linux who need full Monado feature access — custom hand-tracking pipelines, direct scene mesh, or experimental SLAM extensions — native OpenXR remains the correct path.

The key architectural difference is the copy at frame submission: native OpenXR applications render directly into an `XrSwapchain` image (a `VkImage` bound to the runtime's compositor). WebXR applications render into an ANGLE framebuffer or `XRProjectionLayer` texture that the browser must copy or import into the OpenXR swapchain image through the `gpu::SharedImage` / Mojo mailbox system. The WebXR Layers API (`XRProjectionLayer` with direct swapchain image access) reduces but does not eliminate this overhead.

---

## 13. Roadmap: WebXR Specification and Implementation Trajectory

### Near-Term (2026–2027): Spec Stabilisation and the WebGPU Rendering Path

**XRWebGPUBinding (WebXR for WebGPU)** is the single most consequential in-progress item. The `XRWebGPUBinding` interface will allow projection layers backed directly by `GPUTexture` objects wrapping the OpenXR swapchain's `VkImage`, eliminating the ANGLE intermediate copy that the `XRWebGLLayer` path requires today. The Immersive Web Working Group [tracks this at GitHub](https://github.com/immersive-web/WebXR-WebGPU-Binding). Dawn's `WGPUTexture` external memory path (`WGPUSharedTextureMemoryVkImageDescriptor`) already provides the mechanism; the remaining work is the JS binding and the Chromium `gpu::SharedImage` integration. Expect an experimental flag in Chromium and Firefox within 2026.

**WebXR Depth Sensing Module** is at Working Draft stage. The API exposes per-frame depth buffers (CPU or GPU access) from devices that expose `XR_EXT_hand_tracking` depth or from platform depth APIs (ARCore, LiDAR on Apple hardware). On Linux / Monado the depth output from the tracking camera is not yet surfaced to the WebXR layer. Monado's roadmap includes `XR_EXT_hand_tracking_data_source` and depth support; when that lands the Chromium/Firefox OpenXR backend can forward it.

**WebXR Hand Input Level 1** (hand-tracking module) is advancing toward Candidate Recommendation. The `XRHand` interface and `fillJointRadii`/`fillPoses` bulk-fill methods are stable in both Chrome (behind `webxr-hand-input` flag) and Firefox 126. The Level 2 draft adds gesture recognition events and joint velocity data.

**WebXR Hit Test Module Level 1** reached Candidate Recommendation in 2025. It allows a web application to cast a ray from a controller or screen position into the real-world plane geometry and receive intersection poses — the building block for AR object placement. On Linux, hit-test requires the OpenXR runtime to expose `XR_EXT_plane_detection`; Monado's Hydra scene understanding plugin is the path toward this.

**WebXR Anchors Module** stabilises the `XRAnchor` interface for persisting AR reference points across sessions. At Working Draft stage as of mid-2026; Monado's `XR_MSFT_spatial_anchor` implementation is the Linux-side dependency.

**Chromium upstream Vulkan binding for Linux**: The `webxr-linux` community patch adds `OpenXrGraphicsBindingVulkan` using `XR_KHR_vulkan_enable2`. Upstreaming this patch into the Chromium codebase has been discussed in the `crbug.com` tracker. The primary blocker is test coverage (Chromium CI has no Linux XR hardware) and the lack of an official Linux SteamVR/Monado testing environment in the project's infra. The patch is a small and well-scoped addition; upstreaming is realistic within this timeframe.

### Medium-Term (2027–2028): XRWebGPUBinding as Primary Path, Scene Understanding, Eye Tracking

**XRWebGPUBinding ship**: Once the experimental flag stabilises and interop tests pass, the expectation is that `XRWebGPUBinding` becomes the primary rendering path for new WebXR applications. `XRWebGLLayer` will remain for compatibility. The performance improvement on Linux is significant: the current community-patch path copies or imports swapchain images through ANGLE/Vulkan external memory; a native `GPUTexture` path eliminates that indirection.

**WebXR Lighting Estimation** exposes the real-world illumination estimate (spherical harmonics + HDR image) from AR runtimes (ARCore, ARKit, Monado's future lighting extension) to JS as an `XRLightEstimate`. The API is in the Immersive Web Community Group spec; Chromium's ARCore backend already exposes a prototype. On Linux the dependency is an OpenXR lighting extension in Monado, which has no shipping implementation as of mid-2026.

**WebXR Eye Tracking Module** exposes per-eye gaze direction from `XR_EXT_eye_gaze_interaction`. The spec is at Community Group stage; privacy constraints (gaze is considered sensitive biometric data) require a dedicated permission prompt distinct from the base `immersive-vr` permission. Valve Index (SteamVR) exposes `XR_EXT_eye_gaze_interaction`; Monado has a driver stub. The open question is W3C Privacy Interest Group sign-off on the permission model.

**WebXR Mesh Detection Module** will expose the reconstructed surface mesh from devices with spatial mapping (Quest Pro, Apple Vision Pro, and Monado-backed SLAM runtimes). Monado's integration with Kimera-VIO and ILLIXR scene graph targets this capability. The `XRMesh` interface design is under active discussion in the Immersive Web WG.

**WebXR + WebNN integration**: Hand tracking, body pose estimation, and scene understanding in AR can be augmented or fully replaced by in-browser ML inference via WebNN (Chapter 168). The pattern: `XRFrame.getJointPose()` provides hardware tracking; a WebNN model refines or corrects it with additional camera data (once raw camera access is available). The `WebXR Camera Access Module` (a separate Community Group proposal) would expose raw luminance frames for this purpose — subject to strong privacy mitigations.

### Long-Term (2028+): Platform Integration, Passthrough Standardisation, Neural Rendering

**Passthrough standardisation across vendors**: The `XRCompositionLayerPassthrough` API is currently runtime-dependent, with behaviour varying across Quest, SteamVR, Monado, and ARCore backends. A standardised passthrough layer format (blend mode, edge rendering, occlusion depth) tied to the WebXR Layers API Level 2 spec is the goal. This requires alignment across OpenXR extension promotions (`XR_FB_passthrough` → `XR_META_passthrough` → proposed core promotion).

**Native platform WebXR runtimes**: On macOS/visionOS (Apple Vision Pro) and Windows (Mixed Reality, upcoming), platform-native WebXR runtimes would bypass the need for OpenXR. Safari's WebXR implementation targets visionOS; Chrome on Windows already integrates directly. The long-term vision is that every major platform has a first-party WebXR runtime, reducing the OpenXR + community-patch chain that Linux currently requires.

**WebXR for streaming / cloud XR**: Rendering on a server and streaming frames to a thin client (as in NVIDIA CloudXR or PlayStation Remote Play) requires the WebXR API to express video-plane composition layers. `XRMediaBinding` (Layers API extension) provides the hook; coupling it with `WebCodecs` for low-latency decode is the path. This is particularly relevant for Linux XR clients where the local GPU may not support the full application workload.

**Neural rendering integration**: Compressed neural radiance field (NeRF / 3DGS) representations of real scenes could be streamed and rendered via WebGPU compute shaders within a WebXR session, providing photorealistic passthrough without requiring a depth camera. The `WebNN` inference path would run the decoding neural network; `XRWebGPUBinding` would composite the output into the XR frame. This is a research-to-production trajectory rather than a standards deliverable; implementations are expected in research previews by 2028.

### Specification Milestone Summary

| Module | Status (mid-2026) | Expected Milestone |
|---|---|---|
| WebXR Device API | Living Standard (stable) | Continues as living standard |
| WebXR Layers API | CR (Candidate Recommendation) | W3C Recommendation 2026 |
| WebXR Hit Test | CR | W3C Recommendation 2026 |
| WebXR Hand Input Level 1 | Working Draft → CR | CR 2026/2027 |
| WebXR Depth Sensing | Working Draft | CR 2027 |
| WebXR Anchors | Working Draft | CR 2027 |
| WebXR Lighting Estimation | CG Note | WG adoption 2027 |
| XRWebGPUBinding | CG Proposal / experimental | WG adoption 2027 |
| WebXR Eye Tracking | CG Proposal | WG adoption 2028 |
| WebXR Mesh Detection | Editor's Draft | CR 2028 |
| WebXR Camera Access | CG Proposal | Pending privacy review |

---

## 14. Integrations

**Chapter 27 (VR, AR, and OpenXR)**: WebXR is the browser-facing surface above the same Monado/SteamVR OpenXR runtimes described there. `XR_RUNTIME_JSON` discovery, DRM leasing for direct-mode output, and the Basalt VIO tracking pipeline all sit below both the native OpenXR and WebXR paths. The OpenXR session created by Chromium's `OpenXrApiWrapper` is an `XrSession` in the same Monado `monado-service` daemon that native apps use.

**Chapter 33 (Chromium Multi-Process GPU Architecture)**: The `isolated_xr_device` service (on Windows) is a sibling of the GPU process, using the same Mojo IPC and sandbox infrastructure. The VR device service's `gpu::SharedImage` mailboxes share the same `GpuChannelManager` backing that the GPU process uses for WebGL and WebGPU textures.

**Chapter 34 (ANGLE — WebGL on Linux)**: `XRWebGLLayer`'s framebuffer is an ANGLE `FramebufferVk` (Vulkan backend) or `FramebufferGL` (GL backend). On the Linux community patch, the swapchain image import uses ANGLE's `EGL_ANDROID_get_native_client_buffer` or Vulkan external memory to share the OpenXR swapchain `VkImage` with ANGLE's context.

**Chapter 35 (Dawn and WebGPU)**: The forthcoming WebXR `GPULayer` (`XRWebGPUBinding`) will create projection layers backed by Dawn `WGPUTexture` objects wrapping the OpenXR swapchain images directly, eliminating the ANGLE intermediate. Dawn's Vulkan backend already supports `VK_KHR_external_memory` for texture sharing; the WebXR integration is in active specification at the Immersive Web Working Group.

**Chapter 52 (Firefox and WebRender)**: Firefox's WebXR frame loop shares the `wgpu` Vulkan device with WebRender's render backend. Timeline semaphores (`wgpu`'s `Queue::submit()` signalling) coordinate between the WebRender frame and the XR frame without CPU stalls.

**Chapter 87 (Android AR/ARCore)**: WebXR `immersive-ar` on Android is implemented via Chrome's ARCore backend (`//chrome/browser/vr/`), not the OpenXR backend. The JS API surface is identical; only the underlying runtime differs. See Chapter 87, Section 9a for the full ARCore-backed WebXR path.

**Chapter 168 (WebNN)**: For WebXR hand tracking and scene understanding, `WebNN` is the in-browser path for running the same ML models (MediaPipe Hands, depth networks) that Monado runs natively in Section 6b of Chapter 27. The WebNN inference runs in the renderer process, consuming `XRFrame` joint pose estimates as training signals or augmenting them with body-pose network output.

---

## 15. References

1. **W3C WebXR Device API** — Living Standard, June 2026. [https://www.w3.org/TR/webxr/](https://www.w3.org/TR/webxr/)
2. **WebXR Explainer** — Immersive Web Working Group. [https://immersive-web.github.io/webxr/explainer.html](https://immersive-web.github.io/webxr/explainer.html)
3. **Chromium device/vr README** — Architecture overview of the VR device service, process model, and Mojo interfaces. [https://chromium.googlesource.com/chromium/src/+/HEAD/device/vr/README.md](https://chromium.googlesource.com/chromium/src/+/HEAD/device/vr/README.md)
4. **Chromium device/vr/openxr README** — OpenXR backend class hierarchy, extension handling, input profiles. [https://chromium.googlesource.com/chromium/src/+/HEAD/device/vr/openxr/README.md](https://chromium.googlesource.com/chromium/src/+/HEAD/device/vr/openxr/README.md)
5. **webxr-linux** — Community project adding Vulkan graphics binding for Linux. [https://github.com/mrxz/webxr-linux](https://github.com/mrxz/webxr-linux)
6. **WebXR Layers API specification** — `XRProjectionLayer`, `XRWebGLBinding`, `getViewSubImage`. [https://immersive-web.github.io/layers/](https://immersive-web.github.io/layers/)
7. **WebXR Hand Input API** — `XRHand`, `XRJointSpace`, `fillJointRadii`, `fillPoses`. [https://www.w3.org/TR/webxr-hand-input-1/](https://www.w3.org/TR/webxr-hand-input-1/)
8. **Firefox WebXR documentation** — wgpu-backed implementation, `about:config` flags, SteamVR/Monado integration. [https://firefox-source-docs.mozilla.org/gfx/webrender.html](https://firefox-source-docs.mozilla.org/gfx/webrender.html)
9. **vr_service.mojom** — Mojo interface definitions for `VRService`, `XRSession`, `XRFrameDataProvider`, `XRFrameData`. [https://github.com/chromium/chromium/blob/master/device/vr/public/mojom/vr_service.mojom](https://github.com/chromium/chromium/blob/master/device/vr/public/mojom/vr_service.mojom)
10. **MDN WebXR Device API** — Developer reference for `XRSession`, `XRFrame`, `XRView`, `XRInputSource`, `XRHand`. [https://developer.mozilla.org/en-US/docs/Web/API/WebXR_Device_API](https://developer.mozilla.org/en-US/docs/Web/API/WebXR_Device_API)
