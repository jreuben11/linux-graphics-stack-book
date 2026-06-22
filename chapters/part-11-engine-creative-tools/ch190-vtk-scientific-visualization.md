# Chapter 190: VTK — Scientific Visualization on the Linux Graphics Stack

**Target audiences:** Scientific computing developers using VTK for data visualization on Linux; HPC engineers deploying VTK in cluster and cloud environments; researchers using ParaView and 3D Slicer; graphics developers integrating VTK with Vulkan, WebGPU, or OpenGL.

---

## Table of Contents

1. [VTK Overview and Architecture](#1-vtk-overview-and-architecture)
2. [VTK Data Model](#2-vtk-data-model)
3. [VTK Rendering Backends on Linux](#3-vtk-rendering-backends-on-linux)
4. [GPU Volume Rendering](#4-gpu-volume-rendering)
5. [VTK WebGPU and ANARI Backends](#5-vtk-webgpu-and-anari-backends)
6. [VTK-m: GPU-Accelerated Filters](#6-vtk-m-gpu-accelerated-filters)
7. [VTK I/O — Scientific Data Formats](#7-vtk-io--scientific-data-formats)
8. [ParaView — The Flagship VTK Application](#8-paraview--the-flagship-vtk-application)
9. [VTK on Headless Linux Servers and Containers](#9-vtk-on-headless-linux-servers-and-containers)
10. [Integration with the Scientific Ecosystem](#10-integration-with-the-scientific-ecosystem)
11. [Integrations](#11-integrations)

---

## 1. VTK Overview and Architecture

The **Visualization Toolkit (VTK)** is an open-source, cross-platform software system for 3D computer graphics, image processing, and scientific visualization. Originally developed in 1993 by Will Schroeder, Ken Martin, and Bill Lorensen at GE Corporate Research and described in their textbook *The Visualization Toolkit* (Prentice Hall, 1996), VTK has grown into the foundational framework beneath most serious Linux scientific visualization applications. The Kitware company, founded by the VTK authors, continues to drive development. [Source](https://vtk.org/about/)

The primary repository is hosted at [gitlab.kitware.com/vtk/vtk](https://gitlab.kitware.com/vtk/vtk). As of mid-2026, the current stable series is **VTK 9.6**, released February 2026, followed by VTK 9.5.x (September 2025) and the earlier **VTK 9.4** (November 2024). VTK 9.4 introduced the ANARI rendering backend, runtime OpenGL loading via `glad`, WebGPU compute shaders, and `vtkImplicitArray`. VTK 9.5 brought further WebGPU and WASM improvements. VTK 9.6 added composite data texturing, a `vtkCartesianGrid` abstraction unifying `vtkImageData` and `vtkRectilinearGrid`, ONNX inference support, and JavaScript wrappers via Emscripten (`VTK_WRAP_JAVASCRIPT`). [Source](https://docs.vtk.org/en/latest/release_details/9.6.html)

### Pipeline Architecture

VTK is built around a **demand-driven pipeline** model. The three core abstractions are:

- **`vtkAlgorithm`** — the base class for all pipeline stages. Subclasses include sources (generate data with no input), filters (transform data), and sinks (consume data). The pipeline is lazy: `Update()` on a consumer triggers `RequestData()` back through the chain only when upstream data has changed.
- **`vtkDataObject`** — the base class for all data containers. The pipeline passes `vtkDataObject` subclasses between algorithm ports.
- **`vtkRenderWindow` + `vtkRenderer` + `vtkActor`** — the scene graph. `vtkRenderWindow` owns the OS window and GPU context; `vtkRenderer` holds lights, cameras, and actors; `vtkActor` wraps a `vtkMapper` (which connects to the pipeline) plus appearance properties.

The pipeline glue is `vtkPolyDataMapper`, which converts a `vtkPolyData` into GPU-ready vertex buffers and uploads them to the active render backend.

### Language Bindings

VTK exposes its API in several languages:

- **Python**: the `vtkmodules` package (installed via `pip install vtk`) wraps the entire C++ API using Shiboken-generated bindings. VTK 9.4 added Python-style constructor syntax and the `>>` pipeline connection operator. [Source](https://www.kitware.com/vtk-v9-4-0/)
- **Java**: generated Java wrappers ship with VTK.
- **vtk.js**: a JavaScript/TypeScript reimplementation of VTK's core algorithms targeting WebGL and WebGPU, actively maintained at [github.com/Kitware/vtk-js](https://github.com/Kitware/vtk-js). vtk.js v35 (March 2026) supports WebXR for VR/AR visualization. [Source](https://www.kitware.com/vtk-js-transforms-web-based-visualization-with-immersive-virtual-and-augmented-reality/)

### Hello World: A VTK Pipeline in Python

```python
from vtkmodules.vtkFiltersSources import vtkConeSource
from vtkmodules.vtkRenderingCore import (
    vtkActor, vtkPolyDataMapper, vtkRenderer, vtkRenderWindow,
    vtkRenderWindowInteractor,
)
import vtkmodules.vtkRenderingOpenGL2  # registers the OpenGL backend

# Source -> Mapper -> Actor -> Renderer -> Window
cone = vtkConeSource()
cone.SetHeight(3.0)
cone.SetRadius(1.0)
cone.SetResolution(60)

mapper = vtkPolyDataMapper()
mapper.SetInputConnection(cone.GetOutputPort())

actor = vtkActor()
actor.SetMapper(mapper)

renderer = vtkRenderer()
renderer.AddActor(actor)
renderer.SetBackground(0.1, 0.2, 0.4)

window = vtkRenderWindow()
window.AddRenderer(renderer)
window.SetSize(800, 600)

interactor = vtkRenderWindowInteractor()
interactor.SetRenderWindow(window)

window.Render()
interactor.Start()
```

VTK 9.4 also allows the pipeline connection operator `>>`:

```python
# Equivalent pipeline using >> operator (VTK 9.4+)
mapper = vtkPolyDataMapper()
cone >> mapper
actor = vtkActor(mapper=mapper)
```

### Build System

VTK uses CMake. Key top-level options:

```bash
cmake -S vtk -B build \
  -DVTK_USE_X=ON \               # X11 render window (Linux default)
  -DVTK_OPENGL_HAS_EGL=ON \      # EGL headless support
  -DVTK_ENABLE_WEBGPU=OFF \      # WebGPU backend (experimental)
  -DVTK_USE_CUDA=ON \            # CUDA device adapter for VTK-m
  -DVTK_USE_HIP=OFF \            # HIP/ROCm device adapter for VTK-m
  -DVTK_ENABLE_VTKM_OVERRIDES=ON \  # Let VTK-m accelerate core filters
  -DVTK_WRAP_PYTHON=ON \         # Build Python bindings
  -DCMAKE_BUILD_TYPE=Release
```

[Source: VTK Build Settings](https://docs.vtk.org/en/latest/build_instructions/build_settings.html)

---

## 2. VTK Data Model

VTK's data model is a class hierarchy rooted at `vtkDataObject`, with the concrete dataset classes forming the primary abstraction boundary between pipeline stages.

### Dataset Hierarchy

| Class | Description | Typical use |
|---|---|---|
| `vtkPolyData` | Surface meshes, point clouds, lines | Geometry rendering, surface output |
| `vtkUnstructuredGrid` | Mixed cell types, arbitrary topology | FEM results, CFD output |
| `vtkStructuredGrid` | Curvilinear hexahedral grid | Body-fitted CFD meshes |
| `vtkRectilinearGrid` | Axis-aligned, varying spacing | Climate grids, regular structured data |
| `vtkImageData` | Uniform voxel lattice | CT/MRI volumes, simulation grids |
| `vtkHyperTreeGrid` | AMR-like adaptive tree | Adaptive mesh refinement output |
| `vtkMultiBlockDataSet` | Composite container | Distributed/partitioned data in ParaView |

**`vtkPolyData`** is the workhorse for rendering. It holds:
- `vtkPoints` — a typed array of 3D point coordinates.
- `vtkCellArray` — compact connectivity (using the new `vtkCellArrayIterator` format in VTK 9.x that stores 64-bit offsets and connectivity in separate arrays for cache efficiency).
- Named cell arrays: `verts`, `lines`, `polys`, `strips`.

**`vtkUnstructuredGrid`** supports heterogeneous cell types by storing a per-cell type tag alongside the connectivity. VTK defines numeric cell type identifiers:

| Constant | Value | Shape |
|---|---|---|
| `VTK_VERTEX` | 1 | Single point |
| `VTK_LINE` | 3 | Line segment |
| `VTK_TRIANGLE` | 5 | Triangle |
| `VTK_QUAD` | 9 | Quadrilateral |
| `VTK_TETRA` | 10 | Tetrahedron |
| `VTK_HEXAHEDRON` | 12 | Hexahedron (brick) |
| `VTK_WEDGE` | 13 | Wedge (triangular prism) |
| `VTK_PYRAMID` | 14 | Pyramid |

[Source: VTK Cell Types](https://vtk.org/doc/nightly/html/vtkCellType_8h.html)

### Field Data

Point and cell data arrays are stored in `vtkPointData` and `vtkCellData` (both subclasses of `vtkFieldData`). The main typed array classes are `vtkFloatArray`, `vtkDoubleArray`, `vtkIntArray`, `vtkUnsignedCharArray`, and `vtkStringArray`. In VTK 9.3, the `vtkImplicitArray` framework was introduced, enabling zero-memory-overhead virtual arrays backed by a mapping function — useful for storing constant or affine fields without allocating full storage.

### vtkObjectManager and Web Serialisation

VTK 9.4 formalised `vtkObjectManager`, a mechanism for serialising and deserialising the VTK object graph to/from JSON. This infrastructure underpins `vtk-wasm` (VTK compiled to WebAssembly) and the trame framework's remote-rendering protocol: pipeline objects on the server are marshalled to JSON and their state synchronised to browser-side JavaScript objects. `vtkObjectManager` tracks object identity via 64-bit integer identifiers and supports incremental state updates, avoiding full re-serialisation on small property changes. [Source](https://docs.vtk.org/en/latest/release_details/9.4.html)

### Higher-Order Elements: vtkCellGrid

VTK 9.3 introduced `vtkCellGrid`, a parallel dataset class designed for **discontinuous Galerkin (DG)** finite element methods where each element can carry its own polynomial basis and the solution is not required to be continuous across cell boundaries. Traditional `vtkUnstructuredGrid` stores only per-vertex or per-cell scalar data; `vtkCellGrid` stores per-DOF (degree of freedom) data. VTK 9.4 extended this with tessellation shader support for GPU-side evaluation of high-order geometry. [Source](https://docs.vtk.org/en/latest/release_details/9.4.html)

### Building vtkPolyData with Scalar Data (Python)

```python
import numpy as np
from vtkmodules.vtkCommonCore import vtkPoints
from vtkmodules.vtkCommonDataModel import vtkPolyData, vtkCellArray
from vtkmodules.util.numpy_support import numpy_to_vtk

# Three vertices forming a triangle
pts_np = np.array([[0, 0, 0], [1, 0, 0], [0.5, 1, 0]], dtype=np.float32)
vtk_pts = vtkPoints()
vtk_pts.SetData(numpy_to_vtk(pts_np))

# One triangle cell using VTK 9.x CellArray API:
# SetData(offsets_array, connectivity_array)
# offsets: [start_of_cell_0, start_of_cell_1] → [0, 3] for a single tri with 3 verts
cell_arr = vtkCellArray()
cell_arr.SetData(
    numpy_to_vtk(np.array([0, 3], dtype=np.int64)),   # offsets (one per cell + sentinel)
    numpy_to_vtk(np.array([0, 1, 2], dtype=np.int64)) # connectivity (vertex indices)
)

# Scalar field (one value per point)
scalars_np = np.array([0.0, 0.5, 1.0], dtype=np.float32)
scalars = numpy_to_vtk(scalars_np)
scalars.SetName("Temperature")

polydata = vtkPolyData()
polydata.SetPoints(vtk_pts)
polydata.SetPolys(cell_arr)
polydata.GetPointData().SetScalars(scalars)
```

The `numpy_to_vtk` function performs a **zero-copy** conversion when the NumPy array is C-contiguous and the dtype matches a VTK native type, storing only a pointer to the NumPy buffer. [Source](https://docs.vtk.org/en/latest/api/python/vtkmodules/vtkmodules.util.numpy_support.html)

---

## 3. VTK Rendering Backends on Linux

VTK supports multiple rendering backends and window systems on Linux. The active backend is determined at CMake configuration time, though VTK 9.4 now performs runtime selection for some paths.

### OpenGL Backend (Default)

The OpenGL 2 rendering backend (`Rendering/OpenGL2/`) is the production rendering path. On Linux it supports three window system integrations:

**`vtkXOpenGLRenderWindow`** — creates an X11 window via `XCreateWindow`, obtains a GLX context with `glXCreateContextAttribsARB`, and renders to the window using GLX swapbuffers. This is the default on a desktop Linux system with `VTK_USE_X=ON`. The EGL fallback was removed in VTK 9.x in favour of explicit CMake selection. [Source](https://gitlab.kitware.com/vtk/vtk/-/blob/master/Rendering/OpenGL2/vtkXOpenGLRenderWindow.cxx)

**`vtkEGLRenderWindow`** — creates an EGL context via `eglCreateContext` using `EGL_NO_SURFACE` with an offscreen FBO for rendering. This backend enables **GPU-accelerated headless rendering** without an X display: on a Kubernetes node with an NVIDIA or AMD GPU, the EGL driver is accessed through `/dev/nvidia0` or `/dev/dri/renderD128`. Enabled with `VTK_OPENGL_HAS_EGL=ON`. [Source](https://gitlab.kitware.com/vtk/vtk/-/blob/master/Rendering/OpenGL2/vtkEGLRenderWindow.cxx)

**`vtkOSOpenGLRenderWindow`** — offscreen Mesa software renderer via `libOSMesa`. Links against `libOSMesa.so`; no GPU or display required. Enabled with `VTK_OPENGL_HAS_OSMESA=ON`. Useful for CI pipelines and CPU-only containers.

Starting with VTK 9.4, OpenGL symbol loading switched to `glad` at runtime rather than link-time symbols, allowing a single VTK build to function with different OpenGL implementations (Mesa, NVIDIA, AMD) without recompilation. This change carries forward in VTK 9.5 and 9.6. [Source](https://docs.vtk.org/en/latest/release_details/9.4.html)

### Wayland Support

As of VTK 9.4, Wayland support is provided through the EGL path: `VTK_USE_WAYLAND_OPENGL=ON` (requires `VTK_OPENGL_HAS_EGL=ON`) switches the EGL render window's native display from `EGL_DEFAULT_DISPLAY` to the Wayland `wl_display*`, using `EGL_PLATFORM_WAYLAND_EXT`. This is the supported Wayland onscreen path; native `wl_surface` event handling (keyboard, pointer) is provided by `vtkWaylandOpenGLRenderWindow`. [Source](https://docs.vtk.org/en/latest/build_instructions/build_settings.html)

### Vulkan Backend (Not in Mainline)

There is **no `Rendering/Vulkan/` directory in the mainline VTK repository**. Inspection of the `master` branch at `github.com/Kitware/VTK/tree/master/Rendering` confirms the Rendering directory contains: `ANARI`, `OpenGL2`, `VolumeOpenGL2`, `WebGPU`, `VR`, `OpenXR`, and many other backends — but no `Vulkan/` subdirectory. [Source](https://github.com/Kitware/VTK/tree/master/Rendering)

A proof-of-concept Vulkan branch was started by Ken Martin in 2020 at `gitlab.kitware.com/ken-martin/vtk/-/tree/vulkan`, implementing `vtkVulkanRenderWindow` and `vtkVulkanWindowNode`, but as of 2026 it remains a fork experiment and has not been merged. The VTK community has instead invested in the **WebGPU** backend (Section 5), which on desktop Linux runs via Dawn → Vulkan, giving VTK Vulkan-backed rendering without a bespoke VTK Vulkan renderer. [Source](https://discourse.vtk.org/t/vulkan-development/3307)

### CMake Configuration for Each Backend

```bash
# X11 / GLX (standard desktop Linux)
cmake -S vtk -B build-x11 \
  -DVTK_USE_X=ON \
  -DVTK_OPENGL_HAS_EGL=OFF

# EGL headless (GPU without X display)
cmake -S vtk -B build-egl \
  -DVTK_USE_X=OFF \
  -DVTK_OPENGL_HAS_EGL=ON \
  -DVTK_DEFAULT_RENDER_WINDOW_HEADLESS=ON

# EGL + X11 (both available; select at runtime)
cmake -S vtk -B build-both \
  -DVTK_USE_X=ON \
  -DVTK_OPENGL_HAS_EGL=ON

# OSMesa software (no GPU, no display)
cmake -S vtk -B build-osmesa \
  -DVTK_USE_X=OFF \
  -DVTK_OPENGL_HAS_OSMESA=ON \
  -DVTK_DEFAULT_RENDER_WINDOW_OFFSCREEN=ON

# Wayland via EGL
cmake -S vtk -B build-wayland \
  -DVTK_USE_X=OFF \
  -DVTK_OPENGL_HAS_EGL=ON \
  -DVTK_USE_WAYLAND_OPENGL=ON
```

[Source](https://docs.vtk.org/en/latest/build_instructions/build_settings.html)

### Runtime Environment Variables

```bash
# Force headless EGL at runtime (overrides window class)
export VTK_DEFAULT_OPENGL_WINDOW=vtkEGLRenderWindow

# Force offscreen render window
export VTK_DEFAULT_RENDER_WINDOW_OFFSCREEN=1

# Select EGL device (GPU index for multi-GPU headless nodes)
export VTK_DEFAULT_EGL_DEVICE_INDEX=1

# Mesa compatibility override (force GL 3.3 when driver underreports)
export MESA_GL_VERSION_OVERRIDE=3.3
```

---

## 4. GPU Volume Rendering

Volume rendering of 3D scalar fields — CT scans, MRI data, simulation grids — is one of VTK's flagship capabilities. The GPU path uses OpenGL 3D textures and a custom GLSL ray-casting shader.

### vtkGPUVolumeRayCastMapper

`vtkGPUVolumeRayCastMapper` (implemented in `Rendering/VolumeOpenGL2/`) performs GPU ray casting. The algorithm:

1. **Entry/exit point computation**: the volume bounding box is rendered twice — front faces to a floating-point FBO storing entry positions, back faces to another FBO storing exit positions. The difference gives the ray direction and length in texture space.
2. **Ray marching**: a fragment shader loops along each ray in steps of configurable size, sampling the 3D texture `GL_TEXTURE_3D` at each step position.
3. **Transfer function lookup**: scalar values are mapped to RGBA via two 1D textures: `GL_TEXTURE_1D` for color (from `vtkColorTransferFunction`) and another for opacity (from `vtkPiecewiseFunction`).
4. **Gradient shading**: optional Phong shading computes the gradient via central differences in the shader, enabling surface-like appearance for dense regions.
5. **Compositing**: samples are blended front-to-back using standard Porter-Duff `over` compositing.

[Source](https://vtk.org/doc/nightly/html/classvtkGPUVolumeRayCastMapper.html)

The GLSL shader is dynamically generated by `vtkVolumeShaderComposer` — rather than maintaining a combinatorial set of shader variants, the composer constructs the shader string at runtime based on active features (shading, gradient opacity, multi-volume, clipping planes). [Source](https://vtk.org/doc/nightly/html/classvtkOpenGLGPUVolumeRayCastMapper.html)

### vtkSmartVolumeMapper

`vtkSmartVolumeMapper` auto-selects the best available mapper at runtime:

1. `vtkGPUVolumeRayCastMapper` (GPU OpenGL) — selected when `RequestedRenderMode == DefaultRenderMode` and GPU supports it.
2. `vtkFixedPointVolumeRayCastMapper` (CPU fixed-point ray cast) — fallback when GPU memory is insufficient.
3. `vtkOSPRayVolumeMapper` — if OSPRay (Intel's CPU/GPU ray tracer) is linked in.

### Transfer Functions and vtkVolumeProperty

```python
from vtkmodules.vtkIOImage import vtkMetaImageReader
from vtkmodules.vtkCommonDataModel import vtkPiecewiseFunction
from vtkmodules.vtkRenderingCore import (
    vtkColorTransferFunction,
    vtkVolume,
    vtkVolumeProperty,
    vtkRenderer,
    vtkRenderWindow,
    vtkRenderWindowInteractor,
)
from vtkmodules.vtkRenderingVolumeOpenGL2 import vtkSmartVolumeMapper
import vtkmodules.vtkRenderingOpenGL2

# Load a MetaImage (.mhd/.raw) CT dataset
reader = vtkMetaImageReader()
reader.SetFileName("ct_scan.mhd")
reader.Update()

# Color transfer function: bone windowing
ctf = vtkColorTransferFunction()
ctf.AddRGBPoint(-1000, 0.0, 0.0, 0.0)   # air → black
ctf.AddRGBPoint(  400, 1.0, 0.9, 0.8)   # soft tissue → pink
ctf.AddRGBPoint( 1000, 1.0, 1.0, 0.9)   # bone → white

# Opacity transfer function
otf = vtkPiecewiseFunction()
otf.AddPoint(-1000, 0.0)   # air fully transparent
otf.AddPoint(  200, 0.0)   # soft tissue: transparent
otf.AddPoint(  400, 0.15)  # partial opacity at tissue/bone boundary
otf.AddPoint( 1000, 0.85)  # bone mostly opaque

# Gradient opacity (suppress flat interiors)
gof = vtkPiecewiseFunction()
gof.AddPoint(0,   0.0)
gof.AddPoint(90,  0.5)
gof.AddPoint(300, 1.0)

volume_property = vtkVolumeProperty()
volume_property.SetColor(ctf)
volume_property.SetScalarOpacity(otf)
volume_property.SetGradientOpacity(gof)
volume_property.ShadeOn()
volume_property.SetInterpolationTypeToLinear()

mapper = vtkSmartVolumeMapper()
mapper.SetInputConnection(reader.GetOutputPort())

volume = vtkVolume()
volume.SetMapper(mapper)
volume.SetProperty(volume_property)

renderer = vtkRenderer()
renderer.AddVolume(volume)
renderer.SetBackground(0, 0, 0)

window = vtkRenderWindow()
window.AddRenderer(renderer)
window.SetSize(1024, 768)

interactor = vtkRenderWindowInteractor()
interactor.SetRenderWindow(window)
window.Render()
interactor.Start()
```

### Large-Volume Bricking

When a volume exceeds GPU texture memory (e.g., a 2048³ simulation grid at float32 = 32 GB), `vtkGPUVolumeRayCastMapper` uses **bricking**: the volume is partitioned into tiles that fit in GPU memory, each tile is streamed to the GPU in sequence, and the results are composited. The brick size is controlled by `SetMaxMemoryInBytes()`. Note: needs verification on exact bricking implementation details in VTK 9.4.

### Multi-Volume Rendering

VTK supports simultaneous rendering of multiple overlapping volumes via `vtkMultiVolume`. Each input volume has its own `vtkVolume` instance (with independent transfer functions and transformations), all connected to a single `vtkGPUVolumeRayCastMapper`. The mapper handles correct alpha-compositing across overlapping regions by sorting ray segments and blending samples in depth order. This is useful for co-registration of CT and PET scans in medical imaging, or overlaying different simulation scalar fields (temperature + density) in the same view. [Source](https://vtk.org/doc/nightly/html/classvtkGPUVolumeRayCastMapper.html)

### 2D Charts: vtkChartXY

Beyond 3D rendering, VTK provides a 2D charting subsystem via `VTK::ChartsCore`. `vtkChartXY` renders line plots, scatter plots, bar charts, and stacked plots into a `vtkContextScene` using a 2D vector graphics API (`vtkContext2D`) backed by OpenGL. Charts integrate with the same pipeline and data model: a `vtkTable` with column arrays drives a `vtkPlotLine` or `vtkPlotPoints` instance. The charting subsystem is used by ParaView's plot views and by 3D Slicer's Python console for exploratory data analysis within the same application window.

```python
from vtkmodules.vtkChartsCore import vtkChart, vtkChartXY
from vtkmodules.vtkCommonCore import vtkFloatArray
from vtkmodules.vtkCommonDataModel import vtkTable
from vtkmodules.vtkViewsContext2D import vtkContextView
import vtkmodules.vtkRenderingContext2D

# Build a simple table with two columns
x_arr = vtkFloatArray()
x_arr.SetName("X")
y_arr = vtkFloatArray()
y_arr.SetName("Y")
for i in range(100):
    x_arr.InsertNextValue(i * 0.1)
    y_arr.InsertNextValue((i * 0.1) ** 2)

table = vtkTable()
table.AddColumn(x_arr)
table.AddColumn(y_arr)

view = vtkContextView()
chart = vtkChartXY()
view.GetScene().AddItem(chart)

line = chart.AddPlot(vtkChart.LINE)
line.SetInputData(table, 0, 1)    # X column=0, Y column=1
line.SetColor(255, 0, 0, 255)

view.GetRenderWindow().SetSize(600, 400)
view.GetRenderWindow().Render()
view.GetInteractor().Start()
```

---

## 5. VTK WebGPU and ANARI Backends

### WebGPU Backend (vtkRenderingWebGPU)

VTK 9.3 introduced the `VTK::RenderingWebGPU` module as an **experimental** alternative rendering backend. On Linux desktop, it uses **Dawn** (Google's C++ WebGPU implementation, [dawn.googlesource.com/dawn](https://dawn.googlesource.com/dawn)) as the underlying WebGPU implementation, which in turn uses Vulkan as its GPU API. On WebAssembly, it uses the browser's native WebGPU. This avoids a redundant abstraction layer above WebGPU — the VTK WebGPU backend talks directly to the `wgpu::Device` API. [Source](https://docs.vtk.org/en/latest/modules/vtk-modules/Rendering/WebGPU/README.html)

Current capabilities (VTK 9.4–9.6):
- Polygonal geometry rendering (points, lines, triangles) with scalar-mapped coloring.
- Compute shaders for GPU-parallel workloads — notably frustum culling and point cloud rendering. A demonstrated use case renders interactive point clouds of two billion points. [Source](https://www.kitware.com/vtk-v9-4-0/)
- Hardware depth testing and selection.
- Surface-with-edges and wireframe representations.
- Texture mapping for 3D models (added in VTK 9.6). [Source](https://docs.vtk.org/en/latest/release_details/9.6.html)

Volume rendering and advanced lighting are not yet ported to the WebGPU backend as of VTK 9.6. Enable with:

```bash
cmake -S vtk -B build-webgpu \
  -DVTK_ENABLE_WEBGPU=ON \
  -DVTK_USE_X=OFF \
  -DVTK_OPENGL_HAS_EGL=ON   # WebGPU/Dawn may use EGL surface on Linux
```

### ANARI Backend (vtkRenderingANARI)

VTK 9.4 introduced `vtkRenderingANARI`, and the integration has been refined through VTK 9.5 and 9.6. It integrates the [ANARI 1.0 standard](https://www.khronos.org/anari/) (Analytic Rendering Interface) published by Khronos. ANARI provides a portable C API for delegating rendering to advanced backends: path tracers (NVIDIA VisRTX, Intel OSPRay), rasterizers, or custom engines. [Source](https://www.khronos.org/blog/kitware-adds-anari-support-to-vtk-to-simplify-access-to-accelerated-3d-rendering-engines)

Usage:

```bash
# Set the ANARI library implementation (e.g., OSPRay)
export ANARI_LIBRARY=ospray

# Or NVIDIA VisRTX for GPU path tracing
export ANARI_LIBRARY=visrtx
```

```python
import vtkmodules.vtkRenderingANARI  # register ANARI backend
from vtkmodules.vtkRenderingANARI import vtkAnariPass

anari_pass = vtkAnariPass()
renderer.SetPass(anari_pass)
# Renderer now delegates to the ANARI library for path-traced rendering
```

ANARI-compatible backends available on Linux:
- **Intel OSPRay** — CPU/GPU ray tracer, open source, excellent for large-scale HPC visualization.
- **NVIDIA VisRTX** — GPU hardware ray tracing via OptiX/NVIDIA RTX.
- **AMD RadeonProRender** — GPU path tracer for AMD hardware.

The VTK ANARI integration was overhauled at the ANARI Virtual Hackathon 2024. [Source](https://www.khronos.org/blog/khronos-releases-new-anari-sdk-updates-hackathon-results)

---

## 6. VTK-m: GPU-Accelerated Filters

[VTK-m](https://m.vtk.org/) is a companion toolkit providing highly parallel implementations of visualization algorithms for multi-core and many-core architectures. The primary repository is at [gitlab.kitware.com/vtk/vtk-m](https://gitlab.kitware.com/vtk/vtk-m). VTK 9.3 upgraded to VTK-m 2.0, bringing reorganised CMake targets (all now `vtkm_`-prefixed) and improved data model alignment. [Source](https://docs.vtk.org/en/latest/release_details/9.3.html)

### Architecture

VTK-m's design separates the **what** (algorithm logic as worklets) from the **where** (device adapters):

**ArrayHandle** is the primary data container — a typed array that can reside on CPU or GPU memory. Transfers between host and device are managed automatically via the control/execution environment split. Array handles can be constructed from existing memory (including VTK arrays via `ArrayHandleVTKDataArray`) without copying.

**Worklets** express algorithms as per-element functions with explicitly declared inputs and outputs. Three worklet types cover the main patterns:
- `WorkletMapField` — one output element per input element (e.g., threshold, normalise).
- `WorkletMapTopology` — maps over mesh cells or points with neighbourhood access (e.g., gradient, interpolation).
- `WorkletReduceByKey` — grouped reduction (e.g., average, sum per cell type).

**Device adapters** implement the execution model for each hardware target:

| Adapter | Macro | Hardware |
|---|---|---|
| Serial | `VTKM_DEVICE_ADAPTER_SERIAL` | Single CPU thread (debugging) |
| OpenMP | `VTKM_DEVICE_ADAPTER_OPENMP` | Multi-core CPU via OpenMP |
| TBB | `VTKM_DEVICE_ADAPTER_TBB` | Multi-core CPU via Intel TBB |
| CUDA | `VTKM_DEVICE_ADAPTER_CUDA` | NVIDIA GPUs, compiled with `nvcc` |
| HIP | `VTKM_DEVICE_ADAPTER_HIP` | AMD GPUs via ROCm/HIP, with `hipcc` |
| Kokkos | `VTKM_DEVICE_ADAPTER_KOKKOS` | Portable: CUDA, HIP, SYCL, OpenMP via Kokkos |

[Source](https://m.vtk.org/index.php/Building_VTK-m)

The Kokkos adapter (added in VTK-m 1.7) is the recommended path for AMD ROCm 6+ (`VTKM_ENABLE_KOKKOS=ON`, with `CMAKE_CXX_COMPILER=hipcc`). [Source](https://github.com/Kitware/VTK-m)

### Key Filters

- **`vtkm::filter::contour::Contour`** — Marching Cubes isosurface extraction, fully parallel on GPU. On a 512³ volume, GPU execution (CUDA/HIP) typically completes in ~50 ms versus ~5 seconds on CPU. [Note: benchmark figures from VTK-m Users' Guide V.2.0; specific hardware and configuration may vary. Source: [OSTI](https://www.osti.gov/biblio/1959590)]
- **`vtkm::filter::vector_analysis::Gradient`** — point-centred or cell-centred gradient computation.
- **`vtkm::filter::entity_extraction::Threshold`** — retain cells whose scalar field satisfies a predicate.
- **`vtkm::filter::flow::Streamline`** — particle tracing using Euler or Runge-Kutta 4 integration on unstructured or structured grids.
- **`vtkm::filter::geometry_refinement::Triangulate`** — convert polygonal cells to triangles for downstream rendering.

### Integration with VTK 9.x

When `VTK_ENABLE_VTKM_OVERRIDES=ON`, VTK intercepts calls to `vtkContourFilter`, `vtkThreshold`, and `vtkGradientFilter` and delegates them to VTK-m implementations. The conversion between `vtkDataSet` and `vtkm::cont::DataSet` uses `ArrayHandleVTKDataArray` — a zero-copy adapter that wraps a `vtkDataArray`'s memory buffer as a VTK-m array handle, avoiding an extra copy for the filter invocation.

### C++ Example: Marching Cubes on GPU

```cpp
#include <vtkm/cont/DataSet.h>
#include <vtkm/cont/Initialize.h>
#include <vtkm/filter/contour/Contour.h>
#include <vtkm/io/VTKDataSetReader.h>
#include <vtkm/io/VTKDataSetWriter.h>

int main(int argc, char* argv[])
{
    // Select CUDA device adapter (GPU) at startup
    vtkm::cont::Initialize(argc, argv, vtkm::cont::InitializeOptions::DefaultAnyDevice);

    // Read an unstructured volume dataset
    vtkm::io::VTKDataSetReader reader("volume.vtk");
    vtkm::cont::DataSet dataset = reader.ReadDataSet();

    // Run Marching Cubes at isovalue 0.5
    vtkm::filter::contour::Contour contour;
    contour.SetActiveField("Scalars");      // field to contour
    contour.SetIsoValue(0, 0.5);            // one isovalue
    contour.SetMergeDuplicatePoints(true);  // weld shared vertices

    vtkm::cont::DataSet surface = contour.Execute(dataset);

    // Write result
    vtkm::io::VTKDataSetWriter writer("isosurface.vtk");
    writer.WriteDataSet(surface);

    return 0;
}
```

Link against `vtkm::filter_contour` and `vtkm::io` (CMake targets post-2.0 use `vtkm_` prefix). The device adapter is selected at `Initialize()` based on the runtime argument `--vtkm-device` or falls back to the best available device. [Source](https://docs-m.vtk.org/latest/)

### VTK-m CMake Build

```bash
cmake -S vtk-m -B build-vtkm \
  -DVTKm_ENABLE_CUDA=ON \
  -DVTKm_CUDA_Architecture=ampere \   # sm_86 for RTX 3000/A-series
  -DVTKm_ENABLE_OPENMP=ON \
  -DVTKm_ENABLE_TBB=OFF \
  -DVTKm_ENABLE_KOKKOS=OFF \
  -DVTKm_ENABLE_TESTING=OFF \
  -DCMAKE_INSTALL_PREFIX=/usr/local
make -j$(nproc) && make install
```

For AMD/ROCm via Kokkos:

```bash
cmake -S vtk-m -B build-vtkm-hip \
  -DVTKm_ENABLE_KOKKOS=ON \
  -DKokkos_ENABLE_HIP=ON \
  -DCMAKE_CXX_COMPILER=hipcc \
  -DVTKm_ENABLE_TESTING=OFF
```

[Source](https://m.vtk.org/index.php/Building_VTK-m)

### Streamline Tracing with VTK-m

Particle tracing through a vector field (e.g., velocity in a CFD simulation) is a canonical visualization task. VTK-m's `vtkm::filter::flow::Streamline` filter executes Runge-Kutta 4 integration entirely on the GPU, seeding many particles simultaneously:

```cpp
#include <vtkm/filter/flow/Streamline.h>
#include <vtkm/cont/ArrayHandleBasic.h>

// Seed points (particle starting positions)
std::vector<vtkm::Vec3f> seeds = {
    {0.0f, 0.0f, 0.0f}, {0.5f, 0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}
};
auto seedArray = vtkm::cont::make_ArrayHandle(seeds, vtkm::CopyFlag::On);

vtkm::filter::flow::Streamline streamline;
streamline.SetActiveField("Velocity");      // vector field name
streamline.SetStepSize(0.01f);              // integration step
streamline.SetNumberOfSteps(1000);          // max steps per particle
streamline.SetSeeds(seedArray);

vtkm::cont::DataSet result = streamline.Execute(dataset);
// result contains vtkm::cont::CellSetExplicit with LINE cells (one per seed)
```

---

## 7. VTK I/O — Scientific Data Formats

VTK supports a broad range of scientific data formats through its `VTK::IO*` modules.

### Native VTK Formats

**Legacy VTK format** (`.vtk`): ASCII or binary format introduced with the original 1993 toolkit. Human-readable ASCII variant useful for debugging; binary variant for performance. Reader: `vtkDataSetReader` (auto-detects type). All dataset types are supported.

**VTK XML formats** (VTK 4.2+): one format per dataset type, using XML markup for metadata and optionally inline or appended base64/binary data. Extension → reader mapping:

| Extension | Dataset | Reader class |
|---|---|---|
| `.vtp` | PolyData | `vtkXMLPolyDataReader` |
| `.vtu` | UnstructuredGrid | `vtkXMLUnstructuredGridReader` |
| `.vti` | ImageData | `vtkXMLImageDataReader` |
| `.vts` | StructuredGrid | `vtkXMLStructuredGridReader` |
| `.vtr` | RectilinearGrid | `vtkXMLRectilinearGridReader` |
| `.pvtp`, `.pvtu`, ... | Parallel partitioned | `vtkXMLPUnstructuredGridReader`, etc. |

Parallel XML formats (`.pvtu`, `.pvtp`) store a master XML index file that references per-rank piece files — the standard output format for parallel ParaView pipeline runs.

### VTK HDF Format (VTKHDF)

Introduced in VTK 9.1, the VTKHDF format uses HDF5 as the storage layer but with a VTK-defined group/dataset layout that supports parallel I/O via MPI-IO and time-series data via internal caching. Reader: `vtkHDFReader`. Supported dataset types: PolyData, UnstructuredGrid, ImageData, HyperTreeGrid, OverlappingAMR, MultiBlockDataSet, PartitionedDataSetCollection. [Source](https://www.kitware.com/vtk-hdf-reader/)

The format is actively developed as the preferred successor to the legacy `.vtk` and parallel XML formats for large-scale simulations. A 2025 status update reports significant I/O performance improvements for temporal datasets. [Source](https://www.kitware.com/vtkhdf-file-format-2025-status-update/)

### ADIOS2

The `VTK::IOADIOS2` module provides `vtkADIOS2VTXReader`, which reads BP4/BP5 files produced by ADIOS2 (the DOE Exascale Computing Project I/O library). [Source](https://docs.vtk.org/en/latest/modules/vtk-modules/IO/ADIOS2/README.html) ADIOS2 is widely used in DOE simulation codes (WarpX, GENE, LAMMPS) for both file-based I/O and in-situ streaming via SST (Sustainable Staging Transport). When combined with ParaView Catalyst (Section 8), a simulation can stream data over ADIOS2 SST to a live ParaView session. [Source](https://www.kitware.com/in-situ-in-transit-hybrid-analysis-using-catalyst-adios2-and-paraview/)

### DICOM (Medical Imaging)

`vtkDICOMImageReader` provides basic DICOM Series reading (single-frame, CT-compatible). For full DICOM compliance (multi-frame, DICOM-RT, SEG, SR objects), the `vtk-dicom` extension by David Gobbi provides a complete implementation: [github.com/dgobbi/vtk-dicom](https://github.com/dgobbi/vtk-dicom).

Converting a DICOM series to a VTK MetaImage file:

```bash
# Using the DicomToMetaImage utility from vtk-dicom
DicomToMetaImage --image ct_output.mha /path/to/dicom/series/

# Or using VTK's built-in reader in a Python one-liner
python3 -c "
import vtkmodules.all as vtk
reader = vtk.vtkDICOMImageReader()
reader.SetDirectoryName('/path/to/dicom/series/')
reader.Update()
writer = vtk.vtkMetaImageWriter()
writer.SetFileName('output.mhd')
writer.SetInputConnection(reader.GetOutputPort())
writer.Write()
"
```

### Exodus II (FEM)

`vtkExodusIIReader` reads Exodus II format files produced by Sandia National Laboratories finite element codes (Sierra, Salinas, Alegra) and SEACAS/Trilinos. Exodus II files contain time-series FEM results including displacement fields, stress tensors, and nodeset/sideset definitions. [Note: needs verification on exact list of SEACAS-based codes.]

### NetCDF and Climate Data

`vtkNetCDFReader` reads CF-convention NetCDF files from climate and ocean models (CESM, WRF, NEMO). Coordinate variables are automatically mapped to VTK structured grid coordinates. `vtkNetCDFCFReader` handles the Climate and Forecast (CF) metadata conventions for correct geographic projection.

### EnSight Gold

`vtkEnSightGoldBinaryReader` reads the EnSight Gold binary case format, the native output of several commercial CFD solvers (ANSYS Fluent, STAR-CCM+, OpenFOAM post-processed output).

---

## 8. ParaView — The Flagship VTK Application

[ParaView](https://www.paraview.org/) is the primary application built on VTK, developed by Kitware for the US Department of Energy's HPC visualization needs. It extends VTK with a parallel client-server architecture, a Python scripting layer, and the Catalyst in-situ framework. Repository: [gitlab.kitware.com/paraview/paraview](https://gitlab.kitware.com/paraview/paraview).

### Client-Server Architecture

ParaView separates computation from display:

- **`pvserver`** — MPI-parallel rendering server. Runs on the HPC cluster, holds the data, executes filters, and renders to offscreen FBOs. Can use GPU (EGL) or CPU (OSMesa) rendering.
- **`paraview` (GUI)** — Qt-based client runs on the researcher's workstation. Communicates with `pvserver` over TCP/IP using ParaView's custom protocol.
- **Remote rendering**: the server renders a full-resolution image, compresses it (JPEG or LZ4), and sends the encoded image to the client GUI. The client GPU is not involved in rendering; it only decompresses and displays the image. This allows a researcher on a laptop to explore a 100 TB simulation dataset running on thousands of cluster cores.
- **Threshold switching**: for small datasets, the server sends geometry to the client, which renders locally for interactive frame rates. For large datasets beyond a geometry threshold, server-side rendering is used.

### IceT — Sort-Last Compositing

For parallel rendering across many MPI ranks, ParaView uses **IceT** (Image Compositing Engine for Tiles), developed at Sandia National Laboratories: [gitlab.kitware.com/icet/icet](https://gitlab.kitware.com/icet/icet).

In **sort-last** compositing, each MPI rank renders its partition of the data to a local image with depth. IceT then performs a **binary-swap** depth compositing algorithm: ranks exchange and merge image fragments, halving the number of active participants at each round. The final composited image (with correct depth ordering) is available on rank 0 for transmission to the client. IceT has been demonstrated at 64,000 cores on IBM BlueGene systems. [Source](https://www.kennethmoreland.com/scalable-rendering/)

For display wall tiling, IceT assigns each tile of the wall display to a subset of render nodes, compositing only the required region on each node.

### pvpython and pvbatch

- **`pvpython`**: headless Python interpreter with ParaView's full Python API available, no GUI. Used for server-side scripting and automated analysis.
- **`pvbatch`**: MPI-parallel version of pvpython, equivalent to running pvpython under `mpiexec`. Used for batch rendering on HPC clusters.

Both share the same `paraview.simple` module API. A batch rendering script submitted to Slurm:

```bash
#!/bin/bash
#SBATCH --nodes=8
#SBATCH --ntasks-per-node=4
#SBATCH --gres=gpu:4

module load paraview/5.13.2-egl
mpiexec -n 32 pvbatch --use-offscreen-rendering render_iso.py \
  --input /scratch/simulation/output.000500.pvtu \
  --output /scratch/images/frame_500.png
```

The `--use-offscreen-rendering` flag ensures pvbatch uses EGL (if built with EGL support) rather than attempting to open a display.

```python
# pvpython script: render an isosurface from an unstructured grid
# Run as: pvpython render_iso.py (or pvbatch -n 64 render_iso.py)
from paraview.simple import *

# Load data
reader = XMLUnstructuredGridReader(FileName=["simulation.vtu"])
reader.PointArrayStatus = ["Pressure"]

# Extract isosurface at Pressure = 101325 Pa
contour = Contour(Input=reader)
contour.ContourBy = ["POINTS", "Pressure"]
contour.Isosurfaces = [101325.0]
contour.ComputeNormals = True

# Colour by velocity magnitude
ColorBy(contour, ("POINTS", "Velocity"), separate=False)

# Setup rendering
renderView = GetActiveViewOrCreate("RenderView")
display = Show(contour, renderView)
display.Representation = "Surface"

# Camera
renderView.ResetCamera()
renderView.CameraPosition = [0, 0, 10]

# Save image
SaveScreenshot("isosurface.png", renderView, ImageResolution=[2048, 1536])
```

### Catalyst 2.0 — In-Situ Visualization

**Catalyst 2.0** is a lightweight C API that simulation codes link against to perform in-situ visualization — processing data as it is generated, without writing full datasets to disk. [Source](https://catalyst-in-situ.readthedocs.io/en/latest/introduction.html)

Architecture:
1. The simulation code links `libcatalyst.so` (a small stub library — the API layer).
2. At runtime, a **Catalyst implementation** (typically `catalyst-paraview.so`) is loaded via `dlopen` and registered.
3. Per-timestep, the simulation calls `catalyst_execute(conduit_node_t*)`, passing mesh and field data described using the **Conduit Mesh Blueprint** — a JSON-schema-described hierarchical format for computational meshes.
4. The ParaView Catalyst implementation deserialises the Conduit mesh into VTK data structures, applies a ParaView pipeline (defined as a Python script), and produces images or extracted data.

```c
/* simulation code integration — condensed example */
#include <catalyst.h>
#include <conduit.h>

conduit_node* params = conduit_node_create();
conduit_node_set_path_char8_str(params, "catalyst/scripts/script0/filename",
                                 "pipeline.py");
catalyst_initialize(params);

/* per-timestep: build mesh blueprint node and execute */
conduit_node* data = conduit_node_create();
/* ... populate "coordsets", "topologies", "fields" per Blueprint spec ... */
catalyst_execute(data);

catalyst_finalize(NULL);
```

[Source: ParaView Catalyst Blueprint](https://docs.paraview.org/en/latest/Catalyst/blueprints.html)

Catalyst 2.0 separates the API from the implementation, allowing simulations to link a tiny stub library and swap visualization backends at deployment time without recompilation — switching between ParaView, ADIOS2 in-transit, or a custom implementation. [Source](https://warpx.readthedocs.io/en/latest/dataanalysis/catalyst.html)

---

## 9. VTK on Headless Linux Servers and Containers

Scientific visualization increasingly runs in containerised or HPC batch environments without displays. VTK supports several headless configurations.

### OSMesa Build

OSMesa (Mesa Offscreen rendering) provides a software OpenGL implementation that requires no GPU and no display server. Build VTK against `libOSMesa`:

```bash
# On Ubuntu/Debian
apt-get install libglu1-mesa-dev libosmesa6-dev

cmake -S vtk -B build-osmesa \
  -DVTK_USE_X=OFF \
  -DVTK_OPENGL_HAS_OSMESA=ON \
  -DOSMESA_INCLUDE_DIR=/usr/include/GL \
  -DOSMESA_LIBRARY=/usr/lib/x86_64-linux-gnu/libOSMesa.so \
  -DVTK_DEFAULT_RENDER_WINDOW_OFFSCREEN=ON
```

OSMesa is suitable for CI pipelines, fully CPU-based containers, and situations where GPU access is unavailable.

### EGL Build for GPU-Accelerated Headless

EGL headless rendering uses the GPU without an X or Wayland display. On a bare metal server or Kubernetes GPU node:

```bash
cmake -S vtk -B build-egl \
  -DVTK_USE_X=OFF \
  -DVTK_OPENGL_HAS_EGL=ON \
  -DVTK_DEFAULT_RENDER_WINDOW_HEADLESS=ON
```

In a Kubernetes pod with the NVIDIA device plugin, `/dev/nvidia0` and the NVIDIA EGL ICD (`/usr/share/egl/egl_external_platform.d/`) are injected, making `vtkEGLRenderWindow` use the GPU. Select a specific GPU on a multi-GPU node:

```bash
export VTK_DEFAULT_EGL_DEVICE_INDEX=2  # Third GPU
```

For AMD GPUs on ROCm nodes, the Mesa RADV driver exposes an EGL implementation via `/dev/dri/renderD128`; `EGL_PLATFORM=drm` selects the DRM EGL platform. [Note: needs verification of exact EGL platform token for AMD headless rendering.]

### Official Docker Images

Kitware provides Docker images with VTK pre-built:

```bash
# VTK with OSMesa (CPU rendering, no GPU required)
docker pull kitware/vtk:latest-osmesa

# Run a Python VTK script headless
docker run --rm -v $(pwd):/work kitware/vtk:latest-osmesa \
  python3 /work/render.py
```

For GPU-enabled containers, use the NVIDIA Container Toolkit base image and VTK built with EGL:

```bash
docker run --rm --gpus all \
  -v $(pwd):/work \
  -e VTK_DEFAULT_OPENGL_WINDOW=vtkEGLRenderWindow \
  kitware/vtk:latest-egl \
  python3 /work/render.py
```

[Source: Kitware Docker Hub](https://hub.docker.com/r/kitware/vtk)

### vtk.js — Browser-Based VTK

For scenarios where server-side rendering is impractical, [vtk.js](https://kitware.github.io/vtk-js/) provides a JavaScript/TypeScript implementation of VTK's core algorithms running entirely in the browser via WebGL or WebGPU, with no server required. vtk.js supports volume rendering (GPU ray casting via WebGL `OES_texture_3D`), surface rendering, and point clouds. [Source](https://github.com/Kitware/vtk-js)

### trame — Python Web Application Framework

[trame](https://trame.readthedocs.io/en/latest/) (Kitware, 2022+) is a Python framework for building interactive scientific visualization web applications. A trame application:
- Runs a Python process that owns a VTK or ParaView pipeline.
- Serves a Vue.js frontend over WebSocket (using aiohttp or tornado).
- Communicates pipeline state changes and rendered images from server to browser.
- Supports both **remote rendering** (server renders, client displays JPEG) and **local rendering** (server sends geometry to vtk.js in the browser, client GPU renders).

```python
from trame.app import get_server
from trame.ui.vuetify3 import SinglePageLayout
from trame.widgets import vuetify3 as v3, vtk as vtkw
from vtkmodules.vtkFiltersSources import vtkSphereSource
from vtkmodules.vtkRenderingCore import (
    vtkActor, vtkPolyDataMapper, vtkRenderer, vtkRenderWindow,
)

server = get_server()
state, ctrl = server.state, server.controller

sphere = vtkSphereSource()
mapper = vtkPolyDataMapper()
mapper.SetInputConnection(sphere.GetOutputPort())
actor = vtkActor()
actor.SetMapper(mapper)

renderer = vtkRenderer()
renderer.AddActor(actor)
render_window = vtkRenderWindow()
render_window.AddRenderer(renderer)
render_window.OffScreenRenderingOn()

with SinglePageLayout(server) as layout:
    with layout.content:
        with v3.VContainer(fluid=True, classes="fill-height"):
            vtkw.VtkRemoteView(render_window, ref="view")

server.start()
```

[Source: trame documentation](https://trame.readthedocs.io/en/latest/)

---

## 10. Integration with the Scientific Ecosystem

### NumPy Interoperability

The `vtkmodules.util.numpy_support` module provides bidirectional zero-copy conversion between NumPy arrays and VTK arrays:

```python
from vtkmodules.util.numpy_support import numpy_to_vtk, vtk_to_numpy
import numpy as np

# NumPy → VTK (zero-copy when array is C-contiguous float32/float64/int32/...)
velocity_np = np.random.rand(1000, 3).astype(np.float64)
vtk_array = numpy_to_vtk(velocity_np, deep=False)  # zero-copy reference
vtk_array.SetName("Velocity")
polydata.GetPointData().AddArray(vtk_array)

# VTK → NumPy (view into VTK buffer)
scalars_back = vtk_to_numpy(polydata.GetPointData().GetScalars())
print(f"min={scalars_back.min():.3f}, max={scalars_back.max():.3f}")
```

The `deep=False` parameter (default) passes a pointer to the NumPy buffer — the NumPy array must be kept alive as long as the VTK array is in use. [Source](https://docs.vtk.org/en/latest/api/python/vtkmodules/vtkmodules.util.numpy_support.html)

The `vtkmodules.numpy_interface.dataset_adapter` module wraps VTK datasets with a NumPy-like interface for algorithmic work:

```python
from vtkmodules.numpy_interface import dataset_adapter as dsa

# Wrap a vtkUnstructuredGrid for NumPy-style field access
wrapped = dsa.WrapDataObject(ugrid)
pressure = wrapped.PointData["Pressure"]   # returns a VTKArray (np.ndarray subclass)
velocity = wrapped.PointData["Velocity"]   # shape (N, 3)
speed = np.linalg.norm(velocity, axis=1)
```

### Jupyter Notebooks and itkwidgets

VTK renders in Jupyter notebooks through:
- **`ipyvtk-simple`** / **`panel`**: embed VTK render windows as interactive widgets.
- **itkwidgets** ([github.com/InsightSoftwareConsortium/itkwidgets](https://github.com/InsightSoftwareConsortium/itkwidgets)): purpose-built Jupyter widget for 3D and 2D viewing of VTK datasets, ITK images, and NumPy arrays with interactive volume rendering in the browser.

```python
from itkwidgets import view
from vtkmodules.vtkIOImage import vtkMetaImageReader

reader = vtkMetaImageReader()
reader.SetFileName("brain.mhd")
reader.Update()

view(reader.GetOutput())   # interactive 3D volume rendering in Jupyter cell
```

### 3D Slicer

[3D Slicer](https://www.slicer.org/) is the leading open-source medical imaging application, built on VTK and ITK. 3D Slicer 5.x uses VTK 9.3 and ITK 5.6. [Source](https://pmc.ncbi.nlm.nih.gov/articles/PMC3466397/)

Slicer extends VTK with:
- **MRML** (Medical Reality Markup Language) — a scene graph layer above VTK storing nodes for volumes, segmentations, markups, and transforms.
- **VTK-ITK bridge** — `itk::VTKImageToImageFilter` and `itk::ImageToVTKImageFilter` convert between ITK and VTK image representations, enabling ITK's registration and segmentation algorithms to operate on data loaded with VTK's DICOM reader.
- A Python scripting console with `slicer.util.getNode()` access to MRML scene objects.

### VTK versus OpenCASCADE (Ch176)

VTK and OpenCASCADE (OCCT) serve complementary rather than competing roles:

| Aspect | VTK | OCCT |
|---|---|---|
| Geometry representation | Discrete: triangles, voxels, point clouds | Exact: B-spline/NURBS BRep surfaces |
| Primary use case | Data visualization, simulation post-processing | CAD modelling, geometric computation |
| Mesh topology | `vtkUnstructuredGrid` with heterogeneous cells | `BRepMesh` tessellates BRep for display |
| Data exchange | VTU, HDF, NetCDF, DICOM, Exodus | STEP, IGES, BREP, glTF |
| GPU volume rendering | `vtkGPUVolumeRayCastMapper` | Not applicable |
| Boolean operations | Not natively (use `vtkBooleanOperationPolyDataFilter` for surface meshes only) | Full BRep CSG (BRepAlgoAPI) |

FreeCAD uses both: OCCT as the geometric kernel for CAD operations, and VTK (optionally) for FEM results visualization via the FEM workbench. See Chapter 176 for the OCCT side of this comparison.

### VTK in HPC Simulation Workflows

A typical HPC workflow using VTK and ParaView has three phases:

1. **Simulation**: A parallel code (e.g., OpenFOAM, WarpX, LAMMPS) runs on thousands of MPI ranks, writing output as parallel VTK XML files (`.pvtu`) or via Catalyst in-situ.
2. **Post-processing**: ParaView on a viz cluster loads the `.pvtu` files, applies filters (Contour, Clip, Warp, Calculator), and generates images or extracted datasets. VTK-m filters (Contour, Gradient) are used for the compute-intensive steps.
3. **Exploration**: Scientists connect a local ParaView GUI client to a `pvserver` on the viz cluster, exploring results interactively over the network. Large-time-series data may be served by a Catalyst live connection or an ADIOS2 staging server.

VTK's role spans all three phases: `.pvtu` writing uses `vtkXMLPUnstructuredGridWriter`; in-situ uses Catalyst + VTK data structures; post-processing uses the full VTK filter graph; visualization uses `vtkGPUVolumeRayCastMapper` or `vtkSmartVolumeMapper` for volumes and `vtkPolyDataMapper` for surfaces.

### VTK versus Blender Cycles (Ch42)

VTK's volume rendering and Blender Cycles address different ends of the visualization spectrum:

| Aspect | VTK (GPU Ray Cast) | Blender Cycles |
|---|---|---|
| Transfer function control | `vtkColorTransferFunction` + `vtkPiecewiseFunction` | Shader nodes with volume scatter/absorption |
| Data input | `vtkImageData` (3D scalar/vector arrays) | OpenVDB volumes, point clouds |
| Rendering goal | Scientific accuracy, data fidelity | Photorealistic appearance |
| Interactivity | Real-time interactive at typical CT resolutions | Progressive rendering, slower first frame |

A common workflow in scientific publication: visualize in ParaView to understand data structure and tune transfer functions, then export a surface mesh (`.ply` or `.obj`) from VTK and import into Blender for final publication-quality rendering with Cycles. VTK can write `.ply` and `.obj` via `vtkPLYWriter` and `vtkOBJWriter`.

---

## 11. Integrations

This chapter connects to the following chapters across the book:

**Ch12 — Mesa Loader**: VTK's OpenGL backend on Linux uses Mesa as the default OpenGL implementation on systems without proprietary GPU drivers. The Mesa loader (`libGL.so.1` → RADV/ANV/softpipe dispatch) is traversed every time `vtkXOpenGLRenderWindow` or `vtkEGLRenderWindow` calls `glDrawArrays`. VTK 9.4's switch to runtime `glad` loading (carried forward through 9.6) interacts directly with the Mesa dynamic dispatch layer.

**Ch17 — Software Renderers**: VTK's `VTK_OPENGL_HAS_OSMESA=ON` build links against Mesa's `libOSMesa.so`, the Mesa offscreen software renderer described in Ch17. OSMesa is the fallback for CI pipelines and CPU-only container deployments.

**Ch24 — Vulkan and EGL**: `vtkEGLRenderWindow` uses EGL as described in Ch24 for context creation. The experimental Vulkan render window branch would use `VK_KHR_xlib_surface` or `VK_KHR_wayland_surface` for swapchain creation — the same WSI extensions covered in Ch24.

**Ch25 — GPU Compute**: VTK-m's CUDA and HIP device adapters issue GPU compute kernels for parallel filters (Contour, Gradient, Streamlines). The GPU memory model — staging buffers, device synchronisation — mirrors the patterns covered in Ch25.

**Ch42 — Blender GPU**: Blender Cycles (Ch42) and VTK's `vtkGPUVolumeRayCastMapper` both perform GPU volume ray casting but with different priorities: Cycles optimises for photorealism; VTK optimises for scientific data fidelity and interactive transfer function editing. Meshes exported from VTK are a common input to Blender's Cycles renderer.

**Ch48 — ROCm**: VTK-m's HIP/ROCm device adapter (via Kokkos) targets AMD GPUs as described in Ch48. ROCm 6+ is required for the `hipcc`-compiled VTK-m Kokkos backend.

**Ch55 — GPU Containers**: Deploying VTK in Kubernetes with GPU access — NVIDIA device plugin, EGL ICD injection, `VTK_DEFAULT_OPENGL_WINDOW=vtkEGLRenderWindow` — builds on the GPU container infrastructure described in Ch55.

**Ch107 — Headless Rendering**: VTK's OSMesa and EGL headless paths are practical applications of the headless rendering infrastructure described in Ch107, which covers `GBM`, `EGL_EXT_platform_device`, and `libdrm` device enumeration for renderonly GPU devices.

**Ch176 — OpenCASCADE**: VTK and OCCT serve complementary roles in the Linux scientific application ecosystem. Section 10 of this chapter compares their data models, use cases, and geometry representations. FreeCAD bridges both toolkits.

---

*References consulted for this chapter:*

- [VTK Repository](https://gitlab.kitware.com/vtk/vtk) — primary source
- [VTK 9.6 Release Notes](https://docs.vtk.org/en/latest/release_details/9.6.html)
- [VTK 9.4 Release Notes](https://docs.vtk.org/en/latest/release_details/9.4.html)
- [VTK 9.3 Release Notes](https://docs.vtk.org/en/latest/release_details/9.3.html)
- [VTK Build Settings](https://docs.vtk.org/en/latest/build_instructions/build_settings.html)
- [VTK-m Documentation](https://docs-m.vtk.org/latest/)
- [VTK-m Users Guide V.2.0](https://www.osti.gov/biblio/1959590)
- [VTKHDF File Format](https://docs.vtk.org/en/latest/vtk_file_formats/vtkhdf_file_format/index.html)
- [ParaView Documentation](https://docs.paraview.org/en/latest/)
- [Catalyst In-Situ Documentation](https://catalyst-in-situ.readthedocs.io/en/latest/)
- [VTK ANARI Module](https://docs.vtk.org/en/latest/release_details/9.4/add-anari-rendering-capability.html)
- [ANARI Standard — Khronos](https://www.khronos.org/anari/)
- [vtk.js Repository](https://github.com/Kitware/vtk-js)
- [trame Documentation](https://trame.readthedocs.io/en/latest/)
- [vtkmodules.util.numpy_support](https://docs.vtk.org/en/latest/api/python/vtkmodules/vtkmodules.util.numpy_support.html)
- [3D Slicer as Imaging Platform](https://pmc.ncbi.nlm.nih.gov/articles/PMC3466397/)
- [IceT Scalable Rendering](https://www.kennethmoreland.com/scalable-rendering/)
- [Kitware VTK 9.4.0 Blog Post](https://www.kitware.com/vtk-v9-4-0/)

## Roadmap

### Near-term (6–12 months)
- VTK's WebGPU backend (`vtkRenderingWebGPU`) is actively receiving volume rendering support; the near-term target is feature parity with the OpenGL2 `vtkGPUVolumeRayCastMapper` for the Dawn/Vulkan path on Linux desktop.
- The VTKHDF format is on track to supersede parallel VTK XML (`.pvtu`/`.pvtp`) as the recommended output format for new HPC simulation codes, with continued MPI-IO performance work and time-series indexing improvements in VTK 9.7.
- VTK-m 2.1 is expected to ship improved Kokkos/SYCL support targeting Intel GPU (PVC/Xe) compute, enabling VTK-m filters to run on Intel GPUs via Level Zero without a separate CUDA or HIP compilation.
- The `vtkCellGrid` discontinuous Galerkin dataset class is scheduled to receive GPU-side rendering of high-order elements via tessellation shaders in the WebGPU backend, completing the round-trip from DG solver output to interactive visualization without tessellation on the CPU.

### Medium-term (1–3 years)
- A production-quality Vulkan rendering backend for VTK (going beyond the 2020 Ken Martin proof-of-concept) is a stated longer-term goal in VTK community discussions; the WebGPU/Dawn path currently serves as a Vulkan-backed route, but a native `vtkVulkanRenderWindow` would enable direct access to Vulkan ray tracing extensions (VK_KHR_ray_tracing_pipeline) without the WebGPU abstraction layer.
- ParaView's trame-based web frontend is expected to fully replace the legacy Qt client for remote HPC visualization workflows, with vtk.js and WebGPU in the browser handling local rendering for datasets below the geometry threshold.
- VTK-m's Kokkos backend is expected to mature into the recommended single unified GPU compute path, consolidating the separate CUDA and HIP adapters and simplifying multi-vendor HPC deployments.
- ANARI integration within ParaView is planned to expose path-traced rendering (via VisRTX or OSPRay) as a first-class render mode selectable in the GUI, not just via the Python API.

### Long-term
- As WebAssembly and WebGPU mature, a full server-free VTK pipeline running in the browser (built via Emscripten with `VTK_WRAP_JAVASCRIPT`) is a plausible trajectory, enabling scientific visualization applications to run entirely client-side with no HPC backend required for moderate dataset sizes.
- Deep learning-based upsampling and reconstruction filters (leveraging ONNX inference, added experimentally in VTK 9.6) may become first-class VTK pipeline stages, enabling AI-assisted volume rendering and super-resolution for clinical and simulation datasets.
- Convergence of the Catalyst in-situ API with streaming HPC data fabrics (ADIOS2, RDMA-based) and cloud-native object stores (S3-compatible) is a stated direction for post-Exascale workflows, where VTK acts as the on-node serialization and analysis layer rather than a batch post-processor.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
