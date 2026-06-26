# Chapter 161 — Android Game Development Kit (AGDK): Native Game Architecture, Input, Audio, and Frame Pacing

> **Audiences:** Graphics application developers building native Android games and high-performance apps; systems developers who want to understand how Android's C/C++ game stack maps onto the graphics, audio, and input subsystems; browser and tool engineers who need to understand the NDK lifecycle model that underpins Chrome's Android renderer and Android Studio's profiling toolchain.

---

## Table of Contents

1. [What AGDK Is](#1-what-agdk-is)
2. [NativeActivity: The Foundation and Its Limits](#2-nativeactivity-the-foundation-and-its-limits)
3. [GameActivity: The Modern Native App Model](#3-gameactivity-the-modern-native-app-model)
4. [Input Handling with GameActivity](#4-input-handling-with-gameactivity)
5. [Paddleboat: Game Controller Library](#5-paddleboat-game-controller-library)
6. [Android Frame Pacing: Swappy](#6-android-frame-pacing-swappy)
7. [ANativeWindow: The Surface Handle](#7-anativewindow-the-surface-handle)
8. [Oboe: Low-Latency Audio](#8-oboe-low-latency-audio)
9. [Android Performance Tuner](#9-android-performance-tuner)
10. [Memory Advice API](#10-memory-advice-api)
11. [Android GPU Inspector](#11-android-gpu-inspector)
12. [Integrations](#12-integrations)

---

## 1. What AGDK Is

The **Android Game Development Kit** (AGDK) is a collection of C/C++ libraries, tools, and Jetpack packages that Google introduced in 2021 to address the fragmented, low-level nature of Android native game development. Before AGDK, developers writing high-performance games in C++ had to stitch together: the legacy `NativeActivity` API (broken text input, coarse input batching), the Android audio stack (OpenSL ES, complex latency tuning), frame timing (bespoke Choreographer hacks), and controller input (per-OEM fragmentation). AGDK provides production-quality solutions to each of these.

### Component overview

| Library | Gradle artifact | Purpose |
|---|---|---|
| `game-activity` | `androidx.games:games-activity` | Replaces `NativeActivity`; lifecycle + input |
| `game-text-input` | bundled with `game-activity` | Working IME text input from C/C++ |
| `games-controller` / Paddleboat | `androidx.games:games-controller` | Gamepad enumeration, mapping, vibration |
| `games-frame-pacing` / Swappy | `androidx.games:games-frame-pacing` | Vsync-aware frame presentation (GL + Vulkan) |
| `oboe` | `com.google.oboe:oboe` | Low-latency audio (OpenSL ES / AAudio abstraction) |
| `games-performance` | `androidx.games:games-performance` | Frame-timing reporting to Google Play backend |
| `games-memory-advice` | `androidx.games:games-memory-advice` | Memory pressure prediction and callbacks |

All libraries are distributed as **Prefab AAR** packages — they integrate with CMake via `find_package()` after Gradle unpacks the headers and pre-built `.a`/`.so` into the build tree. [Source: AGDK overview](https://developer.android.com/games/agdk)

### Gradle integration

```groovy
// build.gradle (app module)
android {
    defaultConfig {
        externalNativeBuild { cmake { arguments "-DANDROID_STL=c++_shared" } }
    }
}

dependencies {
    implementation 'androidx.games:games-activity:3.0.0'
    implementation 'androidx.games:games-controller:2.0.2'
    implementation 'androidx.games:games-frame-pacing:2.1.0'
    implementation 'com.google.oboe:oboe:1.9.0'
    implementation 'androidx.games:games-memory-advice:1.0.0'
}
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.22)
project(mygame)

find_package(game-activity  REQUIRED CONFIG)
find_package(games-controller REQUIRED CONFIG)
find_package(games-frame-pacing REQUIRED CONFIG)
find_package(oboe REQUIRED CONFIG)

add_library(mygame SHARED src/main.cpp)
target_link_libraries(mygame
    game-activity::game-activity_static
    games-controller::paddleboat_static
    games-frame-pacing::swappy_static
    oboe::oboe
    android log vulkan)
```

### Distribution model

AGDK libraries are **not bundled with the OS**; they ship as part of the app's APK/AAB. This lets Google update the libraries independently of Android OS releases, and lets apps always target the latest version. Pre-built binaries are provided for `arm64-v8a`, `armeabi-v7a`, `x86`, and `x86_64`. The `_static` variants link the library into the `.so`; the shared variants reduce APK size when multiple `.so` files use the same library.

---

## 2. NativeActivity: The Foundation and Its Limits

Understanding `NativeActivity` is prerequisite to understanding why `GameActivity` was needed. `NativeActivity` remains supported and is still used in simpler apps, but its limitations are load-bearing motivations for everything in §3 onwards.

### The android_native_app_glue threading model

`android.app.NativeActivity` is a standard Java `Activity` subclass. On startup it loads the native `.so` specified in `AndroidManifest.xml` and resolves the C entry point `ANativeActivity_onCreate` by name (overridable via the `android.app.func_name` metadata key).

The NDK-provided `android_native_app_glue` library (`sources/android/native_app_glue/`) wraps the raw `ANativeActivityCallbacks` mechanism with a more ergonomic execution model: it spawns a **dedicated background pthread** for `android_main()`, separating the app's C loop from the Java main thread. Android lifecycle callbacks (from the Java main thread) are serialised as `APP_CMD_*` integers into a pipe; the background thread reads them via `ALooper_pollOnce()`. This ensures framework callbacks never stall long enough to trigger an ANR (Application Not Responding watchdog, typically 5 seconds).

```c
// sources/android/native_app_glue/android_native_app_glue.h (simplified)
struct android_app {
    void*             userData;
    void (*onAppCmd)(struct android_app*, int32_t cmd);   // APP_CMD_* handler
    int32_t (*onInputEvent)(struct android_app*, AInputEvent*); // 1 = consumed
    ANativeActivity*  activity;     // Java NativeActivity wrapper
    AConfiguration*   config;
    void*             savedState;
    size_t            savedStateSize;
    ALooper*          looper;       // LOOPER_ID_MAIN=1, LOOPER_ID_INPUT=2, USER=3+
    AInputQueue*      inputQueue;
    ANativeWindow*    window;       // non-NULL when surface is ready (see §7)
    ARect             contentRect;
    int               activityState;
    int               destroyRequested;
};

extern void android_main(struct android_app* app);  // application entry point
```

The `ALooper` is the multiplexed event source: `ALooper_pollOnce(timeoutMs, &fd, &events, &data)` blocks until a command or input event is ready, then calls the associated `android_poll_source::process` callback.

### ANativeActivityCallbacks: all 16 lifecycle points

All callbacks fire on the **Java main thread** and all default to NULL. The glue library fills in its own implementations that write `APP_CMD_*` integers to the pipe:

```c
struct ANativeActivityCallbacks {
    void  (*onStart)(ANativeActivity*);
    void  (*onResume)(ANativeActivity*);
    void* (*onSaveInstanceState)(ANativeActivity*, size_t* outSize);
    void  (*onPause)(ANativeActivity*);
    void  (*onStop)(ANativeActivity*);
    void  (*onDestroy)(ANativeActivity*);
    void  (*onWindowFocusChanged)(ANativeActivity*, int hasFocus);
    void  (*onNativeWindowCreated)(ANativeActivity*, ANativeWindow*);
    void  (*onNativeWindowResized)(ANativeActivity*, ANativeWindow*);
    void  (*onNativeWindowRedrawNeeded)(ANativeActivity*, ANativeWindow*);
    void  (*onNativeWindowDestroyed)(ANativeActivity*, ANativeWindow*);
    void  (*onInputQueueCreated)(ANativeActivity*, AInputQueue*);
    void  (*onInputQueueDestroyed)(ANativeActivity*, AInputQueue*);
    void  (*onContentRectChanged)(ANativeActivity*, const ARect*);
    void  (*onConfigurationChanged)(ANativeActivity*);
    void  (*onLowMemory)(ANativeActivity*);
};
```

### NativeActivity's three hard limitations

**1. Broken text input.** `ANativeActivity_showSoftInput()` can raise the IME keyboard, but there is no C callback to receive the resulting text — it is consumed by the Java `InputConnection` layer with no NDK bridge. Games requiring username entry or chat must invoke JNI to read Java `EditText` contents, defeating the zero-JNI goal.

**2. No `SurfaceView` control.** `NativeActivity` owns the entire window surface; it is impossible to compose native rendering alongside a standard Android `View` hierarchy (e.g., a Jetpack Compose UI overlay or an `AdView` for ads) without significant workarounds.

**3. Coarse input event batching.** Touch events arrive via `AInputQueue` one at a time per `ALooper_pollOnce` iteration. High-frequency stylus or multi-touch events are batched by the system and historical samples are not exposed through the C API — the Java `MotionEvent.getHistoricalX()` path has no NDK equivalent in `NativeActivity`.

---

## 3. GameActivity: The Modern Native App Model

`GameActivity` resolves all three `NativeActivity` limitations while retaining the same threading model and `ANativeWindow` lifecycle.

### Kotlin/Java subclass requirement

Unlike `NativeActivity` (which is final and usable directly), `GameActivity` must be subclassed:

```kotlin
// MainActivity.kt
import com.google.androidgamesdk.GameActivity

class MainActivity : GameActivity() {
    // Override to customise window flags, edge-to-edge, etc.
    override fun onWindowFocusChanged(hasFocus: Boolean) {
        super.onWindowFocusChanged(hasFocus)
        if (hasFocus) {
            window.decorView.systemUiVisibility =
                android.view.View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY or
                android.view.View.SYSTEM_UI_FLAG_FULLSCREEN or
                android.view.View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
        }
    }
}
```

The native entry-point symbol changes from `ANativeActivity_onCreate` to `Java_com_google_androidgamesdk_GameActivity_initializeNativeCode`. The linker flag must reflect this:

```cmake
target_link_options(mygame PRIVATE
    -u Java_com_google_androidgamesdk_GameActivity_initializeNativeCode)
```

Replace the classic glue source with the AGDK version:
```cmake
# Include AGDK's native_app_glue.c (not NDK's android_native_app_glue.c)
target_sources(mygame PRIVATE
    ${GAME_ACTIVITY_INCLUDE}/game-activity/native_app_glue/android_native_app_glue.c)
```

### The AGDK android_app struct

```c
struct android_app {
    GameActivity* activity;       // GameActivity* replaces ANativeActivity*
    ANativeWindow* window;        // same as classic — non-NULL when surface ready
    ALooper*       looper;
    AConfiguration* config;
    void*           userData;
    void (*onAppCmd)(struct android_app*, int32_t);

    // Input arrays — replaces AInputQueue callback entirely:
    GameActivityMotionEvent motionEvents[NATIVE_APP_GLUE_MAX_NUM_MOTION_EVENTS];
    uint64_t                motionEventsCount;
    GameActivityMotionEvent motionEventsFilter;   // bitmask of allowed action types
    GameActivityKeyEvent    keyDownEvents[NATIVE_APP_GLUE_MAX_NUM_KEY_EVENTS];
    uint64_t                keyDownEventsCount;
    GameActivityKeyEvent    keyUpEvents[NATIVE_APP_GLUE_MAX_NUM_KEY_EVENTS];
    uint64_t                keyUpEventsCount;
    int                     textInputState;        // 1 = pending text from IME

    void*  savedState;
    size_t savedStateSize;
    int    activityState;
    int    destroyRequested;
};
```

`GameActivity` struct replaces `ANativeActivity` and renames the mis-named `clazz` field:

```c
struct GameActivity {
    JavaVM*                       vm;
    JNIEnv*                       env;               // main-thread JNI only
    jobject                       javaGameActivity;  // NOT 'clazz' — corrected name
    const char*                   internalDataPath;
    const char*                   externalDataPath;
    const char*                   obbPath;
    int32_t                       sdkVersion;
    void*                         instance;
    struct GameActivityCallbacks* callbacks;
    AAssetManager*                assetManager;
};
```

[Source: GameActivity struct](https://developer.android.com/reference/games/game-activity/struct/game-activity), [android_app struct](https://developer.android.com/reference/games/game-activity/struct/android-app)

### Lifecycle event ordering: surface vs. resume

The critical ordering rule for Vulkan — identical to `NativeActivity` — is that **`APP_CMD_INIT_WINDOW` is the correct place to create the swapchain, not `APP_CMD_RESUME`**. The surface is created after `onResume` fires, and it can be destroyed while the activity remains in resumed state (e.g., when a system overlay covers the window). `android_app::window` is the authoritative signal:

| `APP_CMD_*` | `window` state | Vulkan action |
|---|---|---|
| `APP_CMD_START` | may be NULL | — |
| `APP_CMD_RESUME` | may be NULL | — |
| `APP_CMD_INIT_WINDOW` | **non-NULL** | Create `VkSurfaceKHR` + swapchain |
| `APP_CMD_GAINED_FOCUS` | non-NULL | Begin render loop |
| `APP_CMD_LOST_FOCUS` | non-NULL | Pause render loop |
| `APP_CMD_TERM_WINDOW` | about to be NULL | **Destroy swapchain + `VkSurfaceKHR`** |
| `APP_CMD_PAUSE` | NULL or non-NULL | — |
| `APP_CMD_STOP` | NULL | — |
| `APP_CMD_SAVE_STATE` | NULL | Serialise game state |
| `APP_CMD_DESTROY` | NULL | Destroy all Vulkan resources |

```c
void handle_cmd(struct android_app* app, int32_t cmd) {
    switch (cmd) {
    case APP_CMD_INIT_WINDOW:
        if (app->window) engine_init_vulkan(app->window);
        break;
    case APP_CMD_TERM_WINDOW:
        engine_destroy_vulkan();
        break;
    case APP_CMD_GAINED_FOCUS:
        g_rendering = true;
        break;
    case APP_CMD_LOST_FOCUS:
        g_rendering = false;
        break;
    }
}
```

---

## 4. Input Handling with GameActivity

### Double-buffered input model

`NativeActivity` processes input one event at a time via `ALooper_pollOnce`. `GameActivity` replaces this with a **double-buffered array** model: the framework fills one buffer while the app reads the other, then the app calls `android_app_swap_input_buffers()` to atomically exchange them. This exposes all historical samples and eliminates the ANR risk from a backed-up `AInputQueue`.

```c
// Per frame, before rendering:
android_app_swap_input_buffers(app);  // swap front/back

// Motion events (touch, stylus, mouse, joystick):
for (uint64_t i = 0; i < app->motionEventsCount; i++) {
    GameActivityMotionEvent* e = &app->motionEvents[i];
    int32_t action  = e->action & AMOTION_EVENT_ACTION_MASK;
    int32_t ptrIdx  = (e->action & AMOTION_EVENT_ACTION_POINTER_INDEX_MASK)
                      >> AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT;

    float x = GameActivityPointerAxes_getAxisValue(
                  &e->pointers[ptrIdx], AMOTION_EVENT_AXIS_X);
    float y = GameActivityPointerAxes_getAxisValue(
                  &e->pointers[ptrIdx], AMOTION_EVENT_AXIS_Y);

    // Historical samples — exposed, unlike NativeActivity:
    for (int32_t h = 0; h < e->historicalCount; h++) {
        float hx = GameActivityPointerAxes_getAxisValue(
                       &e->historicalPointers[h * e->pointerCount + ptrIdx],
                       AMOTION_EVENT_AXIS_X);
    }
}
android_app_clear_motion_events(app);

// Key events:
for (uint64_t i = 0; i < app->keyDownEventsCount; i++) {
    GameActivityKeyEvent* e = &app->keyDownEvents[i];
    // e->keyCode (AKEYCODE_*), e->action, e->metaState
}
android_app_clear_key_events(app);
```

### Enabling axes and event filters

By default `GameActivity` only delivers finger touch events. To receive controller axes (joystick, trigger) or stylus data, the axis must be explicitly enabled:

```c
// Enable joystick and trigger axes for Paddleboat/controller input:
android_app_set_motion_event_filter(app, NULL);  // NULL = receive all devices

// Or for fine-grained control, check Paddleboat's active axis mask:
uint64_t axisMask = Paddleboat_getActiveAxisMask();
for (int axis = 0; axis < GAME_ACTIVITY_POINTER_INFO_AXIS_COUNT; axis++) {
    if (axisMask & (1ULL << axis))
        GameActivityPointerAxes_enableAxis(axis);
}
```

### GameTextInput: working IME integration

`game-text-input` (bundled with `game-activity`) provides the IME bridge that `NativeActivity` lacks. The app configures the editor and reads results in C:

```c
// Configure IME (call once, or when editor changes):
GameActivity_setImeEditorInfo(app->activity,
    IME_ACTION_DONE,          // action button label
    IME_FLAG_NO_FULLSCREEN,   // flags
    0);                       // inputType bitmask

GameActivity_showSoftInput(app->activity, 0);

// Check and read pending text (per-frame):
if (app->textInputState) {
    GameTextInputState state;
    GameActivity_getTextInputState(app->activity,
        [](void* ctx, const GameTextInputState* s) {
            // s->text_UTF8, s->text_length, s->selection_start/end
            memcpy(((MyApp*)ctx)->inputBuffer, s->text_UTF8, s->text_length);
        }, myApp);
}
```

[Source: GameActivity input guide](https://developer.android.com/games/agdk/game-activity/migrate-native-activity)

---

## 5. Paddleboat: Game Controller Library

`Paddleboat` (`androidx.games:games-controller`) solves Android's game controller fragmentation problem: the NDK's `AInputQueue` delivers raw `AInputEvent` objects with keycodes and axis values, but axis mapping, button labelling, vibration support, and multi-controller enumeration vary wildly across OEM controllers, generic HID gamepads, and first-party hardware (PS5 DualSense, Xbox, Switch Pro).

### Initialization and per-frame update

```c
#include <paddleboat/paddleboat.h>

// Init once (pass JNI env and Activity jobject):
Paddleboat_ErrorCode err = Paddleboat_init(env, app->activity->javaGameActivity);

// Per frame — processes pending controller connect/disconnect events:
Paddleboat_update(env);
```

### Forwarding GameActivity events

Paddleboat acts as a filter on top of the GameActivity input arrays:

```c
for (uint64_t i = 0; i < app->motionEventsCount; i++) {
    // Returns 1 if Paddleboat consumed the event (controller input)
    if (!Paddleboat_processGameActivityMotionInputEvent(
            &app->motionEvents[i], sizeof(GameActivityMotionEvent))) {
        // Not a controller — handle as touch
        handle_touch(&app->motionEvents[i]);
    }
}
for (uint64_t i = 0; i < app->keyDownEventsCount; i++) {
    if (!Paddleboat_processGameActivityKeyInputEvent(
            &app->keyDownEvents[i], sizeof(GameActivityKeyEvent))) {
        handle_key(&app->keyDownEvents[i]);
    }
}
```

### Reading controller state

```c
typedef struct Paddleboat_Controller_Data {
    uint64_t  timestamp;         // microseconds since epoch
    uint32_t  buttonsDown;       // PADDLEBOAT_BUTTON_* bitmask
    struct {
        float stickX, stickY;   // -1.0 to 1.0
    } leftStick, rightStick;
    float     triggerL1, triggerL2, triggerR1, triggerR2;  // 0.0 to 1.0
    struct {
        float pointerX, pointerY;
    } virtualPointer;
} Paddleboat_Controller_Data;

Paddleboat_Controller_Data ctrl;
if (Paddleboat_getControllerData(0 /*index*/, &ctrl) == PADDLEBOAT_NO_ERROR) {
    if (ctrl.buttonsDown & PADDLEBOAT_BUTTON_A) jump();
    move_character(ctrl.leftStick.stickX, ctrl.leftStick.stickY);
    aim_camera(ctrl.rightStick.stickX, ctrl.rightStick.stickY);
}
```

### Vibration and lights

```c
Paddleboat_Vibration_Data vib = {
    .durationLeft  = 200,   // ms
    .durationRight = 200,
    .intensityLeft = 0.8f,
    .intensityRight = 0.4f,
};
Paddleboat_setControllerVibrationData(0, &vib, env);

// Controller info (name, vendorId, productId, capabilities):
Paddleboat_Controller_Info info;
Paddleboat_getControllerInfo(0, &info);
// info.controllerFlags & PADDLEBOAT_CONTROLLER_FLAG_VIBRATION
// info.controllerFlags & PADDLEBOAT_CONTROLLER_FLAG_TOUCHPAD
```

### Controller connect/disconnect callbacks

```c
Paddleboat_setControllerStatusCallback(
    [](int32_t idx, Paddleboat_ControllerStatus status, void* ctx) {
        if (status == PADDLEBOAT_CONTROLLER_JUST_CONNECTED)
            on_controller_connected(idx);
        else if (status == PADDLEBOAT_CONTROLLER_JUST_DISCONNECTED)
            on_controller_disconnected(idx);
    }, myApp);
```

[Source: Game Controller Library](https://developer.android.com/games/sdk/game-controller)

---

## 6. Android Frame Pacing: Swappy

**Swappy** (`games-frame-pacing`) solves two jank patterns that arise when a native app calls `vkQueuePresentKHR` (or `eglSwapBuffers`) without vsync awareness.

### The two jank problems

**Double-present jank**: A short frame (B) completes early before the next vsync boundary. The preceding frame (A) may be presented at the same vsync as B, causing A to display for only one vsync interval instead of two, then B to also display for one — creating a stutter cadence of 16 ms / 16 ms instead of the intended 33 ms / 33 ms (at 60 Hz targeting 30 fps).

**Buffer stuffing**: A long frame causes the `BufferQueue` between the app and SurfaceFlinger to fill. The app cannot detect this backpressure, so it keeps submitting work. The `BufferQueue` depth grows, and each frame displayed is older than it should be — input latency increases by one full display refresh period per queued buffer.

Swappy addresses both:
- **Short frames**: injects `VkPresentTimesInfoGOOGLE` (`VK_GOOGLE_display_timing`) into the `VkPresentInfoKHR.pNext` chain to schedule presentation at the correct future vsync, preventing early display.
- **Long frames / buffer stuffing**: inserts `VkFence` waits on the CPU when the `BufferQueue` is full, preventing buffer accumulation.
- **Multi-refresh-rate displays**: on 60/90/120 Hz panels, Swappy auto-selects the closest achievable refresh rate. A game running at 45 fps targets 90 Hz (every 2 vsyncs) rather than dropping to 30 fps on 60 Hz.

### Vsync calibration via Choreographer

Swappy uses the NDK `Choreographer` API (API 24+) or Java `Choreographer` (API 16+) to obtain per-device vsync calibration: the `AppVsyncOffsetNanos` (how far ahead of vsync the app should start work) and `PresentationDeadlineNanos` (last moment to submit a frame for the current vsync). These are queried via JNI from `WindowManager.getDefaultDisplay()` during `SwappyVk_initAndGetRefreshCycleDuration`. [Source: Frame Pacing library](https://developer.android.com/games/sdk/frame-pacing)

### SwappyVk integration sequence

```c
#include <swappy/swappyVk.h>

// Step 1 — before vkCreateDevice: query required device extensions
// Swappy adds VK_GOOGLE_display_timing if the driver supports it
uint32_t reqCount = 0;
SwappyVk_determineDeviceExtensions(physicalDevice,
    availExtCount, pAvailExts, &reqCount, NULL);
const char** ppReqExts = malloc(reqCount * sizeof(char*));
SwappyVk_determineDeviceExtensions(physicalDevice,
    availExtCount, pAvailExts, &reqCount, ppReqExts);
// Add ppReqExts to VkDeviceCreateInfo::ppEnabledExtensionNames

// Step 2 — after vkCreateSwapchainKHR:
uint64_t refreshDurationNs;
SwappyVk_initAndGetRefreshCycleDuration(
    env, app->activity->javaGameActivity,
    physicalDevice, device, swapchain, &refreshDurationNs);

// Step 3 — bind ANativeWindow for display-timing queries
SwappyVk_setWindow(device, swapchain, app->window);

// Step 4 — set target swap interval (nanoseconds)
// 16_666_666 = 60 Hz, 11_111_111 = 90 Hz, 8_333_333 = 120 Hz
SwappyVk_setSwapIntervalNS(device, swapchain, refreshDurationNs);

// Step 5 — replace vkQueuePresentKHR in the render loop
VkResult result = SwappyVk_queuePresent(queue, &presentInfo);
// Swappy injects VkPresentTimesInfoGOOGLE into presentInfo.pNext
// and calls vkQueuePresentKHR on the app's behalf

// Cleanup — before vkDestroySwapchainKHR:
SwappyVk_destroySwapchain(device, swapchain);
```

[Source: SwappyVk API reference](https://developer.android.com/games/sdk/reference/frame-pacing/group/swappy-vk)

### SwappyGL for OpenGL ES apps

For apps using GLES/EGL, the integration is identical in structure but uses `SwappyGL_*` functions:

```c
#include <swappy/swappyGL.h>

SwappyGL_init(env, app->activity->javaGameActivity);
SwappyGL_setWindow(app->window);
// Replace eglSwapBuffers:
SwappyGL_swap(display, surface);
SwappyGL_destroy();
```

### Pipeline modes

Swappy supports three pipeline depth modes:
- **Pipeline mode** (default): CPU frame N+1 overlaps GPU frame N across vsync boundaries — maximises throughput, adds ~16 ms input-to-display latency at 60 Hz.
- **Non-pipeline mode**: CPU and GPU both complete within one swap interval — minimises latency at the cost of GPU idle time.
- **Auto mode**: Swappy measures per-frame CPU and GPU durations and dynamically switches between pipeline depths and swap intervals to hit the best achievable frame rate on the device.

```c
SwappyVk_setAutoPipelineMode(true);  // enable auto-mode
// Or manually:
SwappyVk_enableStats(true);
SwappyVk_recordFrameStart(device, swapchain);
// ... render ...
SwappyVk_queuePresent(queue, &presentInfo);
```

---

## 7. ANativeWindow: The Surface Handle

`ANativeWindow*` is the C handle through which all native rendering — Vulkan, EGL, or software — accesses the Android display surface. It is the producer end of a `BufferQueue`; SurfaceFlinger is the consumer. The same handle is used by both `NativeActivity` and `GameActivity` via `android_app::window`.

### Internal architecture

Internally, `ANativeWindow*` points to a C++ `Surface` object in `frameworks/native`. `Surface` implements the `ANativeWindow` C vtable — casting a `Surface*` to `ANativeWindow*` is the bridge. The `Surface` holds a `BufferQueueProducer` that talks to SurfaceFlinger via Binder IPC (`IGraphicBufferProducer`). From the app's perspective, the buffer flow is:

```
App (NDK) → ANativeWindow → BufferQueue producer
                              ↓ Binder IPC
                           SurfaceFlinger (consumer)
                              ↓
                           HWComposer → DRM atomic commit → Display
```

### Reference counting

Both Vulkan and EGL manage surface lifetime through `acquire`/`release`:

```c
ANativeWindow_acquire(window);   // increment refcount
ANativeWindow_release(window);   // decrement; may free underlying Surface
```

`vkCreateAndroidSurfaceKHR` calls `ANativeWindow_acquire` internally. `vkDestroySurfaceKHR` calls `ANativeWindow_release`. A single `ANativeWindow` cannot be simultaneously bound to both a `VkSurfaceKHR` and an `EGLSurface` — creating the second will fail or produce undefined behaviour.

### Geometry and pixel format

```c
int32_t w = ANativeWindow_getWidth(window);
int32_t h = ANativeWindow_getHeight(window);
int32_t fmt = ANativeWindow_getFormat(window);

// Override buffer geometry (pass 0,0 to use window's natural size):
ANativeWindow_setBuffersGeometry(window, 0, 0,
    WINDOW_FORMAT_RGBA_8888);  // = AHARDWAREBUFFER_FORMAT_R8G8B8A8_UNORM (1)
                               // WINDOW_FORMAT_RGBX_8888 (2)
                               // WINDOW_FORMAT_RGB_565 (4)
```

### Vulkan integration

```c
// Obtain once at APP_CMD_INIT_WINDOW — no JNI needed when using GameActivity:
ANativeWindow* anw = app->window;

VkAndroidSurfaceCreateInfoKHR surfaceInfo = {
    .sType  = VK_STRUCTURE_TYPE_ANDROID_SURFACE_CREATE_INFO_KHR,
    .window = anw,
};
vkCreateAndroidSurfaceKHR(instance, &surfaceInfo, NULL, &vkSurface);
// All subsequent Vulkan calls: pure C, zero JNI per frame
```

When obtained from a `SurfaceHolder` or `SurfaceTexture` in a hybrid Java/C++ app, a single `ANativeWindow_fromSurface(env, jSurface)` call crosses JNI once; the handle is then valid for the render lifetime.

### EGL integration

```c
// EGLNativeWindowType is typedef'd to ANativeWindow* on Android
EGLSurface eglSurface = eglCreateWindowSurface(
    display, config, (EGLNativeWindowType)anw, NULL);
```

### CPU software rendering path

For 2D software rendering, screenshot readback, or video frame stamping:

```c
ANativeWindow_Buffer buf;
ARect dirty = {0, 0, w, h};
if (ANativeWindow_lock(window, &buf, &dirty) == 0) {
    // buf.bits: raw pixel pointer; buf.stride: row stride in pixels
    uint32_t* pixels = (uint32_t*)buf.bits;
    for (int y = 0; y < buf.height; y++)
        memset(pixels + y * buf.stride, 0xFF, buf.width * 4);
    ANativeWindow_unlockAndPost(window);
}
```

[Source: ANativeWindow NDK reference](https://developer.android.com/ndk/reference/group/a-native-window)

---

## 8. Oboe: Low-Latency Audio

Audio latency is one of Android's most criticised limitations. A game requiring tight audio/visual synchronisation — footstep sounds on a frame boundary, fighting-game hit feedback — needs round-trip latency (input → process → output) under 20 ms. Android's original audio stack (OpenSL ES) had latency of 50–200 ms on most devices. AAudio (Android 8.0, API 26) introduced a low-latency MMAP path, but OpenSL ES remains necessary for API < 26 devices. **Oboe** (`com.google.oboe:oboe`) abstracts both, automatically selecting AAudio when available and falling back to OpenSL ES.

[Source: Oboe GitHub](https://github.com/google/oboe)

### Audio stream with callback

The preferred pattern is a **callback-driven** stream: Oboe calls `onAudioReady()` from a high-priority audio thread at each buffer period. The callback must not block, allocate, or call JNI.

```c
#include <oboe/Oboe.h>

class AudioEngine : public oboe::AudioStreamCallback {
    std::shared_ptr<oboe::AudioStream> stream_;
    float phase_ = 0.0f;

public:
    oboe::Result open() {
        oboe::AudioStreamBuilder builder;
        builder.setDirection(oboe::Direction::Output)
               .setPerformanceMode(oboe::PerformanceMode::LowLatency)
               .setSharingMode(oboe::SharingMode::Exclusive)
               .setFormat(oboe::AudioFormat::Float)
               .setChannelCount(oboe::ChannelCount::Stereo)
               .setSampleRate(48000)
               .setCallback(this);
        return builder.openStream(stream_);
    }

    oboe::DataCallbackResult onAudioReady(
            oboe::AudioStream* stream,
            void* audioData, int32_t numFrames) override {
        auto* out = static_cast<float*>(audioData);
        for (int i = 0; i < numFrames; i++) {
            float sample = sinf(phase_) * 0.3f;
            phase_ += 2.0f * M_PI * 440.0f / stream->getSampleRate();
            if (phase_ > 2.0f * M_PI) phase_ -= 2.0f * M_PI;
            out[i * 2 + 0] = sample;  // left
            out[i * 2 + 1] = sample;  // right
        }
        return oboe::DataCallbackResult::Continue;
    }

    void start() { stream_->requestStart(); }
    void stop()  { stream_->requestStop(); }
    void close() { stream_->close(); }
};
```

### Performance mode and sharing mode

| Setting | Options | Effect |
|---|---|---|
| `PerformanceMode` | `LowLatency`, `PowerSaving`, `None` | `LowLatency` requests MMAP/exclusive path; `PowerSaving` uses larger buffers |
| `SharingMode` | `Exclusive`, `Shared` | `Exclusive` gets the MMAP path when available (lowest latency); `Shared` shares the mixer |
| `AudioFormat` | `Float`, `I16`, `I32` | `Float` is preferred for mixing; driver may accept directly without conversion |

`PerformanceMode::LowLatency` + `SharingMode::Exclusive` activates AAudio's **MMAP path** on supported devices — the audio data is written directly to a shared memory region mapped between the app and the audio HAL, bypassing the AudioFlinger mixer. Round-trip latency on Pixel devices: ~5–10 ms.

### Input (microphone) stream

```c
oboe::AudioStreamBuilder builder;
builder.setDirection(oboe::Direction::Input)
       .setPerformanceMode(oboe::PerformanceMode::LowLatency)
       .setSharingMode(oboe::SharingMode::Exclusive)
       .setFormat(oboe::AudioFormat::Float)
       .setChannelCount(1)  // mono microphone
       .setCallback(this);
builder.openStream(stream_);
stream_->requestStart();
```

### Stream restart on disconnect

The `onErrorBeforeClose` / `onErrorAfterClose` callbacks handle headphone disconnect and other stream invalidation events. Oboe provides `AudioStreamErrorCallback` with a default restart implementation:

```c
class AudioEngine : public oboe::AudioStreamCallback,
                    public oboe::AudioStreamErrorCallback {
    void onErrorAfterClose(oboe::AudioStream*, oboe::Result) override {
        // Restart the stream on a non-audio thread
        open(); start();
    }
};
```

---

## 9. Android Performance Tuner

The **Android Performance Tuner** (APT, `androidx.games:games-performance`) reports per-frame timing data to the **Google Play backend**, enabling Google Play Console to show frame pacing histograms and recommend quality level adjustments to live apps. For the developer, it is a lightweight frame-tick instrumentation; for end users, Play can automatically suggest lower graphics settings on struggling devices.

### Architecture

APT uses **protocol buffers** to define an app's quality "fidelity parameters" (resolution, shadow quality, texture level, etc.) and "annotation" (which level / scene type is active). It sends timing data per frame and receives back "recommended fidelity parameters" from Play's device-specific performance database.

```c
#include <tuningfork/tuningfork.h>
#include <tuningfork/protobuf_nano_util.h>

// Initialise (once, early in android_main):
TuningFork_Settings settings = {};
settings.swappy_tracer_fn = &SwappyVk_injectTracer;  // integrate with Swappy
settings.swappy_version = Swappy_version();
TuningFork_init(&settings, env, app->activity->javaGameActivity);

// Per frame — tick the default PACED instrument:
TuningFork_frameTick(TFTICK_PACED_FRAME_TIME);

// Custom annotation (e.g., current level):
MyAnnotation ann = {.level = current_level, .scene = SCENE_OUTDOOR};
TuningFork_setCurrentAnnotation(&ann, sizeof(ann));

// Flush (e.g., on pause):
TuningFork_flush();
TuningFork_destroy();
```

### Instrument IDs

APT tracks separate timing histograms per instrument tag. Built-in tags:

| Tag | Measures |
|---|---|
| `TFTICK_PACED_FRAME_TIME` | Wall time per frame (includes Swappy wait) |
| `TFTICK_CPU_TIME` | CPU-only frame time |
| `TFTICK_GPU_TIME` | GPU frame time (via `VK_GOOGLE_display_timing`) |
| `TFTICK_SWAPPY_WAIT_TIME` | Time Swappy spent waiting for vsync |

Custom instruments allow per-subsystem timing (e.g., physics tick, shadow pass).

[Source: Android Performance Tuner](https://developer.android.com/games/sdk/performance-tuner)

---

## 10. Memory Advice API

Android's `onTrimMemory(TRIM_MEMORY_RUNNING_CRITICAL)` callback fires when the OS is about to kill the app's background processes due to memory pressure — but it fires reactively, too late for the app to gracefully reduce its memory footprint. The **Memory Advice API** (`androidx.games:games-memory-advice`) provides **predictive** memory pressure warnings that arrive before the critical threshold.

### Architecture

The library samples Android's `/proc/meminfo`, `ActivityManager.MemoryInfo`, and `Debug.MemoryInfo` on a background thread, builds a simple statistical model of memory pressure trajectory, and issues callbacks when it predicts the threshold will be crossed.

```c
#include <memory_advice/memory_advice.h>

// Init once:
MemoryAdvice_init(env, app->activity->javaGameActivity);

// Register a callback (fires on the sampling thread):
MemoryAdvice_registerWatcher(500 /*poll interval ms*/,
    [](MemoryAdvice_MemoryState state, void* ctx) {
        switch (state) {
        case MEMORYADVICE_STATE_OK:
            break;
        case MEMORYADVICE_STATE_APPROACHING_LIMIT:
            reduce_texture_cache_size();    // proactive quality reduction
            break;
        case MEMORYADVICE_STATE_CRITICAL:
            drop_non_essential_resources(); // emergency release
            break;
        }
    }, myApp);

// Or poll manually per frame:
MemoryAdvice_MemoryState state = MemoryAdvice_getMemoryState();

// Cleanup:
MemoryAdvice_destroy();
```

### Percentage available estimate

```c
float pct = MemoryAdvice_getPercentageAvailableCapacity();
// 0.0 = critically low, 1.0 = plenty available
if (pct < 0.15f) drop_streaming_cache();
```

[Source: Memory Advice API](https://developer.android.com/games/sdk/memory-advice)

---

## 11. Android GPU Inspector

**Android GPU Inspector** (AGI, [github.com/google/agi](https://github.com/google/agi)) is Google's open-source GPU profiler, succeeding the deprecated Android GPU Profiler. It captures Vulkan API call traces and GPU performance counter data, integrates with **perfetto** for system-level tracing, and provides a desktop GUI for analysis.

### Two capture modes

**Frame capture**: Records all Vulkan API calls in one or more frames (analogous to RenderDoc on desktop). The capture is replayed on the device to measure per-draw-call GPU timings and validate API usage.

**System profiling**: Uses the **perfetto** tracing framework (Android 10+) to record GPU counter tracks (vertex throughput, texture cache hit rate, bandwidth, occupancy) alongside CPU scheduling, thread wakeup latency, and Choreographer vsync ticks. Output is a `.perfetto-trace` file viewable in the Perfetto UI.

### Supported GPU families

| Vendor | GPU | Counter support |
|---|---|---|
| Qualcomm | Adreno 5xx / 6xx / 7xx / 8xx | Full GPU counter support |
| ARM | Mali Bifrost / Valhall / Immortalis | Full GPU counter support |
| Imagination | PowerVR Series 9 | Partial |
| Google | Swiftshader (software) | Limited |

### Integration with Vulkan validation

AGI embeds the Khronos Vulkan Validation Layers and can highlight API misuse (missing synchronisation, suboptimal layout transitions, redundant barriers) alongside the performance data.

### Command-line capture for CI

```bash
# Capture 3 frames starting at frame 100:
agi capture --apk com.example.game --frames 100:103 --output capture.agi

# System profile for 5 seconds:
agi capture --apk com.example.game --duration 5000 --type perfetto \
    --output profile.perfetto-trace
```

### Perfetto GPU counter integration in app code

For apps that want to annotate the perfetto trace with their own events:

```c
#include <perfetto/tracing.h>

PERFETTO_DEFINE_CATEGORIES(
    perfetto::Category("game").SetDescription("Game events"));

// In render loop:
TRACE_EVENT("game", "RenderFrame");
TRACE_EVENT("game", "ShadowPass");
vkCmdDrawIndexed(cmd, indexCount, 1, 0, 0, 0);
```

[Source: Android GPU Inspector](https://gpuinspector.dev/), [perfetto GPU tracing](https://perfetto.dev/docs/data-sources/gpu)

---

## 12. Integrations

- **Ch85 — SurfaceFlinger and BufferQueue**: `ANativeWindow` is the producer end of a `BufferQueue`; SurfaceFlinger is the consumer. Every buffer submitted through a Vulkan swapchain or EGL surface traverses the pipeline described in Ch85. The HWComposer path (hardware overlay vs. GPU composition) applies equally to GameActivity windows.

- **Ch86 — Vulkan on Android**: The Vulkan swapchain creation path (`vkCreateAndroidSurfaceKHR`, `ANativeWindow`, TBDR-optimised render passes) is covered in depth in Ch86. `APP_CMD_INIT_WINDOW` / `APP_CMD_TERM_WINDOW` as the correct swapchain creation/destruction hooks are described there. Swappy's `SwappyVk_queuePresent` replaces `vkQueuePresentKHR` but sits above the Vulkan layer described in Ch86.

- **Ch86 — @FastNative / @CriticalNative**: The JNI transition cost that `@FastNative` reduces is the same cost that `GameActivity` eliminates from the per-frame render path. The two approaches are complementary: `@FastNative` optimises unavoidable JNI calls (framework boundaries); GameActivity eliminates JNI from the hot path entirely.

- **Ch87 — Android AR / ARCore / Android XR**: ARCore's `ArSession_update()` loop and the OpenXR `xrWaitFrame()` / `xrEndFrame()` loop are analogous to `android_main()`'s event loop. GameActivity and `ANativeWindow` are used in ALVR and WiVRn to host the decoded VR stream surface on the headset.

- **Ch26 — VA-API and Video Decode**: Android's `MediaCodec` codec pipeline, accessible from C via `AMediaCodec` (NDK), produces decoded frames as `AHardwareBuffer` that can be imported into Vulkan via `VK_ANDROID_external_memory_android_hardware_buffer` — the same AHardwareBuffer model described in Ch85 and Ch86. Oboe's AAudio MMAP path and VA-API's DMA-BUF zero-copy both aim at the same goal: eliminating unnecessary copies between hardware subsystems.

- **Ch75 — Explicit GPU Sync**: Swappy's fence-based backpressure mechanism (`SwappyVk_queuePresent` inserting `VkFence` waits) uses the same `dma_fence` / `sync_file` infrastructure described in Ch75. The explicit sync model now unified across Android and Wayland (`wp_linux_drm_syncobj_v1`) has its Android origin in the `android::Fence` / `sw_sync` timeline that Swappy builds on.

- **Ch96 — libcamera**: Android's `AMediaCodec` / Camera2 NDK (`ACameraManager`, `ACameraDevice`, `AImageReader`) parallels libcamera's role on Linux. The AGDK audio path (Oboe → AAudio → audio HAL) parallels PipeWire's role for audio on Linux desktop.

---

*[Source: AGDK overview](https://developer.android.com/games/agdk) | [GameActivity guide](https://developer.android.com/games/agdk/game-activity/overview) | [Paddleboat](https://developer.android.com/games/sdk/game-controller) | [Swappy](https://developer.android.com/games/sdk/frame-pacing) | [Oboe](https://github.com/google/oboe) | [Performance Tuner](https://developer.android.com/games/sdk/performance-tuner) | [Memory Advice](https://developer.android.com/games/sdk/memory-advice) | [AGI](https://gpuinspector.dev/)*
