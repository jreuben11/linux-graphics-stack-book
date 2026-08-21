# Chapter 211a: Robotics Simulation Engines — MuJoCo, Gazebo, and the Linux GPU Stack

> **Part**: Part VII-B — Multimedia and Simulation Frameworks
> **Audience**: Primarily systems and driver developers and graphics application developers evaluating what these simulators demand of the Linux graphics stack, plus AI/ML systems engineers interested in GPU-parallel RL training.
> **Status**: First draft — 2026-08-21

Robotics simulation on Linux has three architecturally distinct answers to "what renders the scene and what solves the physics," and this chapter covers two of them. NVIDIA's Isaac Sim answer — Omniverse Kit, USD, PhysX 5, and RTX path tracing — is documented in depth in Chapter 69 (the Omniverse/USD/Hydra/RTX substrate) and Chapter 211b (the Isaac-specific extension layer, Isaac Lab, and the GR00T model family). This chapter does not re-derive any of that; where Isaac Sim appears below, it is as a comparison point, cited back to those two chapters.

This chapter's own contribution is the two general-purpose, vendor-neutral simulators that predate and coexist with the NVIDIA stack: **MuJoCo**, DeepMind's Apache-2.0 physics engine built around a soft-constraint contact model and now available in both a classic CPU form and a JAX/XLA-compiled GPU-parallel form (MJX); and **Gazebo**, the ROS ecosystem's long-standing general-purpose simulator, which underwent a complete architectural rewrite from the monolithic "Gazebo Classic" to the modular `gz-*` library family (formerly branded "Ignition"). Both are used far outside any NVIDIA-specific pipeline — MuJoCo dominates locomotion and manipulation RL research, and Gazebo is the default simulator assumed by most ROS 2 tutorials, launch files, and CI pipelines. Both also intersect the Linux graphics stack directly: MuJoCo through EGL/OSMesa offscreen OpenGL contexts and MJX's XLA compilation target, Gazebo through Ogre-Next's OpenGL/Vulkan rendering backend. Where MuJoCo's own GPU-physics path (MJX-Warp / MuJoCo-Warp) converges with Isaac Lab's Newton integration, this chapter states that convergence and defers Newton's own licensing and support-status detail to Chapter 211b §8, which documents it from the Isaac side.

Every version number, license, and library name below was checked against a primary upstream source as of 2026-08-21; several claims that circulate in secondary summaries (Gazebo Classic's exact EOL date, the "MuJoCo has no ROS 2 integration" simplification, the shape of the MJX-JAX/MJX-Warp split) turn out to need correction, and those corrections are made in place with citations.

---

## Table of Contents

- [1. MuJoCo: Origin, Licensing, and Model Format](#1-mujoco-origin-licensing-and-model-format)
  - [1.1 From Roboti LLC to DeepMind: Apache-2.0 Open-Sourcing](#11-from-roboti-llc-to-deepmind-apache-20-open-sourcing)
  - [1.2 MJCF versus URDF](#12-mjcf-versus-urdf)
  - [1.3 The Soft-Constraint Contact Model](#13-the-soft-constraint-contact-model)
- [2. MuJoCo Rendering and GPU Paths on Linux](#2-mujoco-rendering-and-gpu-paths-on-linux)
  - [2.1 Native `mjr_render` and the OpenGL Visualizer](#21-native-mjr_render-and-the-opengl-visualizer)
  - [2.2 Headless Contexts: EGL and OSMesa](#22-headless-contexts-egl-and-osmesa)
  - [2.3 MJX: JAX/XLA-Compiled Parallel Environments](#23-mjx-jaxxla-compiled-parallel-environments)
  - [2.4 The MJX-JAX versus MJX-Warp Split, and Newton as Convergence Point](#24-the-mjx-jax-versus-mjx-warp-split-and-newton-as-convergence-point)
- [3. Gazebo: From Classic to Gazebo Sim](#3-gazebo-from-classic-to-gazebo-sim)
  - [3.1 Gazebo Classic's End-of-Life and the Ignition Rename](#31-gazebo-classics-end-of-life-and-the-ignition-rename)
  - [3.2 The `gz-*` Library Split](#32-the-gz--library-split)
  - [3.3 SDFormat versus URDF and `gz sdf` Conversion](#33-sdformat-versus-urdf-and-gz-sdf-conversion)
- [4. Gazebo's Pluggable Physics Backends](#4-gazebos-pluggable-physics-backends)
- [5. Gazebo Rendering and Sensor Simulation](#5-gazebo-rendering-and-sensor-simulation)
  - [5.1 `gz-rendering`'s Ogre-Next Backend](#51-gz-renderings-ogre-next-backend)
  - [5.2 `gz-sensors` and the Synthetic `sensor_msgs` Pipeline](#52-gz-sensors-and-the-synthetic-sensor_msgs-pipeline)
- [6. ROS 2 Integration for Both Simulators](#6-ros-2-integration-for-both-simulators)
  - [6.1 `ros_gz_bridge` and `ros_gz_sim`: Tight First-Party Coupling](#61-ros_gz_bridge-and-ros_gz_sim-tight-first-party-coupling)
  - [6.2 MuJoCo's Community and ROS-Ecosystem Integrations](#62-mujocos-community-and-ros-ecosystem-integrations)
- [7. Headless and CI Operation](#7-headless-and-ci-operation)
- [8. Sim-to-Real: Domain Randomization as a GPU-Parallelism Payoff](#8-sim-to-real-domain-randomization-as-a-gpu-parallelism-payoff)
- [9. Comparison Table: MuJoCo, Gazebo Sim, and Isaac Sim](#9-comparison-table-mujoco-gazebo-sim-and-isaac-sim)
- [Integrations](#integrations)
- [References](#references)

---

## 1. MuJoCo: Origin, Licensing, and Model Format

### 1.1 From Roboti LLC to DeepMind: Apache-2.0 Open-Sourcing

MuJoCo — Multi-Joint dynamics with Contact — began as a small commercial physics engine distributed under a paid license, aimed at robotics and biomechanics research that needed fast, accurate simulation of articulated structures interacting with their environment [Source](https://github.com/google-deepmind/mujoco/blob/main/README.md). In October 2021, Google DeepMind acquired MuJoCo and committed to making it freely available; the engine was open-sourced under the Apache License 2.0 in May 2022, with the source code moving to `github.com/google-deepmind/mujoco` and copyright held by DeepMind Technologies Limited [Source](https://github.com/google-deepmind/mujoco/blob/main/LICENSE). The documentation itself carries a separate CC BY 4.0 license, the same split this book's own chapters use for code versus prose [Source](https://github.com/google-deepmind/mujoco/blob/main/doc/LICENSE). The project is now maintained institutionally by Google DeepMind as an active repository, not a closed-source legacy artifact with a permissive wrapper — a materially different position from Isaac Sim's Apache-2.0-source-over-proprietary-Kit-runtime split documented in Chapter 211b §2.4.

Every model MuJoCo simulates is described in **MJCF** (MuJoCo XML format), an XML dialect distinct from and older than URDF as a MuJoCo-native format, though URDF import is also supported via an internal converter.

### 1.2 MJCF versus URDF

The structural difference between the two formats is not cosmetic. URDF describes a robot as a flat collection of `<link>` elements connected by `<joint>` elements that encode parent-child relationships as separate graph edges. MJCF instead nests bodies directly in the XML tree: a child body is a literal XML child of its parent body, under a single top-level `<worldbody>` element [Source](https://mujoco.readthedocs.io/en/stable/modeling.html). This is MJCF's `hello.xml`, the model used in MuJoCo's own overview documentation:

```xml
<mujoco>
  <worldbody>
    <light diffuse=".5 .5 .5" pos="0 0 3" dir="0 0 -1"/>
    <geom type="plane" size="1 1 0.1" rgba=".9 0 0 1"/>
    <body pos="0 0 1">
      <joint type="free"/>
      <geom type="box" size=".1 .2 .3" rgba="0 .9 0 1"/>
    </body>
  </worldbody>
</mujoco>
```

[Source](https://mujoco.readthedocs.io/en/stable/overview.html)

A `<body>` in MJCF carries its own `<joint>` and `<geom>` children directly — there is no separate `<link>`/`<joint>` split, and no `<visual>`/`<collision>` split either: a `geom` is simultaneously the visual and (by default) the collision primitive, distinguished by `contype`/`conaffinity` and rendering group attributes rather than by separate elements. MJCF also has first-class support for tendons, actuators, and equality constraints as top-level sibling sections (`<tendon>`, `<actuator>`, `<equality>`), and an extensive default-inheritance mechanism — comparable to CSS cascading — that lets a model set very few attributes explicitly per element [Source](https://mujoco.readthedocs.io/en/stable/modeling.html).

URDF was designed primarily to describe kinematics for motion planning and visualization, not full-fidelity physics; when MuJoCo loads a URDF file, it takes only the collision geometry and ignores the visual mesh, and formats common in URDF assets (COLLADA `.dae` meshes, in particular) are not supported and must be converted [Source](https://github.com/google-deepmind/mujoco_playground/discussions/101). The practical rule for anyone porting a ROS-ecosystem robot description into MuJoCo: treat URDF import as a conversion that needs auditing, not a drop-in load — actuator gains, joint damping and friction, and contact parameters that URDF has no vocabulary for at all must be authored directly in MJCF afterward.

### 1.3 The Soft-Constraint Contact Model

MuJoCo's defining departure from most other rigid-body engines — including Gazebo's DART/Bullet backends (§4) and PhysX's TGS solver underneath Isaac Sim (Ch211b §7.1) — is its contact model. The conventional formulation treats contact and friction as a linear or nonlinear complementarity problem (LCP/NCP): a hard constraint that penetrating geometry is inadmissible, solved by iterative or pivoting methods that are, in the general case, NP-hard. MuJoCo instead formulates contact as a **convex optimization problem**: constraint violation is penalized rather than forbidden outright, with a normal force that grows with penetration depth and closing velocity under user-tunable stiffness and damping parameters. The model is inherently soft in the sense that pushing harder against a constraint always produces a larger, well-defined reaction force, which means MuJoCo's inverse dynamics — recovering the forces that produced an observed motion — are uniquely defined in a way that is not guaranteed for hard-constraint LCP solvers [Source](https://mujoco.readthedocs.io/en/stable/computation/index.html).

This buys two things directly relevant to the GPU-parallel training use case in §8: the convex formulation is fast and has a guaranteed solution rather than a search that can fail to converge, and its differentiability (in the MJX-JAX path, §2.3) falls out naturally from optimizing a smooth convex objective rather than differentiating through a combinatorial solver. The cost is physical accuracy at the margins: soft contacts can visibly interpenetrate under high stiffness settings if the timestep is too large relative to the penalty stiffness, a tuning trade-off that does not exist in the same form for a hard-constraint solver. Chapter 210 (part-29) §50.3 covers the alternative — a GPU-parallel sequential-impulse solver with graph-coloring-based parallel constraint resolution, the architecture underneath most GPU rigid-body engines including Gazebo's Bullet-Featherstone backend — as the direct algorithmic contrast.

---

## 2. MuJoCo Rendering and GPU Paths on Linux

### 2.1 Native `mjr_render` and the OpenGL Visualizer

MuJoCo's built-in interactive viewer is `simulate`, rendered through OpenGL via the library's own `mjr_*` rendering API. `mjr_render` draws a fully composed scene — geometry, lighting, shadows, and any overlays — into whichever framebuffer is currently active, selected by the `mjtFramebuffer` enum's two values, `mjFB_WINDOW` and `mjFB_OFFSCREEN` [Source](https://mujoco.readthedocs.io/en/stable/programming/visualization.html). Notably, MuJoCo does not link against OpenGL at build time; OpenGL entry points are resolved dynamically via GLAD the first time `mjr_makeContext` is called, so the core library has no OpenGL dependency at all until rendering is actually requested — a library that only needs physics, never rendering, never touches the graphics stack.

### 2.2 Headless Contexts: EGL and OSMesa

For offscreen and CI use, MuJoCo on Linux supports three OpenGL context backends, selected by the `MUJOCO_GL` environment variable: `glfw` (an on-screen window, unusable headless), `osmesa` (Mesa's pure-software OpenGL rasterizer, no GPU required), and `egl` (hardware-accelerated headless rendering through a GPU-bound EGL context) [Source](https://mujoco.readthedocs.io/en/2.2.2/programming.html). This is the same GLX/EGL/GBM/OSMesa landscape covered generally in Chapter 107; MuJoCo's binding to it is a single environment variable read at import time:

```python
import os
os.environ["MUJOCO_GL"] = "egl"   # or "osmesa" for pure software
import mujoco

model = mujoco.MjModel.from_xml_path("robot.xml")
renderer = mujoco.Renderer(model)
data = mujoco.MjData(model)
mujoco.mj_forward(model, data)
renderer.update_scene(data)
pixels = renderer.render()
```

On a multi-GPU node, `MUJOCO_EGL_DEVICE_ID` selects which GPU's EGL device backs the context, the same per-process GPU-pinning concern Chapter 107 raises for any EGL-surfaceless workload sharing a node [Source](https://vedder.io/misc/mujoco_py.html). EGL is the documented recommendation whenever a GPU is present; OSMesa remains the fallback for GPU-less build and CI hosts, at a substantial throughput cost since every pixel is rasterized on the CPU.

### 2.3 MJX: JAX/XLA-Compiled Parallel Environments

Everything in §2.1–2.2 describes classic MuJoCo: a CPU-bound C library, single environment per process, stepped in a Python or C++ loop. **MJX** (MuJoCo XLA) is architecturally a different thing — a JAX reimplementation of MuJoCo's model and data structures and stepping functions, compiled by XLA to run "on all compute hardware supported by the XLA compiler" [Source](https://mujoco.readthedocs.io/en/stable/mjx.html). Because `mjx.Model` and `mjx.Data` are ordinary JAX pytrees, adding a leading batch dimension and mapping the step function over it with `jax.vmap` turns a single-environment simulator into thousands of environments stepping in lockstep on one device, with no per-environment Python loop and no host round-trip between steps — the same tensor-batching principle Chapter 211b §7.1 describes for `omni.physics.tensors`, reached from JAX's functional-transform side rather than PhysX's imperative tensor-view side.

MJX functions are not JIT-compiled implicitly; the caller wraps the step (or an outer training-loop function that calls it) in `jax.jit` explicitly, giving the caller control over compilation boundaries [Source](https://mujoco.readthedocs.io/en/stable/mjx.html). Published single-humanoid throughput figures illustrate the payoff and its limits: an Apple M3 Max running classic CPU MuJoCo reaches roughly 650K steps/second and a 64-core AMD Threadripper 3995WX reaches roughly 1.8M steps/second (both single-environment, high per-step clock speed); MJX-JAX on an Nvidia A100 at batch size 8,192 reaches roughly 950K steps/second, and on an 8-chip TPU v5 at batch size 16,384 reaches roughly 2.7M steps/second [Source](https://mujoco.readthedocs.io/en/stable/mjx.html) — the GPU/TPU numbers are aggregate across the whole batch, not per-environment, which is the entire point of batched simulation for RL. The same source notes that as the number of contacts per scene grows, MJX-JAX throughput degrades faster than classic MuJoCo's, a limitation directly relevant to §2.4's MJX-Warp variant.

### 2.4 The MJX-JAX versus MJX-Warp Split, and Newton as Convergence Point

MJX ships in two variants with materially different capabilities, not two builds of the same thing:

- **MJX-JAX** is the pure-JAX implementation described above. It runs on "Nvidia and AMD GPUs, Apple Silicon, and Google Cloud TPUs" — genuine cross-vendor hardware portability inherited from XLA — and differentiability is mostly supported: gradients flow through the physics step, which is the property that makes MJX attractive for policy-gradient methods that want analytic gradients rather than only sampled ones [Source](https://mujoco.readthedocs.io/en/stable/mjx.html).
- **MJX-Warp** is built on NVIDIA Warp instead of JAX, and is optimized specifically for NVIDIA hardware. It exists because MJX-JAX has a known performance bottleneck around contacts and constraints in contact-rich scenes; MJX-Warp resolves that bottleneck and scales better as contact count grows, at the cost of losing differentiability — "MJX-Warp does not support automatic differentiation and has no immediate plans to support auto-diff" [Source](https://mujoco.readthedocs.io/en/stable/mjx.html). MJX-Warp also adds a hardware-accelerated batch renderer, generating RGB and depth pixel observations across many parallel environments in one call — a capability MJX-JAX does not have at all [Source](https://mujoco.readthedocs.io/en/stable/mjx.html).

The same Warp-based engine also exists as a standalone package, **MuJoCo Warp** (`mujoco_warp`, sometimes written MJWarp), developed as a joint effort between Google DeepMind and NVIDIA and living in its own repository rather than inside `google-deepmind/mujoco` [Source](https://github.com/google-deepmind/mujoco_warp). This is where MuJoCo's GPU-physics path and NVIDIA's converge: MuJoCo Warp is the **primary solver backend for Newton**, the Linux Foundation physics project on top of NVIDIA Warp that Chapter 211b §8 documents from the Isaac Lab side [Source](https://github.com/newton-physics/newton). Concretely — Newton generalizes and is intended to eventually supersede Warp's deprecated `warp.sim` module, and its primary solver is MuJoCo Warp, with an XPBD solver also available. Newton is therefore not a third, independent GPU-physics effort alongside MJX-Warp and Isaac Lab's PhysX path; it is the point where MJX's own Warp-based variant and NVIDIA's robot-learning stack meet on the same solver code. Newton's own licensing, its runtime-selectable "experimental" status inside Isaac Sim 6.0 and Isaac Lab 3.0 Beta, and the "no official support commitment" language in the Isaac Lab README are documented in Chapter 211b §8 and are not repeated here — this chapter's contribution is stating that MuJoCo Warp, reached via MJX-Warp, is the backend Newton actually runs.

> **Note: needs verification.** No primary source pins a specific calendar date for when the MJX-JAX/MJX-Warp split was introduced, or a version number at which MuJoCo Warp became Newton's default solver as opposed to one of several. The architectural relationship above — MuJoCo Warp as Newton's primary backend — is confirmed by Newton's own repository description; the release history of that pairing is not.

---

## 3. Gazebo: From Classic to Gazebo Sim

### 3.1 Gazebo Classic's End-of-Life and the Ignition Rename

"Gazebo" now refers to two lineages that share a name and a spiritual mission but essentially no code. **Gazebo Classic** (versions 1 through 11) was the original monolithic simulator, tightly coupled to ROS 1 and later ROS 2 via `gazebo_ros_pkgs`. Gazebo 11, its final release, shipped in January 2020 as a five-year Long Term Support release and reached end-of-life on 2025-01-31, along with every earlier Classic version — an Open Robotics announcement stated plainly that Gazebo Classic is "officially end of life" as of that date [Source](https://community.gazebosim.org/t/gazebo-classic-11-has-reached-end-of-life/3424).

Its replacement began life under a separate brand, **Ignition**, then was renamed to simply **Gazebo** (informally "modern Gazebo" or "Gazebo Sim" to distinguish it in conversation) to consolidate on the more recognizable name once Classic was retired [Source](https://community.gazebosim.org/t/gazebo-classic-11-has-reached-end-of-life/3424). Gazebo's modern releases follow a Ubuntu-style codename and cadence rather than a single incrementing version number: **Harmonic** is the current LTS release, supported from 2023-09 to 2029-05; **Ionic** (2024-09 to 2026-12) and **Jetty** (2025-09 to 2031-05) are the subsequent non-LTS and current releases respectively [Source](https://gazebosim.org/docs/latest/releases/). Anyone reading a tutorial or Stack Overflow answer that imports `gazebo_ros` or refers to `.world` files loaded by `gzserver`/`gzclient` is reading Classic-era material — those binaries and that ROS integration package do not exist in modern Gazebo.

### 3.2 The `gz-*` Library Split

Where Classic was a single large codebase, modern Gazebo is assembled from a family of independently versioned, Apache-2.0-licensed libraries under the `gazebosim` GitHub organization, each with its own repository, release cadence, and API-version suffix (e.g. `gz-sim9`, `gz-physics8`) [Source](https://github.com/gazebosim/gz-sim/blob/main/README.md):

| Library | Role |
|---|---|
| `gz-sim` | The simulation runtime — entity-component-system world state, plugin loading, the `gz sim` CLI |
| `gz-physics` | Abstraction over pluggable physics engines (§4) |
| `gz-rendering` | Abstraction over pluggable rendering engines (§5.1) |
| `gz-transport` | Pub/sub messaging between simulator processes and clients, analogous in role to ROS 2's DDS layer but a separate transport |
| `gz-sensors` | Sensor models (camera, LiDAR, IMU, etc.) consuming `gz-rendering` and `gz-physics` output |

`gz-sim` depends on the others rather than reimplementing their concerns, and each library can in principle be reused outside a full Gazebo install — `gz-transport` and `gz-physics` in particular are used by other projects independently of `gz-sim`.

### 3.3 SDFormat versus URDF and `gz sdf` Conversion

Modern Gazebo's native model and world description language is **SDFormat** (Simulation Description Format, `.sdf`), maintained as its own library (`sdformat`) with its own versioned specification independent of any particular Gazebo release [Source](https://gazebosim.org/libs/sdformat/). SDFormat is a strict superset of what URDF can express in one important structural sense: an SDF file can describe an entire `<world>` containing multiple `<model>` elements, sensors, lights, and physics configuration in one document, where a URDF file is scoped to describing exactly one robot [Source](https://github.com/osrf/gazebo/issues/2289).

Both Gazebo Classic and modern Gazebo accept URDF input, but neither simulates URDF directly — a URDF file loaded by Gazebo is converted to SDFormat internally before the simulator ever sees it, usually invisibly to the user [Source](https://github.com/gazebosim/sdf_tutorials/blob/master/urdf/sdf_extensions.md). That conversion can be run explicitly with the `gz sdf` command-line tool:

```bash
gz sdf -p robot.urdf
```

which prints the resulting SDFormat XML, useful for inspecting exactly what a URDF import produced before trusting it in simulation. Because URDF has no vocabulary for Gazebo-specific concepts — sensor plugins, physics-engine plugin parameters, an SDF world's multiple models — URDF files destined for Gazebo commonly embed a `<gazebo>` extension tag that the URDF→SDF converter recognizes and folds into the generated SDF output, letting a single URDF source file carry both ROS-facing kinematic description and Gazebo-facing simulation configuration [Source](https://sdformat.org/tutorials/specification/sdformat_urdf_extensions/1.6/).

---

## 4. Gazebo's Pluggable Physics Backends

Where MuJoCo (§1.3) ships one fixed solver and Isaac Sim ships PhysX 5 as its primary path (Newton as an experimental alternative, Ch211b §8), Gazebo's `gz-physics` library is explicitly built around a plugin interface with multiple interchangeable backend engines, selectable per world without recompiling Gazebo itself [Source](https://gazebosim.org/libs/physics/):

- **DART** (Dynamic Animation and Robotics Toolkit) is the reference and default implementation — the physics engine gz-physics' own plugin interface is developed and tested against first [Source](https://deepwiki.com/gazebosim/gz-physics/2-physics-engines).
- **Bullet**, in two forms: a preliminary rigid-body plugin (`gz-physics-bullet-plugin`) and a Featherstone-articulated-body variant (`gz-physics-bullet-featherstone-plugin`) that models multi-link robots as reduced-coordinate articulations rather than chains of six-DOF rigid bodies connected by constraints — the same Featherstone formulation Chapter 210 (part-29) §50.4 covers algorithmically.
- **TPE**, the Trivial Physics Engine — a deliberately lightweight custom engine focused on fast kinematics for large environments where full contact-dynamics fidelity is not the point (e.g. large-scale multi-robot traffic or crowd scenarios where collision response detail matters less than raw entity count).

The engine is selected in the world's SDF file by naming the plugin's shared library under the `Physics` system plugin:

```xml
<plugin filename="gz-sim-physics-system" name="gz::sim::systems::Physics">
  <engine>
    <filename>gz-physics-bullet-featherstone-plugin</filename>
  </engine>
</plugin>
```

or overridden from the command line without editing the world file at all:

```bash
gz sim --physics-engine gz-physics-bullet-featherstone-plugin shapes.sdf
```

[Source](https://github.com/gazebosim/gz-sim/blob/gz-sim9/tutorials/physics.md)

This pluggability is a genuine architectural difference from both other simulators in this chapter: MuJoCo's soft-constraint solver and MJX are inseparable from the engine itself, and Isaac Sim's engine choice (PhysX versus the experimental Newton path) is a build/runtime configuration of one application rather than a hot-swappable shared library selected per world file.

---

## 5. Gazebo Rendering and Sensor Simulation

### 5.1 `gz-rendering`'s Ogre-Next Backend

`gz-rendering` is itself an abstraction layer — a C++ library offering a unified scene-graph API over multiple concrete rendering engine plugins — but in practice the engine that matters for current Gazebo releases is **Ogre-Next** (Ogre 2.x, referred to in the codebase and documentation as `ogre2`), which itself targets multiple graphics APIs underneath: OpenGL, Vulkan, and Metal are all supported Ogre-Next render systems, with Vulkan support as an active, ongoing integration effort rather than a settled default across all platforms [Source](https://github.com/gazebosim/gz-rendering). A legacy `ogre` (Ogre 1.x) plugin and an `optix` ray-tracing plugin also exist in the `gz-rendering` tree, but `ogre2` is the engine current Gazebo documentation, tutorials, and default installs assume. This is architecturally distinct from Isaac Sim's fixed RTX/Hydra path-tracing pipeline (Ch211b §1.1, §4.1): Gazebo's synthetic camera and LiDAR sensors are rasterized by a traditional real-time renderer, not ray-traced, which is a meaningfully lower per-frame cost at the expense of photorealistic global illumination and physically based ray-traced sensor returns.

### 5.2 `gz-sensors` and the Synthetic `sensor_msgs` Pipeline

`gz-sensors` implements the sensor models — camera, depth camera, GPU LiDAR, IMU, contact, force-torque, and others — as plugins that consume `gz-rendering`'s scene for anything that needs rasterized pixels (camera, LiDAR-via-rendering) and `gz-physics`'s state for anything that is a pure physics readout (IMU, contact, force-torque), mirroring the same two-pipeline split Chapter 211b §4 documents for Isaac Sim's RTX-versus-physics sensors, but built on a rasterizer rather than a ray tracer for the rendering half. Sensor output inside the simulator is published on `gz-transport` topics carrying Gazebo's own protobuf message types; getting that data into ROS 2's `sensor_msgs` types — the same `Image`, `PointCloud2`, and `Imu` messages Chapter 211 documents as the canonical ROS 2 perception-pipeline vocabulary — is the job of the ROS 2 bridge covered next.

---

## 6. ROS 2 Integration for Both Simulators

### 6.1 `ros_gz_bridge` and `ros_gz_sim`: Tight First-Party Coupling

Gazebo's ROS 2 integration lives in the `gazebosim/ros_gz` repository, split into `ros_gz_bridge` (a bidirectional network bridge translating between `gz-transport` and ROS 2 messages for a configurable, explicit list of supported type pairs) and `ros_gz_sim` (launch files and convenience executables for starting Gazebo alongside a ROS 2 graph) [Source](https://github.com/gazebosim/ros_gz). Topic mappings are declared in YAML, naming the ROS topic, the Gazebo topic, both message types, and a direction:

```yaml
- ros_topic_name: "camera/image"
  gz_topic_name: "camera"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msgs.Image"
  direction: GZ_TO_ROS
```

run with:

```bash
ros2 run ros_gz_bridge parameter_bridge --ros-args -p config_file:=bridge.yaml
```

[Source](https://github.com/gazebosim/ros_gz/blob/ros2/ros_gz_bridge/README.md)

`ros_gz` lives in the same `gazebosim` GitHub organization as the simulator itself and is documented directly on `gazebosim.org` as the canonical ROS 2 integration path [Source](https://gazebosim.org/docs/latest/ros2_integration/) — a first-party relationship in the sense that matters for a reader deciding what to depend on: one organization ships both the simulator and its own ROS 2 bridge, versioned and released against specific Gazebo/ROS distro pairings (e.g. Gazebo Ionic with ROS 2 Kilted via vendor packages) [Source](https://gazebosim.org/docs/latest/releases/).

### 6.2 MuJoCo's Community and ROS-Ecosystem Integrations

No ROS 2 bridge ships inside `google-deepmind/mujoco`. What exists instead is a small landscape of separately maintained integration packages, closer in spirit to community middleware than to a vendor-shipped bridge:

- **`mujoco_ros2_control`**, maintained under the `ros-controls` GitHub organization — the same group that maintains the `ros2_control` framework itself — provides a `ros2_control` hardware-interface plugin wrapping MuJoCo's simulation loop, so a robot described in MJCF (or converted from URDF) can be driven by the standard ROS 2 control stack's controllers exactly as a real-hardware interface would be [Source](https://github.com/ros-controls/mujoco_ros2_control). This is the closest MuJoCo currently has to an "official" ROS 2 story, in the sense that `ros-controls` is itself a recognized ROS working group — but it is still a separate repository from MuJoCo's own, maintained on its own release cadence, released for ROS 2 Rolling rather than tracking each MuJoCo release directly.
- Independent community packages such as `Woolfrey/mujoco_ros2` provide lighter-weight, narrower-scope MuJoCo-to-ROS-2 communication without going through `ros2_control` at all [Source](https://github.com/Woolfrey/mujoco_ros2).

The practical contrast with §6.1: Gazebo's bridge is a single canonical, organizationally-endorsed integration documented on the simulator's own website; MuJoCo's ROS 2 story is fragmented across independently maintained packages with different scopes, release cadences, and maintainers, none of them the MuJoCo project itself. Both, ultimately, feed the same downstream consumers — `sensor_msgs`, `nav_msgs/Odometry`, and the TF2 transform tree that Chapter 211 documents as the ROS 2 perception pipeline's common vocabulary.

---

## 7. Headless and CI Operation

Both simulators support running without a display, but through different mechanisms reflecting their different architectures. Gazebo's `gz sim` binary takes a `-s` (server-only) flag that starts the physics/sensor server without the Qt-based GUI client process at all — not merely hiding a window, but never starting the process that would create one:

```bash
gz sim -s -r shapes.sdf
```

For nodes that need rendered sensor output (camera, LiDAR) without a display server present, Gazebo additionally exposes headless rendering explicitly, since sensor rendering still needs a GPU-bound EGL context even with the GUI client absent [Source](https://gazebosim.org/api/sim/9/headless_rendering.html). This is a server/client split at the process level: `gz sim -s` never touches the display at all, and a second, optional GUI process can attach to the running server over `gz-transport` from a different terminal or machine if visualization is later needed.

MuJoCo's headless story is the `MUJOCO_GL=egl`/`osmesa` environment-variable selection from §2.2 — a single process choosing its OpenGL backend at import time rather than a server/client split, since classic MuJoCo has no separate GUI process to omit in the first place. MJX (§2.3) sidesteps the question further for proprioception-only training: because MJX's physics step never touches OpenGL at all, a training job that doesn't need pixel observations has nothing to make headless — the render/no-render decision only matters at all once MJX-Warp's batch renderer (§2.4) is in the loop.

Both patterns are instances of the general offscreen-rendering technique Chapter 107 documents — GLX for on-screen X11, EGL for hardware-accelerated headless, OSMesa/llvmpipe for software fallback, GBM as the underlying buffer-allocation path for EGL surfaceless rendering — applied by two different applications in two different process architectures, and are the same pattern Chapter 190 and Chapter 212 apply to VTK and Open3D respectively.

---

## 8. Sim-to-Real: Domain Randomization as a GPU-Parallelism Payoff

Domain randomization — training a policy across many randomized variations of physical parameters (mass, friction, actuator gain, sensor noise, visual appearance) so that no single simulated instance is exploitable and the policy generalizes to the unmodeled reality gap — is not new to GPU-parallel simulation. What MJX changes is the unit cost of getting more randomized instances: because `mjx.Model`'s fields are ordinary JAX arrays, adding a batch dimension to a subset of them is a natural, direct way to express per-environment domain randomization, distinct from batching `mjx.Data` purely for RL throughput on an otherwise-identical model [Source](https://mujoco.readthedocs.io/en/stable/mjx.html). A training run can therefore vary friction coefficients, link masses, or actuator gains across the same batch dimension used for parallel rollout, at no additional per-step cost beyond the batch size already paid for throughput — domain randomization becomes close to free once the batched-simulation infrastructure exists at all, rather than requiring a separate randomization pass.

This is architecturally the same payoff Chapter 211b §7.2 describes for Isaac Lab's `replicate_physics=True` environment cloning — GPU-resident parallelism turning what used to be a serial cost (each randomized variant simulated one at a time) into a batch dimension — reached from a different substrate. The constraint is also symmetric: Isaac Lab's `replicate_physics=True` requires clones to be structurally identical, with variation expressed as parameter randomization through tensor views rather than as topology changes (Ch211b §7.2); MJX's batched `mjx.Model` has the same requirement, since a batch dimension on an array only works if every element of the batch shares that array's shape. Neither system's GPU-parallel path supports randomizing the *number* of links or the *topology* of the robot within a single batched run — only its continuous parameters.

For the vision-based case, MJX-Warp's batch renderer (§2.4) extends this to rendered domain randomization — material and lighting variation across a batch — narrowing, without closing, the gap with Isaac Sim's RTX-based domain randomization over materials and physically based lighting that Chapter 240 documents at scale for Isaac Lab. The renderer is a rasterizer producing batched RGB/depth observations, not a path tracer, so photorealism and ray-traced sensor physics remain Isaac Sim's differentiator (Ch211b §11) even as the environment-count and randomization-cost gap narrows.

---

## 9. Comparison Table: MuJoCo, Gazebo Sim, and Isaac Sim

| Dimension | MuJoCo (+ MJX) | Gazebo Sim (Harmonic/Ionic/Jetty) | Isaac Sim |
|---|---|---|---|
| Physics backend | Single fixed solver (classic MuJoCo C engine or MJX-JAX/MJX-Warp reimplementations) | Pluggable via `gz-physics`: DART (default), Bullet, Bullet-Featherstone, TPE | PhysX 5 (default); Newton experimental/runtime-selectable (Ch211b §8) |
| Contact model | Soft-constraint convex optimization (§1.3) | Backend-dependent: DART/Bullet use hard-constraint LCP-family solvers; TPE trades contact fidelity for kinematic throughput | PhysX TGS-family sequential-impulse solver (hard-constraint) |
| Native GPU parallelism | Yes, via MJX (JAX/XLA `vmap`, thousands of batched environments on one device, §2.3) | No native batched-environment mode; each `gz sim` process simulates one world | Yes, via `omni.physics.tensors` batched views over cloned USD stages (Ch211b §7.1–7.2) |
| Rendering backend | Native OpenGL viewer (`mjr_render`) + EGL/OSMesa headless contexts; MJX-Warp adds a batched rasterized renderer | `gz-rendering` / Ogre-Next (`ogre2`): OpenGL, Vulkan, Metal render systems, rasterized | RTX Hydra delegate: GPU ray/path-traced (Ch211b §1.1) |
| Model format | MJCF (native); URDF import via internal converter, collision-only, lossy (§1.2) | SDFormat (native); URDF accepted and converted to SDF internally, `<gazebo>` extension tag for round-tripping (§3.3) | USD, with the full Omniverse content pipeline (Ch211b §1.1) |
| ROS 2 coupling | Community/ROS-ecosystem packages (`ros_gz`-style bridges do not exist for MuJoCo; `mujoco_ros2_control` under `ros-controls`, independent community packages), no first-party MuJoCo-team bridge (§6.2) | First-party `ros_gz_bridge`/`ros_gz_sim`, same GitHub org as the simulator, documented on `gazebosim.org` (§6.1) | `isaacsim.ros2.bridge`/`isaacsim.ros2.core`, C++ against `rcl`, two supported distros (Humble, Jazzy) (Ch211b §5.1) |
| License | Apache-2.0 (code), CC BY 4.0 (docs) (§1.1) | Apache-2.0 across the `gz-*` library family (§3.2) | Apache-2.0 source over a proprietary Kit SDK runtime; bundled assets under separate terms (Ch211b §2.4) |

---

## Integrations

**Chapter 69 (Omniverse, USD, Hydra, and RTX)** and **Chapter 211b (Isaac Sim, Isaac Lab, and GR00T)** together document the third simulator in the field this chapter's comparison table places MuJoCo and Gazebo against. This chapter defers all Isaac Sim/PhysX/Omniverse/GR00T detail to those two chapters rather than re-deriving it; every Isaac Sim claim above is cited back to Ch211b by section number, and Newton's own licensing and support-status detail (§2.4 here) is Ch211b §8's, not restated.

**Chapter 107 (Headless Rendering)** is the general mechanism behind §7 of this chapter: EGL surfaceless contexts, GBM buffer allocation, and OSMesa/llvmpipe software fallback are documented there in depth; §2.2 and §7 here are MuJoCo's and Gazebo's specific bindings to that mechanism (`MUJOCO_GL` and `gz sim -s`/headless-rendering respectively).

**Chapter 210 — GPU Geometry Algorithms: Physics Simulation and Volumetric Methods (part-29-graphics-algorithms)** covers the algorithmic contrast to both simulators' contact solvers in depth: §50.3's sequential-impulse solver with graph-coloring-based parallel constraint resolution is the GPU-parallel hard-constraint architecture that MuJoCo's soft-constraint model (§1.3) departs from, and §50.4's Featherstone articulated-body algorithm is the reduced-coordinate formulation underneath Gazebo's Bullet-Featherstone plugin (§4).

**Chapter 211 (ROS 2 Multimodal Sensor and Perception Pipeline)** is where both simulators' sensor output ultimately lands: `ros_gz_bridge`'s translated `sensor_msgs`/`nav_msgs` topics (§6.1) and MuJoCo's community-bridge equivalents (§6.2) feed the same `sensor_msgs`, TF2 transform tree, and `image_transport`/`point_cloud_transport` machinery Chapter 211 documents in depth; this chapter does not repeat that message-type taxonomy.

**Chapter 240 (Cosmos, OSMO, and Omniverse Farm)** documents Isaac Lab's large-scale domain-randomization and synthetic-data pipeline as the equivalent NVIDIA-stack answer to §8's MJX-batched domain randomization — the same GPU-parallelism payoff, reached from RTX-rendered, PhysX-simulated USD scenes rather than from MJCF models batched through JAX/XLA.

---

## References

- [MuJoCo repository](https://github.com/google-deepmind/mujoco) — current source, maintained by Google DeepMind (§1.1)
- [MuJoCo `LICENSE`](https://github.com/google-deepmind/mujoco/blob/main/LICENSE) — Apache License 2.0, copyright DeepMind Technologies Limited (§1.1)
- [MuJoCo documentation `LICENSE`](https://github.com/google-deepmind/mujoco/blob/main/doc/LICENSE) — CC BY 4.0 for documentation (§1.1)
- [MuJoCo README](https://github.com/google-deepmind/mujoco/blob/main/README.md) — project description, MJCF, native OpenGL visualizer (§1.1, §2.1)
- [MuJoCo Modeling documentation](https://mujoco.readthedocs.io/en/stable/modeling.html) — MJCF body-hierarchy structure, default-inheritance mechanism (§1.2)
- [MuJoCo Overview documentation](https://mujoco.readthedocs.io/en/stable/overview.html) — `hello.xml` minimal MJCF example (§1.2)
- [MuJoCo Playground GitHub Discussion #101](https://github.com/google-deepmind/mujoco_playground/discussions/101) — URDF-to-MuJoCo import limitations (collision-only, unsupported mesh formats) (§1.2)
- [MuJoCo Computation documentation](https://mujoco.readthedocs.io/en/stable/computation/index.html) — the soft-constraint convex contact/friction formulation (§1.3)
- [MuJoCo Visualization programming documentation](https://mujoco.readthedocs.io/en/stable/programming/visualization.html) — `mjr_render`, `mjtFramebuffer` (`mjFB_WINDOW`/`mjFB_OFFSCREEN`) (§2.1)
- [MuJoCo Programming documentation (2.2.2)](https://mujoco.readthedocs.io/en/2.2.2/programming.html) — `MUJOCO_GL` backend selection (glfw/osmesa/egl) (§2.2)
- [MuJoCo-py EGL/OSMesa setup notes](https://vedder.io/misc/mujoco_py.html) — `MUJOCO_EGL_DEVICE_ID` multi-GPU selection (§2.2)
- [MuJoCo XLA (MJX) documentation](https://mujoco.readthedocs.io/en/stable/mjx.html) — MJX architecture, XLA hardware support, MJX-JAX/MJX-Warp feature split, differentiability status, batch renderer, throughput benchmarks, domain-randomization batch-dimension semantics (§2.3, §2.4, §8)
- [`mujoco_warp` (MuJoCo Warp / MJWarp) repository](https://github.com/google-deepmind/mujoco_warp) — GPU-optimized MuJoCo on NVIDIA Warp, joint DeepMind/NVIDIA effort (§2.4)
- [Newton physics engine repository](https://github.com/newton-physics/newton) — Linux Foundation project on NVIDIA Warp; MuJoCo Warp as primary solver backend (§2.4)
- [Gazebo Classic 11 end-of-life announcement](https://community.gazebosim.org/t/gazebo-classic-11-has-reached-end-of-life/3424) — 2025-01-31 EOL date, Ignition-to-Gazebo rename (§3.1)
- [Gazebo Releases documentation](https://gazebosim.org/docs/latest/releases/) — Harmonic/Ionic/Jetty support windows and ROS distro pairings (§3.1, §6.1)
- [`gz-sim` repository README](https://github.com/gazebosim/gz-sim/blob/main/README.md) — library role, Apache-2.0 license, dependency on `gz-physics`/`gz-rendering` (§3.2)
- [`sdformat` library documentation](https://gazebosim.org/libs/sdformat/) — SDFormat as Gazebo's native description language (§3.3)
- [`sdf_tutorials`: SDFormat extensions to URDF](https://github.com/gazebosim/sdf_tutorials/blob/master/urdf/sdf_extensions.md) — URDF-to-SDF conversion happening transparently on load (§3.3)
- [SDFormat.org: SDFormat extensions to URDF (`<gazebo>` tag)](https://sdformat.org/tutorials/specification/sdformat_urdf_extensions/1.6/) — the `<gazebo>` URDF extension tag mechanism (§3.3)
- [`gazebosim/gazebo-classic` issue #2289](https://github.com/osrf/gazebo/issues/2289) — SDF-can-describe-multiple-models vs. URDF-single-robot distinction (§3.3)
- [`gz-physics` DeepWiki: Physics Engines](https://deepwiki.com/gazebosim/gz-physics/2-physics-engines) — DART as reference/default implementation (§4)
- [Gazebo Physics library documentation](https://gazebosim.org/libs/physics/) — plugin architecture, DART/Bullet/Bullet-Featherstone/TPE roster (§4)
- [`gz-sim` physics engine tutorial (gz-sim9)](https://github.com/gazebosim/gz-sim/blob/gz-sim9/tutorials/physics.md) — SDF `<engine>` plugin selection and `--physics-engine` CLI flag (§4)
- [`gz-rendering` repository](https://github.com/gazebosim/gz-rendering) — Ogre-Next (`ogre2`) as the current default engine; OpenGL/Vulkan/Metal render systems; legacy `ogre`/`optix` plugins (§5.1)
- [`gazebosim/ros_gz` repository](https://github.com/gazebosim/ros_gz) — `ros_gz_bridge`/`ros_gz_sim` split and roles (§6.1)
- [`ros_gz_bridge` README (ros2 branch)](https://github.com/gazebosim/ros_gz/blob/ros2/ros_gz_bridge/README.md) — YAML topic-mapping configuration format and `parameter_bridge` invocation (§6.1)
- [Gazebo ROS 2 integration documentation](https://gazebosim.org/docs/latest/ros2_integration/) — `ros_gz` as the documented first-party ROS 2 path (§6.1)
- [`mujoco_ros2_control` repository](https://github.com/ros-controls/mujoco_ros2_control) — `ros2_control` hardware-interface plugin for MuJoCo, maintained under `ros-controls` (§6.2)
- [`Woolfrey/mujoco_ros2` repository](https://github.com/Woolfrey/mujoco_ros2) — independent community MuJoCo/ROS 2 communication package (§6.2)
- [Gazebo Sim headless rendering documentation](https://gazebosim.org/api/sim/9/headless_rendering.html) — `-s`/`--headless-rendering` flags and the GUI-less server process model (§7)
- [Chapter 210 — GPU Geometry Algorithms: Physics Simulation and Volumetric Methods](../part-29-graphics-algorithms/ch210-gpu-physics-and-volumetric.md) — §50.3 sequential-impulse GPU solver, §50.4 Featherstone articulated-body algorithm (§1.3, §4, Integrations)
- [Chapter 107 — Headless Rendering](../part-09-tooling-contributing/ch107-headless-rendering.md) — EGL surfaceless, GBM, OSMesa/llvmpipe general mechanism (§2.2, §7, Integrations)
- [Chapter 211 — ROS 2 Multimodal Sensor and Perception Pipeline](ch211-ros2-sensor-perception-pipeline.md) — `sensor_msgs`, TF2, `image_transport` consumed by both simulators' ROS 2 output (§6, Integrations)
- [Chapter 211b — NVIDIA Isaac Sim, Isaac Lab, and the GR00T Foundation-Model Family](ch211b-isaac-sim-isaac-lab-groot.md) — Isaac Sim/PhysX/RTX substrate, Newton status, Isaac Lab tensor-batched parallelism and domain randomization (§2.4, §8, §9, Integrations)
- [Chapter 69 — Omniverse, USD, and Hydra](../part-15-nvidia-stack/ch69-omniverse-usd.md) — Kit/USD/RTX substrate underneath Isaac Sim (Integrations)
- [Chapter 240 — Cosmos, OSMO, and Omniverse Farm](../part-15-nvidia-stack/ch240-cosmos-osmo-omniverse-farm.md) — Isaac Lab domain randomization and synthetic-data generation at scale (§8, Integrations)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
