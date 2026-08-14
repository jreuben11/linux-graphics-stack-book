# Chapter 211b: NVIDIA Isaac Sim, Isaac Lab, and the GR00T Foundation-Model Family

> **Part**: Part VII-B — Multimedia and Simulation Frameworks
> **Audience**: Systems and driver developers evaluating what a robotics simulator actually demands of the Linux graphics stack; graphics application developers who need to understand headless RTX rendering, tiled multi-camera capture, and GPU-resident physics tensors; and simulation/robotics engineers deciding whether to build on this stack at all.
> **Status**: First draft — 2026-08-10

Isaac Sim is frequently described as "NVIDIA's robotics simulator," which is accurate but unhelpful for anyone trying to understand what it *is* as a piece of software. Almost everything that makes Isaac Sim run — the application shell, the extension loader, the USD scene representation, the Hydra render delegate, the RTX path tracer, the PhysX 5 solver, the Warp kernel JIT — is not Isaac. It is the Omniverse Kit substrate, documented in Chapter 69. Isaac Sim is a set of extensions layered on top of that substrate, plus a curated asset library, plus an opinionated set of Kit experience files. Understanding the boundary is the difference between debugging in the right repository and debugging in the wrong one.

This chapter draws that boundary explicitly, then documents what lives on the Isaac side of it: the extension namespace and its two disruptive renames, the two structurally unrelated sensor pipelines (ray-traced RTX sensors versus physics-solver sensors), the ROS 2 bridge, Isaac Lab's reinforcement-learning environment architecture and its GPU-parallelism mechanics, the in-progress Newton physics integration, and the GR00T family of vision-language-action models that consume the synthetic data this stack produces. Several claims that circulate widely about this stack turn out not to survive contact with the source; those are corrected in place, with citations.

---

## Table of Contents

- [1. Scope: What Is Actually Isaac-Specific](#1-scope-what-is-actually-isaac-specific)
  - [1.1 The Substrate This Chapter Does Not Re-Derive](#11-the-substrate-this-chapter-does-not-re-derive)
  - [1.2 The Isaac-Specific Layer](#12-the-isaac-specific-layer)
- [2. Isaac Sim Architecture and the Extension Namespace](#2-isaac-sim-architecture-and-the-extension-namespace)
  - [2.1 The `omni.isaac.*` to `isaacsim.*` Rename](#21-the-omniisaac-to-isaacsim-rename)
  - [2.2 The Deprecation Sweep and `.experimental.` Naming](#22-the-deprecation-sweep-and-experimental-naming)
  - [2.3 Application Bootstrap and the `SimulationApp` Ordering Constraint](#23-application-bootstrap-and-the-simulationapp-ordering-constraint)
  - [2.4 Licensing: Apache-2.0 Source over a Proprietary Runtime](#24-licensing-apache-20-source-over-a-proprietary-runtime)
- [3. Lineage: Isaac Gym, Orbit, Isaac Lab](#3-lineage-isaac-gym-orbit-isaac-lab)
- [4. Sensor Simulation: Two Structurally Unrelated Pipelines](#4-sensor-simulation-two-structurally-unrelated-pipelines)
  - [4.1 RTX Sensors and the GenericModelOutput Render Variable](#41-rtx-sensors-and-the-genericmodeloutput-render-variable)
  - [4.2 Physics Sensors: The IMU Has No Rendering Path](#42-physics-sensors-the-imu-has-no-rendering-path)
- [5. ROS 2 Integration and External Transports](#5-ros-2-integration-and-external-transports)
  - [5.1 `isaacsim.ros2.bridge` and `isaacsim.ros2.core`](#51-isaacsimros2bridge-and-isaacsimros2core)
  - [5.2 `simulation_interfaces` and `isaacsim.ros2.sim_control`](#52-simulation_interfaces-and-isaacsimros2sim_control)
  - [5.3 IsaacSimZMQ Is a Separate Repository](#53-isaacsimzmq-is-a-separate-repository)
- [6. Isaac Lab: Manager-Based and Direct Workflows](#6-isaac-lab-manager-based-and-direct-workflows)
  - [6.1 Both Workflows Subclass `gym.Env`](#61-both-workflows-subclass-gymenv)
  - [6.2 The Four Supported RL Libraries, and Ray's Actual Role](#62-the-four-supported-rl-libraries-and-rays-actual-role)
  - [6.3 Licensing: BSD-3-Clause Core plus Apache-2.0 Mimic](#63-licensing-bsd-3-clause-core-plus-apache-20-mimic)
- [7. GPU Parallelism and the Rendering Path for RL Workloads](#7-gpu-parallelism-and-the-rendering-path-for-rl-workloads)
  - [7.1 The `omni.physics.tensors` Tensor API](#71-the-omniphysicstensors-tensor-api)
  - [7.2 Environment Cloning and `replicate_physics`](#72-environment-cloning-and-replicate_physics)
  - [7.3 Tiled Rendering and Multi-Camera Capture](#73-tiled-rendering-and-multi-camera-capture)
  - [7.4 Headless Operation and Opt-In Cameras](#74-headless-operation-and-opt-in-cameras)
  - [7.5 What the Published Benchmarks Actually Say](#75-what-the-published-benchmarks-actually-say)
- [8. Newton: The Linux Foundation Physics Engine](#8-newton-the-linux-foundation-physics-engine)
- [9. The GR00T Foundation-Model Family](#9-the-gr00t-foundation-model-family)
  - [9.1 Dual-System Architecture](#91-dual-system-architecture)
  - [9.2 Checkpoint Lineage and Backbones](#92-checkpoint-lineage-and-backbones)
  - [9.3 Embodiment Tags and the GR00T LeRobot Format](#93-embodiment-tags-and-the-gr00t-lerobot-format)
  - [9.4 What Is Frozen During Fine-Tuning](#94-what-is-frozen-during-fine-tuning)
  - [9.5 Weights Licensing Differs Per Checkpoint](#95-weights-licensing-differs-per-checkpoint)
  - [9.6 Inference: In-Process Policy and Client/Server](#96-inference-in-process-policy-and-clientserver)
- [10. Synthetic Data Generation: Mimic and Cosmos Chaining](#10-synthetic-data-generation-mimic-and-cosmos-chaining)
- [11. Isaac Lab versus MJX](#11-isaac-lab-versus-mjx)
- [Integrations](#integrations)
- [References](#references)

---

## 1. Scope: What Is Actually Isaac-Specific

### 1.1 The Substrate This Chapter Does Not Re-Derive

Isaac Sim is a Kit application. That single fact determines most of its behaviour, and the mechanics of Kit are covered in Chapter 69 rather than repeated here:

- **The extension system and application manifests.** Kit's `.kit` experience files, extension dependency resolution, and the `extension.toml` manifest format are Kit mechanisms (Ch69 §9.1). Isaac Sim contributes extensions; it does not contribute the loader.
- **USD as the scene representation, and Fabric/USDRT as the runtime mutation path.** Isaac Sim's robots are USD stages, its articulations are `UsdPhysics` prims, and its per-frame writes go through Fabric (Ch69 §9.2–9.3).
- **The RTX renderer and Hydra delegate.** Every pixel Isaac Sim produces — viewport, Replicator ground truth, RTX sensor return — comes out of the same RTX Hydra delegate documented in Ch69.
- **PhysX 5 and NVIDIA Warp.** The GPU rigid-body and articulation solver, and the Python-to-CUDA kernel JIT that Isaac's newer APIs are written against, are both substrate (Ch69 §12).
- **Headless operation and NGC container deployment.** Running Kit without a display server, and the container images that package it, are Kit-level concerns (Ch69 §10.2–10.3).

Where this chapter touches those topics, it does so only to describe the *Isaac-specific* configuration or constraint layered on top.

### 1.2 The Isaac-Specific Layer

What remains is genuinely robotics-shaped: a robotics extension namespace (`isaacsim.*`), two sensor pipelines that share no code path, a ROS 2 bridge built against the `rcl` C API, Isaac Lab's RL environment abstractions in a separate repository, and the GR00T model family with the synthetic-data tooling that feeds it. Those are the subject of the rest of this chapter.

---

## 2. Isaac Sim Architecture and the Extension Namespace

### 2.1 The `omni.isaac.*` to `isaacsim.*` Rename

Isaac Sim 4.5 renamed essentially every extension in the product. The old namespace `omni.isaac.*` was replaced by `isaacsim.*`, and the flat extension list was reorganised into functional groups — `isaacsim.core`, `isaacsim.sensors`, `isaacsim.robot`, `isaacsim.ros2`, `isaacsim.storage`, and so on. NVIDIA published a full old-name-to-new-name mapping table for the transition [Source](https://docs.isaacsim.omniverse.nvidia.com/4.5.0/overview/extensions_renaming.html).

> **Note: needs verification.** The *version* that introduced the rename (4.5) is confirmed by the documentation URL above, which is versioned. The exact calendar release date of Isaac Sim 4.5 could not be confirmed from a primary source and is therefore not stated here.

The practical consequence for anyone reading tutorials, blog posts, or Stack Overflow answers: any code importing `omni.isaac.core` predates 4.5 and will not run on a current install. This is not a deprecation shim situation — the old module names are gone.

Isaac Sim's own version number and the Omniverse Kit SDK version underneath it (Ch69 §9) are independent numbering schemes tracking different release cadences, and Isaac Sim's release notes pin the mapping explicitly at each release: Isaac Sim 4.5.0 updated its embedded Kit SDK from 106.1.0 to 106.5.0 [Source](https://docs.isaacsim.omniverse.nvidia.com/4.5.0/overview/release_notes.html), and Isaac Sim 6.0.1 updated from Kit SDK 110.1.1 (introduced in Isaac Sim 6.0.0) to 110.1.2 [Source](https://docs.isaacsim.omniverse.nvidia.com/6.0.1/overview/release_notes.html). Concretely: the 4.5-era `omni.isaac.*`→`isaacsim.*` rename above shipped on Kit 106.x, the same Kit generation as Ch69's 25.02 SDK target (Ch69 §4.2); the `.experimental.` naming inversion below and the Newton integration (§8) ship on Kit 110.x, two Kit generations later. A reader debugging a Kit-level rendering or extension-loading issue against a specific Isaac Sim install should check that install's own release notes for the exact Kit SDK point release rather than assuming it matches Ch69's Kit 106.x/109.0/110.1 examples, which were sourced from Omniverse's own release notes rather than from an Isaac Sim build.

### 2.2 The Deprecation Sweep and `.experimental.` Naming

A second, subtler churn event followed. In the Isaac Sim source tree at tag `v6.0.1`, the extensions are split across two directories: `source/extensions/` and `source/deprecated/`. Several extension IDs that most documentation still presents as current now live under `source/deprecated/` — including `isaacsim.core.api`, `isaacsim.sensors.rtx`, `isaacsim.sensors.physics`, and `isaacsim.sensors.camera` [Source](https://github.com/isaac-sim/IsaacSim/tree/v6.0.1/source/deprecated).

Their replacements carry an `.experimental.` infix. The current core API is `isaacsim.core.experimental.*`, spanning `actuators`, `materials`, `objects`, `primdata`, `prims`, and `utils`, and it is written against Warp rather than the older NumPy/PyTorch-flavoured wrappers [Source](https://github.com/isaac-sim/IsaacSim/tree/v6.0.1/source/extensions).

This produces a naming inversion that is worth stating plainly, because it reads backwards: **the extension named `.experimental.` is the current one, and the extension without it is the deprecated one.** When writing new code against Isaac Sim 6.x, reach for `isaacsim.core.experimental.prims`, not `isaacsim.core.api`.

### 2.3 Application Bootstrap and the `SimulationApp` Ordering Constraint

Isaac Sim's Python entry point has a hard ordering requirement. `SimulationApp` must be constructed *before* any other `isaacsim.*` import, because constructing it is what boots the Kit runtime and therefore what makes the extension system — and hence every other `isaacsim` module — importable at all. Every shipped standalone example is written this way, with the construction call sitting between two import blocks rather than at the top of the file.

The shipped standalone examples encode this ordering literally, with the app construction sitting between two import blocks:

```python
# source/standalone_examples/api/isaacsim.sensors.experimental.physics/imu_sensor.py (v6.0.1)
from isaacsim import SimulationApp

simulation_app = SimulationApp({"headless": False})

import isaacsim.core.experimental.utils.app as app_utils
import isaacsim.core.experimental.utils.stage as stage_utils
from isaacsim.core.experimental.objects import DistantLight, GroundPlane
from isaacsim.core.experimental.prims import Articulation
from isaacsim.core.simulation_manager import SimulationManager
from isaacsim.robot.experimental.wheeled_robots.controllers import DifferentialController
from isaacsim.sensors.experimental.physics import IMU, IMUSensor
from isaacsim.storage.native import get_assets_root_path
```

[Source](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/standalone_examples/api/isaacsim.sensors.experimental.physics/imu_sensor.py)

Note also `isaacsim.storage.native.get_assets_root_path`: Isaac Sim's robot and environment assets are not vendored in the repository. They are resolved from a configured asset root — an Omniverse Nucleus server or an HTTP mirror — and referenced by USD paths such as `/Isaac/Robots/NVIDIA/NovaCarter/nova_carter.usd`. A machine with no network route to the asset root will start Kit successfully and then fail at scene construction.

### 2.4 Licensing: Apache-2.0 Source over a Proprietary Runtime

Isaac Sim's repository carries an Apache-2.0 licence, and this is widely reported as "Isaac Sim is now open source." The licence file's own preamble is more careful than that summary, and the distinction matters for anyone planning to redistribute:

The Apache-2.0 grant covers the source in the `isaac-sim/IsaacSim` repository. It does not extend to the Omniverse Kit SDK runtime that the source is compiled and linked against, nor to the bundled models, textures, and other asset content, which carry their own terms [Source](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/LICENSE). The repository is best characterised as *source-available with a permissive licence on the Isaac layer*, sitting on a proprietary runtime — not as an end-to-end open-source stack.

The repository is also explicit that it is not currently a participatory open-source project: the README's Contributing section states "We do not support direct community contributions at the moment" [Source](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/README.md#contributing), and `CONTRIBUTING.md` redirects bug reports and feedback to the NVIDIA Omniverse developer forums [Source](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/CONTRIBUTING.md). Functionally, this is a code drop with a permissive licence rather than an upstream you can send patches to.

> **Note: needs verification.** Claims that redistribution of Isaac Sim requires membership in the NVIDIA Developer Program or an NVIDIA AI Enterprise licence could not be confirmed against a primary source. What *can* be stated from the licence text is that the grant is oriented toward use rather than redistribution, and is conditioned on running on NVIDIA GPUs. Readers with redistribution requirements should read the licence file directly rather than relying on any secondary summary, including this one.
>
> **Note: needs verification.** Specific minimum NVIDIA driver versions for Isaac Sim 5.x and 6.x are deliberately omitted here; they change per release and the repository's own requirements documentation is the authoritative source.

---

## 3. Lineage: Isaac Gym, Orbit, Isaac Lab

Three project names appear in the literature and only one of them is live.

**Isaac Gym (Preview)** was NVIDIA's original standalone GPU-accelerated RL physics package, distributed as a preview release outside Omniverse entirely. It is deprecated, with NVIDIA directing users to Isaac Lab as the successor [Source](https://developer.nvidia.com/isaac-gym).

**Orbit** was a framework built on Isaac Sim providing RL environment abstractions and robot assets. It was renamed to Isaac Lab; the original GitHub repository redirects [Source](https://github.com/isaac-sim/IsaacLab).

**Isaac Lab** is the current project. It is a *separate repository* from Isaac Sim, depends on Isaac Sim as its simulation backend, and carries its own licence, release cadence, and version numbers.

This lineage matters for reading benchmark claims. Numbers published against "Isaac Gym" are measuring a different physics-integration path from numbers published against Isaac Lab, and are not directly comparable.

A related versioning trap: **Isaac Lab 3.0 is not generally available.** The repository's default branch is a `release/3.0.0-beta2` branch, and the latest stable tag is in the 2.3.x series [Source](https://github.com/isaac-sim/IsaacLab/branches). An unpinned `git clone` therefore lands on a beta branch, not on stable — and an unpinned GitHub file URL resolves against that beta branch too. Source citations for Isaac Lab in this chapter name the branch or tag they were read from.

---

## 4. Sensor Simulation: Two Structurally Unrelated Pipelines

The most common architectural misconception about Isaac Sim's sensors is that they are variations on a theme — that a camera, a LiDAR, and an IMU are all "sensors" served by one subsystem with different output formats. They are not. There are two pipelines with no shared execution path, and they have different latency characteristics, different failure modes, and different requirements on the graphics stack.

### 4.1 RTX Sensors and the GenericModelOutput Render Variable

Ray-traced sensors — RTX cameras, RTX LiDAR, RTX radar — are rendering operations. They are evaluated by the RTX renderer as part of the render graph, and their output is delivered through Replicator's annotator mechanism as an arbitrary output variable (AOV).

The concrete integration point is a render variable named `GenericModelOutput` (GMO), registered as a Replicator annotator. The extension startup code registers it defensively, because the annotator was missing from a specific Replicator release:

```python
# source/deprecated/isaacsim.sensors.rtx/python/impl/extension.py (v6.0.1)
# Temporary fix due to GMO annotator missing from omni.replicator.core-1.13.6
if "GenericModelOutput" not in AnnotatorRegistry.get_registered_annotators():
    AnnotatorRegistry.register_annotator_from_aov(
        aov="GenericModelOutput",
        output_data_type=np.uint8,
    )
```

[Source](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/deprecated/isaacsim.sensors.rtx/python/impl/extension.py)

GMO carries a variable payload governed by an auxiliary-output level, with settings spanning `NONE`, `BASIC`, `EXTRA`, and `FULL`. Higher levels emit more per-return metadata — the kind of per-beam auxiliary data a LiDAR model needs — at proportionally higher bandwidth cost.

The consequences of this pipeline being a *rendering* pipeline are significant for deployment:

- RTX sensors require a working RTX render path. On a headless node, that means Kit's headless rendering configuration (Ch69 §10.2), not merely "run without a window."
- Their cost scales with scene complexity and ray count in the same way viewport rendering does, and competes for the same GPU resources as any other rendering work.
- Their update rate is coupled to the render loop, not to the physics step.

### 4.2 Physics Sensors: The IMU Has No Rendering Path

The IMU is the clean counterexample. It is a pure readout of the physics solver, computed by finite differencing of body state across physics substeps. There is no ray, no render variable, and no dependency on the renderer at all.

The C++ header states this structurally: the sensor does its work in `onPhysicsStep()`, and its `tick()` — the per-frame component-manager callback — is an empty function body.

```cpp
// source/deprecated/isaacsim.sensors.physics/include/isaacsim/sensors/physics/ImuSensor.h (v6.0.1)

/**
 * @brief Called by component manager tick to update sensor data
 * @details Processes finite difference data in mRawReadingList and saves in m_readingPair
 */
virtual void onPhysicsStep();

/**
 * @brief Empty tick implementation as processing is done in onPhysicsStep
 */
virtual void tick()
{
}
```

[Source](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/deprecated/isaacsim.sensors.physics/include/isaacsim/sensors/physics/ImuSensor.h)

An empty `tick()` with a populated `onPhysicsStep()` is about as explicit as a codebase gets about which clock a subsystem runs on. Physics sensors:

- run at the physics rate, which is typically far higher than the render rate;
- work correctly with rendering fully disabled;
- are unaffected by scene visual complexity, materials, or lighting.

The API also splits authoring from runtime, which trips up readers expecting a single class. `IMU` is the USD prim authoring interface — `IMU.create()` writes the prim onto the stage — while `IMUSensor` wraps that prim to provide the runtime read interface:

```python
# source/standalone_examples/api/isaacsim.sensors.experimental.physics/imu_sensor.py (v6.0.1)
imu_sensor = IMUSensor(
    IMU.create(
        "/World/Carter/caster_wheel_left/imu_sensor",
        translations=np.array([[0.0, 0.0, 0.0]]),
    )
)

SimulationManager.setup_simulation(dt=1.0 / 60.0, device="cpu")
app_utils.play()
app_utils.update_app(steps=2)
print(imu_sensor.get_data())
```

[Source](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/standalone_examples/api/isaacsim.sensors.experimental.physics/imu_sensor.py)

The practical planning implication: a robot policy that consumes only proprioception, IMU, and contact state can be trained on a fleet of GPUs with no rendering at all. A policy that consumes camera or LiDAR observations cannot, and its throughput ceiling is a rendering ceiling, not a physics ceiling.

---

## 5. ROS 2 Integration and External Transports

### 5.1 `isaacsim.ros2.bridge` and `isaacsim.ros2.core`

ROS 2 integration is split across two extensions: `isaacsim.ros2.core` holds the transport-level implementation, and `isaacsim.ros2.bridge` exposes it to the simulation as OmniGraph (action graph) nodes — publishers, subscribers, clock sources, TF broadcasters, and camera-image publishers that a user wires into a graph rather than calling from Python.

The bridge's transport layer is implemented in C++ against ROS 2's `rcl` C API. (A Python ROS 2 client library is bundled for higher-level use, but it is not what carries messages at the bridge boundary.)

Supported distributions are enumerated exhaustively in a header, and there are exactly two:

```cpp
// source/extensions/isaacsim.ros2.core/include/isaacsim/ros2/core/Ros2Distro.h (v6.0.1)
    eHumble,
    /** @brief ROS 2 Jazzy Jalisco distribution */
    eJazzy,
...
constexpr std::array<Ros2DistroInfo, 2> g_kDistroMapping{ { { "humble", Ros2Distro::eHumble },
                                                            { "jazzy", Ros2Distro::eJazzy } } };
```

[Source](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/extensions/isaacsim.ros2.core/include/isaacsim/ros2/core/Ros2Distro.h)

A `constexpr std::array<..., 2>` is a compile-time enumeration, not a runtime lookup with a fallback. Distributions other than Humble and Jazzy are not "untested" — they are not representable.

### 5.2 `simulation_interfaces` and `isaacsim.ros2.sim_control`

`simulation_interfaces` is a standardised ROS 2 interface package defining a vendor-neutral control surface for simulators: start, pause, step, reset, spawn and delete entities, query and set entity state, and so on. Its value is that a ROS 2 client can drive any conforming simulator without simulator-specific code.

Isaac Sim implements it in the `isaacsim.ros2.sim_control` extension, exposing 19 services plus a `SimulateSteps` action [Source](https://github.com/isaac-sim/IsaacSim/tree/v6.0.1/source/extensions/isaacsim.ros2.sim_control). The specification itself is a multi-vendor effort within the ROS ecosystem rather than an NVIDIA design, which is precisely what gives it portability value.

### 5.3 IsaacSimZMQ Is a Separate Repository

A frequently repeated claim is that Isaac Sim 5.0 added a ZeroMQ bridge, developed alongside the `simulation_interfaces` collaboration. Both halves of that claim are wrong, and they concern two unrelated deliverables.

First, the ZeroMQ bridge is not in Isaac Sim. A search of the `isaac-sim/IsaacSim` tree returns no ZeroMQ code. The bridge lives in a separate repository, `isaac-sim/IsaacSimZMQ`, under an MIT licence, providing the extension `isaacsim.zmq.bridge.examples` and targeting Isaac Sim 6.0.1 as an external add-on [Source](https://github.com/isaac-sim/IsaacSimZMQ). It uses ZeroMQ for transport and Protobuf for message encoding — a deliberately lightweight, language-agnostic alternative for clients that do not want a ROS 2 dependency.

Second, the multi-vendor collaboration in this space is the `simulation_interfaces` standardisation described in §5.2, which Isaac Sim implements via `isaacsim.ros2.sim_control`. It has no relationship to the ZMQ bridge. Conflating the two attributes a standards effort to an unrelated example extension.

---

## 6. Isaac Lab: Manager-Based and Direct Workflows

Isaac Lab offers two ways to define a reinforcement-learning environment, and the choice is genuinely architectural rather than stylistic.

**Manager-based** environments (`ManagerBasedRLEnv`) are assembled declaratively from configuration classes. Observations, rewards, terminations, events, and actions are each described by a config object holding *terms* — named entries binding a function to a weight and parameters. The framework's managers evaluate and combine them.

```python
# source/isaaclab_tasks/isaaclab_tasks/manager_based/classic/cartpole/cartpole_env_cfg.py
class RewardsCfg:
    alive = RewTerm(func=mdp.is_alive, weight=1.0)
    terminating = RewTerm(func=mdp.is_terminated, weight=-2.0)
    pole_pos = RewTerm(func=mdp.joint_pos_target_l2, weight=-1.0,
        params={"asset_cfg": SceneEntityCfg("robot", joint_names=["cart_to_pole"]), "target": 0.0})
```

[Source](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/source/isaaclab_tasks/isaaclab_tasks/manager_based/classic/cartpole/cartpole_env_cfg.py)

The payoff is composability: swapping a robot, adding a reward term, or randomising an observation is a config edit, and terms are reusable across tasks. The cost is indirection — a misbehaving reward requires tracing through the manager.

**Direct** environments (`DirectRLEnv`) hand the developer a single class with explicit `_get_observations`, `_get_rewards`, and `_get_dones` methods operating on batched tensors, typically dispatching into a `@torch.jit.script`-compiled kernel:

```python
# source/isaaclab_tasks/isaaclab_tasks/direct/cartpole/cartpole_env.py
def _get_rewards(self) -> torch.Tensor:
    total_reward = compute_rewards(self.cfg.rew_scale_alive, ..., self.reset_terminated)
```

[Source](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/source/isaaclab_tasks/isaaclab_tasks/direct/cartpole/cartpole_env.py)

Direct suits research on non-standard MDP structure and squeezing per-step cost; manager-based suits building families of related tasks.

### 6.1 Both Workflows Subclass `gym.Env`

A detail with real consequences: both `ManagerBasedRLEnv` and `DirectRLEnv` subclass `gymnasium.Env` — *not* `gymnasium.vector.VectorEnv` — despite stepping thousands of environments in parallel. Isaac Lab environments present a single-environment interface whose observations, actions, rewards, and done flags all carry a leading batch dimension of size `num_envs`.

This is why Isaac Lab ships per-library wrapper classes rather than relying on standard Gymnasium vector adapters: each RL library has its own expectation about how batched data arrives, and none of them can infer it from the `gym.Env` base class. It is also why naïvely wrapping an Isaac Lab environment in a Gymnasium vectorisation utility produces silently wrong shapes rather than an error. The framework pins Gymnasium to a specific version (1.2.1 in the 3.0 beta line) precisely because this interface boundary is version-fragile [Source](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/source/isaaclab/setup.py).

### 6.2 The Four Supported RL Libraries, and Ray's Actual Role

Isaac Lab ships wrappers for exactly four external RL libraries:

| Library | Character |
|---|---|
| **RSL-RL** | Compact GPU-resident PPO implementation; the usual choice for legged-locomotion tasks |
| **RL-Games** | High-throughput implementation originally developed against Isaac Gym-era workloads |
| **skrl** | Modular, multi-backend library with broad algorithm coverage |
| **Stable-Baselines3** | Widely used CPU-oriented reference implementation; the interoperability option |

[Source](https://github.com/isaac-sim/IsaacLab/tree/release/3.0.0-beta2/source/isaaclab_rl)

**Ray RLlib is not among them.** This correction matters because RLlib is a prominent library and its absence is easy to assume away. Ray *does* appear in Isaac Lab, but in a different role entirely: as KubeRay-based cluster orchestration and hyperparameter tuning — launching and sweeping many training jobs — not as a provider of RL algorithms consuming Isaac Lab environments [Source](https://github.com/isaac-sim/IsaacLab/tree/release/3.0.0-beta2/scripts/reinforcement_learning/ray). Ray schedules the runs; RSL-RL, RL-Games, skrl, or SB3 does the learning inside them.

### 6.3 Licensing: BSD-3-Clause Core plus Apache-2.0 Mimic

Isaac Lab's core is BSD-3-Clause. A second licence file, `LICENSE-mimic` at the repository root, applies Apache-2.0 to the `isaaclab_mimic` component [Source](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/LICENSE-mimic). Note the location: it is a root-level `LICENSE-mimic`, not a `LICENSE` file inside the `source/isaaclab_mimic/` package directory.

The split is not merely documentary. It is mechanically enforced at commit time by two separate `insert-license` pre-commit hooks with different header templates and different path filters, so a new file in the mimic tree receives the Apache header and a new file elsewhere receives the BSD header [Source](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/.pre-commit-config.yaml). Anyone vendoring parts of Isaac Lab needs to track which tree a file came from.

---

## 7. GPU Parallelism and the Rendering Path for RL Workloads

### 7.1 The `omni.physics.tensors` Tensor API

Isaac Lab's throughput comes from never moving simulation state off the GPU. The mechanism is the tensor API exposed by `omni.physics.tensors`, which presents batched views over PhysX's GPU-resident state.

The entry point is `create_simulation_view`, called with a backend string:

```python
# source/isaaclab_physx/isaaclab_physx/physics/physx_manager.py
cls._view = omni.physics.tensors.create_simulation_view("warp", stage_id=stage_id)
```

[Source](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/source/isaaclab_physx/isaaclab_physx/physics/physx_manager.py)

From the returned `SimulationView`, the caller acquires typed batched views — `ArticulationView`, `RigidBodyView` — exposed directly under `omni.physics.tensors.api`. Reading joint positions for 4,096 robots is one call returning one tensor of shape `(4096, num_dofs)`; writing joint targets is one call taking a tensor of the same shape. There is no per-environment Python loop and no host round-trip.

Two corrections on this subsystem:

- The API's name is `omni.physics.tensors`. "OmniPhysics" is not a verifiable NVIDIA API or product name and should not be used.
- Confirmed backends are PyTorch and Warp. NumPy appears in Isaac Sim's sensor buffer paths but was **not** confirmed as a physics-tensor backend; treat any claim of NumPy-backed physics tensors as unverified.

`omni.physics.tensors` ships as a closed-source component inside Isaac Sim. Its interface is documented and stable; its implementation is not readable, which is a meaningful constraint if you are debugging a performance cliff at this layer.

### 7.2 Environment Cloning and `replicate_physics`

Thousands of environments are produced by cloning a single authored environment prim. Isaac Sim's `GridCloner` lays out copies on a grid and `clone_environments()` performs the USD-level duplication. The critical flag is `replicate_physics=True`, which tells the physics backend that the clones are structurally identical, allowing PhysX to build one shared physics description and instance it rather than parsing thousands of independent articulations.

Two implications follow directly from that assumption:

- Scene construction time stops scaling linearly with `num_envs` — which is what makes 4,096-environment startup practical at all.
- Clones must be structurally identical. Per-environment variation must be expressed as *parameter* randomisation applied through the tensor views after cloning (masses, friction, gains, initial states), not as structural variation in the USD hierarchy. Domain randomisation that changes topology is incompatible with `replicate_physics=True`.

### 7.3 Tiled Rendering and Multi-Camera Capture

Vision-based RL needs one camera per environment, and the naïve implementation — N cameras, N render products, N GPU-to-host copies per step — collapses well before N reaches interesting numbers.

Isaac Lab's `TiledCamera` addresses this by allocating a **single render product covering all clones of a given camera**, rendering them as tiles of one large image, and delivering the batch with a single device synchronisation [Source](https://github.com/isaac-sim/IsaacLab/tree/release/3.0.0-beta2/source/isaaclab/isaaclab/sensors/camera). The saving is not in raster work — the same pixels are shaded either way — but in eliminating per-camera render-product setup, per-camera annotator dispatch, and per-camera synchronisation, which is where the naïve approach's cost actually lives.

The trade-off is uniformity: tiles in one render product share resolution and render settings. Heterogeneous per-environment camera configurations do not fit the tiled path.

### 7.4 Headless Operation and Opt-In Cameras

Two command-line flags in Isaac Lab's app launcher control the graphics path, and their semantics are worth reading precisely:

```python
# source/isaaclab/isaaclab/app/app_launcher.py
arg_group.add_argument("--headless", action="store_true", help="Force display off at all times.")
arg_group.add_argument("--enable_cameras", action="store_true",
                       help="Enable camera sensors and relevant extension dependencies.")
```

[Source](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/source/isaaclab/isaaclab/app/app_launcher.py)

These are orthogonal. `--headless` suppresses the display; `--enable_cameras` opts *in* to camera sensors and pulls in their extension dependencies. Headless-with-cameras is a valid and common combination: a cluster node rendering tiled observations with no display attached. Camera sensors being opt-in means that a proprioception-only training run — the majority of locomotion work — never loads the camera extension stack at all.

> **Note: needs verification.** A stronger claim sometimes made — that `--enable_cameras` measurably slows startup or per-step time — could not be confirmed against a documented benchmark. What the documentation does state is that headless mode can significantly improve performance when rendering is not required, and it offers hardware-scaled guidance for camera counts (see §7.5). Those are the defensible statements.

### 7.5 What the Published Benchmarks Actually Say

There is no published, hardware-independent "maximum parallel environments per GPU" figure, and inventing one would be misleading — the ceiling is task-dependent, dominated by whether the task renders. What NVIDIA publishes instead is a per-task benchmark table, and the shape of that table is more informative than a single number would be [Source](https://isaac-sim.github.io/IsaacLab/main/source/refs/performance_benchmarks.html):

| Task | Environments | Notes |
|---|---|---|
| Cartpole (direct) | 4,096 | ≈3.7 GB system RAM, ≈3.3 GB VRAM |
| Cartpole RGB Camera | 1,024 | ≈7.5 GB system RAM, ≈16.7 GB VRAM |
| Velocity-Rough-G1 (humanoid locomotion) | 4,096 | Proprioception-only |
| Repose-Cube-Shadow (dexterous manipulation) | 8,192 | Proprioception-only |

The instructive comparison is Cartpole-direct against Cartpole-RGB-Camera: adding vision to the *same task* cuts the environment count by 4× while raising VRAM roughly 5×. Vision-based RL is memory-bound on the render side long before physics becomes the constraint.

Separately, the documentation offers a hardware-scaled recommendation for camera counts — on the order of 512 cameras on an RTX 4090-class GPU — as sizing guidance rather than a hard limit.

---

## 8. Newton: The Linux Foundation Physics Engine

Newton is a GPU-accelerated physics engine hosted as a Linux Foundation project, initiated by Disney Research, Google DeepMind, and NVIDIA, and built on NVIDIA Warp [Source](https://github.com/newton-physics/newton). It generalises and supersedes Warp's deprecated `warp.sim` module, and its primary solver backend is MuJoCo-Warp, with an XPBD solver also available. That solver choice is worth pausing on: it makes Newton the convergence point between this chapter's Isaac Lab path and the MJX-Warp variant of MuJoCo XLA covered in §11's comparison table below — not two independent GPU-physics efforts that happen to share a vendor, but the same underlying solver reached from both the NVIDIA-stack side and the vendor-neutral MuJoCo side. Chapter 211a covers MuJoCo/MJX's own side of that convergence in depth.

Licensing is two-part, and the code licence file has a non-obvious name: `LICENSE.md` carries Apache-2.0 for the code, while documentation is CC-BY-4.0 [Source](https://github.com/newton-physics/newton/blob/main/LICENSE.md).

Its integration status in the NVIDIA stack should be stated conservatively:

- In **Isaac Sim 6.0**, Newton appears as the `isaacsim.physics.newton` extension. The physics engine is *runtime-selectable*: one engine is active at a time, and the Newton integration is documented as experimental. It is not a "dual backend" in the sense of PhysX and Newton co-simulating.
- In **Isaac Lab 3.0 Beta**, Newton support lives on the development branch and the project states explicitly that it does not expect to be able to provide official support for it [Source](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/README.md).

That "no support commitment" statement is a precise engineering signal, not a hedge: Newton is a direction of travel, and production work in 2026 still targets PhysX.

> **Note: needs verification.** The commonly repeated statement that "PhysX remains the default physics engine in Isaac Sim 6.0" appears in summaries but was not confirmed against primary documentation. The verifiable statements are those above — runtime-selectable, one engine active at a time, Newton experimental.

---

## 9. The GR00T Foundation-Model Family

GR00T (Generalist Robot 00 Technology) is NVIDIA's open vision-language-action (VLA) model family for humanoid and manipulator control. It is the consumer of everything the preceding sections produce: Isaac Sim generates the scenes, Isaac Lab and Mimic generate the trajectories, and GR00T is the policy trained on them.

### 9.1 Dual-System Architecture

GR00T N1 and its successors use an explicitly two-system design, with the two halves running at different rates:

- **System 2** is a vision-language model that consumes images and a language instruction and produces a latent representation of intent. It runs at approximately 10 Hz — deliberation speed.
- **System 1** is a flow-matching Diffusion Transformer (DiT) that consumes System 2's latent plus proprioception and emits an action chunk. It runs at control rate.

The DiT conditions on the diffusion timestep through adaptive layer normalisation (AdaLN), the standard conditioning mechanism for diffusion transformers. Cross-embodiment support is handled at the boundaries rather than in the trunk: **embodiment-indexed MLP encoders and decoders** map each robot's proprioception vector into the shared latent space and map shared actions back out to that robot's actuator space. The trunk is embodiment-agnostic; only the thin input and output projections are per-robot [Source](https://github.com/NVIDIA/Isaac-GR00T).

GR00T N1.7 additionally moves to a **relative end-effector action space**, predicting EEF deltas rather than absolute poses — which generalises better across robots whose kinematics differ but whose task-space motions are similar.

### 9.2 Checkpoint Lineage and Backbones

Four N-series checkpoints have been published, and the family is larger than most summaries state:

| Checkpoint | System 2 backbone |
|---|---|
| GR00T-N1-2B | Eagle-2 VLM |
| GR00T-N1.5-3B | Eagle-2 VLM |
| GR00T-N1.6-3B | Eagle-2 lineage |
| GR00T-N1.7-3B | Cosmos-Reason2-2B, using the Qwen3-VL architecture |

The N1.7 backbone description is frequently rendered as an either/or — Cosmos-Reason2 *or* Qwen3-VL. It is both: Cosmos-Reason2-2B is the model, and Qwen3-VL is the architecture it is built on.

There is also a separate **GR00T-H / GR00T-H-N1.7** line targeting medical and surgical robotics, distinct from the general-purpose N series.

Two primary-source defects are worth flagging, because a reader consulting the model cards will hit them. The N1.7 card retains stale boilerplate describing SigLip2 and T5 components inherited from an earlier card template, which does not match the stated Cosmos-Reason2 backbone; and its parameter-size hyperlinks are inconsistent between 2B and 8B figures. Where the card contradicts itself, the accompanying technical report is the more reliable source.

### 9.3 Embodiment Tags and the GR00T LeRobot Format

Cross-embodiment training requires knowing which robot each trajectory came from. GR00T handles this with **embodiment tags** — an enumeration selecting which per-embodiment encoder/decoder MLP pair to use. `EmbodimentTag.NEW_EMBODIMENT` is the value for a robot not in the pretraining set, which is what a user fine-tuning on their own hardware selects.

Data arrives in the **GR00T LeRobot format**: the community LeRobot v2 dataset layout, extended with a `meta/modality.json` file that declares the semantic role and indexing of each state and action dimension. That extra file is what lets one training pipeline consume datasets from robots with different joint counts and different sensor suites — the modality description tells the loader how to slice each record [Source](https://github.com/NVIDIA/Isaac-GR00T/blob/main/getting_started/data_preparation.md).

### 9.4 What Is Frozen During Fine-Tuning

A widely repeated simplification holds that GR00T's VLM is frozen during fine-tuning. The actual policy is partial, and the paper's own hyperparameter table is finer-grained than the prose summary:

- Backbone **language** component — frozen
- Backbone **vision encoder** — unfrozen
- **DiT** (System 1) — unfrozen

The paper's own phrasing is that the language component of the vision-language backbone is kept frozen while the rest of the model is fine-tuned. Freezing the language tower preserves the instruction-following semantics learned at web scale; leaving the vision encoder trainable lets it adapt to the specific cameras, lighting, and viewpoints of the target robot — which is where robot data actually differs from web data.

### 9.5 Weights Licensing Differs Per Checkpoint

This is the single most consequential practical detail in this section, and the answer is not uniform across the family. As observed on 2026-08-10, there is a clean boundary at N1.7:

| Checkpoint | Weights licence | Stated use |
|---|---|---|
| GR00T-N1-2B | NVIDIA OneWay Noncommercial Licence | Non-commercial |
| GR00T-N1.5-3B | NVIDIA OneWay Noncommercial Licence | "ready for non-commercial use" |
| GR00T-N1.6-3B | NVIDIA OneWay Noncommercial Licence | Non-commercial |
| GR00T-N1.7-3B | NVIDIA Open Model Licence Agreement | "ready for commercial/non-commercial use" |

Three cautions follow:

1. **None of the four model cards carries a `license:` YAML frontmatter field.** Tooling that reads Hugging Face metadata to determine a licence will find nothing and may report "unknown" or fall back to a default. The licence is stated in the card body prose and in a linked document, not in machine-readable metadata. Any automated licence audit over these repositories will produce a wrong answer unless it reads the prose.
2. **OpenMDW-1.1 does not apply to any currently-published GR00T checkpoint.** It has been described as the licence for *future* GR00T releases, in a company blog post only — not in the corresponding newsroom press release, and not on any model card. Chapter 240 §2.2 covers OpenMDW-1.1 in the context of the Cosmos model family; do not transfer that conclusion to GR00T's current weights.
3. **The Isaac-GR00T repository README contradicts itself.** One passage describes the release as fully commercially licensable under Apache-2.0, while the README's own License section states the standard split: code under Apache-2.0, weights under an NVIDIA model licence. The per-checkpoint model card is the authority; the README's summary sentence is not.

The structural rule to carry away: **repository code licence and model weights licence are independent**, and for GR00T the weights licence also varies by checkpoint version. Apache-2.0 on the inference code says nothing about whether you may deploy the weights commercially.

> **Note: needs verification.** Model licensing changes without notice and without a version bump on the card. The table above is a point-in-time observation on 2026-08-10. Verify against the specific model card before any deployment decision.

**Licensing across the stack, consolidated.** This chapter has now stated a separate licence for each layer, in the section covering that layer. Because "what licence governs X" is exactly the kind of question a reader returns to the chapter for later, without wanting to re-read every section, it is worth collecting those statements in one place rather than leaving them to be found by cross-reference alone:

| Component | Licence | Section |
|---|---|---|
| Isaac Sim source (`isaac-sim/IsaacSim`) | Apache-2.0, over a proprietary Kit SDK runtime not covered by that grant | §2.4 |
| Isaac Sim bundled assets/models | Separate terms, not Apache-2.0 | §2.4 |
| IsaacSimZMQ bridge | MIT | §5.3 |
| Isaac Lab core | BSD-3-Clause | §6.3 |
| Isaac Lab `isaaclab_mimic` | Apache-2.0 (root-level `LICENSE-mimic`, mechanically enforced per-tree) | §6.3 |
| MimicGen (`NVlabs/mimicgen`, the original algorithm `isaaclab_mimic` reimplements) | "NVIDIA License," not Apache-2.0 | §10 |
| Newton code | Apache-2.0 | §8 |
| Newton documentation | CC-BY-4.0 | §8 |
| Isaac-GR00T repository code | Apache-2.0 | §9.5 |
| GR00T-N1/N1.5/N1.6 weights | NVIDIA OneWay Noncommercial Licence | §9.5 |
| GR00T-N1.7 weights | NVIDIA Open Model Licence Agreement | §9.5 |
| GR00T weights under OpenMDW-1.1 | None currently — stated as a future intention only (Ch240 §2.2 covers OpenMDW-1.1 for Cosmos, not GR00T) | §9.5 |

No single licence governs "Isaac Sim" or "GR00T" as a whole, and the two axes that most often get collapsed into one — repository code licence and shipped-artifact (runtime, asset, or weights) licence — are independent at every layer in this table, not just at the GR00T checkpoint boundary discussed in §9.5.

### 9.6 Inference: In-Process Policy and Client/Server

GR00T offers two inference topologies. In-process loading wraps a checkpoint directly:

```python
from gr00t.policy import Gr00tPolicy
from gr00t.data.embodiment_tags import EmbodimentTag

# Load your trained model
policy = Gr00tPolicy(
    model_path="/path/to/your/checkpoint",
    embodiment_tag=EmbodimentTag.NEW_EMBODIMENT,  # or other embodiment tags
    device="cuda:0",  # or "cpu", or device index like 0
    strict=True  # Enable input/output validation (recommended during development)
)
```

[Source](https://github.com/NVIDIA/Isaac-GR00T/blob/main/getting_started/policy.md)

The client/server topology puts the model behind a socket, which is what physical deployment usually needs — the robot's onboard computer rarely hosts the GPU that runs a 3B-parameter VLA:

```python
from gr00t.policy.server_client import PolicyClient

policy = PolicyClient(host="localhost", port=5555)
env = YourEnvironment()
obs, info = env.reset()
action, info = policy.get_action(obs)
obs, reward, done, truncated, info = env.step(action)
```

[Source](https://github.com/NVIDIA/Isaac-GR00T)

`PolicyClient` is API-compatible with the in-process policy at the `get_action` boundary, so the same rollout loop drives either topology. Note the `strict=True` recommendation on `Gr00tPolicy`: because the modality schema is data-driven, a mismatch between a dataset's `modality.json` and the checkpoint's expected layout otherwise manifests as silently wrong action semantics rather than an exception.

---

## 10. Synthetic Data Generation: Mimic and Cosmos Chaining

The bottleneck in robot learning is demonstration data, and this stack's answer is trajectory multiplication rather than trajectory collection.

The seed demonstrations that `isaaclab_mimic` multiplies still have to come from somewhere, and NVIDIA's `NVIDIA/IsaacTeleop` is the collection-side counterpart: a teleoperation framework for driving both simulated (Isaac Sim) and physical robots from the same operator input — spacemouse, VR controller, or leader-arm — so that a small number of human-piloted demonstrations can be recorded once and then either used directly or handed to Mimic for object-centric multiplication. Where Mimic's job is turning few demonstrations into many, IsaacTeleop's job is producing the few in the first place, and doing so against the same simulated robot and scene Isaac Sim already renders, so a recorded trajectory is immediately usable by the pipelines in this section without a separate real-robot capture rig. [Source: NVIDIA/IsaacTeleop, https://github.com/NVIDIA/IsaacTeleop]

`isaaclab_mimic` implements automated demonstration generation *inspired by* MimicGen. It is an independent implementation, not a vendored copy — a distinction with a licensing consequence, since the original `NVlabs/mimicgen` repository is distributed under an "NVIDIA License" rather than Apache-2.0, which readers are likely to assume given Isaac Lab's mimic component is Apache-2.0.

The technique segments a human demonstration into object-centric subtasks — each expressed relative to the object being manipulated rather than in world coordinates — then adapts each segment to a new scene configuration by applying the rigid transform between the original and new object poses, and stitches the adapted segments into a complete trajectory. The implementation exposes this as `DataGenerator` operating over `Waypoint`, `WaypointSequence`, and `WaypointTrajectory` types [Source](https://github.com/isaac-sim/IsaacLab/tree/release/3.0.0-beta2/source/isaaclab_mimic).

The object-centric framing is what makes the multiplication valid: because each subtask is defined relative to its object, moving the object in the new scene moves the entire sub-trajectory coherently, preserving the demonstrated contact-time behaviour instead of producing a kinematically plausible but physically wrong path.

Two pipelines chain this with generative video models, and they differ in which model does the work:

- **GR00T-Mimic** — Omniverse/Isaac Sim renders the multiplied trajectories, and Cosmos Transfer-1 performs a photorealism transfer on the rendered frames, closing the sim-to-real appearance gap.
- **GR00T-Dreams** — Cosmos Predict-2 generates video from image and text prompts, with Cosmos Reason providing the reasoning stage, producing training data without a simulator rollout at all.

Chapter 240 §6.2 documents these pipelines as a case study with the published throughput and downstream-performance figures; they are not restated here.

---

## 11. Isaac Lab versus MJX

MuJoCo XLA (MJX) is the natural comparison point: another GPU-parallel simulator aimed at large-scale RL. The two make genuinely different bets, and the differences are architectural rather than a matter of maturity.

| Dimension | Isaac Lab (PhysX backend) | MJX |
|---|---|---|
| Solver | PhysX 5, GPU-resident, closed-source | MuJoCo re-implemented for XLA; MJX-JAX and MJX-Warp variants |
| Hardware portability | NVIDIA GPUs only | MJX-JAX runs on "all compute hardware supported by the XLA compiler" — NVIDIA and AMD GPUs, Apple Silicon, Google Cloud TPUs. MJX-Warp targets NVIDIA GPUs specifically |
| Differentiability | Not a design goal of the PhysX path | "Differentiability is mostly supported in MJX-JAX but is not currently available in MJX-Warp" |
| Parallelism model | Batched tensor views over a cloned USD stage (`omni.physics.tensors`) | Batch dimensions on `mjx.Model` / `mjx.Data`, parallelised with `jax.vmap` / `jax.pmap` |
| Rendering | Full RTX path: photorealistic, ray-traced sensors, Replicator ground truth, tiled multi-camera | MJX-Warp includes "a hardware-accelerated batch renderer for generating pixel observations (such as RGB and depth) across multiple parallel environments"; MJX-JAX has no dedicated batch rendering |
| Scene authoring | USD, with the full Omniverse content pipeline | MJCF |
| Integrator / solver options | PhysX TGS-family solvers | MJX-JAX: EULER, RK4, IMPLICITFAST integrators; CG and NEWTON solvers. MJX-Warp: all except the IMPLICITFAST midpoint integrator feature, and all solvers except PGS and noslip |
| Licence | Isaac Lab BSD-3-Clause (mimic Apache-2.0); Isaac Sim Apache-2.0 source over proprietary Kit runtime | Apache-2.0 |

[Source](https://mujoco.readthedocs.io/en/stable/mjx.html)

The decision rule that follows: if the observation space is proprioceptive and the priority is solver throughput, hardware portability, or gradients through the dynamics, MJX is the stronger fit — and its Apache-2.0 licence with no proprietary runtime underneath is materially simpler to redistribute. If the observation space is visual and the requirement is photorealistic RGB, ray-traced LiDAR, or domain randomisation over materials and lighting, Isaac Sim's RTX path is the differentiator, and the Kit substrate is the price of admission. MJX-Warp's batch renderer narrows this gap for pixel observations, but it is a rasterised-observation renderer rather than an RTX path-tracing pipeline, so it does not target photorealism or ray-traced sensor physics.

---

## Roadmap

### Near-term (6–12 months)

- **Isaac Lab 3.0 moving from beta to general availability.** As of this writing, Isaac Lab 3.0 ships only as beta tags (`v3.0.0-beta`, `v3.0.0-beta2`) off the `release/3.0.0-beta2` branch (§3). The repository tracks a dedicated GA milestone whose scoped work is substantially closed out already, which points to GA landing as the next tagged release rather than a distant target. [Source](https://github.com/isaac-sim/IsaacLab/milestones)
- **Newton's point-release cadence continuing.** Newton has shipped several point releases in quick succession, with active milestones tracking further solver and integration work beyond the version this chapter cites. This is consistent with the "experimental," runtime-selectable status this chapter documents for Newton inside Isaac Sim 6.0 and Isaac Lab 3.0 (§8) rather than an imminent default-backend switch. [Source](https://github.com/newton-physics/newton/releases)
- **`simulation_interfaces` still stabilising.** The specification's own documentation flags at least one core message type as likely to be revised before a final version, meaning the interface surface Isaac Sim's `isaacsim.ros2.sim_control` implements (§5.2) is not yet frozen. [Source](https://github.com/ros-simulation/simulation_interfaces)
- **GR00T hardware and inference-path breadth expanding.** Recent Isaac-GR00T commits add support for additional physical robot arms and streamline the inference image path, continuing the pattern of incremental deployment-side hardening between major checkpoint releases rather than a new backbone revision. [Source](https://github.com/NVIDIA/Isaac-GR00T/commits/main)

### Medium-term (1–3 years)

- **Newton and MuJoCo-Warp deepening as a shared convergence point.** `mujoco_warp` — the GPU backend Newton's primary solver builds on (§8) — continues shipping frequent point releases well past the MuJoCo 3.11 baseline this chapter cites for MJX (§11), suggesting the solver convergence between Isaac Lab's Newton path and MJX-Warp will keep deepening rather than having been a one-time integration event. [Source](https://github.com/google-deepmind/mujoco_warp/releases)
- **Newton integration friction in Isaac Lab working itself toward supported status.** Isaac Lab's issue tracker carries an ongoing stream of Newton-specific bug reports and feature proposals — USD import fidelity, Fabric transform correctness, mixed-solver spawns — which is the ordinary signature of a backend maturing from experimental toward officially supported rather than of a stalled integration. [Source](https://github.com/isaac-sim/IsaacLab/issues?q=is%3Aissue+Newton)
- **GR00T checkpoint iteration likely to continue at a similar pace.** The N-series has progressed through several checkpoint revisions in close succession, with backbone changes at nearly every step (Eagle-2 through Cosmos-Reason2/Qwen3-VL, §9.2). Nothing in the family's public release history suggests this pace is slowing, making further backbone or action-space revisions a reasonable expectation, though without a committed date or name. [Source](https://github.com/NVIDIA/Isaac-GR00T/tags)

### Long-term

- **PhysX's position as Isaac Sim's default solver is not guaranteed indefinitely, but no vendor commitment sets a date.** Newton is explicitly a Linux Foundation project rather than an NVIDIA-controlled one, and its stated purpose is to generalise and eventually supersede Warp's own deprecated `warp.sim` module (§8). If Newton graduates from experimental to default status inside Isaac Sim and Isaac Lab, that would narrow the architectural gap with MJX this chapter describes in §11 — but Isaac Lab's own documentation currently declines to commit to official Newton support, so PhysX remaining the practical default for production work is the more likely near-to-medium trajectory. [Source](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/README.md)
- **`simulation_interfaces` standardisation path remains org-level rather than a numbered ROS REP.** The specification lives in the `ros-simulation` GitHub organisation as an evolving package rather than as an adopted, versioned ROS Enhancement Proposal. Broader multi-vendor adoption beyond Isaac Sim's own `isaacsim.ros2.sim_control` implementation (§5.2) is the plausible long-term path for a vendor-neutral simulator control surface, but no formal standards-track timeline is currently published. [Source](https://github.com/ros-simulation/simulation_interfaces)

---

## Integrations

**Chapter 69 (Omniverse, USD, Hydra, and RTX)** is the substrate for everything in this chapter and should be read first by anyone unfamiliar with Kit. Isaac Sim's extension loader, `.kit` experience files, and manifest format are Kit's (§9.1); its scenes are USD with Fabric/USDRT as the runtime mutation path (§9.2–9.3); its pixels come from the RTX Hydra delegate; its physics and kernel JIT are PhysX 5 and Warp (§12). Ch69 §10.2's headless-rendering discussion is the mechanism behind §7.4 of this chapter, and Ch69 §10.3's NGC container deployment is how Isaac Sim is actually shipped to cluster nodes — `ISAAC_SIM_IMAGE=nvcr.io/nvidia/isaac-sim:6.0.1 docker compose -f tools/docker/docker-compose.yml up` is the Isaac-specific incantation over that general mechanism [Source](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/tools/docker/README.md).

**Chapter 240 (Cosmos, OSMO, and Omniverse Farm)** owns the generative-model and orchestration side of the pipeline sketched in §10. Its §6.2 holds the GR00T-Mimic and GR00T-Dreams case study with the published trajectory-count, generation-time, and downstream-performance figures, which this chapter cites by reference rather than restating. Its §2.2 covers the NVIDIA Open Model License and OpenMDW-1.1 in the Cosmos context — relevant to §9.5's warning that those conclusions do *not* transfer to GR00T's currently-published weights. Ch240 §8 also points at Ch69 §10 for Isaac Sim and Isaac Lab headless operation, which §7.4 here refines with the Isaac Lab-specific flag semantics.

**Chapter 115 (NeRFStudio and Gaussian Splatting)** covers NuRec (`NVIDIA/instant-nurec`, `nurec-skills`, `harmonizer`) — a feed-forward 3D Gaussian reconstruction pipeline that converts driving-log sensor recordings into simulation-ready 3DGS scenes without per-scene optimisation. It is a third route into the synthetic-scene problem alongside the Mimic/Cosmos trajectory-multiplication pipelines in §10: rather than generating or replaying trajectories inside an authored USD scene, NuRec reconstructs the scene itself from recorded sensor data, and the result is consumed by Isaac Sim the same way any other RTX-rendered environment is. [Source: NVIDIA/instant-nurec, https://github.com/NVIDIA/instant-nurec]

**Chapter 211a (robotics simulation frameworks)** provides the comparative landscape — Gazebo, MuJoCo, PyBullet, and the ROS-ecosystem simulators — into which Isaac Sim slots. Where 211a's outline defers Isaac Sim's depth to "Ch69 and Ch240," this chapter is that depth; the MJX comparison in §11 here is grounded in MuJoCo's own documentation rather than forward-referencing 211a, and should be read as complementary to 211a's broader survey.

**Chapter 211 (multimedia and simulation frameworks)** situates this stack among the wider set of simulation and media frameworks running on Linux, and is the place to look for how simulation output is encoded, streamed, and consumed outside a training loop.

**Chapter 66 (CUDA and GPU compute)** underpins §7 entirely. `omni.physics.tensors` returns device tensors, Warp JIT-compiles Python to CUDA kernels, and the reason `replicate_physics=True` and `TiledCamera` matter is that both eliminate host-device round trips — the recurring cost model Ch66 develops in general terms.

Readers approaching this stack from the **graphics** side rather than the robotics side should note that §4's two-pipeline split is the load-bearing insight: RTX sensors are a rendering workload subject to every constraint the RTX chapters describe, whereas physics sensors are not a rendering workload at all, and conflating them leads to sizing a cluster for the wrong bottleneck.

---

## References

- [Isaac Sim repository, tag v6.0.1](https://github.com/isaac-sim/IsaacSim/tree/v6.0.1) — Isaac Sim source; extension layout under `source/extensions` and `source/deprecated` (§2)
- [Isaac Sim `LICENSE` (v6.0.1)](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/LICENSE) — Apache-2.0 grant scope and its exclusions for the Kit SDK runtime and bundled assets (§2.4)
- [Isaac Sim extension renaming guide (4.5.0 docs)](https://docs.isaacsim.omniverse.nvidia.com/4.5.0/overview/extensions_renaming.html) — authoritative `omni.isaac.*` to `isaacsim.*` mapping table (§2.1)
- [Isaac Sim 4.5.0 Release Notes](https://docs.isaacsim.omniverse.nvidia.com/4.5.0/overview/release_notes.html) — embedded Kit SDK updated 106.1.0 → 106.5.0 (§2.1)
- [Isaac Sim 6.0.1 Release Notes](https://docs.isaacsim.omniverse.nvidia.com/6.0.1/overview/release_notes.html) — embedded Kit SDK updated 110.1.1 → 110.1.2 (§2.1)
- [`ImuSensor.h` (v6.0.1)](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/deprecated/isaacsim.sensors.physics/include/isaacsim/sensors/physics/ImuSensor.h) — empty `tick()` and populated `onPhysicsStep()`, proving the IMU has no rendering path (§4.2)
- [`isaacsim.sensors.rtx` extension.py (v6.0.1)](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/deprecated/isaacsim.sensors.rtx/python/impl/extension.py) — `GenericModelOutput` AOV registration for RTX sensors (§4.1)
- [`Ros2Distro.h` (v6.0.1)](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/extensions/isaacsim.ros2.core/include/isaacsim/ros2/core/Ros2Distro.h) — compile-time enumeration of the two supported ROS 2 distributions (§5.1)
- [`isaacsim.ros2.sim_control` (v6.0.1)](https://github.com/isaac-sim/IsaacSim/tree/v6.0.1/source/extensions/isaacsim.ros2.sim_control) — Isaac Sim's implementation of the standardised ROS 2 `simulation_interfaces` package (§5.2)
- [IsaacSimZMQ](https://github.com/isaac-sim/IsaacSimZMQ) — the separate MIT-licensed ZeroMQ/Protobuf bridge repository, not part of Isaac Sim (§5.3)
- [Isaac Sim standalone IMU example (v6.0.1)](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/standalone_examples/api/isaacsim.sensors.experimental.physics/imu_sensor.py) — `SimulationApp` ordering constraint and the `IMU`/`IMUSensor` authoring/runtime split (§2.3, §4.2)
- [Isaac Sim Docker README (v6.0.1)](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/tools/docker/README.md) — NGC container invocation for headless cluster deployment (Integrations)
- [Isaac Lab repository](https://github.com/isaac-sim/IsaacLab) — Orbit-to-Isaac Lab rename redirect; project entry point (§3)
- [Isaac Lab branches](https://github.com/isaac-sim/IsaacLab/branches) — evidence that the default branch is `release/3.0.0-beta2` and 3.0 is not GA (§3)
- [Isaac Lab README (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/README.md) — Newton support status and the explicit absence of a support commitment (§8)
- [`cartpole_env_cfg.py` (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/source/isaaclab_tasks/isaaclab_tasks/manager_based/classic/cartpole/cartpole_env_cfg.py) — manager-based reward term configuration (§6)
- [`cartpole_env.py` (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/source/isaaclab_tasks/isaaclab_tasks/direct/cartpole/cartpole_env.py) — direct-workflow `_get_rewards` (§6)
- [`isaaclab_rl` wrappers (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/tree/release/3.0.0-beta2/source/isaaclab_rl) — the four supported RL libraries; no RLlib wrapper exists (§6.2)
- [Isaac Lab Ray scripts (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/tree/release/3.0.0-beta2/scripts/reinforcement_learning/ray) — Ray used for KubeRay orchestration and hyperparameter tuning, not as an RL algorithm provider (§6.2)
- [`LICENSE-mimic` (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/LICENSE-mimic) — root-level Apache-2.0 licence for the mimic component (§6.3)
- [Isaac Lab `.pre-commit-config.yaml` (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/.pre-commit-config.yaml) — dual `insert-license` hooks mechanically enforcing the BSD/Apache split (§6.3)
- [`physx_manager.py` (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/source/isaaclab_physx/isaaclab_physx/physics/physx_manager.py) — `omni.physics.tensors.create_simulation_view("warp", ...)` (§7.1)
- [`app_launcher.py` (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/blob/release/3.0.0-beta2/source/isaaclab/isaaclab/app/app_launcher.py) — `--headless` and `--enable_cameras` flag semantics (§7.4)
- [Isaac Lab camera sensors (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/tree/release/3.0.0-beta2/source/isaaclab/isaaclab/sensors/camera) — `TiledCamera` single-render-product implementation (§7.3)
- [Isaac Lab `isaaclab_mimic` (release/3.0.0-beta2)](https://github.com/isaac-sim/IsaacLab/tree/release/3.0.0-beta2/source/isaaclab_mimic) — `DataGenerator`, `Waypoint`, `WaypointSequence`, `WaypointTrajectory` (§10)
- [Isaac Lab performance benchmarks](https://isaac-sim.github.io/IsaacLab/main/source/refs/performance_benchmarks.html) — per-task environment counts and memory footprints; the vision-versus-proprioception cost split (§7.5)
- [Isaac Gym (deprecated)](https://developer.nvidia.com/isaac-gym) — deprecation notice directing users to Isaac Lab (§3)
- [Newton physics engine](https://github.com/newton-physics/newton) — Linux Foundation project on NVIDIA Warp; MuJoCo-Warp primary backend (§8)
- [Newton `LICENSE.md`](https://github.com/newton-physics/newton/blob/main/LICENSE.md) — Apache-2.0 for code, with CC-BY-4.0 documentation (§8)
- [Isaac-GR00T repository](https://github.com/NVIDIA/Isaac-GR00T) — model family, dual-system architecture, `PolicyClient` client/server inference, and the self-contradicting licence summary (§9)
- [Isaac-GR00T policy guide](https://github.com/NVIDIA/Isaac-GR00T/blob/main/getting_started/policy.md) — `Gr00tPolicy` construction and embodiment tag selection (§9.6)
- [Isaac-GR00T data preparation guide](https://github.com/NVIDIA/Isaac-GR00T/blob/main/getting_started/data_preparation.md) — the GR00T LeRobot format and `meta/modality.json` (§9.3)
- [MuJoCo XLA (MJX) documentation](https://mujoco.readthedocs.io/en/stable/mjx.html) — XLA hardware support, differentiability status, solver/integrator coverage, and the MJX-Warp batch renderer (§11)
- [MimicGen (NVlabs)](https://github.com/NVlabs/mimicgen) — the original implementation, distributed under an NVIDIA License rather than Apache-2.0 (§10)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
