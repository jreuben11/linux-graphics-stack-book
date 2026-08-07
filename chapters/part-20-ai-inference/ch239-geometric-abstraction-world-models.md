# Chapter 239: Geometric Abstraction for World-Model Visual Reasoning

> **Part**: Part XX — AI Inference & Neural Rendering
> **Audience**: AI/ML engineers building embodied and robotics world models, graphics application developers extending Gaussian-splat pipelines toward simulation, systems developers evaluating differentiable-physics toolchains on Linux
> **Status**: First draft — 2026-08-07

---

> **A note on currency.** This chapter surveys a fast-moving research area. Several primary sources cited below are preprints from the first half of 2026, some describing specifications (KHR_gaussian_splatting) that were still Release Candidates — not ratified — at the time of writing. Where a claim rests on a single preprint's self-reported numbers rather than an independently reproduced result, this is noted explicitly. Readers should re-check ratification and version status against the linked sources before depending on them in production.

## Table of Contents

1. [Overview: Two Ways to Read a Scene](#1-overview-two-ways-to-read-a-scene)
2. [Why "Optimal" Is Not Well-Posed](#2-why-optimal-is-not-well-posed)
3. [Empirical Priors: What the Data Actually Shows](#3-empirical-priors-what-the-data-actually-shows)
4. [The Stack: A Layered Architecture for Geometric World Models](#4-the-stack-a-layered-architecture-for-geometric-world-models)
   - [4.1 Layer 0 — Substrate: 3D and 4D Gaussian Splatting](#41-layer-0--substrate-3d-and-4d-gaussian-splatting)
   - [4.2 Layer 1 — Part and Articulation Decomposition](#42-layer-1--part-and-articulation-decomposition)
   - [4.3 Layer 2 — Persistence Through Occlusion](#43-layer-2--persistence-through-occlusion)
   - [4.4 Layer 3 — Kinematic and Contact Scene Graphs](#44-layer-3--kinematic-and-contact-scene-graphs)
   - [4.5 Layer 4 — Per-Part Material and Physical Parameter Inference](#45-layer-4--per-part-material-and-physical-parameter-inference)
   - [4.6 Layer 5 — Executable and Latent Dynamics World Models](#46-layer-5--executable-and-latent-dynamics-world-models)
5. [The Gradient Problem](#5-the-gradient-problem)
   - [5.1 Representational Misalignment](#51-representational-misalignment)
   - [5.2 Contact Non-Smoothness](#52-contact-non-smoothness)
   - [5.3 Where the Gradient Enters](#53-where-the-gradient-enters)
   - [5.4 Parameter Identifiability](#54-parameter-identifiability)
6. [Storage, Loading, and Interchange](#6-storage-loading-and-interchange)
   - [6.1 KHR_gaussian_splatting and SPZ Compression](#61-khr_gaussian_splatting-and-spz-compression)
   - [6.2 UsdPhysics, SimReady, and the AOUSD Robot-Description Mapping](#62-usdphysics-simready-and-the-aousd-robot-description-mapping)
   - [6.3 MCAP and LeRobot: Trajectory and Episode Data](#63-mcap-and-lerobot-trajectory-and-episode-data)
7. [Implementation Toolchain](#7-implementation-toolchain)
   - [7.1 Differentiable Simulation: Warp, Newton, Genesis, and MJX](#71-differentiable-simulation-warp-newton-genesis-and-mjx)
   - [7.2 Geometric Foundation Models: Pointmap Regression](#72-geometric-foundation-models-pointmap-regression)
   - [7.3 3D Data Structures and Differentiable Rendering](#73-3d-data-structures-and-differentiable-rendering)
   - [7.4 Segmentation and Semantic Priors](#74-segmentation-and-semantic-priors)
   - [7.5 Collision Geometry and Serialization](#75-collision-geometry-and-serialization)
8. [Open Problems](#8-open-problems)
9. [Integrations](#9-integrations)

---

## 1. Overview: Two Ways to Read a Scene

Give a world model a video and ask it to predict what happens next, and there are two broad families of answer for how it should represent what it just saw. One family says: recover explicit 3D geometry — positions, shapes, joints, masses — and predict the future by simulating physics over that geometry. The other says: let a large video or latent-state model learn whatever internal representation minimizes prediction error, without committing to geometry as an intermediate target at all.

The second position has a serious existence proof. Meta's V-JEPA 2 is trained self-supervised on more than a million hours of internet video plus under 62 hours of robot interaction data, and reaches competitive planning and manipulation performance without ever being told to output a point cloud, a mesh, or a joint angle [Source](https://arxiv.org/abs/2506.09985). Its objective is prediction in a learned latent space, not geometric reconstruction. If a video model can plan a robot arm's grasp without an explicit 3D representation anywhere in its pipeline, the case for building geometry-heavy world models needs to answer a real objection, not a straw one.

This chapter takes the position that framing this as "geometry vs. no geometry" is itself the error — hence the section title below. The rest of the chapter is a survey of the concrete layered architecture that results when geometry *is* chosen as the substrate: 3D/4D Gaussian Splatting as a differentiable, photorealistic scene representation; a stack of methods that decompose that representation into parts, joints, materials, and persistent object identities; the differentiable-physics gradient problem that appears the moment you try to *simulate* through that representation rather than merely render it; the interchange formats now standardizing around it (glTF's `KHR_gaussian_splatting`, OpenUSD's `UsdPhysics`); and the toolchain that ties it together on Linux GPU hardware — much of which (`gsplat`, NVIDIA Warp, Open3D/PyTorch3D/Kaolin) is already documented elsewhere in this book (Chapter 115, Chapter 212) and is referenced rather than re-derived here.

## 2. Why "Optimal" Is Not Well-Posed

"Should a world model use explicit geometry?" presupposes a fixed evaluation criterion against which "explicit geometry" and "no geometry" can be ranked. In practice the criterion itself is underspecified, and the ranking flips depending on what you hold fixed.

A 2026 position paper proposes resolving this by scoring world models against an explicit ladder of decision-making utility (an "L0–L7" framework) rather than against visual fidelity metrics like PSNR or FID, arguing that a world model's representation should be judged by how well it serves downstream policy optimization and planning, with a minimal reporting set defined for real-robot evaluation [Source](https://arxiv.org/abs/2606.15032). This is a *value-equivalence* framing: a representation is "good" not because it looks realistic, but because a planner or policy that consumes it makes decisions as good as if it had ground-truth state — the same principle that underlies value-equivalent model-based RL more broadly. Under this framing, "geometry vs. latent" isn't a single question; it's a family of questions, one per downstream task, and the answer can differ for grasp planning, long-horizon manipulation, and navigation even on the same robot.

A companion piece of evidence is architectural rather than philosophical: a controlled study of object-centric world models trained unsupervised from pixels (the DLPWM architecture) found strong reconstruction and out-of-distribution robustness, but policies trained on top of that representation underperformed a strong latent baseline (DreamerV3) on actual control — attributed to "representation shift during multi-object interactions" destabilizing policy learning even when the representation itself looked qualitatively good [Source](https://arxiv.org/abs/2511.06136). The lesson generalizes past this one architecture: a world model's *representation quality*, measured by reconstruction or visual metrics, does not automatically transfer to *policy quality* downstream. Any claim that explicit geometry (or its absence) is "better" needs to specify better *for what task*, evaluated *through what downstream consumer* — which is precisely what the L0–L7 framing above tries to force explicit.

## 3. Empirical Priors: What the Data Actually Shows

Rather than adjudicate the framing question from first principles, it is worth looking at what controlled experiments actually say about observation space and geometric priors for robot learning specifically.

**OBSBench** ran a head-to-head comparison of RGB, RGB-D, and point-cloud observation spaces across two simulators and 125 manipulation tasks, training policies from scratch as well as from pretrained visual representations for each modality. The finding: point-cloud-based policies "frequently outperform" their RGB and RGB-D counterparts, both from-scratch and pretrained [Source](https://arxiv.org/abs/2402.02500). This is a data point in favor of explicit geometric observation spaces for policy learning specifically — not a resolution of the philosophical question, but empirical evidence that, at least for the manipulation tasks in this benchmark, throwing away 3D structure and working from raw pixels costs something measurable.

That evidence sits alongside a rapidly maturing family of **geometric foundation models** that make recovering that 3D structure from ordinary video cheap. DUSt3R demonstrated that a single feed-forward network, given two uncalibrated images, can regress a dense 3D pointmap without any separate camera-calibration or bundle-adjustment stage [Source](https://arxiv.org/abs/2312.14132). MASt3R extends this with a local-feature descriptor head trained with an InfoNCE matching loss and metric-scale pointmap output, turning DUSt3R's stereo reconstruction into a model also capable of robust image matching [Source](https://github.com/naver/mast3r). VGGT (Visual Geometry Grounded Transformer) generalizes the same feed-forward idea to arbitrary numbers of views in a single pass [Source](https://arxiv.org/abs/2503.11651), and π³ removes the fixed-reference-view inductive bias that DUSt3R-family models inherit, via a permutation-equivariant architecture [Source](https://arxiv.org/abs/2507.13347). Microsoft's MoGe-2 takes the single-image case further, recovering *metric-scale* point maps, depth, normals, and field of view from one photograph via a synthetic-data-guided refinement pipeline [Source](https://arxiv.org/abs/2507.02546). A dedicated benchmark, E3D-Bench, now standardizes evaluation of this whole model family (depth estimation, reconstruction, pose estimation, novel view synthesis) across 16 state-of-the-art geometric foundation models [Source](https://arxiv.org/abs/2506.01933).

Together, these two threads — a controlled preference for point-cloud observation spaces in policy learning, and pointmap regression models that make 3D reconstruction from arbitrary uncalibrated video essentially free at inference time — form the empirical case for building the geometric stack described in the rest of this chapter, without needing to claim it is universally superior to latent-only approaches like V-JEPA 2.

## 4. The Stack: A Layered Architecture for Geometric World Models

If geometry is the chosen substrate, it does not arrive as a single monolithic representation. A working geometric world model is better understood as a stack of layers, each consuming the layer below and adding structure the raw substrate does not have on its own: first a photorealistic differentiable scene representation, then decomposition into parts and joints, then persistence of those parts through occlusion, then a graph of kinematic and contact relationships, then per-part physical material parameters, and finally an executable or latent dynamics model that can actually be rolled forward in time. Each layer below is a distinct, actively developed research area with its own toolchain.

### 4.1 Layer 0 — Substrate: 3D and 4D Gaussian Splatting

3D Gaussian Splatting (3DGS), introduced by Kerbl et al. and covered in depth in Chapter 115, represents a static scene as a set of anisotropic Gaussian primitives with position, covariance, opacity, and view-dependent color, rasterized in real time via tile-based EWA splatting. For a world model, 3DGS is attractive as a substrate for a specific reason: it is simultaneously photorealistic *and* differentiable, which means gradients from a rendering loss can, in principle, flow back into any physical quantity that influences a Gaussian's parameters.

PhysGaussian was among the first to exploit this directly, embedding a Material Point Method (MPM) simulator so that physical simulation and rendering share a single discrete representation — "what you see is what you simulate" — avoiding the mesh or cage embedding that earlier physics-on-NeRF approaches required, and demonstrating generative dynamics across elastic, metal, non-Newtonian fluid, and granular materials [Source](https://arxiv.org/abs/2311.12198). UniGS generalizes the rendering side of this substrate: a unified differentiable Gaussian Splatting framework that renders RGB, depth (via ray-ellipsoid intersection rather than reading off Gaussian centers, which is more geometrically accurate), surface normals, and semantic logits simultaneously with analytic gradients, plus a learnable pruning attribute for efficiency [Source](https://arxiv.org/abs/2510.12174). This RGB+depth+normal+semantic output is exactly the geometry- and semantics-aware substrate that the part-decomposition and material-inference layers above it (§4.2, §4.5) build on — they need more than color to disentangle "this Gaussian belongs to the door" from "this Gaussian belongs to the door frame."

### 4.2 Layer 1 — Part and Articulation Decomposition

A raw Gaussian splat is a soup of primitives with no notion of "object" or "part." The next layer partitions that soup into semantically and kinematically meaningful pieces — rigid parts connected by joints — directly within the Gaussian representation.

Part²GS learns per-part attributes for disentangled, physically consistent articulation using a motion-aware canonical representation with explicit contact, velocity, and vector-field constraints, plus a "repel points" field that prevents different parts from interpenetrating during optimization; it reports outperforming prior state-of-the-art by up to 10× in Chamfer Distance on movable parts [Source](https://arxiv.org/abs/2506.17212). PD²GS takes a different approach to the same problem, learning a shared canonical Gaussian field plus a continuous, latent-code-conditioned deformation — rather than a discrete set of interaction states — so that geometry and kinematics are jointly encoded and part identity does not fragment or drift across states; it also releases a new real-to-sim RGB-D benchmark, RS-Art [Source](https://arxiv.org/abs/2506.09663). SPLATART decouples the two sub-problems more explicitly, learning Gaussian splat representations of articulated objects from posed images (with or without part segmentation supplied) and separately estimating articulation and joint structure, which lets it recover deeper kinematic trees than prior single-joint-focused methods; it is evaluated on the Paris synthetic dataset plus real-world and serial-chain manipulator captures [Source](https://arxiv.org/abs/2506.12184).

Two adjacent, non-Gaussian-splat methods are worth knowing because they are strong points of comparison for joint-estimation accuracy specifically. A 2026 URDF-synthesis pipeline (internally called KinemaForge) infers part geometry, joint topology, and joint parameters jointly from RGB-D via a kinematic constraint graph, a differentiable screw-axis solver, and an energy-conservation residual loss that reduces long-horizon simulation drift; it reports 37.4% lower joint-axis error than PARIS, 46.6% lower than Ditto, 64% lower long-horizon simulation drift, and a 14.6 percentage-point improvement in closed-loop manipulation success rate [Source](https://arxiv.org/abs/2606.18861). Inst4DGS addresses a related but distinct failure mode — instance-label inconsistency across multiple video captures of a dynamic scene — using a differentiable Sinkhorn layer for cross-video label permutation plus instance-decomposed motion scaffolds for long-horizon per-Gaussian trajectories; on Panoptic Studio it reports PSNR improving from 26.10 to 28.36 and instance mIoU from 0.6310 to 0.9129 over the strongest baseline [Source](https://arxiv.org/abs/2603.18402). Inst4DGS is about *object/instance* decomposition and consistent tracking rather than joint articulation specifically, but the two problems compose: a world model needs both consistent instance identity and correct joint structure.

### 4.3 Layer 2 — Persistence Through Occlusion

Object permanence — knowing a cup is still on the table even while a hand blocks the camera's view of it — is trivial for humans and a genuine failure mode for naive dynamic-scene representations, which tend to let a Gaussian cluster simply vanish or freeze in place the moment it stops receiving photometric supervision.

PersistGS restores object permanence during full occlusion in dynamic 4D Gaussian Splatting by combining differentiable rigid-body simulation — per-object Gaussians paired with collision meshes, with friction and velocity estimated from the pre-occlusion trajectory — with a centroid silhouette loss designed to isolate positional gradient signal from appearance noise. It reports 40% lower trajectory error than photometric supervision alone, and a +2.46 dB PSNR improvement over naive constant-velocity extrapolation through the occluded interval [Source](https://arxiv.org/abs/2606.03479). This is the layer that turns a Gaussian-splat world model from "a very good renderer of what the camera currently sees" into something that can maintain a coherent belief about a scene through a temporary loss of observation — a prerequisite for using the representation in a real control loop, where occlusion by the robot's own gripper is routine rather than exceptional.

### 4.4 Layer 3 — Kinematic and Contact Scene Graphs

Individual part-and-joint decompositions (§4.2) need to be organized into a structure a planner can actually query — "what is connected to what, by what kind of joint, and what is currently in contact with what."

MoMa-SG builds articulated 3D scene graphs for open-world mobile manipulation from RGB-D by temporally segmenting interactions, tracking through them with occlusion-robust point tracking, lifting the result to 3D, and estimating revolute and prismatic joint parameters via a unified twist-estimation formulation solved in a single optimization pass — rather than the multi-stage segment-then-fit pipelines common in earlier work. It introduces the Arti4D-Semantic dataset (62 sequences, 600 labeled interactions) and validates the resulting scene graphs with real robot experiments [Source](https://arxiv.org/abs/2602.16356). The output — a graph of parts, joints, and contact relations, grounded in the same 3D coordinate frame as the underlying Gaussian splat — is the structure that both the material-inference layer (§4.5) and the dynamics layer (§4.6) consume: material inference needs to know which Gaussians constitute a rigid part before it can estimate that part's Young's modulus, and a dynamics model needs the joint graph before it can simulate anything more sophisticated than free-floating point clouds.

### 4.5 Layer 4 — Per-Part Material and Physical Parameter Inference

Knowing the geometry and the kinematic structure is not enough to *simulate* a scene — a differentiable physics engine also needs material parameters: stiffness, friction, density, damping, per part. Inferring these from ordinary video, rather than from a lab measurement, is its own line of research, and it has moved quickly from expensive per-scene test-time optimization toward fast feed-forward prediction.

PIXIE trains a supervised, feed-forward network that predicts a per-scene physical material field directly from 3D visual features — no per-scene optimization loop — paired with Gaussian Splatting for the physics simulation itself, trained on a new PIXIEVERSE dataset of 3D assets with physics annotations. It reports being 1.46–4.39× more accurate and orders of magnitude faster than test-time optimization baselines, and generalizes zero-shot to real scenes despite training only on synthetic data, via CLIP feature grounding [Source](https://arxiv.org/abs/2508.17437). UniPixie reframes the same problem probabilistically: rather than predicting a single point estimate for, say, Young's modulus, it learns a continuous, controllable distribution over material properties (soft-to-stiff) via flow matching on an extended PIXIEMULTIVERSE dataset, with a unified architecture that outputs parameters usable directly by MPM, Linear Blend Skinning, or Spring-Mass solvers. It reports reducing Young's-modulus prediction error by more than 50% against the strongest deterministic baseline [Source](https://arxiv.org/abs/2606.05399). ReconPhys tackles the harder single-video case: the first feedforward, self-supervised framework (no ground-truth physics labels required at all) that jointly learns physical attribute estimation and 3D Gaussian Splatting reconstruction from one monocular video, via a dual-branch architecture. It reports 21.64 dB PSNR on future-frame prediction versus 13.27 dB for SOTA optimization baselines, Chamfer Distance reduced from 0.349 to 0.004, and sub-second inference versus hours for prior optimization-based methods [Source](https://arxiv.org/abs/2604.07882).

The trend across all three is the same: material inference is moving from a per-scene inverse-optimization problem (expensive, but grounded in the specific observed video) to a feed-forward regression problem (cheap, generalizes, but only as good as its training distribution) — mirroring the exact trajectory NeRF-to-3DGS took for the rendering problem itself a few years earlier.

### 4.6 Layer 5 — Executable and Latent Dynamics World Models

The final layer takes the fully assembled representation — geometry, parts, joints, persistent identity, materials — and rolls it forward in time to predict what happens next, either by literally executing a physics simulator over it (an *executable* world model) or by learning a compact latent transition function trained to match simulated or observed outcomes (a *latent* world model).

DreMa combines Gaussian Splatting with physics simulators to build compositional, learnable digital twins that let a robot "imagine" novel object configurations and predict action outcomes, using equivariant transformations on a handful of demonstrations to synthesize additional training data; on a real Franka Emika Panda arm it enables one-shot policy learning — a novel physical task learned from a single demonstration per task variation, using DreMa's imagined rollouts to fill in the rest [Source](https://arxiv.org/abs/2412.14957). PhysTwin takes sparse multi-view video (as few as three camera views) of a deformable object under interaction and reconstructs a simulatable digital twin combining spring-mass physics, generative shape priors, and Gaussian-splat rendering, via a multi-stage inverse-modeling pipeline that jointly recovers geometry, physical parameters, and appearance; it supports real-time interactive simulation and model-based robot planning for objects like ropes, stuffed animals, cloth, and delivery packages [Source](https://arxiv.org/abs/2503.17973). PhysWorld builds on the same digital-twin idea but optimizes for downstream learning speed rather than simulation fidelity alone: it constructs an MPM-based digital twin from real video via constitutive-model selection and global-to-local property optimization, applies part-aware perturbations to synthesize a diverse set of demonstrations, and then trains a lightweight graph neural network world model embedded with the inferred physical properties — reporting inference 47× faster than PhysTwin, at the cost of trading simulator fidelity for a learned, amortized approximation [Source](https://arxiv.org/abs/2510.21447).

Not every recent result in this layer is a straightforward win for the executable, physics-grounded approach. MPMWorlds, a benchmark of 2D Material-Point-Method simulations spanning deformables, fluids, and kinetic objects, was built specifically to compare code-generation models against video-diffusion models on the task of inferring and extrapolating physical dynamics; its qualitative finding is that code-generation models produce physically and temporally stable extrapolations but infer material parameters poorly from vision, while video-diffusion models read scene geometry well but produce physically implausible extrapolations [Source](https://arxiv.org/abs/2606.01538) — a reminder that "generate an executable simulation" and "predict pixels" fail in different, complementary ways, and that neither family is uniformly better. Combined with the object-centric-world-model caution from §2 [Source](https://arxiv.org/abs/2511.06136), the state of this layer in mid-2026 is: executable, physics-grounded world models are winning on sample efficiency and physical plausibility for tasks with good material and geometry priors, but no single architecture in this layer is yet a clear default across task types.

## 5. The Gradient Problem

Every method in §4.6 that trains *through* a physics simulator — rather than merely rendering a static scene — runs into the same structural difficulty, which is worth treating as its own topic rather than a footnote to each paper: differentiating through contact-rich physical simulation is hard in ways that differentiating through rendering is not, and the two representations (rendering primitives, simulation primitives) frequently disagree about what a "gradient" through the scene should even mean.

### 5.1 Representational Misalignment

A Gaussian splat's parameters (position, covariance, opacity, SH color) are optimized for one objective: minimizing photometric reconstruction error. A physics simulator's state variables (particle deformation gradients in MPM, rigid-body poses and velocities, contact forces) are optimized for a different objective: predicting how the scene evolves under forces. These are not the same representation, and there is no a priori reason a gradient that reduces rendering loss also reduces physical-prediction error, or vice versa. PhysGaussian's "what you see is what you simulate" design (§4.1) is one resolution — collapse the two representations into one so misalignment cannot occur — but it constrains simulation to whatever MPM can express over Gaussian kernels directly, rather than allowing an independently-chosen physics representation (a mesh, a signed distance field, a spring network) to sit behind the renderer. ContactGaussian-WM makes the same design choice explicitly for contact-rich scenes: it uses a single, unified Gaussian representation for both visual appearance and collision geometry, and differentiates end-to-end through a closed-form physics engine directly on sparse, contact-rich video, rather than maintaining separate render and collision meshes that could drift out of alignment [Source](https://arxiv.org/abs/2602.11021).

### 5.2 Contact Non-Smoothness

Even with representations aligned, contact itself is the harder problem. Rigid-body contact is fundamentally non-smooth: a small change in an object's position can switch it discontinuously from "not touching" to "touching," and the resulting contact force is not a differentiable function of position at that boundary. A 2022 analysis comparing four contact formulations — linear complementarity problems (LCP), convex optimization, compliant (penalty-based) contact, and position-based dynamics — concludes that the gradients produced by differentiable contact simulators are "not always correct," validating this by comparing against analytical optimal-control solutions where ground-truth gradients are known [Source](https://arxiv.org/abs/2207.05060). This is a foundational caution for the entire layer: a simulator can be differentiable in the software-engineering sense (it runs `.backward()` without error) while still returning gradients that do not actually point in a direction that improves the objective, particularly near contact events.

A more recent analysis focused specifically on MuJoCo's MJX backend identifies concrete gradient-degradation mechanisms in penalty-based contact simulation under stiff or hard-contact settings, and proposes an approach — internally named DiffMJX — combining adaptive time integration with penalty simulation, plus a "contacts from distance" (CFD) technique: a straight-through gradient estimator applied only during the backward pass, which recovers usable pre-contact gradients that the forward-pass penalty formulation would otherwise destroy [Source](https://arxiv.org/abs/2506.14186). The practical upshot for anyone building a world model that trains through contact: expect the contact formulation to matter as much as the choice of rendering representation, and budget for the forward-pass and backward-pass physics to legitimately need different numerical treatments — a stiffer, more accurate forward integrator, paired with a deliberately smoothed backward pass.

### 5.3 Where the Gradient Enters

Given a coupled render+simulate pipeline, a separate design question is *at which variable* the rendering loss's gradient should be injected into the physics state. PhysMorph-GS makes this choice explicit and is, among the sources surveyed for this chapter, the clearest illustration of the tradeoff: it couples Material Point Method simulation with differentiable 3DGS and deliberately injects the render-supervision gradient through the **deformation gradient F** — the tensor MPM already uses to track how a material element has stretched and rotated from its rest configuration — rather than through raw Gaussian particle positions, specifically to keep the render gradient compatible with elastic dynamics rather than fighting them. It reports silhouette-error reductions of 25.8%, 10.8%, and 49.9% on representative test cases versus a physics-only baseline, with the largest gains on thin-feature geometry where positional gradients are least reliable [Source](https://arxiv.org/abs/2511.16988). PG-3DGS demonstrates the opposite direction of gradient flow — from a physics *objective* back into Gaussian shape parameters, rather than from a render loss into physics state — optimizing 3DGS geometry so the resulting shape satisfies physical objectives (a target pouring behavior, aerodynamic lift) jointly with photometric loss; it validates the result physically, 3D-printing the optimized geometries (a Cessna, a B-2 Spirit, a paper airplane) and measuring higher lift on a bench scale than an appearance-matching-only baseline in all three cases [Source](https://arxiv.org/abs/2605.11266). Together these two papers make the injection-point choice concrete: PhysMorph-GS pushes gradients from pixels *into* a physics state variable chosen for numerical compatibility with the simulator; PG-3DGS pushes gradients from a physics objective *into* geometry, treating the physics loss as just another differentiable term alongside the photometric one.

### 5.4 Parameter Identifiability

A quieter but persistent issue across every method in §4.5 is identifiability: from video alone, is the inferred material parameter actually the physical quantity it claims to be, or merely *a* parameter setting that happens to reproduce the observed motion under the assumed simulator? Two different (density, stiffness) pairs can produce visually indistinguishable trajectories under a given force; a feed-forward regressor trained on synthetic data (PIXIE, UniPixie) inherits whatever identifiability structure — or lack of it — exists in its training distribution, and a self-supervised single-video method (ReconPhys) has strictly less signal to disambiguate with than a multi-view, multi-frame method (PhysTwin). None of the sources surveyed for this chapter claim to have solved general parameter identifiability from monocular video; UniPixie's shift to a *distribution* over material properties rather than a point estimate (§4.5) is best read as a partial, probabilistic acknowledgment of this limit rather than a resolution of it — it reports the shape of its uncertainty, not a guarantee of correctness. *Note: needs verification* — whether any of these methods report calibration or coverage statistics for their predicted distributions (i.e., whether the true parameter actually falls within the predicted uncertainty band at the claimed rate) was not confirmed from the abstracts alone and would require reading each paper's evaluation section in full before relying on it.

## 6. Storage, Loading, and Interchange

A geometric world model is only as useful as the tooling around it that can save, load, compress, and hand a scene off between the rendering, simulation, and robotics halves of a pipeline that, in practice, are usually separate processes — often on separate machines.

### 6.1 KHR_gaussian_splatting and SPZ Compression

Until recently, 3D Gaussian Splatting had no standardized interchange format — the de facto convention was an ad hoc `.ply` file with a fixed, unversioned set of per-Gaussian properties (Chapter 115 §7.8 documents this convention as used by `gsplat`). That changed with `KHR_gaussian_splatting`, a Khronos Group glTF 2.0 extension backed by Autodesk, Cesium/Bentley Systems, Esri, Huawei, Niantic Spatial, NVIDIA, and XGRIDS, with the Alliance for OpenUSD (AOUSD), the Academy Software Foundation, and the UHD World Association as supporting bodies [Source](https://www.khronos.org/news/press/gltf-gaussian-splatting-press-release). As of this writing the extension's own specification header lists its status as **Release Candidate** [Source](https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md) — not yet ratified — so implementers should treat the attribute layout below as stable in shape but subject to final sign-off before depending on wire-format compatibility across tools.

The extension stores each Gaussian's position, rotation, scale, opacity, and spherical-harmonic color coefficients as standard glTF mesh-primitive attributes, treating the whole splat set as a point-primitive mesh with a graceful fallback to an ordinary point cloud for viewers that don't understand the extension:

```json
{
  "meshes": [{
    "primitives": [{
      "attributes": {
        "POSITION": 0,
        "KHR_gaussian_splatting:ROTATION": 1,
        "KHR_gaussian_splatting:SCALE": 2,
        "KHR_gaussian_splatting:OPACITY": 3,
        "KHR_gaussian_splatting:SH_DEGREE_0_COEF_0": 4
      },
      "mode": 0,
      "extensions": {
        "KHR_gaussian_splatting": {
          "kernel": "ellipse",
          "colorSpace": "srgb_rec709_display"
        }
      }
    }]
  }]
}
```

Higher-order SH coefficients (`SH_DEGREE_1_COEF_0` through `SH_DEGREE_3_COEF_6`) are optional attributes for view-dependent color beyond flat diffuse. The extension is deliberately designed to be extended further — new kernel types, color spaces, projection methods, and sorting methods are all expected to arrive as companion extensions rather than revisions to this base spec.

Compression is one such companion concern, handled separately by **SPZ**, a format originated by Niantic's Scaniverse team and open-sourced under the MIT license in September 2024 [Source](https://radiancefields.com/scaniverse-open-sources-spz-compression-for-3dgs). SPZ applies quantization plus arithmetic coding to reduce raw-PLY splat file sizes by roughly 90% (one reported example: 118 MB → 12 MB for 500K splats) with no perceptible quality loss, and remains Niantic-maintained rather than transferred to Khronos — the two projects are related (Niantic Spatial is a listed contributor to `KHR_gaussian_splatting` itself) but SPZ is referenced by, not absorbed into, the glTF extension family, via a proposed `KHR_gaussian_splatting_compression_spz` companion extension.

### 6.2 UsdPhysics, SimReady, and the AOUSD Robot-Description Mapping

Where `KHR_gaussian_splatting` standardizes the rendering-facing half of the stack, `UsdPhysics` standardizes the simulation-facing half. `UsdPhysics` is part of core OpenUSD (`pxr.UsdPhysics`), and its scope is governed by the Alliance for OpenUSD's Physics Working Group, chartered in December 2024 with the explicit goal of a normative, implementation-agnostic physics schema, focused initially on rigid bodies [Source](https://aousd.org/working-groups/). A minimal rigid-body USD stage using the standard schema APIs looks like:

```usda
#usda 1.0

def PhysicsScene "physicsScene"
{
    vector3f physics:gravityDirection = (0, 0, -1)
    float physics:gravityMagnitude = 9.81
}

def Xform "Table" (
    prepend apiSchemas = ["PhysicsRigidBodyAPI", "PhysicsCollisionAPI", "PhysicsMassAPI"]
)
{
    bool physics:rigidBodyEnabled = 1
    bool physics:kinematicEnabled = 0
    float physics:mass = 12.0

    def Mesh "CollisionGeom" (
        prepend apiSchemas = ["PhysicsCollisionAPI"]
    )
    {
        # mesh points/faceVertexIndices omitted
    }
}
```

This schema is deliberately engine-agnostic — it says nothing about *how* a specific simulator integrates the rigid body, only what mass and collision geometry it has. Bridging that to an actual robotics-native description (URDF, MJCF, SDFormat) is handled by converter tooling maintained under the Newton project rather than Pixar's core USD repository: `newton-physics/urdf-usd-converter` (URDF → USD) and `newton-physics/mujoco-usd-converter` (MJCF → USD, which layers Google DeepMind's `MjcPhysics` USD schema extension on top of core `UsdPhysics` to carry MuJoCo-specific features that have no `UsdPhysics` equivalent, such as MJCF's actuator and tendon definitions). A general-purpose Python URDF parser/validator, `yourdfpy`, is a related but distinct tool — a URDF-in, URDF-out library, not a USD converter — and should not be conflated with the conversion tooling above. NVIDIA's **SimReady** specification builds one more layer on top of `UsdPhysics`, standardizing not just physical properties but the broader set of metadata (semantic labels, physical material presets, articulation metadata) a simulation-ready asset needs to be dropped into a robotics or synthetic-data pipeline without manual re-authoring [Source](https://docs.omniverse.nvidia.com/simready/latest/overview/simready-spec.html).

### 6.3 MCAP and LeRobot: Trajectory and Episode Data

The formats above cover static or per-frame scene state. The remaining interchange problem is time-series: sensor streams and robot trajectories recorded while a world model's outputs are being validated against, or used to train, a real system. **MCAP**, a Foxglove-originated, self-describing, serialization-agnostic log container format, is the leading answer at the raw-sensor-stream level — `rosbag2_storage_mcap` has been the default rosbag2 storage plugin since ROS 2 Iron (May 2023), alongside continued SQLite3 support [Source](https://foxglove.dev/blog/mcap-as-the-ros2-default-bag-format). At the training-dataset level, HuggingFace's **LeRobot** library defines `LeRobotDataset`, which stores synchronized MP4 video/image streams alongside Parquet files carrying per-timestep state and action data, and spans the robot-learning stack from low-level motor middleware through large-scale dataset collection, storage, and streaming, plus an asynchronous inference stack for deploying trained policies [Source](https://arxiv.org/abs/2602.22818). *Note: needs verification* — a direct integration between LeRobot's dataset pipeline and MCAP specifically was not confirmed from the sources checked for this chapter; treat any claim of native LeRobot↔MCAP interop as unconfirmed until checked against current LeRobot documentation, since LeRobot's own dataset format is Parquet-based rather than MCAP-based.

## 7. Implementation Toolchain

The layers and formats above are implemented, in practice, by a fairly small and Linux-native set of open-source libraries. This section is a map of that toolchain rather than a tutorial; several of these tools already have dedicated coverage elsewhere in this book, cross-referenced below rather than re-explained.

### 7.1 Differentiable Simulation: Warp, Newton, Genesis, and MJX

NVIDIA **Warp** is the differentiable GPU-kernel substrate underneath most of the physics tooling in this chapter: Python-decorated kernels (`@wp.kernel`) that JIT-compile to CUDA, with automatic differentiation provided by `wp.Tape` — record a forward pass, call `tape.backward(loss)`, and read gradients back off `tape.gradients[array]`. Chapter 212 §7 already documents this API in the context of differentiable mesh queries; the same tape-based autodiff mechanism is what Newton and several of the papers in §5 build their end-to-end differentiable physics on top of:

```python
import warp as wp

wp.init()

@wp.kernel
def spring_energy(x: wp.array(dtype=wp.vec3),
                   rest_length: float, k: float,
                   energy: wp.array(dtype=float)):
    i = wp.tid()
    if i > 0:
        d = wp.length(x[i] - x[i - 1]) - rest_length
        wp.atomic_add(energy, 0, 0.5 * k * d * d)

x = wp.array(positions, dtype=wp.vec3, requires_grad=True, device="cuda")
energy = wp.zeros(1, dtype=float, requires_grad=True, device="cuda")

tape = wp.Tape()
with tape:
    wp.launch(spring_energy, dim=len(positions), inputs=[x, rest_length, k, energy])

tape.backward(loss=energy)
grad_x = tape.gradients[x]   # dEnergy/dx, per particle
```

**Newton** is a Linux Foundation project (Apache-2.0 license) built directly on Warp, integrating Google DeepMind's `mujoco_warp` as its primary simulation backend, and is backed by Disney Research, Google DeepMind, and NVIDIA among others [Source](https://github.com/newton-physics/newton). *Note: needs verification* — no single stable version number for Newton was confirmed at the time of writing; treat it as an actively developed pre-1.0-style community project and check the repository directly for current maturity before depending on API stability. NVIDIA's **Isaac Lab**, the successor to Isaac Gym, is a related GPU-accelerated simulation framework for multi-modal robot learning that combines parallel physics, photorealistic rendering, and modular environment/policy tooling, and is documented to have upcoming integration with Newton specifically [Source](https://arxiv.org/abs/2511.04831).

**Genesis** is a separate physics-simulation project with a notably different performance profile and, as of 2026, a different compiler backend than its original Taichi/CUDA design: it now compiles via Quadrants, a compiler forked from Taichi in mid-2025, which lowers Python simulation kernels to CUDA, AMD ROCm, Apple Metal, Vulkan, x86, and ARM64 — i.e., it is no longer CUDA-exclusive. Its README reports roughly 10–80× speedup over other GPU robotic simulators, and up to 43M FPS for a simulated Franka arm on a single RTX 4090 [Source](https://github.com/Genesis-Embodied-AI/genesis-world). *Note: needs verification* — this is a self-reported benchmark from the project's own README rather than an independently reproduced figure; treat the specific multiplier with appropriate skepticism until checked against a third-party benchmark. On the MuJoCo side, **MJX** (MuJoCo's JAX-based reimplementation) and `mujoco_warp` are the two GPU-parallel backends feeding both Newton (via `mujoco_warp`) and the DiffMJX contact-gradient work discussed in §5.2. **Dojo**, **Brax**, and **PyBullet** round out the broader differentiable/parallel physics landscape referenced across the papers in §4 and §5, each with different tradeoffs between contact accuracy, differentiability, and raw throughput that are outside this chapter's scope to compare directly.

### 7.2 Geometric Foundation Models: Pointmap Regression

The pointmap regression models surveyed empirically in §3 — DUSt3R, MASt3R, VGGT, π³, and MoGe-2 — are the practical entry point for turning ordinary video into the Gaussian-splat or point-cloud substrate described in §4.1, replacing the classical Structure-from-Motion pipeline (COLMAP, and its GPU-parallel successor **GLOMAP**) for many world-model pipelines where per-frame calibration overhead is a bottleneck. Chapter 115 §8 and §17–18 document COLMAP integration and the DUSt3R/MASt3R family in the specific context of NeRFStudio's `ns-process-data` pipeline; the same models are increasingly used as a pose-and-geometry front end feeding directly into the part-decomposition and material-inference layers of this chapter, bypassing an explicit SfM stage entirely.

### 7.3 3D Data Structures and Differentiable Rendering

Once a scene is reconstructed, the data-structure and differentiable-rendering layer is the one already documented in Chapter 212: Open3D (TSDF integration, point-cloud registration), PyTorch3D (`Meshes`/`Pointclouds`, differentiable mesh rasterization), Kaolin (SPC octrees, `GaussianSplatModel`, USD I/O), and trimesh (CPU mesh utilities, format conversion). The Gaussian-splat-specific rasterizer, `gsplat`, is documented in Chapter 115 §6–7 as the CUDA backend behind NeRFStudio's `splatfacto`; it is maintained by the nerfstudio-project organization and, notably for anyone building on non-NVIDIA hardware, has an AMD-maintained HIP/ROCm fork at `github.com/ROCm/gsplat` targeting Instinct MI300X-class GPUs, with a pull request to merge native ROCm support directly into the upstream `nerfstudio-project/gsplat` repository opened in May 2026 but not confirmed merged as of this writing [Source](https://github.com/ROCm/gsplat). Two additional differentiable renderers appear across the papers surveyed in this chapter but are not otherwise documented in this book: **Mitsuba 3**, a physically based, retargetable differentiable renderer used where accurate light-transport gradients (rather than rasterized Gaussian splatting) are needed, and **nvdiffrast**, NVIDIA's CUDA-accelerated, PyTorch-integrated differentiable rasterizer for triangle meshes [Source](https://github.com/NVlabs/nvdiffrast), which Chapter 115 §14 also references in the context of mesh-based differentiable rendering pipelines.

### 7.4 Segmentation and Semantic Priors

Part and object decomposition (§4.2) frequently needs a semantic or instance prior to bootstrap from, rather than discovering part boundaries purely from motion. Meta's **SAM 3** ("Segment Anything with Concepts") unifies detection, segmentation, and tracking behind a "Promptable Concept Segmentation" interface — prompting with short noun phrases or image exemplars rather than only points or boxes — trained on a data engine covering 4M unique concept labels, and reports roughly doubling the accuracy of prior systems on both image and video concept segmentation [Source](https://arxiv.org/abs/2511.16719). Paired with **DINOv2/v3** self-supervised visual features (used directly by several of the material-inference methods in §4.5 for zero-shot generalization), SAM 3 is a common front end for turning a raw Gaussian splat or point cloud into the segmented input the part-decomposition layer expects.

### 7.5 Collision Geometry and Serialization

Two smaller but load-bearing utilities close the loop between a reconstructed scene and a runnable simulation. **CoACD** (Approximate Convex Decomposition for 3D Meshes with Collision-Aware Concavity and Tree Search, SIGGRAPH 2022) generates the convex-decomposed collision meshes that rigid-body simulators require but that raw reconstructed geometry — Gaussian splats or arbitrary meshes — does not natively have [Source](https://github.com/SarahWeiii/CoACD); **V-HACD** is the older, still widely used alternative for the same job. For serializing trained model weights and inferred parameter fields (material fields, network checkpoints) safely across the Python/Rust/C++ boundary common in this toolchain, **safetensors** has become the default over pickle-based formats, for the same reason it has in the broader ML ecosystem: no arbitrary code execution on load. The **Foxglove SDK** provides client libraries for writing and visualizing MCAP logs (§6.3) directly from a robotics or simulation process, closing the loop from "physics simulation ran" to "the trajectory is inspectable in a standard log viewer."

## 8. Open Problems

Pulling the threads of §5 through §7 together, three problems remain genuinely open as of mid-2026, rather than merely "not yet productized":

- **No unified benchmark spans the whole stack.** E3D-Bench (§3) evaluates geometric foundation models; MPMWorlds (§4.6) evaluates dynamics inference and extrapolation; OBSBench (§3) evaluates observation-space choice for policy learning. No benchmark surveyed for this chapter evaluates the full pipeline — reconstruction, decomposition, material inference, and downstream control — end to end, which makes it hard to know whether an error introduced at, say, the part-decomposition layer (§4.2) is amplified or absorbed by the time it reaches a policy trained on top of the resulting world model.
- **Gradient correctness through contact remains partially unresolved.** §5.2's foundational skepticism about whether differentiable contact simulators produce correct gradients at all [Source](https://arxiv.org/abs/2207.05060) predates most of the specific methods surveyed in §4, and none of those later methods directly re-litigate that finding for their specific contact formulation. DiffMJX's straight-through estimator (§5.2) is a mitigation for one class of gradient-degradation mechanism, not a general proof of correctness for arbitrary contact-rich scenes.
- **Identifiability from monocular or sparse-view video is unresolved in general**, as discussed in §5.4. The field's current response is architectural (feed-forward regression to amortize the cost of a poorly-conditioned inverse problem, or probabilistic output to represent the resulting uncertainty) rather than a claim that the underlying ambiguity has been removed.

A world-model builder choosing among the layers in §4 should treat each layer's reported numbers as measured under that paper's specific evaluation protocol, not as a composable error budget for the full stack — the tooling to actually measure the composed system does not yet exist in a standardized form.

## 9. Integrations

**Chapter 115 — NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting:** The substrate layer of this chapter (§4.1) is the same 3D Gaussian Splatting representation covered there in depth — the `gsplat` CUDA rasterizer (§6–7), COLMAP/DUSt3R/MASt3R integration (§8, §17–18), and the PLY/interchange conventions (§7.8) that `KHR_gaussian_splatting` (§6.1 of this chapter) is now standardizing around.

**Chapter 212 — Python 3D ML Libraries (Open3D, PyTorch3D, Kaolin, and Warp):** The data-structure layer beneath every method in §4 and §7 — PyTorch3D `Meshes`/`Pointclouds`, Kaolin's `GaussianSplatModel` and USD I/O, Open3D TSDF and registration, and NVIDIA Warp's `wp.Tape` differentiable-kernel autodiff (§7.1 of this chapter reuses that exact API) — is documented there.

**Chapter 69 — Omniverse and USD:** `UsdPhysics` (§6.2) and NVIDIA's SimReady specification build directly on the OpenUSD foundations covered there; Kaolin's USD I/O (Chapter 212) and Isaac Lab's simulation environments (§7.1) both round-trip through the same USD stage format.

**Chapter 64 — glTF:** `KHR_gaussian_splatting` (§6.1) is a Khronos extension to the core glTF 2.0 format documented there; readers implementing a loader for the extension need the base glTF mesh-primitive and accessor model from that chapter first.

**Chapter 209 — OpenSLAM and Chapter 210 — SLAM Theory and SOTA:** Pointmap regression models (§7.2 — DUSt3R, MASt3R, VGGT) and Gaussian-splat scene representations increasingly substitute for or complement classical visual SLAM front ends; the pose-graph and loop-closure theory in those chapters underlies why feed-forward pointmap regression, which sidesteps explicit bundle adjustment, is such a significant simplification for the reconstruction stage of this chapter's stack (§3).

**Chapter 211 — ROS 2 Sensor and Perception Pipelines:** MCAP (§6.3) is the same log format documented there as the ROS 2 rosbag2 default storage backend; the kinematic/contact scene graphs of §4.4 and the LeRobot trajectory data of §6.3 are natural companions to the sensor pipelines described in that chapter.

**Chapter 25 — GPU Compute and Chapter 66 — CUDA Runtime:** Every differentiable-simulation and pointmap-regression tool in §7 (Warp, `mujoco_warp`, gsplat, nvdiffrast) is a CUDA-first workload; the memory-management and kernel-launch fundamentals in those chapters apply directly to tuning the toolchain in §7.1–7.3.

**Chapter 48 — ROCm and Machine Learning on Linux GPUs:** Genesis's Quadrants-based multi-backend compilation and gsplat's AMD-maintained HIP/ROCm fork (§7.3) are the two ROCm on-ramps into this chapter's toolchain for readers on AMD hardware; Chapter 48's coverage of ROCm's maturity and gaps applies directly to both.

**Chapter 117 — Slang and the Differentiable Shading Language:** Chapter 115 §7.5 documents a Slang reimplementation of the gsplat tile rasterizer using `[Differentiable]` autodiff for SPIR-V/Vulkan targets; the same differentiable-shading approach is a plausible future path for the render-side half of the gradient-injection question discussed in §5.3 of this chapter, as an alternative to hand-written CUDA backward kernels.

---

*References used in this chapter:*

- [How Should World Models Be Evaluated for Embodied Decision-Making? A Decision-Making-Centric Position](https://arxiv.org/abs/2606.15032) — 2026
- [Point Cloud Matters: Rethinking the Impact of Different Observation Spaces on Robot Learning (OBSBench)](https://arxiv.org/abs/2402.02500) — 2024
- [V-JEPA 2: Self-Supervised Video Models Enable Understanding, Prediction and Planning](https://arxiv.org/abs/2506.09985) — Meta, 2025
- [When Object-Centric World Models Meet Policy Learning: From Pixels to Policies, and Where It Breaks](https://arxiv.org/abs/2511.06136) — 2025
- [PhysGaussian: Physics-Integrated 3D Gaussians for Generative Dynamics](https://arxiv.org/abs/2311.12198) — Xie et al., 2023
- [UniGS: Unified Geometry-Aware Gaussian Splatting for Multimodal Rendering](https://arxiv.org/abs/2510.12174) — 2025
- [E3D-Bench: A Benchmark for End-to-End 3D Geometric Foundation Models](https://arxiv.org/abs/2506.01933) — Cong et al., 2025
- [Part²GS: Part-aware Modeling of Articulated Objects using 3D Gaussian Splatting](https://arxiv.org/abs/2506.17212) — 2025
- [PD²GS: Part-Level Decoupling and Continuous Deformation of Articulated Objects via Gaussian Splatting](https://arxiv.org/abs/2506.09663) — 2025
- [SPLATART: Articulated Gaussian Splatting with Estimated Object Structure](https://arxiv.org/abs/2506.12184) — Lewis et al., University of Michigan, 2025
- [URDF Synthesis from RGB-D Sequences via Differentiable Joint Inference and Energy-Consistent Verification](https://arxiv.org/abs/2606.18861) — Zhang et al., 2026
- [Inst4DGS: Instance-Decomposed 4D Gaussian Splatting with Multi-Video Label Permutation Learning](https://arxiv.org/abs/2603.18402) — Lee et al., 2026
- [PersistGS: Differentiable Physics for Object Permanence in 4D Gaussian Splatting](https://arxiv.org/abs/2606.03479) — 2026
- [Articulated 3D Scene Graphs for Open-World Mobile Manipulation](https://arxiv.org/abs/2602.16356) — Büchner et al., 2026
- [Pixie: Fast and Generalizable Supervised Learning of 3D Physics from Pixels](https://arxiv.org/abs/2508.17437) — 2025
- [UniPixie: Unified and Probabilistic 3D Physics Learning via Flow Matching](https://arxiv.org/abs/2606.05399) — 2026
- [ReconPhys: Reconstruct Appearance and Physical Attributes from Single Video](https://arxiv.org/abs/2604.07882) — 2026
- [Dream to Manipulate: Compositional World Models Empowering Robot Imitation Learning with Imagination (DreMa)](https://arxiv.org/abs/2412.14957) — 2024
- [PhysTwin: Physics-Informed Reconstruction and Simulation of Deformable Objects from Videos](https://arxiv.org/abs/2503.17973) — Jiang et al., ICCV 2025
- [PhysWorld: From Real Videos to World Models of Deformable Objects via Physics-Aware Demonstration Synthesis](https://arxiv.org/abs/2510.21447) — 2025
- [MPMWorlds: Material-Point-Method Simulations for Inferring and Extrapolating Physical Dynamics](https://arxiv.org/abs/2606.01538) — 2026
- [ContactGaussian-WM: Learning Physics-Grounded World Model from Videos](https://arxiv.org/abs/2602.11021) — 2026
- [PhysMorph-GS: Render-Guided Volumetric Morphing with Differentiable Physics](https://arxiv.org/abs/2511.16988) — 2025
- [PG-3DGS: Optimizing 3D Gaussian Splatting to Satisfy Physics Objectives](https://arxiv.org/abs/2605.11266) — 2026
- [Differentiable Simulation of Hard Contacts with Soft Gradients for Learning and Control (DiffMJX)](https://arxiv.org/abs/2506.14186) — 2025
- [Differentiable Physics Simulations with Contacts: Do They Have Correct Gradients w.r.t. Position, Velocity and Control?](https://arxiv.org/abs/2207.05060) — 2022
- [DUSt3R: Geometric 3D Vision Made Easy](https://arxiv.org/abs/2312.14132) — Wang et al., CVPR 2024
- [MASt3R: Grounding Image Matching in 3D with MASt3R](https://github.com/naver/mast3r) — NAVER LABS Europe
- [VGGT: Visual Geometry Grounded Transformer](https://arxiv.org/abs/2503.11651) — 2025
- [π³: Permutation-Equivariant Visual Geometry Learning](https://arxiv.org/abs/2507.13347) — 2025
- [MoGe-2: Accurate Monocular Geometry with Metric Scale and Sharp Details](https://arxiv.org/abs/2507.02546) — Microsoft, 2025
- [SAM 3: Segment Anything with Concepts](https://arxiv.org/abs/2511.16719) — Meta, 2025
- [Isaac Lab: A GPU-Accelerated Simulation Framework for Multi-Modal Robot Learning](https://arxiv.org/abs/2511.04831) — NVIDIA, 2025
- [LeRobot: An Open-Source Library for End-to-End Robot Learning](https://arxiv.org/abs/2602.22818) — HuggingFace, 2026
- [KHR_gaussian_splatting Extension](https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md) — Khronos Group
- [Khronos, OGC, and Geospatial Industry Leaders Add 3D Gaussian Splats to the glTF Asset Standard](https://www.khronos.org/news/press/gltf-gaussian-splatting-press-release) — Khronos Group press release
- [Scaniverse Open-Sources SPZ Compression for 3DGS](https://radiancefields.com/scaniverse-open-sources-spz-compression-for-3dgs) — Niantic
- [SimReady Asset Specification](https://docs.omniverse.nvidia.com/simready/latest/overview/simready-spec.html) — NVIDIA
- [AOUSD Physics Working Group](https://aousd.org/working-groups/) — Alliance for OpenUSD
- [Newton Physics Engine](https://github.com/newton-physics/newton) — Linux Foundation
- [Genesis: A Universal and Generative Physics Engine](https://github.com/Genesis-Embodied-AI/genesis-world) — Genesis-Embodied-AI
- [MCAP as the ROS 2 Default Bag Format](https://foxglove.dev/blog/mcap-as-the-ros2-default-bag-format) — Foxglove
- [gsplat: An Open-Source Library for Gaussian Splatting](https://github.com/nerfstudio-project/gsplat) — nerfstudio-project
- [ROCm/gsplat: HIP Port for AMD GPUs](https://github.com/ROCm/gsplat) — AMD
- [nvdiffrast: Modular Primitives for High-Performance Differentiable Rendering](https://github.com/NVlabs/nvdiffrast) — Laine, Hellsten, Karras et al., NVIDIA, SIGGRAPH Asia 2020
- [CoACD: Approximate Convex Decomposition for 3D Meshes with Collision-Aware Concavity and Tree Search](https://github.com/SarahWeiii/CoACD) — SIGGRAPH 2022

## Roadmap

### Near-term (6–12 months)
- **`KHR_gaussian_splatting` ratification.** The extension is a Release Candidate as of this writing; ratification would let engine and viewer vendors commit to the wire format without hedging against breaking changes, and would likely accelerate adoption of `KHR_gaussian_splatting_compression_spz` as the default compressed-transport companion.
- **Consolidation of feed-forward material-inference models.** PIXIE, UniPixie, and ReconPhys (§4.5) all published within months of each other across late 2025 and early-to-mid 2026, each with a different dataset and solver-output convention; a shared benchmark and output schema (analogous to what E3D-Bench did for geometric foundation models) would make these directly comparable.
- **Native ROCm support landing in upstream `gsplat`.** The pending pull request noted in §7.3 would move AMD GPU support for the Gaussian-splat substrate layer from a maintained fork to a first-class upstream target.

### Medium-term (1–3 years)
- **A composed, end-to-end benchmark spanning §4's full stack**, from raw video through reconstruction, part decomposition, material inference, and downstream policy performance — the gap identified in §8 — is a plausible next step once the individual-layer benchmarks (E3D-Bench, MPMWorlds, OBSBench) mature further.
- **Differentiable-shading-language backward passes** (Chapter 115 §7.5's Slang reimplementation) replacing hand-written CUDA backward kernels across more of the toolchain in §7, reducing the maintenance burden of keeping forward and backward passes manually synchronized as gradient-injection strategies (§5.3) continue to diversify.
- **UsdPhysics maturing beyond rigid bodies.** The AOUSD Physics Working Group's initial charter scope is rigid-body dynamics (§6.2); soft-body, cloth, and fluid schema coverage — needed to represent the MPM-based digital twins of §4.6 natively in USD rather than through engine-specific extensions — is a plausible medium-term expansion.

### Long-term
- **Resolution, or at least a widely adopted mitigation, for the contact-gradient correctness problem (§5.2, §8)** — either through provably-correct smoothed contact formulations, or through a shift toward learned dynamics models (§4.6) that sidestep differentiating through a hand-written contact solver entirely.
- **Convergence between the "explicit geometry" and "latent" world-model families framed as opposed in §1–§2** — plausibly not a winner-take-all outcome, but a hybrid where geometric layers (§4.1–§4.4) ground a latent dynamics model's early layers, combining V-JEPA-2-style broad video pretraining with the sample efficiency and physical plausibility that explicit geometry provides for the specific tasks the empirical evidence in §3 favors it for.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
