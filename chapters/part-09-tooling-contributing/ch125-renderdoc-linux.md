# Chapter 125: RenderDoc on Linux

**Part IX — Tooling and Contributing**

**Audiences**: This chapter serves Vulkan and OpenGL application developers debugging rendering issues, graphics engineers diagnosing visual artifacts or performance bottlenecks, and Mesa driver developers who use frame capture for driver regression testing and shader correctness validation. Sections 2–5 cover the architecture and capture mechanisms that all three audiences need; Sections 6–7 focus on the UI tools primarily used by application developers; Section 8 (programmatic API) and Section 9 (Mesa driver workflow) are weighted toward CI engineers and driver developers.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [RenderDoc Architecture](#2-renderdoc-architecture)
3. [Installation on Linux](#3-installation-on-linux)
4. [Vulkan Frame Capture](#4-vulkan-frame-capture)
5. [OpenGL Frame Capture on Linux](#5-opengl-frame-capture-on-linux)
6. [The RenderDoc UI](#6-the-renderdoc-ui)
7. [Shader Debugging](#7-shader-debugging)
8. [Programmatic Capture API](#8-programmatic-capture-api)
9. [RenderDoc for Mesa Driver Development](#9-renderdoc-for-mesa-driver-development)
10. [Integrations with Linux Tooling](#10-integrations-with-linux-tooling)
11. [Integrations](#11-integrations)

---

## 1. Introduction

RenderDoc ([https://renderdoc.org/](https://renderdoc.org/), [https://github.com/baldurk/renderdoc](https://github.com/baldurk/renderdoc)) is the dominant open-source frame capture and GPU debugging tool for Vulkan, OpenGL, and OpenGL ES. Created by Baldur Karlsson, it is MIT-licensed and has become a de facto standard in Linux graphics development, shipping with or alongside Mesa's CI infrastructure, used by every major Mesa Vulkan driver team (RADV, ANV, NVK), and listed as a required tool for Khronos CTS debugging.

On Linux, RenderDoc captures a complete frame's API call stream, GPU resource contents, pipeline state, descriptor bindings, and enough context to replay the frame offline against any compatible driver. The resulting `.rdc` archive is the unit of exchange: a developer can capture against a shipping RADV build on their gaming machine, transfer the file to a developer workstation with a patched Mesa tree, replay it against the development driver, and compare pixel-by-pixel outputs. This workflow is impossible to replicate with traditional debuggers that operate only on CPU code.

RenderDoc's utility spans four categories of problem:

- **Visual artifacts**: incorrect pixels, missing geometry, wrong blend output — the Texture Viewer, Event Browser, and Pixel History pin the exact draw call and shader invocation responsible.
- **API correctness**: combined with `VK_LAYER_KHRONOS_validation`, RenderDoc correlates validation errors with specific pipeline events in the captured call stream.
- **Shader bugs**: the built-in shader debugger steps through SPIR-V (or GLSL for OpenGL) for a selected pixel or vertex, exposing intermediate values that GPU printf cannot reach.
- **Driver regression testing**: the programmatic Python API and `renderdoccmd` CLI enable automated capture-and-compare workflows in Mesa's CI pipelines.

As of v1.44 (May 2026, the latest stable release at time of writing), RenderDoc runs on x86_64 and AArch64 Linux, supports Vulkan 1.3 and OpenGL 3.2 through 4.6 / OpenGL ES 2.0+, and ships a Python 3 module for scriptable replay.

This chapter covers the architecture, capture mechanisms, UI tools, programmatic API, and Mesa developer workflows in depth. Chapter 30 introduces RenderDoc in the broader debugging context; this chapter is the dedicated deep-dive.

---

## 2. RenderDoc Architecture

RenderDoc is structured as a split-process system with an in-process capture library and an out-of-process replay tool, connected by the self-contained `.rdc` capture file.

### The Capture Library: `librenderdoc.so`

`librenderdoc.so` is loaded into the target application's address space before the application initialises its graphics context. It must be present before any Vulkan `vkCreateInstance` or OpenGL context creation call, because it hooks these entry points to intercept all subsequent graphics API calls. Two injection strategies exist on Linux:

1. **Via the RenderDoc launcher**: `renderdoccmd capture -- ./myapp` sets `LD_PRELOAD=librenderdoc.so` and launches the child process. The launcher also sets `RENDERDOC_CAPFILE` and `RENDERDOC_CAPOPTS` to communicate the capture configuration to the library.

2. **Programmatic preloading**: when the application itself loads `librenderdoc.so` at startup (via `dlopen`) and queries the in-application API, the library self-initialises without the launcher. This is the correct approach for headless servers and CI environments.

The library exposes a single exported C function, `RENDERDOC_GetAPI`, which returns a versioned struct of function pointers (the in-application API, detailed in Section 8). The library singleton `RenderDoc::Inst()` manages the overall capture state machine: idle → capturing → serialising → idle.

### The RenderDoc Singleton and Driver Abstraction Layer

Inside `librenderdoc.so`, the `RenderDoc` singleton owns two driver-specific sub-systems: `VulkanLayerManager` and `GLHook`. These implement the capture logic for each API and share common serialisation infrastructure. The singleton also manages the replay network connection (when `qrenderdoc` attaches to a running capture target) and the capture trigger state.

A critical design decision is that the `.rdc` file records *API calls*, not GPU command packets. RenderDoc does not capture PM4 packets, i915 batch buffers, or any hardware-specific encoding. It serialises the Vulkan or OpenGL calls in order, together with resource contents at the frame boundary. During replay, these API calls are re-issued through the system's current driver. This architecture means:

- Captures are completely portable across hardware from the same vendor (e.g., RX 6800 capture replays on RX 7900 via RADV).
- Captures are *not* portable across GPU architectures — a capture made with a vendor-specific extension or a particular VRAM layout may not replay on a different GPU family without the driver adapting resource placement.
- The replay tool can exercise a different Mesa build than was used during capture, making driver testing feasible.

### `qrenderdoc`: The GUI

`qrenderdoc` is the Qt-based graphical replay tool. It opens `.rdc` files, loads the appropriate driver (selected by Vulkan ICD or OpenGL loader), replays the API call stream up to a user-selected event, and exposes the GPU state at that point through its collection of inspection windows. It also functions as the capture host: when a target application is launched from `qrenderdoc`, the GUI maintains a network connection to the in-process capture library for live capture control and file retrieval.

### `renderdoccmd`: Headless Operation

`renderdoccmd` is the CLI companion. Its primary modes are:

```bash
# Capture mode: launch application, capture one frame, exit
renderdoccmd capture --wait-for-exit -c /tmp/output.rdc -- ./myapp --args

# Replay mode: replay a capture headlessly (for CI)
renderdoccmd replay /tmp/output.rdc

# Convert mode: export textures or data from a capture
renderdoccmd convert /tmp/output.rdc
```

In CI environments, `renderdoccmd` is the tool that performs replay against a development Mesa build, and the Python API (Section 8) extracts pixel data for comparison.

### In-Process vs Out-of-Process Replay

When `qrenderdoc` replays a capture, it does so in-process (within the `qrenderdoc` binary) by default. An out-of-process replay host (`renderdoccmd replay --remote-host`) is available for scenarios where replay must run on a headless GPU server while the GUI runs on a separate workstation. The two communicate over a TCP socket, allowing the GPU-side replay to run close to the target hardware.

---

## 3. Installation on Linux

### Distribution Packages

Most major distributions ship RenderDoc from their standard repositories:

```bash
# Fedora / RHEL
sudo dnf install renderdoc

# Ubuntu / Debian (backports PPA recommended for current versions)
sudo add-apt-repository ppa:baldurk/renderdoc
sudo apt install renderdoc

# Arch Linux
sudo pacman -S renderdoc
```

The distribution package installs `librenderdoc.so`, `qrenderdoc`, `renderdoccmd`, the Python module (`renderdoc.so`), and the Vulkan implicit layer manifest.

### AppImage Distribution

For users on distributions with older packages, RenderDoc ships an AppImage at [https://renderdoc.org/builds](https://renderdoc.org/builds). The AppImage bundles Qt and all runtime dependencies; extract and run:

```bash
chmod +x RenderDoc-v1.44-Linux-x86_64.tar.bz2.AppImage
./RenderDoc-v1.44-Linux-x86_64.tar.bz2.AppImage
```

The AppImage version installs the Vulkan layer manifest to `~/.local/share/vulkan/implicit_layer.d/` on first run.

### Flatpak Considerations

Running RenderDoc as a Flatpak, or capturing Flatpak-sandboxed applications, requires special care. The Flatpak sandbox restricts filesystem access, preventing `librenderdoc.so` from seeing the host's Vulkan ICD files and implicit layer manifests by default. Capturing a Flatpak application from a host-installed RenderDoc works if the application grants the `--filesystem=host` permission or `--device=all` and the `org.freedesktop.Platform` extension provides the necessary host libraries. Conversely, if RenderDoc itself is installed as a Flatpak (`org.freedesktop.Platform`-based), the implicit layer manifest will only be visible within the sandbox; host Vulkan applications will not load it without explicit layer path configuration (`VK_LAYER_PATH` pointing into the Flatpak's export directory). The safest configuration for development is to install RenderDoc from the distribution package or AppImage to the host system, outside any Flatpak sandbox.

### Building from Source

Building RenderDoc from source is straightforward. The primary Linux-specific dependencies (from [the Dependencies doc](https://github.com/baldurk/renderdoc/blob/v1.x/docs/CONTRIBUTING/Dependencies.md)) are:

- **Qt5 >= 5.6** (with `svg` and `x11extras` modules) or **Qt6** (recent builds; check `Dependencies.md` for the current requirement) for the GUI (`qrenderdoc`)
- **libx11**, **libxcb**, **libxcb-keysyms** (X11 window system integration)
- **libGL** (OpenGL capture hooks)
- **Vulkan headers** (Vulkan layer support)
- **python3-dev**, **bison**, **autoconf**, **automake**, **libpcre3-dev** (for the SWIG Python bindings generator)

On Ubuntu 22.04+ (Qt5 path; substitute `qt6-base-dev` etc. for Qt6 if your build system requires it):

```bash
sudo apt install cmake libx11-dev libx11-xcb-dev mesa-common-dev libgl1-mesa-dev \
    libxcb-keysyms1-dev pkg-config python3-dev bison autoconf automake \
    libpcre3-dev qt5-qmake libqt5svg5-dev libqt5x11extras5-dev
```

On Fedora:

```bash
sudo dnf install cmake qt5-qtbase-devel qt5-qtsvg-devel qt5-qtx11extras-devel \
    libX11-devel libxcb-devel mesa-libGL-devel python3-devel
```

Build:

```bash
git clone https://github.com/baldurk/renderdoc.git
cd renderdoc
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo ..
make -j$(nproc)
sudo make install
```

For distributions where `qmake` is not the Qt5 default (Fedora, CentOS), add `-DQMAKE_QT5_COMMAND=qmake-qt5` to the CMake invocation. [Source: RenderDoc CONTRIBUTING/Dependencies.md](https://github.com/baldurk/renderdoc/blob/v1.x/docs/CONTRIBUTING/Dependencies.md)

### Vulkan Implicit Layer Registration

RenderDoc registers as a Vulkan implicit layer, meaning the Vulkan loader activates it for every Vulkan application without requiring any change to the application. The manifest file is `renderdoc_capture.json` and is installed to one of the following standard implicit layer search paths, which the Vulkan loader scans in order:

- `/usr/share/vulkan/implicit_layer.d/renderdoc_capture.json` (system-wide, distribution packages)
- `/etc/vulkan/implicit_layer.d/renderdoc_capture.json` (system-wide override)
- `~/.local/share/vulkan/implicit_layer.d/renderdoc_capture.json` (per-user, AppImage install)

The manifest JSON specifies the layer name `VK_LAYER_RENDERDOC_Capture`, the path to `librenderdoc.so`, and a `disable_environment` key. Because RenderDoc uses a *disable* environment variable rather than an enable variable, the layer loads by default whenever the manifest is present. The exact variable name is specified in the `disable_environment` field of the manifest JSON (inspect `renderdoc_capture.json` on your system); a common convention is:

```bash
# Check your installed renderdoc_capture.json for the exact variable name
cat /usr/share/vulkan/implicit_layer.d/renderdoc_capture.json | grep disable
# Then disable accordingly, e.g.:
DISABLE_RENDERDOC=1 ./myapp
```

The disable-environment approach is intentional: RenderDoc must be active before `vkCreateInstance` to intercept the dispatch table, so opt-in (enable-only) registration would require restarting the process. [Source: RenderDoc Vulkan layer guide](https://renderdoc.org/vulkan-layer-guide.html)

### Explicit Layer Activation

To activate RenderDoc as an explicit layer (only when requested), use:

```bash
VK_INSTANCE_LAYERS=VK_LAYER_RENDERDOC_Capture ./myapp
```

Explicit activation is useful in environments where implicit layers are disabled by policy (e.g., security-hardened servers) or where you want to combine RenderDoc capture with another implicit layer's disable flag.

### System Requirements

- Linux kernel 4.x or later (render node DRM support)
- x86_64 or AArch64 (ARMv8-A)
- A Mesa or vendor Vulkan driver, or Mesa OpenGL 3.2+
- Sufficient GPU memory: RenderDoc retains one or more copies of all mapped memory allocations during capture (the "shadow copy" tracking described in Section 4), so peak VRAM usage during capture exceeds the application's normal baseline by the size of the mappable resource set

---

## 4. Vulkan Frame Capture

### The `VK_LAYER_RENDERDOC_Capture` Layer Mechanism

RenderDoc's Vulkan capture path is implemented entirely through the Vulkan layer system, which is purpose-built for this use case. Unlike the OpenGL path (Section 5), which requires process hooking, Vulkan layers receive a complete, officially-supported interception mechanism: the dispatch table.

Every dispatchable Vulkan object (`VkInstance`, `VkPhysicalDevice`, `VkDevice`, `VkCommandBuffer`, `VkQueue`) stores a pointer to a dispatch table as its first member. When the Vulkan loader calls through the layer chain, each layer replaces the dispatch table entries with its own function pointers. The real driver function is called from the end of the chain.

RenderDoc's layer registers wrapper functions for every Vulkan command, generated by a macro expansion system that handles 1 to 11 parameters. The core manually-implemented functions are `vkCreateInstance` (where RenderDoc initialises its per-instance state) and `vkCreateDevice` (where per-device state and command buffer tracking are initialised). All subsequent Vulkan calls flow through RenderDoc's wrappers. [Source: baldurk/renderdoc `vk_layer.cpp`](https://github.com/baldurk/renderdoc/blob/v1.x/renderdoc/driver/vulkan/vk_layer.cpp)

Importantly, RenderDoc does not register any library hooks or function pointer interceptions in Vulkan mode — it uses exclusively the Vulkan layer mechanism, avoiding the reliability issues that affect the OpenGL path.

### Capture Trigger

The default capture trigger is **F12** (the printscreen shortcut in many desktop environments may shadow this; `Print` may need to be rebound). Alternative triggers:

- Programmatic via the in-application C API (`StartFrameCapture` / `EndFrameCapture`, Section 8)
- Via the `qrenderdoc` GUI's "Trigger Capture" button when connected to a running application
- Via `renderdoccmd capture` which triggers on a specified frame number

### What Gets Captured

When the trigger fires, RenderDoc serialises:

1. **All `VkCommandBuffer` recordings** for the frame — every `vkCmd*` call in submission order, with arguments.
2. **Resource contents at frame start** — the content of every `VkImage` and `VkBuffer` that is read during the frame (textures, vertex buffers, uniform buffers, storage buffers). For images, RenderDoc uses the Vulkan API to read back contents, not GPU command packets.
3. **Pipeline state objects** — all `VkPipeline`, `VkPipelineLayout`, `VkRenderPass`, `VkDescriptorSetLayout` objects referenced during the frame.
4. **Descriptor sets** — the binding state at every draw call: which `VkImageView`, `VkBufferView`, or `VkSampler` is bound to each descriptor slot.
5. **Memory allocations** — RenderDoc instruments `vkMapMemory`/`vkUnmapMemory` and tracks every CPU write to mapped GPU-visible memory, so it can reconstruct buffer contents even when the buffer is written without an explicit staging copy.

### Resource Tracking

GPU memory tracking is one of the most complex aspects of Vulkan capture. RenderDoc wraps the return value of `vkMapMemory`: the application receives a pointer to a RenderDoc-controlled "shadow" buffer, not the raw driver memory. When the application unmaps, RenderDoc copies the written data to the real driver allocation and records the data in its capture state. This "shadow copy" approach adds memory overhead but is the only reliable way to reconstruct CPU-written buffer contents without GPU-side readback, which would block the render loop.

For GPU-written resources (shader storage buffers, images written by compute or fragment shaders), RenderDoc reads back the resource contents after the frame ends, before the capture is serialised to disk.

### Descriptor Tracking

Descriptor tracking records the complete state of every descriptor set at every `vkCmdDraw*` and `vkCmdDispatch` call. This is necessary because descriptors are not immutable in Vulkan — an application may reuse a descriptor set between draw calls by updating bindings. RenderDoc snapshots the effective descriptor state per-draw, so that replaying any individual draw call in the UI shows the correct resources.

### The `.rdc` File

The output `.rdc` file is a structured binary archive using RenderDoc's custom serialisation format. It is self-contained: opening a `.rdc` file on any machine with a compatible GPU and driver is sufficient to replay the frame. The file includes:

- Frame-level metadata (API version, application name, capture timestamp, GPU info)
- The full API call stream with arguments
- All resource initial contents (images and buffers at frame start)
- Shader module SPIR-V (as captured from the application — if the application compiled from GLSL at runtime, the SPIR-V is captured, not the source GLSL unless `OpLine` / `NonSemantic.Shader.DebugInfo.100` debug info is embedded)

Preview thumbnails (PNG) of framebuffer attachments are embedded for quick identification in the RenderDoc file browser.

### Multi-Frame Capture

RenderDoc captures a single frame by default. For applications where a visual issue only manifests after multiple frames of state accumulation, `StartFrameCapture` / `EndFrameCapture` (Section 8) can bracket multiple frames. The `.rdc` will contain all frames in sequence, and the Event Browser will show inter-frame boundaries.

---

## 5. OpenGL Frame Capture on Linux

### Hooking Mechanism

OpenGL's function dispatch on Linux does not have an official layer mechanism equivalent to Vulkan's dispatch tables. RenderDoc must hook into the process using dynamic library interception. The mechanism on Linux is:

1. **PLT (Procedure Linkage Table) hooking**: `librenderdoc.so` intercepts `dlopen()` at the system level using `RTLD_NEXT`. When the application (or its libraries) load `libEGL.so`, `libGL.so`, or `libGLX.so`, RenderDoc's `dlopen` hook fires. It then uses `plthook` to replace entries in the loaded library's PLT with RenderDoc's function pointers. [Source: baldurk/renderdoc `linux_hook.cpp`](https://github.com/baldurk/renderdoc/blob/v1.x/renderdoc/os/posix/linux/linux_hook.cpp)

2. **EGL function export**: RenderDoc exports EGL functions under their real names (e.g., `eglSwapBuffers`, `eglGetProcAddress`, `eglCreateContext`) directly from `librenderdoc.so`. When `librenderdoc.so` is preloaded via `LD_PRELOAD` or loaded first in the dynamic linker chain, the linker resolves these names to RenderDoc's implementations rather than the real EGL library's. RenderDoc's exported functions call `EnsureRealLibraryLoaded()` to obtain handles to the real EGL functions via `RTLD_NEXT`, then forward the call after recording it. [Source: baldurk/renderdoc `egl_hooks.cpp`](https://github.com/baldurk/renderdoc/blob/v1.x/renderdoc/driver/gl/egl_hooks.cpp)

The primary hooked EGL functions include:
- `eglCreateContext` / `eglDestroyContext` — context lifecycle tracking
- `eglMakeCurrent` — tracks which context is active on which thread
- `eglSwapBuffers` — the frame boundary trigger
- `eglGetProcAddress` — returns RenderDoc's implementations for any extension function that RenderDoc hooks; passes through unhooked functions to the real EGL

`glXGetProcAddress` (X11/GLX path) is similarly intercepted, allowing RenderDoc to intercept extension function pointer queries.

### Limitations vs. Vulkan

OpenGL capture is fundamentally harder than Vulkan for several reasons:

- **Implicit state accumulation**: OpenGL state is accumulated across calls with no explicit "frame start" marker equivalent to a render pass begin. RenderDoc must snapshot the complete OpenGL state machine at the capture start point.
- **Multi-context complexity**: OpenGL allows multiple contexts with shared objects. Shared textures and buffers must be tracked across contexts, and the capture must record the per-context and shared state independently.
- **Stateful extensions**: many OpenGL extensions add state that RenderDoc must explicitly handle. Unhandled extension state will cause replay divergence.
- **Thread safety**: OpenGL contexts are per-thread; `eglMakeCurrent` calls on different threads interleave with rendering calls, requiring RenderDoc's capture to be thread-aware.

### Captured State

For OpenGL, RenderDoc captures:

- Vertex Array Objects (VAOs) and Vertex Buffer Objects (VBOs) with their attribute layouts
- All active texture units and their bound texture objects (2D, cube, array, buffer)
- Framebuffer Objects (FBOs) and their color/depth/stencil attachment states
- Uniform Buffer Objects (UBOs) and their binding ranges
- Active shader program (`glUseProgram`) and shader source or binary
- Transform feedback state, atomic counter buffers, shader storage buffers (SSBO)
- Full fixed-function state: depth test, blend, stencil, culling, viewport, scissor

### Debug Group Markers

OpenGL debug groups (`GL_KHR_debug` extension, `glPushDebugGroup` / `glPopDebugGroup`) appear as labelled regions in the RenderDoc Event Browser, making it easy to identify subsystems within a captured frame:

```c
// C example: marking a shadow map pass for RenderDoc
glPushDebugGroup(GL_DEBUG_SOURCE_APPLICATION, 0,
                 -1, "Shadow Map Pass");
// ... shadow map draw calls ...
glPopDebugGroup();
```

[Source: GL_KHR_debug extension spec, https://registry.khronos.org/OpenGL/extensions/KHR/KHR_debug.txt]

### EGL + GBM for Headless OpenGL

Headless OpenGL capture (no display, DRM render node only) uses the EGL + GBM path. The application creates an `EGLDisplay` from a `gbm_device` (backed by `/dev/dri/renderD128`), creates an EGL context, and renders into an `EGLImage`-backed renderbuffer. RenderDoc captures this path because it hooks `eglCreateContext` and `eglSwapBuffers` regardless of whether a window system surface is involved. For purely offscreen rendering that never calls `eglSwapBuffers`, use the programmatic `StartFrameCapture`/`EndFrameCapture` API (Section 8) to demarcate the frame boundary.

### OpenGL ES on Linux

OpenGL ES capture on Linux is supported via two paths:
1. **Mesa's GLES implementation**: Mesa exposes GLES through its Gallium or Vulkan (Zink) backends. RenderDoc captures through the EGL hooks.
2. **ANGLE** (OpenGL ES over Vulkan): ANGLE ([https://chromium.googlesource.com/angle/angle](https://chromium.googlesource.com/angle/angle)) translates GLES to Vulkan; RenderDoc can capture at either the GLES level (via the EGL hook) or at the underlying Vulkan level (via the Vulkan layer). Capturing at the Vulkan level provides better introspection since the Vulkan path has full resource tracking.

---

## 6. The RenderDoc UI

`qrenderdoc` provides seven primary inspection windows, each tied to the currently selected event in the Event Browser.

### Event Browser

The Event Browser is the timeline of the captured frame. API calls are presented in submission order, colour-coded by type:

| Color | Call type |
|-------|-----------|
| Blue | Draw calls (`vkCmdDraw`, `vkCmdDrawIndexed`, `glDrawElements`, etc.) |
| Orange | Dispatch calls (`vkCmdDispatch`, `glDispatchCompute`) |
| Yellow | Pipeline barriers, layout transitions, synchronisation |
| Red | Clear operations |
| White | State-setting calls, buffer copies, render pass management |

Render passes (Vulkan) and debug groups (OpenGL) appear as collapsible tree nodes. Selecting any event — including non-draw calls — advances the replay to that point and refreshes all other windows to show the state immediately after the selected call.

The Event Browser also shows the **Event ID (EID)**, a monotonically increasing integer that uniquely identifies every call in the capture. EIDs are the primary addressing mechanism in the Python API (Section 8).

### Texture Viewer

The Texture Viewer renders any captured texture (or framebuffer attachment) at any event in the frame. Controls include:

- **Mip level** and **array layer** selectors (for mipmapped or arrayed textures)
- **Channel mask**: isolate R, G, B, or A channels; view depth buffer as a linear greyscale
- **HDR tonemapping**: for floating-point textures with out-of-[0,1] values
- **Flip Y**: for textures in upside-down coordinate systems (common with Vulkan vs. OpenGL origin differences)
- **Sub-region zoom**: click to inspect individual pixels; the status bar shows the exact texel value in the texture's native format

The Texture Viewer also supports the **Pixel History** workflow: right-clicking a pixel shows every draw call that wrote to that pixel during the frame, with pre- and post-draw colour values, depth test result, and coverage information. This is the primary tool for diagnosing "why is this pixel wrong?" without requiring a shader debugger session.

### Mesh Output Viewer

The Mesh Output viewer visualises vertex data at two stages:

- **VS Input**: the raw vertex buffer contents as fed to the vertex shader — positions, normals, UVs, as defined by the VAO/VkPipelineVertexInputState
- **VS Output**: the clip-space positions output by the vertex shader

The viewer renders a 3D interactive viewport of the geometry, colour-coded by a selectable vertex attribute. This is invaluable for diagnosing geometry that disappears (clipped, behind the camera, degenerate triangles) or shows incorrect positions (vertex shader bug, wrong buffer binding).

### Pipeline State Inspector

The Pipeline State window presents the complete pipeline configuration active at the selected event. For Vulkan, this includes:

- **Vertex Input**: VkPipelineVertexInputStateCreateInfo — binding strides, attribute locations and formats
- **Vertex Shader** / **Fragment Shader** / all other shader stages: SPIR-V module hash, entry point, shader resource interface
- **Rasterizer**: fill mode, cull mode, front face, depth bias
- **Depth/Stencil**: depth test function, depth write enable, stencil operations
- **Colour Blend**: per-attachment blend state, blend factors, blend operations
- **Descriptor Sets**: for each set and binding, the resource type, the bound `VkImageView` or `VkBuffer`, and whether the resource is valid at this point
- **Dynamic State**: all values set via `vkCmdSet*` calls — viewport, scissor, blend constants

This window is the fastest path to diagnosing state-setting bugs: depth test disabled when it shouldn't be, wrong cull mode, incorrect descriptor binding.

### Resource Inspector

The Resource Inspector lists all API objects captured in the frame — every `VkImage`, `VkBuffer`, `VkPipeline`, `VkDescriptorSet`, `VkShaderModule`, and their OpenGL equivalents. Each resource shows its creation parameters, usage events (every draw call that reads or writes it), and its contents at any selected point.

### Performance Counters Window

RenderDoc provides GPU timing for each draw call via driver-level query pools. On Vulkan, per-draw call GPU timestamps are collected using `VkQueryPool` with `VK_QUERY_TYPE_TIMESTAMP`; the resulting nanosecond-resolution timestamps are displayed per-EID in the Event Browser. This is distinct from hardware performance counters (Section 10): RenderDoc reports wall-clock GPU time per call, not cache miss rates or shader occupancy. For that level of detail, hardware tools such as RadeonGPUProfiler or `intel_gpu_top` (Section 10) are needed.

---

## 7. Shader Debugging

RenderDoc's shader debugger is its most technically ambitious feature. It allows stepping through a shader invocation for a specific pixel or vertex, observing intermediate values at each SPIR-V (or GLSL) instruction.

### How It Works: Software SPIR-V Emulation

RenderDoc does **not** replay the problematic shader invocation on the GPU for debugging. Instead, it emulates the SPIR-V execution in software on the CPU. When you launch a debug session for a pixel, RenderDoc:

1. Replays the full frame up to the selected draw call, so all input resources (textures, buffers, descriptors) are in the correct state.
2. Reads back the interpolated vertex attributes and per-fragment inputs for the selected invocation from the GPU.
3. Executes the SPIR-V module in a software interpreter, feeding the captured input values, and records the complete trace of intermediate values per-instruction.
4. Presents the recorded trace to the user as a stepping debugger, with a watch window showing variable values at each step.

This CPU-side emulation approach means the debug session does not require GPU time during stepping — the GPU work is done in step 1, and stepping through the trace is entirely CPU-side. It also means there is no risk of GPU device loss or timeout during the debug session.

### Debug Pixel Workflow

To debug a fragment shader:

1. Open the **Texture Viewer** and navigate to the render target output of the draw call of interest.
2. Right-click the target pixel.
3. Select **"Debug pixel (X, Y)"** from the context menu.
4. If the current draw does not write to the selected pixel, the Pixel History window opens first; select the draw from the list, then launch the debugger.

The debugger opens with the SPIR-V disassembly or (if debug info is present) the GLSL/HLSL source, positioned at the first instruction. Stepping controls (F10 step-over, F11 step-into) advance through the shader. The **Registers window** shows all active temporaries and output variables updating in real time. The **Watch window** allows expressions over shader variables.

### Source-Level Debugging with `OpLine` / `NonSemantic.Shader.DebugInfo.100`

When SPIR-V contains debug information, RenderDoc displays source-level GLSL or HLSL and steps through source lines instead of SPIR-V instructions. Two mechanisms provide debug information:

1. **`OpLine` decorations**: generated by `glslang`/`glslc` with the `-g` flag, `OpLine` maps each SPIR-V instruction back to a source file and line number. This is the minimal debug info.

2. **`NonSemantic.Shader.DebugInfo.100`**: the extended non-semantic instruction set authored by Baldur Karlsson (from RenderDoc) and derived from `OpenCL.DebugInfo.100` and DWARF. It provides full variable names, types, scopes, and function call information. Generated by `glslc -g` or `glslang --generate-debug-info`. [Source: LunarG Source-level Shader Debugging paper, https://www.lunarg.com/wp-content/uploads/2023/02/Source-level-Shader-Debugging-VULFEB2023.pdf]

Compiling shaders with debug info for RenderDoc:

```bash
# glslc: embed OpLine and NonSemantic.Shader.DebugInfo.100
glslc -g -o shader.frag.spv shader.frag

# glslang: equivalent
glslangValidator -g -V shader.frag -o shader.frag.spv
```

### Vertex and Compute Debugging

**Vertex shader debugging**: from the Mesh Output viewer, right-click any vertex in the VS Input list and select "Debug Vertex". The debugger opens with the input attributes for that vertex pre-loaded.

**Compute shader debugging**: from the Pipeline State window, navigate to the compute shader stage. Enter the **thread group** (X, Y, Z) and **thread within group** (X, Y, Z) to debug. The software emulator runs that specific thread's execution.

### Limitations

The software SPIR-V emulation has well-defined limitations:

- **Subgroup operations**: `OpGroupNonUniform*` instructions that depend on inter-lane communication within a subgroup (SIMD wave) cannot be accurately emulated when the emulator runs a single invocation in isolation. RenderDoc may produce incorrect results for shaders that rely on subgroup ballot, shuffle, or vote.
- **Divergent control flow**: branches that depend on non-uniform values across a subgroup are emulated correctly for the selected invocation, but the emulator cannot capture the wavefront's actual divergence behaviour.
- **Extensions**: some Vulkan extensions introduce SPIR-V instructions that the emulator may not implement. The debugger will report "unsupported feature" and abort if it encounters an unrecognised instruction.
- **Mesh shaders and task shaders**: debugging for these stages has limited support as of v1.44 and may be incomplete.
- **OpenGL**: RenderDoc supports vertex, fragment, and compute shader debugging for OpenGL (core profile 3.2+). Geometry and tessellation shader debugging has more limited support.

### Shader Editing and Replacement

In addition to the debugger, RenderDoc supports live shader editing during replay. In any shader stage view within the Pipeline State window, click "Edit" to open the shader's source (GLSL if available, or SPIR-V disassembly) in an editor. Modify the shader and press F5 ("Apply"). RenderDoc compiles the edited shader (using bundled tools: `spirv-as`, SPIRV-Cross, `glslang`) and re-runs the frame with the replacement, updating all windows to show the new output. This allows rapid hypothesis testing — "what if the normal calculation is wrong?" — without recompiling the application. [Source: RenderDoc shader editing docs, https://renderdoc.org/docs/how/how_edit_shader.html]

---

## 8. Programmatic Capture API

The in-application API allows a Vulkan or OpenGL application to control RenderDoc capture programmatically — starting and stopping captures, setting output paths, and querying completed captures. This is the correct mechanism for CI pipelines where F12 is not viable.

### Loading the API

The API is loaded at runtime via `dlopen` + `dlsym`, avoiding a hard dependency on `librenderdoc.so` being present. The single exported function is `RENDERDOC_GetAPI`:

```c
#include <dlfcn.h>
#include "renderdoc_app.h"   // from https://github.com/baldurk/renderdoc/blob/v1.x/renderdoc/api/app/renderdoc_app.h

RENDERDOC_API_1_6_0 *rdoc_api = NULL;

void init_renderdoc(void)
{
    /* RTLD_NOLOAD: only succeed if librenderdoc.so is already loaded
     * (i.e. the RenderDoc launcher or LD_PRELOAD injected it).
     * Falls back gracefully if RenderDoc is absent. */
    void *mod = dlopen("librenderdoc.so", RTLD_NOW | RTLD_NOLOAD);
    if (!mod)
        return;  // RenderDoc not present, proceed normally

    pRENDERDOC_GetAPI get_api =
        (pRENDERDOC_GetAPI)dlsym(mod, "RENDERDOC_GetAPI");
    if (!get_api)
        return;

    int ret = get_api(eRENDERDOC_API_Version_1_6_0, (void **)&rdoc_api);
    if (ret != 1)
        rdoc_api = NULL;  // version not supported
}
```

[Source: RenderDoc in-application API docs, https://renderdoc.org/docs/in_application_api.html]

### RENDERDOC_API_1_6_0 Function Pointers

The `RENDERDOC_API_1_6_0` struct (current as of v1.44, which is API-compatible with 1.6.0 as the minimum) exposes the following key functions:

```c
// Begin capture immediately.
// RENDERDOC_DevicePointer: VkDevice cast to void*, or NULL for wildcard.
// RENDERDOC_WindowHandle: native window handle (e.g. xcb_window_t), or NULL.
rdoc_api->StartFrameCapture(
    (RENDERDOC_DevicePointer)vk_device,
    (RENDERDOC_WindowHandle)NULL);

// End capture and write to disk; returns 1 on success
uint32_t result = rdoc_api->EndFrameCapture(
    (RENDERDOC_DevicePointer)vk_device,
    (RENDERDOC_WindowHandle)NULL);

// Set the base path for output files
// Generates: "my_captures/test_frame0000.rdc", "test_frame0001.rdc", ...
rdoc_api->SetCaptureFilePathTemplate("my_captures/test");

// Trigger capture on next present (equivalent to F12)
rdoc_api->TriggerCapture();

// Query number of completed captures
uint32_t n = rdoc_api->GetNumCaptures();

// Get path of completed capture i
char path[512];
uint32_t pathlen = sizeof(path);
uint64_t timestamp;
rdoc_api->GetCapture(i, path, &pathlen, &timestamp);

// Set a capture option (enum eRENDERDOC_CaptureOption)
rdoc_api->SetCaptureOptionU32(
    eRENDERDOC_Option_APIValidation, 1);      // enable API validation during replay
rdoc_api->SetCaptureOptionU32(
    eRENDERDOC_Option_CaptureCallstacks, 1);  // record CPU callstack per API call
```

Passing `NULL` for both `device` and `window` captures the active API on the current thread, which is sufficient for single-GPU, single-window applications.

As of API version 1.6.0, `SetCaptureTitle(const char*)` allows assigning a human-readable title to a capture in progress, visible in the qrenderdoc file browser.

### CI Capture Pattern

The canonical CI pattern captures a specific frame during a render test, then uses the Python API to extract pixel data for comparison:

```c
// Application: capture frame N
static int frame_count = 0;

void my_present_frame(void)
{
    if (frame_count == TARGET_FRAME && rdoc_api) {
        rdoc_api->SetCaptureFilePathTemplate("/ci/captures/test_" TEST_NAME);
        rdoc_api->StartFrameCapture(NULL, NULL);
    }

    do_actual_present();  // vkQueuePresentKHR or eglSwapBuffers

    if (frame_count == TARGET_FRAME && rdoc_api) {
        rdoc_api->EndFrameCapture(NULL, NULL);
        // Retrieve the path of the completed capture
        char path[512]; uint32_t len = sizeof(path); uint64_t ts;
        rdoc_api->GetCapture(0, path, &len, &ts);
        printf("Capture written: %s\n", path);
    }
    frame_count++;
}
```

The resulting `.rdc` file is then analysed by a Python script (shown below) to extract the final framebuffer for pixel comparison.

### Python API for Automated Analysis

`renderdoc.so` (the Python module) exposes the complete replay infrastructure:

```python
import renderdoc as rd
import sys

def compare_capture(rdc_path, reference_png, output_png):
    rd.InitialiseReplay(rd.GlobalEnvironment(), [])

    cap = rd.OpenCaptureFile()
    if cap.OpenFile(rdc_path, '', None) != rd.ResultCode.Succeeded:
        raise RuntimeError(f"Cannot open {rdc_path}")
    if not cap.LocalReplaySupport():
        raise RuntimeError("No local replay support")

    result, ctrl = cap.OpenCapture(rd.ReplayOptions(), None)
    if result != rd.ResultCode.Succeeded:
        raise RuntimeError(f"Replay init failed: {result}")

    # Navigate to the last action (final present)
    actions = ctrl.GetRootActions()
    ctrl.SetFrameEvent(actions[-1].eventId, True)

    # Export the swapchain / back-buffer texture
    textures = ctrl.GetTextures()
    for tex in textures:
        if tex.width >= 1280 and tex.height >= 720:
            save = rd.TextureSave()
            save.resourceId = tex.resourceId
            save.destType = rd.FileType.PNG
            save.mip = 0
            save.slice.sliceIndex = 0
            ctrl.SaveTexture(save, output_png)
            break

    ctrl.Shutdown()
    cap.Shutdown()
    rd.ShutdownReplay()

    # Pixel comparison (example using Pillow)
    from PIL import Image
    import numpy as np
    ref = np.array(Image.open(reference_png))
    out = np.array(Image.open(output_png))
    diff = np.abs(ref.astype(int) - out.astype(int))
    max_diff = diff.max()
    if max_diff > THRESHOLD:
        raise AssertionError(f"Pixel regression: max diff {max_diff} > {THRESHOLD}")

if __name__ == '__main__':
    compare_capture(sys.argv[1], sys.argv[2], sys.argv[3])
```

[Source: RenderDoc Python API docs, https://renderdoc.org/docs/python_api/index.html]

Running without a display requires setting `LD_LIBRARY_PATH` to the RenderDoc library path and using the Mesa software renderer or a headless GPU:

```bash
LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu/renderdoc \
VK_ICD_FILENAMES=/path/to/dev/mesa/radeon_icd.x86_64.json \
python3 compare.py test.rdc reference.png output.png
```

---

## 9. RenderDoc for Mesa Driver Development

### The Mesa Driver Testing Workflow

Mesa driver developers use RenderDoc at multiple stages of the development cycle. The typical workflow:

1. **Capture**: run a real application (game, benchmark, Blender scene) against the shipping Mesa build and capture a frame that exercises the code path under development.
2. **Replay against development build**: set `VK_ICD_FILENAMES` to the development Mesa ICD and replay.
3. **Compare**: use the Python API or manual visual inspection to compare the output of the development build against the capture.

```bash
# Capture phase: use shipping RADV
renderdoccmd capture --wait-for-exit -c game_frame.rdc -- ./game

# Replay phase: use development RADV build
VK_ICD_FILENAMES=/home/dev/mesa-build/src/amd/vulkan/radeon_icd.x86_64.json \
renderdoccmd replay game_frame.rdc
```

This workflow is particularly powerful for:
- **Regression hunting**: a new Mesa commit produces wrong output. Capture before and after the regression commit, replay both through the new build, compare.
- **New extension development**: implement a new extension, capture a real application that uses it, replay to verify the output matches the reference.
- **Shader compiler bugs**: a shader produces wrong pixels. Use RenderDoc's Texture Viewer to confirm which shader stage and which draw call are responsible before diving into the NIR/ACO dumps.

### `renderdoccmd replay` for Headless Driver Testing

`renderdoccmd replay` runs a captured frame headlessly, using the Vulkan driver specified by the environment (or `VK_ICD_FILENAMES`). The exit code reflects replay success; a non-zero exit indicates replay errors. Combined with the Python API for pixel extraction, this provides a complete headless GPU test framework:

```bash
#!/bin/bash
# Mesa CI script: replay a captured frame and compare pixels

MESA_BUILD=$1
CAPTURE=$2
REFERENCE=$3

VK_ICD_FILENAMES=$MESA_BUILD/radeon_icd.x86_64.json \
LD_LIBRARY_PATH=/usr/lib/renderdoc \
python3 /ci/scripts/renderdoc_compare.py "$CAPTURE" "$REFERENCE" /tmp/output.png

if [ $? -ne 0 ]; then
    echo "REGRESSION: pixel comparison failed for $CAPTURE"
    exit 1
fi
echo "PASS: $CAPTURE"
```

### Shader Hot-Patching for Driver Debugging

When a shader produces incorrect output and the driver's shader compiler is suspected, the RenderDoc shader editor (Section 7) allows modifying the SPIR-V and immediately replaying. For driver developers:

1. Open the capture in `qrenderdoc`.
2. Navigate to the failing draw call.
3. Open the fragment shader in the Pipeline State viewer → Edit.
4. Simplify the shader (remove operations one by one) to identify the instruction that triggers the driver bug.
5. Use the simplified SPIR-V as a test case for a Mesa bug report or unit test.

This is faster than modifying the application source, recompiling, and re-capturing for each hypothesis.

### RenderDoc vs. apitrace

`apitrace` ([https://github.com/apitrace/apitrace](https://github.com/apitrace/apitrace)) is the alternative OpenGL trace tool on Linux. The two tools have complementary strengths:

| Feature | RenderDoc | apitrace |
|---------|-----------|----------|
| Vulkan support | Full | No |
| OpenGL support | Core 3.2–4.6 | Broader legacy GL coverage |
| GPU resource inspection | Full (Texture Viewer, etc.) | Limited |
| Shader debugging | Yes (SPIR-V emulation) | No |
| Replay portability | Limited (same GPU arch) | Better (API-level) |
| CI/headless | Yes (Python API) | Yes (`glretrace`) |
| Multi-frame capture | Via API | Yes |

For pure OpenGL traces, particularly legacy OpenGL 1.x/2.x or older extension usage that RenderDoc's core-profile hooking may miss, `apitrace` remains the more compatible choice. For Vulkan or any scenario requiring GPU resource inspection or shader debugging, RenderDoc is the correct tool. [Source: gfxreconstruct LunarG blog, https://www.lunarg.com/mastering-gfxreconstuct-part-1/]

---

## 10. Integrations with Linux Tooling

### `VK_LAYER_KHRONOS_validation` + RenderDoc

Running both `VK_LAYER_KHRONOS_validation` and `VK_LAYER_RENDERDOC_Capture` simultaneously is the standard development configuration. Validation errors appear in `stderr` as usual; RenderDoc's Event Browser in the replayed capture shows the event ID of the call that triggered the validation error, allowing correlation. The Validation Layer should be listed before RenderDoc in `VK_INSTANCE_LAYERS` to ensure validation runs on the unintercepted API calls before RenderDoc processes them:

```bash
VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation:VK_LAYER_RENDERDOC_Capture ./myapp
```

### gfxreconstruct (GFXR)

`gfxreconstruct` ([https://github.com/LunarG/gfxreconstruct](https://github.com/LunarG/gfxreconstruct), layer name `VkLayer_gfxreconstruct`) is a LunarG-maintained Vulkan capture tool with a different design philosophy from RenderDoc. Differences relevant to Linux graphics developers:

- **Multi-frame capture**: GFXR captures multiple frames by default; RenderDoc captures one frame.
- **Cross-architecture replay**: GFXR can remap Vulkan memory types and alignment requirements between GPU architectures (e.g., capture on discrete GPU, replay on mobile GPU). RenderDoc does not currently do this.
- **Headless focus**: GFXR is designed for long-capture, headless, and CI scenarios. Its `gfxrecon-replay` binary supports trimming captures to specific frame ranges.
- **No GPU resource inspection UI**: GFXR is a replay tool, not an interactive debugger. It does not have a texture viewer, shader debugger, or pipeline state inspector.

GFXR captures can be imported into RenderDoc in some workflows, and vice versa (with limitations). For regression testing across GPU architectures, GFXR is the correct primary tool; for interactive debugging, RenderDoc is the correct choice.

### `perf` + RenderDoc: CPU + GPU Correlation

`perf` captures CPU-side events (cache misses, branch mispredictions, function call graphs) independently of RenderDoc's GPU capture. Correlating the two requires a common timestamp. `VK_EXT_calibrated_timestamps` (available in RADV and ANV) provides GPU timestamps in terms of `CLOCK_MONOTONIC`, the same clock `perf` uses. Workflow:

1. Instrument the application to write `CLOCK_MONOTONIC` timestamps around the captured frame.
2. Capture the frame with RenderDoc; per-draw GPU timestamps are available in the Performance Counters window.
3. Record a `perf` trace for the same application run (separate run, or use perf's attach mode).
4. Correlate CPU-side driver overhead (visible in the `perf` flamegraph for `radv_CmdDraw`, `anv_CmdDraw`, etc.) with GPU timestamps from RenderDoc.

This combined view is described in more detail in Chapter 30 (Section 8: CPU-side profiling) and Chapter 93 (GPU performance analysis methodology).

### `intel_gpu_top` and AMD `umr`

After identifying the expensive draw call in RenderDoc, hardware-level tools provide the next layer of detail:

- **`intel_gpu_top`** ([https://drm.pages.freedesktop.org/igt-gpu-tools/](https://drm.pages.freedesktop.org/igt-gpu-tools/)): reads Intel GPU engine utilisation from the `i915_perf` OA stream. Run it during the application's render loop to see render/blitter/compute engine busy percentages correlated with the RenderDoc-identified bottleneck.

- **AMD `umr`** (Userspace Micro Register Debugger, [https://gitlab.freedesktop.org/tomstdenis/umr](https://gitlab.freedesktop.org/tomstdenis/umr)): provides live register inspection, wave state dumps, and PM4 packet traces from the amdgpu driver. After RenderDoc identifies the shader stage or draw call responsible for a GPU hang, `umr`'s wave dump (`umr -wa`) can show the per-CU wave state at the time of the hang. Requires `CAP_SYS_ADMIN` or root.

### `VK_LAYER_LUNARG_api_dump` as Alternative

`VK_LAYER_LUNARG_api_dump` logs every Vulkan call to a file or stdout. Unlike RenderDoc, it captures all frames (not just a trigger frame) and includes call arguments as text. It is useful for debugging Vulkan loader or layer initialization issues that occur before any frame is rendered — a scenario where RenderDoc cannot capture because the application never reaches a present. [Source: LunarG Vulkan tools docs]

### PIX on Linux (Microsoft)

Microsoft's PIX GPU debugger, available for DirectX 12 via VKD3D-Proton on Linux, provides a RenderDoc-equivalent for DXVK-translated D3D12 workloads. PIX on Linux targets DirectX 12 traces captured through VKD3D-Proton and is discussed in the context of DXVK/Proton in Chapter 104. For native Vulkan applications on Linux, RenderDoc remains the primary tool.

### Mesa VKMS for Fully Headless Capture

Mesa's VKMS (Virtual Kernel Mode Setting) virtual DRM driver provides a software KMS output with no physical display hardware required. Combined with Mesa's software Vulkan (Lavapipe) or the hardware driver on a headless cloud GPU, VKMS enables fully headless RenderDoc capture in CI environments where no display output is connected:

```bash
# Load VKMS
sudo modprobe vkms

# Set DISPLAY for Xvfb backed by VKMS
Xvfb :99 -screen 0 1280x720x24 &
DISPLAY=:99 renderdoccmd capture --wait-for-exit -c /ci/output.rdc -- ./gl_app
```

For Vulkan headless capture (no display server at all), the programmatic API path (Section 8) using `StartFrameCapture`/`EndFrameCapture` is the cleanest option and does not require a window system surface.

---

## 11. Integrations

This chapter connects to the following chapters across the book:

- **Chapter 18 (RADV — Mesa AMD Vulkan driver)**: RADV is the primary Linux Vulkan driver captured and replayed by RenderDoc. The RADV team uses RenderDoc captures for regression detection; `VK_ICD_FILENAMES` replay against development RADV builds is a standard workflow.

- **Chapter 19 (ANV / Intel Vulkan, iris OpenGL)**: ANV is the second major Mesa Vulkan driver targeted by RenderDoc. Intel's SPIR-V debug info (`OpLine` via `VK_EXT_debug_utils`) enables source-level shader debugging in RenderDoc on Intel hardware. The iris OpenGL driver is captured via RenderDoc's EGL hooks.

- **Chapter 24 (Vulkan API)**: The Vulkan layer mechanism (`VkLayerDispatchTable`, implicit layer JSON manifests) is the foundation of RenderDoc's Vulkan capture. Understanding Vulkan's dispatch architecture (Chapter 24) explains why RenderDoc's Vulkan path is more reliable than its OpenGL path.

- **Chapter 30 (Debugging and Profiling)**: Chapter 30 introduces RenderDoc as part of the broader Linux GPU debugging toolkit. This chapter expands that introduction into a complete reference. Chapter 30's apitrace, validation layers, and Mesa debug environment variable sections complement the tooling described here.

- **Chapter 75 (Explicit sync — DRM sync objects and Vulkan timeline semaphores)**: RenderDoc must correctly track Vulkan semaphores and fences to replay frames in the correct submission order. Timeline semaphores (`VK_KHR_timeline_semaphore`) require particular care during serialisation, because their signalled values advance monotonically and must be reset correctly during replay.

- **Chapter 104 (DXVK and VKD3D-Proton — DirectX on Linux)**: RenderDoc can capture DXVK's Vulkan translation layer. The captured frame reflects the translated Vulkan calls, not the original D3D calls, making RenderDoc directly applicable to DXVK driver debugging without needing D3D-level capture.

- **Chapter 109 (Mesa Testing and CI)**: Mesa's CI uses RenderDoc captures for visual regression testing via the Python API. The automated capture-and-compare workflow described in Section 8 feeds directly into the dEQP and piglit image comparison infrastructure described in Chapter 109.

- **Chapter 110 (SPIR-V)**: RenderDoc captures, displays, and software-emulates SPIR-V shaders. The `NonSemantic.Shader.DebugInfo.100` extended instruction set (Chapter 110, Section: debug info extensions) enables source-level shader stepping in RenderDoc. `spirv-opt`, `spirv-dis`, and SPIRV-Cross (all described in Chapter 110) are the external tools RenderDoc invokes for shader editing and disassembly.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
