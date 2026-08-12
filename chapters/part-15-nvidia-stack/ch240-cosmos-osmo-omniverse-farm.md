# Chapter 240: NVIDIA Cosmos, OSMO, and Omniverse Farm — Orchestrating Physical AI at Scale

> **Part**: Part XV — The NVIDIA Proprietary Graphics Stack
> **Audience**: Systems developers deploying GPU compute workloads on Linux/Kubernetes; ML engineers building robotics and autonomous-vehicle data pipelines; graphics application developers extending Omniverse/Kit pipelines toward synthetic-data generation
> **Status**: First draft — 2026-08-07

---

## Table of Contents

1. [Overview: Physical AI Needs a Different Kind of Farm](#1-overview-physical-ai-needs-a-different-kind-of-farm)
2. [NVIDIA Cosmos: World Foundation Models as Deployable Infrastructure](#2-nvidia-cosmos-world-foundation-models-as-deployable-infrastructure)
   - [2.1 The Model Family: Predict, Transfer, Reason, and Cosmos 3](#21-the-model-family-predict-transfer-reason-and-cosmos-3)
   - [2.2 Licensing: NVIDIA Open Model License and OpenMDW-1.1](#22-licensing-nvidia-open-model-license-and-openmdw-11)
   - [2.3 Deployment as NIM Microservices](#23-deployment-as-nim-microservices)
   - [2.4 Cosmos-Curate: GPU-Accelerated Data Curation on Ray](#24-cosmos-curate-gpu-accelerated-data-curation-on-ray)
3. [OSMO: Kubernetes-Native Orchestration for Physical AI](#3-osmo-kubernetes-native-orchestration-for-physical-ai)
   - [3.1 The Three-Computer Problem](#31-the-three-computer-problem)
   - [3.2 Architecture: Workflow Engine, Federation, and Smart Scheduling](#32-architecture-workflow-engine-federation-and-smart-scheduling)
   - [3.3 Defining a Workflow](#33-defining-a-workflow)
   - [3.4 Compute Backends and Plug-and-Play Registration](#34-compute-backends-and-plug-and-play-registration)
   - [3.5 OSMO as an Agentic Orchestrator](#35-osmo-as-an-agentic-orchestrator)
   - [3.6 License and Roadmap](#36-license-and-roadmap)
4. [Omniverse Farm: Job Dispatch for the Kit Ecosystem](#4-omniverse-farm-job-dispatch-for-the-kit-ecosystem)
   - [4.1 Farm Queue and Farm Agent](#41-farm-queue-and-farm-agent)
   - [4.2 Deployment Topologies](#42-deployment-topologies)
   - [4.3 Job Types](#43-job-types)
5. [OSMO vs. Omniverse Farm: Two Orchestrators, Two Scopes](#5-osmo-vs-omniverse-farm-two-orchestrators-two-scopes)
6. [The Physical AI Data Factory Blueprint](#6-the-physical-ai-data-factory-blueprint)
   - [6.1 Curate → Augment → Evaluate](#61-curate--augment--evaluate)
   - [6.2 Case Study: GR00T-Mimic and GR00T-Dreams](#62-case-study-gr00t-mimic-and-gr00t-dreams)
   - [6.3 Partner Ecosystem](#63-partner-ecosystem)
7. [Linux Deployment Specifics](#7-linux-deployment-specifics)
   - [7.1 GPU Scheduling: Device Plugin, GPU Operator, and Feature Discovery](#71-gpu-scheduling-device-plugin-gpu-operator-and-feature-discovery)
   - [7.2 Multi-Node Training Backends](#72-multi-node-training-backends)
   - [7.3 Farm Agent as a Linux Service](#73-farm-agent-as-a-linux-service)
8. [Integrations](#8-integrations)

---

## 1. Overview: Physical AI Needs a Different Kind of Farm

Chapter 42 §9.8 introduced a distinction worth restating precisely here: render farms such as Flamenco, OpenCue, and Deadline Cloud exist to dispatch Cycles/EEVEE frames across a worker fleet and collect the images back onto shared storage. NVIDIA's **Physical AI** stack — the software used to train robots, autonomous vehicles, and embodied agents — has an orchestration problem shaped similarly but scoped very differently: instead of one job type (render a frame) running on one kind of worker (a GPU with a renderer installed), a Physical AI pipeline moves data and compute across three fundamentally different execution environments — large-scale training clusters, physics/sensor simulation clusters, and edge hardware-in-the-loop devices — a framing NVIDIA calls the "Three Computer Problem" (§3.1). [Source](https://github.com/NVIDIA/OSMO)

This chapter is the deep dive that Chapter 42 §9.8 deliberately did not attempt, because Chapter 42's scope is Blender's render farms and this material belongs to NVIDIA's proprietary stack instead. Three products anchor it. **NVIDIA Cosmos** is a family of world foundation models (WFMs) that generate and evaluate physically plausible synthetic video — the data-generation half of the pipeline. **OSMO** is a Kubernetes-native workflow orchestrator purpose-built for the Three Computer Problem — the dispatcher that decides which cluster runs which stage. **Omniverse Farm** is a separate, older orchestration layer scoped specifically to dispatching Omniverse Kit application jobs (headless rendering, USD Composer batch processing, Replicator synthetic-data generation) across workstations, bare metal, VMs, or Kubernetes. Understanding why NVIDIA maintains two orchestrators rather than one, and how Cosmos's model family plugs into both, is the throughline of this chapter.

The chapter's technical architecture material is deliberately narrow. Cosmos's internal model architecture — tokenizers, diffusion/autoregressive decoders, the Slang/CUDA training and Vulkan inference split — is covered in depth in Chapter 117 §14.5 and is cross-referenced rather than repeated. Cosmos Transfer's ControlNet-style structured-signal conditioning technique is covered in Chapter 232 §8 and is likewise cross-referenced. This chapter's job is to document Cosmos, OSMO, and Omniverse Farm as **deployable infrastructure** — what license governs them, how they're packaged and run on Linux, how their APIs and workflow definitions are shaped, and how they compose into NVIDIA's end-to-end synthetic-data pipeline.

## 2. NVIDIA Cosmos: World Foundation Models as Deployable Infrastructure

### 2.1 The Model Family: Predict, Transfer, Reason, and Cosmos 3

NVIDIA announced the Cosmos world foundation model platform at CES 2025, with named early adopters including 1X, Figure AI, Uber, Skild AI, and XPeng. [Source](https://nvidianews.nvidia.com/news/nvidia-launches-cosmos-world-foundation-model-platform-to-accelerate-physical-ai-development) [Source](https://www.nvidia.com/en-us/ai/cosmos/) A March 2025 release substantially expanded the platform with Cosmos Transfer, Cosmos Predict, Cosmos Reason, and a NeMo Curator-based data-curation pipeline. [Source](https://nvidianews.nvidia.com/news/nvidia-announces-major-release-of-cosmos-world-foundation-models-and-physical-ai-data-tools) The family, as packaged for deployment via NIM (§2.3) as of this writing, breaks down as:

- **Cosmos Predict** — a generalist predictive-video model. Text2World and Video2World variants generate future frame sequences from a text, image, or video prompt, at 7B and 14B parameter sizes for Cosmos-Predict1, and as a unified 2B model (Cosmos-Predict2.5) supporting all three prompt modalities. [Source](https://docs.nvidia.com/nim/cosmos/latest/introduction.html) [Source](https://huggingface.co/blog/nvidia/cosmos-predict-and-transfer2-5)
- **Cosmos Transfer** — a multicontrol model that generates video conditioned on ground-truth simulation output or structured video inputs (segmentation, depth, edge, LiDAR/HD-map for the autonomous-vehicle variant), amplifying a limited set of real or simulated inputs into diverse environments, lighting conditions, and long-tail scenarios. Cosmos-Transfer2.5 is a lightweight 2B-parameter model with faster inference than the original Transfer1. [Source](https://docs.nvidia.com/nim/cosmos/latest/introduction.html) The structured-conditioning mechanism itself — how a ControlNet-style branch injects segmentation/depth/edge signal into the diffusion backbone — is documented in Chapter 232 §8 and is not repeated here.
- **Cosmos Reason** — a vision-language model (VLM) that evaluates video for physical plausibility and common-sense reasoning, rather than generating pixels itself. Cosmos Reason 2 is the version integrated into DeepStream 9.0's `nvvllmvlm` element (Chapter 59) and is the model that powers the Cosmos Evaluator stage of the Physical AI Data Factory Blueprint (§6.1).
- **Cosmos 3** — released May 31, 2026 as an "omnimodal" successor generation, including **Cosmos3-Generator**, a Mixture-of-Transformers model for text-to-video and image-to-video generation available at 8B ("nano") and 32B ("super") parameter sizes. [Source](https://research.nvidia.com/labs/cosmos-lab/cosmos3/technical-report.pdf)

Cosmos Predict's diffusion/autoregressive world-model decoder architecture, its tokenizer design, and the Slang-based differentiable-rendering training loop used for the visual loss term are documented in Chapter 117 §14.5 — including the specific note that model training remains CUDA/NCCL-first while only the inference decoder has a portable Vulkan/SPIR-V + `VK_KHR_cooperative_matrix` path. That split matters for this chapter's Linux deployment discussion in §7: the NIM containers described below are the inference-side deployment artifact, running on the same CUDA path as training, not the experimental Vulkan path.

### 2.2 Licensing: NVIDIA Open Model License and OpenMDW-1.1

Cosmos NIM microservices are available for commercial use under the **NVIDIA Open Model License**, and the underlying model weights, code, curated synthetic datasets, and evaluation benchmarks are released under **OpenMDW-1.1** (the Open Model, Data, and Weights license), a licensing framework stewarded by the Linux Foundation together with the PyTorch Foundation. [Source](https://docs.nvidia.com/nim/cosmos/latest/introduction.html) [Source](https://www.linuxfoundation.org/press/linux-foundation-releases-openmdw-1.1-nvidia-adopts-openmdw-for-cosmos-isaac-gr00t-ising-and-nemotron-ai-model-families)

OpenMDW-1.1 is a single legal framework covering models (architecture, weights, and parameters), code, documentation, and data, purpose-built for AI model distributions rather than adapted from a general-purpose software license. NVIDIA has stated an intent to adopt it across the Cosmos, Isaac GR00T, Ising, and Nemotron model families. The license has no field-of-use restrictions and no separate commercial-license requirement: developers can inspect, post-train, self-host, distill, modify, and deploy commercially under its terms alone. [Source](https://www.linuxfoundation.org/press/linux-foundation-releases-openmdw-1.1-nvidia-adopts-openmdw-for-cosmos-isaac-gr00t-ising-and-nemotron-ai-model-families) This is a meaningfully more permissive posture than a typical "open weights, restricted commercial use" model license, and it is a precondition for the render-farm-adjacent workflows described in §6: a studio or robotics team can fine-tune Cosmos Transfer on its own proprietary sensor data and redistribute the result internally without a separate commercial-licensing negotiation, for whichever model family has actually shipped weights under OpenMDW-1.1 terms.

> **Scope note.** This section describes OpenMDW-1.1 as it applies to the Cosmos family specifically, which is what the press release above documents. Do not extend this conclusion to GR00T: Chapter 211b §9.5 finds that OpenMDW-1.1 does not apply to any currently-published GR00T checkpoint — it has been described as the license for *future* GR00T releases in a company blog post only, not on any GR00T model card or in the corresponding newsroom release.

### 2.3 Deployment as NIM Microservices

NVIDIA distributes Cosmos models for inference as **NIM** (NVIDIA Inference Microservices) containers — one Docker container per model variant, pulled from the NGC container registry, each exposing a RESTful HTTP/gRPC interface for inference requests, health checks, and metadata retrieval, backed by Triton Inference Server. [Source](https://docs.nvidia.com/nim/cosmos/latest/introduction.html) NIM containers select an appropriate model profile automatically based on the local hardware configuration at startup.

```bash
# Pull and run the Cosmos-Predict1 7B Text2World NIM
# (illustrative — consult docs.nvidia.com/nim/cosmos for current image tags)
docker login nvcr.io   # username: $oauthtoken, password: your NGC API key

docker run --rm --gpus all \
    -e NGC_API_KEY=$NGC_API_KEY \
    -v /mnt/weights:/opt/nim/.cache \
    -p 8000:8000 \
    nvcr.io/nim/nvidia/cosmos-predict1-7b-text2world:1.0.0
```
*Source: [NVIDIA NIM for Cosmos WFM documentation](https://docs.nvidia.com/nim/cosmos/latest/introduction.html); standard NIM container launch pattern, confirmed against NGC container deployment conventions used across other NIM model families.*

The 14B variant runs the same way with explicit multi-GPU device selection (`--gpus '"device=0,1"'`) where a single GPU lacks sufficient VRAM. As of this writing, NIM for Cosmos WFM packages Cosmos-Predict1 and Cosmos-Transfer2.5; Cosmos-Reason1/Reason2 are distributed separately under NVIDIA NIM for Vision Language Models rather than under the Cosmos WFM NIM catalog, reflecting their different role (evaluation, not generation) in the pipeline described in §6. [Source](https://docs.nvidia.com/nim/cosmos/3.0.0/support-matrix.html)

### 2.4 Cosmos-Curate: GPU-Accelerated Data Curation on Ray

Before a world model can be trained or fine-tuned, raw video needs filtering, deduplication, and annotation at a scale that makes CPU-bound curation impractical. **Cosmos-Curate** is NVIDIA's open-sourced video curation system — the software that curates NVIDIA's own Cosmos training data — built as a GPU-accelerated streaming pipeline on top of **Ray**, and itself built on a lower-level streaming framework, **Cosmos-Xenna**, which NVIDIA has open-sourced as an independent project. [Source](https://github.com/nvidia-cosmos/cosmos-curate) Cosmos-Curate is the concrete implementation behind the "Cosmos Curator" stage named in the Physical AI Data Factory Blueprint (§6.1); the two names refer to the same system, with "Cosmos Curator" used in NVIDIA's blueprint marketing material and "Cosmos-Curate" as the GitHub repository and package name.

## 3. OSMO: Kubernetes-Native Orchestration for Physical AI

### 3.1 The Three-Computer Problem

OSMO's own framing for the problem it solves is the "Three Computer Problem": training a robot foundation model requires large-scale GPU training clusters, high-fidelity physics/sensor simulation clusters, and edge hardware-in-the-loop devices, and a real pipeline moves data and jobs between all three repeatedly — not as a one-time hand-off, but as an iterative loop (train → simulate → evaluate on real edge hardware → retrain). [Source](https://github.com/NVIDIA/OSMO) A generic Kubernetes scheduler treats every pod as a fungible unit of compute; it has no native concept of "this task needs an RTX Pro 6000 for ray-traced sensor simulation, this one needs 8×GB200 for training, and this one needs to run on a physical Jetson AGX Thor board sitting on a bench." OSMO is built specifically to express and schedule across that heterogeneity.

### 3.2 Architecture: Workflow Engine, Federation, and Smart Scheduling

OSMO is a Kubernetes-native job scheduler and workflow orchestrator, released as open source by NVIDIA. [Source](https://github.com/NVIDIA/OSMO) Its architecture centers on:

- **Workflow engine** — orchestrates task execution across multiple Kubernetes clusters with automatic dependency management and resource allocation, based on a declarative task graph (§3.3) rather than imperative orchestration scripts.
- **Multi-cluster federation** — coordinates task execution across geographically distributed compute backends without requiring the workflow author to specify infrastructure details in the task definition itself.
- **Smart scheduling** — routes each task to an appropriate backend based on declared platform requirements: simulation tasks to RTX Pro-class GPUs, training tasks to GB200-class clusters, edge-validation tasks to Jetson AGX Thor hardware.
- **Data pipeline integration** — manages data flow between dependent tasks via upstream task outputs or object storage (S3-compatible buckets), eliminating manual data-staging steps between pipeline stages.
- **Interactive development** — supports remote VSCode, SSH, and Jupyter sessions attached to a running task's compute allocation, for interactive debugging on cloud or cluster resources rather than only batch submission.

[Source](https://github.com/NVIDIA/OSMO)

### 3.3 Defining a Workflow

Workflows are authored as YAML task graphs rather than as Python orchestration scripts — OSMO's own stated design goal is "write workflows in YAML and iterate, not Python scripts." [Source](https://github.com/NVIDIA/OSMO) Each task declares a container image, a target platform, resource requirements, and its input dependencies on other tasks' outputs:

```yaml
# OSMO workflow: simulate, train, then validate on physical edge hardware
workflow:
  tasks:
  - name: simulation
    image: nvcr.io/nvidia/isaac-sim
    platform: rtx-pro-6000

  - name: train-policy
    image: nvcr.io/nvidia/pytorch
    platform: gb200
    resources:
      gpu: 8
    inputs:
    - task: simulation

  - name: evaluate-thor
    image: my-robot:latest
    platform: jetson-agx-thor
    inputs:
    - task: train-policy
    outputs:
    - url: s3://my-bucket/thor-benchmark/
```
*Source: [OSMO User Guide](https://nvidia.github.io/OSMO/main/user_guide/index.html)*

This single file expresses exactly the Three-Computer-Problem loop from §3.1: `simulation` runs on a workstation-class RTX GPU, its output feeds `train-policy` on a data-center GB200 cluster, and the trained policy is finally validated on a physical Jetson AGX Thor board, with results written to object storage. OSMO's CLI exposes this workflow lifecycle through subcommands including `osmo workflow` (submit and monitor), `osmo task` (inspect individual task state), `osmo pool` and `osmo resource` (query available compute), `osmo data` (dataset versioning and content-addressable storage), and `osmo credential`/`osmo login`/`osmo profile` for authentication against registered backends. [Source](https://nvidia.github.io/OSMO/main/user_guide/index.html)

### 3.4 Compute Backends and Plug-and-Play Registration

OSMO supports local development against Docker or KIND (Kubernetes-in-Docker), the major managed Kubernetes offerings (EKS on AWS, AKS on Azure, GKE on Google Cloud), custom on-premise Kubernetes clusters, and NVIDIA Jetson-class edge devices — with backends registered dynamically via CLI rather than requiring workflow YAML changes when infrastructure changes. [Source](https://github.com/NVIDIA/OSMO) Registration follows a "plug-and-play" model: an **OSMO operator** agent is deployed onto the target compute cluster and initiates an *outbound* connection back to the OSMO control plane to register itself, which lets clusters sit behind corporate firewalls or in restricted/NAT'd networks without inbound connectivity requirements — a materially different trust model than a control plane that must reach into a cluster directly. [Source](https://github.com/NVIDIA/OSMO)

### 3.5 OSMO as an Agentic Orchestrator

Beyond scheduling, OSMO is explicitly positioned as infrastructure for AI coding agents, not only for human operators. NVIDIA describes OSMO as delivering an "agent context file" that "turns your coding agent into a physical AI platform expert with full situational awareness over your development environment" — enabling a coding agent to reason about pipeline structure and task dependencies, query running workflows in real time, inspect current GPU capacity across registered backends, and monitor execution status, all without the agent needing bespoke integration code per cluster. [Source](https://developer.nvidia.com/osmo) The Physical AI Data Factory Blueprint (§6) states that OSMO "integrates with leading coding agents such as Claude Code, OpenAI Codex and Cursor, enabling AI-native operations where agents proactively manage resources, resolve bottlenecks and accelerate model delivery at scale." [Source](https://nvidianews.nvidia.com/news/nvidia-announces-open-physical-ai-data-factory-blueprint-to-accelerate-robotics-vision-ai-agents-and-autonomous-vehicle-development) In practice this means the same YAML-based workflow model in §3.3 is designed to be legible to, and modifiable by, an LLM-driven coding agent operating the pipeline — not just to a human infrastructure engineer.

### 3.6 License and Roadmap

OSMO is licensed under **Apache License 2.0**. [Source](https://github.com/NVIDIA/OSMO) NVIDIA's stated near-term roadmap (Q1 2026) adds OAuth 2.0 authentication, one-click cloud deployment, and native cloud IAM integration; further out, planned work includes Python-native workflow authoring APIs (as an alternative to hand-written YAML), load-aware scheduling across multiple backends simultaneously, execution result caching, and dynamic workflow scaling. [Source](https://github.com/NVIDIA/OSMO)

## 4. Omniverse Farm: Job Dispatch for the Kit Ecosystem

### 4.1 Farm Queue and Farm Agent

**Omniverse Farm** predates and is architecturally unrelated to OSMO — it is a job-dispatch layer scoped to the Omniverse Kit SDK ecosystem covered in Chapter 69, not a general Physical-AI workflow orchestrator. Farm is built as a microservice architecture around two service roles that share a common design rather than differing in implementation: **Farm Queue** receives and collects tasks submitted by users and holds the information that processing Agents need to execute them; **Farm Agent** executes tasks it is configured for, via job definitions, on a system with the appropriate resources. At a high level, Farm Queue maintains the list of pending tasks, and Agents equipped with the relevant capabilities poll it to find work. [Source](https://docs.omniverse.nvidia.com/farm/latest/index.html) Every Farm deployment requires exactly one running Farm Queue instance; every system available to execute tasks runs its own Farm Agent instance. [Source](https://docs.omniverse.nvidia.com/farm/latest/index.html)

### 4.2 Deployment Topologies

Farm is explicitly designed to be infrastructure-agnostic: it runs on typical workstations, bare-metal servers, virtual machines, and cloud/Kubernetes platforms alike, with documented deployment guides for standalone (single-machine) setups, Kubernetes clusters, and the major cloud providers (AWS, Azure, GCP, Oracle). [Source](https://docs.omniverse.nvidia.com/farm/latest/index.html) NVIDIA's own guidance is that, absent a hard requirement for bare-metal or VM deployment, running Farm on Kubernetes is the recommended path, since it gives better operational control and scalability than the standalone alternative.

### 4.3 Job Types

Farm's documented use cases are rendering, resource sharing across a team, task automation, asset preview generation, physics simulation, and USD scene creation for ML training pipelines. [Source](https://docs.omniverse.nvidia.com/farm/latest/index.html) The `create-render` job definition renders a USD stage using any Omniverse Kit application — USD Composer or a custom Kit application built from the Kit App Template — and is the mechanism through which Farm Agent actually invokes the headless `--no-window` Kit rendering path and NGC container deployment pattern documented in full in Chapter 69 §10.2–10.3; this chapter does not repeat that mechanism, only names Farm as the dispatcher that triggers it at scale. Omniverse Replicator synthetic-data-generation jobs — the labeled RGB/depth/segmentation output that feeds into Cosmos Transfer augmentation (§6.1) — are dispatched through the same Farm job-definition mechanism. [Source](https://developer.nvidia.com/blog/build-custom-synthetic-data-generation-pipelines-with-omniverse-replicator/)

## 5. OSMO vs. Omniverse Farm: Two Orchestrators, Two Scopes

It is a reasonable question why NVIDIA maintains two separate orchestration systems rather than one. The answer is scope, not redundancy:

| | **Omniverse Farm** | **OSMO** |
|---|---|---|
| Scope | Omniverse/Kit application jobs only | General Physical AI workflow orchestration |
| Job unit | A Kit application invocation (render, Replicator capture, physics sim) | Any containerized task on any registered backend |
| Compute model | Single-cluster/single-topology per deployment | Multi-cluster federation, heterogeneous platforms in one workflow |
| Cross-stage dependencies | Not native — one job type per invocation | Native task graph (`inputs: - task: ...`) spanning training, simulation, and edge |
| Edge/robot hardware | Not a target | First-class platform target (Jetson AGX Thor) |
| Coding-agent integration | Not documented | Explicit "agent context file" design goal |
| License | Proprietary (part of Omniverse) | Apache 2.0, open source |

A workflow that only needs to render USD stages or capture Replicator synthetic data on a render/compute farm fits Omniverse Farm's job model directly and needs nothing more. A workflow that spans "generate simulation data, train a policy on a different cluster class entirely, then validate on physical edge hardware, with Cosmos-based augmentation somewhere in between" is exactly the case Farm was never built for and OSMO was — which is precisely the shape of the Physical AI Data Factory Blueprint pipeline in §6, where OSMO is the orchestrator and Omniverse Replicator/Isaac Sim (dispatched via Farm, or directly) is one of several data sources it coordinates.

## 6. The Physical AI Data Factory Blueprint

### 6.1 Curate → Augment → Evaluate

At GTC 2026 (March 16, 2026), NVIDIA announced the **Physical AI Data Factory Blueprint**, an open reference architecture that unifies and automates training-data generation, augmentation, and evaluation for physical AI systems, aiming to reduce the cost, time, and complexity of training robotics, autonomous-vehicle, and vision-AI-agent models at scale. [Source](https://nvidianews.nvidia.com/news/nvidia-announces-open-physical-ai-data-factory-blueprint-to-accelerate-robotics-vision-ai-agents-and-autonomous-vehicle-development) It is built on Cosmos (§2) and orchestrated by OSMO (§3), and composes into three stages:

1. **Curate** — Cosmos-Curate (§2.4) processes, refines, and annotates large-scale real-world and synthetic video datasets.
2. **Augment** — Cosmos Transfer (§2.1) exponentially expands and diversifies the curated data, multiplying real and simulated inputs to capture rare and long-tail scenarios across environments and lighting conditions that would be impractical to collect by hand.
3. **Evaluate** — Cosmos Evaluator, powered by Cosmos Reason (§2.1) and available on GitHub, automatically scores, verifies, and filters the generated data for physical accuracy and training readiness, rejecting synthetic samples that violate physical plausibility before they ever reach a training run.

[Source](https://nvidianews.nvidia.com/news/nvidia-announces-open-physical-ai-data-factory-blueprint-to-accelerate-robotics-vision-ai-agents-and-autonomous-vehicle-development)

OSMO's role in this specific pipeline is exactly the multi-backend workflow orchestration described in §3: it manages distributed compute resources and coordinates the stages of the training workflow, and — per §3.5 — can integrate with coding agents that monitor infrastructure usage and automate operational tasks across the pipeline. [Source](https://nvidianews.nvidia.com/news/nvidia-announces-open-physical-ai-data-factory-blueprint-to-accelerate-robotics-vision-ai-agents-and-autonomous-vehicle-development) The blueprint was expected to be available on GitHub in April 2026, with the Cosmos-Curate and Cosmos Evaluator components already accessible as independent GitHub repositories ahead of the full blueprint release. [Source](https://nvidianews.nvidia.com/news/nvidia-announces-open-physical-ai-data-factory-blueprint-to-accelerate-robotics-vision-ai-agents-and-autonomous-vehicle-development)

### 6.2 Case Study: GR00T-Mimic and GR00T-Dreams

**Isaac GR00T N1**, described by NVIDIA as the world's first open, fully customizable foundation model for generalized humanoid robot reasoning and skills, is the clearest concrete example of this pipeline's payoff. [Source](https://nvidianews.nvidia.com/news/nvidia-isaac-gr00t-n1-open-humanoid-robot-foundation-model-simulation-frameworks) GR00T N1 was trained on a combination of teleoperated robot data, synthetic data generated via the GR00T Blueprint, and human demonstration video from the internet. Chapter 211b §9 covers the GR00T model family's own architecture — its dual-system design, checkpoint lineage, and fine-tuning surface — in depth; this section stays scoped to the data-generation pipeline that feeds it.

**GR00T-Mimic** is the reference workflow for the synthetic-data half of that mix: built on Omniverse and Cosmos Transfer 1, it generates large volumes of synthetic robot-manipulation motion trajectories from a small number of human demonstrations, using domain randomization and 3D upscaling to multiply a handful of real demonstrations into a much larger, more diverse training set. [Source](https://github.com/NVIDIA-Omniverse-blueprints/synthetic-manipulation-motion-generation) In one reported run, this workflow generated 780,000 synthetic trajectories — equivalent to roughly 6,500 hours, or nine continuous months, of human demonstration data — in 11 hours. [Source](https://developer.nvidia.com/blog/enhance-robot-learning-with-synthetic-trajectory-data-generated-by-world-foundation-models) **GR00T-Dreams** extends this with an evaluation loop: it uses Cosmos Predict2 as the world model generating candidate synthetic trajectories, and Cosmos Reason 2 to evaluate whether each generated trajectory adheres to physical laws before it is accepted into the training set — an end-to-end, smaller-scale instance of exactly the Curate/Augment/Evaluate shape described in §6.1. [Source](https://nvidia-cosmos.github.io/cosmos-cookbook/recipes/end2end/gr00t-dreams/post-training.html) Combining this synthetic data with real teleoperation data produced a 40% performance improvement for GR00T N1 compared to training on real data alone. [Source](https://developer.nvidia.com/blog/enhance-robot-learning-with-synthetic-trajectory-data-generated-by-world-foundation-models)

### 6.3 Partner Ecosystem

Cloud infrastructure partners integrating the Physical AI Data Factory Blueprint at launch include Microsoft Azure, which is incorporating the blueprint into its own open toolchain, and Nebius, which has integrated OSMO directly into its AI Cloud offering. Named early Physical AI developer adopters include FieldAI, Hexagon Robotics, Linker Vision, Milestone Systems, Skild AI, Teradyne Robotics, Uber, RoboForce, and Voxel51. [Source](https://nvidianews.nvidia.com/news/nvidia-announces-open-physical-ai-data-factory-blueprint-to-accelerate-robotics-vision-ai-agents-and-autonomous-vehicle-development)

## 7. Linux Deployment Specifics

### 7.1 GPU Scheduling: Device Plugin, GPU Operator, and Feature Discovery

Both OSMO and any Kubernetes-hosted Omniverse Farm deployment rely on the standard Kubernetes GPU-scheduling stack, not a bespoke NVIDIA scheduling mechanism of their own. The **NVIDIA device plugin for Kubernetes** (`k8s-device-plugin`) runs as a DaemonSet on each GPU node and exposes GPUs as a schedulable Kubernetes resource (`nvidia.com/gpu`), acting as the translation layer that lets `kubectl` and the Kubernetes scheduler track, allocate, and manage GPU resources the same way they manage CPU or memory. [Source](https://github.com/nvidia/k8s-device-plugin) The **NVIDIA GPU Operator** automates deployment of this device plugin together with the NVIDIA driver, the NVIDIA Container Toolkit, and monitoring components, so that a bare Kubernetes cluster becomes GPU-aware without hand-installing each piece per node. [Source](https://developer.nvidia.com/blog/nvidia-gpu-operator-simplifying-gpu-management-in-kubernetes/) **GPU Feature Discovery (GFD)** inspects each node's GPUs and automatically applies labels describing GPU model, memory size, and MIG capability to the corresponding Kubernetes node object; OSMO's "smart scheduling" (§3.2) — routing a simulation task to an RTX Pro node and a training task to a GB200 node from the same workflow YAML — is implemented on top of exactly these node labels and Kubernetes node-selector/affinity mechanics, not a separate NVIDIA-specific scheduling API.

The **MPS** and **MIG** multi-tenant GPU-sharing mechanisms documented in Chapter 66 §7 apply directly inside OSMO- or Farm-scheduled pods: a Kubernetes pod requesting a MIG-partitioned GPU instance (`nvidia.com/mig-1g.10gb`, for example) behaves identically whether the pod was scheduled by raw `kubectl apply`, by an OSMO workflow task, or by a Farm Agent's Kubernetes-hosted job — the scheduling layer differs, the underlying GPU-partitioning mechanism does not.

### 7.2 Multi-Node Training Backends

The `train-policy` task in the example OSMO workflow (§3.3) requesting 8 GPUs on a `gb200` platform is, underneath OSMO's scheduling layer, an ordinary multi-GPU (and likely multi-node, given GB200's NVLink-domain scale) PyTorch training job. The collective-communications mechanics — **NCCL**'s `ncclCommInitAll`/`ncclCommInitRank` initialization, the ring-allreduce and tree-allreduce algorithms, and transport selection across NVLink, PCIe peer-to-peer, and InfiniBand — are documented in full in Chapter 66 §18 and apply without modification to any Cosmos fine-tuning or GR00T policy-training job that OSMO dispatches to a multi-node backend; OSMO's contribution is getting the right containers onto the right nodes with the right topology, not replacing NCCL as the communication layer.

This is not hypothetical for Isaac Lab specifically, and it is worth being precise about which layer does what: Isaac Lab supports exactly four RL libraries for the learning algorithm itself — RSL-RL, RL-Games, skrl, and Stable-Baselines3 (Chapter 211b §6.2) — none of which is Ray RLlib. Ray's actual role in Isaac Lab is KubeRay-based cluster orchestration and hyperparameter sweeps: launching and managing many training runs, not learning inside any one of them (Chapter 211b §6.2). That puts KubeRay and OSMO in adjacent, overlapping territory — both are Kubernetes-native launchers for many-run Isaac Lab training jobs — with OSMO additionally handling the cross-cluster simulate/train/validate handoff (§3.3) that a KubeRay sweep confined to one training cluster does not. Within whichever launcher places the job, NCCL (as above) still handles the cross-GPU collective communication.

### 7.3 Farm Agent as a Linux Service

Where OSMO backends are Kubernetes-native by design, Omniverse Farm's standalone deployment mode (§4.2) targets a bare-metal or VM Linux host directly: the Farm Agent process runs as a long-lived service on each execution host — typically managed via `systemd` on a dedicated render/simulation node — polling the Farm Queue for work rather than being scheduled per-job by Kubernetes. On a Kubernetes-hosted Farm deployment, by contrast, Farm Agent instances run as Kubernetes-managed pods (a DaemonSet or a scaled Deployment, depending on whether every node should run an Agent or only a subset), which is the deployment NVIDIA itself recommends when there is no hard requirement for bare-metal or VM hosting (§4.2).

## 8. Integrations

**Chapter 42 (Blender GPU Rendering) §9.8 — AI World Models and Render Farm Integration.** That section's brief overview of Cosmos, OSMO, and Omniverse Farm — framed around whether they intersect with Flamenco/OpenCue/Deadline Cloud render-farm dispatch — is superseded by this chapter's full treatment. The conclusion there stands: the render-farm ecosystem and the Physical-AI/OSMO ecosystem are separate orchestration stacks with no documented direct integration, and readers wanting the render-farm side of that boundary (Flamenco, OpenCue, Deadline Cloud) should consult Chapter 42 §9.1–§9.7.

**Chapter 117 (Slang — Differentiable and Modular Shading Language) §14.5.** Cosmos's internal model architecture — the tokenizer, the diffusion/autoregressive world-model decoder, the Slang-based differentiable rendering pipeline used to compute a visual training loss, and the CUDA-training-vs-Vulkan-inference split — is documented there in depth and is assumed, not re-derived, by this chapter's §2.

**Chapter 232 (GPU Generative AI and LLM Inference) §8.** Cosmos Transfer's ControlNet-style structured-signal conditioning technique (segmentation/depth/edge/LiDAR-HD-map inputs steering video-diffusion generation) is documented there, alongside a comparison to raw-pixel world models (GameNGen, DIAMOND, DeepMind Genie 2) that skip structured-buffer conditioning entirely.

**Chapter 59 (DeepStream) and Part XIII intro.** Cosmos Reason 2's integration into DeepStream 9.0 via the `nvvllmvlm` element is documented there; this chapter's §2.1 and §6.1 describe the same Cosmos Reason model in its role as the Physical AI Data Factory Blueprint's Evaluator stage, applied to offline synthetic-data filtering rather than DeepStream's live video-analytics pipeline.

**Chapter 69 (NVIDIA Omniverse, OpenUSD, and the RTX Renderer) §10.** The headless Kit `--no-window` rendering path, NGC container deployment pattern, and Isaac Lab/Isaac Sim headless operation documented there is the exact mechanism Omniverse Farm's `create-render` job definition (§4.3) invokes at scale; this chapter names Farm as the dispatcher, Chapter 69 documents what gets dispatched.

**Chapter 211b (NVIDIA Isaac Sim, Isaac Lab, and the GR00T Foundation-Model Family).** This chapter's §6.2 GR00T-Mimic/GR00T-Dreams case study assumes Ch211b's Isaac Sim/Isaac Lab/GR00T architecture depth rather than re-deriving it; Ch211b §9.5's GR00T licensing findings, in turn, are the correction to this chapter's §2.2 OpenMDW-1.1 discussion — the two sections should be read together on that point.

**Chapter 66 (CUDA Runtime, Streams, and NVRTC) §7 and §18.** MPS/MIG multi-tenant GPU partitioning and NCCL multi-node collective communications, both used without modification inside OSMO- or Farm-scheduled Kubernetes pods, as detailed in §7.1–§7.2 of this chapter.

**Chapter 211 (ROS 2 Multimodal Sensor and Perception Pipeline).** Real-world sensor data ingested into the Curate stage of the Physical AI Data Factory Blueprint (§6.1) — camera, LiDAR, and other sensor streams — typically originates from ROS 2 bag recordings on physical robot or vehicle platforms; that chapter's coverage of ROS 2 sensor pipelines and data formats is the upstream data source this chapter's pipeline consumes.

**Chapter 25 (GPU Compute).** The general Linux GPU-compute scheduling and memory-management fundamentals underlying every container this chapter dispatches, whether via OSMO or Omniverse Farm.

---

*References used in this chapter:*

- [NVIDIA Launches Cosmos World Foundation Model Platform to Accelerate Physical AI Development](https://nvidianews.nvidia.com/news/nvidia-launches-cosmos-world-foundation-model-platform-to-accelerate-physical-ai-development) — NVIDIA Newsroom, CES 2025
- [NVIDIA Announces Major Release of Cosmos World Foundation Models and Physical AI Data Tools](https://nvidianews.nvidia.com/news/nvidia-announces-major-release-of-cosmos-world-foundation-models-and-physical-ai-data-tools) — NVIDIA Newsroom, March 2025
- [NVIDIA Cosmos: World Foundation Models Powering Physical AI](https://www.nvidia.com/en-us/ai/cosmos/) — NVIDIA product page
- [Cosmos 3: Omnimodal World Models for Physical AI — Technical Report](https://research.nvidia.com/labs/cosmos-lab/cosmos3/technical-report.pdf) — NVIDIA Research, 2026
- [NVIDIA NIM for Cosmos World Foundation Models — Introduction](https://docs.nvidia.com/nim/cosmos/latest/introduction.html) — NVIDIA docs
- [NVIDIA NIM for Cosmos WFM — Supported Models](https://docs.nvidia.com/nim/cosmos/3.0.0/support-matrix.html) — NVIDIA docs
- [Cosmos Predict 2.5 & Transfer 2.5: Evolving the World Foundation Models for Physical AI](https://huggingface.co/blog/nvidia/cosmos-predict-and-transfer2-5) — NVIDIA/HuggingFace
- [Linux Foundation Releases OpenMDW-1.1; NVIDIA Adopts OpenMDW for Cosmos, Isaac GR00T, Ising and Nemotron AI Model Families](https://www.linuxfoundation.org/press/linux-foundation-releases-openmdw-1.1-nvidia-adopts-openmdw-for-cosmos-isaac-gr00t-ising-and-nemotron-ai-model-families) — Linux Foundation
- [nvidia-cosmos/cosmos-curate](https://github.com/nvidia-cosmos/cosmos-curate) — GitHub
- [NVIDIA/OSMO](https://github.com/NVIDIA/OSMO) — GitHub, Apache 2.0
- [OSMO User Guide](https://nvidia.github.io/OSMO/main/user_guide/index.html) — NVIDIA
- [OSMO Platform](https://developer.nvidia.com/osmo) — NVIDIA Developer
- [NVIDIA Announces Open Physical AI Data Factory Blueprint to Accelerate Robotics, Vision AI Agents and Autonomous Vehicle Development](https://nvidianews.nvidia.com/news/nvidia-announces-open-physical-ai-data-factory-blueprint-to-accelerate-robotics-vision-ai-agents-and-autonomous-vehicle-development) — NVIDIA Newsroom, GTC 2026
- [Omniverse Farm Documentation](https://docs.omniverse.nvidia.com/farm/latest/index.html) — NVIDIA
- [Build Custom Synthetic Data Generation Pipelines with Omniverse Replicator](https://developer.nvidia.com/blog/build-custom-synthetic-data-generation-pipelines-with-omniverse-replicator/) — NVIDIA Developer Blog
- [NVIDIA Announces Isaac GR00T N1 — the World's First Open Humanoid Robot Foundation Model — and Simulation Frameworks to Speed Robot Development](https://nvidianews.nvidia.com/news/nvidia-isaac-gr00t-n1-open-humanoid-robot-foundation-model-simulation-frameworks) — NVIDIA Newsroom
- [NVIDIA-Omniverse-blueprints/synthetic-manipulation-motion-generation](https://github.com/NVIDIA-Omniverse-blueprints/synthetic-manipulation-motion-generation) — GitHub (GR00T-Mimic)
- [Enhance Robot Learning With Synthetic Trajectory Data Generated by World Foundation Models](https://developer.nvidia.com/blog/enhance-robot-learning-with-synthetic-trajectory-data-generated-by-world-foundation-models) — NVIDIA Developer Blog
- [Leveraging World Foundation Models for Synthetic Trajectory Generation in Robot Learning (GR00T-Dreams)](https://nvidia-cosmos.github.io/cosmos-cookbook/recipes/end2end/gr00t-dreams/post-training.html) — Cosmos Cookbook
- [NVIDIA/k8s-device-plugin](https://github.com/nvidia/k8s-device-plugin) — GitHub
- [NVIDIA GPU Operator: Simplifying GPU Management in Kubernetes](https://developer.nvidia.com/blog/nvidia-gpu-operator-simplifying-gpu-management-in-kubernetes/) — NVIDIA Developer Blog

## Roadmap

### Near-term (6–12 months)
- **OSMO OAuth 2.0 and one-click cloud deployment.** NVIDIA's stated Q1 2026 target adds OAuth 2.0 authentication and native cloud IAM integration to OSMO, removing a current operational gap for enterprise multi-tenant deployments (§3.6).
- **Physical AI Data Factory Blueprint general availability on GitHub.** The blueprint was expected in April 2026; Cosmos-Curate and Cosmos Evaluator were already independently available ahead of that date (§6.1), with the full orchestrated blueprint (including OSMO workflow definitions) landing after.
- **Cosmos 3 NIM packaging.** Cosmos3-Generator (8B/32B) was announced as a research release in May 2026; NIM containerization following the Cosmos-Predict1/Transfer2.5 pattern (§2.3) is a plausible near-term follow-on, though not yet confirmed in NIM documentation as of this writing.

### Medium-term (1–3 years)
- **OSMO Python-native workflow APIs.** Planned as an alternative to hand-authored YAML (§3.3), lowering the barrier for programmatic workflow generation — particularly relevant to the agentic-orchestrator use case in §3.5, where a coding agent composing a workflow programmatically is a more natural fit than generating YAML text.
- **Load-aware multi-backend scheduling.** OSMO's roadmap includes scheduling that accounts for real-time load across multiple registered backends simultaneously, rather than routing purely by static platform-requirement matching as in the current `platform: gb200`-style task declaration (§3.3).
- **Convergence pressure between Farm and OSMO scope.** As OSMO's coding-agent integration (§3.5) matures and Omniverse Kit jobs increasingly participate in larger multi-cluster Physical AI pipelines, there is a plausible medium-term case for OSMO workflows to directly target Omniverse Farm as one of its pluggable backends (§3.4) rather than the two systems remaining fully separate, though no such integration is currently documented.

### Long-term
- **OpenMDW-1.1 as the default license posture across NVIDIA's open model families.** Its adoption for Cosmos, Isaac GR00T, Ising, and Nemotron (§2.2) suggests NVIDIA intends it as the standard going forward for future Physical AI model releases, reducing the licensing-diligence burden for downstream integrators of this chapter's stack.
- **Broader industry adoption of the Three-Computer-Problem orchestration pattern.** If OSMO's open-source Apache 2.0 model succeeds outside NVIDIA's own stack, the heterogeneous training/simulation/edge scheduling pattern it embodies (§3.1–§3.2) is a plausible template other robotics and autonomous-vehicle organizations converge on, independent of whether they adopt Cosmos specifically for the generative half of the pipeline.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
