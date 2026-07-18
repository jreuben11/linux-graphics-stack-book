# Chapter 205: AI-Driven 3D Creation — Blender MCP, Claude Code, and Generative Tools

> **Part**: Part XI — Engines and Creative Tools
> **Audience**: Graphics application developers and technical artists who want to integrate AI language models and generative tools into Blender workflows; systems developers building headless Blender pipelines; tooling engineers embedding bpy in web services or CI pipelines
> **Status**: First draft — 2026-07-18

The emergence of the Model Context Protocol (MCP) has transformed how AI coding assistants interact with creative applications. Where previous integrations required manual copy-paste of Python snippets into Blender's script console, MCP enables a live bidirectional channel between Claude Code (or any MCP-capable agent) and a running Blender session: the AI can inspect the scene, execute operations, and observe results in a tight feedback loop. In parallel, a new generation of AI generative tools — text-to-3D, image-to-3D, AI texturing, AI rigging — has matured to the point where production-ready assets can be generated in seconds.

This chapter examines the full stack: the two-component Blender MCP bridge and its thread-safety architecture; how Claude Code reasons about Blender's Python API; the generative 3D ecosystem from Meshy to open-source models; AI-assisted denoising, texturing, and rigging within Blender; and how headless Blender fits into automated Linux pipelines.

---

## Table of Contents

- [1. The Model Context Protocol and Blender](#1-the-model-context-protocol-and-blender)
- [2. Blender MCP Architecture: The Two-Component Bridge](#2-blender-mcp-architecture-the-two-component-bridge)
- [3. MCP Tool Surface: What the AI Sees](#3-mcp-tool-surface-what-the-ai-sees)
- [4. Thread Safety and bpy.app.timers](#4-thread-safety-and-bpyapptimers)
- [5. Blender's Python API: The bpy Module](#5-blenders-python-api-the-bpy-module)
- [6. The RNA/DNA Bridge: How bpy Maps to C Internals](#6-the-rnadna-bridge-how-bpy-maps-to-c-internals)
- [7. Writing Blender Add-ons and Extensions](#7-writing-blender-add-ons-and-extensions)
- [8. Claude Code Workflows with bpy](#8-claude-code-workflows-with-bpy)
- [9. Meshy and the AI 3D Generation Ecosystem](#9-meshy-and-the-ai-3d-generation-ecosystem)
- [10. Open-Source Generative 3D Models](#10-open-source-generative-3d-models)
- [11. AI Denoising in Cycles: OIDN and OptiX](#11-ai-denoising-in-cycles-oidn-and-optix)
- [12. Dream Textures and AI Texturing](#12-dream-textures-and-ai-texturing)
- [13. AI Rigging and UV Automation](#13-ai-rigging-and-uv-automation)
- [14. Headless Blender and Pipeline Integration on Linux](#14-headless-blender-and-pipeline-integration-on-linux)
- [15. Practical Limits of AI-Driven Blender](#15-practical-limits-of-ai-driven-blender)
- [16. A Prompt Toolkit for Common Blender Operations](#16-a-prompt-toolkit-for-common-blender-operations)
- [17. Blender glTF Export for Three.js and React Three Fiber](#17-blender-gltf-export-for-threejs-and-react-three-fiber)
- [Integrations](#integrations)
- [References](#references)

---

## 1. The Model Context Protocol and Blender

MCP, published by Anthropic in late 2024 and now governed as an open specification at [modelcontextprotocol.io](https://modelcontextprotocol.io), defines a JSON-RPC-over-stdio (or SSE) wire format for connecting AI agents to *resources* and *tools* exposed by a server. A tool is a named function with a JSON Schema for its parameters; the agent calls it and receives structured output. Resources are read-only data blobs (files, database rows, live sensor readings) the agent can include in its context window.

For Blender, MCP addresses a fundamental ergonomic problem: Blender's entire UI — every menu item, slider, and button — is represented as a Python operator call. A capable agent with access to the operator reference can write correct bpy scripts, but without live scene feedback it cannot inspect what is actually present, verify results, or react to errors. MCP closes that loop.

**Why MCP over a simple subprocess pipe?** The MCP specification standardises tool discovery (the client asks the server for its tool list at connection time), error reporting (tools return structured `isError` responses), and sampling (the client can ask the server to run LLM completions). This standardisation means a single Blender MCP server works unmodified with Claude Desktop, Claude Code, Cursor, VS Code with Copilot MCP extensions, or any open-source MCP client.

---

## 2. Blender MCP Architecture: The Two-Component Bridge

Every Blender MCP implementation splits into two processes:

```
┌──────────────────┐  stdio/MCP  ┌─────────────────┐  TCP :9876  ┌──────────────────┐
│  MCP client      │◄───────────►│  MCP server     │◄───────────►│  Blender addon   │
│ (Claude Code,    │             │  server.py      │             │  addon.py        │
│  Claude Desktop, │             │  (FastMCP)      │             │  (bpy, TCP srv)  │
│  Cursor …)       │             └─────────────────┘             └──────────────────┘
└──────────────────┘
```

The split is forced by the fact that Blender embeds its own Python interpreter: you cannot `import bpy` from an external process. The addon runs *inside* Blender; the MCP server runs as a separate Python process that speaks the MCP protocol to the client.

### 2.1 The Addon Component

The community project at [github.com/ahujasid/blender-mcp](https://github.com/ahujasid/blender-mcp) (24,000+ stars as of mid-2026, MIT licence) provides the canonical addon. The core class is `BlenderMCPServer`:

```python
# addon.py (simplified)
import bpy, socket, threading, json, queue

class BlenderMCPServer:
    def __init__(self):
        self.host = "localhost"
        self.port = 9876
        self._server_thread = None
        self._sock = None
        self.command_queue = queue.Queue()

    def start(self):
        self._sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self._sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self._sock.bind((self.host, self.port))
        self._sock.listen(1)
        self._sock.settimeout(1.0)
        self._server_thread = threading.Thread(
            target=self._server_loop, daemon=True
        )
        self._server_thread.start()
        # Register main-thread pump (see §4)
        bpy.app.timers.register(self._process_queue, first_interval=0.005)

    def _server_loop(self):
        while not self._stop_event.is_set():
            try:
                conn, _ = self._sock.accept()
            except socket.timeout:
                continue
            self._handle_client(conn)

    def _handle_client(self, conn):
        # Read JSON command, enqueue for main-thread dispatch
        raw = self._recv_full(conn)
        cmd = json.loads(raw)
        result_holder = []
        evt = threading.Event()
        self.command_queue.put((cmd, result_holder, evt))
        evt.wait(timeout=180)
        conn.sendall(json.dumps(result_holder[0]).encode())
        conn.close()
```

The `_process_queue` timer (see §4) dequeues commands on Blender's main thread, executes them, stores the result, and sets the threading event so `_handle_client` can return the response.

The official Blender Foundation server at [projects.blender.org/lab/blender_mcp](https://projects.blender.org/lab/blender_mcp) (v1.0.0, 27 April 2026) uses an identical two-process design, is bundled as a Blender Extension rather than a classic addon, and targets Blender 5.1+.

### 2.2 The MCP Server Component

`server.py` is a separate process that speaks MCP to the AI client and TCP to the Blender addon:

```python
# server.py (simplified)
from mcp.server.fastmcp import FastMCP
import socket, json
from contextlib import asynccontextmanager
from dataclasses import dataclass

@dataclass
class BlenderConnection:
    host: str = "localhost"
    port: int = 9876
    _sock: socket.socket = None

    def connect(self):
        self._sock = socket.socket()
        self._sock.connect((self.host, self.port))
        self._sock.settimeout(180)

    def send_command(self, cmd: dict) -> dict:
        payload = json.dumps(cmd).encode()
        self._sock.sendall(payload)
        return self._recv_full()

    def _recv_full(self) -> dict:
        chunks = []
        while True:
            chunk = self._sock.recv(65536)
            if not chunk:
                break
            chunks.append(chunk)
            try:
                return json.loads(b"".join(chunks))
            except json.JSONDecodeError:
                continue  # incomplete; read more

@asynccontextmanager
async def server_lifespan(app):
    conn = BlenderConnection()
    conn.connect()
    yield {"blender": conn}

mcp = FastMCP("BlenderMCP", lifespan=server_lifespan)
```

Each tool is decorated with `@mcp.tool()`. The FastMCP library handles JSON-RPC framing, tool-list advertisement, and error wrapping automatically.

### 2.3 Connecting Claude Code

Add the server to Claude Code's MCP config (`~/.claude/settings.json` or project `.claude/settings.json`):

```json
{
  "mcpServers": {
    "blender": {
      "command": "uvx",
      "args": ["blender-mcp"]
    }
  }
}
```

`uvx` (from the `uv` package manager) creates an ephemeral virtual environment, installs `blender-mcp` from PyPI, and executes the server — no manual `pip install` required. After enabling the addon inside Blender and clicking "Start Server" in the N-panel "BlenderMCP" tab, Claude Code can immediately call Blender tools.

For Claude Desktop, add the same block under `mcpServers` in `claude_desktop_config.json` (`~/.config/Claude/claude_desktop_config.json` on Linux).

---

## 3. MCP Tool Surface: What the AI Sees

The base ahujasid/blender-mcp server exposes 16 tools, grouped into functional categories:

| Tool | Category | Description |
|---|---|---|
| `get_scene_info` | Inspection | Scene name, object list with types, material list |
| `get_object_info` | Inspection | Transform, mesh stats, materials, AABB for a named object |
| `get_viewport_screenshot` | Inspection | PNG of the 3D viewport (base64), with max-dimension limiting |
| `execute_blender_code` | Execution | Run arbitrary Python inside Blender; captures stdout/stderr |
| `get_polyhaven_categories` | PolyHaven | List available asset categories |
| `search_polyhaven_assets` | PolyHaven | Search HDRIs, textures, models by category |
| `download_polyhaven_asset` | PolyHaven | Download and import into the scene |
| `set_texture` | PolyHaven | Apply a downloaded texture, return node graph info |
| `search_sketchfab_models` | Sketchfab | Search with face count, licence, category filters |
| `get_sketchfab_model_preview` | Sketchfab | Base64 thumbnail |
| `download_sketchfab_model` | Sketchfab | Download GLB/GLTF with auto-scale to metres |
| `generate_hyper3d_model_via_text` | Hyper3D Rodin | Text prompt → async 3D generation |
| `generate_hyper3d_model_via_images` | Hyper3D Rodin | Image paths/URLs → async 3D generation |
| `poll_rodin_job_status` | Hyper3D Rodin | Check async job progress |
| `import_generated_asset` | Hyper3D Rodin | Import completed generation into the scene |
| `get_*_status` | Diagnostics | PolyHaven / Sketchfab / Hyper3D availability checks |

The `execute_blender_code` tool is the escape hatch: anything not covered by a dedicated tool can be accomplished by injecting arbitrary `bpy` Python. The AI can use it for one-off operations but structured tools are preferred because they return typed, predictable data that is easier to reason about.

Alternative community implementations add substantially more surface area. `RFingAdam/mcp-blender` exposes 218 tools covering modelling, physics, animation, particle systems, and MSFS content creation. `glonorce/Blender_mcp` exposes 69 tool groups (550+ actions), 499 unit tests, a multilingual intent router (EN/TR/FR), and operates on TCP port 9879.

---

## 4. Thread Safety and bpy.app.timers

Blender's internal state — including all data accessible through `bpy.data`, `bpy.context`, and `bpy.ops` — is not thread-safe. Calling any of these from a background thread causes crashes or data corruption, typically manifesting as `EXCEPTION_ACCESS_VIOLATION` (Windows) or a SIGSEGV (Linux). This is because Blender acquires no lock on its main-thread state; the GIL is insufficient protection since the underlying C data structures are mutated lock-free on the assumption of single-threaded access.

The correct pattern for all socket-server addons is `bpy.app.timers`:

```python
import queue, threading, bpy

_queue: queue.Queue = queue.Queue()

def _process_queue() -> float:
    """Executed on Blender's main thread every 5 ms."""
    while not _queue.empty():
        fn, args, result_box, done_event = _queue.get()
        try:
            result_box.append(fn(*args))
        except Exception as exc:
            result_box.append({"error": str(exc)})
        finally:
            done_event.set()
    return 0.005  # re-schedule after 5 ms

def dispatch_to_main(fn, *args):
    """Call from any thread; blocks until result is ready."""
    result_box = []
    done = threading.Event()
    _queue.put((fn, args, result_box, done))
    done.wait(timeout=60)
    return result_box[0] if result_box else None

# Register once at addon enable time
bpy.app.timers.register(_process_queue, first_interval=0.005)
```

The `bpy.app.timers` API guarantees that the registered function is called on Blender's main thread during the event loop. Returning a `float` schedules the next call after that many seconds; returning `None` cancels the timer.

This pattern — enqueue on socket thread, execute on timer, return result via threading.Event — is used identically in ahujasid/blender-mcp, the official Blender Foundation server, and all serious community implementations.

---

## 5. Blender's Python API: The bpy Module

Blender ships its own embedded Python interpreter; it does not use the system Python on Linux. Each Blender release pins exactly one Python version:

| Blender | Python | Notes |
|---|---|---|
| 5.2 | 3.13 | VFX Platform 2026 |
| 5.1 | 3.13 | VFX Platform 2026 |
| 5.0 | 3.12 | |
| 4.4 | 3.11 | |
| 4.2 LTS | 3.11 | Extended support |
| 4.0 | 3.10 | |

The `bpy` PyPI package (`pip install bpy==5.2.0`) provides a standalone build for use in web services and CI pipelines outside Blender; it matches the Blender release and Python version exactly. The standalone build omits the GPU/OpenGL context — it is suitable for data manipulation, export/import scripting, and procedural geometry generation, but not for rendering or viewport operations.

### 5.1 Core Submodules

| Module | Role |
|---|---|
| `bpy.ops` | All Blender UI operations as Python callables |
| `bpy.data` | Data-block container: objects, meshes, materials, scenes, images, … |
| `bpy.context` | Current editor state: active object, selected objects, mode, scene |
| `bpy.types` | RNA type hierarchy: `Operator`, `Panel`, `PropertyGroup`, `Mesh`, `Object`, … |
| `bpy.props` | Property factory functions for registration |
| `bpy.utils` | Class registration, preview collections, unit conversion |
| `bpy.app` | Timers, handlers, version info, build flags |
| `bpy.path` | Path utilities (`abspath`, `basename`, `ensure_ext`, …) |

### 5.2 C Extension Modules

| Module | Role |
|---|---|
| `mathutils` | `Vector`, `Matrix`, `Quaternion`, `Euler` — thin wrappers over C math |
| `bmesh` | Low-level mesh editing: create/modify vertices, edges, faces, loops |
| `gpu` | Shader programs, compute shaders, framebuffer management (GPU context required) |
| `imbuf` | Pixel-level image manipulation |
| `idprop` | Custom property data-blocks (arbitrary key-value per data-block) |

### 5.3 Key bpy.ops Categories

`bpy.ops` namespaces map to editor categories. Frequently used namespaces:

```python
bpy.ops.mesh        # subdivide, extrude, loop_cut, inset_faces, dissolve_*
bpy.ops.object      # add, delete, join, duplicate, parent_set, transform_apply
bpy.ops.material    # new, slot_add
bpy.ops.render      # render(animation=False, write_still=True)
bpy.ops.export_scene   # gltf, fbx, obj
bpy.ops.import_scene   # gltf, fbx, obj
bpy.ops.node        # add_node, links_mute
bpy.ops.action      # keyframe_insert
bpy.ops.pose        # armature_apply, select_all
```

Every operator has a `poll()` classmethod that Blender calls before `execute()`. Many operators silently fail or raise `RuntimeError: Operator bpy.ops.mesh.subdivide.poll() failed` if called in the wrong context (wrong editor type, wrong object mode, no active object). The `execute_blender_code` MCP tool captures these errors and returns them to the AI so it can correct its approach.

### 5.4 Accessing and Modifying Data Directly

`bpy.ops` is convenient for interactive use but slow for bulk operations — each call goes through the full operator lifecycle, including undo-stack integration and depsgraph updates. For performance-critical scripts, read and write `bpy.data` directly:

```python
import bpy, bmesh

# Direct mesh modification via bmesh
obj  = bpy.data.objects["Cube"]
mesh = obj.data                   # bpy.types.Mesh
bm   = bmesh.new()
bmesh.from_edit_mesh(mesh) if obj.mode == 'EDIT' else bm.from_mesh(mesh)

bm.verts.ensure_lookup_table()
for v in bm.verts:
    v.co.z += 0.1                 # lift all vertices 10 cm

bm.to_mesh(mesh)
bm.free()
mesh.update()

# Direct material property access
mat  = bpy.data.materials["Material"]
bsdf = mat.node_tree.nodes["Principled BSDF"]
bsdf.inputs["Roughness"].default_value = 0.25
```

The depsgraph (dependency graph) is Blender's incremental update system; after bulk edits via `bpy.data`, call `bpy.context.view_layer.update()` or `bpy.ops.object.update_all()` to propagate changes before rendering.

---

## 6. The RNA/DNA Bridge: How bpy Maps to C Internals

Understanding the RNA system is essential for writing reliable add-ons and debugging unexpected attribute errors.

**DNA** is Blender's raw in-memory and on-disk data layer: plain C structs (`Object`, `Mesh`, `Material`, `Scene`, …) defined in `source/blender/makesdna/`. Every `.blend` file is a serialised snapshot of these structs.

**RNA** (Runtime Native Access) wraps DNA with a rich metadata layer: type information, value ranges, UI hints, description strings, and read/write callbacks. RNA definitions live in `source/blender/makesrna/intern/rna_*.c`. A compile-time helper program `makesrna` processes these files and generates `rna_*_gen.c` — static C structs (`StructRNA`, `PropertyRNA`) populated at runtime by `RNA_init()`.

The Python bridge (`source/blender/python/intern/bpy_rna.c`) wraps each `StructRNA` as a Python type (`BPy_StructRNA`) and each property as a descriptor. Attribute access on a Python `bpy.types.Object` goes through `pyrna_prop_array_getattro()` → `RNA_property_*()` → the underlying C field on the DNA struct. There is no intermediate Python object; the RNA bridge reads and writes the live C memory directly.

This design means:
- `bpy.types.Object` attributes are always current — there is no staleness
- Attribute access performance is a function of RNA dispatch overhead, not Python dict lookup
- Type errors from RNA (`AttributeError: 'NoneType' object has no attribute …`) propagate from RNA validation, not from Python `None` dereferences
- The API reference at [docs.blender.org/api/current/](https://docs.blender.org/api/current/) is auto-generated from RNA via `sphinx_doc_gen.py`

For the Blender MCP server, the RNA system is also what makes `execute_blender_code` robust: any code that runs in Blender's Python context has full access to the live RNA-wrapped scene, so the AI can inspect and modify any data-block by name.

---

## 7. Writing Blender Add-ons and Extensions

### 7.1 Classic Add-on Structure (Blender < 4.2)

```python
# my_addon/__init__.py
bl_info = {
    "name": "My Add-on",
    "blender": (4, 0, 0),
    "version": (1, 0, 0),
    "category": "3D View",
}

import bpy

class MY_OT_Action(bpy.types.Operator):
    """Tooltip shown in UI"""
    bl_idname  = "my.action"
    bl_label   = "Run Action"
    bl_options = {'REGISTER', 'UNDO'}

    # Properties declared as class annotations
    count: bpy.props.IntProperty(name="Count", default=3, min=1, max=100)

    @classmethod
    def poll(cls, context):
        return context.active_object is not None

    def execute(self, context):
        obj = context.active_object
        for _ in range(self.count):
            bpy.ops.object.duplicate()
        self.report({'INFO'}, f"Duplicated {self.count} times")
        return {'FINISHED'}

    def invoke(self, context, event):
        return context.window_manager.invoke_props_dialog(self)


class MY_PT_Panel(bpy.types.Panel):
    bl_idname    = "MY_PT_panel"
    bl_label     = "My Tools"
    bl_space_type  = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category    = "My Tab"

    def draw(self, context):
        self.layout.operator("my.action")


class MyAddonSettings(bpy.types.PropertyGroup):
    enabled: bpy.props.BoolProperty(name="Enabled", default=True)


CLASSES = (MyAddonSettings, MY_OT_Action, MY_PT_Panel)

def register():
    for cls in CLASSES:
        bpy.utils.register_class(cls)
    # Attach PropertyGroup to scene data-block
    bpy.types.Scene.my_addon = bpy.props.PointerProperty(type=MyAddonSettings)

def unregister():
    del bpy.types.Scene.my_addon
    for cls in reversed(CLASSES):
        bpy.utils.unregister_class(cls)
```

Registration order matters: `PropertyGroup` subclasses must be registered *before* they are referenced in a `PointerProperty`. Unregistration must be the reverse order to avoid use-after-free in Blender's type system.

### 7.2 Extensions (Blender 4.2+)

Blender 4.2 introduced the Extensions system, which replaces `bl_info` with a TOML manifest and supports bundled Python wheel dependencies:

```toml
# blender_manifest.toml
schema_version = "1.0.0"
id             = "my_addon"
version        = "1.0.0"
name           = "My Add-on"
tagline        = "Does something"
maintainer     = "Author Name <author@example.com>"
type           = "add-on"
blender_version_min = "4.2.0"
license        = ["SPDX:GPL-3.0-or-later"]
[build]
paths = ["my_addon"]
```

Extensions are installable from the Blender Extensions platform (extensions.blender.org) or from local paths. The permission system in Extensions requires explicit declaration of network access, file I/O outside the Blender directory, and execution of subprocesses — a security improvement over classic add-ons.

The official Blender MCP server ships as an Extension with `blender_manifest.toml` and is hosted at extensions.blender.org alongside the traditional `addon.py` for backward compatibility.

### 7.3 Shader Node Graphs via Python

Procedural material creation is a common Claude Code task when working with Blender MCP. Here is the Principled BSDF pattern:

```python
import bpy

def make_pbr_material(name, base_color=(0.8, 0.2, 0.1, 1.0),
                      metallic=0.0, roughness=0.5):
    mat = bpy.data.materials.new(name=name)
    mat.use_nodes = True
    nt    = mat.node_tree
    nodes = nt.nodes
    links = nt.links
    nodes.clear()

    out   = nodes.new('ShaderNodeOutputMaterial')
    bsdf  = nodes.new('ShaderNodeBsdfPrincipled')
    bsdf.inputs['Base Color'].default_value = base_color
    bsdf.inputs['Metallic'].default_value   = metallic
    bsdf.inputs['Roughness'].default_value  = roughness

    links.new(bsdf.outputs['BSDF'], out.inputs['Surface'])
    return mat

# Assign to active object
mat = make_pbr_material("Robot_Metal", metallic=0.9, roughness=0.2)
bpy.context.active_object.data.materials.append(mat)
```

---

## 8. Claude Code Workflows with bpy

Claude Code interacts with Blender in two modes:

**Mode 1 — Script generation (offline):** Claude Code writes a `.py` file that is executed via `blender --background scene.blend --python script.py` or pasted into Blender's Script console. This works without Blender MCP; the AI reasons from the API reference alone. Suitable for batch processing, export pipelines, and procedural asset generation.

**Mode 2 — Live via MCP:** With Blender MCP active, Claude Code can follow an inspect-act-verify loop:

```
1. get_scene_info            → discover what objects exist
2. get_object_info("Cube")   → inspect geometry and materials
3. execute_blender_code      → apply a transformation
4. get_viewport_screenshot   → verify the result visually
5. execute_blender_code      → iterate on shader parameters
```

The `get_viewport_screenshot` tool returns a PNG encoded as base64. Claude's vision capability analyses the image, allowing it to make judgements about visual quality (shading, geometry, proportion) and iterate accordingly — a capability that was impossible with text-only tool output.

### 8.1 Typical Prompt Patterns

When instructing Claude Code to work with Blender via MCP, the most effective prompts are goal-oriented and leave implementation details to the model:

- "Create a futuristic street lamp object — metal pole, glass sphere at the top, emission shader in the glass"
- "Import all GLB files in ~/assets/ and centre each object on the world origin"
- "Render frames 1–100 of the current scene to ~/render/ at 1920×1080"
- "Inspect the active material and reduce its roughness by half"

Claude Code will call `get_scene_info` first to orient itself, then sequence appropriate `execute_blender_code` calls with error recovery if a tool call fails. The structured error return from the addon means Claude can distinguish "operator poll failed (wrong mode)" from "NameError in user code" and adjust accordingly.

### 8.2 The Info Editor as a Learning Tool

Blender's Info editor logs every UI action as its Python equivalent. A highly effective workflow when building scripts with Claude Code is to perform the desired action manually in Blender's UI, copy the operator call from the Info log, then ask Claude to generalise it into a parameterised script:

```python
# Info log for a manual loop cut:
bpy.ops.mesh.loopcut_slide(
    MESH_OT_loopcut={"number_cuts":1, "factor":0.0, "edge_index":2},
    TRANSFORM_OT_edge_slide={"value":0.0}
)
```

This eliminates guesswork about operator parameter names and values.

---

## 9. Meshy and the AI 3D Generation Ecosystem

Meshy ([meshy.ai](https://www.meshy.ai), [docs.meshy.ai](https://docs.meshy.ai)) is the leading commercial AI 3D generation platform as of mid-2026. Its two-step preview/refine pipeline produces textured, PBR-ready meshes from text or image prompts.

### 9.1 Text-to-3D Pipeline

```bash
# Step 1: Generate shape (preview)
curl https://api.meshy.ai/openapi/v2/text-to-3d \
  -H "Authorization: Bearer ${MESHY_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "mode":            "preview",
    "prompt":          "a sci-fi battle drone, aggressive design",
    "ai_model":        "meshy-6",
    "topology":        "quad",
    "target_polycount": 50000,
    "should_remesh":   true,
    "target_formats":  ["glb", "fbx"]
  }'

# Response: {"result": "018a210d-8ba4-705c-b111-1f1776f7f578"}

# Step 2: Add PBR textures (refine)
curl https://api.meshy.ai/openapi/v2/text-to-3d \
  -H "Authorization: Bearer ${MESHY_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "mode":            "refine",
    "preview_task_id": "018a210d-8ba4-705c-b111-1f1776f7f578",
    "enable_pbr":      true,
    "hd_texture":      true
  }'

# Poll status
curl "https://api.meshy.ai/openapi/v2/text-to-3d/018a210d-..." \
  -H "Authorization: Bearer ${MESHY_API_KEY}"
```

The status response transitions through `PENDING` → `IN_PROGRESS` → `SUCCEEDED` (or `FAILED` / `CANCELED`). The `SUCCEEDED` response includes URLs for each texture map:

```json
{
  "status": "SUCCEEDED",
  "model_urls": { "glb": "https://assets.meshy.ai/.../model.glb" },
  "thumbnail_url": "https://assets.meshy.ai/.../thumbnail.png",
  "texture_urls": [{
    "base_color": "https://assets.meshy.ai/.../texture_0.png",
    "metallic":   "https://assets.meshy.ai/.../texture_0_metallic.png",
    "normal":     "https://assets.meshy.ai/.../texture_0_normal.png",
    "roughness":  "https://assets.meshy.ai/.../texture_0_roughness.png",
    "emission":   "https://assets.meshy.ai/.../texture_0_emission.png"
  }]
}
```

Key API parameters for the preview step:

| Parameter | Type | Notes |
|---|---|---|
| `prompt` | string | Max 600 characters |
| `ai_model` | string | `meshy-5`, `meshy-6`, `latest` |
| `topology` | string | `quad` (clean edgeflow) or `triangle` |
| `target_polycount` | int | 100 – 300,000; default 30,000 |
| `pose_mode` | string | `a-pose`, `t-pose` for characters |
| `should_remesh` | bool | Auto re-topology after generation |
| `target_formats` | string[] | `glb`, `obj`, `fbx`, `stl`, `usdz`, `3mf` |

**Smart Topology (2026)** is a new fast-path model that produces clean quad topology in approximately 10 seconds, suitable for characters requiring subsequent rigging or subdivision.

### 9.2 Meshy Blender Plugin

Meshy provides a native Blender plugin available from [meshy.ai/integrations](https://www.meshy.ai/integrations). It adds a sidebar panel in the 3D View that allows text or image prompts to be submitted and the generated model to be imported directly into the active scene without manual download-and-import steps. The plugin requires a Meshy API key and sends requests to the same REST API described above.

### 9.3 Hyper3D Rodin

Hyper3D Rodin ([hyper3d.ai](https://hyper3d.ai), [developer.hyper3d.ai](https://developer.hyper3d.ai)) is integrated directly into the ahujasid/blender-mcp server and is accessible from Claude Code without a separate API key. The `generate_hyper3d_model_via_text` and `generate_hyper3d_model_via_images` tools submit jobs; `poll_rodin_job_status` monitors them; `import_generated_asset` imports the result.

Rodin is also available via [fal.ai/models/fal-ai/hyper3d/rodin](https://fal.ai/models/fal-ai/hyper3d/rodin):

```python
from fal_client import submit, status

handle = submit("fal-ai/hyper3d/rodin", arguments={
    "prompt": "a carved wooden chess piece, bishop",
    "geometry_file_format": "glb",
    "quality": "high",       # 50k faces
    "material": "PBR",
    "use_hyper": True,       # HighPack 4K textures
})
# Poll until done
result = fal_client.result("fal-ai/hyper3d/rodin", handle.request_id)
glb_url = result["model_file"]
```

Quality tiers: `high` (50k faces), `medium` (18k), `low` (8k), `extra_low` (4k). The `HighPack` option upgrades textures to 4096×4096 at triple credit cost.

### 9.4 Competing Platforms

| Platform | Speciality | Output |
|---|---|---|
| **Tripo3D** ([tripo3d.ai](https://www.tripo3d.ai)) | TripoSR open model; sub-second on A100; official Blender add-on | GLB + PBR textures, quad remesh option |
| **CSM.ai** ([csm.ai](https://csm.ai)) | Physics-accurate generation (weight, friction, collision), SAM2 segmentation | Multi-format |
| **Sloyd** ([sloyd.ai](https://www.sloyd.ai)) | Procedural/parametric game props, clean low-poly topology | Consistent style control |
| **Kaedim** | Human-in-the-loop quality layer, sketch/photo/brief inputs | Supervised production assets |

---

## 10. Open-Source Generative 3D Models

### 10.1 Point-E (OpenAI, December 2022)

Point-E ([openai/point-e](https://github.com/openai/point-e)) uses a two-stage diffusion pipeline: text → synthetic 2D view, then conditioned view → 3D point cloud. An optional upsampling network refines the coarse point cloud. Generation runs in seconds on a single consumer GPU. Output is a point cloud that can be converted to a mesh via Poisson reconstruction or marching cubes.

Point-E established the pattern of using a 2D diffusion model as an intermediate to bootstrap 3D shape generation, avoiding the need for vast quantities of paired text–3D training data.

### 10.2 Shap-E (OpenAI, May 2023)

Shap-E ([openai/shap-e](https://github.com/openai/shap-e), [huggingface.co/openai/shap-e](https://huggingface.co/openai/shap-e)) generates Implicit Neural Representations (INRs) rather than explicit point clouds. An encoder maps 3D training assets to the weights of a small MLP that represents an implicit surface (or NeRF). A latent diffusion model then learns to sample from this latent space conditioned on text or images. The same output can be rendered as a textured mesh or as a NeRF. Generation takes approximately 13 seconds and produces sharper edges than Point-E.

### 10.3 TRELLIS (Microsoft Research, 2024)

TRELLIS ([microsoft/TRELLIS](https://github.com/microsoft/TRELLIS)) uses a two-stage approach: a sparse 3D voxel representation is generated first (preserving coarse structure), then a Rectified Flow Transformer refines it to a high-resolution mesh. The sparse voxel intermediate is key to tractable memory use at high resolution. TRELLIS accepts single images as input and produces clean textured meshes competitive with commercial APIs.

### 10.4 Stable Fast 3D (Stability AI, 2024)

SF3D ([stable-fast-3d.github.io](https://stable-fast-3d.github.io)) generates a textured mesh from a single image in under one second. It builds on the TripoSR Large Reconstruction Model architecture but with significant inference optimisations (compiled CUDA kernels, reduced network depth). The sub-second latency makes it viable for interactive use within Blender with the right integration.

### 10.5 InstantMesh

InstantMesh ([InstantMesh paper](https://arxiv.org/abs/2404.07191)) combines multi-view diffusion (generating 6 canonical views from a single input) with a transformer-based reconstruction model that produces an explicit mesh. The reconstruction model is trained end-to-end on the multi-view output, allowing it to handle occlusions that single-view methods cannot resolve.

---

## 11. AI Denoising in Cycles: OIDN and OptiX

AI denoising is the most mature AI feature integrated directly into Blender and has been production-ready since Blender 2.81 (OIDN) and 2.83 (OptiX viewport denoising).

### 11.1 Intel OpenImageDenoise (OIDN)

OIDN ([openimagedenoise.github.io](https://openimagedenoise.github.io)) is an open-source, cross-hardware ML denoiser integrated into Cycles since Blender 2.81. OIDN 2.0 (bundled with Blender 4.x and 5.x) adds GPU acceleration via SYCL (Intel), CUDA (NVIDIA), and HIP (AMD), selecting the best available device automatically.

In Cycles render settings, select **Denoising** → **OpenImageDenoise**. For final renders, OIDN produces the highest quality output regardless of GPU vendor. For the compositing node graph, the **Denoise** compositor node applies OIDN to pre-rendered image data, including auxiliary `Denoising Normal` and `Denoising Albedo` render passes for guided denoising:

```python
# Programmatic setup of Cycles OIDN + auxiliary passes
import bpy

scene  = bpy.context.scene
cycles = scene.cycles

cycles.use_denoising  = True
cycles.denoiser       = 'OPENIMAGEDENOISE'
cycles.denoising_input_passes = 'RGB_ALBEDO_NORMAL'

# Enable auxiliary render passes
vl = scene.view_layers[0]
vl.use_pass_denoising_normal = True
vl.use_pass_denoising_albedo = True
```

### 11.2 NVIDIA OptiX Denoiser

The OptiX AI-Accelerated Denoiser uses NVIDIA's RT Core hardware via the OptiX SDK and is available in Cycles as `cycles.denoiser = 'OPTIX'`. It requires:
- NVIDIA driver ≥ 440.59 on Linux
- An RTX-class GPU (Turing or later) for hardware RT Core acceleration; Maxwell/Pascal GPUs fall back to CUDA compute

The OptiX denoiser is fastest of the three options (OptiX / OIDN / NLM) on RTX hardware but produces slightly lower quality on complex scenes compared to OIDN, and has historically shown temporal instability in animations. Best use case: interactive viewport preview during lighting iteration, where latency matters more than pixel-perfect accuracy.

```python
# OptiX denoiser (RTX required)
cycles.denoiser = 'OPTIX'
```

### 11.3 Denoiser Node in the Compositor

Both OIDN and OptiX are available as the **Denoise** compositor node (Blender 2.81+), enabling denoising in the node-based post-processing pipeline independent of the render setting:

```
[Render Layers] → Noisy Image ─────────────────────┐
[Render Layers] → Denoising Normal ─────────────────→ [Denoise] → [Composite]
[Render Layers] → Denoising Albedo ─────────────────┘
```

This separation allows the raw noisy render to be retained for debugging while the denoised output is used for final compositing.

---

## 12. Dream Textures and AI Texturing

### 12.1 Dream Textures

Dream Textures ([github.com/carson-katri/dream-textures](https://github.com/carson-katri/dream-textures), 8,200+ stars, GPL-3.0) embeds Stable Diffusion directly inside Blender as a Blender add-on. It uses the Hugging Face `diffusers` library as its backend and supports:

- **Text-to-texture**: Generate seamlessly tiling textures from a text prompt
- **Project Dream Texture**: Depth-to-image mapping onto a full 3D scene — the current scene view is used as a depth/structure guide, and Stable Diffusion fills in photorealistic detail
- **Inpainting and outpainting**: Edit sections of existing textures
- **AI upscaling**: 4× upscale with structure preservation
- **Real-time viewport integration**: Updates the viewport texture as the diffusion process runs
- **Cycles render pass integration**: Post-processes rendered frames using img2img

```
dream-textures/
├── __init__.py               # Registration, bl_info
├── property_groups/          # UI parameters (prompt, seed, steps, …)
├── operators/                # Operator classes that trigger generation
│   ├── dream_texture.py      # Main text-to-image operator
│   ├── project_dream_texture.py
│   └── upscale.py
├── generator_process/        # Background subprocess queue for diffusers
│   ├── actions/generate.py   # Calls diffusers pipeline
│   └── actions/upscale.py
└── ui/                       # Panel definitions for Shader Editor, Image Editor
```

The `generator_process` module spawns a separate Python subprocess (using the system Python, not Blender's embedded interpreter) to run `diffusers` pipelines. Results are sent back to Blender via inter-process communication and applied to Blender image data-blocks. VRAM requirement: 4 GB minimum; 8 GB recommended for 512×512 generation.

### 12.2 AI Render

AI Render ([github.com/benrugg/AI-Render](https://github.com/benrugg/AI-Render)) hooks into Blender's render output pipeline and transforms rendered frames using Stable Diffusion img2img. It supports:
- Automatic1111's Stable Diffusion Web UI as a local backend
- Cloud API backends as alternatives
- Per-frame prompt animation (the prompt text can be keyframed)
- Batch processing of animation sequences

### 12.3 Using Meshy AI Texturing on Existing Meshes

Meshy's AI Texturing endpoint accepts an untextured mesh and a text description:

```bash
curl https://api.meshy.ai/openapi/v2/ai-texture \
  -H "Authorization: Bearer ${MESHY_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model_url":  "https://example.com/model.glb",
    "object_prompt": "wooden medieval chest with iron hinges",
    "style_prompt": "photorealistic, game-ready PBR"
  }'
```

The response follows the same async pattern as text-to-3D.

---

## 13. AI Rigging and UV Automation

### 13.1 Meshy AI Animation

Meshy's AI Animation feature auto-rigs and animates generated or imported meshes. For humanoid models in A-pose or T-pose, the system detects the skeleton structure, places control bones, and can export pre-made animation clips (idle, walk, run) bound to the generated rig.

### 13.2 AI UV Unwrapping

Autodesk Research demonstrated a **Graph Neural Network–based UV seaming** system (2024) that analyses mesh topology to replicate artist UV seaming style while minimising distortion. The GNN is trained on examples of artist-unwrapped models; at inference it outputs seam candidates that respect semantic surface boundaries (along silhouettes, at material transitions). Tripo3D offers a similar AI unwrapping service in its commercial pipeline.

For open-source UV automation, the traditional smart UV project and angle-based unwrap operators in `bpy.ops.uv` remain the practical standard; GNN-based approaches are available as commercial add-ons or cloud APIs but not yet integrated into upstream Blender.

### 13.3 NVIDIA OptiX Denoising for Renders in Pipeline

When building automated render pipelines on Linux with NVIDIA GPUs, the OptiX denoiser requires no X display but does require `libcuda.so` and `liboptix.so.8`:

```bash
# Check OptiX availability in headless Blender
blender --background --python-expr "
import bpy
bpy.context.scene.cycles.denoiser = 'OPTIX'
bpy.ops.render.render(write_still=True)
" 2>&1 | grep -E "Error|OptiX|denoiser"
```

OIDN GPU acceleration works without OptiX libraries and functions correctly in headless EGL-based rendering environments on any vendor GPU.

---

## 14. Headless Blender and Pipeline Integration on Linux

### 14.1 CLI Flags

```bash
# Background render: no window, execute Python, render current frame
blender --background scene.blend --python render.py

# Inline Python (no script file)
blender --background --python-expr "
import bpy
bpy.context.scene.render.filepath = '/tmp/output'
bpy.ops.render.render(write_still=True)
"

# Render frame range
blender -b scene.blend -o //renders/frame_ -s 1 -e 100 -a

# Run with specific add-ons enabled
blender --background --addons my_addon --python process.py
```

Key flags:

| Flag | Effect |
|---|---|
| `--background` / `-b` | No window, no event loop |
| `--python` / `-P` | Execute a `.py` script file after startup |
| `--python-expr` | Execute a Python expression |
| `--python-exit-code N` | Set process exit code on Python exception |
| `--addons a,b,c` | Enable named add-ons at startup |
| `-y` / `--enable-autoexec` | Trust embedded Python in `.blend` files |
| `-f N` | Render frame N |
| `-a` | Render full animation |
| `-o //path/` | Output path template |

### 14.2 Virtual Display for GPU Rendering

Headless GPU rendering with EEVEE (Vulkan) requires a Wayland or X display for the swapchain:

```bash
# Option 1: Xvfb
Xvfb :99 -screen 0 1280x720x24 &
DISPLAY=:99 blender --background scene.blend --python render.py

# Option 2: xvfb-run (convenience wrapper)
xvfb-run --auto-servernum blender --background scene.blend --python render.py
```

Cycles with the OIDN GPU denoiser and HIP/CUDA compute backends does **not** require a display; headless operation works without Xvfb on a GPU-equipped Linux server.

### 14.3 The standalone bpy Package

For server-side asset processing that does not require rendering, the standalone `bpy` PyPI package allows importing and using Blender's Python API outside of Blender:

```python
# In a FastAPI service or CLI tool
import bpy

bpy.ops.wm.read_factory_settings(use_empty=True)   # clear default scene
bpy.ops.wm.open_mainfile(filepath="/path/to/scene.blend")

# Export to glTF
bpy.ops.export_scene.gltf(
    filepath="/output/exported.glb",
    export_format='GLB',
    export_apply=True,
)
```

```bash
# Install standalone bpy (must match Blender release)
pip install bpy==4.2.0   # for Blender 4.2 LTS
```

The standalone `bpy` is useful for:
- CI validation of `.blend` files (check object counts, material assignments, missing textures)
- Server-side export pipelines (`.blend` → `.glb` for web delivery)
- Asset database generation (extract metadata from many `.blend` files)
- Integration testing of add-ons without launching the Blender UI

### 14.4 Blender MCP in Headless Pipelines

The two-component MCP architecture requires a running Blender session with its event loop active. Blender's `--background` flag suppresses the window and the windowing-system event loop. The `bpy.app.timers` mechanism (which the MCP addon uses for thread-safe command dispatch — see §4) relies on that event loop: without it, timers do not fire and the MCP addon cannot process incoming commands.

**Practical guidance:** Do not attempt to run the MCP server in `--background` mode. Use MCP for interactive creative sessions where Blender is running with a display (or a virtual display via Xvfb for EEVEE). For automated pipelines, use the headless approaches in §14.1–14.3 instead:

- **Script pipelines**: `blender --background scene.blend --python script.py` — the AI writes the script offline and it is executed in a single Blender invocation.
- **Asset processing**: the standalone `bpy` PyPI package handles import/export/inspection without Blender's event loop at all.
- **Batch rendering**: `blender -b scene.blend -o //renders/frame_ -s 1 -e 100 -a` — no MCP needed.

The official Blender Lab MCP server (v1.0.0) includes an **Auto Start** preference in its Extension settings that automatically starts the TCP server when Blender opens in interactive mode, eliminating the need to click a panel button each session. This is the recommended configuration for MCP-driven creative workflows.

---

## 15. Practical Limits of AI-Driven Blender

Understanding where AI-driven Blender works reliably versus where it struggles is as important as knowing what is possible. The limits fall into three distinct categories: API surface gaps (no Python hook exists), context-guarding failures (the `poll()` barrier), and feedback-loop latency.

### 15.1 What Works Reliably

AI agents are effective for any Blender task that maps cleanly to a scripted sequence of `bpy` calls and whose correctness can be checked from structured data returned by inspection tools:

**Scene construction.** Creating objects, setting transforms, parenting, organizing collections — all accomplished through `bpy.ops.object` and `bpy.data` with predictable results. This is exactly the domain where Blender's menu system is most intimidating to new users; Claude can navigate it by operator name without needing to locate menus.

**Material and shader graph authoring.** The Principled BSDF node wiring pattern (§7.3) is well-understood from training data. Claude can create physically plausible PBR materials from a description, wire texture image nodes, set up emission shaders, and build node group hierarchies. The node graph API (`mat.node_tree.nodes`, `mat.node_tree.links`) is consistent and rarely changes between Blender releases.

**Procedural geometry.** Anything expressible as a `bmesh` operation or a modifier stack — arrays, subdivision surfaces, boolean operations, edge loops, displacement — is reliable. The bmesh API (§5.2) is low-level and predictable; results can be verified by requesting a `get_object_info` call afterwards.

**Export and import pipelines.** Converting `.blend` files to glTF/GLB, FBX, or OBJ; batch-importing AI-generated assets; extracting scene metadata. These operations are parameter-rich but the parameters are well-documented and stable.

**Render configuration.** Setting output resolution, frame range, sample count, denoiser, output path, and triggering a render. The Cycles and EEVEE settings APIs have not changed significantly since Blender 3.x; Claude's knowledge of them is reliable.

**Info editor generalisation.** Performing one operation manually in Blender's UI generates a Python line in the Info editor (`Window` → `Toggle System Console` is not needed — the Info editor (`Scripting` workspace, top bar) logs every action). Copying that line and asking Claude to generalise it into a parameterised function is one of the highest-reliability workflows available. The operator names, keyword argument names, and value types are directly readable from the log.

### 15.2 The Operator poll() Barrier

The largest source of runtime failures in AI-generated Blender code is `bpy.ops` context requirements. Every Blender operator has a `poll()` classmethod that Blender evaluates before calling `execute()`. If `poll()` returns `False`, the operator raises:

```
RuntimeError: Operator bpy.ops.mesh.subdivide.poll() failed, context is incorrect
```

Common `poll()` requirements and how to satisfy them:

| Operator namespace | Required context | Fix |
|---|---|---|
| `bpy.ops.mesh.*` | Object in `EDIT` mode | `bpy.ops.object.mode_set(mode='EDIT')` |
| `bpy.ops.object.shade_smooth` | Object selected, in Object mode | `obj.select_set(True); bpy.context.view_layer.objects.active = obj` |
| `bpy.ops.render.render` | A camera in the scene | `bpy.ops.object.camera_add()` if none exists |
| `bpy.ops.uv.*` | Object in Edit mode, UV editor active | Requires overriding `bpy.context.area.type = 'IMAGE_EDITOR'` |
| `bpy.ops.pose.*` | Armature in Pose mode | `bpy.ops.object.mode_set(mode='POSE')` |
| `bpy.ops.node.*` | Node editor area active | Context override needed |

The `execute_blender_code` MCP tool captures the full Python traceback and returns it as a structured error. Claude can read the error, identify the missing context setup, and retry — but each retry is a round trip. For reliable scripts, prefer direct `bpy.data` manipulation (which has no `poll()` requirements) over `bpy.ops` wherever the operation is available at the data level.

Context overrides (`with bpy.context.temp_override(...)`) allow operator calls outside their normal editor context and are the correct solution for headless scripts that call operators requiring specific area types:

```python
# Call a node editor operator without switching the UI
import bpy

for area in bpy.context.screen.areas:
    if area.type == 'NODE_EDITOR':
        with bpy.context.temp_override(area=area):
            bpy.ops.node.select_all(action='SELECT')
        break
```

### 15.3 Where the API Surface Runs Out

Several major Blender workflows have minimal or unusable Python API surface:

**Sculpting.** The sculpt tool (`bpy.ops.sculpt.*`) exposes brush parameter settings but has no API for applying brush strokes programmatically with arbitrary position, pressure, and direction. Sculpted forms must be created by other means (displacement modifiers, imported meshes, procedural geometry) and then refined interactively.

**Grease Pencil strokes.** Grease Pencil v3 (Blender 4.3+) has a Python API for reading and writing stroke point data, but stroke simulation (pressure sensitivity, velocity-dependent width) is not scriptable with the nuance a manual artist achieves.

**Physics simulation debugging.** Setting up rigid body, cloth, or fluid simulation parameters is fully scriptable. Diagnosing *why* a simulation produces wrong results — cloth self-intersecting, fluid leaking, rigid body jitter — requires interactive playback with parameter tweaking. There is no API for "play simulation and return quality metrics."

**Video Sequence Editor.** The VSE Python API exists but is complex enough that generating correct strip timing, transitions, and audio sync from a script requires substantial error-prone bookkeeping. Claude can do it, but failures are common.

**Compositor node trees.** The compositing node graph uses the same `node_tree` API as material shaders, but the node type names and socket names differ and are not consistent across Blender versions. Claude frequently generates compositor node connections that silently fail to link because a socket name changed (e.g., `"Image"` vs `"Color"` depending on version and node type).

### 15.4 The Visual Feedback Loop

Without `get_viewport_screenshot`, the AI works blind: it can construct geometry and materials entirely from the structured data returned by `get_scene_info` and `get_object_info`, but it cannot judge proportions, lighting feel, or render quality. With the screenshot tool, a feedback loop becomes possible:

```
prompt → execute_blender_code → get_viewport_screenshot → analyse → adjust → repeat
```

The practical constraint is latency: each round trip (tool call → Blender execution → screenshot encode → Claude vision analysis) takes several seconds. For fine aesthetic iteration — adjusting a light angle by a few degrees, tweaking roughness — this loop is slow compared to dragging a slider interactively. The sweet spot is coarser creative decisions: "is the overall composition balanced?", "does this material read as metallic?", "is the lighting direction correct?"

Claude's vision capability is competent at identifying gross problems (wrong hemisphere lighting, obviously incorrect scale, materials with clearly wrong albedo) but unreliable for fine-grained aesthetic judgements that experienced artists make intuitively.

### 15.5 Animation and Rigging

Keyframe animation is fully scriptable through `bpy.data.objects[name].keyframe_insert()` and FCurve manipulation. Rigging is scriptable through armature creation, bone parenting, and constraint assignment. Both work; both are verbose and error-prone for complex cases:

- **Armature construction**: bone head/tail positions must be specified in exact coordinates; an off-by-one in bone hierarchy causes the entire rig to deform incorrectly.
- **IK chain setup**: requires setting `bone.use_ik_limit_x`, `bone.ik_stiffness_x`, `bone.ik_stretch` and adding an `INVERSE_KINEMATICS` constraint via `bpy.ops.pose.ik_add()`, which itself requires Pose mode context.
- **NLA (Non-Linear Animation)**: the NLA editor Python API involves `action`, `nla_track`, and `nla_strip` objects whose timing relationships are easy to get wrong.
- **Shape keys**: creating and keying shape keys (`obj.shape_key_add()`, `key.keyframe_insert("value")`) is reliable for simple cases; driver-based shape key animation requires the driver API (`fcurve.driver`, `driver.variables`) which is complex.

For AI-generated character assets from Meshy or Hyper3D Rodin, the most practical approach is to import the mesh and use Blender's auto-rigging tools (Rigify) via `bpy.ops`, which produces a complete rig from a rest-pose mesh with one operator call, rather than constructing the armature bone-by-bone.

### 15.6 Geometry Nodes

Geometry Nodes (Blender 3.0+) is a procedural modelling system implemented as a node graph. It is Turing-complete and can produce complex parametric geometry, but its Python API has a significant limitation: node type identifiers and socket names are internal strings that change between Blender releases without deprecation warnings.

```python
# This works in Blender 4.2 but may fail in 5.x if node identifiers changed
modifier = obj.modifiers.new(name="GeoNodes", type='NODES')
tree = bpy.data.node_groups.new("Procedural", 'GeometryNodeTree')
node = tree.nodes.new('GeometryNodeMeshCube')   # internal type string
```

The alternative is to create Geometry Nodes setups in the UI, save them in a library `.blend` file, and append the node group via `bpy.ops.wm.append()`:

```python
bpy.ops.wm.append(
    filepath="/path/to/library.blend/NodeTree/PipeGenerator",
    directory="/path/to/library.blend/NodeTree/",
    filename="PipeGenerator",
)
```

This is significantly more robust than constructing node trees programmatically and is the pattern Claude should use for complex Geometry Nodes setups.

---

## 16. A Prompt Toolkit for Common Blender Operations

Effective AI-driven Blender workflows depend on prompts that are **goal-oriented, not procedure-oriented**. The AI knows the API; the user knows the intent. Prompts should specify what to achieve, relevant constraints (style, scale, poly budget), and what to verify — not which operators to call.

The following toolkit covers the highest-value recurring operations. Each prompt is designed to work with Claude Code + Blender MCP active, but the scene-modifying ones also work as offline script generation requests (replace "do this in Blender" with "write a bpy script that").

### 16.1 Scene Inspection and Audit

```
Audit the current scene:
- List all objects with their type (MESH/LIGHT/CAMERA/EMPTY), poly count, and assigned materials.
- Flag any mesh objects with zero materials, any materials with missing image textures, and any objects with unapplied scale or rotation.
- Report total triangle count across all meshes.
Output as a structured summary I can paste into my notes.
```

```
What is in this scene? Give me the name, type, world-space location,
bounding box dimensions (in metres), and first material name for every object.
Sort by poly count descending.
```

### 16.2 Material Creation

```
Create a [description] material on the active object.
Use the Principled BSDF shader with:
- Base color: [colour name or hex]
- Metallic: [0–1]
- Roughness: [0–1]
- Optional: emission colour [colour] at strength [N]
Name the material "[name]".
Take a viewport screenshot so I can confirm it looks correct.
```

```
Apply a toon/cel-shaded material to [object name]:
use a Diffuse BSDF fed into a ColorRamp with hard bands,
then into the Material Output. Set 3 bands: shadow, midtone, highlight.
```

```
Import the PBR texture set from ~/textures/[name]/:
  base_color.png, metallic.png, roughness.png, normal.png
Create a new material named "[name]", wire all four maps using
Image Texture nodes into a Principled BSDF, with the normal map
going through a Normal Map node. Assign it to [object name].
```

### 16.3 Lighting Rigs

```
Delete any existing lights. Set up a three-point lighting rig
for the active object:
- Key light: Area, 1000 W, warm white (6000 K), positioned 3 m up and
  45° to the left of the object.
- Fill light: Area, 300 W, cool white (8000 K), 45° to the right,
  2 m height, no shadows.
- Rim light: Spot, 500 W, pure white, positioned 2 m behind and above,
  aimed at the object's top.
Take a viewport screenshot in rendered preview mode.
```

```
Set up an HDRI lighting environment using [filename].hdr from ~/hdri/.
Set the World shader to use this HDRI with rotation [degrees] degrees
and strength [N]. Disable all other lights in the scene.
```

### 16.4 Batch Object Operations

```
For every mesh object in the scene:
1. Apply all modifiers (preserving the result as a new mesh).
2. Set origin to geometry.
3. Apply scale and rotation transforms.
4. Rename the object to its mesh data-block name.
Report any failures with the object name and error.
```

```
Select all mesh objects whose name starts with "prop_".
Join them into a single object named "props_combined",
then set origin to the combined geometry's bounding box centre.
```

```
For every object in the collection named "[collection]",
export it as a separate GLB file to ~/exports/[collection]/.
Use these export settings: apply modifiers, Y-up axis, no materials
(geometry only). Name each file after the object.
```

### 16.5 Procedural Geometry

```
Create a procedural city block footprint:
- A 20×20 m ground plane.
- 8 to 12 box buildings randomly distributed across it,
  heights between 5 m and 30 m, footprints 3×3 m to 8×8 m.
- No building should overlap another or the ground plane edge.
- Add a simple grey concrete material to all buildings,
  a dark asphalt material to the ground plane.
Take a viewport screenshot from above (top orthographic) to confirm layout.
```

```
Subdivide the active mesh [N] times and add a displacement modifier
using a Musgrave texture with scale [S] and depth [D].
Apply the modifier. Report the resulting vertex count.
```

### 16.6 Render and Output

```
Configure the scene for a final Cycles render:
- Resolution: 3840×2160 (4K)
- Samples: 512
- Denoiser: OpenImageDenoise with RGB + Albedo + Normal passes
- Output format: PNG, 16-bit, sRGB colour space
- Output path: ~/renders/[project]/frame_####.png
Render frame [N] and report the output file path when done.
```

```
Set up a turntable animation of [object name]:
- 120 frames at 24 fps (5-second spin).
- Add an Empty at the object's origin; parent the object to it.
- Keyframe the Empty's Z rotation: 0° at frame 1, 360° at frame 120,
  set both keyframes to LINEAR interpolation (no easing).
- Position the camera 5 m away at 30° elevation, looking at the object origin.
- Render the animation to ~/renders/turntable/ as a PNG sequence.
```

```
Render all cameras in the scene, one output file per camera.
Name each output ~/renders/[scene_name]/[camera_name].png.
Use the current render settings. Report each file path as it completes.
```

### 16.7 Asset Import and Organisation

```
Import all GLB files from ~/assets/[folder]/ into the current scene.
For each imported object:
1. Move it to a collection named after the source folder.
2. Set its origin to its bounding box centre.
3. Scale it so its longest axis is exactly 1 m.
4. Place it on the world origin.
Report any files that fail to import.
```

```
Connect to Meshy and generate a "[description]" 3D asset:
- Model: meshy-6, quad topology, target 30,000 polygons
- Enable PBR textures in the refine step
- Import the result into the scene and scale to [N] metres tall
Take a viewport screenshot when done.
```

### 16.8 Info Editor Generalisation

This workflow requires no prior knowledge of the Blender API:

1. Perform the desired operation manually in Blender's UI.
2. Open the Info editor (top bar of the default layout or `Scripting` workspace header).
3. Copy the Python line(s) logged by the operation.
4. Paste into Claude with this prompt:

```
I performed this operation manually in Blender and the Info editor logged:

[paste logged Python here]

Please generalise this into a Python function that:
- Takes [describe what should be a parameter] as arguments
- Handles the case where [describe edge case]
- Works on [all selected objects / a named object / every object in collection X]
- Adds appropriate error handling for poll() failures
```

This is the highest-reliability workflow for operations Claude might not have memorised exactly: the operator names, argument names, and argument types come directly from Blender itself.

### 16.9 Debugging Prompts

When Claude's generated code fails, these prompts help diagnose the issue efficiently:

```
The previous script failed with:
[paste error]

The active object is [name], type [type], current mode [mode].
The scene has [describe relevant state].
Fix the script and explain what context was missing.
```

```
Get scene info and then get object info for [name].
Tell me: is the object's scale applied? Are there unapplied modifiers?
Does the object have the right number of materials for what I described?
What would need to be true before [operation] could succeed?
```

```
Take a viewport screenshot. Describe what you see:
object count, approximate scale, lighting direction,
any visible material or geometry problems.
Compare to what I asked for: [original request].
What is wrong and what should be done next?
```

---

## 17. Blender glTF Export for Three.js and React Three Fiber

glTF 2.0 is the standard interchange format between Blender and web 3D frameworks. Blender ships `io_scene_gltf2`, a full-featured glTF exporter maintained jointly by the Blender Foundation and Khronos Group contributors, capable of producing assets that load directly into Three.js and React Three Fiber with complete material fidelity. This section covers the exporter's parameters, coordinate system mapping, KHR extension coverage, and the full optimisation pipeline for production web delivery.

### 17.1 The io_scene_gltf2 Exporter

`io_scene_gltf2` is bundled with Blender and enabled by default. It produces three output formats:

| Format | Flag | Description |
|---|---|---|
| **GLB** | `'GLB'` | Single binary file: JSON header + binary chunk + embedded textures. Preferred for web. |
| **GLTF + separate** | `'GLTF_SEPARATE'` | JSON file + `.bin` geometry buffer + separate texture files. Useful for inspecting the JSON or substituting textures. |
| **GLTF embedded** | `'GLTF_EMBEDDED'` | JSON file with buffers and textures base64-encoded inline. Largest size; rarely used in production. |

The exporter is invoked via `bpy.ops.export_scene.gltf()`:

```python
import bpy

bpy.ops.export_scene.gltf(
    filepath        = "/output/model.glb",
    export_format   = 'GLB',

    # Geometry
    export_apply    = True,   # apply modifiers and transforms
    export_yup      = True,   # convert Blender Z-up to glTF Y-up (default True)
    export_normals  = True,
    export_tangents = False,  # compute in shader instead; smaller file
    export_texcoords = True,
    export_colors   = False,  # vertex colors; omit unless needed

    # Materials and textures
    export_materials        = 'EXPORT',    # 'PLACEHOLDER' or 'NONE' for geometry only
    export_image_format     = 'AUTO',      # 'JPEG', 'WEBP', 'NONE' or 'AUTO'
    export_jpeg_quality     = 75,
    export_image_add_webp   = True,        # add WebP fallback alongside JPEG/PNG

    # Draco mesh compression (KHR_draco_mesh_compression)
    export_draco_mesh_compression_enable = True,
    export_draco_mesh_compression_level  = 6,   # 0 (fastest) – 6 (smallest)
    export_draco_position_quantization   = 14,  # bits for vertex positions
    export_draco_normal_quantization     = 10,
    export_draco_texcoord_quantization   = 12,

    # Animation and rigging
    export_animations = True,
    export_skins      = True,
    export_morph      = True,   # shape keys as morph targets

    # Scene elements
    export_cameras = False,
    export_lights  = True,      # KHR_lights_punctual

    # Scope
    use_selection           = False,  # export all, not just selection
    use_active_collection   = False,
)
```

### 17.2 Coordinate System and Scale

Blender uses a right-handed **Z-up** coordinate system. glTF 2.0 specifies right-handed **Y-up**. With `export_yup=True` (the default), the exporter applies a −90° rotation around the X axis to every root node, converting:

```
Blender        glTF / Three.js
  Z  ↑             Y  ↑
     │                │
     └──→ Y           └──→ X
    ╱                ╱
   X                Z (into screen)
```

Three.js's default camera looks down the −Z axis; glTF assets appear correctly oriented after the Y-up conversion without additional transforms in Three.js.

**Scale.** Blender works in metres by default; glTF units are dimensionless but conventionally treated as metres by Three.js, Babylon.js, and the Khronos Viewer. Before exporting, ensure all scale transforms are applied (`bpy.ops.object.transform_apply(scale=True)`) or use `export_apply=True`. An object with an unapplied scale of 100 arrives in Three.js as a mesh 100× larger than intended.

**Apply transforms before export** is the most common source of broken web assets from Blender. The prompt in §16.4 ("Apply all modifiers, set origin to geometry, apply scale and rotation") should always precede an export step.

### 17.3 KHR Extension Coverage

The Blender exporter supports a comprehensive set of official Khronos extensions. Understanding which extensions Three.js supports determines which Blender material features survive the round-trip:

| Extension | Blender source | Three.js support | Notes |
|---|---|---|---|
| `KHR_draco_mesh_compression` | `export_draco_*` flags | DRACOLoader required | 40–80% geometry size reduction |
| `KHR_lights_punctual` | Point/Spot/Sun lamps | `RectAreaLight` excluded | Requires `export_lights=True` |
| `KHR_materials_clearcoat` | Principled BSDF Clearcoat | r125+ | Car paint, lacquered surfaces |
| `KHR_materials_sheen` | Principled BSDF Sheen | r138+ | Fabric, velvet |
| `KHR_materials_transmission` | Principled BSDF Transmission | r130+ | Glass, water; requires physical camera |
| `KHR_materials_volume` | Principled BSDF IOR + absorption | r138+ | Solid glass colour |
| `KHR_materials_emissive_strength` | Emission Strength > 1 | r148+ | Required for bloom effects |
| `KHR_materials_unlit` | Unlit shader node | Full | Flat-shaded UI elements, skyboxes |
| `KHR_texture_transform` | UV Mapping node offset/scale | Full | Tiling, rotation, offset |
| `KHR_mesh_quantization` | Vertex attribute quantization | Limited | File size reduction; check loader version |
| `EXT_meshopt_compression` | Via post-processing (gltf-transform) | r139+ | Better than Draco for animated meshes |
| `KHR_texture_basisu` | Via post-processing (gltf-transform) | KTX2Loader required | GPU-native texture compression |

The Principled BSDF is the authoritative source for all standard PBR material channels. The mapping to glTF `pbrMetallicRoughness` is lossless for the core channels:

| Principled BSDF input | glTF field | Three.js MeshStandardMaterial |
|---|---|---|
| Base Color | `baseColorFactor` / `baseColorTexture` | `color` / `map` |
| Metallic | `metallicFactor` / `metallicRoughnessTexture` (B channel) | `metalness` / `metalnessMap` |
| Roughness | `roughnessFactor` / `metallicRoughnessTexture` (G channel) | `roughness` / `roughnessMap` |
| Normal | `normalTexture` | `normalMap` |
| Emission | `emissiveFactor` / `emissiveTexture` | `emissive` / `emissiveMap` |
| Alpha (Transmission ≈ 0) | `alphaCutoff` / `alphaMode` | `transparent` / `opacity` |

Metallic and Roughness are packed into a single texture (ORM: Occlusion in R, Roughness in G, Metallic in B) by the exporter to save texture slots.

### 17.4 Scripting the Export Pipeline

A complete batch export script suitable for a CI pipeline:

```python
#!/usr/bin/env python3
# export_web_assets.py — run as: blender -b scene.blend --python export_web_assets.py

import bpy, os, sys

OUTPUT_DIR = os.environ.get("EXPORT_DIR", "/output/web-assets")
os.makedirs(OUTPUT_DIR, exist_ok=True)

EXPORT_SETTINGS = dict(
    export_format  = 'GLB',
    export_apply   = True,
    export_yup     = True,
    export_normals = True,
    export_materials = 'EXPORT',
    export_image_format = 'WEBP',
    export_jpeg_quality = 80,
    export_draco_mesh_compression_enable = True,
    export_draco_mesh_compression_level  = 5,
    export_animations = True,
    export_skins      = True,
    export_morph      = True,
    export_lights     = True,
)

errors = []
for col in bpy.data.collections:
    if col.name.startswith("_"):   # skip internal collections
        continue
    # deselect all, select this collection's objects
    bpy.ops.object.select_all(action='DESELECT')
    for obj in col.objects:
        obj.select_set(True)

    out_path = os.path.join(OUTPUT_DIR, f"{col.name}.glb")
    try:
        bpy.ops.export_scene.gltf(
            filepath             = out_path,
            use_active_collection = False,
            use_selection        = True,
            **EXPORT_SETTINGS,
        )
        print(f"Exported: {out_path}")
    except Exception as exc:
        errors.append((col.name, str(exc)))
        print(f"ERROR exporting {col.name}: {exc}", file=sys.stderr)

if errors:
    print("\nFailed collections:", file=sys.stderr)
    for name, msg in errors:
        print(f"  {name}: {msg}", file=sys.stderr)
    sys.exit(1)
```

```bash
# Run headless
blender --background scene.blend --python export_web_assets.py
# Or with environment override
EXPORT_DIR=/srv/assets blender -b scene.blend -P export_web_assets.py
```

### 17.5 Post-Processing with gltf-transform

The [`@gltf-transform/cli`](https://gltf.report/transform) tool applies optimisation passes to an already-exported GLB. It handles tasks the Blender exporter cannot:

```bash
npm install --global @gltf-transform/cli

# Full optimisation pipeline
npx @gltf-transform/cli \
    optimize input.glb output.glb \
    --texture-compress webp \        # KHR_texture_basisu (WebP → KTX2 fallback)
    --draco \                        # KHR_draco_mesh_compression
    --meshopt                        # EXT_meshopt_compression for animated meshes

# Individual passes
npx @gltf-transform/cli draco input.glb draco.glb --quantize-position 14
npx @gltf-transform/cli webp input.glb webp.glb --quality 80
npx @gltf-transform/cli resize input.glb resized.glb --width 1024 --height 1024

# Validation
npx gltf-validator output.glb
# Returns JSON report: errors, warnings, info (triangle count, texture count, etc.)
```

Integrate validation into CI to catch broken exports before they reach production:

```bash
#!/bin/bash
# ci-validate-glb.sh
set -e
for glb in /output/web-assets/*.glb; do
    result=$(npx gltf-validator "$glb" --format json)
    errors=$(echo "$result" | jq '.issues.numErrors')
    warnings=$(echo "$result" | jq '.issues.numWarnings')
    echo "$glb: $errors errors, $warnings warnings"
    if [ "$errors" -gt 0 ]; then
        echo "$result" | jq '.issues.messages'
        exit 1
    fi
done
```

### 17.6 Three.js Integration

Three.js loads GLB assets via `GLTFLoader`. For Draco-compressed assets, a `DRACOLoader` must be attached; for KTX2 textures, a `KTX2Loader`:

```javascript
import * as THREE from 'three'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js'

const renderer = new THREE.WebGLRenderer()

const dracoLoader = new DRACOLoader()
dracoLoader.setDecoderPath('/draco/')   // path to WASM decoders

const ktx2Loader = new KTX2Loader()
ktx2Loader.setTranscoderPath('/basis/') // path to Basis Universal WASM
ktx2Loader.detectSupport(renderer)      // choose GPU format (BC7, ASTC, ETC2, etc.)

const loader = new GLTFLoader()
loader.setDRACOLoader(dracoLoader)
loader.setKTX2Loader(ktx2Loader)

loader.load('/assets/model.glb', (gltf) => {
    scene.add(gltf.scene)

    // Play all animations
    const mixer = new THREE.AnimationMixer(gltf.scene)
    gltf.animations.forEach(clip => mixer.clipAction(clip).play())
})
```

`KHR_materials_emissive_strength` values above 1.0 require a Three.js `BloomPass` or `UnrealBloomPass` in the post-processing pipeline to render correctly — the raw emissive output is clamped to [0,1] by the default renderer.

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js'
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js'

const bloomPass = new UnrealBloomPass(resolution, strength=1.2, radius=0.4, threshold=0.8)
composer.addPass(bloomPass)
```

### 17.7 React Three Fiber and gltfjsx

React Three Fiber (R3F) wraps Three.js in a React component model. The `@react-three/drei` library provides `useGLTF`, a hook that wraps `GLTFLoader` with Suspense-compatible caching and preloading:

```tsx
// Basic usage
import { useGLTF } from '@react-three/drei'

function Model({ url = '/assets/robot.glb' }) {
  const { scene } = useGLTF(url)
  return <primitive object={scene} />
}

// Preload to avoid waterfall loading
useGLTF.preload('/assets/robot.glb')
```

`useGLTF` returns `{ scene, nodes, materials, animations }`. `nodes` is a flat map of all named objects in the glTF hierarchy keyed by name; `materials` is a map of all materials. These allow targeting specific parts of the scene without traversal:

```tsx
function Robot(props) {
  const { nodes, materials, animations } = useGLTF('/assets/robot.glb')
  const { actions } = useAnimations(animations, nodes.Armature)

  useEffect(() => {
    actions['Walk']?.play()
  }, [actions])

  return (
    <group {...props} dispose={null}>
      <skinnedMesh
        geometry={nodes.Body.geometry}
        material={materials.RobotMetal}
        skeleton={nodes.Body.skeleton}
      />
      <skinnedMesh
        geometry={nodes.Visor.geometry}
        material={materials.Glass}
        skeleton={nodes.Visor.skeleton}
      />
    </group>
  )
}
```

**`gltfjsx`** is a CLI tool that auto-generates a typed React component from a GLB file, producing exactly the pattern above without manual traversal:

```bash
# Install
npm install --global gltfjsx

# Generate JSX component
npx gltfjsx robot.glb --output Robot.jsx

# TypeScript component with types
npx gltfjsx robot.glb --output Robot.tsx --types

# With Draco decompression
npx gltfjsx robot.glb --output Robot.jsx --draco

# With instance deduplication (for repeated props/assets)
npx gltfjsx robot.glb --output Robot.jsx --instance
```

The generated component names mesh nodes after their Blender object names. Consistent naming in Blender (no spaces, using snake_case or PascalCase) produces clean component code. Blender's object naming convention becomes the React component's node API.

A complete pipeline from Blender to a deployed R3F component:

```bash
# 1. Export from Blender
blender -b scene.blend -P export_web_assets.py

# 2. Optimise
npx @gltf-transform/cli optimize robot.glb robot.opt.glb \
  --texture-compress webp --draco

# 3. Validate
npx gltf-validator robot.opt.glb

# 4. Generate R3F component
npx gltfjsx robot.opt.glb --output src/components/Robot.tsx --types --draco

# 5. Copy to public assets
cp robot.opt.glb public/assets/robot.glb
```

### 17.8 AI-Assisted Export Prompts

Claude Code with Blender MCP can drive the entire export preparation stage:

```
Prepare all mesh objects in the collection named "hero_props" for web export:
1. Apply all modifiers on each object.
2. Apply all transforms (location, rotation, scale).
3. Set origin to bounding box centre for each object.
4. Rename each object to lowercase with underscores (no spaces, no special chars).
5. Export each as a separate GLB to ~/web-assets/hero_props/:
   - Apply: true, Y-up: true
   - Draco compression level 5
   - WebP textures at quality 80
   - Include animations if the object has an armature
Report any failures and the file size of each exported GLB.
```

```
Check the current scene for web-export readiness:
- List any objects with unapplied scale (scale != (1,1,1)).
- List any objects with unapplied rotation.
- List any materials using nodes that do not map cleanly to Principled BSDF
  (e.g. custom node groups, shader mix chains without a Principled root).
- List any image textures that are missing or have dimensions over 2048px.
- Report total triangle count and flag any single object over 100,000 triangles.
```

```
The GLB exported from Blender looks wrong in Three.js:
the model is rotated 90 degrees. Diagnose:
- Check export_yup setting (should be True).
- Check if any root objects have unapplied X-rotation.
- Check if the armature root has a non-identity transform.
Provide the correct export command and any transform fixes needed.
```

---

## Integrations

**Chapter 42 — Blender GPU: Cycles and EEVEE** establishes the GPU rendering architecture (GPUBackend abstraction, VKBackend/Vulkan, Cycles multi-backend compute) that the Blender Python API scripts and MCP server operate on top of. Add-on scripts that modify materials interact with the same shader compilation pipeline described there.

**Chapter 94 — ComfyUI and ComfyScript** covers the node-graph AI image generation workflow that is the primary alternative to Dream Textures for diffusion-based texture generation on Linux. ComfyUI's scripting interface (ComfyScript) has a similar "AI-writes-Python" paradigm to the Blender MCP workflow described here.

**Chapter 115 — NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting** covers the open-source 3D reconstruction approaches (NeRF, 3DGS) that underpin some AI 3D generation tools. The NeRF representation is also the output format of Shap-E.

**Chapter 124 — Local LLM Inference on Linux GPUs** covers the model serving infrastructure (llama.cpp, Ollama, vLLM) that the official Blender MCP server targets when running without cloud AI subscriptions. Any local LLM exposed via an MCP-compatible endpoint can drive Blender MCP in place of Claude.

**Chapter 64 — glTF 2.0: The 3D Asset Pipeline Standard** covers the GLB/glTF format that Meshy, Hyper3D Rodin, Tripo3D, and all other AI generation platforms use as their primary output format. Understanding glTF PBR material structure is prerequisite to correctly importing generated assets into Blender's material system.

**Chapter 25 — GPU Compute** covers the CUDA/HIP/oneAPI compute stack that Cycles uses for path tracing and that OIDN GPU acceleration relies on. The same ROCm stack described there is the prerequisite for OIDN AMD GPU acceleration in Blender 5.x.

**Chapter 20 — Wayland Protocol Fundamentals** is relevant to Blender MCP in interactive mode: the `GHOST_SystemWayland` compositor client that Blender's viewport uses is what keeps the Blender event loop running. When Blender is started headless with `--background`, this event loop is suppressed, which requires the MCP addon to use `bpy.app.timers` rather than relying on frame events from the compositor.

---

## References

- [GitHub — ahujasid/blender-mcp](https://github.com/ahujasid/blender-mcp) — Primary community Blender MCP server (MIT, 24k+ stars)
- [Blender MCP Server — Blender Foundation](https://www.blender.org/lab/mcp-server/) — Official Blender Lab project, v1.0.0 April 2026
- [projects.blender.org/lab/blender_mcp](https://projects.blender.org/lab/blender_mcp) — Blender Foundation source repository
- [Model Context Protocol specification](https://modelcontextprotocol.io) — Wire format, tool/resource schema
- [FastMCP — Python SDK](https://gofastmcp.com) — FastMCP library used by blender-mcp server.py
- [blender-mcp on PyPI](https://pypi.org/project/blender-mcp/) — `uvx blender-mcp` installation
- [Blender Python API Overview](https://docs.blender.org/api/current/info_overview.html) — Official bpy module documentation
- [bpy API reference — Blender 5.2](https://docs.blender.org/api/current/) — Auto-generated from RNA
- [bpy standalone on PyPI](https://pypi.org/project/bpy/) — Server-side bpy without Blender UI
- [RNA — Blender Developer Documentation](https://developer.blender.org/docs/features/core/rna/) — DNA/RNA system design
- [Blender Extensions system — 4.2](https://developer.blender.org/docs/release_notes/4.2/extensions/) — blender_manifest.toml, dependency wheels
- [bpy.app.timers — API reference](https://docs.blender.org/api/current/bpy.app.timers.html) — Main-thread callback scheduling
- [bpy.types.Operator — API reference](https://docs.blender.org/api/current/bpy.types.Operator.html) — Operator lifecycle: poll/invoke/execute/modal
- [GitHub — glonorce/Blender_mcp](https://github.com/glonorce/Blender_mcp) — 69-tool community implementation with 499 tests
- [GitHub — RFingAdam/mcp-blender](https://github.com/RFingAdam/mcp-blender) — 218-tool implementation for Blender 4.2/5.0
- [Meshy AI — Text to 3D API](https://docs.meshy.ai/en/api/text-to-3d) — REST API reference, parameters, status codes
- [Meshy AI — Authentication](https://docs.meshy.ai/en/api/authentication) — API key setup
- [Meshy AI — Integrations](https://www.meshy.ai/integrations) — Blender plugin, Unity, Unreal, Omniverse
- [Hyper3D Rodin API specification](https://developer.hyper3d.ai/api-specification/rodin-generation) — Generation tiers, HighPack textures
- [fal.ai — Hyper3D Rodin](https://fal.ai/models/fal-ai/hyper3d/rodin/api) — FAL.ai proxy endpoint
- [Tripo3D API](https://www.tripo3d.ai/api) — TripoSR-based commercial API
- [tripo3d Python SDK on PyPI](https://pypi.org/project/tripo3d/) — `pip install tripo3d`
- [CSM.ai](https://csm.ai) — Physics-accurate 3D generation
- [Sloyd.ai](https://www.sloyd.ai) — Procedural game-asset generation
- [GitHub — carson-katri/dream-textures](https://github.com/carson-katri/dream-textures) — Stable Diffusion in Blender (8.2k stars, GPL-3.0)
- [GitHub — benrugg/AI-Render](https://github.com/benrugg/AI-Render) — SD img2img render post-processing
- [openai/point-e on GitHub](https://github.com/openai/point-e) — Text → 3D point cloud, December 2022
- [openai/shap-e on HuggingFace](https://huggingface.co/openai/shap-e) — Text/image → implicit neural representation, May 2023
- [InstantMesh paper — arXiv:2404.07191](https://arxiv.org/abs/2404.07191) — Multi-view LRM reconstruction
- [Stable Fast 3D](https://stable-fast-3d.github.io) — Stability AI sub-1-second image-to-3D
- [TRELLIS — Microsoft Research](https://github.com/microsoft/TRELLIS) — Sparse voxel + Rectified Flow reconstruction
- [Intel OpenImageDenoise](https://openimagedenoise.github.io) — Open-source ML denoiser (OIDN 2.0, multi-GPU)
- [NVIDIA OptiX AI-Accelerated Denoiser](https://developer.nvidia.com/optix-denoiser) — RT Core hardware denoising
- [Denoise Node — Blender 5.2 Manual](https://docs.blender.org/manual/en/latest/compositing/types/filter/denoise.html) — Compositor denoising node
- [Blender Lab Activity Report Q1 2026](https://www.blender.org/development/blender-lab-activity-report-q1-2026/) — Official Blender Foundation AI research update
- [Blender glTF 2.0 exporter — Blender Manual](https://docs.blender.org/manual/en/latest/addons/import_export/scene_gltf2.html) — io_scene_gltf2 parameters, KHR extension coverage
- [glTF 2.0 specification — Khronos Group](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html) — Wire format, PBR material schema
- [KHR_draco_mesh_compression](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_draco_mesh_compression) — Draco geometry compression extension
- [KHR_materials_emissive_strength](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_materials_emissive_strength) — HDR emission for bloom
- [Three.js GLTFLoader](https://threejs.org/docs/#examples/en/loaders/GLTFLoader) — Three.js loader, DRACOLoader, KTX2Loader wiring
- [React Three Fiber — @react-three/drei useGLTF](https://drei.docs.pmnd.rs/loaders/use-gltf) — Suspense-compatible GLTF hook
- [gltfjsx — pmndrs](https://github.com/pmndrs/gltfjsx) — GLB → typed React/JSX component generator
- [@gltf-transform/cli](https://gltf.report/transform) — Draco, meshopt, KTX2/WebP post-processing pipeline
- [gltf-validator — Khronos Group](https://github.com/KhronosGroup/glTF-Validator) — Spec-conformance validation tool
