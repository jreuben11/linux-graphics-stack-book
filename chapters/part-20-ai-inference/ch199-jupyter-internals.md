# Chapter 199: Jupyter Internals: Architecture, Python Runtime, Multi-Kernel Support, and GPU Computing

**Target audiences**: GPU compute developers using Jupyter for CUDA/ROCm/JAX workflows who want to understand what actually happens between pressing Shift+Enter and seeing a result; Python and IPython framework developers extending the kernel or building new language backends; data scientists who want to debug memory leaks, slow startups, or widget failures; browser and web platform engineers interested in JupyterLab's TypeScript rendering pipeline and how it maps onto MIME types and WebSocket protocol.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [The ZMQ Kernel Protocol](#2-the-zmq-kernel-protocol)
3. [IPython and Python Kernel Internals](#3-ipython-and-python-kernel-internals)
4. [Multi-Kernel Support](#4-multi-kernel-support)
5. [The Notebook File Format](#5-the-notebook-file-format)
6. [JupyterLab Rendering Pipeline](#6-jupyterlab-rendering-pipeline)
7. [Interactive Widgets: ipywidgets and the comm Protocol](#7-interactive-widgets-ipywidgets-and-the-comm-protocol)
8. [GPU Computing in Notebooks](#8-gpu-computing-in-notebooks)
9. [GPU Visualisation](#9-gpu-visualisation)
10. [Real-Time Collaboration](#10-real-time-collaboration)
11. [JupyterLite: WASM Kernels in the Browser](#11-jupyterlite-wasm-kernels-in-the-browser)
12. [Linux-Specific: GPU Access, Containers, and JupyterHub](#12-linux-specific-gpu-access-containers-and-jupyterhub)
13. [Roadmap](#13-roadmap)
14. [Integrations](#14-integrations)

---

## 1. Architecture Overview

Jupyter separates the *frontend* — whatever renders cells and captures keystrokes — from the *kernel* — whatever executes them. That split is the entire architectural secret: a browser tab, VS Code, nteract, or a plain terminal can all drive the same Python (or Julia, or R) process over a well-specified protocol.

The server-side anchor is **`jupyter_server`** ([source](https://github.com/jupyter-server/jupyter_server)), a Tornado-based HTTP and WebSocket server. It exposes a REST API for contents (`/api/contents`), sessions (`/api/sessions`), and kernel lifecycle (`/api/kernels`), and for each live kernel it opens a single WebSocket at `kernels/<kernel-id>/channels` over which all ZMQ socket traffic is multiplexed. **JupyterLab** ([source](https://github.com/jupyterlab/jupyterlab)) is a TypeScript/React single-page application served from `jupyter_server`; the *Classic Notebook* (`notebook` package ≤ 6) is a separate, lighter frontend; **nteract** and **VS Code** are Electron-based frontends that connect to the same HTTP/WebSocket surface.

**JupyterHub** ([source](https://github.com/jupyterhub/jupyterhub)) adds multi-user orchestration on top. Its *five-process model* consists of:

1. **Hub** — the central Tornado process that authenticates users (via an `Authenticator`) and manages the session database.
2. **Configurable HTTP Proxy** (CHP, a Node.js reverse proxy or `traefik`) — routes incoming HTTP/WebSocket traffic to either the Hub or individual user servers.
3. **Spawner** — Hub-side component that starts a `jupyter_server` subprocess (or Kubernetes pod, or Docker container) for each user.
4. **KernelManager** / **MultiKernelManager** — inside the per-user `jupyter_server`, manages kernel lifecycle.
5. **Kernel subprocess** — the language-specific process (e.g., `python -m ipykernel_launcher`) that actually executes user code.

The REST API design is intentionally minimal. A `POST /api/kernels` body contains `{"name": "python3"}` and returns a kernel-id and connection info. A `GET /api/sessions` returns active sessions mapping notebooks to kernels. The single WebSocket at `kernels/<id>/channels` carries all five ZMQ socket channels multiplexed by a `channel` field in each JSON message envelope, described in the next section.

---

## 2. The ZMQ Kernel Protocol

The Jupyter messaging specification ([source](https://jupyter-client.readthedocs.io/en/stable/messaging.html)) defines five ZMQ socket pairs between `jupyter_server` and the kernel process:

| Socket | ZMQ type (server side) | Purpose |
|---|---|---|
| `shell` | DEALER ↔ ROUTER | `execute_request`, `complete_request`, `inspect_request`, … |
| `iopub` | SUB ↔ PUB | `stream`, `display_data`, `execute_result`, `status`, … |
| `stdin` | DEALER ↔ ROUTER | `input_request` / `input_reply` for `input()` calls |
| `control` | DEALER ↔ ROUTER | `interrupt_request`, `shutdown_request`, DAP debug messages |
| `hb` (heartbeat) | REQ ↔ REP | keepalive ping; kernel process echoes any bytes received |

Rather than exposing these five TCP sockets to the browser, `jupyter_server` proxies them over the single `kernels/<id>/channels` WebSocket. Each ZMQ multipart message is encoded as a JSON object with a `channel` field (`"shell"`, `"iopub"`, `"stdin"`, or `"control"`) plus the standard Jupyter message fields:

```python
# Conceptual framing used by jupyter_server's WebSocket handler
# Source: jupyter_server/services/kernels/handlers.py
{
    "channel": "shell",           # which ZMQ socket this came from/goes to
    "header": {
        "msg_id": "abc123",
        "msg_type": "execute_request",
        "session": "<session-id>",
        "username": "",
        "version": "5.3",
        "date": "2026-06-21T10:00:00.000Z"
    },
    "parent_header": {},
    "metadata": {},
    "content": {
        "code": "import cupy as cp; a = cp.zeros(1024)",
        "silent": False,
        "store_history": True,
        "user_expressions": {},
        "allow_stdin": True
    },
    "buffers": []
}
```

The browser-side `@jupyterlab/services` `KernelConnection` class ([source](https://github.com/jupyterlab/jupyterlab/tree/main/packages/services/src/kernel)) opens this WebSocket and demultiplexes by `channel` field, routing messages to the appropriate `IKernelConnection` stream: `anyMessage`, `iopubMessage`, `statusChanged`, and so on. For full protocol detail see Chapter 198 §18.

---

## 3. IPython and Python Kernel Internals

The Python kernel shipped with Jupyter is **ipykernel** ([source](https://github.com/ipython/ipykernel)), which wraps IPython's `InteractiveShell` in a ZMQ-speaking process. The class hierarchy is:

```
InteractiveShell (IPython core)
  └── ZMQInteractiveShell   (ipykernel — adds ZMQ I/O)
        └── IPythonKernel   (ipykernel — handles protocol dispatch)
```

When a cell arrives as an `execute_request`, `IPythonKernel.execute_request()` calls `ZMQInteractiveShell.run_cell()`. The cell source passes through a *transformation pipeline* before compilation:

1. **`input_transformers_cleanup`** — strip leading/trailing whitespace, handle magic escapes (`!`, `?`, `%`).
2. **`input_transformers_post`** — `IPythonInputSplitter` resolves continuation lines.
3. **AST transformation** — registered `ast_transformers` walk the `ast.Module` before `compile()`.
4. **`compile()`** and **`exec()`** — executed in the `InteractiveShell.user_ns` namespace.

```python
# Custom AST transformer that instruments every top-level expression
# to record its execution time without changing display output
import ast

class TimingTransformer(ast.NodeTransformer):
    def visit_Expr(self, node):
        """Wrap each top-level expression in a timing block."""
        return node  # simplified — real version inserts ast.Call nodes

from IPython import get_ipython
ip = get_ipython()
ip.ast_transformers.append(TimingTransformer())
```

**Display dispatch.** When a cell expression evaluates to a non-`None` object, `sys.displayhook` — replaced by IPython with `ZMQShellDisplayHook` — calls `IPython.core.formatters.DisplayFormatter.format()`. This iterates a registry of `BaseFormatter` subclasses in priority order: `_repr_mimebundle_()` → `_repr_html_()` → `_repr_svg_()` → `_repr_png_()` → `_repr_jpeg_()` → `_repr_latex_()` → `_repr_json_()` → `_repr_pretty_()`. The result is a dict of `{mime_type: data}` packed into a `display_data` or `execute_result` message on iopub.

```python
class GPUArray:
    """A minimal example implementing rich display for a GPU buffer."""
    def __init__(self, data: bytes, shape: tuple):
        self._data = data
        self.shape = shape

    def _repr_mimebundle_(self, include=None, exclude=None):
        import base64
        png_bytes = self._render_to_png()          # hypothetical GPU→PNG path
        return {
            "image/png": base64.b64encode(png_bytes).decode(),
            "text/plain": f"GPUArray(shape={self.shape})"
        }, {}  # (data dict, metadata dict)

    def _render_to_png(self) -> bytes:
        raise NotImplementedError
```

**stdout/stderr replacement.** ipykernel replaces `sys.stdout` and `sys.stderr` with `ipykernel.iostream.OutStream`, which batches writes and publishes `stream` messages (`{"name": "stdout", "text": "…"}`) to the iopub socket rather than writing to the process TTY. This is why `print()` output appears in the notebook even though the kernel is a subprocess with no terminal.

**Magic commands** are implemented through `MagicsMixin`. A `%line_magic` receives everything after the magic name as a string; a `%%cell_magic` receives the first line and the cell body as separate strings. Commonly used GPU-related magics include `%%timeit` (benchmarking), `%matplotlib inline` (matplotlib backend switching), `%load_ext` (activating extensions like `%%cython` or `%%cuda` via pycuda). Custom magics are registered with `@register_line_magic` or `@register_cell_magic`.

**Traitlets** ([source](https://github.com/ipython/traitlets)) is the reactive configuration system underpinning all Jupyter components. `HasTraits` subclasses declare typed attributes that fire `observe` callbacks on change and can be validated with `validate`.

```python
import traitlets

class KernelConfig(traitlets.HasTraits):
    timeout = traitlets.Float(60.0, help="Cell execution timeout in seconds")
    gpu_device = traitlets.Int(0, help="CUDA/ROCm device index")

    @traitlets.validate("gpu_device")
    def _validate_gpu_device(self, proposal):
        if proposal["value"] < 0:
            raise traitlets.TraitError("gpu_device must be non-negative")
        return proposal["value"]

    @traitlets.observe("gpu_device")
    def _gpu_device_changed(self, change):
        print(f"Switching to GPU {change['new']}")

cfg = KernelConfig()
cfg.gpu_device = 1   # triggers _gpu_device_changed
```

**Debugger integration.** Since ipykernel 6.0 and JupyterLab 3.0, the `control` socket carries [Debug Adapter Protocol](https://microsoft.github.io/debug-adapter-protocol/) (DAP) messages. The kernel embeds `debugpy` ([source](https://github.com/microsoft/debugpy)), which listens on a local TCP port and translates DAP `setBreakpoints` / `stackTrace` / `variables` requests forwarded through the `control` socket. This allows JupyterLab's debugger UI (the bug icon in the sidebar) to set breakpoints and step through cell code using the same protocol as VS Code.

---

## 4. Multi-Kernel Support

`jupyter_server` manages kernels through two layered classes: `KernelManager` (single kernel) and `MultiKernelManager` (a dict of `KernelManager` instances keyed by kernel-id) ([source](https://github.com/jupyter-server/jupyter_server/blob/main/jupyter_server/services/kernels/kernelmanager.py)). The lifecycle methods are `start_kernel()`, `interrupt_kernel()` (sends SIGINT on Linux), `restart_kernel()`, and `shutdown_kernel()`.

**kernelspec** files declare available kernels. Each spec lives at `~/.local/share/jupyter/kernels/<name>/kernel.json` (user scope) or `/usr/share/jupyter/kernels/<name>/kernel.json` (system scope):

```json
{
  "argv": ["python3", "-m", "ipykernel_launcher", "-f", "{connection_file}"],
  "display_name": "Python 3 (ipykernel)",
  "language": "python",
  "interrupt_mode": "signal",
  "env": {
    "CUDA_VISIBLE_DEVICES": "0",
    "OMP_NUM_THREADS": "4"
  },
  "metadata": {
    "debugger": true
  }
}
```

```bash
# List installed kernels
jupyter kernelspec list

# Install a custom kernelspec directory
jupyter kernelspec install ./my-kernel-dir --user --name my-gpu-kernel
```

**xeus** ([source](https://github.com/jupyter-xeus/xeus)) is a C++ framework for writing kernels without implementing the ZMQ protocol from scratch. The implementer subclasses `xeus::xinterpreter` and overrides virtual methods:

```cpp
// xeus-based kernel interpreter skeleton (simplified; consult xinterpreter.hpp
// for the current async variant introduced in xeus 5.x)
// Source: https://github.com/jupyter-xeus/xeus/blob/main/include/xeus/xinterpreter.hpp
#include "xeus/xinterpreter.hpp"

class my_interpreter : public xeus::xinterpreter {
public:
    void configure_impl() override { /* one-time setup */ }

    // Note: xeus 5.x introduced a request-context parameter for async execution.
    // The conceptual interface is: receive 'code', publish results via
    // publish_execution_result(), return a status JSON.
    nl::json execute_request_impl(
        int execution_counter,
        const std::string& code,
        bool silent,
        bool store_history,
        nl::json user_expressions,
        bool allow_stdin) override
    {
        // evaluate 'code', publish display_data via publish_execution_result()
        nl::json result;
        result["status"] = "ok";
        result["execution_count"] = execution_counter;
        return result;
    }

    nl::json complete_request_impl(const std::string& code, int cursor_pos) override {
        return nl::json{{"matches", nl::json::array()},
                        {"cursor_start", cursor_pos},
                        {"cursor_end", cursor_pos},
                        {"status", "ok"}};
    }
    // ... inspect_request_impl, is_complete_request_impl, kernel_info_request_impl
};
```

`xeus` handles all ZMQ socket creation, HMAC authentication, and message framing, leaving the implementer to focus only on language semantics. Projects built on xeus include **xeus-python** (alternative CPython kernel), **xeus-cling** (CUDA C++ via the Cling LLVM-based interpreter), and **xeus-lua**.

**Kernel provisioners** (introduced in `jupyter_client` 7) decouple kernel *launch* from local process management. `LocalProvisioner` is the default (fork + exec). `KubernetesProvisioner` (via `jupyter_enterprise_gateway`) launches kernels as Kubernetes pods with GPU resource requests:

```python
# c.KubeSpawner / enterprise-gateway kernel config snippet
# Source: https://github.com/jupyter-server/enterprise_gateway
kernel_spec_overrides = {
    "env": {"KERNEL_NAMESPACE": "gpu-kernels"},
    "resources": {
        "limits": {"nvidia.com/gpu": "1", "memory": "16Gi"},
        "requests": {"nvidia.com/gpu": "1"}
    }
}
```

**SSH-tunnelled remote kernels** allow connecting to a kernel started on a remote machine:

```bash
# On remote host: start kernel and write connection file
python -m ipykernel_launcher --ip=127.0.0.1

# On local host: SSH tunnel all five ZMQ ports, then connect
jupyter console --existing kernel-12345.json \
    --ssh user@gpu-server.example.com
```

---

## 5. The Notebook File Format

A `.ipynb` file is a JSON document conforming to the **nbformat** schema ([source](https://nbformat.readthedocs.io/en/latest/)). The current stable version is **nbformat 4.5**. The top-level structure:

```json
{
  "nbformat": 4,
  "nbformat_minor": 5,
  "metadata": {
    "kernelspec": {
      "display_name": "Python 3 (ipykernel)",
      "language": "python",
      "name": "python3"
    },
    "language_info": {
      "name": "python",
      "version": "3.11.9"
    }
  },
  "cells": [
    {
      "cell_type": "code",
      "execution_count": 1,
      "metadata": {},
      "source": ["import cupy as cp\n", "a = cp.zeros((1024, 1024))\n", "print(a.device)"],
      "outputs": [
        {
          "output_type": "stream",
          "name": "stdout",
          "text": ["<CUDA Device 0>\n"]
        }
      ]
    }
  ]
}
```

Output types in `outputs`: `stream` (stdout/stderr text), `display_data` (rich MIME bundle from `display()`), `execute_result` (the cell's return value), and `error` (exception traceback). The `data` field of `display_data` / `execute_result` is a dict of MIME keys: `text/plain`, `image/png` (base64), `text/html`, `application/json`, etc.

**nbformat library** provides `read()`, `write()`, `validate()`, and `convert()` for moving notebooks between versions. **nbconvert** ([source](https://github.com/jupyter/nbconvert)) converts notebooks to HTML, LaTeX, or scripts using Jinja2 templates:

```python
import nbformat
from nbconvert import HTMLExporter
from nbconvert.preprocessors import ExecutePreprocessor

# Load, execute, and export a notebook programmatically
with open("analysis.ipynb") as f:
    nb = nbformat.read(f, as_version=4)

# Execute all cells (requires a running kernel)
ep = ExecutePreprocessor(timeout=600, kernel_name="python3")
ep.preprocess(nb, {"metadata": {"path": "."}})

# Export to HTML
exporter = HTMLExporter()
exporter.template_name = "lab"          # JupyterLab-styled output
html_body, resources = exporter.from_notebook_node(nb)

with open("analysis.html", "w") as f:
    f.write(html_body)
```

**papermill** ([source](https://github.com/nteract/papermill)) executes parameterised notebooks by injecting parameter values into a tagged cell, enabling pipeline-style batch execution: `pm.execute_notebook("template.ipynb", "output.ipynb", parameters={"lr": 0.001, "batch_size": 64})`.

**jupytext** ([source](https://github.com/mwouts/jupytext)) bidirectionally synchronises `.ipynb` files with plain `.py` files (percent format: cells delimited by `# %%`) or MyST Markdown, enabling version-controlled notebooks without JSON diffs.

---

## 6. JupyterLab Rendering Pipeline

JupyterLab is a TypeScript single-page application built on the **Lumino** library ([source](https://github.com/jupyterlab/lumino)), the successor to PhosphorJS. Lumino provides the widget system: `Widget` (a DOM-backed UI node), `Panel` (container), `DockPanel` (drag-and-drop tab layout), `CommandRegistry` (keyboard shortcuts and menu items), and `MessageLoop` (a micro-task scheduler for widget lifecycle messages: `Msg.AfterAttach`, `Msg.BeforeDetach`, `Msg.UpdateRequest`).

JupyterLab's extension system is token-based dependency injection. Each plugin is a `JupyterFrontEndPlugin<T>` object declaring its `provides` token, `requires` tokens, and `activate` factory function. Extensions register themselves with `JupyterFrontEnd.registerPlugin()` and are bundled into the SPA via webpack at build time (or loaded as federated modules at runtime in JupyterLab 3+):

```typescript
// Registering a custom MIME renderer extension for JupyterLab
// Source: https://jupyterlab.readthedocs.io/en/stable/extension/mime_renderer.html
import {
  JupyterFrontEnd,
  JupyterFrontEndPlugin
} from "@jupyterlab/application";
import {
  IRenderMimeRegistry,
  IRenderMime
} from "@jupyterlab/rendermime";

class GPUArrayRenderer implements IRenderMime.IRenderer {
  readonly mimeType = "application/x-gpu-array";
  constructor(private options: IRenderMime.IRendererOptions) {}

  async renderModel(model: IRenderMime.IMimeModel): Promise<void> {
    const data = model.data[this.mimeType] as string;  // base64 array data
    const canvas = document.createElement("canvas");
    this.options.node.appendChild(canvas);
    // ... decode and render via WebGL
  }
}

const extension: JupyterFrontEndPlugin<void> = {
  id: "my-gpu-array-renderer",
  autoStart: true,
  requires: [IRenderMimeRegistry],
  activate: (app: JupyterFrontEnd, registry: IRenderMimeRegistry) => {
    registry.addFactory({
      safe: true,
      mimeTypes: ["application/x-gpu-array"],
      createRenderer: (options) => new GPUArrayRenderer(options),
    }, 100 /* rank; higher rank = tried first */);
  }
};

export default extension;
```

The cell output area is managed by `OutputArea` (Lumino widget) backed by an `OutputAreaModel` (list of `IOutputModel`). When a `display_data` message arrives from iopub, `OutputArea` iterates the `IRenderMimeRegistry` to find the highest-rank renderer that handles one of the MIME types in the `data` dict, creates a renderer instance, and calls `renderModel()`. Built-in renderers handle `text/plain`, `text/html`, `image/png`, `image/svg+xml`, `application/json`, `application/vnd.vega.v5+json`, and `application/vnd.plotly.v1+json`.

**Voilà** ([source](https://github.com/voila-dashboards/voila)) serves notebook outputs as a standalone web page without the JupyterLab chrome: it executes all cells server-side and streams output MIME data to a minimal HTML/JS client. The flag `--strip_sources` prevents the cell source code from appearing in the page.

**anywidget** ([source](https://github.com/manzt/anywidget)) enables custom widget authoring without the classic ipywidgets build toolchain. The Python class subclasses `anywidget.AnyWidget` and sets `_esm` to an ECMAScript module string. The frontend module exports a `render({ model, el })` function (or a default object with `render` and optional `initialize` keys) that manipulates `el` directly:

```python
import anywidget
import traitlets

class ColorPicker(anywidget.AnyWidget):
    _esm = """
    export function render({ model, el }) {
        const input = document.createElement("input");
        input.type = "color";
        input.value = model.get("color");
        input.addEventListener("input", () => {
            model.set("color", input.value);
            model.save_changes();
        });
        model.on("change:color", () => {
            input.value = model.get("color");
        });
        el.appendChild(input);
    }
    """
    color = traitlets.Unicode("#ff0000").tag(sync=True)
```

No webpack, no npm install required — the ESM string is served inline.

---

## 7. Interactive Widgets: ipywidgets and the comm Protocol

The Jupyter `comm` protocol adds a lightweight bidirectional message channel on top of the existing ZMQ shell and iopub sockets. The kernel opens a comm with `comm_open` (on shell); the frontend acknowledges. Subsequent `comm_msg` messages flow in both directions on shell (kernel→frontend) and iopub (broadcast). `comm_close` tears down the channel.

**ipywidgets** ([source](https://github.com/jupyter-widgets/ipywidgets)) uses one comm per widget instance. The Python `Widget` base class holds `traitlets` attributes tagged with `.tag(sync=True)`; any change to a synced trait triggers serialisation and a `comm_msg` carrying `{"method": "update", "state": {"value": 42}}` to the frontend. The JS `WidgetModel` receives this and updates its Backbone.js model, triggering re-render of any bound view (`WidgetView`).

**End-to-end slider walkthrough:**

1. Python: `w = ipywidgets.IntSlider(value=0, min=0, max=100)` → `comm_open` to target `"jupyter.widget"` with `data={"state": {"_model_name": "IntSliderModel", "_model_module": "@jupyter-widgets/controls", "_view_name": "IntSliderView", "_view_module": "@jupyter-widgets/controls", "value": 0, "min": 0, "max": 100}, "buffer_paths": []}` ([source](https://github.com/jupyter-widgets/ipywidgets/blob/main/packages/schema/messages.md)).
2. Browser: `WidgetManager` instantiates `IntSliderModel` and `IntSliderView`; view renders `<input type="range">`.
3. User drags slider to 50 → JS calls `model.set("value", 50)`, `model.save_changes()` → `comm_msg {"method": "update", "state": {"value": 50}}` on shell socket.
4. Python: `Widget._handle_msg()` receives the comm message, sets `self.value = 50`, fires registered `@observe("value")` callbacks.
5. An `@observe` callback may trigger a GPU computation and call `display(result)` → `display_data` on iopub → browser updates output area.

```python
import ipywidgets as widgets
import cupy as cp

# GPU computation linked to a slider
device_mem = cp.zeros((1024, 1024), dtype=cp.float32)

slider = widgets.IntSlider(value=1, min=1, max=64, description="Scale:")
output = widgets.Output()

@output.capture(clear_output=True)
def on_scale_change(change):
    scale = change["new"]
    # Run GPU kernel — stays in device memory
    result = device_mem * scale
    print(f"GPU result mean: {float(result.mean()):.4f}  "
          f"(VRAM in use: {cp.get_default_memory_pool().used_bytes() // 1024**2} MB)")

slider.observe(on_scale_change, names="value")
widgets.VBox([slider, output])
```

**Widget protocol version negotiation.** When ipywidgets opens a comm, the `comm_open` metadata carries `{"version": "2.1.0"}` (the widget messaging protocol version, distinct from the ipywidgets Python package version). The frontend checks this against its own supported range; a mismatch produces the classic "widget not found" error that often means the frontend extension is not installed or is at the wrong version.

---

## 8. GPU Computing in Notebooks

The kernel subprocess owns all GPU state for the notebook's lifetime. A CUDA context is created on first GPU access (typically when a CuPy or PyTorch import triggers `cuInit`), and that context persists until the kernel process exits or is restarted. All cells in the notebook share that single context.

**CuPy** ([source](https://github.com/cupy/cupy)) provides a NumPy-compatible array API that lives entirely on-device:

```python
import cupy as cp

# Cell 1 — allocates 8 MB on GPU 0; allocation persists across cells
a = cp.random.randn(1024, 1024, dtype=cp.float32)

# Cell 2 — `a` is still alive; this adds 8 MB
b = cp.linalg.eigvalsh(a @ a.T)  # GPU-side linear algebra
print(f"Eigenvalue range: {float(b.min()):.3f} to {float(b.max()):.3f}")

# Cell 3 — explicit device selection
with cp.cuda.Device(1):          # switch to GPU 1
    c = cp.zeros_like(a)

# Cell 4 — pool-based allocator reports usage
pool = cp.get_default_memory_pool()
print(f"GPU memory in use: {pool.used_bytes() // 1024**2} MB")
print(f"GPU memory cached: {pool.total_bytes() // 1024**2} MB")
pool.free_all_blocks()           # return cached blocks to CUDA
```

**PyTorch GPU memory across cells.** Tensors allocated in one cell remain alive as long as the Python variable is reachable from the kernel's global namespace (`In`, `Out`, `_`, or any user variable). Deleting the variable and calling `torch.cuda.empty_cache()` returns allocator-cached blocks but does *not* reclaim memory held by live tensors.

```python
import torch

# Cell 1
x = torch.randn(4096, 4096, device="cuda")

# Cell 2 — inspect fragmentation and peak usage
print(torch.cuda.memory_summary(device=0, abbreviated=True))

# Cell 3 — free cached blocks (does not free `x`)
torch.cuda.empty_cache()

# Cell 4 — truly free: delete the variable
del x
torch.cuda.empty_cache()
print(f"Free after del: {torch.cuda.mem_get_info()[0] // 1024**2} MB")
```

**JAX in notebooks.** JAX JIT-compiles functions on first call, producing XLA HLO graphs compiled by the XLA compiler to CUDA/ROCm PTX/HSAIL. Compilation results are cached in memory by default; enable persistent caching to avoid re-compilation across kernel restarts:

```python
import jax
import jax.numpy as jnp

# Persistent compilation cache — must be set before first JAX operation
jax.config.update("jax_compilation_cache_dir", "/tmp/jax_cache")
# [Source: https://docs.jax.dev/en/latest/persistent_compilation_cache.html]

# Equivalent: set JAX_COMPILATION_CACHE_DIR env var in kernel.json

@jax.jit
def matmul_relu(a, b):
    return jax.nn.relu(a @ b)

# First call compiles to GPU; subsequent calls reuse the compilation
key = jax.random.PRNGKey(0)
a = jax.random.normal(key, (2048, 2048))
b = jax.random.normal(key, (2048, 2048))
result = matmul_relu(a, b)  # compiles on first execution
print(jax.devices("gpu"))
```

**ROCm/HIP in notebooks.** PyTorch on ROCm exposes `torch.version.hip` (e.g., `"6.2.0"`) when built against ROCm. GPU detection and memory monitoring use the same PyTorch API; `rocm-smi` can be called via subprocess:

```python
import subprocess, torch

if torch.version.hip:
    result = subprocess.run(["rocm-smi", "--showmeminfo", "vram"],
                            capture_output=True, text=True)
    print(result.stdout)
```

**Multi-GPU.** The `CUDA_VISIBLE_DEVICES` environment variable in `kernel.json` restricts which devices the kernel sees, useful for sharing a node among multiple notebook users. Within a notebook, `torch.cuda.device_count()` and `cp.cuda.runtime.getDeviceCount()` return the number of visible devices. Multi-GPU tensor parallelism requires explicit device placement (`torch.device("cuda:1")`) or a framework like DeepSpeed or PyTorch's `DeviceMesh`.

---

## 9. GPU Visualisation

Interactive visualisation in notebooks faces a fundamental constraint: the kernel process runs server-side and has no display server. The dominant patterns are (1) render on GPU, export to PNG/typed-array, send to browser as `display_data`; and (2) send raw data to browser, render client-side with WebGL/WebGPU.

**ipyvolume** ([source](https://github.com/maartenbreddels/ipyvolume)) renders 3D volumes, scatter plots, and quiver plots via Three.js WebGL in the browser. Data arrays pass through `CuPy/PyTorch → .get()/cpu().numpy() → base64 PNG or typed Float32Array → comm_msg → JS → WebGL texture upload`. There is no zero-copy path; the round-trip through host memory is the current bottleneck for large tensors.

**plotly with WebGL traces** ([source](https://github.com/plotly/plotly.py)) allows rendering millions of points using `go.Scattergl`, `go.Heatmapgl`, or `go.Scatter3d`. The figure JSON is embedded in a `display_data` bundle with MIME type `application/vnd.plotly.v1+json`; the browser-side Plotly.js library reads typed arrays from this bundle and uploads them to WebGL. Data must be transferred as JSON-serialisable arrays (host memory), not raw GPU tensors.

**pydeck** ([source](https://github.com/visgl/deck.gl/tree/master/bindings/pydeck)) exposes the deck.gl WebGL point-cloud, tile, and heatmap layers. A `pydeck.Deck` object implements `_repr_html_()` for static HTML display; in Jupyter with the `@deck.gl/jupyter-widget` extension installed, it also registers as an ipywidget that renders via WebGL2, enabling two-way interaction (click events, viewport changes) between the browser and Python.

```python
import pydeck as pdk
import cupy as cp
import numpy as np

# Compute on GPU, transfer to CPU for display
gpu_points = cp.random.uniform(-122.5, -122.4, size=(100_000, 2))
cpu_points = cp.asnumpy(gpu_points)  # GPU → CPU transfer

layer = pdk.Layer(
    "ScatterplotLayer",
    data=[{"position": p.tolist()} for p in cpu_points[:1000]],  # subsample
    get_position="position",
    get_radius=5,
    get_fill_color=[255, 0, 0, 160],
)
deck = pdk.Deck(layers=[layer], initial_view_state=pdk.ViewState(
    latitude=-122.45, longitude=37.76, zoom=11))
deck  # display via IRenderMimeRegistry handler
```

**wgpu-py / rendercanvas.** The `rendercanvas` package ([source](https://github.com/pygfx/rendercanvas)) provides a Jupyter backend that uses `jupyter_rfb` (a remote frame buffer ipywidget) to stream rendered frames from a server-side `wgpu` (WebGPU-native) canvas to the browser. The canvas is imported via `rendercanvas.auto.RenderCanvas`, which selects the Jupyter backend automatically when running in a notebook. This is a server-side render-and-stream approach; a fully browser-side WebGPU widget is tracked as an `anywidget` + `wgpu-py` target for future work. Note: the API is evolving — consult the `rendercanvas` docs for the current import path.

**anywidget + WebGL custom visualisation.** For truly interactive in-browser GPU rendering, `anywidget` can host a WebGL canvas that communicates with Python over the comm protocol. The pattern is: Python computes on the GPU, serialises the result to a typed array (host memory), sends it via a synced traitlet, and the JS `render` function uploads it to a WebGL texture or vertex buffer:

```python
import anywidget
import traitlets
import numpy as np

class WebGLHeatmap(anywidget.AnyWidget):
    """Renders a 2D float32 array as a WebGL heatmap in the browser."""
    _esm = """
    export function render({ model, el }) {
        const canvas = document.createElement("canvas");
        canvas.width = 512;
        canvas.height = 512;
        el.appendChild(canvas);

        const gl = canvas.getContext("webgl2");
        // Create a 2D texture for heatmap data
        const tex = gl.createTexture();
        gl.bindTexture(gl.TEXTURE_2D, tex);

        function updateTexture() {
            // data is a base64-encoded Float32Array
            const b64 = model.get("data_b64");
            if (!b64) return;
            const binary = atob(b64);
            const bytes = new Uint8Array(binary.length);
            for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
            const floats = new Float32Array(bytes.buffer);
            const side = Math.sqrt(floats.length) | 0;

            gl.texImage2D(gl.TEXTURE_2D, 0, gl.R32F, side, side, 0,
                          gl.RED, gl.FLOAT, floats);
            gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
            // ... draw fullscreen quad with colormap fragment shader
        }

        model.on("change:data_b64", updateTexture);
        updateTexture();
    }
    """
    # Synced trait: base64-encoded raw float32 bytes
    data_b64 = traitlets.Unicode("").tag(sync=True)

    def update_from_cupy(self, gpu_array):
        """Transfer CuPy array to browser: GPU → CPU → base64 → comm_msg."""
        import base64, cupy as cp
        host = cp.asnumpy(gpu_array.astype(cp.float32).ravel())
        self.data_b64 = base64.b64encode(host.tobytes()).decode()
```

The `update_from_cupy` method illustrates the unavoidable host-memory round-trip for GPU-to-browser data in the current comm protocol. A zero-copy path would require the WebSocket binary-frame extension proposed for the kernel protocol v2 (see §13).

---

## 10. Real-Time Collaboration

JupyterLab's real-time collaboration (RTC) feature is implemented using **Yjs** ([source](https://github.com/yjs/yjs)), a CRDT (Conflict-free Replicated Data Type) library. Yjs documents (`Y.Doc`) contain typed shared data structures: `Y.Text` (for cell source), `Y.Map` (for cell metadata), and `Y.Array` (for the cells list). Concurrent edits from multiple users are merged deterministically without a central coordinator — each insert/delete carries a logical clock (Lamport timestamp + client ID) that determines its position in the merged sequence.

**`@jupyter/ydoc`** ([source](https://github.com/jupyter-server/jupyter_ydoc)) maps the Yjs types onto the `.ipynb` schema: `YNotebook` holds a `Y.Array` of `YCell` objects; each `YCell` holds a `Y.Text` for `source` and a `Y.Map` for `metadata`. Every keystroke in a cell is translated into a Yjs `Y.Text` delta and broadcast to all collaborators.

**`jupyter-collaboration`** ([source](https://github.com/jupyterlab/jupyter-collaboration)) is the server extension that provides the WebSocket room endpoint. Clients connect to `/api/collaboration/room/json:notebook:{file-path}` (for `.ipynb` files) or `/api/collaboration/room/text:file:{file-path}` (for text files). The server uses `pycrdt-websocket` to maintain a server-side Yjs document replica and relay updates between all connected clients using the `y-websocket` binary protocol.

**Awareness states** carry ephemeral per-user data (cursor position, selection, username, user colour) without being persisted to the Yjs document. Each client broadcasts its awareness state as a compact binary update; all peers render coloured cursors and selections in real time.

**Conflict resolution example.** User A types `"hello"` and User B simultaneously types `"world"` into the same empty cell source. Yjs assigns each character insertion a unique (client-id, clock) tuple. The merge deterministically orders all insertions by clock, breaking ties by client-id. The merged result might be `"heworldllo"` or `"hellworld o"` depending on the exact timing — not semantically ideal, but *consistent* across all peers without coordination. In practice, the character-level granularity means conflicts in code cells are rare; authors typically edit different parts of the notebook.

The `y-websocket` binary protocol frames each Yjs update as a Uint8Array (a compact binary-encoded CRDT delta). The server relays these updates to all room participants; each client applies them to its local `Y.Doc` replica. The protocol is content-agnostic — it does not parse the notebook schema, only propagates opaque binary update blobs. The `pycrdt` Python library ([source](https://github.com/jupyter-server/pycrdt)) provides server-side access to the same `Y.Doc` model, enabling features such as server-side save-to-disk and notebook format validation on update.

```javascript
// Browser-side: subscribing to Yjs notebook document updates
// Source: @jupyter/ydoc package
import * as Y from "yjs";
import { WebsocketProvider } from "y-websocket";

const doc = new Y.Doc();
// Connect to the jupyter-collaboration room for a specific notebook
const provider = new WebsocketProvider(
    "ws://localhost:8888/api/collaboration/room",
    "json:notebook:/home/user/work/analysis.ipynb",
    doc
);

// Access the cells array
const ycells = doc.getArray("cells");
ycells.observe(event => {
    // event.changes.delta contains insert/delete/retain operations
    console.log("Cells changed:", event.changes.delta);
});

// Awareness: broadcast local cursor position to collaborators
provider.awareness.setLocalStateField("user", {
    name: "Alice",
    color: "#3b82f6"
});
provider.awareness.on("change", ({ added, updated, removed }) => {
    const states = provider.awareness.getStates();
    // Render remote cursors from `states` map
});
```

---

## 11. JupyterLite: WASM Kernels in the Browser

**JupyterLite** ([source](https://github.com/jupyterlite/jupyterlite)) is a fully static deployment of JupyterLab that requires no server process. A **service worker** intercepts `fetch()` calls from the JupyterLab SPA and emulates the `jupyter_server` REST API and WebSocket endpoints entirely in the browser. Kernels run as Web Workers.

The default kernel is based on **Pyodide** ([source](https://github.com/pyodide/pyodide)), CPython compiled to WebAssembly via Emscripten. Pyodide provides `micropip` for installing pure-Python packages from PyPI, and ships pre-built Wasm versions of numpy, pandas, scipy, matplotlib, and other packages with C extensions. Because there are no Unix sockets in the browser, the ZMQ protocol is replaced by `SharedArrayBuffer`-based message passing between the service worker and the kernel Web Worker.

Limitations relative to a server-side kernel:
- **No GPU access**: CUDA, ROCm, and Vulkan compute are unavailable from Wasm. WebGPU (via `wgpu` compiled to Wasm) is an emerging exception — see §13.
- **No native ZMQ**: the ZMQ protocol is emulated with `SharedArrayBuffer`/`Atomics`; binary buffer transfers are limited by the `SharedArrayBuffer` security requirements (`COOP`/`COEP` HTTP headers).
- **OPFS filesystem**: persistent storage uses the Origin Private File System API; there is no access to the host filesystem.
- **Package constraints**: packages with C extensions must be pre-compiled to `wasm32-emscripten`. Not all PyPI packages are available.

```bash
# Build a JupyterLite static site with custom packages
pip install jupyterlite-core jupyterlite-pyodide-kernel

# jupyter_lite_config.json controls which packages are bundled
jupyter lite build --output-dir dist/

# Serve locally for testing
jupyter lite serve
# → site available at http://localhost:8000; open in browser, no server required
```

```json
{
  "PyodideAddon": {
    "packages": ["numpy", "pandas", "matplotlib", "scikit-learn"]
  }
}
```

The **xeus-python** Wasm kernel ([source](https://github.com/jupyter-xeus/xeus-python)) is a lighter alternative to Pyodide for notebooks that do not need the full Python standard library; it starts faster and uses less memory. Both kernels are installable as JupyterLite extensions.

---

## 12. Linux-Specific: GPU Access, Containers, and JupyterHub

On Linux, the kernel subprocess needs read-write access to GPU device nodes. For NVIDIA this means `/dev/nvidia0`, `/dev/nvidiactl`, and `/dev/nvidia-uvm`; for AMD ROCm it means `/dev/kfd` (KFD compute interface) and `/dev/dri/renderD128` (DRM render node). Udev rules from the driver packages assign `uaccess` tags so that the logged-in user gains access automatically on a desktop system; in containers and multi-user server environments explicit grants are required.

**Docker GPU passthrough for NVIDIA** uses `nvidia-container-toolkit` ([source](https://github.com/NVIDIA/nvidia-container-toolkit)):

```bash
# Run a JupyterLab container with all NVIDIA GPUs visible
docker run --gpus all \
    -p 8888:8888 \
    -v "$HOME/notebooks:/home/jovyan/work" \
    jupyter/scipy-notebook

# Or restrict to specific GPUs
docker run --gpus '"device=0,1"' jupyter/scipy-notebook

# For AMD ROCm
docker run --device /dev/kfd --device /dev/dri/renderD128 \
    --group-add video \
    -p 8888:8888 rocm/pytorch:latest \
    jupyter lab --ip=0.0.0.0
```

`nvidia-container-toolkit` works via CDI (Container Device Interface) or the legacy `nvidia-container-runtime` shim. It bind-mounts `/dev/nvidia*` device nodes and injects `libcuda.so` into the container's library search path via a preload mechanism. The CDI spec (`nvidia-ctk cdi generate`) produces a JSON file listing all required bind mounts and environment variable injections for a given device set.

**JupyterHub Kubernetes spawner** ([source](https://github.com/jupyterhub/kubespawner)):

```python
# jupyterhub_config.py — GPU resource request per user pod
c.KubeSpawner.extra_resource_limits = {"nvidia.com/gpu": "1"}
c.KubeSpawner.extra_resource_guarantees = {"nvidia.com/gpu": "1"}
c.KubeSpawner.image = "jupyter/scipy-notebook:latest"
c.KubeSpawner.namespace = "jupyter-gpu"
```

The NVIDIA GPU Operator installs the device plugin daemonset that advertises `nvidia.com/gpu` resources to the Kubernetes scheduler. For AMD, the `amd.com/gpu` resource is advertised by the ROCm device plugin.

**MIG (Multi-Instance GPU)** partitions a single A100 (40 GB) or H100 (80 GB) into isolated slices. On the A100-40GB, up to 7 `1g.5gb` instances (1 GPC, 5 GB memory each) or 3 `2g.10gb` instances can be created simultaneously ([source](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)). Each instance appears as a separate device to the container:

```bash
# List available MIG profiles on an A100-40GB
nvidia-smi mig -lgip

# Create 7 x 1g.5gb instances
nvidia-smi mig -cgi 19,19,19,19,19,19,19 -C

# Each notebook pod gets one MIG device via NVIDIA_VISIBLE_DEVICES
# e.g., MIG-GPU-abc123/0/0
```

**cgroup v2 memory and CPU limits.** JupyterHub's `SystemdSpawner` creates a transient systemd user unit for each notebook server, setting `MemoryMax` and `CPUQuota` via the unit's `[Service]` section. GPU memory limits require either MIG partitioning or NVML accounting mode (`nvmlDeviceSetAccountingMode`), which *tracks* per-process GPU memory usage but does not cap it — true GPU memory capping requires MIG or time-slicing with `nvidia-smi --compute-mode=EXCLUSIVE_PROCESS`.

**`jupyter-server-proxy`** ([source](https://github.com/jupyterhub/jupyter-server-proxy)) reverse-proxies arbitrary web services (TensorBoard on port 6006, Dask dashboard on port 8787) through the `jupyter_server` HTTP server, making them accessible to notebook users without opening additional firewall ports.

---

## 13. Roadmap

**JupyterLab 4.x** (current stable as of mid-2026) delivers virtual scrolling in large output areas (eliminating DOM node explosion from print-heavy cells), real-time collaboration promoted from an opt-in extension to a core feature, and Lumino 2 with improved widget lifecycle management and TypeScript strict-mode compliance.

**Notebook format v5** is under discussion, with richer cell metadata fields for execution provenance, cell-level tags for papermill integration, and binary buffer support for embedding large data arrays without base64 encoding.

**WebSocket kernel protocol v2** (tracked in [jupyter-client](https://github.com/jupyter/jupyter_client)) proposes binary WebSocket frames for zero-copy array transfer between the kernel and browser, eliminating the base64 encoding step that currently adds ~33% overhead to large `display_data` payloads (e.g., a 100 MB numpy array image).

**Jupyter AI (`jupyter_ai`)** ([source](https://github.com/jupyterlab/jupyter_ai)) adds LLM-backed cell completion and an `%%ai` magic that sends cell content to a configurable LLM backend. The `%%ai ollama:llama3` form routes requests to a local Ollama server (see Chapter 124); the `%%ai anthropic:claude-sonnet-4-5` form uses the Anthropic API. The `jupyter_ai` architecture uses LangChain as its provider abstraction layer.

**WebGPU widget ecosystem.** `anywidget` combined with browser-native WebGPU enables GPU-accelerated interactive widgets that render entirely in the browser without server-side OpenGL. This approach avoids the PNG-export bottleneck of server-side vispy/moderngl and the latency of streaming frames via `jupyter_rfb`. The `rendercanvas` `anywidget` backend is being developed to make this accessible from `wgpu-py`. Note: browser WebGPU availability and the `rendercanvas`/`anywidget` integration are actively evolving — consult upstream for the current API.

**JupyterLite WebGPU** is a natural extension: once Wasm-compiled `wgpu` is stable, JupyterLite notebooks running in a WebGPU-capable browser could perform real GPU compute (matrix multiplications, shader-based image processing) without any server infrastructure, enabling fully static, GPU-accelerated notebooks deployable from a CDN.

**Jupyter AI (`jupyter_ai`) in practice.** The `%%ai` cell magic sends the cell body to the configured LLM provider and streams the response as Markdown into the cell output. Multiple providers are supported simultaneously:

```python
# Install jupyter_ai and the ollama provider
# pip install jupyter_ai jupyter_ai_magics langchain_community

# Cell 1: load the extension
%load_ext jupyter_ai_magics

# Cell 2: ask a local Ollama model to explain GPU memory layout
# %%ai ollama:llama3 sends the prompt to http://localhost:11434
```

```
%%ai ollama:llama3
Explain the difference between CuPy's default memory pool and
PyTorch's caching allocator, and when each is preferred in notebooks.
```

```python
# Cell 3: code generation via cloud provider (requires API key in env)
# %%ai anthropic:claude-sonnet-4-5
```

```
%%ai anthropic:claude-sonnet-4-5 -f code
Write a CuPy kernel that computes a 2D Gaussian blur on a float32 array
using shared memory. Include a host wrapper function.
```

The `jupyter_ai` configuration (via `~/.jupyter/jupyter_ai_config.py`) maps provider names to their connection parameters. When Ollama is used as the backend, the entire inference path traverses the GGUF/llama.cpp stack described in Chapter 124 — model weights are memory-mapped from disk, executed on the GPU, and tokens are streamed back to the notebook as `display_data` `text/markdown` chunks.

**Kernel protocol binary extensions.** The current messaging protocol version 5.x encodes all `display_data` payloads as JSON. A 512×512 float32 array (1 MB) becomes ~1.33 MB base64 in the JSON envelope, plus JSON parsing overhead. The proposed kernel protocol v2 — discussed in the `jupyter_client` issue tracker — would allow binary ZMQ frames to carry raw buffer data alongside the JSON header, using the same multipart-frame mechanism already present in pyzmq. Until that lands, the recommended pattern for large GPU arrays in notebooks is explicit downsampling or streaming via `jupyter_rfb`.

**DAP debugger for GPU kernels.** Full step-debugging of CUDA kernels from JupyterLab — setting breakpoints inside `__global__` functions and inspecting thread-local variables — would require bridging CUDA-GDB's DAP adapter with ipykernel's `debugpy` control path. This remains an open research area as of mid-2026; `cuda-gdb --interpreter=mi2` integration with DAP is not yet standardised.

**Performance improvements in JupyterLab 4.x.** Large notebooks with many code outputs (logging, training loss curves printed every epoch) previously created thousands of DOM nodes that caused visible UI lag. JupyterLab 4's virtual output scroller uses an `IntersectionObserver`-based approach: only the outputs currently in the viewport are attached to the DOM; out-of-view outputs are replaced with placeholder divs of the correct height. This reduces DOM node count by 90%+ in heavy-output notebooks and makes training-loop notebooks practical to work with interactively.

---

## 14. Integrations

**Chapter 198 §18 — ZMQ Kernel Protocol.** This chapter describes Jupyter's architecture *above* the ZMQ wire protocol; Chapter 198 §18 covers the five ZMQ socket types, HMAC message authentication, and the full message schema in detail. The WebSocket multiplexing described in §2 of this chapter is the bridge between the ZMQ layer (Chapter 198 §18) and the browser-side `@jupyterlab/services` client.

**Chapter 34 — ANGLE and WebGL.** Every WebGL-based Jupyter visualisation described in §9 — ipyvolume (Three.js), plotly WebGL traces (`go.Scattergl`), and deck.gl layers — runs through ANGLE on Linux when the browser is Chromium or Chrome. ANGLE translates WebGL 2 (OpenGL ES 3.0) draw calls to Vulkan on Linux (the default backend since Chrome 113). Chapter 34 covers the ANGLE shader translation pipeline and EGL surface management that underlies all of these notebook visualisations.

**Chapter 98 — WebAssembly.** JupyterLite's Pyodide kernel is CPython compiled to WebAssembly via Emscripten, as described in §11. The COOP/COEP HTTP header requirements for `SharedArrayBuffer` access (which JupyterLite uses for kernel-to-worker messaging), the OPFS filesystem constraints, and the Wasm memory model are covered in Chapter 98 and apply directly to understanding JupyterLite's limitations and deployment requirements.

**Chapter 48 — ROCm and ML on Linux.** The RAPIDS suite (`cudf`, `cuml`, `cugraph`) and PyTorch ROCm operate inside the ipykernel subprocess via the KFD compute interface (`/dev/kfd`). Chapter 48 covers the ROCm software stack — HSA runtime, KFD driver, HIP compilation — that underpins GPU computing in ROCm-backed notebooks. The `docker run --device /dev/kfd` invocation in §12 of this chapter maps directly to the device access model described in Chapter 48.

**Chapter 55 — GPU Containers.** The DockerSpawner and KubeSpawner GPU passthrough described in §12 relies on `nvidia-container-toolkit` (CDI spec, runtime shim, device capability files) and the AMD ROCm container stack. Chapter 55 covers these container-level GPU access mechanisms — including `/dev/nvidia-caps/` capability files, `nvidia-ctk cdi generate`, and the Kubernetes GPU device plugin — in depth.

**Chapter 88 — NPU and AI Accelerators.** Jupyter kernels can target NPUs via OpenVINO or ONNX Runtime's NPU execution provider (Intel NPU EP for Meteor Lake and later). The device access model for Intel NPU (`/dev/accel/accel0`) and the OpenVINO `HETERO:NPU,GPU,CPU` heterogeneous dispatch described in Chapter 88 applies directly to notebook cells that call `ort.InferenceSession(..., providers=["OpenVINOExecutionProvider"])` with NPU device selection.

**Chapter 124 — LLM Inference on Linux.** The `%%ai` magic in `jupyter_ai` can route requests to a local `llama.cpp` / Ollama server. The GGUF memory-mapping approach described in Chapter 124 — `mmap(2)` with `MAP_SHARED`, `madvise` prefetch, and GPU weight upload via `vkCmdCopyBuffer` — applies when running LLMs from notebook cells. Notebook users invoking `ollama.chat()` or `openai.ChatCompletion.create(base_url="http://localhost:11434")` are traversing the inference stack described in Chapter 124 for each `%%ai` cell execution.
