# Chapter 250: The Meta XR SDK Suite and Horizon OS Platform — Core, Interaction, Spatial, Audio, Haptics, Voice, Platform, Simulator, and MRUK on Quest

> **Part**: Part XIX — Android Graphics
> **Audience**: Graphics application developers building Meta Quest applications; readers of Ch27 (OpenXR/Monado on desktop Linux) and Ch166 (Android XR/Jetpack XR) who want the Meta-specific application-SDK layer that sits above the same OpenXR contract.
> **Status**: First draft — 2026-08-29

Meta Quest headsets run Horizon OS, a fork of Android — Meta's own documentation states this directly, though (as §2 discusses) it does not name specific shared components. On top of that Android substrate, Horizon OS ships a proprietary OpenXR 1.0/1.1 runtime — the same standard Ch27 documents for Monado on desktop Linux and Ch166 documents for Android XR's system runtime. Meta does not expose that runtime to application developers directly; instead it ships a suite of engine-integrated and native SDKs — collectively branded the **Meta XR SDK**, plus the newer, engine-free **Meta Spatial SDK** — that wrap the runtime's OpenXR core plus a large surface of Meta-proprietary `XR_FB_*` and `XR_META_*` extensions behind Unity, Unreal, Kotlin, and native C/C++ APIs. This chapter documents that suite: what each package actually does, which OpenXR extensions it wraps where that mapping is documented, how Horizon OS itself is structured as a platform, and how the pieces depend on one another.

This is deliberately narrower than Ch27 or Ch166. Those chapters cover the OpenXR session/swapchain/action model and the Android platform contract in depth; this chapter does not re-derive that material. It instead answers a different question: given that contract, what does Meta actually ship on top of it, and where does each SDK's abstraction meet the underlying extension it wraps?

---

## Table of Contents

1. [Where the Suite Sits](#1-where-the-suite-sits)
2. [Horizon OS as a Platform](#2-horizon-os-as-a-platform)
3. [Android, Native, and OpenXR Integration](#3-android-native-and-openxr-integration)
4. [Meta XR Core SDK](#4-meta-xr-core-sdk)
5. [Meta XR Interaction SDK: Essentials vs. Full](#5-meta-xr-interaction-sdk-essentials-vs-full)
6. [Meta Spatial SDK](#6-meta-spatial-sdk)
7. [Meta XR Audio SDK](#7-meta-xr-audio-sdk)
8. [Meta XR Haptics SDK](#8-meta-xr-haptics-sdk)
9. [Meta XR Platform SDK](#9-meta-xr-platform-sdk)
10. [Meta XR Voice SDK](#10-meta-xr-voice-sdk)
11. [Meta Mixed Reality Utility Kit (MRUK)](#11-meta-mixed-reality-utility-kit-mruk)
12. [Meta XR Simulator and Synthetic Environment Builder](#12-meta-xr-simulator-and-synthetic-environment-builder)
13. [How the Suite Composes](#13-how-the-suite-composes)
14. [Integrations](#14-integrations)
15. [References](#15-references)

---

## 1. Where the Suite Sits

Every application-layer package in this chapter ships as a Unity Package Manager (UPM) package, an Unreal plugin, a Maven artifact consumed from Kotlin/Gradle (Meta Spatial SDK, §6), or a native C/C++ library — but in every case it runs inside the application process and talks to Horizon OS's system-level OpenXR runtime (and, for the Platform SDK, to Meta's backend services over HTTPS) rather than replacing any part of it. None of these SDKs is a system component of Horizon OS itself. §2 covers Horizon OS as the platform underneath; §3 covers the native/NDK integration path that Unity and Unreal quietly wrap and that Meta Spatial SDK uses more directly.

This makes the suite structurally analogous to two things this book already covers:

- **Jetpack XR on Android XR** (Ch166 §1): a vendor's opinionated application SDK sitting above a standard OpenXR system runtime, adding a scene graph, component model, and convenience APIs that a raw OpenXR application would otherwise have to build itself. §6 revisits this comparison directly for Meta Spatial SDK, which is a closer structural analogue to Jetpack XR than the Unity/Unreal-centric Core SDK is.
- **Monado on desktop Linux** (Ch27 §3): the open-source implementation of the runtime layer these SDKs sit *above* — Monado is a peer to Horizon OS's proprietary runtime, not to any SDK in this chapter. Both runtimes implement the same core OpenXR specification, but OVRPlugin's session negotiates a large set of Meta vendor extensions — `XR_FB_spatial_entity`, `XR_FB_haptic_pcm`, `XR_FB_scene`, and others documented throughout this chapter — that Monado does not implement. An application hard-coded against those extensions loses the corresponding functionality (spatial anchors, PCM haptics, Scene Model) if pointed at Monado's runtime, even though its base OpenXR calls would still succeed.

Meta made this layering explicit in its own developer communications: as of the "Oculus All In on OpenXR" announcement, the proprietary VRAPI/CAPI native APIs that predated OpenXR on Quest are deprecated, and "all new features are available on the OpenXR backend only." [Source: Meta — Oculus All In on OpenXR](https://developers.meta.com/horizon/blog/oculus-all-in-on-openxr-deprecates-proprietary-apis/) Everything downstream in this chapter — Interaction, Spatial SDK, Audio, Haptics, Platform, Voice, MRUK — is therefore, to varying degrees, an engine-side or Kotlin-side convenience wrapper over an OpenXR extension set, not an alternative to OpenXR. The native path those wrappers sit on is documented directly in §3.

---

## 2. Horizon OS as a Platform

Meta's own documentation states plainly that "Horizon OS is built on Android" and, more specifically, that "Meta Horizon OS is built on the Android Open Source Project." [Source: Meta — Platforms overview](https://developers.meta.com/horizon/discover/platforms/); [Source: Meta — Features overview](https://developers.meta.com/horizon/documentation/android-apps/features-overview/) This AOSP lineage traces back to the original Oculus Go in 2018 by secondary account, but no page in Meta's own developer documentation states a specific AOSP or Android API-level version number that Horizon OS forks from.

**Note: needs verification.** This chapter's opening paragraph, and the earlier text of this chapter, described Horizon OS as sharing "the ART runtime, Zygote process model, and SurfaceFlinger compositor pipeline" with mainline Android. That claim is a reasonable architectural inference from "built on AOSP" — consistent with how Ch85 and Ch164 describe stock Android's own architecture — but it is an inference, not something Meta states by name anywhere found during this chapter's research. Treat any specific-component claim (ART, Zygote, SurfaceFlinger by name) as inferred rather than Meta-confirmed until a primary source names them directly.

**Licensing to third-party hardware makers.** In April 2024, Meta opened Horizon OS licensing to Asus (a dedicated gaming headset), Lenovo (a productivity/mixed-reality headset), and a limited-edition Xbox-branded Quest. [Source: TechCrunch — Meta opens Quest OS to third-party headset makers](https://techcrunch.com/2024/04/22/meta-opens-quest-os-to-third-party-headset-makers-taps-lenovo-and-xbox-as-partners/) This was a licensing deal, not an open-source release — Horizon OS itself was not published as buildable AOSP source. Later reporting (2025, secondary source, not Meta's own newsroom) indicates Meta paused the Asus ROG and Lenovo third-party headsets to focus on its own first-party hardware; treat this as a licensing initiative that stalled rather than one that reversed outright, and flag the current shipping status of any specific third-party Horizon OS device as unconfirmed and in flux. [Source: UploadVR — Meta pauses third-party Horizon OS headsets](https://www.uploadvr.com/meta-pauses-third-party-horizon-os-headsets/)

**App types: Panels vs. Immersive.** Horizon OS distinguishes two first-class app shapes. **Panels** are rectangular surfaces displaying 2D app content — ordinary native Android apps (Java/Kotlin/Jetpack) rendered as floating windows, with a default size of 1024dp×640dp and a documented minimum of 384dp×500dp. [Source: Meta — Panels](https://developers.meta.com/horizon/design/panels/) **Immersive apps** are the full-OpenXR-session VR/MR experiences that every other SDK in this chapter targets. Running several Activities as separate panels simultaneously is a plain Android capability — standard Activity launch flags, no Meta-specific SDK required. [Source: Meta — Features overview](https://developers.meta.com/horizon/documentation/android-apps/features-overview/)

**Distribution.** Three tiers are documented: the **Meta Horizon Store** (renamed from the Meta Quest Store), the primary storefront; **App Lab**, direct-to-customer distribution that bypasses store review; and **sideloading**, which requires enabling "Unknown Sources" in the Meta Horizon companion app and forgoes updates, Home Library integration, and Horizon services entirely. [Source: Meta — Distribution options](https://developers.meta.com/horizon/policy/distribution-options/); [Source: Meta — Introducing App Lab](https://developers.meta.com/horizon/blog/introducing-app-lab-a-new-way-to-distribute-oculus-quest-apps/) Meta is actively merging App Lab into the Store as a single unified surface.

**OS versioning vs. SDK versioning.** These are two independent numbering tracks. Horizon OS system software ships as "vNN" builds — for example, v76.1023–v76.1027 rolling out the week of 2025-02-18, and v78.1027/1028 later. [Source: Meta — Release notes](https://developers.meta.com/horizon/release-notes/); [Source: Meta Quest Help — release notes](https://www.meta.com/help/quest/172903867975450/) This is unrelated to the per-package SDK version numbers cited elsewhere in this chapter (Platform SDK 77.0, §9; Core SDK/Interaction SDK Essentials 205.0, §13) — a system-software upgrade and an SDK-package upgrade happen on separate schedules and are not the same event.

**Security and permissions.** **Boundary** (renamed from "Guardian") is Horizon OS's play-space safety system — a just-in-time edge warning as a user approaches the bounds of their tracked space — and is not documented as a hard security or process-sandboxing boundary in the way the term might suggest. A genuine, confirmed OS-level extension to Android's stock permission model does exist for camera access: passthrough camera data requires both the standard `android.permission.CAMERA` (or a Horizon-specific `horizonos.permission.HEADSET_CAMERA`) permission, layered on top of Android's ordinary runtime-permission flow. [Source: Meta — Passthrough Camera](https://developers.meta.com/horizon/documentation/android-apps/passthrough-camera/)

---

## 3. Android, Native, and OpenXR Integration

Every engine-level abstraction elsewhere in this chapter — Core SDK's `OVRPlugin` (§4), Platform SDK's native init sequence (§9), the Haptics SDK's C renderer (§8), the Audio SDK's `OVR_Audio.h` (§7) — ultimately sits on the same native Android/OpenXR foundation. This section documents that foundation directly, for readers building against it without Unity or Unreal, or who want to know precisely what the engine wrappers are hiding.

**Loader.** Meta requires the standard **Khronos OpenXR Android Loader** exclusively, distributed as the Gradle dependency `org.khronos.openxr:openxr_loader_for_android:1.0.34` — Meta's documentation notes that versions below 1.0.34 crash on Quest. Two Android-specific Khronos extensions gate startup: **`XR_KHR_loader_init_android`**, which an application resolves via `xrGetInstanceProcAddress(XR_NULL_HANDLE, "xrInitializeLoaderKHR", ...)` and calls with an `XrLoaderInitInfoAndroidKHR` struct carrying the JNI `JavaVM` and `Activity` references before any other OpenXR call is legal, and **`XR_KHR_android_create_instance`**, which passes those same Android-specific parameters into the subsequent `xrCreateInstance` call. [Source: Meta — OpenXR Support for Meta Quest Headsets](https://developers.meta.com/horizon/documentation/native/android/mobile-openxr/)

**Manifest.** The confirmed required intent-filter block for an immersive native app:

```xml
<intent-filter>
  <action android:name="android.intent.action.MAIN" />
  <category android:name="org.khronos.openxr.intent.category.IMMERSIVE_HMD" />
  <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
```

[Source: Meta — OpenXR Support for Meta Quest Headsets](https://developers.meta.com/horizon/documentation/native/android/mobile-openxr/)

**Official native SDK and samples.** The reference implementation and sample set lives at **`github.com/meta-quest/Meta-OpenXR-SDK`**, which supersedes the deprecated VrApi Mobile SDK (VrApi itself has been unsupported since 2022-08-31). Samples are organized per feature — `XrHandsFB`, `XrSceneModel`, `XrVirtualKeyboard`, `XrPassthrough`, `XrSpatialAnchor`, and others — each with its own README, loaded collectively via a top-level `Samples/build.gradle`. [Source: Meta — OpenXR Mobile SDK](https://developers.meta.com/horizon/documentation/native/android/mobile-intro/); [Source: GitHub — meta-quest/Meta-OpenXR-SDK](https://github.com/meta-quest/Meta-OpenXR-SDK)

**Relationship to OVRPlugin.** `libOVRPlugin.so` is a real native library shipped under `Assets/Plugins/Android` in Unity Quest builds — it appears in build logs alongside `libOculusXRPlugin.so` and `libunity.so` — and Meta's documentation states that OVRPlugin lets "Unity talk to the OpenXR, VRAPI, and CAPI backends." No primary document found during this chapter's research states explicitly that OVRPlugin internally calls the same `xrInitializeLoaderKHR`/`xrCreateInstance` sequence documented above; Meta's own comparison of OpenXR, VrApi, and LibOVR describes the three as application-facing API choices without describing OVRPlugin's internals at all. Treat "OVRPlugin uses this same native loader path" as a plausible architectural inference — both ultimately negotiate the same Horizon OS OpenXR runtime — rather than a confirmed fact.

**Further vendor extensions for a hand-written native frame loop.** Beyond the spatial-entity chain covered in §11 and the haptics extensions covered in §8, three more confirmed `XR_FB_*` vendor extensions matter to anyone writing a native compositor loop directly: **`XR_FB_passthrough`** (with companion **`XR_FB_triangle_mesh`**, added in OpenXR 1.0.20) for real-time passthrough compositing; **`XR_FB_display_refresh_rate`** (`xrEnumerateDisplayRefreshRatesFB`, `xrRequestDisplayRefreshRateFB`); and **`XR_FB_color_space`**, which lets an application declare the color space it authored content in. [Source: Meta — Implement Passthrough](https://developers.meta.com/horizon/documentation/native/android/mobile-passthrough/); [Source: Meta — OpenXR Support for Meta Quest Headsets](https://developers.meta.com/horizon/documentation/native/android/mobile-openxr/)

**Note: needs verification.** Meta's native "Core Concepts" page is confirmed to exist at the URL cited below, but it defines `XrSession` only at a glossary level and does not document the frame loop, `XrEventDataSessionStateChanged` handling, or the Android `onResume`/`onPause` lifecycle tie-in — it explicitly directs readers to the Khronos `hello_xr` sample and the community OpenXR Tutorial for that material instead. [Source: Meta — Core Concepts](https://developers.meta.com/horizon/documentation/native/android/mobile-openxr-core-concepts/) Ch27's generic OpenXR session-lifecycle material is the better citation for the frame-loop and session-state mechanics themselves. Manifest metadata keys sometimes cited in secondary material (`com.oculus.vr.focusaware`, `com.oculus.handtracking.frequency`, `com.oculus.supportedDevices`) were not independently verified during this chapter's research and should not be quoted as exact strings without checking current native samples directly.

---

## 4. Meta XR Core SDK

Core SDK is the foundation every other Unity/Unreal package in this chapter (other than Interaction SDK Essentials) depends on. Meta's own framing: it "provides the foundation for every Quest app: rendering, tracking, and platform integration, plus mixed reality primitives like Passthrough, Anchors, and Scene." [Source: Meta XR Core SDK Overview](https://developers.meta.com/horizon/documentation/unity/unity-core-sdk/)

Concretely, Core SDK bundles **OVRPlugin**, the native plugin documented directly in §3 that is the actual bridge between engine-side script APIs and Horizon OS's runtime backends. The component names surfacing in Unity — `OVRManager`, `OVRCameraRig` — retain the `OVR` prefix inherited from the pre-2019 "Oculus Integration" Unity asset, and Meta states these remain the supported API surface even after the underlying transport migrated from the proprietary VRAPI to OpenXR. [Source: Meta — OVRCameraRig documentation](https://developers.meta.com/horizon/documentation/unity/unity-ovrcamerarig/)

Two migrations matter for anyone integrating Core SDK today:

- **XR plugin provider**: Unity's older **Oculus XR Plugin** (last documented at v4.5.1) is deprecated and scheduled for removal; it remains required only for Meta XR SDK v73 and earlier on Unity 2022+. The forward path is Unity's own **OpenXR Plugin** (v1.15.1 at time of writing, requiring Unity 6+ and Meta XR SDK v74+), which routes Core SDK's calls through Unity's standard OpenXR provider layer rather than a Meta-specific one. [Source: Meta — Unity XR Plugin Management](https://developers.meta.com/horizon/documentation/unity/unity-xr-plugin/)
- **Meta-specific OpenXR feature package**: because Unity's generic OpenXR Plugin has no knowledge of Meta's proprietary extensions, a separate **"Unity OpenXR: Meta"** package (v2.1.0 at time of writing) is required specifically to unlock Meta features not in the OpenXR core spec — Depth API occlusion is the example Meta calls out explicitly. [Source: Meta — Unity XR Plugin Management](https://developers.meta.com/horizon/documentation/unity/unity-xr-plugin/)

This two-package split (Unity's generic OpenXR provider, plus Meta's extension feature-set package) is the mechanism by which `OVRManager`/`OVRCameraRig` continue to compile against Meta-proprietary `XR_FB_*`/`XR_META_*` extensions while the underlying transport is standard OpenXR rather than a closed API. Interaction SDK is documented as usable either as a Core-SDK-independent package (§5) or riding on Core SDK's OpenXR plumbing via "Unity XR." [Source: Meta — Interaction SDK + Unity XR](https://developers.meta.com/horizon/documentation/unity/unity-isdk-getting-started-unityxr/)

Distribution is primarily through the Unity Asset Store as the "Meta XR All-In-One SDK," which bundles Core SDK alongside Interaction, Audio, Haptics, Platform, and the other packages in this chapter as a single multi-package download with dependencies pre-resolved. [Source: Unity Asset Store — Meta XR All-In-One SDK](https://assetstore.unity.com/packages/tools/integration/meta-xr-all-in-one-sdk-269657)

**Note: needs verification.** The exact literal UPM package identifier for Core SDK (by analogy with the confirmed `com.meta.xr.sdk.audio`, likely `com.meta.xr.sdk.core`) was not found verbatim on any fetched page during this chapter's research and should be confirmed against the extracted `package.json` inside the UPM tarball before being quoted as an exact string elsewhere.

---

## 5. Meta XR Interaction SDK: Essentials vs. Full

Meta ships two tiers of the same interaction framework, and the split is a dependency boundary, not just a feature boundary:

- **Meta XR Interaction SDK Essentials** is a lightweight, platform-independent UPM package with **no dependency on Meta XR Core SDK**. It provides the base Interactor/Interactable framework, hand-pose detection, and visual indicators, and Meta recommends pairing it with Unity's `XR Hands` package for engine-agnostic hand-tracking work that does not need any other Meta-specific package. [Source: Meta XR Interaction SDK Essentials (UPM)](https://developers.meta.com/horizon/downloads/package/meta-xr-interaction-sdk/)
- **Meta XR Interaction SDK (full)** requires Core SDK and adds the Meta-hardware-specific superset: prebuilt grab/poke/ray/locomotion presets, hand *and* body pose detection, "Quick Actions" rapid-setup tooling, and full sample scenes. [Source: Meta XR Interaction SDK (UPM)](https://developers.meta.com/horizon/downloads/package/meta-xr-interaction-sdk-ovr-integration/); [Source: Interaction SDK overview](https://developers.meta.com/horizon/documentation/unity/unity-isdk-interaction-sdk-overview/)

Both tiers share the same architectural pattern: **`IInteractor`** components attach to a hand or controller and initiate an interaction; **`IInteractable`** components attach to scene objects and respond to it. The SDK composes these into higher-level constructs — an "Interaction Rig" grouping the interactors a hand or controller exposes, "Interactor Groups" for prioritizing among simultaneously-candidate interactors, and a "Pointer Events" system layered on top for UI-style interaction. The three baseline interaction types are **Poke** (near-field, e.g. pressing a virtual button), **Ray** (far-field pointing, including a distance-grab variant), and **Grab** (near-field hand/controller grasp), plus a **Locomotion**/teleportation interactor family. [Source: Interaction SDK overview](https://developers.meta.com/horizon/documentation/unity/unity-isdk-interaction-sdk-overview/)

The hand-tracking data source underneath this pattern has itself migrated onto OpenXR on the same timeline as Core SDK's transport: **OpenXR hand-skeleton support was added in Interaction SDK v71**, and the **legacy `OVRHand` skeleton was deprecated in v78** — concrete version markers for exactly when Interaction SDK stopped depending on Meta's pre-OpenXR hand-tracking path and started depending on the standard OpenXR hand-joint model that Ch27 §7 (`XR_EXT_hand_tracking`) documents generically. [Source: Interaction SDK overview](https://developers.meta.com/horizon/documentation/unity/unity-isdk-interaction-sdk-overview/)

**Note: needs verification.** Literal UPM package-ID strings (`com.meta.xr.sdk.interaction`, `com.meta.xr.sdk.interaction.ovr`) and any Unreal-specific interaction-component class name were not independently confirmed against primary text during this chapter's research; verify against the shipped `package.json`/Unreal plugin descriptor before citing as exact identifiers elsewhere in this book. Whether Meta Spatial SDK (§6) shares code or concepts with Interaction SDK, beyond both implementing an interactor/interactable-style pattern, was likewise not confirmed by any primary source.

---

## 6. Meta Spatial SDK

Meta Spatial SDK is a Kotlin-based SDK, distinct from any game engine, for building Horizon OS immersive apps "using familiar Android languages, tools, and libraries." Meta frames it as additive to mobile: a developer can build a new immersive app from scratch, or add spatial features to an existing ordinary Android app. [Source: Meta Spatial SDK overview](https://developers.meta.com/horizon/documentation/spatial-sdk/spatial-sdk-explainer/)

**Not the same thing as Jetpack XR.** These are confirmed as distinct, competing approaches, not variants of the same technology and not built on top of each other. Meta Spatial SDK is Meta's own proprietary application layer for Meta devices; Google's Jetpack XR SDK (Ch166) is an Android-focused framework built on the OpenXR standard, targeting the broader multi-vendor Android XR device ecosystem. [Source: Android Developers — Jetpack XR SDK](https://developer.android.com/develop/xr/jetpack-xr-sdk); [Source: UploadVR — Meta Spatial SDK](https://www.uploadvr.com/meta-spatial-sdk/) This makes Spatial SDK a closer structural analogue to Jetpack XR than Core SDK is (§1): both are vendor-opinionated, engine-free application frameworks sitting above an OpenXR-based runtime, for different and non-overlapping device ecosystems.

**Architecture: entity-component-system (ECS).** Entities are lightweight containers that group components together and hold no data of their own. Components are pure data — "what an entity *is* or *has*, but not what it *does*." Systems hold all of the logic, querying entities by whatever combination of components they carry. Meta's own worked example: a drone entity composed of a `Transform`, a `Mesh` (loaded from glTF/glXF), a `Grabbable` component, and an `Audio` component. [Source: Meta Spatial SDK — ECS](https://developers.meta.com/horizon/documentation/spatial-sdk/spatial-sdk-ecs/)

The confirmed base Activity class is **`AppSystemActivity`** (`class MainActivity : AppSystemActivity()`); custom components and systems are registered against a `componentManager`/`systemManager` inside `onCreate`. [Source: Meta Spatial SDK — template walkthrough](https://developers.meta.com/horizon/documentation/spatial-sdk/spatial-sdk-template-walkthrough/) VR-specific behavior is opted into via **`VRFeature`**, a confirmed real class imported from `com.meta.spatial.vr` and registered through a `registerFeatures()` call.

**MRUK on Spatial SDK is a separate implementation, not a shared binding.** It is distributed as a distinct artifact, `meta-spatial-sdk-mruk.aar` (plus a `kotlinx-serialization-json` dependency), implemented as a `SpatialFeature` — `MRUKFeature` — registered the same way as `VRFeature`. It exposes `MRUKRoom`, `MRUKAnchor`, `MRUKPlane`, and `MRUKVolume` types, `loadSceneFromDevice`/`loadSceneFromJsonString` entry points, `raycastRoom`/`raycastEnvironment` queries, `AnchorMeshSpawner`/`AnchorProceduralMesh` helpers, and keyboard/QR-code `Tracker` support. Meta's own documentation frames this MRUK layer explicitly as "a set of utilities and tools on top of Scene API" — the same underlying on-device Scene API that §11's Unity/Unreal MRUK wraps, but via an independent Kotlin binding rather than a shared cross-platform one. [Source: Meta Spatial SDK — MRUK](https://developers.meta.com/horizon/documentation/spatial-sdk/spatial-sdk-mruk/)

**Distribution.** Spatial SDK ships on **Maven Central** under the group `com.meta.spatial`, with confirmed exact coordinates including `com.meta.spatial:meta-spatial-sdk`, `:meta-spatial-sdk-toolkit`, `:meta-spatial-sdk-vr`, `:meta-spatial-sdk-physics`, and `:meta-spatial-sdk-mruk` (the last as an `.aar`), alongside a Gradle KSP plugin, `com.meta.spatial.plugin`. [Source: Maven Central — com.meta.spatial:meta-spatial-sdk](https://central.sonatype.com/artifact/com.meta.spatial/meta-spatial-sdk) At time of writing, Maven Central lists `0.13.2` as the latest published version (documentation examples reference `0.13.0`), with older `0.8.0` artifacts still present — the SDK is best described as pre-1.0 and actively iterating, rather than cited by a single fixed version number.

**Samples and tooling.** The official samples repository is **`github.com/meta-quest/Meta-Spatial-SDK-Samples`** (MIT-licensed, five open-sourced "Showcase" full applications), with a companion **`github.com/meta-quest/Meta-Spatial-SDK-Templates`** repository backing the project/component/system templates offered by a **Meta Horizon Plugin for Android Studio**. That plugin also adds a "Data Model Inspector" for live ECS debugging. A separate visual authoring tool, **Meta Spatial Editor**, builds and exports scenes for Spatial SDK apps to consume. [Source: Meta Spatial SDK overview](https://developers.meta.com/horizon/documentation/spatial-sdk/spatial-sdk-explainer/)

**Note: needs verification.** The exact class name `PassthroughFeature` was not confirmed by any fetched page (only `VRFeature` and `MRUKFeature` were confirmed by name) — passthrough is documented as a supported capability, but not under a verified literal class name. The exact class name(s) for embedding 2D Android View content as an interactable panel in a 3D scene (documented conceptually, plausibly `Panel`/`Panel2D`) were likewise not confirmed. Whether Spatial SDK talks to the OpenXR runtime directly or is itself routed through an OVRPlugin-style native layer, as Core SDK is (§3, §4), was not stated by any primary source found and should be treated as an open architectural question, not an assumed fact.

---

## 7. Meta XR Audio SDK

Meta XR Audio SDK is the direct successor to the **Oculus Spatializer** plugin, which is now end-of-life and unsupported beyond version 47; the feature set carried forward under the new name. [Source: Meta XR Audio SDK Features](https://developers.meta.com/horizon/documentation/unity/meta-xr-audio-sdk-features/)

Its core capabilities:

- **HRTF-based spatialization** of mono/point-source audio, giving each sound source a perceived 3D position via head-related transfer function filtering.
- **Ambisonic spatialization**, implemented as what Meta calls an "advanced spherical harmonic binaural renderer" — Meta's documentation frames this as offering better frequency response and externalization than decoding ambisonics to a virtual speaker array first and then binauralizing that. [Source: Meta XR Audio SDK Features](https://developers.meta.com/horizon/documentation/unity/meta-xr-audio-sdk-features/); [Source: Meta XR Audio SDK for Unreal — Ambisonics](https://developers.meta.com/horizon/documentation/unreal/meta-xr-audio-sdk-unreal-ambisonic/)
- **Near-field rendering**, approximating the acoustic diffraction effects that become audible when a virtual sound source is closer than roughly one meter to the listener's head.
- **Room acoustics**: early reflections plus late reverberation, driven by scene geometry, to give distance and enclosure cues beyond pure HRTF panning.

Unity distribution is confirmed as UPM package `com.meta.xr.sdk.audio` (identifier read directly from the distributed package tarball's filename), configured under Project Settings → Audio by setting both the Spatializer Plugin and the Ambisonics Decoder Plugin to "Meta XR Audio." [Source: Meta — Audio SDK requirements and setup](https://developers.meta.com/horizon/documentation/unity/meta-xr-audio-sdk-unity-req-setup/)

A notable integration constraint: **the plugins are mutually exclusive per project**. Unity's native Meta XR Audio spatializer must not be installed alongside FMOD or Wwise — Meta ships separate, dedicated Meta XR Audio plugin packages for FMOD and for Wwise instead, and a distinct native plugin path for Unreal. [Source: Meta — Audio SDK requirements and setup](https://developers.meta.com/horizon/documentation/unity/meta-xr-audio-sdk-unity-req-setup/); [Source: Meta XR Audio SDK — FMOD/Unreal example](https://developers.meta.com/horizon/documentation/unreal/meta-xr-audio-sdk-fmod-unreal-example/)

Below all of the engine wrappers sits a native C API, header `OVR_Audio.h`, confirmed present in a public mirror of Unreal Engine's third-party source tree at `Engine/Source/ThirdParty/Oculus/LibOVRAudio/LibOVRAudio/include/OVR_Audio.h`. Confirmed function names from that header include `ovrAudio_SpatializeMonoSourceLR` and `ovrAudio_FinishTailInterleaved`. [Source: `OVR_Audio.h` mirror](https://github.com/a3pelawi/UnrealEngine/blob/master/Engine/Source/ThirdParty/Oculus/LibOVRAudio/LibOVRAudio/include/OVR_Audio.h)

Unlike Haptics (§8) and MRUK (§11), **the Audio SDK has no corresponding OpenXR extension**. No search of Meta's OpenXR extension documentation or the extension registry during this chapter's research turned up an `XR_FB_spatial_audio` or equivalent. Audio spatialization on Quest is purely an engine/native-library concern — the OpenXR runtime carries no audio state, and the spatializer runs entirely inside the application's audio-processing graph. This is a genuine architectural difference from the other SDKs in this chapter, not a research gap: treat it as confirmed absence.

**Note: needs verification.** The function name `ovrAudio_SpatializeMonoSourceInterleaved` sometimes cited in secondary material was not found in the primary header; the confirmed interleaved-output function in the mirrored header is `ovrAudio_FinishTailInterleaved`. Use only the names confirmed above until the current header is re-checked directly.

---

## 8. Meta XR Haptics SDK

Haptics SDK is a media-based API: instead of driving controller vibration motors with raw amplitude values per frame, an application plays back a pre-authored `.haptic` clip, and the SDK's renderer converts that clip into the motor commands appropriate for whichever controller is attached.

Clips are authored in **Meta Haptics Studio**, a desktop tool, and stored as JSON. The clip format is device-agnostic: normalized (0.0–1.0) amplitude envelopes under `signals.continuous.envelopes` as a series of `{time, value}` control points, an optional "emphasis" channel for short transient hits layered on top of the continuous envelope, and a normalized frequency envelope (0.0 = a round/soft texture, 1.0 = a sharp texture). [Source: `facebook/meta-haptics-sdk`](https://github.com/facebook/meta-haptics-sdk)

The renderer and engine bindings are **open-sourced under the MIT license** at `github.com/facebook/meta-haptics-sdk`, with a layout worth knowing if you need to read past the black-box API:

- `core/renderer_parametric` — a C++ `ParametricHapticRenderer` that interprets the normalized `.haptic` clip data.
- `core/renderer_c` — a lightweight C "Haptic Renderer" that generates output samples at a target hardware sample rate.
- `core/haptic_data_parametric` — format-conversion utilities for the clip data.
- `interfaces/unity`, `interfaces/unreal` — the engine-specific bindings.
- `resources/acf` — Actuator Configuration File (ACF) templates: JSON5 files that map the device-agnostic normalized clip data onto a specific actuator's real hardware range, via fields like `frequency_min`/`frequency_max`, `gain`, and separate tuning for the continuous and emphasis channels. [Source: `facebook/meta-haptics-sdk`](https://github.com/facebook/meta-haptics-sdk)

The Unity-facing API exposes two interchangeable entry points with identical playback semantics: `HapticClipPlayer`, a plain C# class, and `HapticSource`, a `MonoBehaviour`/Editor component wrapping the same functionality; either plays a `.haptic` clip on the left, right, or both controllers, and a `HapticClip` can be wired in the Inspector or instantiated at runtime as a `ScriptableObject`. [Source: Meta — Haptics SDK integration guide](https://developers.meta.com/horizon/documentation/unity/unity-haptics-sdk-integrate/) The Unreal equivalent is `UMetaXRHapticsPlayerComponent`. [Source: Meta — `UMetaXRHapticsPlayerComponent` reference](https://developers.meta.com/horizon/reference/unreal-haptics/v201/class_u_meta_x_r_haptics_player_component/)

Hardware capability varies by controller generation: Touch Plus and Touch Pro controllers have variable-frequency actuators and play back the full amplitude-and-frequency envelope; original Quest 2 Touch controllers have a fixed vibration frequency, so the same clip plays back with amplitude variation only and no perceptible frequency modulation. The SDK auto-detects which controller is connected and adapts. [Source: Meta — Haptics SDK overview](https://developers.meta.com/horizon/documentation/unity/unity-haptics-sdk/)

Underneath the clip abstraction, Quest's OpenXR runtime exposes the raw capability via **`XR_FB_haptic_pcm`**: an application calls `xrApplyHapticFeedback()` with an `XrHapticPcmVibrationFB` struct carrying a PCM waveform buffer describing the vibration pattern, available on Quest 2 and later. A second relevant vendor extension, **`XR_FB_touch_controller_pro`**, adds OpenXR-level support for the Touch Pro controller's additional haptic elements (localized thumb and trigger haptics). [Source: Meta — Haptic Feedback (native)](https://developers.meta.com/horizon/documentation/native/android/mobile-openxr-haptic/) The Haptics SDK's `.haptic`/ACF authoring pipeline is best understood as a content and renderer layer built to target this PCM extension, sparing the application from hand-authoring raw PCM buffers.

---

## 9. Meta XR Platform SDK

Platform SDK is Meta's identity, entitlement, and commerce layer: achievements, app invites, "Destinations" (deep-linkable in-app locations), DLC, in-app purchases, leaderboards, cloud-synced saves, and matchmaking, exposed in parallel across native C++, Android, Unity, and Unreal. [Source: Meta — Platform SDK entitlement check (native)](https://developers.meta.com/horizon/documentation/native/ps-entitlement-check/); [Source: Meta XR Platform SDK (UPM) v77.0](https://developers.meta.com/horizon/downloads/package/meta-xr-platform-sdk/77.0/)

The version pinned during this chapter's research, **77.0** (dated 2025-06-05), gives a concrete sense of release granularity: its changelog includes a fix to `Leaderboards.writeEntry`'s return type (`LeaderboardUpdateStatus`) and documentation updates to `asset_file_download_update`. [Source: Meta XR Platform SDK (UPM) v77.0](https://developers.meta.com/horizon/downloads/package/meta-xr-platform-sdk/77.0/)

The native entitlement-check flow, which every Quest app is required to perform within 10 seconds of launch, follows a message-pump pattern rather than a blocking call:

1. Initialize the platform connection: `ovr_PlatformInitializeAndroid()` (Quest) or `ovr_PlatformInitializeWindows()` (PC VR / Link), or their `*Asynchronous` variants.
2. Issue `ovr_Entitlement_GetIsViewerEntitled()`, which returns a request handle rather than a result.
3. Drain the async message queue with `ovr_PopMessage()`, check `ovr_Message_GetType()` against the constant `ovrMessage_Entitlement_GetIsViewerEntitled`, and check `ovr_Message_IsError()` to determine success or failure. [Source: Meta — Platform SDK entitlement check](https://developers.meta.com/horizon/documentation/native/ps-entitlement-check/)

For server-side verification, Meta exposes `POST https://graph.oculus.com/$APP_ID/verify_entitlement`, which returns a `success` boolean and a `grant_time` Unix timestamp — useful for backend services that need to confirm entitlement independently of the client. [Source: Meta — Platform SDK entitlement check](https://developers.meta.com/horizon/documentation/native/ps-entitlement-check/)

In Unity, the equivalent entry point is `Oculus.Platform.Core.Initialize()`. [Source: Meta — Set up Platform SDK entitlements (Unity)](https://developers.meta.com/horizon/documentation/unity/unity-platform-entitlements/) The app's identity is supplied to all of these entry points via an app-ID constant (referred to in native samples as `OCULUS_APP_ID`), obtained from the Meta Horizon developer dashboard for the specific app.

**Note: needs verification.** The claim that the native shared library is named `libovrplatformloader.so` could not be confirmed from any fetched documentation page during this chapter's research and should not be treated as fact without a direct citation to Meta's native Android integration guide.

---

## 10. Meta XR Voice SDK

Voice SDK adds hands-free voice interaction to a Quest app, built on **Wit.ai**, Meta's natural-language-understanding service, which performs speech-to-text and intent/entity extraction server-side (cloud NLU, not on-device). It is distributed as several related pieces — Voice Command & TTS, Voice SDK Dictation, Voice SDK Composer — and is also bundled inside the Meta XR All-in-One SDK. [Source: Meta — Voice SDK overview](https://developers.meta.com/horizon/documentation/unity/voice-sdk-overview/)

The primary integration point is **`AppVoiceExperience`**, a component holding a reference to a Wit.ai App Config asset; the typical pattern is calling `voiceExperience.Activate()` on a controller-button press to begin capturing and streaming audio to Wit.ai, with the component firing Unity Events as recognition results arrive. [Source: Meta — Voice SDK tutorials](https://developers.meta.com/horizon/documentation/unity/voice-sdk-tutorials-1/) A separate `AppDictationExperience` class handles free-form dictation rather than intent matching; Meta's reference docs still expose it under the legacy `Oculus.Voice.Dictation` namespace as of the v67 API reference. [Source: Meta — `AppDictationExperience` reference (v67)](https://developers.meta.com/horizon/reference/voice/v67/class_oculus_voice_dictation_app_dictation_experience/)

The SDK ships more than 50 built-in intents, entities, and traits out of the box and supports dynamic, app-state-driven entity personalization — for example, restricting a "select item" intent's valid entity values to whatever is actually in the player's inventory at that moment — alongside real-time, low-latency transcription. [Source: Meta — Voice SDK overview](https://developers.meta.com/horizon/documentation/unity/voice-sdk-overview/)

Because processing happens through Wit.ai's cloud service rather than on-device, Meta's documentation explicitly requires that apps give users "clear and comprehensive information about, and consent to" voice data collection before any processing occurs, publish a privacy policy covering the collection, use, retention, and processing of voice data, and obtain explicit consent as required by applicable law. [Source: Meta — Voice SDK overview](https://developers.meta.com/horizon/documentation/unity/voice-sdk-overview/) This chapter did not find documentation of an on-device/offline NLU mode distinct from the Wit.ai cloud path; treat the cloud-only model as the current state unless a specific offline mode is independently confirmed.

The package predates its current name — it originally shipped as "Presence Platform Voice SDK." [Source: Meta — Presence Platform Voice SDK and Passthrough announcement](https://developers.meta.com/horizon/blog/presence-platform-voice-sdk-and-passthrough-now-available/)

---

## 11. Meta Mixed Reality Utility Kit (MRUK)

MRUK is a Unity/Unreal convenience layer over Horizon OS's on-device Scene API: room mesh, spatial anchors, and semantic labels for the objects in a user's physical space, intended to replace the older, lower-level `OVRSceneManager`. (§6 documents an independent Kotlin implementation of the same idea for Meta Spatial SDK.)

The Unity API is accessed through a singleton, `MRUK.Instance` (namespace `Meta.XR.MRUtilityKit`), exposing `MRUKRoom` and `MRUKAnchor` types for the room and its individual tracked objects respectively. [Source: Meta — Migrating from OVRSceneManager to MRUK](https://developers.meta.com/horizon/documentation/unity/unity-scene-migrate-mruk/); [Source: `MRUKRoom` reference](https://developers.meta.com/horizon/reference/mruk/v201/class_meta_x_r_m_r_utility_kit_m_r_u_k_room/); [Source: `MRUKAnchor` reference](https://developers.meta.com/horizon/reference/mruk/v77/class_meta_x_r_m_r_utility_kit_m_r_u_k_anchor/)

Meta's own migration guide is explicit about *why* MRUK replaced the older `OVRSceneManager`/`OVRSceneAnchor` model: with `OVRSceneManager`, applications had to poll `OVRSceneAnchor` pose data periodically to keep tracked content aligned with the physical room. MRUK instead keeps anchors fixed in Unity world space once loaded and re-anchors the *camera's* tracking updates against them, which removes the need for per-anchor polling and simplifies content synchronization. [Source: Meta — Migrating from OVRSceneManager to MRUK](https://developers.meta.com/horizon/documentation/unity/unity-scene-migrate-mruk/)

Every anchor MRUK exposes is labeled with the semantic classification assigned during the OS-level Space Setup flow, where the user walks through their room and labels the surfaces and furniture it contains. On the Unreal side, `AMRUKAnchor` exposes this as a `SemanticClassifications` array (`TArray<FString>`), plus a `HasLabel()` convenience check and `AMRUKRoom::GetAnchorsByLabel()`; label string constants are collected in `FMRUKLabels` (`FMRUKLabels::Floor`, `::Table`, `::Couch`, and others including wall, ceiling, door frame, and window frame). [Source: Meta — Unreal Scene supported semantic labels](https://developers.meta.com/horizon/documentation/unreal/unreal-scene-supported-semantic-labels/)

This semantic-labeling and room-mesh capability is built on a chain of OpenXR spatial-entity extensions, several of which this chapter's research confirmed directly against Meta's native OpenXR documentation and the Khronos registry:

- **`XR_FB_spatial_entity`** — the foundational extension: world-locked anchors as Entity-Components, the dependency every other Meta spatial-entity extension builds on. [Source: Khronos — `XR_FB_spatial_entity`](https://registry.khronos.org/OpenXR/specs/1.0/man/html/XR_FB_spatial_entity.html)
- **`XR_FB_scene`** and **`XR_FB_scene_capture`** — the extension pair backing the Scene Model itself and the Space Setup capture flow ("lets users walk around and capture their scene to generate a Scene Model"), with the semantic label carried as a space component, `XR_SPACE_COMPONENT_TYPE_SEMANTIC_LABELS_FB`, inside `XR_FB_scene`. [Source: Meta — OpenXR Scene Overview](https://developers.meta.com/horizon/documentation/native/android/openxr-scene-overview/)
- **`XR_FB_spatial_entity_query`**, **`XR_FB_spatial_entity_storage`**, **`XR_FB_spatial_entity_container`** — query, cross-session persistence, and container-relationship extensions layered on the base spatial-entity model. [Source: Meta — OpenXR Scene Overview](https://developers.meta.com/horizon/documentation/native/android/openxr-scene-overview/)
- **`XR_META_spatial_entity_semantic_label`** — a newer META-vendor extension defining semantic labels for spatial entities within the `XR_FB_spatial_entity` framework, distinct from (and superseding parts of) the older `XR_FB_scene`-embedded semantic-labels component.

MRUK's `MRUKRoom`/`MRUKAnchor` API is, in effect, the engine-side convenience wrapper over exactly this extension chain, in the same relationship that Ch27's Monado material bears to the base `XR_EXT_*` extensions it implements, and that Ch166 §6 describes for `XR_EXT_plane_detection`/`XR_EXT_spatial_plane_tracking` on Android XR.

**Note: needs verification.** No documentation reviewed for this chapter mentioned MRUK-level support for Shared Spatial Anchors (multi-user colocation); Meta documents that capability separately elsewhere in its Presence Platform material, and it should not be assumed to be part of MRUK itself without a direct citation.

---

## 12. Meta XR Simulator and Synthetic Environment Builder

Meta XR Simulator is a desktop OpenXR runtime and API layer that stands in for a physical headset during development, explicitly described as "compatible with any OpenXR application" and supporting Unity, Unreal, and native development against simulated Quest 2, Quest 3, and Quest Pro devices. [Source: Meta XR Simulator overview](https://developers.meta.com/horizon/documentation/unity/xrsim-intro/); [Source: Meta XR Simulator getting started (native)](https://developers.meta.com/horizon/documentation/native/xrsim-getting-started/) Its Unity package identifier is confirmed as `com.meta.xr.simulator`. [Source: Meta XR Simulator overview](https://developers.meta.com/horizon/documentation/unity/xrsim-intro/)

Mixed-reality-specific simulation — passthrough, Scene, anchors, depth — runs through a client/server split: the **Synthetic Environment Server (SES)**, a standalone application, feeds this data to the app under test over the simulated OpenXR runtime. SES itself is built using the **Synthetic Environment Builder**, a separate Unity-based authoring tool that provides a scene-annotation tool and a scene-data-recorder tool, letting a developer capture and package their *own* room layout as a testable synthetic environment rather than being limited to whatever Meta bundles by default. [Source: Meta — Simulate a Mixed Reality Environment](https://developers.meta.com/horizon/documentation/unity/xrsim-ses/); [Source: Meta — Build Your Own SES (Synthetic Environment Builder)](https://developers.meta.com/horizon/documentation/unreal/xrsim-synthetic-environment-builder/) Meta ships the Synthetic Environment Builder as a distinct downloadable package in its own right — "Meta XR Simulator - Synthetic Environment Builder" — separate from the Simulator core install. [Source: Meta XR Simulator - Synthetic Environment Builder (download)](https://developers.meta.com/horizon/downloads/package/meta-xr-simulator-synthetic-environment-builder/)

Out of the box, Meta bundles three realistic test rooms (game room, living room, bedroom) plus six additional "feature rooms" purpose-built for exercising mixed-reality compatibility edge cases. [Source: Meta — Built-in Test Rooms](https://developers.meta.com/horizon/documentation/unity/xrsim-heroscenes/) The Simulator's own documentation notes a recent architectural rework — a "newly released Standalone XR Simulator," with earlier material moved to an archived documentation section — and flags that older Core SDK releases may not be fully compatible with the current Simulator, which is a useful debugging first-check if a project's Simulator session fails to connect after a Core SDK upgrade. [Source: Meta XR Simulator overview](https://developers.meta.com/horizon/documentation/unity/xrsim-intro/)

**Note: needs verification.** The exact mechanism by which the Simulator registers itself as an OpenXR API layer (manifest filename, environment variable, or library name — analogous to `XR_API_LAYER_*` layer discovery in the standard OpenXR loader) was not found in the documentation fetched for this chapter and should be confirmed directly against the native "Getting Started" installation steps before being described in more technical detail elsewhere in this book. Supported host operating systems for running the Simulator were likewise not stated explicitly in the material reviewed.

---

## 13. How the Suite Composes

Pulling the preceding sections together, the dependency shape of the suite is:

- **Horizon OS** (§2) and the native Android/OpenXR loader-and-manifest contract (§3) are the platform floor everything else in this chapter stands on — not a package with a version number, but the substrate every SDK below ultimately targets.
- **Core SDK** is the shared foundation for the Unity/Unreal side of the suite. Every Unity/Unreal package in this chapter except Interaction SDK Essentials depends on it, because Core SDK's `OVRPlugin` is what actually opens the OpenXR session and exposes Meta's extension set to the rest of the suite.
- **Interaction SDK Essentials** is the one deliberate exception on the Unity/Unreal side — a Core-SDK-independent package meant for teams that want Meta's interaction patterns without pulling in the rest of the Meta-specific stack, typically paired with Unity's own `XR Hands` package instead.
- **Meta Spatial SDK** (§6) is a parallel track, not a dependent of Core SDK: it is Meta's Kotlin-native alternative to the Unity/Unreal-plus-Core-SDK path, built directly against Android/Gradle tooling and — plausibly, though not confirmed — against the same native OpenXR loader documented in §3, rather than through `OVRPlugin`.
- **Audio SDK** and **Haptics SDK** are, relative to the rest of the suite, comparatively self-contained: Audio never touches OpenXR at all (§7), and Haptics touches only one narrow extension pair (`XR_FB_haptic_pcm`, `XR_FB_touch_controller_pro`, §8).
- **Platform SDK** is orthogonal to the XR-rendering path entirely — it is a network/identity service that happens to ship alongside the rest of the suite because Quest apps need both.
- **MRUK** is the deepest OpenXR-extension wrapper in the suite, riding on five-plus distinct `XR_FB_*`/`XR_META_*` spatial-entity extensions (§11) — and, as §6 notes, is reimplemented independently (not shared) as a Spatial SDK feature.
- **Simulator/SEB** sits outside the runtime dependency graph altogether — it *replaces* the runtime Core SDK talks to during development, rather than depending on any of the other SDKs.

One release-process detail is visible across the version numbers this chapter cites directly: Platform SDK 77.0 and, later, Core SDK and Interaction SDK Essentials both at 205.0, captured from download pages consulted during this chapter's research. Meta's Unity/Unreal packages appear to share a synchronized version-number cadence across the suite rather than versioning each package independently by its own feature history — a monolithic release-train model, in contrast to the independent per-extension versioning that governs the underlying Khronos OpenXR extensions each package wraps, and in contrast to Meta Spatial SDK's own, separately-numbered pre-1.0 release track on Maven Central (§6). A reader integrating the Unity/Unreal suite should expect to upgrade most of those packages together rather than picking and choosing versions freely; the Simulator's own compatibility caveat against older Core SDK releases (§12) is a direct symptom of this coupling. Horizon OS system-software versioning (§2) is a fourth, entirely independent numbering track from all of the above.

---

## 14. Integrations

- **[Chapter 27: VR, AR, and OpenXR](../part-07a-gpu-apis/ch27-vr-ar.md)** — the OpenXR session, swapchain, action-binding, and extension model this entire suite is built on top of; Monado is the open-runtime counterpart to Horizon OS's proprietary runtime, not to any SDK in this chapter. Its generic OpenXR session-lifecycle material is the right citation for the frame-loop mechanics §3's native path does not itself document.
- **[Chapter 166: Android XR — Jetpack XR SDK, OpenXR Platform Contracts, and Spatial Extensions](ch166-android-xr-expanded.md)** — the structurally analogous "vendor application SDK above a standard OpenXR runtime" layering on non-Meta Android XR headsets, and the closest peer to Meta Spatial SDK specifically (§6); §8's `XR_EXT_spatial_plane_tracking` material is the Khronos-standard counterpart to this chapter's `XR_FB_scene`/`XR_META_spatial_entity_semantic_label` chain.
- **[Chapter 87: Android AR: ARCore Architecture, Camera HAL Integration, and the Android XR Platform](ch87-android-ar-arcore.md)** — the phone-side AR SDK; a useful contrast to MRUK and Core SDK's headset-side scene/anchor model.
- **[Chapter 85: Android Compositor: SurfaceFlinger, HardwareBuffer, and the Buffer Pipeline](ch85-android-surfaceflinger.md)** — the compositor and buffer-pipeline foundation Horizon OS is built from as an AOSP fork (§2), underneath everything OVRPlugin and Spatial SDK's Panel rendering do.
- **[Chapter 86: Vulkan on Android: Drivers, ANGLE, and Mobile GPU Performance](ch86-android-vulkan.md)** — the Vulkan rendering backend Core SDK's swapchain and compositor-layer submission ultimately runs on.
- **[Chapter 164: Android Runtime and Native Interop: ART, JNI, and the NDK](ch164-android-runtime-native-interop.md)** and **[Chapter 161: Android Game Development Kit](ch161-android-game-development-kit.md)** — Horizon OS's underlying Android runtime and native-interop model that any native-C++ Quest app (§3's loader/manifest contract, Platform SDK's native init path, the Haptics SDK's C renderer, the Audio SDK's `OVR_Audio.h`) ultimately runs against, and the JNI boundary Meta Spatial SDK's Kotlin code crosses less often than the Unity/Unreal suite does.

---

## 15. References

- [Meta — Oculus All In on OpenXR, Deprecates Proprietary APIs](https://developers.meta.com/horizon/blog/oculus-all-in-on-openxr-deprecates-proprietary-apis/)
- [Meta — Platforms overview](https://developers.meta.com/horizon/discover/platforms/)
- [Meta — Android apps features overview](https://developers.meta.com/horizon/documentation/android-apps/features-overview/)
- [TechCrunch — Meta opens Quest OS to third-party headset makers](https://techcrunch.com/2024/04/22/meta-opens-quest-os-to-third-party-headset-makers-taps-lenovo-and-xbox-as-partners/)
- [UploadVR — Meta pauses third-party Horizon OS headsets](https://www.uploadvr.com/meta-pauses-third-party-horizon-os-headsets/)
- [Meta — Panels](https://developers.meta.com/horizon/design/panels/)
- [Meta — Distribution options](https://developers.meta.com/horizon/policy/distribution-options/)
- [Meta — Introducing App Lab](https://developers.meta.com/horizon/blog/introducing-app-lab-a-new-way-to-distribute-oculus-quest-apps/)
- [Meta — Release notes](https://developers.meta.com/horizon/release-notes/)
- [Meta Quest Help — release notes](https://www.meta.com/help/quest/172903867975450/)
- [Meta — Passthrough Camera](https://developers.meta.com/horizon/documentation/android-apps/passthrough-camera/)
- [Meta — OpenXR Support for Meta Quest Headsets](https://developers.meta.com/horizon/documentation/native/android/mobile-openxr/)
- [Meta — OpenXR Mobile SDK (intro)](https://developers.meta.com/horizon/documentation/native/android/mobile-intro/)
- [GitHub — meta-quest/Meta-OpenXR-SDK](https://github.com/meta-quest/Meta-OpenXR-SDK)
- [Meta — Implement Passthrough (native)](https://developers.meta.com/horizon/documentation/native/android/mobile-passthrough/)
- [Meta — Core Concepts (native OpenXR)](https://developers.meta.com/horizon/documentation/native/android/mobile-openxr-core-concepts/)
- [Meta — Meta XR Core SDK Overview (Unity)](https://developers.meta.com/horizon/documentation/unity/unity-core-sdk/)
- [Meta — OVRCameraRig documentation](https://developers.meta.com/horizon/documentation/unity/unity-ovrcamerarig/)
- [Meta — Unity XR Plugin Management](https://developers.meta.com/horizon/documentation/unity/unity-xr-plugin/)
- [Meta — Interaction SDK + Unity XR getting started](https://developers.meta.com/horizon/documentation/unity/unity-isdk-getting-started-unityxr/)
- [Unity Asset Store — Meta XR All-In-One SDK](https://assetstore.unity.com/packages/tools/integration/meta-xr-all-in-one-sdk-269657)
- [Meta XR Interaction SDK Essentials (UPM)](https://developers.meta.com/horizon/downloads/package/meta-xr-interaction-sdk/)
- [Meta XR Interaction SDK (UPM, full)](https://developers.meta.com/horizon/downloads/package/meta-xr-interaction-sdk-ovr-integration/)
- [Meta — Interaction SDK overview](https://developers.meta.com/horizon/documentation/unity/unity-isdk-interaction-sdk-overview/)
- [Meta Spatial SDK overview](https://developers.meta.com/horizon/documentation/spatial-sdk/spatial-sdk-explainer/)
- [Android Developers — Jetpack XR SDK](https://developer.android.com/develop/xr/jetpack-xr-sdk)
- [UploadVR — Meta Spatial SDK](https://www.uploadvr.com/meta-spatial-sdk/)
- [Meta Spatial SDK — ECS](https://developers.meta.com/horizon/documentation/spatial-sdk/spatial-sdk-ecs/)
- [Meta Spatial SDK — template walkthrough](https://developers.meta.com/horizon/documentation/spatial-sdk/spatial-sdk-template-walkthrough/)
- [Meta Spatial SDK — MRUK](https://developers.meta.com/horizon/documentation/spatial-sdk/spatial-sdk-mruk/)
- [Maven Central — com.meta.spatial:meta-spatial-sdk](https://central.sonatype.com/artifact/com.meta.spatial/meta-spatial-sdk)
- [GitHub — meta-quest/Meta-Spatial-SDK-Samples](https://github.com/meta-quest/Meta-Spatial-SDK-Samples)
- [GitHub — meta-quest/Meta-Spatial-SDK-Templates](https://github.com/meta-quest/Meta-Spatial-SDK-Templates)
- [Meta XR Audio SDK Features](https://developers.meta.com/horizon/documentation/unity/meta-xr-audio-sdk-features/)
- [Meta XR Audio SDK for Unreal — Ambisonics](https://developers.meta.com/horizon/documentation/unreal/meta-xr-audio-sdk-unreal-ambisonic/)
- [Meta — Audio SDK requirements and setup (Unity)](https://developers.meta.com/horizon/documentation/unity/meta-xr-audio-sdk-unity-req-setup/)
- [Meta XR Audio SDK — FMOD/Unreal example](https://developers.meta.com/horizon/documentation/unreal/meta-xr-audio-sdk-fmod-unreal-example/)
- [`OVR_Audio.h` (Unreal third-party mirror)](https://github.com/a3pelawi/UnrealEngine/blob/master/Engine/Source/ThirdParty/Oculus/LibOVRAudio/LibOVRAudio/include/OVR_Audio.h)
- [`facebook/meta-haptics-sdk` (GitHub, MIT)](https://github.com/facebook/meta-haptics-sdk)
- [Meta — Haptics SDK integration guide (Unity)](https://developers.meta.com/horizon/documentation/unity/unity-haptics-sdk-integrate/)
- [Meta — Haptics SDK overview](https://developers.meta.com/horizon/documentation/unity/unity-haptics-sdk/)
- [Meta — `UMetaXRHapticsPlayerComponent` reference (Unreal)](https://developers.meta.com/horizon/reference/unreal-haptics/v201/class_u_meta_x_r_haptics_player_component/)
- [Meta — Haptic Feedback (native, `XR_FB_haptic_pcm`)](https://developers.meta.com/horizon/documentation/native/android/mobile-openxr-haptic/)
- [Meta — Platform SDK entitlement check (native)](https://developers.meta.com/horizon/documentation/native/ps-entitlement-check/)
- [Meta XR Platform SDK (UPM) v77.0](https://developers.meta.com/horizon/downloads/package/meta-xr-platform-sdk/77.0/)
- [Meta — Set up Platform SDK entitlements (Unity)](https://developers.meta.com/horizon/documentation/unity/unity-platform-entitlements/)
- [Meta — Voice SDK overview](https://developers.meta.com/horizon/documentation/unity/voice-sdk-overview/)
- [Meta — Voice SDK tutorials](https://developers.meta.com/horizon/documentation/unity/voice-sdk-tutorials-1/)
- [Meta — `AppDictationExperience` reference (v67)](https://developers.meta.com/horizon/reference/voice/v67/class_oculus_voice_dictation_app_dictation_experience/)
- [Meta — Presence Platform Voice SDK and Passthrough announcement](https://developers.meta.com/horizon/blog/presence-platform-voice-sdk-and-passthrough-now-available/)
- [Meta — Migrating from OVRSceneManager to MRUK](https://developers.meta.com/horizon/documentation/unity/unity-scene-migrate-mruk/)
- [`MRUKRoom` reference](https://developers.meta.com/horizon/reference/mruk/v201/class_meta_x_r_m_r_utility_kit_m_r_u_k_room/)
- [`MRUKAnchor` reference](https://developers.meta.com/horizon/reference/mruk/v77/class_meta_x_r_m_r_utility_kit_m_r_u_k_anchor/)
- [Meta — Unreal Scene supported semantic labels](https://developers.meta.com/horizon/documentation/unreal/unreal-scene-supported-semantic-labels/)
- [Meta — OpenXR Scene Overview (native)](https://developers.meta.com/horizon/documentation/native/android/openxr-scene-overview/)
- [Khronos — `XR_FB_spatial_entity`](https://registry.khronos.org/OpenXR/specs/1.0/man/html/XR_FB_spatial_entity.html)
- [Meta XR Simulator overview](https://developers.meta.com/horizon/documentation/unity/xrsim-intro/)
- [Meta XR Simulator getting started (native)](https://developers.meta.com/horizon/documentation/native/xrsim-getting-started/)
- [Meta — Simulate a Mixed Reality Environment (SES)](https://developers.meta.com/horizon/documentation/unity/xrsim-ses/)
- [Meta — Build Your Own SES (Synthetic Environment Builder, Unreal)](https://developers.meta.com/horizon/documentation/unreal/xrsim-synthetic-environment-builder/)
- [Meta XR Simulator - Synthetic Environment Builder (download)](https://developers.meta.com/horizon/downloads/package/meta-xr-simulator-synthetic-environment-builder/)
- [Meta — Built-in Test Rooms](https://developers.meta.com/horizon/documentation/unity/xrsim-heroscenes/)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
