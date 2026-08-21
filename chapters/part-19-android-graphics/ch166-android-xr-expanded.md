# Chapter 166: Android XR: Jetpack XR SDK, OpenXR Platform Contracts, and Spatial Extensions

> **Part**: Part XIX — Android Graphics
> **Audience**: Graphics application developers and systems/driver developers building Android AR/XR applications and runtimes. Readers focused on desktop/embedded Linux OpenXR and Monado should treat this chapter as the Android-specific counterpart to Ch27's cross-platform OpenXR session model.
> **Status**: First draft — 2026-08-21

This chapter is an expanded companion to [Chapter 87](ch87-android-ar-arcore.md), which covers ARCore's SLAM pipeline, the Camera HAL3 capture path, and the core Vulkan hardware-buffer import mechanics for camera frames. This chapter does not re-derive any of that; instead it goes one layer up and one layer sideways. Up: the Jetpack XR SDK's Kotlin scene-graph API and the OpenXR 1.1 contract that Android XR headsets expose natively (as opposed to the translation layer ARCore provides on phones). Sideways: Qualcomm's Snapdragon Spaces XDK, a separate XR SDK lineage that predates Android XR and is now converging toward it, and Monado, the open-source Linux OpenXR runtime that plays the same architectural role on desktop Linux that Android XR's system runtime plays on-device. It closes with two corrections to API names that appear in this book's chapter outline but do not exist in the shipping specifications, and a deeper mathematical treatment of the environmental HDR spherical-harmonics lighting data that Ch87 §8 introduces but does not derive.

---

## Table of Contents

1. [Android XR Platform and the Jetpack XR SDK](#1-android-xr-platform-and-the-jetpack-xr-sdk)
   1. [Session Lifecycle](#11-session-lifecycle)
   2. [Entity-Component Scene Graph](#12-entity-component-scene-graph)
   3. [SpatialCapabilities](#13-spatialcapabilities)
   4. [Components: Movable, Resizable, Anchorable, Interactable](#14-components-movable-resizable-anchorable-interactable)
2. [OpenXR on Android: Native Contract vs. Translation Layer](#2-openxr-on-android-native-contract-vs-translation-layer)
   1. [XR_KHR_android_create_instance](#21-xr_khr_android_create_instance)
   2. [Android Surface Swapchains: Correcting XrSwapchainImageAndroidKHR](#22-android-surface-swapchains-correcting-xrswapchainimageandroidkhr)
   3. [Phones vs. Headsets: Translation Layer vs. Native Runtime](#23-phones-vs-headsets-translation-layer-vs-native-runtime)
3. [Qualcomm Snapdragon Spaces XDK](#3-qualcomm-snapdragon-spaces-xdk)
   1. [OpenXR Profile and Hand-Interaction Extensions](#31-openxr-profile-and-hand-interaction-extensions)
   2. [OpenXRFeature-Based Passthrough](#32-openxrfeature-based-passthrough)
   3. [Convergence with Android XR](#33-convergence-with-android-xr)
4. [Monado as the Open Linux XR Runtime (Forward Reference)](#4-monado-as-the-open-linux-xr-runtime-forward-reference)
5. [Vulkan Camera Import: Deeper Mechanics](#5-vulkan-camera-import-deeper-mechanics)
   1. [VkExternalFormatANDROID and the Opaque Format Contract](#51-vkexternalformatandroid-and-the-opaque-format-contract)
   2. [The VkSamplerYcbcrConversion pNext Chain](#52-the-vksamplerycbcrconversion-pnext-chain)
6. [Spatial Plane Detection: Correcting VK_EXT_plane_detection](#6-spatial-plane-detection-correcting-vk_ext_plane_detection)
   1. [The Real Mechanism: XR_EXT_plane_detection and XR_EXT_spatial_plane_tracking](#61-the-real-mechanism-xr_ext_plane_detection-and-xr_ext_spatial_plane_tracking)
   2. [Why Plane Detection Cannot Live in Vulkan](#62-why-plane-detection-cannot-live-in-vulkan)
7. [Environmental HDR Light Estimation: The Spherical-Harmonics Math](#7-environmental-hdr-light-estimation-the-spherical-harmonics-math)
   1. [L2 Spherical Harmonics and Irradiance Reconstruction](#71-l2-spherical-harmonics-and-irradiance-reconstruction)
   2. [From Irradiance to Diffuse IBL](#72-from-irradiance-to-diffuse-ibl)
8. [Integrations](#8-integrations)
9. [References](#9-references)

---

## 1. Android XR Platform and the Jetpack XR SDK

Ch87 §9 introduces the Jetpack XR SDK's module layout (`androidx.xr.runtime`, `scenecore`, `compose`, `arcore`) and shows an early, simplified `Session.create(activity)` call. The SDK has since settled on an asynchronous, coroutine-based session-creation contract and a considerably richer entity-component scene graph than that earlier sample suggests; this section covers the current shape of both.

### 1.1 Session Lifecycle

`androidx.xr.runtime.Session` is the root object for all spatial state — perception, rendering anchors, and scene content. Session creation is asynchronous because it may need to bind to the system XR service and negotiate capabilities with the device, and it returns a sealed result type rather than throwing:

```kotlin
import androidx.xr.runtime.Session
import androidx.xr.runtime.SessionCreateSuccess
import androidx.xr.runtime.SessionCreatePermissionsNotGranted

lifecycleScope.launch {
    when (val result = Session.create(this@MainActivity)) {
        is SessionCreateSuccess -> {
            val session = result.session
            // session.config, session.scene, session.state are now valid
        }
        is SessionCreatePermissionsNotGranted -> {
            // result.permissions lists what must be granted before retrying
        }
        else -> {
            // other failure subtypes (unsupported device, etc.)
        }
    }
}
```

[Source](https://developer.android.com/develop/xr/jetpack-xr-sdk/session)

A `Session` exposes `session.config: Config` to declare which runtime features an app needs (plane tracking, hand tracking, depth, environment lighting) and a `Session.StateEvent`/`session.state` flow that the app collects each frame to obtain the current `TrackingState`. Compose applications reach the active session through `LocalSession.current` inside a `Subspace { }` composable rather than passing it down manually.

### 1.2 Entity-Component Scene Graph

`androidx.xr.scenecore` models spatial content as a tree of `Entity` objects rooted at the session's `ActivitySpace`, using a right-handed, meters-based coordinate system (unlike ARCore's `ArAnchor` poses, which are meters in an OpenGL-style right-handed *camera*-relative frame — SceneCore poses are relative to the entity's parent in the scene graph, not the camera). The entity types most application code touches:

| Entity type | Purpose |
|---|---|
| `PanelEntity` | Hosts 2D Android UI (a `SurfaceView` or Jetpack Compose subtree) as a flat spatial panel |
| `GltfModelEntity` | Renders a loaded glTF 2.0 model as 3D spatial content |
| `SurfaceEntity` | A drawable surface for custom GPU rendering (stereo video, game engine output) |
| `AnchorEntity` | Binds a subtree to a real-world anchor (plane, point, or persisted anchor) |
| `ActivitySpace` | The root of the per-activity scene graph |

A `GltfModelEntity` is created in two steps — asynchronously loading the model data, then instantiating the entity from it:

```kotlin
val gltfModel = GltfModel.create(session, "models/robot.glb")
val entity = GltfModelEntity.create(session, gltfModel)
entity.setPose(Pose(Vector3(0f, 0f, -2f), Quaternion.Identity))
entity.setScale(0.5f)
```

[Source](https://developer.android.com/develop/xr/jetpack-xr-sdk/scenecore)

Every `Entity` supports `setPose()`, `setScale()`, `setEnabled()`, and parent/child management via `addChild()` / `setParent()`; scale and pose are always relative to the parent entity, so moving a `PanelEntity` moves every `GltfModelEntity` parented to it.

### 1.3 SpatialCapabilities

Because the same Kotlin code path targets both phones running ARCore-backed Android XR emulation and dedicated headsets, apps query `session.getSpatialCapabilities(): SpatialCapabilities` at runtime rather than branching on device model. `SpatialCapabilities.hasCapability(capability: Int): Boolean` tests bitflags including `SPATIAL_CAPABILITY_UI` (spatial panels), `SPATIAL_CAPABILITY_3D_CONTENT` (glTF entities), `SPATIAL_CAPABILITY_PASSTHROUGH_CONTROL`, `SPATIAL_CAPABILITY_APP_ENVIRONMENT`, `SPATIAL_CAPABILITY_SPATIAL_AUDIO`, and `SPATIAL_CAPABILITY_EMBED_ACTIVITY`.

```kotlin
val caps = session.getSpatialCapabilities()
if (caps.hasCapability(SpatialCapabilities.SPATIAL_CAPABILITY_3D_CONTENT)) {
    // safe to instantiate GltfModelEntity
}
```

[Source](https://developer.android.com/develop/xr/jetpack-xr-sdk)

**Note: needs verification.** The exact enumerated constant set for `SpatialCapabilities` is drawn from Android XR developer documentation surveyed during research for this chapter; the SDK was still at a pre-1.0 (`1.0.0-alphaNN`) release during that research pass (as already flagged in Ch87 §9), so flag names and bit values should be re-checked against the `androidx.xr.scenecore` reference for the SDK version actually targeted by a project before shipping.

### 1.4 Components: Movable, Resizable, Anchorable, Interactable

SceneCore separates *what an entity is* (its type) from *what it can do* (its behavior), attached as composable `Component` objects — a pattern absent from Ch87's simpler entity examples:

- `MovableComponent.createSystemMovable()` lets the platform's system UI (not app code) drag an entity in response to controller/hand input; `createAnchorable()` additionally lets the user drop the entity onto a detected plane, converting its parent to an `AnchorEntity`.
- `ResizableComponent.create()` exposes system-provided resize handles and reports size changes via a listener.
- `InteractableComponent.create()` routes low-level pointer/hover/select input events to app code for entities that need custom interaction beyond move/resize.
- `AnchorPlacement.createForPlanes(planeTypeFilter, planeSemanticFilter)` configures which detected-plane types (horizontal/vertical) and semantic labels (floor, wall, table) a `MovableComponent` is allowed to anchor onto.

```kotlin
val movable = MovableComponent.createAnchorable(
    session,
    anchorPlacement = setOf(AnchorPlacement.createForPlanes())
)
entity.addComponent(movable)
```

[Source](https://developer.android.com/develop/xr/jetpack-xr-sdk/scenecore)

This component model is SceneCore-specific scaffolding built on top of the perception data (planes, anchors) that ARCore and OpenXR supply underneath — the plane and anchor *detection* itself is the subject of §6 and Ch87 §6–7, not this section.

## 2. OpenXR on Android: Native Contract vs. Translation Layer

### 2.1 XR_KHR_android_create_instance

Every Android OpenXR application must chain an `XrInstanceCreateInfoAndroidKHR` structure onto `XrInstanceCreateInfo::next` when calling `xrCreateInstance`, because unlike desktop OpenXR, Android has no implicit process-global JNI context the loader can assume:

```c
typedef struct XrInstanceCreateInfoAndroidKHR {
    XrStructureType    type;   // XR_TYPE_INSTANCE_CREATE_INFO_ANDROID_KHR
    const void* XR_MAY_ALIAS   next;
    void*               applicationVM;        // JavaVM*
    void*               applicationActivity;   // jobject (android.app.Activity)
} XrInstanceCreateInfoAndroidKHR;
```

`XR_KHR_android_create_instance` is a ratified instance extension (extension number 9, revision 3) and is a *hard requirement* on Android — an OpenXR runtime on the platform is expected to reject `xrCreateInstance` calls that omit it. [Source](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_KHR_android_create_instance.html)

For a native (NDK, non-Kotlin) OpenXR application this is the loader-level equivalent of what `Session.create(activity)` handles implicitly for Jetpack XR apps in §1.1 — the Activity/JavaVM plumbing is unavoidable on Android whether the app talks to the runtime through Kotlin or raw C.

### 2.2 Android Surface Swapchains: Correcting XrSwapchainImageAndroidKHR

This book's chapter outline for Ch166 names a struct `XrSwapchainImageAndroidKHR` as part of Android's OpenXR swapchain model, by analogy with the generic per-backend image structs `XrSwapchainImageVulkanKHR` and `XrSwapchainImageOpenGLESKHR`. **This struct does not exist.** A direct check of the OpenXR registry and of vendor-mirrored `openxr_platform.h` headers finds no `XrSwapchainImageAndroidKHR` type anywhere in the specification.

What Android actually has is a different, non-analogous mechanism, defined by `XR_KHR_android_surface_swapchain`: a single function, `xrCreateSwapchainAndroidSurfaceKHR`, that creates an OpenXR swapchain backed by an `android.view.Surface` instead of an array of enumerable per-image handles:

```c
XrResult xrCreateSwapchainAndroidSurfaceKHR(
    XrSession                     session,
    const XrSwapchainCreateInfo*  info,
    XrSwapchain*                  swapchain,
    jobject*                      surface);  // out: android.view.Surface
```

The generic swapchain model (`xrEnumerateSwapchainImages` returning an array of `XrSwapchainImageVulkanKHR`/`XrSwapchainImageOpenGLESKHR` structs, each wrapping a `VkImage` or GL texture name the app renders into directly) assumes the graphics API owns image allocation and the runtime just consumes finished images. The Android surface path inverts this: the runtime hands the app an opaque `Surface` object, and any Android graphics producer that can write to a `Surface` — `MediaCodec`, a `SurfaceView`, or a `GLES`/EGL context created against it — can be the swapchain's image source, without OpenXR ever seeing individual image handles. This is architecturally closer to how `ANativeWindow`-backed rendering works elsewhere in the Android graphics stack (see Ch85) than to the rest of OpenXR's swapchain model.

**Note: needs verification / correction.** Treat `XrSwapchainImageAndroidKHR` in this book's outline as referring to this `xrCreateSwapchainAndroidSurfaceKHR` mechanism, not to a literal struct of that name. [Source](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_KHR_android_surface_swapchain.html)

### 2.3 Phones vs. Headsets: Translation Layer vs. Native Runtime

Ch87 §9 already shows ARCore's OpenXR extension surface on phones (`XR_EXT_hand_tracking`, `XR_ANDROID_trackables`, and related extensions). What that section does not spell out is that the two form factors implement the OpenXR contract at architecturally different depths:

- **On phones**, ARCore's own C API (`ArSession`, described in depth in Ch87) is the platform's native interface; OpenXR support is layered as a loader/runtime shim that translates OpenXR calls onto ARCore's session, tracking, and anchor primitives. An app can use either interface, but the ARCore API is closer to the metal.
- **On Android XR headsets**, OpenXR 1.1 *is* the native application interface — there is no separate proprietary session API underneath that OpenXR is translating onto. Platform documentation states this explicitly: Android XR "does not simply offer an OpenXR loader that translates calls to a proprietary API"; instead the system compositor and perception services expose their state through the OpenXR runtime contract directly. [Source](https://developer.android.com/develop/xr/openxr)

The practical consequence for application developers: OpenXR extension coverage and behavior can differ between an ARCore-on-phone target and an Android-XR-headset target even though both advertise OpenXR 1.1, because on phones the extension is implemented by a translation shim with ARCore's own capabilities and limits underneath, while on headsets it is implemented natively by the system runtime. The Jetpack XR SDK (§1) is designed to abstract over this difference for Kotlin-level app code, but native OpenXR applications talking to the loader directly are exposed to it.

## 3. Qualcomm Snapdragon Spaces XDK

Ch87 §14 covers Snapdragon Spaces' feature comparison against ARCore and its foveated-rendering extensions (`GL_QCOM_texture_foveated`, `VK_EXT_fragment_density_map`, `XR_FB_foveation_vulkan`) in detail. This section adds the OpenXR-extension-level detail and platform-strategy context Ch87 does not cover.

### 3.1 OpenXR Profile and Hand-Interaction Extensions

Snapdragon Spaces is built as a Unity/Unreal/native XDK on top of an underlying OpenXR runtime running on Snapdragon XR platforms, and its interaction model has moved onto the ratified `XR_EXT_hand_interaction` extension — a hand-tracking *interaction profile* (bindings for aim/pinch/grip poses and gesture-based input actions), distinct from the lower-level `XR_EXT_hand_tracking` joint-pose extension both ARCore and Snapdragon Spaces also support. `XR_EXT_hand_interaction` is a ratified instance extension. [Source](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_EXT_hand_interaction.html)

### 3.2 OpenXRFeature-Based Passthrough

Passthrough camera compositing in the current Snapdragon Spaces SDK is exposed through an `OpenXRFeature`-derived plugin architecture (matching the OpenXR Unity plugin's general extension-feature pattern) rather than a bespoke Snapdragon-only passthrough API, which is a shift from earlier Spaces SDK releases that shipped a standalone passthrough manager. Practically, this means passthrough is toggled and configured the same way other OpenXR-mediated features are — through a feature object registered with the OpenXR Unity/Native loader — rather than through Snapdragon-specific singleton calls.

**Note: needs verification.** The `OpenXRFeature` passthrough architecture and the `XR_EXT_hand_interaction` adoption were both surfaced via general web search of Qualcomm's developer documentation rather than a primary Khronos or Qualcomm spec page; verify current SDK version behavior against `spaces.qualcomm.com` documentation before citing exact class names in application code.

### 3.3 Convergence with Android XR

Qualcomm ships a "Snapdragon Spaces Compatibility Plugin" positioned as a migration aid for moving Spaces-based applications onto Android XR, including mapping Spaces' Qualcomm-specific hand-tracking types (QCHTi) onto the standard `XR_EXT_hand_tracking` surface Android XR expects. This, combined with §3.1's move to ratified Khronos extensions instead of Qualcomm-proprietary ones, indicates Snapdragon Spaces is trending toward becoming a hardware-enablement layer *under* Android XR on Qualcomm silicon rather than a fully separate application-facing SDK competing with it — though both remain independently shippable targets at the time of writing.

## 4. Monado as the Open Linux XR Runtime (Forward Reference)

Ch87 §15 already covers Monado's architecture in depth — its `xrt_device`/`xrt_prober` driver model, SLAM integration via Basalt/OpenVINS behind `t_imu.h`/`t_camera_slam.h`, SteamVR DRM-lease interop, and ALVR/WiVRn streaming runtimes — as the open-source counterpart to Android XR's system runtime on desktop and embedded Linux. That material is not repeated here.

What is worth noting at this layer is scope: everything in §2's OpenXR contract discussion (instance creation, swapchains, hand-interaction and plane-detection extensions) is runtime-agnostic OpenXR surface. An application written against the OpenXR 1.1 core plus `XR_EXT_hand_tracking`/`XR_EXT_spatial_plane_tracking` can, in principle, run unmodified against Android XR's native runtime, ARCore's translation shim, or Monado — provided each runtime implements the extensions the app requires. Monado's extension coverage (including experimental plane-detection support, tracked in its changelog) determines which of this chapter's Android-oriented extension set is actually usable on desktop Linux; see Ch87 §15 and Ch27 for the Linux-side runtime and compositor details.

## 5. Vulkan Camera Import: Deeper Mechanics

Ch87 already shows a working `VK_ANDROID_external_memory_android_hardware_buffer` import path, including `VkAndroidHardwareBufferPropertiesANDROID`, `VkSamplerYcbcrConversionCreateInfo`, and `VkImportAndroidHardwareBufferInfoANDROID`. This section fills in two mechanics that sample glosses over: how an opaque camera pixel format is communicated to Vulkan at all, and the exact `pNext` chain the sampler side requires to consume it.

### 5.1 VkExternalFormatANDROID and the Opaque Format Contract

Camera frames delivered as `AHardwareBuffer`s frequently use vendor-private YUV layouts that have no corresponding `VkFormat` enum value — the whole point of `VK_ANDROID_external_memory_android_hardware_buffer` is to let Vulkan consume buffers whose format Vulkan itself cannot name. The extension handles this with `VkExternalFormatANDROID`, chained onto both the image and the sampler-conversion create-info structs:

```c
typedef struct VkExternalFormatANDROID {
    VkStructureType    sType;  // VK_STRUCTURE_TYPE_EXTERNAL_FORMAT_ANDROID
    void*              pNext;
    uint64_t           externalFormat;  // opaque, driver-defined
} VkExternalFormatANDROID;
```

`vkGetAndroidHardwareBufferPropertiesANDROID(device, buffer, &props)` — where `props.pNext` chains a `VkAndroidHardwareBufferFormatPropertiesANDROID` — is called first to *discover* the buffer's `externalFormat` value (and, if the buffer happens to map to a real `VkFormat`, that too). When the buffer has no native `VkFormat`, `VkImageCreateInfo.format` must be set to `VK_FORMAT_UNDEFINED` and a `VkExternalFormatANDROID` populated with the discovered `externalFormat` chained onto `VkImageCreateInfo.pNext` — the driver uses the opaque token, not a `VkFormat` enum, to select the correct hardware sampling path for that specific vendor pixel layout. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_ANDROID_external_memory_android_hardware_buffer.html)

### 5.2 The VkSamplerYcbcrConversion pNext Chain

Because `VK_FORMAT_UNDEFINED` carries no information a sampler can use, the *same* `VkExternalFormatANDROID.externalFormat` value discovered in §5.1 must also be chained onto `VkSamplerYcbcrConversionCreateInfo.pNext` when creating the `VkSamplerYcbcrConversion` used to sample the image — this is a hard cross-dependency between the image side and the sampler side that is easy to get wrong, since the two structs are created in different parts of typical Vulkan camera-import code:

```c
VkExternalFormatANDROID extFmt = {
    .sType = VK_STRUCTURE_TYPE_EXTERNAL_FORMAT_ANDROID,
    .externalFormat = discoveredExternalFormat,  // from vkGetAndroidHardwareBufferPropertiesANDROID
};
VkSamplerYcbcrConversionCreateInfo ycbcrInfo = {
    .sType = VK_STRUCTURE_TYPE_SAMPLER_YCBCR_CONVERSION_CREATE_INFO,
    .pNext = &extFmt,
    .format = VK_FORMAT_UNDEFINED,   // format comes from externalFormat instead
    .ycbcrModel = VK_SAMPLER_YCBCR_MODEL_CONVERSION_YCBCR_709,
    .ycbcrRange = VK_SAMPLER_YCBCR_RANGE_ITU_NARROW,
    // component mapping, chroma offsets, filter as in Ch87's sample
};
```

The resulting `VkSamplerYcbcrConversion` is then wrapped in a `VkSamplerYcbcrConversionInfo` chained onto both the `VkSampler` and the `VkImageView` used to sample the camera image — every combined image sampler touching an opaque-format camera buffer needs this three-way agreement (image, image view, sampler) on the same conversion object, or the driver has no defined behavior for interpreting the opaque bits. On Android XR headsets with dual or quad passthrough cameras, this import path runs once per camera stream, each potentially yielding a distinct `externalFormat` token if the sensors differ, which callers must not assume are interchangeable.

## 6. Spatial Plane Detection: Correcting VK_EXT_plane_detection

This book's outline for Ch166 names `VK_EXT_plane_detection` as a Vulkan extension for spatial plane queries. **No such Vulkan extension exists.** A search of the official Vulkan registry (`vk.xml`) finds no extension name containing "plane_detection" anywhere in the specification — only unrelated display-plane (`VK_KHR_display`) and image-plane-memory (multi-planar format) names exist, neither of which concerns real-world spatial planes.

### 6.1 The Real Mechanism: XR_EXT_plane_detection and XR_EXT_spatial_plane_tracking

Plane detection is exclusively an OpenXR-layer concept. The original extension, `XR_EXT_plane_detection`, defines a plane-detector object and query functions:

- `xrCreatePlaneDetectorEXT` / `xrDestroyPlaneDetectorEXT` — manage an `XrPlaneDetectorEXT` handle scoped to a session
- `xrBeginPlaneDetectionEXT` — asynchronously starts a detection pass over a spatial volume
- `xrGetPlaneDetectionStateEXT` — polls whether a pass has completed
- `xrGetPlaneDetectionsEXT` — retrieves the resulting `XrPlaneDetectorLocationEXT` array (pose, extent, semantic label per plane)
- `xrGetPlanePolygonBufferEXT` — retrieves a plane's boundary polygon when finer-than-rectangle geometry is needed

The Khronos registry marks `XR_EXT_plane_detection` as **deprecated**, superseded by `XR_EXT_spatial_plane_tracking`, part of OpenXR's newer generalized "spatial entity" extension family (alongside `XR_EXT_spatial_entity`) that models planes, meshes, and other real-world structures as typed components of a common spatial-entity object rather than each having its own bespoke detector API. [Source](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_EXT_plane_detection.html)

Android's own extension set builds on this newer spatial-entity family directly: Android XR's plane and scene-understanding surface is documented as `XR_EXT_spatial_plane_tracking`, `XR_EXT_spatial_entity`, `XR_ANDROID_trackables`, `XR_ANDROID_scene_meshing`, `XR_ANDROID_spatial_entity_bound_anchor`, and `XR_ANDROID_spatial_object_tracking`, rather than the older per-feature `XR_EXT_plane_detection`. [Source](https://developer.android.com/develop/xr/openxr/extensions)

### 6.2 Why Plane Detection Cannot Live in Vulkan

The absence of a Vulkan-side plane API is not an oversight — it reflects a layering boundary that holds throughout OpenXR's design. Vulkan's contract ends at "here is a swapchain image, submit rendering commands against it"; it has no concept of world-tracked semantic geometry, camera pose history, or SLAM state, and no mechanism for asynchronous scene-understanding queries that complete over multiple frames. Spatial planes are runtime/perception-layer state — produced by the same tracking pipeline that produces head pose and hand joints (Ch87's SLAM material) — and are consumed by the application at the OpenXR level, then used to *drive* ordinary Vulkan draw calls (placing geometry at a detected plane's pose, clipping against its polygon). The plane detector object itself never touches the Vulkan device, queue, or command buffer; it is queried through `XrSession`, not `VkDevice`.

## 7. Environmental HDR Light Estimation: The Spherical-Harmonics Math

Ch87 §8 already shows the ARCore API surface for this feature — `ArLightEstimate_getEnvironmentalHdrAmbientSphericalHarmonics()` returning a 27-element `float[]` (9 RGB coefficients), HDR cubemap acquisition via `ArLightEstimate_acquireEnvironmentalHdrCubemap`, and a GLSL split-sum specular IBL shader that consumes the cubemap. What that section does not derive is *why* 9 RGB coefficients are sufficient to represent ambient lighting, or how to turn them into usable irradiance — this section fills that gap.

### 7.1 L2 Spherical Harmonics and Irradiance Reconstruction

Spherical harmonics are an orthonormal basis for functions defined on a sphere, analogous to how a Fourier series decomposes a periodic 1D function into sine/cosine terms. Truncating the basis at band *l = 2* ("L2 SH") yields exactly 9 basis functions:

$$
Y_{00},\; Y_{1,-1}, Y_{1,0}, Y_{1,1},\; Y_{2,-2}, Y_{2,-1}, Y_{2,0}, Y_{2,1}, Y_{2,2}
$$

— which is precisely why ARCore's Environmental HDR mode reports 27 floats: 9 coefficients × 3 color channels (RGB), one full L2 SH projection of the scene's incident radiance per channel. Reconstructing *irradiance* (not raw radiance) at a surface normal **n** from these 9 coefficients per channel uses the closed-form result from Ramamoorthi and Hanrahan's irradiance-environment-map derivation: convolving incident radiance with the cosine lobe collapses to a simple weighted sum of the same 9 SH coefficients, with fixed weighting constants $c_1 \ldots c_5$ that come from the cosine lobe's own SH projection:

$$
E(\mathbf{n}) = c_1 L_{22}(n_x^2 - n_y^2) + c_3 L_{20} n_z^2 + c_4 L_{00} - c_5 L_{20}
+ 2c_1(L_{2,-2} n_x n_y + L_{2,1} n_x n_z + L_{2,-1} n_y n_z)
+ 2c_2(L_{11} n_x + L_{1,-1} n_y + L_{10} n_z)
$$

with $c_1 = 0.429043$, $c_2 = 0.511664$, $c_3 = 0.743125$, $c_4 = 0.886227$, $c_5 = 0.247708$ — constants derived once from the SH coefficients of the clamped-cosine BRDF lobe and reusable for any L2 SH lighting environment, not just ARCore's. [Source](https://cseweb.ucsd.edu/~ravir/papers/envmap/envmap.pdf)

This is why the API is cheap enough to run every frame on a phone: instead of integrating a hemisphere of incident radiance per shaded pixel, the app evaluates one 9-term polynomial in the surface normal, using coefficients ARCore already computed once per frame from its own internal estimate of the scene's lighting.

### 7.2 From Irradiance to Diffuse IBL

The formula in §7.1 yields *irradiance*, which is the correct input for a Lambertian diffuse term directly — $L_o^{diffuse}(\mathbf{n}) = \frac{\rho}{\pi} E(\mathbf{n})$ for surface albedo $\rho$ — with no further integration needed, because the cosine-lobe convolution is already baked into the $c_1 \ldots c_5$ constants. This is the natural counterpart to the specular split-sum path Ch87 §8 already shows operating on the *cubemap* form of the same environmental estimate: SH coefficients drive the low-frequency diffuse term cheaply, while the higher-resolution HDR cubemap drives the specular term where high-frequency detail (reflections of scene structure) actually matters visually. Using the cubemap for diffuse lighting too would be correct but wasteful — a full hemisphere convolution over cubemap texels for every shaded fragment, when the same result is already available as a 9-term polynomial evaluation from data ARCore computed once per frame.

## 8. Integrations

- **[Chapter 87: Android AR: ARCore Architecture and Camera HAL Integration](ch87-android-ar-arcore.md)** — the base chapter this one expands: ARCore SLAM internals, Camera HAL3 pipeline, `ArSession` lifecycle, plane/anchor/light-estimation APIs, and the Monado/SteamVR Linux AR material referenced in §4 above.
- **[Chapter 27: VR/AR — OpenXR Session Model](../part-07a-gpu-apis/ch27-vr-ar.md)** — the platform-agnostic OpenXR session, swapchain, and action-binding model that §2 of this chapter specializes for Android; covers Monado's non-Android runtime role in more depth.
- **[Chapter 86: Android Vulkan](ch86-android-vulkan.md)** — Android's broader Vulkan platform extensions (`VK_ANDROID_native_buffer`, `VK_KHR_android_surface`) that §5's hardware-buffer import mechanics build on.
- **[Chapter 85: Android SurfaceFlinger](ch85-android-surfaceflinger.md)** — the `ANativeWindow`/`Surface` compositing model that §2.2's `xrCreateSwapchainAndroidSurfaceKHR` mechanism reuses instead of a per-image swapchain.

## 9. References

- [Jetpack XR SDK — Session](https://developer.android.com/develop/xr/jetpack-xr-sdk/session) — `Session.create()` async lifecycle, `SessionCreateSuccess`/`SessionCreatePermissionsNotGranted`
- [Jetpack XR SDK — SceneCore](https://developer.android.com/develop/xr/jetpack-xr-sdk/scenecore) — Entity types, Component API (`MovableComponent`, `ResizableComponent`, `InteractableComponent`, `AnchorPlacement`)
- [Jetpack XR SDK overview](https://developer.android.com/develop/xr/jetpack-xr-sdk) — `SpatialCapabilities` and module layout
- [Jetpack XR SDK — ARCore for Jetpack XR](https://developer.android.com/develop/xr/jetpack-xr-sdk/arcore) — perception capability surface (planes, anchors, hand/face tracking, depth, device pose) via `androidx.xr.runtime.Session.configure()`
- [Android XR and OpenXR](https://developer.android.com/develop/xr/openxr) — native OpenXR 1.1 platform contract on Android XR headsets vs. ARCore's translation layer on phones
- [Android XR OpenXR extensions](https://developer.android.com/develop/xr/openxr/extensions) — current spatial/plane/scene-understanding extension set (`XR_EXT_spatial_plane_tracking`, `XR_EXT_spatial_entity`, `XR_ANDROID_trackables`, `XR_ANDROID_scene_meshing`, `XR_ANDROID_spatial_entity_bound_anchor`, `XR_ANDROID_spatial_object_tracking`)
- [XR_KHR_android_create_instance](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_KHR_android_create_instance.html) — `XrInstanceCreateInfoAndroidKHR` structure
- [XR_KHR_android_surface_swapchain](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_KHR_android_surface_swapchain.html) — `xrCreateSwapchainAndroidSurfaceKHR`, the real Android swapchain mechanism (corrects the plan's `XrSwapchainImageAndroidKHR`)
- [XR_EXT_hand_interaction](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_EXT_hand_interaction.html) — ratified hand-interaction profile used by current Snapdragon Spaces
- [XR_EXT_plane_detection](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_EXT_plane_detection.html) — deprecated plane-detector API; confirms no Vulkan-side equivalent exists
- [XR_EXT_spatial_plane_tracking](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XR_EXT_spatial_plane_tracking.html) — current spatial-entity-based plane tracking extension
- [VK_ANDROID_external_memory_android_hardware_buffer](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_ANDROID_external_memory_android_hardware_buffer.html) — `vkGetAndroidHardwareBufferPropertiesANDROID`, `VkExternalFormatANDROID`, opaque external-format contract
- Ramamoorthi, R. and Hanrahan, P., ["An Efficient Representation for Irradiance Environment Maps"](https://cseweb.ucsd.edu/~ravir/papers/envmap/envmap.pdf), SIGGRAPH 2001 — the L2 spherical-harmonics irradiance reconstruction formula used in §7.1
- [Monado developer site](https://monado.freedesktop.org/) — open-source Linux OpenXR runtime, forward-referenced in §4; see Ch87 §15 for full architectural coverage

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
