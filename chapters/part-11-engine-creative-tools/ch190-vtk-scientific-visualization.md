# Chapter 190: VTK — Scientific Visualization on the Linux Graphics Stack

**Target audiences:** Scientific computing developers using VTK for data visualization on Linux; HPC engineers deploying VTK in cluster and cloud environments; researchers using ParaView and 3D Slicer; graphics developers integrating VTK with Vulkan, WebGPU, or OpenGL.

---

## Table of Contents

1. [VTK Overview and Architecture](#1-vtk-overview-and-architecture)
   - [1.1 What is Scientific Visualization?](#11-what-is-scientific-visualization)
   - [1.2 What is VTK?](#12-what-is-vtk)
   - [1.3 What is ParaView?](#13-what-is-paraview)
   - [Design Note: Is VTK's Architecture Showing Its Age?](#design-note-is-vtks-architecture-showing-its-age)
2. [VTK Data Model](#2-vtk-data-model)
3. [VTK Rendering Backends on Linux](#3-vtk-rendering-backends-on-linux)
4. [GPU Volume Rendering](#4-gpu-volume-rendering)
5. [VTK WebGPU and ANARI Backends](#5-vtk-webgpu-and-anari-backends)
6. [VTK-m: GPU-Accelerated Filters](#6-vtk-m-gpu-accelerated-filters)
7. [VTK I/O — Scientific Data Formats](#7-vtk-io--scientific-data-formats)
8. [ParaView — The Flagship VTK Application](#8-paraview--the-flagship-vtk-application)
9. [VTK on Headless Linux Servers and Containers](#9-vtk-on-headless-linux-servers-and-containers)
   - [9.1 vtk.js — Browser-Based VTK](#vtkjs--browser-based-vtk)
   - [9.2 vtk-wasm — C++ VTK Compiled to WebAssembly](#vtk-wasm--c-vtk-compiled-to-webassembly)
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

### 1.1 What is Scientific Visualization?

Scientific visualization is the discipline concerned with constructing graphical representations of data generated by physical simulations, sensor arrays, medical scanners, and computational models. The defining characteristic of scientific visualization is that the underlying data is inherently spatial: volumetric fields such as temperature, pressure, and fluid velocity; surface meshes produced by finite element solvers; and structured grids from computational fluid dynamics codes. Converting these numerical arrays into geometry — isosurfaces, streamlines, vector glyphs, volume renderings — allows researchers to perceive patterns and structures that would remain invisible in tabular form.

On Linux, scientific visualization sits at the boundary between the HPC software stack and the graphics driver stack. A parallel simulation running under MPI writes output in a domain-specific format such as HDF5, CGNS, or NetCDF. A visualization framework reads those files, builds a geometric representation through a series of data-transformation filters, and submits the resulting geometry to a rendering backend that communicates with the kernel DRM subsystem through OpenGL, EGL, or Vulkan. The data pipeline must therefore handle both the numerical precision requirements of scientific data — 64-bit floating point, sparse and adaptive meshes, multi-component field arrays — and the GPU resource model imposed by the graphics API. VTK is designed specifically to occupy this intersection, providing the data model, filter library, and rendering abstraction that connect scientific datasets to Linux GPU drivers without requiring the application developer to interact with DRM, Mesa internals, or compositor protocols directly.

### 1.2 What is VTK?

VTK, the Visualization Toolkit, is an open-source C++ library for 3D computer graphics, image processing, and scientific visualization. It provides three integrated layers: a typed dataset hierarchy for representing volumetric, surface, and point-cloud data; a filter library of several hundred algorithms for transforming, resampling, and analyzing those datasets; and a rendering abstraction that delegates to OpenGL (the production backend), WebGPU, or the ANARI scene-description API. Python bindings distributed as the `vtkmodules` package expose the full C++ API without native compilation, making VTK accessible in scientific Python workflows alongside NumPy and SciPy.

VTK's pipeline model is demand-driven: sources, filters, and sinks are connected through typed ports, and calling `Update()` on a consumer node traverses the dependency graph and re-executes only those stages whose inputs have changed since the last evaluation. On Linux, VTK integrates with the native graphics stack through three render window implementations: `vtkXOpenGLRenderWindow` for X11/GLX desktop display, `vtkEGLRenderWindow` for headless and Wayland rendering via EGL and the `/dev/dri/renderD128` device node, and `vtkOSOpenGLRenderWindow` backed by Mesa's `libOSMesa` for CPU-only environments. VTK does not itself implement a compositor or DRM client; it relies on the Mesa or proprietary driver beneath EGL or GLX to arbitrate GPU access through the kernel DRM subsystem. The current stable series is VTK 9.6, with active development tracked at [gitlab.kitware.com/vtk/vtk](https://gitlab.kitware.com/vtk/vtk).

### 1.3 What is ParaView?

ParaView is an open-source, parallel scientific visualization application built on top of VTK. Its distinguishing design is a client-server architecture suited to large-scale HPC datasets: the ParaView server (`pvserver`) executes VTK filters in parallel across MPI ranks on the compute cluster, composites the rendered tile images using the IceT parallel image compositing library, and streams a compressed frame to a thin GUI client running on the user's workstation. This separation allows datasets that exceed the memory capacity of any single node — hundreds of gigabytes or more — to be explored interactively without transferring raw simulation data across the network.

Beyond the parallel rendering architecture, ParaView extends VTK with its own reader and filter plugin system, the Catalyst in-situ analysis framework for embedding visualization directly inside a running simulation without writing full checkpoint files, and the trame web framework for deploying VTK-backed visualization as a browser application. On Linux, `pvserver` typically executes under a job scheduler alongside MPI-parallel simulation codes and uses VTK's EGL render window backend for headless GPU rendering on compute nodes equipped with NVIDIA or AMD GPUs. ParaView and VTK share a CMake-based build system and are co-developed on overlapping release schedules; the required VTK version is pinned in ParaView's `CMakeLists.txt`. [Source](https://www.paraview.org/paraview-guide/)

### Design Note: Is VTK's Architecture Showing Its Age?

VTK's core object model dates to 1993, and several of its defining mechanisms predate the C++ standard library features that would now be the default choice. This is not a matter of opinion — the toolkit's own architects have written candidly about which decisions they still endorse and which they consider regrets.

**Intrusive reference counting instead of RAII/GC.** Every `vtkObject` subclass carries its own `Register()`/`UnRegister()` count rather than relying on garbage collection or exclusively on `std::shared_ptr`. The rationale, per the toolkit's own architecture writeup, is that visualization datasets can be enormous — "a volume of byte data 1000×1000×1000 in size is a gigabyte in size, and it is not a good idea to leave such data lying around while the garbage collector decides whether or not it is time to release it" — and reference counting also makes the pipeline's routine shallow-copying of arrays between filters cheap. `vtkSmartPointer<T>` was added later as an RAII wrapper around the same underlying mechanism, but plenty of APIs still hand back raw pointers, and correct lifetime management still depends on conventions (protected constructors/destructors, deleted copy/assignment) rather than the type system. [Source: The Architecture of Open Source Applications — VTK](https://aosabook.org/en/v1/vtk.html) [Source: VTK Coding Conventions](https://docs.vtk.org/en/latest/developers_guide/coding_conventions.html)

**Macro-generated boilerplate.** `vtkTypeMacro`, `vtkSetMacro`/`vtkGetMacro`, and `vtkStandardNewMacro` generate RTTI, accessors, and factory construction via the preprocessor rather than templates. This is not purely cosmetic: the Set/Get macros also update the object's modified-time (`MTime`) stamp that drives the demand-driven pipeline's dirty-checking, so bypassing them for a hand-written setter is, in the architects' own words, "a particularly pernicious bug" — the pipeline silently fails to re-execute because it never observed the change.

**A pipeline the authors call too complex.** The three-request (`RequestInformation`/`RequestUpdateExtent`/`RequestData`) demand-driven executive that Section 2 and Section 3 build on was not part of VTK's original design — an explicit `vtkExecutive` object was added only after the implicit version proved unworkable. VTK's own architecture chapter states plainly: "the data processing pipeline in VTK is still too complex. Methods are under way to simplify and refactor this subsystem." [Source: AOSA — VTK](https://aosabook.org/en/v1/vtk.html)

**No C++ exceptions.** VTK reports errors through `vtkErrorMacro`/`vtkWarningMacro` plus manual return-code checks rather than throwing. The practical failure mode this creates — flagged repeatedly on the VTK mailing lists — is that an error early in a pipeline does not stop downstream filters from executing on bad data unless every intermediate stage explicitly checks `GetErrorCode()` or `vtkAlgorithm::GetErrorOccurred()`. [Source: vtkusers mailing list, "VTK C++ Exception handling?"](https://vtk.org/pipermail/vtkusers/2011-July/068927.html)

None of this makes VTK unfit for its job — it makes it a large, load-bearing codebase from an era with different C++ idioms, still under active renovation rather than left to rot. The modernization efforts described elsewhere in this chapter are direct responses to these exact pain points: `vtkArrayDispatch` (Section 6) replaces per-element virtual-call dispatch with compile-time type resolution; `vtkImplicitArray` (Section 2) avoids materializing memory for constant or affine fields; the `>>` pipeline-connection operator (Section 1) is a small but real ergonomics fix over chained `SetInputConnection()` calls; and the VTK-m device-adapter split (Section 6) moves the performance-critical filter code onto a modern, GPU-portable execution model entirely separate from the legacy `vtkAlgorithm` dispatch path. The pattern across all four is the same: keep the 30-year-old object model and pipeline contract stable for the enormous body of dependent code (ParaView, 3D Slicer, and downstream HPC applications), and modernize underneath it incrementally rather than through a rewrite.

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

### Kokkos: The Portability Layer Behind the Adapter

Kokkos is not a VTK-m-specific technology — it is a general-purpose C++ performance-portability library that originated at Sandia National Laboratories, first released in 2017 and now maintained at [github.com/kokkos/kokkos](https://github.com/kokkos/kokkos). It exposes a single template-based C++ API for parallel loops, reductions, and multidimensional array views, with a pluggable backend that compiles the same source against CUDA (NVIDIA), HIP (AMD/ROCm), SYCL (Intel oneAPI and other cross-vendor targets), OpenMP, or serial CPU execution — the calling application does not maintain separate device-specific code paths. [Source](https://www.osti.gov/servlets/purl/1457941) Kokkos underpins Sandia's Trilinos numerical libraries and has been used to port production HPC codes such as LAMMPS (molecular dynamics) and SPARTA (direct simulation Monte Carlo) to run, largely unmodified, across NVIDIA, AMD, and Intel GPUs, including on OLCF Frontier, the first exascale system. [Source](https://www.nas.nasa.gov/pubs/ams/2024/04-04-24.html)

VTK-m's `VTKM_DEVICE_ADAPTER_KOKKOS` adapter delegates worklet execution to Kokkos rather than talking to CUDA, HIP, or SYCL directly, so a single Kokkos-backed VTK-m build can retarget GPU vendor by switching `Kokkos_ENABLE_*` CMake flags instead of VTK-m maintaining separate CUDA and HIP device adapters in parallel. This is the basis for the Roadmap's (below) expectation that Kokkos matures into VTK-m's single unified GPU compute path, and for VTK-m 2.1's targeted Kokkos/SYCL work for Intel GPUs (PVC/Xe), which reaches the hardware through the Level Zero backend that Intel's SYCL implementation uses on its own GPUs. [Source](https://github.com/Kitware/VTK-m)

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

Introduced in VTK 9.1, the VTKHDF format uses HDF5 as the storage layer but with a VTK-defined group/dataset layout, avoiding the auxiliary XML mapping file that plain XDMF-over-HDF5 requires and the extra machinery ADIOS2 brings for developers who just want a self-describing HDF5 file. Reader: `vtkHDFReader`. Supported dataset types: PolyData, UnstructuredGrid, ImageData, RectilinearGrid, StructuredGrid, HyperTreeGrid, Table, and the composite types OverlappingAMR, MultiBlockDataSet, and PartitionedDataSetCollection. [Source](https://www.kitware.com/vtk-hdf-reader/)

**Group/dataset schema.** Every VTKHDF file opens with a top-level `/VTKHDF` group carrying two attributes: `Version` (a two-integer array — 2.5 as of the 2025 status update) and `Type` (a string naming the dataset class stored in the file, e.g. `UnstructuredGrid`). For an UnstructuredGrid, that group holds the flattened `Points` and `Connectivity` datasets plus the count datasets `NumberOfPoints`, `NumberOfCells`, and `NumberOfConnectivityIds`, alongside `PointData`, `CellData`, and `FieldData` subgroups that mirror VTK's in-memory attribute model directly, so no separate schema translation is needed on read. ImageData instead stores its regular grid using HDF5 chunking, so hyperslabs align with VTK's native array layout without re-splitting arrays. [Source](https://docs.vtk.org/en/latest/vtk_file_formats/vtkhdf_file_format/vtkhdf_specifications.html)

**Parallel I/O.** For UnstructuredGrid and PolyData, each MPI rank writes its own contiguous "piece" into the shared, flattened `Points`/`Connectivity` arrays; the `NumberOfPoints`, `NumberOfCells`, and `NumberOfConnectivityIds` datasets record one entry per piece, letting every rank compute its own byte offset and issue an independent HDF5 hyperslab read or write — no intermediate copy and no per-rank piece file. This replaces the master-index-plus-piece-files design of parallel VTK XML (`.pvtu`/`.pvtp`, above) with a single shared file that HDF5's own parallel I/O layer (built on MPI-IO) writes to concurrently. ImageData parallelizes the same way, via chunked hyperslabs over the regular grid. [Source](https://www.kitware.com/vtk-hdf-reader/)

**Time-series indexing.** Temporal data is stored by flattening every timestep's arrays end-to-end and tracking positions in a dedicated `Steps` group: `NSteps` records the timestep count, `Values` holds the simulation time for each step, and a family of offset datasets — `PointOffsets`, `CellOffsets`, `ConnectivityIdOffsets`, `PartOffsets`/`NumberOfParts`, `PointDataOffsets` — tell the reader where each timestep's geometry and field data begin within the flattened arrays. Because the offsets are explicit rather than implied by a fixed per-step stride, a static-geometry simulation can repeat the same `PointOffsets`/`CellOffsets` entry across steps and store only the changing field data — the "static mesh" caching added in the 2025 update, which lets `vtkHDFReader` skip re-reading unchanging topology at every timestep. VTK 9.6/9.7 development extended this composite/parallel/time-dependent writer support specifically to unstructured meshes. [Source](https://www.kitware.com/how-to-write-time-dependent-data-in-vtkhdf-files/)

This is why the Roadmap (below) expects VTKHDF to supersede `.pvtu`/`.pvtp` for new HPC codes: one shared file replaces a master-index-plus-N-piece-files layout, HDF5's native parallel I/O substitutes for VTK's own MPI-IO glue code around the XML readers/writers, and the `Steps` group gives ParaView's animation scene (Section 8, above) direct random access to any timestep without scanning a directory of piece files. A 2025 status update reports these I/O performance improvements for temporal datasets specifically. [Source](https://www.kitware.com/vtkhdf-file-format-2025-status-update/)

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

**Connecting from a Python client.** `pvpython` (or any Python interpreter with the `paraview` package on its path) can drive a `pvserver` that is already running on a cluster's login or visualization node:

```python
from paraview.simple import *

# pvserver -sp=11111  (started separately on the remote host)
Connect("viz03.cluster.example.org", 11111)

reader = OpenDataFile("/scratch/simulation/output.pvtu")
Show(reader)
Render()
```

`Connect()` establishes the client-server socket connection that all subsequent `paraview.simple` calls are routed through — data stays server-side, and only render results or explicitly-fetched arrays cross the network. The threshold-switching behaviour above is exposed as the `RemoteRenderThreshold` property on the active render view, in megabytes of geometry: ParaView remote-renders once geometry exceeds this size, and local-renders (streams geometry to the client GPU) below it. Setting it to `0` forces remote rendering unconditionally — useful when scripting a batch job where the client machine has no GPU at all. [Source](https://docs.paraview.org/en/latest/ReferenceManual/parallelDataVisualization.html)

```python
view = GetActiveViewOrCreate("RenderView")
view.RemoteRenderThreshold = 0  # always render server-side, ignore local geometry size
```

### IceT — Sort-Last Compositing

For parallel rendering across many MPI ranks, ParaView uses **IceT** (Image Compositing Engine for Tiles), developed at Sandia National Laboratories: [gitlab.kitware.com/icet/icet](https://gitlab.kitware.com/icet/icet).

In **sort-last** compositing, each MPI rank renders its partition of the data to a local image with depth. IceT then performs a **binary-swap** depth compositing algorithm: ranks exchange and merge image fragments, halving the number of active participants at each round. The final composited image (with correct depth ordering) is available on rank 0 for transmission to the client. IceT has been demonstrated at 64,000 cores on IBM BlueGene systems. [Source](https://www.kennethmoreland.com/scalable-rendering/)

For display wall tiling, IceT assigns each tile of the wall display to a subset of render nodes, compositing only the required region on each node. A tile display is built by calling `icetAddTile` once per physical tile, where the first four arguments give the tile's viewport `⟨x, y, width, height⟩` within the overall wall and the last argument names the MPI rank responsible for displaying it:

```c
/* IceT: describe a 2x2 CAVE/power-wall of 1920x1080 tiles, one rank per tile */
icetAddTile(0,    0, 1920, 1080, /* display_rank = */ 0);
icetAddTile(1920, 0, 1920, 1080, /* display_rank = */ 1);
icetAddTile(0, 1080, 1920, 1080, /* display_rank = */ 2);
icetAddTile(1920, 1080, 1920, 1080, /* display_rank = */ 3);
```

Each process can query `ICET_NUM_TILES`, `ICET_TILE_VIEWPORTS`, and `ICET_DISPLAY_NODES` to inspect the resulting layout at runtime, and a process reads its own `ICET_TILE_DISPLAYED` state variable (`-1` if it displays nothing) to know whether it owns a tile. [Source: IceT Users' Guide](https://www.sandia.gov/app/uploads/sites/150/2021/10/IceTUsersGuide-2-0.pdf) ParaView applications do not call these functions directly — `pvserver` derives the same tile layout from its own command-line flags when launched against a physical display wall (`--tdx`/`--tile-dimensions-x` and `--tdy`/`--tile-dimensions-y` for the tile counts, `--tmx`/`--tmy` for the pixel gap between adjacent tiles), translating them into the equivalent `icetAddTile` calls internally. [Source](https://docs.paraview.org/en/latest/UsersGuide/commandLineArguments.html)

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

**Generating scripts instead of hand-writing them.** ParaView's GUI has a **Trace Recorder** (Tools → Start Trace / Stop Trace) that records every action taken interactively — loading a file, adding a filter, clicking Apply — and emits the equivalent `paraview.simple` calls, including the lower-level rendering and view-update calls the GUI performs implicitly that a hand-written script would otherwise omit. This is normally the fastest way to bootstrap a batch script: build the pipeline once in the GUI against a small representative dataset, trace it, then generalise the traced script (parameterise the filename, wrap the render loop) for `pvbatch` on the full dataset. A companion, non-recording route is **File → Save State**, saved as a `.py` state file rather than the binary `.pvsm` format, which reconstructs the pipeline as it exists at save time without a full action trace. Both a traced script and a saved state script can be replayed headlessly:

```python
# Reload a previously saved/traced pipeline and re-render at a new resolution
from paraview.simple import *

LoadState("/scratch/state/iso_pipeline.py")
view = GetActiveViewOrCreate("RenderView")
SaveScreenshot("/scratch/images/iso_hires.png", view, ImageResolution=[3840, 2160])
```

[Source: ParaView Documentation — Python & Batch Tutorial](https://docs.paraview.org/en/latest/Tutorials/ClassroomTutorials/pythonAndBatchParaViewAndPython.html)

**Animating a time series.** `paraview.simple` exposes one process-wide `GetAnimationScene()`, which is driven over the reader's available timesteps and rendered frame-by-frame with `SaveAnimation` (writing directly to a movie container) or a manual `SaveScreenshot` loop for per-frame PNGs:

```python
scene = GetAnimationScene()
scene.UpdateAnimationUsingDataTimeSteps()  # step through every timestep in the reader

SaveAnimation(
    "/scratch/images/run.avi",
    GetActiveViewOrCreate("RenderView"),
    ImageResolution=[1920, 1080],
    FrameRate=24,
)
```

[Source: ParaView/Python `simple.animation` module documentation](https://www.paraview.org/paraview-docs/latest/python/paraview.simple.animation.html)

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

**The `pipeline.py` script itself.** The file named in `catalyst/scripts/script0/filename` above is an ordinary Python module using the `paraview.simple` API, with three optional lifecycle hooks that the ParaView Catalyst implementation calls at the points their names suggest — module-level code (including any `paraview.simple` pipeline construction) runs once, on first import at `catalyst_initialize` time, and is *not* re-run on subsequent timesteps:

```python
from paraview.simple import *
from paraview import print_info

# Runs once, at import: build the pipeline against the registered producer.
# "input" must match the channel name the simulation registers via Conduit
# (i.e. the node at catalyst/channels/input in the per-timestep data tree).
producer = TrivialProducer(registrationName="input")

def catalyst_initialize():
    print_info("catalyst_initialize: pipeline ready")

def catalyst_execute(info):
    # info.cycle / info.time / info.timestep describe the current step
    contour = Contour(Input=producer)
    contour.ContourBy = ["POINTS", "Pressure"]
    contour.Isosurfaces = [101325.0]

    view = GetActiveViewOrCreate("RenderView")
    Show(contour, view)
    SaveScreenshot(f"catalyst_output_{info.timestep:04d}.png", view)

def catalyst_finalize():
    print_info("catalyst_finalize: simulation run complete")
```

[Source: ParaView — Anatomy of a Catalyst Python Module (Version 2.0)](https://www.paraview.org/paraview-docs/latest/cxx/CatalystPythonScriptsV2.html) The `TrivialProducer` stands in for the live simulation data VTK receives that timestep — it is bound to the Conduit channel by `registrationName`, not by reading a file, which is what makes the same pipeline script usable both for interactive prototyping against a saved dataset and for genuine in-situ execution against live simulation memory. In practice this script is rarely hand-written from scratch: building the pipeline in the ParaView GUI against a representative dataset, adding one or more **Extractors** (Extractors menu) to define what gets written out each step, and then using **File → Save Catalyst State** exports exactly this module structure automatically. [Source](https://docs.paraview.org/en/latest/Catalyst/getting_started.html)

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

For scenarios where server-side rendering is impractical, [vtk.js](https://kitware.github.io/vtk-js/) provides a JavaScript implementation of VTK's core algorithms running entirely in the browser, with no server required. Unlike vtk-wasm (below), which compiles the actual C++ VTK to WebAssembly, vtk.js is a **ground-up rewrite in ES6 JavaScript** — it does not share a code base with the C++ library, though it deliberately mirrors its object model and naming conventions so that a developer familiar with C++ VTK or ParaView can transfer that knowledge directly. The project is maintained at [github.com/Kitware/vtk-js](https://github.com/Kitware/vtk-js) and published to npm as `@kitware/vtk.js`. [Source](https://github.com/Kitware/vtk-js)

**Object model.** vtk.js reproduces the source → filter → mapper → actor → renderer → render-window pipeline from C++ VTK, but objects are created through `newInstance()` factory functions rather than constructors (a consequence of the mixin-based class system used throughout the codebase), and properties are accessed through generated `get`/`set` methods rather than public fields:

```javascript
import '@kitware/vtk.js/Rendering/Profiles/Geometry';

import vtkFullScreenRenderWindow from '@kitware/vtk.js/Rendering/Misc/FullScreenRenderWindow';
import vtkActor from '@kitware/vtk.js/Rendering/Core/Actor';
import vtkMapper from '@kitware/vtk.js/Rendering/Core/Mapper';
import vtkConeSource from '@kitware/vtk.js/Filters/Sources/ConeSource';

// vtkFullScreenRenderWindow bundles a RenderWindow + Renderer + Interactor
// into the full browser viewport — the JS analogue of the C++ Hello World
// in Section 1 (vtkRenderWindow + vtkRenderWindowInteractor)
const fullScreenRenderer = vtkFullScreenRenderWindow.newInstance();
const renderer = fullScreenRenderer.getRenderer();
const renderWindow = fullScreenRenderer.getRenderWindow();

const coneSource = vtkConeSource.newInstance({ height: 1.0 });

const mapper = vtkMapper.newInstance();
mapper.setInputConnection(coneSource.getOutputPort());

const actor = vtkActor.newInstance();
actor.setMapper(mapper);

renderer.addActor(actor);
renderer.resetCamera();
renderWindow.render();
```

[Source: VTK.js Vanilla Getting-Started Guide](https://kitware.github.io/vtk-js/docs/vtk_vanilla.html)

**Rendering backends.** vtk.js ships two render-window implementations selected at runtime rather than build time: `vtkOpenGLRenderWindow`, the default WebGL 2 backend, and `vtkWebGPURenderWindow`. `vtkFullScreenRenderWindow` picks between them based on a `defaultViewAPI: 'WebGPU'` constructor option or a `?viewAPI=WebGPU` URL query parameter, registering each backend's view constructor under a string key (`'WebGL'` / `'WebGPU'`) via `registerViewConstructor()`. This mirrors the C++ side's shift to runtime `glad` loading (Section 3) in spirit — the same object graph can be rendered through either API without recompilation, here without a build step at all. The WebGPU backend is also where vtk.js does its newest rendering work: it was the first vtk.js backend to implement physically based rendering (metallic/roughness PBR materials), which became its default lighting model. [Source](https://www.kitware.com/introducing-physically-based-rendering-to-vtk-js-webgpu/) [Source: WebGPU RenderWindow API](https://kitware.github.io/vtk-js/api/Rendering_WebGPU_RenderWindow.html)

**Data I/O and the ITK-Wasm bridge.** vtk.js includes native readers for `.vtp`/`.vti` (VTK XML), `.obj`, `.stl`, and — as of v35 — `.ply`, `.gltf`, `.tiff`, and IFC, plus a GCode reader and OBJ export. It does not include a native DICOM reader; DICOM, NIfTI, and other medical formats are instead decoded by [itk-wasm](https://github.com/InsightSoftwareConsortium/ITK-Wasm) (ITK's algorithms compiled to WebAssembly) and handed to vtk.js through `ITKHelper`, which converts an itk-wasm image object into a `vtkImageData` without a data copy. This division of labour — itk-wasm for format decoding and image processing, vtk.js for the scene graph and GPU upload — is the same pattern 3D Slicer uses on the desktop (Section 10's VTK-ITK bridge), relocated to the browser. [Source](https://github.com/Kitware/itk-vtk-viewer) [Source](https://discourse.vtk.org/t/showing-dicoms-using-vtk-js-and-itk-wasm/14461)

**Volume rendering and widgets.** vtk.js performs GPU ray-cast volume rendering through `vtkVolumeMapper` / `vtkOpenGLVolumeMapper`, the WebGL analogue of `vtkGPUVolumeRayCastMapper` (Section 4), driven by the same `vtkColorTransferFunction`/`vtkPiecewiseFunction` pair used in C++ VTK. As of v35, multi-image volume rendering renders a background image and a segmentation label image as independent GPU textures composited in a single pass — a common requirement for radiotherapy contouring and tumour-segmentation review tools. The `Widgets` module provides interaction primitives — handle widgets, a reslice-cursor widget for multi-planar reformatting (MPR) of medical volumes, and (new in v35) a `TransformControlsWidget` for interactively translating, rotating, and scaling actors. [Source: VTK.js v35 Release](https://www.kitware.com/vtk-js-v35-release/)

**WebXR.** vtk.js v35 (March 2026) added `RenderingWebXR`, letting a render window request an immersive VR or AR session through the browser's [WebXR Device API](https://www.w3.org/TR/webxr/) — the same standard covered in Chapter 203 — so that volume renderings of medical images or CFD isosurfaces can be viewed head-mounted directly from a vtk.js page, with no native VR SDK involved. [Source](https://www.kitware.com/vtk-js-transforms-web-based-visualization-with-immersive-virtual-and-augmented-reality/)

**Applications built on vtk.js.** [itk-vtk-viewer](https://github.com/Kitware/itk-vtk-viewer) is Kitware's reference 2D/3D viewer for images, meshes, and point sets, combining itk-wasm I/O with vtk.js rendering behind a thin web UI; it is commonly launched from Python (`itkwidgets`, Section 10) or as a standalone drag-and-drop page. **VolView** is Kitware's fuller browser-based DICOM viewer built the same way, adding cinematic volume rendering and segmentation tools with no server-side component required. **Glance**, by contrast, is a vtk.js/ParaViewWeb client that *does* talk to a server — it drives a remote `pvserver` or `trame` backend, occupying the "local rendering" role described in the trame subsection above: geometry streamed from the server is rendered by the client's own GPU via vtk.js instead of the server rendering and streaming a compressed image.

**Interop with three.js.** vtk.js and three.js are independent scene-graph libraries that each expect to own the WebGL/WebGPU context and render loop, so there is no supported way to embed one inside the other's canvas, and Kitware does not offer a first-party bridge. Two narrower interop paths exist instead. First, at the *file* level: three.js ships its own `VTKLoader`, which parses legacy `.vtk` files directly into a `THREE.BufferGeometry` — but only the `POLYDATA` dataset type in ASCII or binary, with no support for the VTK XML formats (`.vtp`/`.vtu`, Section 7), structured/rectilinear/unstructured grids, or appended data; it has no dependency on the vtk.js library itself. [Source](https://threejs.org/docs/pages/VTKLoader.html) Second, at the *object* level, for applications that want vtk.js's readers and filter pipeline but three.js doing the drawing: run vtk.js headless, skipping its `Rendering` module, and pull the typed arrays off the mapper's input directly — `mapper.getInputData().getPoints().getData()` for vertex positions and the equivalent calls on `getPolys()`/`getPointData().getNormals()` — to populate a `THREE.BufferGeometry`'s `position`/`index`/`normal` attributes by hand. Polygon-to-triangle conversion and any scale or coordinate-frame bookkeeping are the caller's responsibility; no library automates this hand-off. [Source: VTK Discourse — Integrating vtk.js into a Babylon.js GUI](https://discourse.vtk.org/t/integrating-vtk-js-capabilities-into-a-babylon-js-centric-gui/2670)

For the WebGL and WebGPU implementations vtk.js runs against, see Chapter 34 (ANGLE) and Chapter 35 (Dawn and WebGPU); for the underlying immersive API, see Chapter 203 (WebXR); for compiling C++ VTK itself to WebAssembly (`VTK_WRAP_JAVASCRIPT`), see the next subsection.

### vtk-wasm — C++ VTK Compiled to WebAssembly

Where vtk.js is a ground-up ES6 rewrite, **vtk-wasm** (also called VTK.wasm) takes the opposite approach: it compiles the *actual* C++ VTK source tree — the same filters, mappers, and readers used by desktop VTK and ParaView — to WebAssembly with Emscripten, and generates JavaScript bindings that mirror the C++ class hierarchy directly. Any C++ pipeline, including custom filters that only exist as compiled code, runs unmodified in the browser; nothing has to be reimplemented in JavaScript the way it does for vtk.js. [Source](https://www.kitware.com/introducing-webassembly-support-in-vtk/) [Source: vtk-wasm architecture overview](https://kitware.github.io/vtk-wasm/)

**Build: `VTK_WRAP_JAVASCRIPT`.** JavaScript wrapping is a CMake-time opt-in, parallel to `VTK_WRAP_PYTHON`: `VTK_WRAP_JAVASCRIPT` is `OFF` by default and requires `VTK_ENABLE_WRAPPING` (`ON` by default), and its help text reads simply "Whether JavaScript support will be available or not." [Source](https://docs.vtk.org/en/latest/build_instructions/build_settings.html) A minimal build, run under the Emscripten SDK's `emcmake` wrapper:

```bash
emcmake cmake -S ${VTK_SOURCE_DIR} -B ${VTK_BUILD_DIR} \
  -G "Ninja" \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=OFF \
  -DVTK_WRAP_JAVASCRIPT=ON \
  -DVTK_ENABLE_WEBGPU=ON
ninja -C ${VTK_BUILD_DIR}
```

VTK also ships CMake presets that wrap the same invocation — `cmake --workflow --preset wasm32` or `--preset wasm64`. Relevant flags beyond the basics: `VTK_WEBASSEMBLY_64_BIT` adds Emscripten's `-sMEMORY64`, widening the addressable heap from 4 GiB to 16 GiB for large datasets; `VTK_WEBASSEMBLY_THREADS` adds `-pthread`, enabling Web Worker-backed multithreading; `VTK_WASM_DEBUGINFO` selects `NONE`/`READABLE_JS`/`PROFILE`/`DEBUG_NATIVE` symbol levels for debugging a build; and `VTK_ENABLE_WEBGPU` builds the WebGPU rendering backend from Section 5 into the WASM binary alongside WebGL. Static linking (`BUILD_SHARED_LIBS=OFF`) is required, since Emscripten needs a single self-contained module. The build produces a `.wasm` binary plus `.js`/`.mjs` loader glue. [Source](https://docs.vtk.org/en/latest/advanced/build_wasm_emscripten.html)

**Runtime: sessions over a loaded bundle.** The npm package `@kitware/vtk-wasm` wraps the compiled bundle for use without a local Emscripten toolchain:

```javascript
import { loadAsync } from "@kitware/vtk-wasm";

const BUNDLE = "https://raw.githack.com/Kitware/vtk-wasm/dist/latest/vtk-wasm32-emscripten.tar.gz";
const runtime = await loadAsync({ url: BUNDLE });
const session = runtime.createStandaloneSession();
const vtk = session.vtk;

const cone = vtk.vtkConeSource();
// ...build and render the scene using the same class names as C++ VTK
```

A plain `<script src="https://unpkg.com/@kitware/vtk-wasm/vtk-umd.js"></script>` tag works too, with no bundler required. [Source: Kitware/vtk-wasm README](https://github.com/Kitware/vtk-wasm) Note the API shape contrast with vtk.js's `newInstance()`/`get`/`set` factory pattern (above): vtk-wasm's `vtk.vtkConeSource()` mirrors C++ construction directly, because it *is* the C++ class, reached through Emscripten's generated bindings rather than a parallel JavaScript implementation. [Source](https://kitware.github.io/vtk-wasm/)

vtk-wasm's runtime distinguishes two session types. A **`vtkStandaloneSession`** lets the page create and manipulate VTK objects entirely client-side, as in the snippet above — there is no server. A **`vtkRemoteSession`** instead mirrors a server-owned pipeline: the server builds and holds the real `vtkRenderWindow` object graph, and the WASM client deserializes streamed state to reconstruct the equivalent objects locally, rather than constructing anything of its own. [Source](https://kitware.github.io/vtk-wasm/) The state that crosses the network is produced by **`vtkWasmSceneManager`**, which lets an application register any serializable `vtkObject` (a `vtkRenderWindow`, for instance) and extract a serializable object-tree for transfer — the same `vtkObjectManager` machinery, formalized in VTK 9.4, discussed in Section 2. This is a third point on the remote/local rendering spectrum introduced in the trame subsection below: trame's "remote rendering" streams a compressed image, its "local rendering" streams geometry into a *reimplemented* vtk.js pipeline, and vtk-wasm mirroring streams serialized VTK object state into the *same* compiled C++ pipeline running client-side — the closest of the three to bit-for-bit parity with the server. The Kitware trame widget **`trame-vtklocal`** packages this mode: a server-side Python process owns a normal VTK pipeline, and `trame-vtklocal` synchronizes it to a `vtkRemoteSession` in the browser so the identical compiled filters execute on the client GPU, which matters for applications with custom C++ filters that vtk.js has no JavaScript equivalent for. [Source: Kitware — VTK.wasm and its trame integration](https://www.kitware.com/vtk-wasm-and-its-trame-integration/)

Kitware's vtk-wasm demo gallery gives a sense of the ceiling this approach reaches: a Porsche CAD assembly with interactive picking, a procedurally generated terrain of 351k triangles, a "Starfighter" scene exercising glyphs and 3D widgets, volume rendering of a 531k-voxel dataset, and a stress test rendering a thousand independent actors — all running the genuine VTK pipeline inside a browser tab via WebGL or WebGPU. [Source](https://kitware.github.io/vtk-wasm/) The trade-off against vtk.js is the one implied by shipping a compiled C++ runtime rather than hand-picked ES modules: the WASM binary carries the weight of whichever VTK modules were linked in, against which vtk.js's per-widget JavaScript imports are lighter but reimplemented and therefore incomplete relative to the C++ filter library.

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

### PyVista — Pythonic VTK Wrapper

[PyVista](https://pyvista.org/) is a 3D plotting and mesh-analysis library that wraps VTK in a "streamlined interface" — the project most widely reached for when a script needs VTK's data model and filters without VTK's C++-mirroring verbosity. [Source](https://docs.pyvista.org/getting-started/why) The wrapping is by direct **subclassing**, not composition: `pyvista.PolyData` *is* a `vtkPolyData` (and likewise `UnstructuredGrid`, `ImageData`, `MultiBlock`, …), so a PyVista mesh can be passed anywhere a raw VTK filter expects one, and any VTK object can be lifted into PyVista's friendlier API with `pyvista.wrap()`:

```python
import vtk
import pyvista as pv

polygon_source = vtk.vtkRegularPolygonSource()
polygon_source.GeneratePolygonOff()
polygon_source.SetNumberOfSides(50)
polygon_source.SetRadius(5.0)
polygon_source.Update()

mesh = pv.wrap(polygon_source.GetOutput())   # same object, PyVista's PolyData subclass
mesh.plot(line_width=3, cpos="xy", color="k")
```

[Source](https://tutorial.pyvista.org/tutorial/06_vtk/a_1_transition_vtk.html) Common filters (smoothing, clipping, decimation, slicing) are exposed as chainable methods directly on the mesh object rather than as separately instantiated `vtkAlgorithm` objects wired to sources and mappers:

```python
import pyvista as pv

mesh = pv.Sphere()
mesh.smooth(n_iter=25).plot()          # equivalent to a vtkSmoothPolyDataFilter pipeline stage
```

[Source](https://docs.pyvista.org/user-guide/simple.html) For Jupyter, PyVista's `trame` backend (superseding the deprecated `ipyvtklink`) reuses the same VTK-Python/`wslink` machinery documented in the trame subsection above (Section 8), offering the identical `'server'` (image streaming), `'client'` (geometry streamed to a browser-side renderer), and hybrid `'trame'` modes as a Jupyter cell widget rather than a standalone web app. [Source](https://docs.pyvista.org/user-guide/jupyter/trame.html)

### Mayavi — Pythonic Scientific 3D Plotting

[Mayavi](https://github.com/enthought/mayavi) (Enthought) is the older of the two general-purpose Python/VTK wrapper projects, built around **TVTK** (Traited VTK) — a layer that wraps essentially every VTK class with [Traits](https://docs.enthought.com/traits/), giving each one a Pythonic feel, transparent NumPy array handling, and elementary pickling support, without altering the underlying VTK object model the way PyVista's subclassing does. [Source](https://tvtk.readthedocs.io/en/latest/README.html) On top of TVTK, the **`mlab`** module offers matplotlib-style one-line plotting for quick exploratory visualization:

```python
from numpy import pi, sin, cos, mgrid
dphi, dtheta = pi / 250.0, pi / 250.0
phi, theta = mgrid[0:pi + dphi * 1.5:dphi, 0:2 * pi + dtheta * 1.5:dtheta]
m0, m1, m2, m3, m4, m5, m6, m7 = 4, 3, 2, 3, 6, 2, 6, 4
r = sin(m0 * phi) ** m1 + cos(m2 * phi) ** m3 + sin(m4 * theta) ** m5 + cos(m6 * theta) ** m7
x, y, z = r * sin(phi) * cos(theta), r * cos(phi), r * sin(phi) * sin(theta)

from mayavi import mlab
mlab.mesh(x, y, z)
mlab.show()
```

[Source](https://docs.enthought.com/mayavi/mayavi/mlab.html) For explicit pipeline construction rather than the `mlab`-generated one, `mlab.pipeline` mirrors VTK's source-filter-mapper graph directly (e.g. `mlab.pipeline.array2d_source` → `warp_scalar` → `poly_data_normals` → `surface`), and the standalone `tvtk` package can be used independently of Mayavi's GUI wherever only the Traits-wrapped VTK classes are wanted. [Source](https://docs.enthought.com/mayavi/mayavi/mlab_pipeline.html) Mayavi requires VTK ≥ 9.0 and ships current wheels for Linux, but its community and release cadence are smaller than PyVista's, which has become the more commonly recommended default for new VTK-adjacent Python code; Mayavi remains relevant chiefly where its Traits-based UI framework (Qt panels, dialogs bound to Traited VTK properties) is itself the point.

### itkwidgets — Jupyter Widget for ITK and VTK Data

[itkwidgets](https://github.com/InsightSoftwareConsortium/itkwidgets) is a purpose-built Jupyter widget, maintained by the Insight Software Consortium (the ITK project), for interactive 2D/3D viewing of images, point sets, and geometry directly from a notebook cell — distinct from PyVista and Mayavi in that it targets *browser-based* rendering rather than wrapping the desktop VTK object model. Its viewer is "built on itk.js and vtk.js," [Source](https://github.com/InsightSoftwareConsortium/itkwidgets) i.e. the same ES6 vtk.js runtime documented in Section 9.1, reached from Python via the itk-wasm bridge covered in that section — a notebook cell effectively runs a small vtk.js/itk-wasm application, communicating with the Python kernel through the standard Jupyter widget protocol rather than streaming a rendered image or raw VTK render-window state:

```python
from itkwidgets import view
from vtkmodules.vtkIOImage import vtkMetaImageReader

reader = vtkMetaImageReader()
reader.SetFileName("brain.mhd")
reader.Update()

view(reader.GetOutput())   # interactive 3D volume rendering in Jupyter cell
```

itkwidgets correctly loads and displays every file type ITK supports, including anisotropic images, and accepts VTK datasets, ITK images, and plain NumPy arrays interchangeably as its `view()` argument. [Source](https://github.com/InsightSoftwareConsortium/itkwidgets/blob/main/README.md) Where PyVista's `trame` backend (above) and the older `ipyvtk-simple`/`panel` widgets embed a genuine VTK render window (server- or client-rendered) inside a notebook cell, itkwidgets instead embeds vtk.js — the two approaches trade the same server/client rendering choice described throughout this chapter's headless and browser sections, just entered from the Jupyter side rather than a standalone web app.

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

### VTK versus FOSS Alternatives — VisIt, Ascent/Alpine, and yt

VTK has few genuine open-source peers at its own layer — a general-purpose, GPU-accelerated dataset model plus rendering pipeline. Most tools that look like alternatives are built *on* VTK rather than competing with it; the closest independent competitors sit one layer up (the application) or address a narrower problem than VTK's full generality.

**VisIt (LLNL)** is the direct open-source competitor to *ParaView*, not to VTK — it uses "the Visualization Toolkit (VTK) library for its data model and many of its visualization algorithms," with VisIt's own engineering focused on parallelization for very large datasets, non-standard data models (AMR meshes, mixed-material zones), and a plugin architecture with well over a hundred database readers. It is BSD-licensed and developed at [github.com/visit-dav/visit](https://github.com/visit-dav/visit). [Source](https://visit-dav.github.io/visit-website/about/) Choosing between ParaView and VisIt is therefore mostly a choice between two mature, VTK-based *applications* with different plugin ecosystems and DOE-lab lineages (Kitware/DOE for ParaView, LLNL for VisIt), not a choice about the underlying visualization engine.

**Ascent / Alpine (LLNL, part of the Exascale Computing Project)** is a genuine architectural alternative for the in-situ use case Catalyst 2.0 covers (Section 8): rather than embedding VTK's full C++ class hierarchy, Ascent is built on **VTK-m** (Section 6) — the header-only, GPU-portable filter library already covered earlier in this chapter — deliberately avoiding an OpenGL dependency and minimizing the runtime footprint linked into a simulation binary. It has been demonstrated performing in-situ filtering and ray tracing across 16,384 GPUs on LLNL's Sierra cluster. [Source](https://www.ascent-dav.org/tutorial/2023_08_22_ascent_intro.pdf) Because Catalyst 2.0's Conduit Mesh Blueprint is the same in-memory description Ascent consumes, a simulation instrumented for Catalyst can typically swap in Ascent as the implementation loaded at runtime rather than needing two separate integrations — Ascent is a genuine competitor to *how* in-situ rendering gets done, but a complementary one at the API level. [Source](https://www.exascaleproject.org/highlight/alpine-zfp-addresses-analysis-visualization-and-data-reduction-needs-for-exascale-science-applications/)

**yt** is the one tool here with no VTK dependency at all: a pure Python/NumPy analysis and visualization toolkit built for astrophysical simulation data (originally) and now a broader set of volumetric, multi-resolution, and particle datasets. It trades VTK's general-purpose C++ dataset/filter architecture for a domain-specific, scriptable Python stack — well suited to analysis-heavy astrophysics workflows where the visualization is one step in a larger NumPy/Matplotlib-based pipeline, but without VTK's breadth of dataset types (Section 2), GPU volume rendering path (Section 4), or WebGPU/ANARI backends (Section 5). [Source](https://yt-project.org/about.html)

None of the three displaces VTK's role as the shared data-model-and-filter substrate underneath the Python scientific-visualization ecosystem: `PyVista` and `Mayavi` (above) — the two most widely used "friendlier front door" Python wrappers around VTK, offering Pythonic object APIs over the same `vtkPolyData`/`vtkImageData`/mapper classes documented in Section 2 — are themselves built on top of VTK rather than being alternatives to it, the same relationship `itkwidgets` (above) and 3D Slicer (above) have to VTK (and, for itkwidgets, to vtk.js — Section 9.1 — one level further down).

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

**Ch34 — ANGLE and Ch35 — Dawn and WebGPU**: vtk.js's two browser rendering backends (Section 9) run on top of these layers — `vtkOpenGLRenderWindow` through the browser's WebGL implementation (Ch34) and `vtkWebGPURenderWindow` through the browser's WebGPU implementation (Ch35, and Dawn specifically on Linux desktop Chromium, mirroring VTK's own C++ WebGPU backend in Section 5).

**Ch98 — WebAssembly and WebGPU as a Deployment Target**: itk-wasm, the WebAssembly-compiled ITK library that vtk.js relies on for DICOM and NIfTI decoding (Section 9.1), and vtk-wasm, VTK's own `VTK_WRAP_JAVASCRIPT` Emscripten build of the full C++ library (Section 9.2), are both instances of the WASM deployment patterns covered in Ch98.

**Ch203 — WebXR**: vtk.js's `RenderingWebXR` module (Section 9) exposes the WebXR Device API described in Ch203, letting browser-rendered volumes and isosurfaces be viewed in a VR or AR headset without a native SDK.

---

*References consulted for this chapter:*

- [VTK Repository](https://gitlab.kitware.com/vtk/vtk) — primary source
- [The Architecture of Open Source Applications — VTK](https://aosabook.org/en/v1/vtk.html)
- [VTK Coding Conventions](https://docs.vtk.org/en/latest/developers_guide/coding_conventions.html)
- [vtkusers mailing list — VTK C++ Exception handling?](https://vtk.org/pipermail/vtkusers/2011-July/068927.html)
- [VTK Build Settings — VTK_WRAP_JAVASCRIPT and wrapping options](https://docs.vtk.org/en/latest/build_instructions/build_settings.html)
- [Building VTK for WebAssembly with Emscripten](https://docs.vtk.org/en/latest/advanced/build_wasm_emscripten.html)
- [vtk-wasm architecture and demo gallery](https://kitware.github.io/vtk-wasm/)
- [Kitware/vtk-wasm Repository](https://github.com/Kitware/vtk-wasm)
- [Kitware — Introducing WebAssembly Support in VTK](https://www.kitware.com/introducing-webassembly-support-in-vtk/)
- [Kitware — VTK.wasm and its trame integration](https://www.kitware.com/vtk-wasm-and-its-trame-integration/)
- [ParaView Documentation — Remote and Parallel Visualization](https://docs.paraview.org/en/latest/ReferenceManual/parallelDataVisualization.html)
- [ParaView Documentation — Command Line Arguments](https://docs.paraview.org/en/latest/UsersGuide/commandLineArguments.html)
- [IceT Users' Guide and Reference](https://www.sandia.gov/app/uploads/sites/150/2021/10/IceTUsersGuide-2-0.pdf)
- [ParaView Documentation — Python & Batch Tutorial](https://docs.paraview.org/en/latest/Tutorials/ClassroomTutorials/pythonAndBatchParaViewAndPython.html)
- [ParaView/Python `simple.animation` module](https://www.paraview.org/paraview-docs/latest/python/paraview.simple.animation.html)
- [ParaView — Anatomy of a Catalyst Python Module (Version 2.0)](https://www.paraview.org/paraview-docs/latest/cxx/CatalystPythonScriptsV2.html)
- [ParaView Documentation — Catalyst Getting Started](https://docs.paraview.org/en/latest/Catalyst/getting_started.html)
- [About VisIt (visit-dav)](https://visit-dav.github.io/visit-website/about/)
- [visit-dav/visit Repository](https://github.com/visit-dav/visit)
- [Ascent Introduction Tutorial (2023)](https://www.ascent-dav.org/tutorial/2023_08_22_ascent_intro.pdf)
- [Alpine/ZFP — Exascale Computing Project Highlight](https://www.exascaleproject.org/highlight/alpine-zfp-addresses-analysis-visualization-and-data-reduction-needs-for-exascale-science-applications/)
- [The yt Project — About](https://yt-project.org/about.html)
- [PyVista — Why PyVista?](https://docs.pyvista.org/getting-started/why)
- [PyVista — Transitioning from VTK to PyVista](https://tutorial.pyvista.org/tutorial/06_vtk/a_1_transition_vtk.html)
- [PyVista — Basic API Usage](https://docs.pyvista.org/user-guide/simple.html)
- [PyVista — Trame Jupyter Backend](https://docs.pyvista.org/user-guide/jupyter/trame.html)
- [tvtk — An Introduction to Traited VTK](https://tvtk.readthedocs.io/en/latest/README.html)
- [Mayavi — mlab: Python Scripting for 3D Plotting](https://docs.enthought.com/mayavi/mayavi/mlab.html)
- [Mayavi — Assembling Pipelines with mlab](https://docs.enthought.com/mayavi/mayavi/mlab_pipeline.html)
- [enthought/mayavi Repository](https://github.com/enthought/mayavi)
- [InsightSoftwareConsortium/itkwidgets Repository and README](https://github.com/InsightSoftwareConsortium/itkwidgets/blob/main/README.md)
- [VTK 9.6 Release Notes](https://docs.vtk.org/en/latest/release_details/9.6.html)
- [VTK 9.4 Release Notes](https://docs.vtk.org/en/latest/release_details/9.4.html)
- [VTK 9.3 Release Notes](https://docs.vtk.org/en/latest/release_details/9.3.html)
- [VTK Build Settings](https://docs.vtk.org/en/latest/build_instructions/build_settings.html)
- [VTK-m Documentation](https://docs-m.vtk.org/latest/)
- [VTK-m Users Guide V.2.0](https://www.osti.gov/biblio/1959590)
- [VTKHDF File Format](https://docs.vtk.org/en/latest/vtk_file_formats/vtkhdf_file_format/index.html)
- [VTKHDF Format Specification](https://docs.vtk.org/en/latest/vtk_file_formats/vtkhdf_file_format/vtkhdf_specifications.html)
- [Kitware — VTK HDF Reader](https://www.kitware.com/vtk-hdf-reader/)
- [Kitware — VTKHDF File Format: 2025 Status Update](https://www.kitware.com/vtkhdf-file-format-2025-status-update/)
- [Kitware — How to Write Time-Dependent Data in VTKHDF Files](https://www.kitware.com/how-to-write-time-dependent-data-in-vtkhdf-files/)
- [Kokkos: The C++ Performance Portability Programming Ecosystem (OSTI)](https://www.osti.gov/servlets/purl/1457941)
- [Kokkos: Performance Portability for the Exascale Era — NASA Advanced Supercomputing](https://www.nas.nasa.gov/pubs/ams/2024/04-04-24.html)
- [kokkos/kokkos Repository](https://github.com/kokkos/kokkos)
- [ParaView Documentation](https://docs.paraview.org/en/latest/)
- [Catalyst In-Situ Documentation](https://catalyst-in-situ.readthedocs.io/en/latest/)
- [VTK ANARI Module](https://docs.vtk.org/en/latest/release_details/9.4/add-anari-rendering-capability.html)
- [ANARI Standard — Khronos](https://www.khronos.org/anari/)
- [vtk.js Repository](https://github.com/Kitware/vtk-js)
- [VTK.js Vanilla Getting-Started Guide](https://kitware.github.io/vtk-js/docs/vtk_vanilla.html)
- [VTK.js v35 Release Notes](https://www.kitware.com/vtk-js-v35-release/)
- [Introducing Physically Based Rendering to VTK.js WebGPU](https://www.kitware.com/introducing-physically-based-rendering-to-vtk-js-webgpu/)
- [itk-vtk-viewer Repository](https://github.com/Kitware/itk-vtk-viewer)
- [three.js VTKLoader Documentation](https://threejs.org/docs/pages/VTKLoader.html)
- [VTK Discourse — Integrating vtk.js into a Babylon.js GUI](https://discourse.vtk.org/t/integrating-vtk-js-capabilities-into-a-babylon-js-centric-gui/2670)
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
- A full server-free VTK pipeline running in the browser is no longer just a trajectory: vtk-wasm (Section 9.2) already compiles the C++ library itself via Emscripten with `VTK_WRAP_JAVASCRIPT`, demonstrated on datasets up to several hundred thousand triangles/voxels. What remains open is the ceiling for genuinely HPC-scale datasets — WASM linear-memory limits (mitigated but not eliminated by `VTK_WEBASSEMBLY_64_BIT`), single-tab GPU memory budgets, and the download cost of a compiled binary carrying the full linked module set — versus the smaller, hand-picked footprint of a vtk.js-only deployment.
- Deep learning-based upsampling and reconstruction filters (leveraging ONNX inference, added experimentally in VTK 9.6) may become first-class VTK pipeline stages, enabling AI-assisted volume rendering and super-resolution for clinical and simulation datasets.
- Convergence of the Catalyst in-situ API with streaming HPC data fabrics (ADIOS2, RDMA-based) and cloud-native object stores (S3-compatible) is a stated direction for post-Exascale workflows, where VTK acts as the on-node serialization and analysis layer rather than a batch post-processor.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
