# Chapter 205: AI-Driven 3D Creation — Blender MCP, Claude Code, and Generative Tools

> **Part**: Part XI — Engines and Creative Tools
> **Audience**: Graphics application developers and technical artists who want to integrate AI language models and generative tools into Blender workflows; systems developers building headless Blender pipelines; tooling engineers embedding bpy in web services or CI pipelines
> **Status**: First draft — 2026-07-18

The emergence of the Model Context Protocol (MCP) has transformed how AI coding assistants interact with creative applications. Where previous integrations required manual copy-paste of Python snippets into Blender's script console, MCP enables a live bidirectional channel between Claude Code (or any MCP-capable agent) and a running Blender session: the AI can inspect the scene, execute operations, and observe results in a tight feedback loop. In parallel, a new generation of AI generative tools — text-to-3D, image-to-3D, AI texturing, AI rigging — has matured to the point where production-ready assets can be generated in seconds.

This chapter examines the full stack: the two-component Blender MCP bridge and its thread-safety architecture; how Claude Code reasons about Blender's Python API; the generative 3D ecosystem from Meshy to open-source models; AI-assisted denoising, texturing, and rigging within Blender; and how headless Blender fits into automated Linux pipelines.

---

## Table of Contents

- [1. The Model Context Protocol and Blender](#1-the-model-context-protocol-and-blender)
  - [1.1 What is the Model Context Protocol (MCP)?](#11-what-is-the-model-context-protocol-mcp)
  - [1.2 What is Blender's Python API (bpy)?](#12-what-is-blenders-python-api-bpy)
  - [1.3 What is Claude Code?](#13-what-is-claude-code)
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
- [18. Roadmap](#18-roadmap)
- [Integrations](#integrations)
- [References](#references)

---

## 1. The Model Context Protocol and Blender

MCP, published by Anthropic in late 2024 and now governed as an open specification at [modelcontextprotocol.io](https://modelcontextprotocol.io), defines a JSON-RPC-over-stdio (or SSE) wire format for connecting AI agents to *resources* and *tools* exposed by a server. A tool is a named function with a JSON Schema for its parameters; the agent calls it and receives structured output. Resources are read-only data blobs (files, database rows, live sensor readings) the agent can include in its context window.

For Blender, MCP addresses a fundamental ergonomic problem: Blender's entire UI — every menu item, slider, and button — is represented as a Python operator call. A capable agent with access to the operator reference can write correct bpy scripts, but without live scene feedback it cannot inspect what is actually present, verify results, or react to errors. MCP closes that loop.

**Why MCP over a simple subprocess pipe?** The MCP specification standardises tool discovery (the client asks the server for its tool list at connection time), error reporting (tools return structured `isError` responses), and sampling (the client can ask the server to run LLM completions). This standardisation means a single Blender MCP server works unmodified with Claude Desktop, Claude Code, Cursor, VS Code with Copilot MCP extensions, or any open-source MCP client.

### 1.1 What is the Model Context Protocol (MCP)?

The Model Context Protocol (MCP) is an open specification, published by Anthropic in late 2024 and now maintained at [modelcontextprotocol.io](https://modelcontextprotocol.io), that standardises how AI language model agents communicate with external tools and data sources. At its core, MCP defines a JSON-RPC-over-stdio (or Server-Sent Events) wire format with three primitives: tools (callable functions with typed JSON Schema parameters and structured return values), resources (read-only named data blobs the agent can pull into its context window), and prompts (server-defined prompt templates). A conforming MCP server advertises its tool list at connection time, so any MCP-capable client — Claude Code, Claude Desktop, Cursor, VS Code with Copilot extensions, or open-source alternatives — can discover and invoke those tools without bespoke integration code for each client.

In the Linux graphics and creative-tools context, MCP solves the impedance mismatch between a language model that reasons about code and an application whose internal state is only accessible at runtime. For Blender in particular, a script written without live scene feedback cannot know the names of existing objects, the state of materials, or the geometry counts produced by a modifier stack. MCP provides a bidirectional channel so the AI can inspect the scene, execute operations via Blender's Python API, and observe results in a tight reasoning loop without any manual copy-paste between the agent and Blender's script console. The specification is intentionally application-agnostic: the same protocol used to connect an agent to Blender is used to connect it to databases, version-control systems, browser automation engines, and Linux system tools.

### 1.2 What is Blender's Python API (bpy)?

Blender exposes almost its entire feature set through an embedded CPython interpreter and a module called `bpy`. Every mesh operation, object property, shader node, render setting, and UI panel accessible through Blender's graphical interface corresponds to a Python operator, property, or data accessor reachable through `bpy`. The module is divided into four main namespaces: `bpy.data` (the Blender internal database — all scenes, objects, meshes, materials, and other data-blocks); `bpy.context` (the current active object, mode, scene, and selection state); `bpy.ops` (operator calls that mirror menu actions and are undo-aware); and `bpy.types` (the RNA type system that defines the schema of every data-block, used for property registration and introspection).

This architecture means that any Blender workflow automatable through the GUI is also automatable through Python — which is what makes an AI agent capable of driving Blender programmatically, given only the API reference as context. A critical constraint is that `bpy` is only importable from within Blender's embedded interpreter; it cannot be imported from an external Python process. This is the fundamental architectural constraint that necessitates the two-component MCP bridge described in §2: a Blender addon runs inside Blender's interpreter to access `bpy`, while a separate MCP server process handles the protocol conversation with the AI client and relays commands to the addon over a local TCP socket.

### 1.3 What is Claude Code?

Claude Code is Anthropic's terminal-based AI coding assistant, distributed as the `claude` command-line tool. Unlike chat-oriented interfaces, Claude Code has direct access to the local filesystem, can execute shell commands, read compiler and interpreter output, and maintain multi-step task context across an entire implementation session. It functions as a first-class MCP client: when MCP server entries are declared in its configuration (`~/.claude/settings.json` or a project-level `.claude/settings.json`), Claude Code discovers and calls those servers' tools as part of its normal reasoning loop — alongside filesystem reads, shell invocations, and web searches — without requiring the user to manually relay outputs.

In the context of this chapter, Claude Code acts as the orchestrating agent that drives Blender through the MCP bridge. Given a natural-language description of a 3D modelling or rendering task, Claude Code queries the scene state via MCP inspection tools, formulates `bpy` Python code, executes it through the `execute_blender_code` tool, and iterates based on the returned output and viewport screenshots. This closed-loop workflow eliminates the context-switching between an editor and Blender's script console that characterised earlier AI-assisted Blender approaches, and it makes the full Blender operator and API surface available to the agent without any Blender-specific fine-tuning.

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

> **Which server should you use?** The two implementations solve overlapping but not identical problems, and picking wrong costs a re-install later.
>
> | | `ahujasid/blender-mcp` | Blender Foundation `lab/blender_mcp` |
> |---|---|---|
> | Licence | MIT | GPL-3.0-or-later ([`blender_manifest.toml`](#72-extensions-blender-42), §7.2) |
> | Distribution | classic addon, manual `.zip` install or `uvx blender-mcp` | Blender Extension, `extensions.blender.org` |
> | Minimum Blender | 3.6 LTS and up | 5.1+ |
> | Scene inspection / `execute_blender_code` / viewport screenshot | yes | yes |
> | Tool surface | flat, 16 tools (§3) | narrower, inspection-first — read-only tools plus one write path (§3.1) |
> | API docs grounded in bundled reference (anti-hallucination) | no | yes, `search_api_docs`/`get_python_api_docs`/`search_manual_docs` (§3.1) |
> | PolyHaven HDRIs & textures, Sketchfab import, Hyper3D Rodin / Hunyuan3D generation (§7, §9.3) | yes | no |
> | Auto-start TCP server on Blender launch | no (manual "Start Server" click) | yes, via an Auto Start preference (§14) |
> | Maintainer | independent community project, 24,000+ stars | Blender Foundation (Blender Lab, incubating project) |
>
> Default to the **official server** if you're already on Blender 5.1+ and your workflow is the inspect-act-verify loop this chapter builds around (§3.1, §8): it has direct upstream backing, so it will track future `bpy` API changes without waiting on a third party, and its documentation tools are grounded in Blender's own bundled reference rather than the model's memory. Reach for **`ahujasid/blender-mcp`** instead if you need the generative-asset tools in §7/§9.3 (PolyHaven/Sketchfab/Hyper3D), or if you're pinned to a Blender version below 5.1. The two can coexist — they claim the same TCP port (9876), so only run one addon's server at a time. Both grant the AI arbitrary Python execution inside Blender via `execute_blender_code`; treat that tool the same way regardless of which server exposes it (§15).

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

The official Blender Foundation server has the same shape — a FastMCP-style process that speaks JSON-RPC to the client and TCP to the addon on the same `localhost:9876` port — but a different distribution mechanism. Instead of a PyPI package fetched by `uvx`, it ships as an `.mcpb` bundle (Anthropic's packaging format for one-step MCP server installation) alongside the Blender Extension ZIP, so there is no separate `server.py` for a user to invoke manually.

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

The official server skips manual JSON entirely for Claude Desktop users: **Settings → Connectors → Add Connector**, then search "Blender" (or drag the downloaded `.mcpb` file onto the Connectors pane), installs and wires up the server in one step. On the Blender side, install the extension from [blender.org/lab/mcp-server](https://www.blender.org/lab/mcp-server/) — either directly from Blender's extension-acquisition screen or via "Install from Disk" for the downloaded ZIP — and enable the **Auto Start** preference (§14) so the TCP listener comes up with Blender itself. *Note: needs verification* — the exact `mcpServers` entry for pointing Claude Code (rather than Claude Desktop) at the official server could not be independently confirmed for this chapter; `projects.blender.org`'s own setup docs return HTTP 403 to automated fetching, so treat the Claude Code path as provisional until checked against the extension's bundled README.

---

## 3. MCP Tool Surface: What the AI Sees

The base ahujasid/blender-mcp server exposes 16 tools, grouped into functional categories. Three of those categories wrap third-party asset services rather than Blender itself, and are worth defining before reading the table: [Poly Haven](https://polyhaven.com) is a CC0-licensed library of HDRIs, PBR texture sets, and models, used here to pull in ready-made environment lighting and materials; [Sketchfab](https://sketchfab.com) is a hosting marketplace for pre-made 3D models (free and paid) that the AI can search and import directly into the scene; and Hyper3D Rodin (§9.3) is a generative model — from Deemos/Hyper3D — that produces a new mesh from a text prompt or reference image (text-to-3D / image-to-3D) rather than retrieving an existing one. In short, Poly Haven and Sketchfab *retrieve* assets someone else made; Hyper3D Rodin *generates* one from scratch.

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

### 3.1 The Official Server's Tool Surface

The Blender Foundation server takes a narrower, inspection-first approach instead of chasing tool count:

| Tool | Category | Description |
|---|---|---|
| `get_objects_summary` | Inspection | Lightweight list of objects in the current scene |
| `get_object_detail_summary` | Inspection | Deep detail on one object: transform, modifiers, materials, constraints |
| `get_blendfile_summary_datablocks` | Inspection | Per-type data-block counts (meshes, materials, textures, …) |
| `get_blendfile_summary_usage_guess` | Inspection | Heuristic guess at what the file is for (modelling, animation, VFX, …) |
| `search_api_docs` / `get_python_api_docs` | Documentation | Look up a `bpy` identifier, or list modules matching a pattern |
| `search_manual_docs` | Documentation | Search the Blender user manual |
| `get_screenshot_of_window_as_image` / `get_screenshot_of_area_as_image` | Inspection | PNG capture of the whole window or one editor area |
| `jump_to_tab_by_name` / `jump_to_tab_by_space_type` / `jump_to_view3d_object_by_name` | Navigation | Change the active workspace tab or focus an object in the viewport, mirroring what a human would click |
| `execute_blender_code` | Execution | The one write path: run arbitrary `bpy` Python, same role as the community server's tool of the same name |

Every tool above `execute_blender_code` is read-only, which is a deliberate design choice rather than an oversight: the server is built around an inspect-first loop where the AI is expected to call `get_objects_summary`/`get_object_detail_summary` and `search_api_docs`/`get_python_api_docs` *before* writing any code, and the documentation tools are grounded in Blender's actual bundled RST reference rather than the model's training data — a hedge against recommending an operator that was renamed or removed since the model's knowledge cutoff. There is no PolyHaven/Sketchfab/Hyper3D-equivalent asset integration; scene population from external sources is out of scope for v1.0.0.

*Note: needs verification* — this table is reconstructed from third-party integration guides that document the official server's tool names, not from the server's own source or changelog: `projects.blender.org` returns HTTP 403 to automated fetching, so the exact tool count, full parameter schemas, and any additional write-capable tools beyond `execute_blender_code` should be checked against the extension itself before being treated as authoritative.

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

### 4.1 Is a Thread-Safe bpy Coming?

No. There is no known Blender Foundation roadmap item to make `bpy`/RNA safe for concurrent access from multiple threads, and the constraint is unlikely to be lifted soon. Blender's own documentation states the rule without qualification: while a background thread is running, *no* code — not even the main thread — may call any `bpy`/Blender API function; only plain Python and non-Blender third-party modules are safe to touch from a worker thread ([Python Threads are Not Supported](https://docs.blender.org/api/current/info_gotchas_threading.html)).

The root cause is architectural, not an oversight waiting to be patched: RNA property updates run through a global `bContext` pointer that reflects whatever the UI happens to be doing at that instant, and the DNA structs backing `bpy.data` are mutated lock-free on the assumption of single-threaded main-loop access — a bug class first reported in 2010 ([`developer.blender.org/T23401`](https://developer.blender.org/T23401)) and still open. The GIL doesn't help: it prevents two Python bytecodes from interleaving, but does nothing to stop a C struct being read while half-written by a concurrent caller. Community pressure is long-standing — a devtalk thread titled "Multithreading support (please :))" has run for years past 1000 replies — but volume of requests has not translated into a funded fix at the RNA/bpy level.

Multithreading does exist deep inside Blender's C core — depsgraph evaluation, modifier stacks, Cycles path tracing, and compositor tiles all run on Blender's internal `BLI_task` thread pool — but none of it is reachable from, or exposes concurrent write access to, Python. Scripts still hit `bpy.data` strictly serially through the main thread, which is exactly why the timer-queue pattern above exists.

One development worth watching rather than relying on: Python 3.14's free-threaded ("no-GIL") build reached officially supported status in 2026 (PEP 779), and Blender 5.x already ships Python 3.13. Removing CPython's GIL only removes *Python-level* serialization, though — it does nothing for Blender's unlocked C data structures, and could plausibly make latent races easier to trigger rather than fixing them, since the GIL previously hid many of them by accident.

More concretely, Blender does not pin its own Python version independently — it tracks the [VFX Reference Platform](https://vfxplatform.com), an industry-wide coordinated dependency stack that the table in §5 (Python 3.13 = "VFX Platform 2026") already reflects. The VFX Platform 2027 proposal, discussed on a devtalk thread titled ["VFX Platform to stay on Python 3.13 in 2027 — Reasons to try to request 3.14 instead?"](https://devtalk.blender.org/t/vfx-platform-to-stay-on-python-3-13-in-2027-reasons-to-try-to-request-3-14-instead/44974), stays on Python 3.13 rather than moving to 3.14 — for a reason unrelated to threading: 3.14 would pull in PySide 6.10 and therefore Qt 6.10, and the coordinating studios were not ready to make that Qt jump for the 2027 cycle. Since Blender's Python version is downstream of this industry-wide pin, staying on 3.13 through the 2027 platform cycle means Blender does not gain *access* to Python 3.14's free-threaded build in that window at all — independent of whether `bpy`'s C-level thread-unsafety (above) would make it usable even if it did. *Note: needs verification* — this is reconstructed from search-engine snippets of the devtalk thread, not the thread's full text; `devtalk.blender.org` returns HTTP 403 to automated fetching, so the exact wording and any counter-arguments raised in the thread's replies should be checked directly before treating the 2027-cycle timeline as final.

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

#### 5.1.1 bpy.ops — Operators

Every entry under `bpy.ops.<namespace>.<name>(...)` wraps a C `wmOperatorType` and returns a Python `set` of result flags — `{'FINISHED'}`, `{'CANCELLED'}`, or `{'RUNNING_MODAL'}` for modal operators. Keyword arguments map directly to the operator's `bpy.props`-declared properties, so `bpy.ops.mesh.subdivide(number_cuts=2)` is equivalent to running Subdivide from the menu and typing `2` into the redo panel. Blender 4.0 replaced the old dict-based context override (`bpy.ops.object.delete({'selected_objects': [...]})`) with a context manager:

```python
import bpy

with bpy.context.temp_override(selected_objects=[obj], active_object=obj):
    bpy.ops.object.delete()
```

`temp_override()` is required whenever an operator needs a context attribute (an active object, an area of a particular editor type) that doesn't match whatever window/area happened to be focused when the MCP addon's TCP handler ran the command — a frequent source of `poll() failed` errors in AI-driven sessions, since the addon executes on Blender's main thread with whatever context existed at that instant (§4).

#### 5.1.2 bpy.data — Data-Blocks

`bpy.data` exposes one `bpy_prop_collection` per ID type — `bpy.data.objects`, `.meshes`, `.materials`, `.scenes`, `.images`, `.collections`, `.node_groups`, and more. Each collection supports name-based lookup (`bpy.data.objects["Cube"]`), iteration, and `.new()`/`.remove()` for creating or deleting data-blocks directly, bypassing the operator layer entirely:

```python
mesh = bpy.data.meshes.new("Plate")
obj  = bpy.data.objects.new("Plate", mesh)
bpy.context.collection.objects.link(obj)   # data-blocks exist independently of any scene until linked
```

Creating a data-block with `bpy.data.*.new()` does not make it visible anywhere — an `Object` must be linked into a `Collection`, and a `Collection` into the active `Scene`'s collection hierarchy, before it renders or appears in the outliner. This two-step create-then-link pattern is the most common mistake in AI-generated `bpy` code that "runs without error but nothing shows up."

#### 5.1.3 bpy.context — Editor State

`bpy.context` is a read-mostly view of whatever window, screen, area, and mode is currently active: `context.object`/`context.active_object`, `context.selected_objects`, `context.scene`, `context.view_layer`, `context.mode`. It reflects live UI state, not a stable snapshot — the same expression can return different values between two calls if the user (or a previous MCP command) changed the active object in between.

The important gotcha for pipeline work (§14): when Blender runs with `--background`, there is no window, so most `context.window`/`context.area`/`context.screen`-dependent attributes are `None` and any operator whose `poll()` checks them fails immediately. Headless scripts must read and write through `bpy.data` and `bpy.context.view_layer` directly rather than relying on `bpy.ops` calls that assume an interactive editor is present.

#### 5.1.4 bpy.types — RNA Type Hierarchy

`bpy.types` is where every RNA-registered class lives, both Blender's built-ins (`bpy.types.Object`, `bpy.types.Mesh`, `bpy.types.Scene`) and the base classes an add-on subclasses to extend the UI: `Operator`, `Panel`, `PropertyGroup`, `Menu`, `UIList`, `AddonPreferences`. §7.1 walks through a complete `Operator`/`Panel`/`PropertyGroup` example with registration; the key point for MCP-driven code is that any new class must be registered with `bpy.utils.register_class()` (§5.1.6) before Blender's UI or `bpy.ops` can see it — defining a class is not enough.

#### 5.1.5 bpy.props — Property Factory Functions

`bpy.props` provides the factory functions used as class-body annotations on `Operator`, `PropertyGroup`, and `bpy.types.Scene`/`bpy.types.Object` extensions: `BoolProperty`, `IntProperty`, `FloatProperty`, `StringProperty`, `EnumProperty`, `PointerProperty` (a reference to another ID or `PropertyGroup`), and `CollectionProperty` (an ordered list of `PropertyGroup` instances). Each accepts `name`, `description`, `default`, and type-specific bounds (`min`/`max`/`subtype` for numeric types — `subtype='DISTANCE'` or `'COLOR'` changes both UI widget and unit display without changing the stored value). `PointerProperty`/`CollectionProperty` are what let an add-on attach custom, undo-aware, UI-editable state to Blender's own data-blocks, as in the `bpy.types.Scene.my_addon = bpy.props.PointerProperty(...)` line in §7.1 — a plain Python attribute assigned to `bpy.types.Scene` would not survive a `.blend` file save/reload or appear in the undo stack.

#### 5.1.6 bpy.utils — Registration and Support Utilities

`bpy.utils.register_class()`/`unregister_class()` are the calls that make a Python class visible to Blender's operator search, UI panels, and `bpy.ops` namespace — every add-on's `register()`/`unregister()` functions (§7.1) exist to wrap these two calls for a whole set of classes in one place. Beyond registration, `bpy.utils` carries a grab-bag of infrastructure an add-on needs but that doesn't belong in `bpy.data` or `bpy.ops`: `bpy.utils.previews` manages custom icon thumbnails for panels, `bpy.utils.resource_path()` returns Blender's config/scripts/system directories, and `bpy.utils.register_manual_map()` lets an add-on hook its own documentation into Blender's "online manual" right-click menu entries.

#### 5.1.7 bpy.app — Application State

`bpy.app` is read-only application metadata and the two extension points used throughout this chapter: `bpy.app.timers` (§4) for scheduling main-thread callbacks from a background socket thread, and `bpy.app.handlers` — lists of callbacks such as `depsgraph_update_post`, `load_post`, and `save_pre` that Blender invokes at specific lifecycle events, used by add-ons that need to react to scene changes rather than poll for them. `bpy.app.version`, `bpy.app.version_string`, and `bpy.app.binary_path` are the standard way for a script to branch on the running Blender version rather than assuming one, which matters given how much of the RNA/`bpy` surface changes between the versions in the table above.

#### 5.1.8 bpy.path — Path Utilities

A small module of path-normalisation helpers tailored to Blender's on-disk conventions: `bpy.path.abspath()` resolves Blender's `//`-prefixed relative paths (relative to the current `.blend` file, not the process's working directory) to absolute ones, `bpy.path.basename()` and `bpy.path.ensure_ext()` handle filenames, and `bpy.path.clean_name()` sanitises arbitrary strings into names safe for data-block IDs. Any MCP tool that accepts a filesystem path from the AI and later passes it to `bpy.ops.export_scene.*`/`import_scene.*` should round-trip it through `bpy.path.abspath()` first, since Blender treats `//textures/wood.png` as a valid, portable path relative to the saved file.

### 5.2 C Extension Modules

| Module | Role |
|---|---|
| `mathutils` | `Vector`, `Matrix`, `Quaternion`, `Euler` — thin wrappers over C math |
| `bmesh` | Low-level mesh editing: create/modify vertices, edges, faces, loops |
| `gpu` | Shader programs, compute shaders, framebuffer management (GPU context required) |
| `imbuf` | Pixel-level image manipulation |
| `idprop` | Custom property data-blocks (arbitrary key-value per data-block) |

#### 5.2.1 mathutils — Vector Math

`Vector`, `Matrix`, `Quaternion`, and `Euler` are thin Python wrappers over Blender's internal C math library, used everywhere transforms are read or written:

```python
from mathutils import Vector, Matrix, Euler
import math

v = Vector((1.0, 0.0, 0.0))
rot = Euler((0, 0, math.radians(90)), 'XYZ').to_matrix().to_4x4()
v_rotated = rot @ v                       # matrix/vector multiplication uses @, not *

obj.matrix_world = Matrix.Translation((0, 0, 2)) @ rot
```

`obj.location`, `obj.rotation_euler`, and `obj.matrix_world` all return `mathutils` types rather than plain tuples, so code that reads them back gets `Vector`/`Euler`/`Matrix` objects with the full set of arithmetic operators (`@` for matrix/vector and matrix/matrix products, mirroring NumPy's convention) rather than needing manual trigonometry.

#### 5.2.2 bmesh — Mesh Editing

`bmesh` is the editable half-edge mesh representation Blender's own mesh tools operate on internally; §5.4 already shows the `bm.verts`/`from_mesh`/`to_mesh` round-trip for direct vertex edits. Beyond raw vertex/edge/face access, `bmesh.ops` exposes the same primitive operations the mesh operators (§5.3.1) are built from — `bmesh.ops.subdivide_edges()`, `bmesh.ops.bevel()`, `bmesh.ops.triangulate()` — callable on a `BMesh` without going through `bpy.ops` and its context/`poll()` requirements at all, which makes `bmesh` the preferred tool for procedural mesh generation scripts that build geometry from scratch rather than editing an existing selection.

#### 5.2.3 gpu — Custom Drawing and Compute

The `gpu` module wraps Blender's GPU backend (OpenGL, Vulkan, or Metal depending on platform) for custom viewport drawing and GPU compute, independent of the render engine. A `GPUShader` pairs a vertex and fragment (and optional geometry) program; Blender ships built-in shaders — `'UNIFORM_COLOR'`, `'FLAT_COLOR'`, `'POLYLINE_UNIFORM_COLOR'` — for common cases so a script rarely needs to write GLSL by hand:

```python
import gpu
from gpu_extras.batch import batch_for_shader

coords = [(1, 1, 1), (-2, 0, 0), (-2, -1, 3), (0, 1, 1)]
shader = gpu.shader.from_builtin('UNIFORM_COLOR')
batch  = batch_for_shader(shader, 'LINE_STRIP', {"pos": coords})

def draw():
    shader.uniform_float("color", (1, 1, 0, 1))
    batch.draw(shader)

bpy.types.SpaceView3D.draw_handler_add(draw, (), 'WINDOW', 'POST_VIEW')
```

`gpu.types.GPUOffScreen` renders to a texture rather than the visible viewport, which is how the `get_viewport_screenshot`/`get_screenshot_of_*` MCP tools (§3, §3.1) capture pixels without depending on window compositing — and, per §14.2, why headless rendering on Linux still needs a virtual display or EGL context even though nothing is shown on a physical screen.

#### 5.2.4 imbuf — Image Buffers

`imbuf` operates one level below `bpy.data.images`: it is the pixel-buffer type (`ImBuf`) that backs `bpy.types.Image` at runtime, exposed for scripts that want to load, generate, or write image data without creating a full `Image` data-block. `imbuf.load(filepath)` reads a file into an `ImBuf`; `imbuf.new(size, planes=32, buffer_type='FLOAT')` allocates a blank one; `imbuf.write(image, filepath=...)` writes it back out. Pixel data is reachable as a `memoryview` via the buffer's pixel-access context manager, which is significantly faster than iterating `Image.pixels` (a `bpy_prop_array` with per-element RNA overhead) for anything larger than a few thousand pixels — relevant to any AI-driven texture post-processing that touches every pixel rather than a handful of parameters.

#### 5.2.5 idprop — Custom Properties

`idprop` is the C extension backing arbitrary custom properties on any ID data-block — the `obj["my_key"] = 1.0` syntax available on every `Object`, `Mesh`, `Scene`, and so on, independent of the `bpy.props`/`PropertyGroup` system in §5.1.5. Where `PropertyGroup` requires a registered Python class, `idprop`-backed custom properties are schema-free key/value pairs (`int`, `float`, `str`, or nested arrays/groups) attached at runtime, which is how glTF exporters (§17) and many generative-asset pipelines round-trip metadata — provenance tags, generation seeds, source-asset IDs — through a `.blend` file without needing an add-on installed to read them back.

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

#### 5.3.1 bpy.ops.mesh and bpy.ops.object — Modelling

Mesh operators only work in Edit Mode on the active mesh's selection; object operators work in Object Mode on the selected objects. AI-generated modelling code almost always needs an explicit mode switch between the two:

```python
bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.mesh.select_all(action='SELECT')
bpy.ops.mesh.subdivide(number_cuts=2)
bpy.ops.mesh.inset_faces(thickness=0.05)
bpy.ops.object.mode_set(mode='OBJECT')

bpy.ops.object.duplicate()
bpy.ops.object.transform_apply(location=True, rotation=True, scale=True)
```

`transform_apply` is the operator equivalent of "Apply" in the Object menu — it bakes the object's transform into the mesh data and resets `location`/`rotation_euler`/`scale` to identity, which matters for any downstream export (§17) or physics setup that assumes unit scale.

#### 5.3.2 bpy.ops.material and bpy.ops.node — Shading

`bpy.ops.material` manages material slots on an object; the shader graph itself is edited through `bpy.ops.node` when working interactively, though §7.3's `node_tree.nodes.new()`/`links.new()` pattern is more reliable for AI-generated code since it doesn't depend on a Shader Editor area being the active context:

```python
bpy.ops.object.material_slot_add()
bpy.ops.material.new()
obj.material_slots[-1].material.name = "Generated_Material"
```

#### 5.3.3 bpy.ops.render — Rendering

A single call renders and optionally writes the result to `scene.render.filepath`:

```python
scene = bpy.context.scene
scene.render.filepath = "//renders/frame_001.png"
scene.render.image_settings.file_format = 'PNG'
bpy.ops.render.render(write_still=True)          # single frame
bpy.ops.render.render(animation=True)             # scene.frame_start .. frame_end
```

`write_still=True` is required to actually save the image — without it, `render()` produces the result in Blender's in-memory render buffer (viewable in the Image Editor) but writes nothing to disk, a frequent source of "the render ran but no file appeared" reports from AI-driven pipelines.

#### 5.3.4 bpy.ops.import_scene and bpy.ops.export_scene — Interchange

Format-specific operators, one function per format, taking a `filepath` and format-specific keyword arguments:

```python
bpy.ops.export_scene.gltf(filepath="//export/model.glb", export_format='GLB')
bpy.ops.export_scene.fbx(filepath="//export/model.fbx", use_selection=True)
bpy.ops.import_scene.gltf(filepath="//assets/downloaded.glb")
```

This is the operator family the community server's `download_polyhaven_asset`/`download_sketchfab_model` tools (§3) call internally after fetching a file, and the one Chapter 64's glTF pipeline discussion (§17) builds on for exporting AI-generated scenes to Three.js/React Three Fiber.

#### 5.3.5 bpy.ops.action and bpy.ops.pose — Animation and Rigging

`bpy.ops.action` inserts and manages keyframes on the active object's action; `bpy.ops.pose` operates on an armature's pose bones in Pose Mode:

```python
obj   = bpy.context.active_object
scene = bpy.context.scene
obj.location = (0, 0, 0)
scene.frame_set(1)
bpy.ops.anim.keyframe_insert(data_path="location")

scene.frame_set(24)
obj.location = (0, 0, 2)
bpy.ops.anim.keyframe_insert(data_path="location")

bpy.ops.object.mode_set(mode='POSE')
bpy.ops.pose.select_all(action='SELECT')
bpy.ops.pose.rot_clear()
```

For keyframing simple properties, `obj.keyframe_insert(data_path="location", frame=24)` — a method on the data-block itself rather than an operator call — avoids the context/mode requirements entirely and is generally preferred in generated scripts for the same reason direct `bpy.data` access is preferred over `bpy.ops` in §5.4.

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

### 6.1 Are There Alternatives to bpy?

Nothing that replaces `bpy`, but several projects route around it for specific tasks:

- **Rust via PyO3.** Patterns like `rust_extension_api` wrap Rust logic into a Python wheel that Blender's extension system loads and that calls back into `bpy` for UI registration and data access. This substitutes the *implementation language* for performance-sensitive logic; it still goes through `bpy` for anything Blender-facing and does not bypass the GIL or the thread-safety constraints in §4.1.
- **`bpy` as a standalone PyPI module** (§5) is a different embedding of the same API — useful for CI and server-side scripting without a Blender install, but not an alternative API surface.
- **Pure-Python/Rust `.blend` file parsers that skip `bpy` entirely** — `tinyblend`, `blender-file-reader`, and the Rust crate `lukebitts/blend` — read (rarely write) `.blend` files directly from their on-disk DNA layout without launching Blender. None supports compressed `.blend` files or full write-back, so they work as asset-inspection tooling rather than a general substitute for scripting inside a running Blender.
- **Geometry Nodes** is arguably Blender's own real answer to `bpy`'s limits: node graphs are evaluated through the multithreaded depsgraph in C, not single-threaded Python, so procedural work expressed as nodes sidesteps both the GIL and the RNA thread-safety constraint entirely. The Foundation's practical direction has trended toward moving more procedural logic into nodes rather than fixing `bpy`'s threading model.
- **USD/Hydra** sidesteps `bpy` for cross-DCC scene interchange (asset handoff between Blender, other DCCs, and game engines), but that addresses the *interchange format*, not in-Blender scripting — it is not something an MCP server would use in place of `execute_blender_code`.

The practical takeaway for MCP-driven workflows: `execute_blender_code` and direct `bpy.data` access (§5.4) remain the only way to script a *running* Blender session, and none of the above changes that.

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

### 10.6 Neural Reconstruction: NeRFStudio and 3D Gaussian Splatting

Everything in §10.1–§10.5 generates 3D content from a text prompt or a single image. Neural reconstruction methods — NeRF and 3D Gaussian Splatting (3DGS) — instead reconstruct a scene from many photographs of something that already exists, and Blender's role shifts from a target for AI-generated meshes to a data source and a viewer. **Chapter 115 — NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting** covers the reconstruction techniques themselves in depth; this section covers only the Blender-side plumbing that connects to them.

**Blender as a synthetic training-data generator: BlenderNeRF.** [`maximeraafat/BlenderNeRF`](https://github.com/maximeraafat/BlenderNeRF) (1,000+ stars, MIT) solves the opposite problem from importing a reconstruction — it exports camera poses and rendered images *from* a Blender scene, in the `transforms.json` format NeRFStudio/Instant-NGP expect for training, so that a synthetic (rendered, not photographed) scene can be used as NeRF/3DGS training data. It ships three camera-generation operators, exposed as ordinary Blender operators an MCP agent can call via `execute_blender_code`:

```python
bpy.ops.object.subset_of_frames()   # SOF: render a manually keyframed camera as the training set
bpy.ops.object.train_test_cameras() # TTC: split into train/test camera sets for evaluation
bpy.ops.object.camera_on_sphere()   # COS: synthesize a camera sphere around the subject
```

Output format is NeRF/NGP-style `transforms.json` — **not** COLMAP — and an optional "Gaussian Points" setting additionally writes a `points3d.ply` file to seed 3DGS training with an initial point cloud rather than random initialization. Note: needs verification — BlenderNeRF's last tagged release (v6) predates Blender 5.x; compatibility with current Blender versions should be checked before relying on it in a pipeline.
[Source: BlenderNeRF README](https://github.com/maximeraafat/BlenderNeRF)

**Importing a trained splat back into Blender: KIRI Engine's 3DGS Render.** Going the other direction — bringing an already-trained Gaussian Splat `.ply` into Blender to composite against native Cycles/EEVEE geometry — is covered by [`Kiri-Innovation/3dgs-render-blender-addon`](https://github.com/Kiri-Innovation/3dgs-render-blender-addon) (1,100+ stars, Apache-2.0, requires Blender 5.1+), the most actively maintained option in this space. It exposes three modes (**Edit**, **Render**, **Mesh 2 3DGS**) and only accepts `.ply` files that actually contain 3DGS data — attempting to import an ordinary point-cloud `.ply` produces an error rather than a silent misinterpretation. Its "Combine with Native Render" feature composites the splat render against standard Blender render output, which is the piece that makes hybrid scenes (photorealistic reconstructed backdrop, hand-modeled foreground props) possible. Note: needs verification — whether splats render through Cycles, EEVEE, or an internal rasterizer is not documented by the add-on itself.

**The MCP gap.** Unlike the Meshy/Hyper3D generative pipeline (§9) or the ComfyUI integration (§12.5), no documented AI-agent or MCP-driven workflow ties BlenderNeRF or 3DGS import together end-to-end — there is no equivalent of AI Render's `ai_render.generate_new_image_from_current` operator convention to script against. The mechanism exists in principle (an agent can call BlenderNeRF's operator IDs above via `execute_blender_code` exactly as it would any other operator, per §15.7.5's pre-flight-check pattern), but as of this writing this is an unexplored combination rather than an established one.

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
- **ControlNet conditioning** ([wiki](https://github.com/benrugg/AI-Render/wiki/ControlNet)): rather than letting img2img denoise freely from the rendered frame, AI Render can pass a ControlNet preprocessor input — most commonly the scene's own depth pass, but also Canny edges or an OpenPose skeleton derived from armature bones — to Automatic1111's ControlNet extension. This constrains the diffusion output to respect Blender's actual scene geometry and object silhouettes frame-to-frame, which is what makes stylized-but-coherent animation sequences (rather than a slideshow of independently hallucinated frames) practical with a plain img2img backend.

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

### 12.4 Combining Dream Textures and AI Render

Dream Textures (§12.1) and AI Render (§12.2) operate at different stages of the pipeline — one on scene assets before rendering, the other on rendered frames — and combining them covers ground that neither tool reaches alone:

1. **Material and texture authoring with Dream Textures.** Generate base-colour and detail textures once, in UV space, before any camera or lighting is finalised: text-to-texture for tileable surfaces, Project Dream Texture (depth-to-image) to paint photorealistic detail directly onto existing low-poly or untextured geometry using the viewport's own depth as structure guidance, and inpainting to patch or restyle specific UV islands. Because this step edits `bpy.data.images`/material node graphs directly, its output is a persistent, reusable asset — the same generated texture survives across every subsequent render, camera angle, and animation frame.
2. **A standard Cycles or EEVEE render with a depth pass.** Render the scene normally (§5.3.3), enabling the `Z`/`Mist` or `Denoising Normal`/`Depth` render pass (`view_layer.use_pass_z = True`) alongside the beauty pass. This depth data is the connective tissue between the two AI tools: it is the same kind of structure signal Dream Textures used in step 1, now captured per-frame for use in step 3.
3. **Per-frame stylization with AI Render + ControlNet.** Feed each rendered frame through AI Render's img2img pipeline, conditioning the ControlNet extension on that frame's depth pass (§12.2) rather than letting Stable Diffusion denoise unconstrained. Depth conditioning is what prevents the "AI slideshow" failure mode — independently hallucinated geometry per frame — since every frame's diffusion output is pinned to the same underlying 3D structure Dream Textures' materials were authored against. The prompt itself can still be keyframed (§12.2) to evolve the visual style over the course of the shot, and Batch processing renders the full sequence unattended once a single test frame looks right.

**Why this ordering, not the reverse.** Running AI Render's stylization *before* Dream Textures' material pass would defeat the purpose of ControlNet depth-conditioning — there would be no clean render to derive a depth pass from, and no stable base texture for the stylized look to stay consistent with across frames. Dream Textures establishes a persistent, geometry-correct asset; AI Render's job is purely post-render image transformation, and it needs that geometry-correct render (and its depth data) as an input, not the other way around.

**MCP/Claude Code orchestration.** Because both add-ons expose registered operators, an agent driving this workflow via `execute_blender_code` can script the full chain without leaving the inspect-act-verify loop (§8): generate/apply a Dream Textures material, trigger `bpy.ops.render.render(write_still=True)` with the depth pass enabled, then call AI Render's own operators — `ai_render.generate_new_image_from_current` (single-frame img2img/ControlNet pass on the current render) or `ai_render.inpaint_from_last_sd_image` (targeted re-generation of a masked region of the last AI Render output) — before using `get_viewport_screenshot`/`get_screenshot_of_*` (§3, §3.1) to judge the stylized result and iterate on the prompt or ControlNet strength.

### 12.5 ComfyUI and ComfyScript Integration

Automatic1111's Web UI (the backend AI Render targets in §12.2) is a fixed pipeline with configuration knobs. ComfyUI ([github.com/comfyanonymous/ComfyUI](https://github.com/comfyanonymous/ComfyUI)) is a node-graph diffusion engine instead — every stage of the pipeline (model load, conditioning, sampling, ControlNet, upscaling) is an explicit, rewireable node — which makes it a natural fit for Blender's own node-based mental model, and several projects connect the two directly rather than through Automatic1111's fixed API.

**Direct Blender↔ComfyUI add-ons.** Two actively maintained options take different integration approaches:
- [`alexisrolland/ComfyUI-Blender`](https://github.com/alexisrolland/ComfyUI-Blender) (GPL-3.0, Blender 5.0-compatible as of v4.0.0) is the closest existing equivalent to §12.4's combined workflow, packaged as a single add-on. A workflow is authored in ComfyUI using the project's own Blender-specific I/O nodes (`blender_input_load_image`, `blender_input_load_mask`, `blender_input_load_3d`, `blender_output_save_image`, `blender_output_save_glb`, among others), then exported as API-format JSON and imported into the Blender add-on, which auto-generates a UI panel from the workflow's inputs and outputs. On the Blender side, the add-on ships its own render operators — `render_depth_map`, `render_lineart`, `render_preview`, `render_view` — plus `create_material`/`project_material` to bring a generated image back as a material projected onto the mesh. This is a complete scene → control-pass → diffusion → reimport loop, built specifically around Blender's compositor rather than a generic image round-trip.
- [`AIGODLIKE/ComfyUI-BlenderAI-node`](https://github.com/AIGODLIKE/ComfyUI-BlenderAI-node) (GPL-3.0) takes the opposite approach: it embeds ComfyUI's own node graph *inside* Blender's node editor as a custom node-tree type, so the diffusion workflow is edited in the same UI as Blender's shader/geometry nodes rather than in a separate ComfyUI browser tab.

**ComfyScript.** [`Chaoses-Ib/ComfyScript`](https://github.com/Chaoses-Ib/ComfyScript) (MIT) is a Python front end for ComfyUI: it transpiles an exported ComfyUI workflow into an equivalent Python script and exposes every ComfyUI node as a directly callable, type-hinted Python function, runnable against either a local ComfyUI install or a remote instance over its API. Where the two add-ons above wire a fixed workflow into Blender's UI, ComfyScript is the better fit when an AI agent needs to *construct or modify* the diffusion workflow itself as part of a scripted pipeline, rather than run one that a human already built and exported.

**AI Render's ComfyUI backend: shipped code, unmerged branch.** AI Render (§12.2) has real, substantial ComfyUI backend code — `sd_backends/comfyui_api.py`, `operators_comfyui.py`, a dedicated UI panel — but it lives on the upstream repo's [`comfyui-support` branch](https://github.com/benrugg/AI-Render/tree/comfyui-support), not on `main`, and has never been merged into a release. It requires installing the add-on from that branch's ZIP directly rather than through the normal release. Treat this as real but unshipped: functional per its own `README_comfy.md`, explicitly labeled work-in-progress, and not present in the AI Render most users install.

**MCP/Claude Code orchestration.** `blender-mcp` (§2) has no built-in ComfyUI tool, but community ComfyUI MCP servers exist — e.g. [`artokun/comfyui-mcp`](https://github.com/artokun/comfyui-mcp) — that an agent can hold alongside a Blender MCP connection simultaneously, the same two-server pattern used elsewhere in this chapter for Meshy/Hyper3D (§9). In principle this lets an agent drive the full loop itself: call `execute_blender_code` to render a depth/line-art pass, call the ComfyUI MCP server's tools to run a ComfyScript-defined or pre-built workflow against that pass, then call `execute_blender_code` again to load the returned image as a texture — mirroring §12.4's pipeline but with ComfyUI's node graph in place of Automatic1111/ControlNet. Note: needs verification — this is a plausible mechanism given each piece's documented capabilities individually, not a published, end-to-end tutorial; no worked example combining a Blender MCP server and a ComfyUI MCP server this way was found at the time of writing. `alexisrolland/ComfyUI-Blender`'s packaged operators (above) are the more concretely documented path to the same result today.

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
*Suggested workaround:* have the AI generate the coarse form procedurally — a displacement/multiresolution modifier driven by a noise or image texture, a metaball assembly converted to mesh, or an imported base mesh from Meshy/Hyper3D Rodin (§9) — and hand the result to a human for interactive sculpted refinement rather than attempting to script individual strokes. `get_object_info`/`get_viewport_screenshot` can still verify the procedural base before handoff.

**Grease Pencil strokes.** Grease Pencil v3 (Blender 4.3+) has a Python API for reading and writing stroke point data, but stroke simulation (pressure sensitivity, velocity-dependent width) is not scriptable with the nuance a manual artist achieves.
*Suggested workaround:* script the stroke *geometry* (point positions, layer/frame placement, colour) programmatically where the shape itself is what matters — diagrams, motion guides, simple vector-style art — and reserve manual drawing for strokes where pressure/velocity nuance is the point. Uniform-width strokes generated via the point-data API are usually indistinguishable from manual ones once rendered.

**Physics simulation debugging.** Setting up rigid body, cloth, or fluid simulation parameters is fully scriptable. Diagnosing *why* a simulation produces wrong results — cloth self-intersecting, fluid leaking, rigid body jitter — requires interactive playback with parameter tweaking. There is no API for "play simulation and return quality metrics."
*Suggested workaround:* bake the simulation headlessly (`bpy.ops.ptcache.bake_all()`), then have the AI inspect *proxy signals* rather than play back the animation itself — self-intersection can be approximated by sampling mesh bounding-box overlap or vertex-to-vertex distances across baked frames via `bpy.data`, and a rendered turntable of a few sampled frames via `get_viewport_screenshot` lets Claude's vision flag gross failures (fluid escaping its domain, an object clipping through the floor) even without true physical-accuracy metrics.

**Video Sequence Editor.** The VSE Python API exists but is complex enough that generating correct strip timing, transitions, and audio sync from a script requires substantial error-prone bookkeeping. Claude can do it, but failures are common.
*Suggested workaround:* apply the same Info-editor-generalisation pattern from §8.2 — assemble one representative cut manually in the VSE UI, copy the resulting `bpy.ops.sequencer.*` calls and strip properties from the Info log, and have Claude generalise that into a parameterised script rather than deriving VSE timing logic from the API reference alone.

**Compositor node trees.** The compositing node graph uses the same `node_tree` API as material shaders, but the node type names and socket names differ and are not consistent across Blender versions. Claude frequently generates compositor node connections that silently fail to link because a socket name changed (e.g., `"Image"` vs `"Color"` depending on version and node type).
*Suggested workaround:* before wiring links, have the script (or the AI via `execute_blender_code`) enumerate `node.inputs.keys()`/`node.outputs.keys()` on the actual nodes just created and match against that live list instead of a remembered socket name; alternatively, build the compositor graph once in the UI, save it as a `.blend` library file, and append it with `bpy.ops.wm.append()` the same way §15.6 recommends for Geometry Nodes.

### 15.4 The Visual Feedback Loop

Without `get_viewport_screenshot`, the AI works blind: it can construct geometry and materials entirely from the structured data returned by `get_scene_info` and `get_object_info`, but it cannot judge proportions, lighting feel, or render quality. With the screenshot tool, a feedback loop becomes possible:

```
prompt → execute_blender_code → get_viewport_screenshot → analyse → adjust → repeat
```

The practical constraint is latency: each round trip (tool call → Blender execution → screenshot encode → Claude vision analysis) takes several seconds. For fine aesthetic iteration — adjusting a light angle by a few degrees, tweaking roughness — this loop is slow compared to dragging a slider interactively. The sweet spot is coarser creative decisions: "is the overall composition balanced?", "does this material read as metallic?", "is the lighting direction correct?"

Claude's vision capability is competent at identifying gross problems (wrong hemisphere lighting, obviously incorrect scale, materials with clearly wrong albedo) but unreliable for fine-grained aesthetic judgements that experienced artists make intuitively.

*Suggested mitigations:* match the render settings to the question being asked rather than always screenshotting at full fidelity — use EEVEE's viewport (not Cycles) for the screenshot itself, since interactive-rate rendering removes most of the round-trip latency and coarse composition/lighting-direction judgements don't need path-traced accuracy; batch several candidate parameter values into one `execute_blender_code` call (e.g., render three roughness values to three separate images in one pass) and screenshot once rather than looping one-value-at-a-time; and reserve the loop for the coarse decisions it's actually good at (§15.4 above), routing fine aesthetic tuning — the few-degree light nudges, the exact roughness value — back to a human turning a slider interactively, with the AI only re-entering the loop once a new coarse decision is needed.

### 15.5 Animation and Rigging

Keyframe animation is fully scriptable through `bpy.data.objects[name].keyframe_insert()` and FCurve manipulation. Rigging is scriptable through armature creation, bone parenting, and constraint assignment. Both work; both are verbose and error-prone for complex cases:

- **Armature construction**: bone head/tail positions must be specified in exact coordinates; an off-by-one in bone hierarchy causes the entire rig to deform incorrectly.
  *Suggested solution:* for humanoid/creature rigs, skip manual bone-by-bone construction entirely and generate the armature from Blender's Rigify add-on (`bpy.ops.object.armature_human_metarig_add()` or another metarig preset) via `bpy.ops`, then have the AI reposition the metarig's bones to match the target mesh proportions before the single `bpy.ops.pose.rigify_generate()` call — this confines AI-authored coordinate math to bone *placement* rather than the far more error-prone hierarchy and roll/axis setup Rigify handles internally.
- **IK chain setup**: requires setting `bone.use_ik_limit_x`, `bone.ik_stiffness_x`, `bone.ik_stretch` and adding an `INVERSE_KINEMATICS` constraint via `bpy.ops.pose.ik_add()`, which itself requires Pose mode context.
  *Suggested solution:* Rigify-generated rigs (above) ship IK/FK switching already wired on the control bones, so the practical fix is the same one — prefer generating from a metarig over hand-assembling `INVERSE_KINEMATICS` constraints. Where a custom IK chain is unavoidable, use `temp_override()` (§5.1.1) to satisfy the Pose-mode `poll()` requirement rather than switching the UI's active mode, so the script doesn't depend on whatever mode Blender happened to be in when the MCP command arrived.
- **NLA (Non-Linear Animation)**: the NLA editor Python API involves `action`, `nla_track`, and `nla_strip` objects whose timing relationships are easy to get wrong.
  *Suggested solution:* apply the Info-editor-generalisation pattern (§8.2) — build one representative strip arrangement manually, read back the resulting `action`/`nla_track`/`nla_strip` properties from `bpy.data`, and have Claude generalise the *offsets between* strips (which is what's error-prone) from that concrete example rather than deriving NLA timing math from first principles.
- **Shape keys**: creating and keying shape keys (`obj.shape_key_add()`, `key.keyframe_insert("value")`) is reliable for simple cases; driver-based shape key animation requires the driver API (`fcurve.driver`, `driver.variables`) which is complex.
  *Suggested solution:* for the common case (a shape key value driven directly by another single property, such as a bone's rotation angle for a facial rig), use `driver_add()` and inspect the generated `FCurve.driver.expression`/`variables` structure afterward via `get_object_info`-style introspection to verify it before trusting it, rather than hand-writing the driver's internal graph; keep multi-variable or Python-expression drivers as a human-authored step.

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

### 15.7 Programmatic Verification Beyond Vision

Every workaround in §15.2–§15.6 shares a weakness: the fallback verification step is `get_viewport_screenshot` plus Claude's vision analysis (§15.4), which is slow (a multi-second round trip per check) and imprecise (competent at gross errors, unreliable at fine ones). Most of what an AI needs to verify about a Blender scene — did this operator actually change anything, does this mesh have a hole in it, is this object floating above the floor, does this downloaded glTF have the material channels the target engine expects — is answerable as a structured boolean or number from `bpy`/`bmesh` data, without rendering a single pixel. The following techniques extend the inspect-act-verify loop (§8) with checks precise enough to replace a screenshot, and coarse enough to run on every step rather than being reserved for a final visual pass. None of these are bespoke to this chapter's MCP servers — either server's `execute_blender_code` (§3, §3.1) is sufficient to run all of them, and a production MCP deployment would typically promote the most-used ones to dedicated structured tools rather than re-generating the same Python each time.

#### 15.7.1 Hit Testing and Ray Casting

`Scene.ray_cast(depsgraph, origin, direction, distance=...)` casts against every visible object in the scene and returns `(result, location, normal, index, object, matrix)` — the same query the 3D cursor's "snap to surface" and physics engines use internally, now available to verify placement without a screenshot:

```python
import bpy
from mathutils import Vector

depsgraph = bpy.context.evaluated_depsgraph_get()
scene = bpy.context.scene

def is_resting_on_surface(obj, max_gap=0.001):
    """Cast downward from the object's base to confirm it touches something."""
    base = obj.matrix_world @ Vector((0, 0, obj.bound_box[0][2]))
    hit, loc, normal, index, hit_obj, matrix = scene.ray_cast(
        depsgraph, base + Vector((0, 0, 0.01)), Vector((0, 0, -1))
    )
    return hit and (base.z - loc.z) < max_gap
```

For a single known object rather than the whole scene, `Object.ray_cast(origin, direction, distance=...)` returns `(result, location, normal, index)` against that object alone — use `obj.evaluated_get(depsgraph)` first if the object has modifiers, since the un-evaluated `Object` exposes only the base mesh. This directly replaces the "does it look like it's floating?" judgement call from §15.4 with a numeric gap the AI can threshold on, and gives the physics-debugging gap in §15.3 a cheap self-intersection proxy: ray-cast between two objects' surfaces along their shared axis and check for a hit at a shallower distance than either object's own extent.

**Existing tooling.** For actual geometric intersection between two objects — not just "is there a surface below this point" — `mathutils.bvhtree.BVHTree` is the module to reach for instead of hand-rolled ray sweeps: `BVHTree.FromObject(obj, depsgraph)` builds an acceleration structure per object, and `tree_a.overlap(tree_b)` returns the actual overlapping triangle-index pairs between two meshes, which is real intersection testing rather than a bounding-box approximation ([mathutils.bvhtree API reference](https://docs.blender.org/api/current/mathutils.bvhtree.html)). This is the more precise version of the AABB pre-filter in §15.7.3.

#### 15.7.2 Geometry Analysis

`bmesh` (§5.2.2) exposes the same primitives Blender's own mesh-cleanup tools use, callable read-only via the `bm.from_mesh()` pattern from §5.4 without needing Edit Mode or an operator `poll()` to pass:

```python
import bpy, bmesh

def analyze_mesh(obj):
    me = obj.data
    bm = bmesh.new()
    bm.from_mesh(me)
    bm.normal_update()

    report = {
        "non_manifold_edges": sum(1 for e in bm.edges if not e.is_manifold),
        "zero_area_faces":    sum(1 for f in bm.faces if f.calc_area() < 1e-8),
        "ngons":              sum(1 for f in bm.faces if len(f.verts) > 4),
        "signed_volume":      bm.calc_volume(signed=True),
    }
    bm.free()
    return report
```

`edge.is_manifold` (true only when an edge borders exactly two faces) flags holes and non-watertight seams — the exact defect class that makes a mesh unsuitable for boolean operations, 3D printing, or physics collision. `calc_area()` near zero flags degenerate faces left over from a bad boolean or remesh. `calc_volume(signed=True)` going negative on a mesh that should be closed and outward-facing is a cheap global flipped-normals check, since a consistently outward-facing closed mesh always integrates to a positive volume. Running this immediately after any sculpting-workaround, boolean, or remesh step (§15.3) turns "does this mesh look broken?" into a pass/fail gate before the AI even requests a screenshot.

**Existing tooling.** Much of this is already implemented, tested, and UI-exposed rather than needing to be hand-rolled:
- **3D-Print Toolbox** (`object_print3d_utils`, bundled with Blender — enable it in Preferences → Add-ons) provides dedicated check operators for non-manifold edges, intersecting faces, and wall thickness, plus a **Make Manifold** operator that attempts automatic repair of bad normals, holes, and empty geometry ([3D Print Toolbox — Blender Manual](https://docs.blender.org/manual/en/4.0/addons/mesh/3d_print_toolbox.html), [source](https://github.com/blender/blender-addons/blob/main/object_print3d_utils/__init__.py)).
- Blender's built-in **Mesh Analysis overlay** (`Mesh.statvis`, toggled via the viewport Overlays panel) computes five per-face heatmaps — Overhang, Thickness, Intersect, Distort, and Sharp — as a vertex-color layer, viewable directly or read back programmatically instead of writing the equivalent bmesh checks by hand ([Mesh Analysis — Blender Manual](https://docs.blender.org/manual/en/latest/modeling/meshes/mesh_analysis.html), [MeshStatVis API reference](https://docs.blender.org/api/current/bpy.types.MeshStatVis.html)); a third-party [Mesh Analysis Overlay extension](https://extensions.blender.org/add-ons/mesh-analysis-overlay/) adds further display modes.
- The community add-on **MeshLint** bundles tris/ngons/non-manifold/interior-face/unapplied-scale checks into one panel ([rking/meshlint](https://github.com/rking/meshlint)), though it has seen little maintenance in recent Blender versions and should be checked for 4.x/5.x compatibility before relying on it.
- Outside Blender entirely, the standalone Python library **`trimesh`** loads GLB/OBJ/STL/PLY directly and exposes `.is_watertight`, volume, convex hull, and ray-mesh queries as plain properties/methods — a maintained alternative for auditing a mesh (including one that never gets imported into Blender at all) with one dependency instead of bmesh-based scripting ([trimesh.org](https://trimesh.org/), [PyPI](https://pypi.org/project/trimesh/)).

#### 15.7.3 Scene Analysis

`get_scene_info`/`get_objects_summary` (§3, §3.1) return a flat object listing; the checks below turn that listing into the kind of audit §16.1's prompt toolkit asks for in natural language, as reusable code instead of freshly generated script each time:

```python
import bpy
from mathutils import Vector

def world_aabb(obj):
    corners = [obj.matrix_world @ Vector(c) for c in obj.bound_box]
    lo = Vector(min(c[i] for c in corners) for i in range(3))
    hi = Vector(max(c[i] for c in corners) for i in range(3))
    return lo, hi

def aabb_overlap(obj_a, obj_b):
    a_lo, a_hi = world_aabb(obj_a)
    b_lo, b_hi = world_aabb(obj_b)
    return all(a_lo[i] <= b_hi[i] and b_lo[i] <= a_hi[i] for i in range(3))

def scene_audit(scene):
    return {
        "orphan_materials":  [m.name for m in bpy.data.materials if m.users == 0],
        "orphan_images":     [i.name for i in bpy.data.images if i.users == 0],
        "non_uniform_scale": [o.name for o in scene.objects
                               if len(set(round(s, 4) for s in o.scale)) > 1],
        "overlapping_pairs": [(a.name, b.name)
                               for i, a in enumerate(scene.objects)
                               for b in list(scene.objects)[i + 1:]
                               if a.type == 'MESH' and b.type == 'MESH'
                               and aabb_overlap(a, b)],
    }
```

Every ID data-block carries a `.users` count, so a zero-user material or image is guaranteed orphaned regardless of how it got that way — useful for catching the "generated a material, never assigned it" mistake §5.1.2 warns about. Non-uniform scale is worth flagging proactively rather than discovering it downstream: it silently breaks physics collision shapes and can invert normals on export. AABB overlap is a coarse pre-filter — two objects' bounding boxes touching doesn't mean their geometry actually interpenetrates — but it is enough to prioritise which object pairs are worth a real ray-cast or `BVHTree.overlap()` check (§15.7.1) instead of testing every pair in a large scene at full precision.

**Existing tooling.** Unlike geometry analysis (§15.7.2) and glTF analysis (§15.7.4 below), no widely-used add-on or library was found that specifically covers this combination of checks — orphan data-block detection, non-uniform-scale flagging, and cross-object overlap auditing at the scene level. This looks like genuine DIY territory: `.users`-based orphan detection and `bound_box`/`BVHTree`-based overlap checks are simple enough in isolation (as above) that no dedicated tool appears to have consolidated them, rather than one being deliberately avoided for some hidden reason. *Note: needs verification* — absence of a found tool is not proof none exists.

#### 15.7.4 glTF Asset Analysis

Assets arriving via Sketchfab/Meshy/Hyper3D Rodin (§3, §9) or leaving via the glTF export pipeline (§17) are worth validating structurally before they reach a game engine or Three.js, independent of whatever `.blend`-side checks already ran. [`gltf-validator`](https://github.com/KhronosGroup/glTF-Validator) (already used in §17's pipeline) checks spec conformance:

```bash
gltf_validator asset.glb --format json > report.json
python3 -c "import json; r = json.load(open('report.json')); print(r['issues']['numErrors'], r['issues']['numWarnings'])"
```

For questions the validator doesn't answer — does this asset have the specific channels a target renderer expects? — [`pygltflib`](https://pypi.org/project/pygltflib/) parses the glTF JSON structure directly:

```python
from pygltflib import GLTF2

gltf = GLTF2().load("asset.glb")
print("extensions used:", gltf.extensionsUsed)
print("materials:", len(gltf.materials or []))

for i, mesh in enumerate(gltf.meshes or []):
    for prim in mesh.primitives:
        if prim.targets:
            print(f"mesh {i}: {len(prim.targets)} morph targets")

for skin in gltf.skins or []:
    print(f"skin with {len(skin.joints)} joints — needs a skinning-capable renderer")
```

This is the structural equivalent of §15.3's compositor socket-name check applied to an external file rather than a live node tree: rather than assuming a downloaded or exported asset has the PBR channels, morph targets, or skin data a workflow needs, read the actual glTF JSON and branch on what is really there. It runs as ordinary Python outside Blender entirely — in the MCP server process or a CI step — so it applies equally to assets the AI never loads into Blender at all.

**Existing tooling.** `pygltflib` above is the minimal, dependency-light option when only a few specific fields matter; for a fuller picture, **`gltf-transform inspect model.glb`** (from `@gltf-transform/cli`, already used elsewhere in this chapter's export pipeline for Draco/meshopt post-processing, §17) prints a structured report covering every scene, mesh, material, texture, and animation in the file — including which extensions are required to load it and how much of the file size is geometry versus textures — with `--format=csv`/`--format=md` for machine-readable output ([gltf-transform.dev/cli](https://gltf-transform.dev/cli)). `trimesh` (§15.7.2) is also a reasonable choice here since it loads GLB natively and gives `.is_watertight`/volume checks on top of the format-level inspection `gltf-transform` and `gltf-validator` provide — the three tools cover, respectively, geometric integrity, structural/spec conformance, and asset-budget reporting.

#### 15.7.5 Operator Pre-Flight Checks

Every `bpy.ops` entry supports calling `.poll()` without executing the operator, which turns the `poll() failed` `RuntimeError` from §15.2 into a checkable condition instead of a caught exception:

```python
import bpy

def safe_call(op, label, **kwargs):
    if not op.poll():
        return {"error": f"{label}.poll() failed — wrong context, not executed"}
    return {"result": op(**kwargs)}

safe_call(bpy.ops.mesh.subdivide, "bpy.ops.mesh.subdivide", number_cuts=2)
```

This does not eliminate the context-setup problem §15.2 describes — the AI still has to know *why* `poll()` failed and how to fix it (wrong mode, no active object, wrong area type) — but it removes one full round trip per failure: instead of executing, hitting a traceback, parsing the error message, and retrying, `execute_blender_code` can check-then-call in a single pass and return a clean structured reason immediately.

#### 15.7.6 Before/After Geometry Diffing

The most common silent failure in AI-generated `bpy` code is an operator that returns `{'FINISHED'}` without actually changing anything — usually because the selection was empty or the wrong object was active. A cheap signature comparison catches this without any visual check at all:

```python
def mesh_signature(obj):
    me = obj.data
    return (len(me.vertices), len(me.edges), len(me.polygons),
             tuple(round(d, 5) for d in obj.dimensions))

before = mesh_signature(obj)
bpy.ops.mesh.subdivide(number_cuts=2)
after = mesh_signature(obj)

if before == after:
    raise RuntimeError("bpy.ops.mesh.subdivide ran but produced no detectable change")
```

This must run in Object Mode, or after `bmesh.update_edit_mesh(me)`/an Edit-to-Object mode toggle — `me.vertices`/`me.polygons` reflect the mesh's last synced state, not live edits still pending inside an active `BMesh` edit session (§5.2.2, §5.4). The same signature-comparison pattern generalises past mesh edits: comparing `len(bpy.data.objects)` before/after an import, or a material's node count before/after a generation step, catches the same class of "ran without error but did nothing" failure anywhere in the pipeline.

#### 15.7.7 Pixel-Diff Screenshots

Not every visual check needs Claude's vision model — a plain pixel difference between two `get_viewport_screenshot`/`get_screenshot_of_*` (§3, §3.1) captures can decide whether a full vision pass is even warranted, cutting the latency §15.4 flags for the common case where a parameter change was too small to matter yet:

```python
import base64
from io import BytesIO
from PIL import Image, ImageChops

def screenshot_diff(png_b64_before: str, png_b64_after: str) -> float:
    a = Image.open(BytesIO(base64.b64decode(png_b64_before))).convert("RGB")
    b = Image.open(BytesIO(base64.b64decode(png_b64_after))).convert("RGB")
    diff = ImageChops.difference(a, b).convert("L")
    hist = diff.histogram()
    weighted = sum(i * n for i, n in enumerate(hist))
    return weighted / (a.width * a.height * 255)   # 0.0 identical, 1.0 fully different
```

This runs in the MCP server process or Claude Code's own environment, not inside Blender's `execute_blender_code` — Blender's bundled interpreter does not ship Pillow by default, but `get_viewport_screenshot` has already handed both images back as base64 PNG by the time this comparison is needed. A practical policy: skip the vision call entirely below some diff threshold (nothing meaningfully changed — likely a no-op or a change too small to matter), and only spend a vision round trip above it, reserving Claude's actual visual judgement for the cases in §15.4 it is genuinely needed for.

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

## 18. Roadmap

The workflows in this chapter sit on two independently governed timelines: Blender Foundation's own three-release-a-year cadence, and the Model Context Protocol specification maintained by Anthropic and the wider MCP community. Both move fast enough that the API surface this chapter targets — `bpy`, the Geometry Nodes modifier RNA, and the MCP wire format itself — should be expected to shift within a year of this draft.

### 18.1 Blender Release Cadence

Blender ships three releases a year, one of which is a Long-Term Support (LTS) release maintained with monthly bug-fix updates for two years; non-LTS releases are supported only until the next release ships.

| Release | Date | LTS | EOL |
|---|---|---|---|
| 4.2 LTS | July 2024 | ✓ | July 2026 |
| 4.5 LTS | July 15, 2025 | ✓ | July 2027 |
| 5.0 | November 18, 2025 | — | — |
| 5.1 | March 2026 | — | — |
| 5.2 LTS | July 14, 2026 | ✓ | July 2028 |
| 5.3 | November 2026 (planned) | — | — |

[Source: Blender Releases](https://www.blender.org/releases/) · [Source: Blender 5.2 LTS Release](https://www.blender.org/press/blender-5-2-lts-release/)

At the time of writing, **5.2 LTS** (released July 2026) is the current recommended baseline for production add-ons and MCP-driven pipelines, superseding 4.5 LTS. §4.1's note on the VFX Reference Platform staying on Python 3.13 through the 2027 platform year applies independently of this cadence — Blender's own release schedule and the VFX Platform's Python pin are set by different bodies and do not move in lockstep.

### 18.2 What Blender 5.2 LTS Changes for AI-Driven Workflows

Two Python API changes in 5.2 LTS are directly relevant to workarounds described earlier in this chapter:

- **`gpu.init()` for background mode.** Previously, calling into the `gpu` module while running with `--background` raised `SystemError: GPU API is not available in background mode`, which is why §14.2 recommends a virtual display (`Xvfb`/EGL) purely to get a GPU context for scripted rendering. `gpu.init()` initializes the GPU backend explicitly in headless mode, narrowing the cases where a virtual display is still required to genuinely interactive (viewport-dependent) operations rather than all off-screen GPU work.
- **Geometry Nodes modifier inputs move from custom properties to RNA.** §15.6 notes that Geometry Nodes' Python surface is fragile because inputs were historically set via ID-properties (`modifier["Input_2"] = 5.0`, keyed by an opaque per-tree identifier). 5.2 LTS replaces this with proper RNA — `modifier.properties.inputs.<identifier>.value`, `.type`, and `.attribute_name`, with a parallel `.outputs.<identifier>.attribute_name` for output attribute mapping. This doesn't fix the node-identifier fragility §15.6 describes for *constructing* node trees, but it does make *driving* an existing Geometry Nodes setup from Python considerably more discoverable and typo-resistant than string-keyed custom properties.

[Source: Blender 5.2 LTS Python API Release Notes](https://developer.blender.org/docs/release_notes/5.2/python_api/) — Note: needs verification (developer.blender.org returns HTTP 403 to automated fetching; the summary above is reconstructed from third-party citations of the release notes and should be checked against the live page before publication.)

### 18.3 The Blender MCP Server's Place in Blender Lab

The official Blender Foundation MCP server (§2, referenced in this chapter's tooling comparisons) lives under **Blender Lab**, an incubator track for experimental Blender Foundation projects that is explicitly not part of Blender's numbered release roadmap. Lab projects have no committed release timeline or version-parity guarantee with the main Blender release train; the MCP server reached v1.0.0 in April 2026 as a Lab milestone, not a Blender point release. Practically, this means an MCP-driven pipeline that depends on the official server should track Blender Lab's own repository and activity reports directly, rather than assuming the server's feature set advances in step with `bpy` itself.

[Source: Blender Lab](https://www.blender.org/lab/) · [Source: MCP Server — Blender Lab](https://www.blender.org/lab/mcp-server/)

### 18.4 Model Context Protocol Specification Roadmap

The MCP specification itself is on a faster and less predictable cadence than Blender. The protocol's 2026 roadmap abandons date-based release planning in favor of four Working-Group-owned priority areas:

- **Transport evolution and scalability** — improving HTTP streaming transport for horizontal scaling, stateless sessions, and server discoverability via `.well-known` metadata.
- **Agent communication** — refining the experimental **Tasks** primitive (long-running, poll-able server operations) with retry semantics and result-expiry policies, based on production feedback.
- **Governance maturation** — a contributor ladder and delegation of Specification Enhancement Proposal (SEP) review to domain-expert Working Groups.
- **Enterprise readiness** — audit trails, SSO integration, gateway behavior, and configuration portability, deliberately left under-specified to invite community input.

The **2026-07-28** specification release formally locks in an extensions framework — Tasks, MCP Apps, and Enterprise Managed Authorization (EMA) all now live behind that framework rather than as ad hoc additions to the core spec — plus a formal deprecation policy with a twelve-month minimum window. For the Blender MCP integrations in this chapter, the most consequential of these is the Tasks primitive: long-running operations like a Cycles bake or a Meshy generation job (§9–§11) are natural fits for a poll-able task rather than a blocking tool call, though neither MCP server discussed in this chapter has adopted Tasks as of this writing.

[Source: The 2026 MCP Roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) · [Source: The 2026-07-28 Specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/)

### 18.5 Long-Term Direction

Three threads worth tracking for anyone building on this chapter's patterns:

- **Headless GPU access is normalizing.** `gpu.init()` (§18.2) is one data point in a broader trend of background-mode Blender gaining parity with interactive Blender — relevant to every pipeline in §14.
- **MCP is moving from wiring to infrastructure.** The shift from release-based to priority-area planning, and the Tasks primitive specifically, signal that MCP servers built today as thin `execute_python`-style bridges (§3) will likely be expected to expose structured, long-running, poll-able operations rather than one blocking call per request.
- **Blender Lab is a leading indicator, not a commitment.** Because Lab projects (including the official MCP server) sit outside the numbered release roadmap, the community `ahujasid/blender-mcp` server (§2) remains the more stable dependency for production use until — and unless — Blender Lab's MCP server graduates into a supported, versioned part of Blender proper.

---

## Integrations

**Chapter 42 — Blender GPU: Cycles and EEVEE** establishes the GPU rendering architecture (GPUBackend abstraction, VKBackend/Vulkan, Cycles multi-backend compute) that the Blender Python API scripts and MCP server operate on top of. Add-on scripts that modify materials interact with the same shader compilation pipeline described there.

**Chapter 94 — ComfyUI and ComfyScript** covers the node-graph AI image generation workflow itself in depth; §12.5 of this chapter covers only the Blender-side integration (ComfyUI-Blender, ComfyUI-BlenderAI-node, AI Render's unmerged ComfyUI branch) that connects it to a running Blender session. ComfyScript's "AI-writes-Python-against-a-node-graph" paradigm mirrors the Blender MCP workflow described here closely enough that the two are natural to pair behind a single agent.

**Chapter 115 — NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting** covers the open-source 3D reconstruction approaches (NeRF, 3DGS) themselves in depth; §10.6 of this chapter covers only the Blender-side plumbing (BlenderNeRF's training-data export, KIRI Engine's splat import) that connects Blender to that reconstruction pipeline. The NeRF representation is also the output format of Shap-E.

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
- [Poly Haven](https://polyhaven.com) — CC0 HDRI/texture/model library used by the community server's PolyHaven tools
- [Sketchfab](https://sketchfab.com) — 3D model marketplace used by the community server's Sketchfab tools
- [FastMCP — Python SDK](https://gofastmcp.com) — FastMCP library used by blender-mcp server.py
- [Blender Releases](https://www.blender.org/releases/) — Release schedule and version history
- [Blender 5.2 LTS Release](https://www.blender.org/press/blender-5-2-lts-release/) — Press release, July 2026
- [Blender 5.2 LTS Python API Release Notes](https://developer.blender.org/docs/release_notes/5.2/python_api/) — gpu.init() and Geometry Nodes modifier RNA changes
- [Blender Lab](https://www.blender.org/lab/) — Incubator track for experimental Blender Foundation projects, outside the numbered release roadmap
- [The 2026 MCP Roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) — Priority-area planning, Tasks primitive, governance changes
- [The 2026-07-28 MCP Specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/) — Extensions framework (Tasks, MCP Apps, EMA) and deprecation policy
- [blender-mcp on PyPI](https://pypi.org/project/blender-mcp/) — `uvx blender-mcp` installation
- [Blender Python API Overview](https://docs.blender.org/api/current/info_overview.html) — Official bpy module documentation
- [bpy API reference — Blender 5.2](https://docs.blender.org/api/current/) — Auto-generated from RNA
- [Python Threads are Not Supported — API reference](https://docs.blender.org/api/current/info_gotchas_threading.html) — Official statement of bpy's thread-safety limits
- [Blender Developer T23401 — bContext isn't thread safe](https://developer.blender.org/T23401) — 2010 bug report underlying §4.1's root-cause explanation
- [PEP 779 — Free-threaded Python](https://peps.python.org/pep-0779/) — 2026 official-support status for no-GIL CPython
- [VFX Reference Platform](https://vfxplatform.com) — Industry-coordinated dependency stack (Python, Qt/PySide, …) that Blender's Python version tracks
- [devtalk.blender.org — VFX Platform to stay on Python 3.13 in 2027](https://devtalk.blender.org/t/vfx-platform-to-stay-on-python-3-13-in-2027-reasons-to-try-to-request-3-14-instead/44974) — PySide 6.10/Qt 6.10 dependency cited as the reason for staying on 3.13
- [bpy standalone on PyPI](https://pypi.org/project/bpy/) — Server-side bpy without Blender UI
- [RNA — Blender Developer Documentation](https://developer.blender.org/docs/features/core/rna/) — DNA/RNA system design
- [Blender Extensions system — 4.2](https://developer.blender.org/docs/release_notes/4.2/extensions/) — blender_manifest.toml, dependency wheels
- [bpy.app.timers — API reference](https://docs.blender.org/api/current/bpy.app.timers.html) — Main-thread callback scheduling
- [bpy.types.Operator — API reference](https://docs.blender.org/api/current/bpy.types.Operator.html) — Operator lifecycle: poll/invoke/execute/modal
- [bpy.props — API reference](https://docs.blender.org/api/current/bpy.props.html) — Property factory functions and PropertyGroup usage
- [bpy.utils — API reference](https://docs.blender.org/api/current/bpy.utils.html) — register_class/unregister_class, previews, resource_path
- [mathutils — API reference](https://docs.blender.org/api/current/mathutils.html) — Vector/Matrix/Quaternion/Euler types
- [gpu — API reference](https://docs.blender.org/api/current/gpu.html) — GPUShader/GPUBatch/GPUOffScreen, built-in shaders
- [imbuf — API reference](https://docs.blender.org/api/current/imbuf.html) — ImBuf load/write/new and pixel-buffer access
- [bmesh — API reference](https://docs.blender.org/api/current/bmesh.html) — BMesh/BMVert/BMEdge/BMFace, is_manifold, calc_area, calc_volume
- [bpy.types.Object.ray_cast — API reference](https://docs.blender.org/api/current/bpy.types.Object.html#bpy.types.Object.ray_cast) — Single-object ray casting
- [bpy.types.Scene.ray_cast — API reference](https://docs.blender.org/api/current/bpy.types.Scene.html#bpy.types.Scene.ray_cast) — Scene-wide ray casting via the depsgraph
- [pygltflib on PyPI](https://pypi.org/project/pygltflib/) — Programmatic glTF/GLB JSON parsing outside Blender
- [mathutils.bvhtree — API reference](https://docs.blender.org/api/current/mathutils.bvhtree.html) — BVHTree.FromObject/overlap for real triangle-level intersection testing
- [3D Print Toolbox — Blender Manual](https://docs.blender.org/manual/en/4.0/addons/mesh/3d_print_toolbox.html) — Bundled non-manifold/intersection/thickness checks and Make Manifold repair
- [Mesh Analysis — Blender Manual](https://docs.blender.org/manual/en/latest/modeling/meshes/mesh_analysis.html) — Built-in Overhang/Thickness/Intersect/Distort/Sharp overlay (MeshStatVis)
- [Mesh Analysis Overlay — Blender Extensions](https://extensions.blender.org/add-ons/mesh-analysis-overlay/) — Third-party extension of the built-in overlay
- [GitHub — rking/meshlint](https://github.com/rking/meshlint) — Community mesh-quality linting add-on (tris/ngons/non-manifold/interior faces)
- [trimesh](https://trimesh.org/) — Standalone Python triangular-mesh library: watertight checks, ray casting, GLB/OBJ/STL/PLY loading
- [glTF-Transform CLI — inspect](https://gltf-transform.dev/cli) — Structured per-file report of scenes/meshes/materials/textures/animations
- [BlenderNeRF](https://github.com/maximeraafat/BlenderNeRF) — Exports Blender camera paths as NeRF/NGP training datasets
- [3DGS Render by KIRI Engine](https://github.com/Kiri-Innovation/3dgs-render-blender-addon) — Imports and renders trained 3D Gaussian Splats inside Blender
- [ComfyUI](https://github.com/comfyanonymous/ComfyUI) — Node-graph Stable Diffusion engine
- [ComfyUI-Blender](https://github.com/alexisrolland/ComfyUI-Blender) — Packaged scene→control-pass→diffusion→reimport add-on
- [ComfyUI-BlenderAI-node](https://github.com/AIGODLIKE/ComfyUI-BlenderAI-node) — Embeds the ComfyUI node graph inside Blender's node editor
- [ComfyScript](https://github.com/Chaoses-Ib/ComfyScript) — Python front end transpiling ComfyUI workflows into callable functions
- [AI Render — comfyui-support branch](https://github.com/benrugg/AI-Render/tree/comfyui-support) — Unmerged, work-in-progress ComfyUI backend for AI Render
- [comfyui-mcp](https://github.com/artokun/comfyui-mcp) — Community MCP server for a local ComfyUI instance
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
- [AI-Render wiki — ControlNet](https://github.com/benrugg/AI-Render/wiki/ControlNet) — Depth/Canny/OpenPose conditioning via Automatic1111's ControlNet extension
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
