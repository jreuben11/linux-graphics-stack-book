# Chapter 39h: Dear ImGui — Immediate-Mode GUI for Developer Tools and Debug Overlays

**Audience**: Graphics application developers adding debug overlays or editor UIs to Vulkan/OpenGL applications on Linux; systems developers integrating ImGui into an existing renderer; anyone comparing immediate-mode and retained-mode UI toolkit trade-offs.

---

## Table of Contents

- [1. Design Philosophy and Use Cases](#1-design-philosophy-and-use-cases)
  - [1.1 The Immediate-Mode Paradigm](#11-the-immediate-mode-paradigm)
  - [1.2 Why Tools, Not Consumer UI](#12-why-tools-not-consumer-ui)
  - [1.3 Comparison with Retained-Mode Toolkits](#13-comparison-with-retained-mode-toolkits)
- [2. Core Architecture](#2-core-architecture)
  - [2.1 ImGuiContext](#21-imguicontext)
  - [2.2 The Draw List Pipeline](#22-the-draw-list-pipeline)
  - [2.3 Vertex and Index Format](#23-vertex-and-index-format)
- [3. Font Atlas](#3-font-atlas)
  - [3.1 Rasterisation: stb_truetype vs FreeType](#31-rasterisation-stb_truetype-vs-freetype)
  - [3.2 Texture Packing and Upload](#32-texture-packing-and-upload)
  - [3.3 Dynamic Atlas (1.92.0+)](#33-dynamic-atlas-1920)
- [4. The ID System](#4-the-id-system)
  - [4.1 Hash Chain](#41-hash-chain)
  - [4.2 PushID / PopID](#42-pushid--popid)
  - [4.3 `##` and `###` Label Syntax](#43--and--label-syntax)
- [5. Backend Architecture](#5-backend-architecture)
  - [5.1 Platform vs Renderer Split](#51-platform-vs-renderer-split)
  - [5.2 Linux Backend Inventory](#52-linux-backend-inventory)
- [6. The Vulkan Backend](#6-the-vulkan-backend)
  - [6.1 Initialization](#61-initialization)
  - [6.2 Dynamic Rendering Path](#62-dynamic-rendering-path)
  - [6.3 Vertex/Index Buffer Upload](#63-vertexindex-buffer-upload)
  - [6.4 Descriptor Management](#64-descriptor-management)
  - [6.5 Custom Textures](#65-custom-textures)
- [7. The OpenGL 3 Backend](#7-the-opengl-3-backend)
- [8. Platform Backends on Linux](#8-platform-backends-on-linux)
  - [8.1 SDL3 Backend and Wayland](#81-sdl3-backend-and-wayland)
  - [8.2 GLFW Backend and Wayland](#82-glfw-backend-and-wayland)
- [9. Build and Integration](#9-build-and-integration)
  - [9.1 Copy-Source Approach](#91-copy-source-approach)
  - [9.2 CMake Patterns](#92-cmake-patterns)
  - [9.3 Compile-Time Configuration](#93-compile-time-configuration)
- [10. The Docking Branch](#10-the-docking-branch)
- [11. Ecosystem and Extensions](#11-ecosystem-and-extensions)
  - [11.1 ImPlot](#111-implot)
  - [11.2 ImNodes / imgui-node-editor](#112-imnodes--imgui-node-editor)
  - [11.3 ImGuizmo](#113-imguizmo)
  - [11.4 imgui_club](#114-imgui_club)
  - [11.5 Tracy Profiler](#115-tracy-profiler)
- [12. Integrations](#12-integrations)

---

## 1. Design Philosophy and Use Cases

### 1.1 The Immediate-Mode Paradigm

Dear ImGui (v1.92.9 at time of writing) implements the **Immediate Mode GUI** (IMGUI) paradigm. The core contract is stated at the top of `imgui.cpp`:

> "Your code creates the UI every frame of your application loop; if your code doesn't run, the UI is gone."

Every frame, the application submits all widgets it wants to see, and the library generates a draw list from them. No widget objects are allocated; no callbacks are registered. The typical per-frame sequence is:

```cpp
// Platform + renderer backends together constitute one "frame tick"
ImGui_ImplVulkan_NewFrame();
ImGui_ImplGlfw_NewFrame();
ImGui::NewFrame();                        // consume pending input, reset frame state

ImGui::Begin("Scene Stats");
ImGui::Text("Draw calls: %d", stats.drawCalls);
ImGui::SliderFloat("LOD bias", &lod, 0.0f, 4.0f);
if (ImGui::Button("Reload Shaders"))
    scheduler.Post(Task::ReloadShaders);
ImGui::End();

ImGui::Render();                          // finalise draw lists
ImDrawData* draw_data = ImGui::GetDrawData();
ImGui_ImplVulkan_RenderDrawData(draw_data, cmd); // backend renders
```

The application data (`stats.drawCalls`, `lod`) is the single source of truth. ImGui reads from and writes to application memory directly; there is no synchronisation layer, no model class, and no change notification needed. [Source](https://github.com/ocornut/imgui)

### 1.2 Why Tools, Not Consumer UI

Dear ImGui explicitly targets **developer tools, debug overlays, game editors, and profilers** — not consumer-facing polished UIs. The FAQ explains:

- Layout features are limited to stack/table placement; complex alignment and resizing are harder to express than in a retained toolkit.
- Skinning is intentionally minimal compared to CSS-driven systems.
- The draw-everything-every-frame model is well-suited to constantly changing data (profiler values, entity positions) but wasteful for static UIs where a retained toolkit can skip redraws.

Widely deployed users of Dear ImGui for tools include in-house game-engine editors, RenderDoc's GPU debugger, the Tracy profiler viewer, and hardware debug UIs on embedded targets where Qt and GTK are unavailable. [Source](https://github.com/ocornut/imgui/blob/master/docs/FAQ.md)

### 1.3 Comparison with Retained-Mode Toolkits

| Aspect | Dear ImGui | Qt6 (Ch39a) / GTK4 (Ch39c) |
|---|---|---|
| Widget identity | Label hash (ID stack) | Object pointer / widget tree |
| Data binding | Direct `float*` / `int*` | Signals/slots, model-view, `QProperty` |
| Layout | Code-driven every frame | Declarative, cached, CSS-styled |
| Dynamic list of N items | `for` loop, `PushID(i)` | Subclass `QAbstractListModel` |
| Invisible widget | Not submitted → gone | `setVisible(false)` on persistent object |
| State bugs | Widget disappears silently | Stale display from missed notify |
| Styling | `PushStyleColor/Var` | Theme engines, Qt QSS, GTK CSS |
| Integration cost | ~25 lines + copy 9 files | Build system dependency, event loop |
| Consumer application | Limited | Full-featured |

For a debug overlay drawn on top of an existing Vulkan renderer, ImGui's integration cost is orders of magnitude lower than introducing Qt or GTK.

---

## 2. Core Architecture

### 2.1 ImGuiContext

`ImGuiContext` (exposed fully via `imgui_internal.h`) is the centralised store for all frame state: the active window stack, widget hover/active state, the ID stack, clipboard, navigation, draw list shared data, style, `ImGuiIO`, font atlas pointer, and the frame's draw data. It is created by `ImGui::CreateContext()` and retrieved via `ImGui::GetCurrentContext()`. Multiple contexts are supported, which is useful across DLL boundaries or when embedding two independent ImGui UIs in the same process. [Source](https://github.com/ocornut/imgui/blob/master/imgui_internal.h)

### 2.2 The Draw List Pipeline

All widget rendering builds vertex/index geometry into `ImDrawList` objects, one per ImGui window plus the background and foreground overlay lists. After `ImGui::Render()`, `ImGui::GetDrawData()` returns an `ImDrawData*` referencing all draw lists for the frame.

```c
// imgui.h — draw data root
struct ImDrawData
{
    bool                    Valid;           // true after Render(), false after next NewFrame()
    int                     TotalIdxCount;
    int                     TotalVtxCount;
    ImVector<ImDrawList*>   CmdLists;        // one per ImGui window
    ImVec2                  DisplayPos;      // top-left of viewport (usually 0,0)
    ImVec2                  DisplaySize;     // viewport size
    ImVec2                  FramebufferScale; // e.g. (2,2) on Retina
    ImGuiViewport*          OwnerViewport;
    ImVector<ImTextureData*>* Textures;      // pending texture create/update/destroy (1.92.0+)
};
```

Each `ImDrawList` holds parallel arrays of commands, indices, and vertices:

```c
struct ImDrawList
{
    ImVector<ImDrawCmd>   CmdBuffer;   // one cmd per texture change or clip rect change
    ImVector<ImDrawIdx>   IdxBuffer;   // triangle indices (uint16 by default)
    ImVector<ImDrawVert>  VtxBuffer;   // interleaved pos/uv/col
    // ...
};
```

Within a draw list, consecutive draw calls that share the same texture and clip rectangle are merged into a single `ImDrawCmd` covering a contiguous range of the index buffer:

```c
struct ImDrawCmd
{
    ImVec4       ClipRect;    // scissor rectangle
    ImTextureRef TexRef;      // texture reference (since 1.92.0; replaces raw ImTextureID field)
    unsigned int VtxOffset;   // base vertex (enables >64k vertices with uint16 indices)
    unsigned int IdxOffset;   // start of index range
    unsigned int ElemCount;   // number of indices (ElemCount/3 = triangle count)
    ImDrawCallback UserCallback; // if set, call instead of issuing draw command
    void*        UserCallbackData;
};
```

The backend iterates `CmdLists`, uploads vertex/index data to GPU buffers, then iterates `CmdBuffer` inside each list: set scissor to `ClipRect`, bind the texture indicated by `TexRef.GetTexID()`, and issue a `vkCmdDrawIndexed` / `glDrawElementsBaseVertex` for `ElemCount` indices.

**Note**: In v1.92.0 the `TextureId` field was removed from `ImDrawCmd`. Code that accessed `cmd.TextureId` directly must switch to `cmd.GetTexID()`. [Source](https://github.com/ocornut/imgui/blob/master/docs/CHANGELOG.txt)

### 2.3 Vertex and Index Format

The default vertex format is 20 bytes:

```c
// imgui.h
struct ImDrawVert
{
    ImVec2  pos;   // 8 bytes — screen-space position
    ImVec2  uv;    // 8 bytes — texture coordinates
    ImU32   col;   // 4 bytes — RGBA packed (0xAABBGGRR)
};
```

The layout can be replaced by defining `IMGUI_OVERRIDE_DRAWVERT_STRUCT_LAYOUT` in `imconfig.h`; the comment specifies the ordering constraint: pos first (8 bytes), uv second (8 bytes), col third (4 bytes). Engine integrations that share a vertex buffer with their own geometry sometimes override this to match their native vertex layout.

Index type is `ImDrawIdx = unsigned short` (uint16) by default. For meshes exceeding 65 536 vertices, either redefine `ImDrawIdx` as `unsigned int`, or use the `VtxOffset` field in `ImDrawCmd` (supported by backends that set `ImGuiBackendFlags_RendererHasVtxOffset`) to index into a window of the vertex buffer starting at a 16-bit-range boundary. [Source](https://github.com/ocornut/imgui/blob/master/imgui.h)

---

## 3. Font Atlas

### 3.1 Rasterisation: stb_truetype vs FreeType

The default font rasteriser is **stb_truetype** (`imstb_truetype.h`), bundled directly in the repository — zero external dependencies for basic text. It converts TTF/OTF outlines to bitmaps using a custom scanline rasteriser. Quality is adequate for small sizes in developer tools.

An optional higher-quality path activates the **FreeType** rasteriser by:

1. Enabling `IMGUI_ENABLE_FREETYPE` in `imconfig.h`.
2. Adding `misc/freetype/imgui_freetype.cpp` to the build.
3. Calling `ImGuiFreeType::SetDefaultAllocatorFunctions()` and activating FreeType in the atlas flags: `atlas->FontBuilderIO = ImGuiFreeType::GetBuilderForFreeType()`.

FreeType provides superior hinting, subpixel rendering, and — when combined with plutosvg or lunasvg via `IMGUI_ENABLE_FREETYPE_PLUTOSVG` — SVG colour emoji glyph support. On Linux desktop applications with system fonts, FreeType output more closely matches the rendering in GTK4 and Qt applications. [Source](https://github.com/ocornut/imgui/blob/master/misc/freetype/imgui_freetype.h)

### 3.2 Texture Packing and Upload

`ImFontAtlas` accumulates glyph requests from all loaded fonts, then calls `Build()` to:

1. Rasterise each requested glyph to a greyscale bitmap via stb_truetype or FreeType.
2. Pack bitmaps into a rectangular texture atlas using **stb_rect_pack** (`imstb_rectpack.h`). The default atlas size grows up to `TexMaxWidth` × `TexMaxHeight` (both default to 8192).
3. Insert a 1×1 white pixel at a known UV coordinate (`TexUvWhitePixel`) — used for all solid-colour draws.
4. Produce an 8-bit alpha texture (`GetTexDataAsAlpha8()`) or a 32-bit RGBA texture (`GetTexDataAsRGBA32()`).

In the pre-1.92 upload model, the application called `Build()`, uploaded the atlas texture to GPU, and called `SetTexID(gpu_handle)`. The backend referenced that handle for all text draws. This model is still supported but considered legacy.

### 3.3 Dynamic Atlas (1.92.0+)

Version 1.92.0 introduced incremental font atlas management and **dynamic font sizes** (`PushFont(font, size)` now takes an explicit size argument; `ImFont*` is no longer single-sized). The key new types are `ImTextureData` and `ImTextureRef`.

Backends that set `ImGuiBackendFlags_RendererHasTextures` monitor `ImDrawData::Textures[]` each frame: it is a vector of `ImTextureData*` pointers, each describing a pending GPU operation:

```c
for (ImTextureData* tex : *draw_data->Textures) {
    if (tex->Status == ImTextureStatus_WantCreate) {
        // upload tex->GetPixels() (RGBA32) to GPU, store handle in tex->SetTexID()
    } else if (tex->Status == ImTextureStatus_WantUpdates) {
        // partial update: iterate tex->Updates[] for dirty rects
    } else if (tex->Status == ImTextureStatus_WantDestroy) {
        // destroy GPU texture, call tex->SetTexID(ImTextureID_Invalid)
    }
}
```

This enables incremental glyph loading (e.g., CJK glyphs on first use), hot font reloading, and multi-size font rendering without full atlas rebuilds. The Vulkan backend (v1.92.1+) implements this path fully. [Source](https://github.com/ocornut/imgui/blob/master/docs/CHANGELOG.txt)

---

## 4. The ID System

### 4.1 Hash Chain

Every widget is identified by an `ImGuiID` — a `uint32_t` CRC32 hash. The hash is computed over the **entire ID stack** concatenated with the widget's label string. The ID stack is per-window (`ImGuiWindow::IDStack`, a `ImVector<ImGuiID>` in `imgui_internal.h`), and each call to `PushID` appends to it using the previous top as a seed:

```c
// imgui_internal.h
ImGuiID ImHashStr(const char* data, size_t size = 0, ImGuiID seed = 0);
ImGuiID ImHashData(const void* data, size_t size, ImGuiID seed = 0);
```

The seed for each hash operation is the current stack top, forming a hash chain. Two widgets in different scopes can share the same label string without collision, as long as their enclosing `PushID` scopes differ.

ImGui tracks which `ImGuiID` is hovered, active, or focused. Two widgets with the same ID will fight over this state — the most common pitfall for new users. [Source](https://github.com/ocornut/imgui/blob/master/imgui_internal.h)

### 4.2 PushID / PopID

```c
// imgui.h
void PushID(const char* str_id);           // push string onto ID stack
void PushID(const char* begin, const char* end); // push substring
void PushID(const void* ptr_id);           // push pointer (address as seed)
void PushID(int int_id);                   // push integer
void PopID();
ImGuiID GetID(const char* str_id);         // peek: what ID would str_id produce now?
```

The canonical pattern for a dynamic list of N widgets:

```c
for (int i = 0; i < (int)entities.size(); i++) {
    ImGui::PushID(i);
    ImGui::Button("Delete");     // each "Delete" button has a unique ID
    ImGui::SameLine();
    ImGui::Text("%s", entities[i].name.c_str());
    ImGui::PopID();
}
```

Without `PushID`, every "Delete" button hashes to the same ID, causing the hover/active state to bleed between buttons. [Source](https://github.com/ocornut/imgui/blob/master/docs/FAQ.md)

### 4.3 `##` and `###` Label Syntax

The `##` separator within a label string splits the **visible text** from the **ID contribution**:

```c
ImGui::Button("Play##player1");  // visible: "Play", ID hashed from "Play##player1"
ImGui::Button("Play##player2");  // visible: "Play", different ID
```

The `###` separator causes the hash to be computed **only from the suffix**, discarding the prefix:

```c
ImGui::Button("Frame 1###my_button"); // visible: "Frame 1", ID from "my_button"
ImGui::Button("Frame 2###my_button"); // visible: "Frame 2", SAME ID
```

The `###` form is used for widgets with dynamic labels (e.g., a window whose title updates with a frame counter) that must maintain stable state across label changes. From `imgui.cpp`: "`GetID("Hello###World")` now produces identical results to `GetID("World")`." [Source](https://github.com/ocornut/imgui/blob/master/imgui.cpp)

---

## 5. Backend Architecture

### 5.1 Platform vs Renderer Split

The split into orthogonal **Platform** and **Renderer** backends was established in v1.62 (June 2018), motivated by the then-forthcoming multi-viewport work, which required each backend type to grow significant new responsibilities:

- **Platform backends** handle windowing and input: window creation, `ImGuiKey` mapping from OS events, mouse cursor shape, clipboard, IME positioning, timing. They write to `ImGuiIO` and set `io.BackendPlatformName`, `io.BackendFlags`, and `io.BackendPlatformUserData`.
- **Renderer backends** handle GPU resources: font atlas texture creation, vertex/index buffer upload, and issuing draw calls from `ImDrawData`. They set `io.BackendRendererName`, `io.BackendFlags`, and `io.BackendRendererUserData`.

Any platform backend can be combined with any renderer backend. On Linux the typical pairings are `SDL3 + Vulkan`, `GLFW + Vulkan`, `SDL3 + OpenGL3`, and `GLFW + OpenGL3`. [Source](https://github.com/ocornut/imgui/blob/master/docs/BACKENDS.md)

### 5.2 Linux Backend Inventory

**Platform backends relevant to Linux** (in `backends/`):

| File | Library | Notes |
|---|---|---|
| `imgui_impl_sdl2.cpp` | SDL 2.x | Legacy; SDL3 preferred |
| `imgui_impl_sdl3.cpp` | SDL 3.x | Wayland-aware via `SDL_GetCurrentVideoDriver()` |
| `imgui_impl_glfw.cpp` | GLFW 3.x | Wayland-aware via `ImGui_ImplGlfw_IsWayland()` |
| `imgui_impl_glut.cpp` | FreeGLUT | Legacy, avoid for new code |

**Renderer backends relevant to Linux** (in `backends/`):

| File | API | Notes |
|---|---|---|
| `imgui_impl_vulkan.cpp` | Vulkan | Preferred for GPU-rendered applications |
| `imgui_impl_opengl3.cpp` | OpenGL 3.2+ / ES2 / ES3 | Broadest compatibility; also covers WebGL |
| `imgui_impl_opengl2.cpp` | OpenGL 2.x | Legacy; only for pre-3.0 contexts |
| `imgui_impl_wgpu.cpp` | WebGPU (Dawn/wgpu) | Via `imgui_impl_sdl3.cpp` on desktop |
| `imgui_impl_sdlgpu3.cpp` | SDL_GPU (SDL3) | SDL3's modern GPU abstraction |
| `imgui_impl_null.cpp` | — | Headless testing |

**No native Wayland backend exists.** Wayland is accessed exclusively through SDL3 or GLFW, which manage `wl_surface`, `wl_keyboard`, and `wl_data_device` internally. [Source](https://github.com/ocornut/imgui/blob/master/docs/BACKENDS.md)

---

## 6. The Vulkan Backend

### 6.1 Initialization

The Vulkan backend (`backends/imgui_impl_vulkan.cpp`) was first added in v1.48 (April 2016, PR #549) as a combined GLFW+Vulkan example, split into a standalone backend in v1.62 (June 2018). Current version: 1.92.9.

Initialization requires filling `ImGui_ImplVulkan_InitInfo` and passing it to `ImGui_ImplVulkan_Init()`:

```cpp
ImGui_ImplVulkan_InitInfo init_info = {};
init_info.ApiVersion         = VK_API_VERSION_1_3;
init_info.Instance           = instance;
init_info.PhysicalDevice     = physicalDevice;
init_info.Device             = device;
init_info.QueueFamily        = graphicsQueueFamily;
init_info.Queue              = graphicsQueue;
init_info.DescriptorPool     = descriptorPool; // or set DescriptorPoolSize
init_info.MinImageCount      = 2;
init_info.ImageCount         = swapchainImageCount;
init_info.PipelineCache      = pipelineCache;  // optional

// Render pass path:
init_info.PipelineInfoMain.RenderPass = renderPass;

// OR dynamic rendering path (see §6.2):
init_info.UseDynamicRendering = true;
init_info.PipelineInfoMain.PipelineRenderingCreateInfo = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_RENDERING_CREATE_INFO,
    .colorAttachmentCount = 1,
    .pColorAttachmentFormats = &swapchainFormat,
};

init_info.Allocator          = nullptr;        // optional
init_info.CheckVkResultFn    = check_vk;       // optional error handler

ImGui_ImplVulkan_Init(&init_info);
```

The backend creates its own graphics pipeline (vertex + fragment shaders compiled from embedded SPIR-V bytecode), a sampler, and descriptor set layouts. [Source](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_vulkan.h)

### 6.2 Dynamic Rendering Path

Vulkan 1.3 `VK_KHR_dynamic_rendering` support was added in v1.89.7 (July 2023, PR #5446). When `UseDynamicRendering = true`:

- `PipelineInfoMain.RenderPass` must be `VK_NULL_HANDLE`.
- `PipelineInfoMain.PipelineRenderingCreateInfo` must specify color attachment formats.
- The pipeline is created with a `VkPipelineRenderingCreateInfo` chain on `VkGraphicsPipelineCreateInfo`.
- At render time, the backend calls `vkCmdBeginRendering` / `vkCmdEndRendering` instead of `vkCmdBeginRenderPass` / `vkCmdEndRenderPass`.
- The backend loads these functions first as Vulkan 1.3 core, then falls back to KHR extension variants via `vkGetDeviceProcAddr`.

The caller must place `ImGui_ImplVulkan_RenderDrawData()` between `vkCmdBeginRendering` and `vkCmdEndRendering` when using the dynamic rendering path. [Source](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_vulkan.cpp)

Optional **volk** support (v1.92.4+): define `IMGUI_IMPL_VULKAN_USE_VOLK` to let the backend use volk's pre-loaded function pointers instead of the Vulkan loader trampoline. [Source](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_vulkan.h)

### 6.3 Vertex/Index Buffer Upload

The backend allocates one pair of device-local, host-visible vertex and index buffers per swapchain image index (a rotating set of `ImageCount` pairs). Each frame:

1. `vkMapMemory()` on both buffers.
2. Copy each `ImDrawList::VtxBuffer` / `ImDrawList::IdxBuffer` sequentially.
3. Flush via `VkMappedMemoryRange`, aligning size to `VkPhysicalDeviceLimits::nonCoherentAtomSize` using `(size + alignment - 1) & ~(alignment - 1)`.
4. `vkUnmapMemory()`.

Buffers grow to accommodate larger frames but never shrink within a session. Resize is implemented by destroying and recreating with `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT` when `MinAllocationSize` (configurable in `InitInfo`) is exceeded.

The vertex and index buffers are submitted together with the draw command through `VkBuffer` bind calls before each draw list's `vkCmdDrawIndexed`. [Source](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_vulkan.cpp)

### 6.4 Descriptor Management

As of v1.92.8 (May 2026), the backend switched from `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER` to separate `VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE` + `VK_DESCRIPTOR_TYPE_SAMPLER` descriptors, enabling runtime sampler changes via `ImDrawCallback_SetSamplerLinear` and `ImDrawCallback_SetSamplerNearest`.

The descriptor pool provided by the caller (or created internally when `DescriptorPoolSize` is set instead of `DescriptorPool`) must include:
- At least `IMGUI_IMPL_VULKAN_MINIMUM_SAMPLED_IMAGE_POOL_SIZE` (= 8) descriptors of type `VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE`.
- At least `IMGUI_IMPL_VULKAN_MINIMUM_SAMPLER_POOL_SIZE` (= 2) descriptors of type `VK_DESCRIPTOR_TYPE_SAMPLER`.

The fragment shader logic (expressed as GLSL, compiled to embedded SPIR-V):

```glsl
layout(set=0, binding=0) uniform sampler  _Sampler;
layout(set=0, binding=1) uniform texture2D _Texture;

void main() {
    out_Color = frag_Color * texture(sampler2D(_Texture, _Sampler), frag_UV);
}
```

Custom shaders can be injected via `InitInfo.CustomShaderVertCreateInfo` / `CustomShaderFragCreateInfo` (available since v1.92.4). [Source](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_vulkan.cpp)

### 6.5 Custom Textures

To render application textures (e.g., a game viewport rendered to an offscreen image) inside an ImGui window, register the image with the backend:

```cpp
// Add a texture (returns a VkDescriptorSet cast to ImTextureID):
ImTextureID tex_id = ImGui_ImplVulkan_AddTexture(image_view, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);

// In widget code:
ImGui::Image(tex_id, ImVec2(width, height));

// Cleanup:
ImGui_ImplVulkan_RemoveTexture((VkDescriptorSet)tex_id);
```

`ImGui_ImplVulkan_AddTexture()` allocates a descriptor set from the pool and writes the image view into it. The descriptor set pointer becomes the `ImTextureID` used by `ImDrawCmd::GetTexID()` at render time. [Source](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_vulkan.h)

---

## 7. The OpenGL 3 Backend

`imgui_impl_opengl3.cpp` targets OpenGL 3.2 core, OpenGL ES 2.0, OpenGL ES 3.0, and WebGL 2.0. It auto-detects the version from GLSL `#version` and `GL_VERSION`, selecting the appropriate shader variant.

The backend embeds GLSL shaders as string constants:

```glsl
// Vertex (OpenGL 3.2+)
layout (location = 0) in vec2  Position;
layout (location = 1) in vec2  UV;
layout (location = 2) in vec4  Color;
uniform mat4 ProjMtx;
out vec2 Frag_UV;
out vec4 Frag_Color;
void main() {
    Frag_UV    = UV;
    Frag_Color = Color;
    gl_Position = ProjMtx * vec4(Position.xy, 0, 1);
}
```

A per-frame `glBufferData(GL_ARRAY_BUFFER, ...)` / `glBufferData(GL_ELEMENT_ARRAY_BUFFER, ...)` uploads vertex/index data. `glScissor()` sets the clip rectangle from `ImDrawCmd::ClipRect`. The backend saves and restores ~20 GL state variables (blend mode, scissor, cull face, VAO, bound textures, etc.) around the draw calls, making it safe to insert into an existing GL render loop without restructuring GL state management. [Source](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_opengl3.cpp)

`imgui_impl_opengl3_loader.h` is a bundled minimal OpenGL function loader, eliminating the dependency on GLEW or GLAD for the backend itself (though applications may still use those for their own code).

---

## 8. Platform Backends on Linux

### 8.1 SDL3 Backend and Wayland

`imgui_impl_sdl3.cpp` detects the active video driver at runtime:

```c
const char* sdl_video_driver = SDL_GetCurrentVideoDriver();
bd->IsWayland = (strcmp(sdl_video_driver, "wayland") == 0);
```

On Wayland, the backend disables global mouse capture (`SDL_CaptureMouse`), which is disallowed by Wayland's security model. Global pointer grabbing — used on X11 to track drag operations that leave the window boundary — is not available. Drag operations that exit the application window may not track correctly.

Clipboard is handled via `SDL_GetClipboardText()` / `SDL_SetClipboardText()`, which SDL3 maps to the `wl_data_device` protocol. Cursor shapes use `SDL_CreateSystemCursor()`, internally backed by `wl_cursor` / `wp_cursor_shape_v1`. IME text input uses `SDL_StartTextInput()` / `SDL_StopTextInput()` mapped to `zwp_text_input_v3`. HiDPI scale is available via `SDL_GetWindowDisplayScale()`.

The backend installs SDL event handlers via `SDL_EVENT_*` polling (no `SDL_SetEventFilter`), converting SDL events to `ImGuiKey` codes and writing them to `ImGuiIO`. [Source](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_sdl3.cpp)

### 8.2 GLFW Backend and Wayland

`imgui_impl_glfw.cpp` exposes `ImGui_ImplGlfw_IsWayland()` for runtime detection. Compile-time macros `IMGUI_IMPL_GLFW_DISABLE_X11` and `IMGUI_IMPL_GLFW_DISABLE_WAYLAND` allow restricting backend code paths.

Known Wayland limitation: `glfwGetMonitorContentScale()` may return 1.0 on some Wayland compositor configurations, causing incorrect DPI assumptions. Applications should query `glfwGetWindowContentScale()` per-window for correct fractional scale. Clipboard is provided by GLFW's `glfwSetClipboardString()` (GLFW 3.3+ window-agnostic form), backed by `wl_data_device_manager`. Cursor shapes use `glfwCreateStandardCursor()` backed by `libwayland-cursor`.

The backend installs GLFW callbacks (`glfwSetMouseButtonCallback`, `glfwSetScrollCallback`, `glfwSetKeyCallback`, `glfwSetCharCallback`, `glfwSetCursorPosCallback`, `glfwSetCursorEnterCallback`, `glfwSetWindowFocusCallback`, `glfwSetMonitorCallback`) and chains previously installed callbacks. [Source](https://github.com/ocornut/imgui/blob/master/backends/imgui_impl_glfw.cpp)

---

## 9. Build and Integration

### 9.1 Copy-Source Approach

The upstream documentation explicitly recommends **against** building Dear ImGui as a static or shared library:

> "It is recommended that you follow those steps and not attempt to build Dear ImGui as a static or shared library!"

The rationale: ImGui is call-heavy; shared library overhead is significant; DLL users would also need cross-DLL `ImGui::SetCurrentContext()` and `ImGui::SetAllocatorFunctions()` calls. The canonical approach is to copy the required files directly into the project source tree and compile them as part of the application.

**Minimum files required** for a Vulkan + GLFW application:

```
imgui.h           imgui_internal.h  imconfig.h
imgui.cpp         imgui_draw.cpp    imgui_tables.cpp   imgui_widgets.cpp
imstb_rectpack.h  imstb_textedit.h  imstb_truetype.h
backends/imgui_impl_glfw.cpp   backends/imgui_impl_glfw.h
backends/imgui_impl_vulkan.cpp backends/imgui_impl_vulkan.h
```

`imgui_demo.cpp` is optional but strongly recommended during development — it contains the `ImGui::ShowDemoWindow()` implementation, which exercises the entire widget API.

Place `imgui/` and `imgui/backends/` on the include path. The `IMGUI_CHECKVERSION()` macro (call from one `.cpp` file) verifies struct layout binary compatibility between translation units and catches mismatched version builds. [Source](https://github.com/ocornut/imgui/wiki/Getting-Started)

### 9.2 CMake Patterns

There is no official `CMakeLists.txt` in the ImGui repository root. Three common approaches:

**1. Direct source inclusion** (simplest, matches the recommended copy-source workflow):

```cmake
add_executable(my_app
    src/main.cpp
    vendor/imgui/imgui.cpp
    vendor/imgui/imgui_draw.cpp
    vendor/imgui/imgui_tables.cpp
    vendor/imgui/imgui_widgets.cpp
    vendor/imgui/backends/imgui_impl_glfw.cpp
    vendor/imgui/backends/imgui_impl_vulkan.cpp
)
target_include_directories(my_app PRIVATE vendor/imgui vendor/imgui/backends)
target_link_libraries(my_app PRIVATE glfw Vulkan::Vulkan)
```

**2. FetchContent** (pins a tag, keeps the source tree clean):

```cmake
include(FetchContent)
FetchContent_Declare(imgui
    GIT_REPOSITORY https://github.com/ocornut/imgui
    GIT_TAG        v1.92.9
)
FetchContent_MakeAvailable(imgui)

# Build a static lib from the fetched sources:
add_library(imgui STATIC
    ${imgui_SOURCE_DIR}/imgui.cpp
    ${imgui_SOURCE_DIR}/imgui_draw.cpp
    ${imgui_SOURCE_DIR}/imgui_tables.cpp
    ${imgui_SOURCE_DIR}/imgui_widgets.cpp
    ${imgui_SOURCE_DIR}/backends/imgui_impl_glfw.cpp
    ${imgui_SOURCE_DIR}/backends/imgui_impl_vulkan.cpp
)
target_include_directories(imgui PUBLIC
    ${imgui_SOURCE_DIR} ${imgui_SOURCE_DIR}/backends)
target_link_libraries(imgui PUBLIC glfw Vulkan::Vulkan)
```

**3. vcpkg**: `vcpkg install imgui[glfw-binding,opengl3-binding,vulkan-binding]` generates CMake integration targets. vcpkg's port wraps the sources into an `imgui::imgui` target with feature-selected backends.

### 9.3 Compile-Time Configuration

All compile-time knobs live in `imconfig.h` (or a file named by `-DIMGUI_USER_CONFIG="my_config.h"`):

```c
// imconfig.h — key options
//#define IMGUI_DISABLE_OBSOLETE_FUNCTIONS    // catch API breaks early
//#define IMGUI_ENABLE_FREETYPE               // FreeType font rasteriser
//#define IMGUI_ENABLE_FREETYPE_PLUTOSVG      // SVG colour emoji (needs plutosvg)
//#define IMGUI_USE_BGRA_PACKED_COLOR         // BGRA vertex color (some backends)
//#define IMGUI_USE_WCHAR32                   // 32-bit Unicode (emoji, CJK planes 2+)
//#define IMGUI_OVERRIDE_DRAWVERT_STRUCT_LAYOUT // custom vertex format
//#define ImDrawIdx unsigned int              // 32-bit indices instead of uint16
//#define IMGUI_DISABLE_DEFAULT_FONT          // strip embedded fonts (~23 KB)
//#define IMGUI_DISABLE_SSE                   // disable SSE2/SSE4 optimisations
```

All translation units that include `imgui.h` must agree on these defines; mixing configurations within a binary produces undefined behaviour. [Source](https://github.com/ocornut/imgui/blob/master/imconfig.h)

---

## 10. The Docking Branch

**Docking and multi-viewport are not in the `master` branch as of 2026-07-25.** They live in a separate `docking` branch at `https://github.com/ocornut/imgui/tree/docking`. The master branch `imgui.h` contains no `ImGuiConfigFlags_DockingEnable` or `ImGuiConfigFlags_ViewportsEnable` entries.

**Docking** (`docking` branch): windows can be dragged onto each other to form tabbed groups or split panes. The host window calls `ImGui::DockSpaceOverViewport()` or `ImGui::DockSpace(id, size, flags)` to designate a docking area. Internally, docking creates `ImGuiDockNode` objects that manage the split layout tree.

**Multi-viewport** (`docking` branch, `ImGuiConfigFlags_ViewportsEnable`): when a user drags an ImGui window outside the application's main OS window, the platform backend creates a new OS-level child window for it via `ImGuiPlatformIO` callbacks:

```c
// Backend must implement these callbacks in the docking branch:
io.PlatformIO.Platform_CreateWindow  = MyPlatform_CreateWindow;
io.PlatformIO.Platform_DestroyWindow = MyPlatform_DestroyWindow;
io.PlatformIO.Platform_RenderWindow  = MyPlatform_RenderWindow;
io.PlatformIO.Platform_SwapBuffers   = MyPlatform_SwapBuffers;
// etc.
```

After the main `Render()` / `RenderDrawData()`, the application calls `ImGui::UpdatePlatformWindows()` + `ImGui::RenderPlatformWindowsDefault()` to drive rendering into all secondary OS windows.

On Wayland, secondary `xdg_toplevel` windows created by multi-viewport work in principle with SDL3 and GLFW, but Wayland clients cannot set absolute window positions (a known mismatch with the multi-viewport API that assumes coordinate control). Multi-viewport window positioning on Wayland may behave differently from X11.

---

## 11. Ecosystem and Extensions

### 11.1 ImPlot

ImPlot ([https://github.com/epezent/implot](https://github.com/epezent/implot)) is an immediate-mode GPU-accelerated plotting library for Dear ImGui. It follows the same paradigm:

```cpp
ImPlot::CreateContext();  // once at startup, alongside ImGui::CreateContext()

if (ImPlot::BeginPlot("Frame Time")) {
    ImPlot::PlotLine("CPU", times.data(), cpu_ms.data(), times.size());
    ImPlot::PlotLine("GPU", times.data(), gpu_ms.data(), times.size());
    ImPlot::EndPlot();
}
```

Chart types: line, scatter, shaded, bar (vertical/horizontal/stacked), error bars, stem, stair, pie, heatmap, 1D/2D histogram, digital (signal traces), image overlays. Multi-axis: up to 3 x-axes and 3 y-axes per plot. Handles hundreds of thousands of data points without issue. Time series with microsecond precision, ISO 8601 formatting. 16 built-in colormaps; supports float, double, and all integer widths.

Distribution: four source files added to the project alongside the ImGui files. No additional dependencies.

### 11.2 ImNodes / imgui-node-editor

**imnodes** ([https://github.com/Nelarius/imnodes](https://github.com/Nelarius/imnodes)): a minimal immediate-mode node graph editor. Nodes and links are submitted each frame:

```cpp
ImNodes::CreateContext();  // once at startup
ImNodes::BeginNodeEditor();
  ImNodes::BeginNode(node_id);
    ImNodes::BeginOutputAttribute(attr_id);
    ImGui::Text("Output");
    ImNodes::EndOutputAttribute();
  ImNodes::EndNode();
ImNodes::EndNodeEditor();

int start_attr, end_attr;
if (ImNodes::IsLinkCreated(&start_attr, &end_attr))
    graph.AddLink(start_attr, end_attr);
```

Features: link creation detection, selection, box-select, mini-map overlay, per-node colour styling. Distribution: three files.

**imgui-node-editor** ([https://github.com/thedmd/imgui-node-editor](https://github.com/thedmd/imgui-node-editor)): a richer node graph editor styled after Unreal Engine 4 blueprints. Features Bézier curve links, auto-highlight, group dragging, context menus, cut/copy/paste, and persistent node layout serialisation via a user-supplied context. More complex API than imnodes; appropriate when the target look is an Unreal-style blueprint canvas.

### 11.3 ImGuizmo

ImGuizmo ([https://github.com/CedricGuillemet/ImGuizmo](https://github.com/CedricGuillemet/ImGuizmo)) provides immediate-mode 3D transform gizmos operating on 4×4 `float` matrices:

```cpp
float view[16], projection[16], model[16];
// ... fill matrices from scene ...
ImGuizmo::SetOrthographic(false);
ImGuizmo::SetDrawlist();
ImGuizmo::Manipulate(view, projection,
    ImGuizmo::TRANSLATE, ImGuizmo::LOCAL, model);
// model[] updated in place if user dragged a handle
```

Operations: translate, rotate, scale with visual axis handles in a 3D viewport. Additional widgets:

- `ImViewGizmo` (orientation cube, one line of code)
- `ImSequencer` (timeline / keyframe editor)
- `GraphEditor` (node graph with fully custom rendering)
- `ImVectorEditor` (2D path / curve editing)

### 11.4 imgui_club

`imgui_club` ([https://github.com/ocornut/imgui_club](https://github.com/ocornut/imgui_club)) is the official repository for small Dear ImGui extensions:

- **`imgui_memory_editor`**: A hex editor widget with keyboard navigation, read-only mode, ASCII/HexII side display, goto-address, range highlighting, and pluggable read/write handlers. Call `MemoryEditor::DrawWindow(label, data, size)`.
- **`imgui_multicontext_compositor`**: Manages multiple simultaneous `ImGuiContext` instances with z-order and cross-context drag-and-drop.
- **`imgui_threaded_rendering`**: Captures `ImDrawData` snapshots for deferred multi-threaded rendering (isolates the render thread from the UI thread).

### 11.5 Tracy Profiler

Tracy ([https://github.com/wolfpld/tracy](https://github.com/wolfpld/tracy)) is a real-time, nanosecond-resolution frame and sampling profiler with GPU support (Vulkan, OpenGL, Direct3D 11/12, Metal, OpenCL, CUDA, WebGPU). The Tracy **viewer application** (`tracy/profiler/`) is itself an ImGui application — one of the most prominent examples of ImGui used as the UI framework for a complex developer tool rather than a simple overlay.

GPU timeline zones in a Vulkan application:

```cpp
// Application side (zero overhead in release builds):
TracyVkContext(physDev, dev, queue, cmdbuf);
{
    TracyVkZone(ctx, cmdbuf, "Shadow Pass");
    // ... shadow map render commands ...
}
TracyVkCollect(ctx, cmdbuf);
```

The profiling instrumentation in the application does not add an ImGui dependency; only the Tracy server viewer application uses ImGui. [Source](https://github.com/wolfpld/tracy)

---

## 12. Integrations

- **Ch18 (Mesa Vulkan — ANV, RADV, NVK)**: `ImGui_ImplVulkan_RenderDrawData()` submits draw calls through `vkCmdDrawIndexed` into the application's command buffer. The vertex and fragment shaders compiled from embedded SPIR-V bytecode enter Mesa's NIR compiler pipeline (Ch14) on ANV, RADV, and NVK. The sampled-image + sampler descriptor pair (since v1.92.8) interacts with Mesa's descriptor heap management.

- **Ch24 (EGL and Vulkan for application developers)**: When the `imgui_impl_opengl3` backend is used, it requires an active OpenGL context created via EGL (`eglCreateContext`) or GLX. The backend saves and restores GL state cleanly, making it composable with an EGL-based renderer.

- **Ch20 (Wayland protocols)**: ImGui has no direct `libwayland-client` dependency. Wayland protocol interactions (surface, keyboard, clipboard, cursor, IME) are handled entirely by SDL3 or GLFW, as described in §8. The `xdg_surface` / `xdg_toplevel` lifecycle for multi-viewport secondary windows (docking branch) is managed by the platform backend.

- **Ch82 (Vulkan Ecosystem Toolkit — VMA, volk, vk-bootstrap)**: ImGui is commonly integrated alongside vk-bootstrap (for `VkDevice` setup), VMA (for buffer allocation), and volk (via `IMGUI_IMPL_VULKAN_USE_VOLK`). The combination — vk-bootstrap + VMA + volk + imgui — reduces a minimal Vulkan application from ~2000 raw API lines to ~200 lines of application-specific code.

- **Ch39a (Qt6)**: For Vulkan/OpenGL applications that also embed Qt, ImGui and Qt can coexist in the same process with separate GL/Vulkan contexts; they do not share a toolkit event loop. ImGui draws directly into the renderer's command buffer; Qt uses QRhi or the QPA Vulkan backend. Mixing in the same surface requires careful synchronisation of rendering submissions and semaphores.

- **Ch39c (GTK4)**: Similar to Qt: ImGui and GTK do not share compositor integration. An ImGui overlay inside a GTK4 application requires a GtkGLArea (OpenGL3 backend) or a custom GtkWidget with a Vulkan swapchain.
