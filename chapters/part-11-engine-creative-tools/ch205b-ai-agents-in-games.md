# Chapter 205b: AI Agents in Games — RL Environments and LLM-Driven NPCs

> **Part**: Part XI — Engines and Creative Tools
> **Audience**: Graphics and engine developers who need to attach a learning agent or an LLM-driven NPC to an existing render loop. The central question is not "which algorithm" but "where does the agent's control flow attach, and what does that attachment cost in frames, readbacks, and IPC round-trips?"
> **Status**: First draft — 2026-08-08

A game engine and a reinforcement-learning trainer want opposite things from time. The engine wants a wall-clock-locked tick that presents a frame every 16.6 ms, forever, with soft-real-time latency guarantees on input. The trainer wants to advance simulation state as fast as silicon allows, in lockstep with a gradient update, with no presentation at all and no notion of wall-clock time. An LLM-driven NPC wants something different again: a control loop whose individual decisions take seconds to tens of seconds and which must never be allowed to stall the frame.

These three appetites produce three genuinely distinct software architectures, and almost every framework in this space is an instance of one of them. **Synchronous lockstep** puts the engine and the trainer in separate processes exchanging observations and actions over a local socket, each blocking on the other; Godot RL Agents and `bevy_rl` are the engine-native examples. **Vectorized headless simulation** rewrites the environment as pure array code, compiled by XLA and `vmap`-ed so that thousands of copies advance in a single kernel launch with the policy update fused into the same program — no socket, no process boundary, and in the extreme case no graphics API involvement whatsoever; Craftax is the reference example. The **decoupled asynchronous agent loop** runs a simulation tick at interactive rates while agent cognition — retrieval, reflection, planning, dialogue — runs on a far slower clock, with results landing whenever they arrive; AI Town is the reference example.

This chapter reads each architecture out of primary source: the wire protocol in `godot_env.py`, the REST integration test in `bevy_rl`, the `step_env` method in Craftax's pixel environment, and the timing constants in AI Town's `convex/constants.ts`. It treats the **GPU-to-CPU readback path for pixel observations** as a first-class subject, because that readback is where the graphics stack and the learning stack actually collide, and it is the dominant cost in vision-based training on a conventional engine. One further caution: this corner of the ecosystem has an unusually high ratio of celebrated papers to maintained code, and several of the most-cited projects have not received a commit in close to three years. Section 11 lists them with last-commit dates rather than pretending they are live options.

---

## Table of Contents

- [1. Scope: What Counts as "AI in the Game Loop"](#1-scope-what-counts-as-ai-in-the-game-loop)
- [2. Classical Game AI: Navmeshes, Behaviour Trees, GOAP, and Utility Systems](#2-classical-game-ai-navmeshes-behaviour-trees-goap-and-utility-systems)
  - [2.1 Navmeshes and Pathfinding](#21-navmeshes-and-pathfinding)
  - [2.2 Behaviour Trees](#22-behaviour-trees)
  - [2.3 GOAP: Goal-Oriented Action Planning](#23-goap-goal-oriented-action-planning)
  - [2.4 Utility AI Systems](#24-utility-ai-systems)
- [3. How Modern AI Integrates and Subsumes Classical Game AI](#3-how-modern-ai-integrates-and-subsumes-classical-game-ai)
  - [3.1 The LLM Generates the Classical Structure](#31-the-llm-generates-the-classical-structure)
  - [3.2 The Classical Structure Disciplines the LLM](#32-the-classical-structure-disciplines-the-llm)
  - [3.3 State Trees: A Data-Driven Successor Already Absorbing Part of BT's Role](#33-state-trees-a-data-driven-successor-already-absorbing-part-of-bts-role)
  - [3.4 Commercial Productization](#34-commercial-productization)
  - [3.5 NPC SDKs: A Wider Commercial Landscape](#35-npc-sdks-a-wider-commercial-landscape)
- [4. The Environment API Standard: Gymnasium and PettingZoo](#4-the-environment-api-standard-gymnasium-and-pettingzoo)
  - [4.1 The Single-Agent Contract](#41-the-single-agent-contract)
  - [4.2 `terminated` vs `truncated`](#42-terminated-vs-truncated)
  - [4.3 Rendering Is Not in the Core Dependency Set](#43-rendering-is-not-in-the-core-dependency-set)
  - [4.4 PettingZoo: AEC and Parallel](#44-pettingzoo-aec-and-parallel)
- [5. Synchronous Lockstep I: Godot RL Agents](#5-synchronous-lockstep-i-godot-rl-agents)
  - [5.1 The Inverted Client/Server Polarity](#51-the-inverted-clientserver-polarity)
  - [5.2 Framing and Message Types](#52-framing-and-message-types)
  - [5.3 The Engine-Side Contract: AIController](#53-the-engine-side-contract-aicontroller)
  - [5.4 Pixel Observations and the Hex-Encoding Readback](#54-pixel-observations-and-the-hex-encoding-readback)
  - [5.5 Amortizing the Round-Trip: Action Repeat and Speed Up](#55-amortizing-the-round-trip-action-repeat-and-speed-up)
  - [5.6 Escaping Python: ONNX In-Engine Inference](#56-escaping-python-onnx-in-engine-inference)
- [6. Synchronous Lockstep II: bevy_rl](#6-synchronous-lockstep-ii-bevy_rl)
  - [6.1 Opposite Socket Ownership](#61-opposite-socket-ownership)
  - [6.2 The Plugin Surface and the Pause Handshake](#62-the-plugin-surface-and-the-pause-handshake)
  - [6.3 Render-to-Buffer and the wgpu Readback](#63-render-to-buffer-and-the-wgpu-readback)
  - [6.4 An Honest Maturity Assessment](#64-an-honest-maturity-assessment)
- [7. In-Engine Inference, External Training: Unreal Engine Learning Agents](#7-in-engine-inference-external-training-unreal-engine-learning-agents)
  - [7.1 Status, Modules, and Dependencies](#71-status-modules-and-dependencies)
  - [7.2 The Interactor Contract: Observations and Actions](#72-the-interactor-contract-observations-and-actions)
  - [7.3 The Trainer: Rewards, Completions, and the External Process](#73-the-trainer-rewards-completions-and-the-external-process)
  - [7.4 Composing with Classical AI: Neural Networks Inside a Behaviour Tree](#74-composing-with-classical-ai-neural-networks-inside-a-behaviour-tree)
  - [7.5 An Honest Maturity Assessment](#75-an-honest-maturity-assessment)
- [8. Vectorized Headless GPU Simulation: Craftax](#8-vectorized-headless-gpu-simulation-craftax)
  - [8.1 The gymnax Functional Interface](#81-the-gymnax-functional-interface)
  - [8.2 Rasterization as Array Code](#82-rasterization-as-array-code)
  - [8.3 Optimistic Resets and Why Branching Is Expensive](#83-optimistic-resets-and-why-branching-is-expensive)
  - [8.4 Compilation Latency as the New Startup Cost](#84-compilation-latency-as-the-new-startup-cost)
- [9. Beyond Craftax: The Wider GPU-Native Environment Ecosystem](#9-beyond-craftax-the-wider-gpu-native-environment-ecosystem)
  - [9.1 The `vmap` Family: Brax, Pgx, and Jumanji](#91-the-vmap-family-brax-pgx-and-jumanji)
  - [9.2 Hand-Written CUDA Batching: WarpDrive](#92-hand-written-cuda-batching-warpdrive)
  - [9.3 A GPU-Native ECS Engine: Madrona](#93-a-gpu-native-ecs-engine-madrona)
  - [9.4 The Deliberate Counter-Example: EnvPool](#94-the-deliberate-counter-example-envpool)
  - [9.5 A Different Axis: PufferLib as Wrapper-Plus-Trainer](#95-a-different-axis-pufferlib-as-wrapper-plus-trainer)
  - [9.6 Choosing Among These Frameworks](#96-choosing-among-these-frameworks)
- [10. Decoupled Asynchronous Agent Loops: AI Town](#10-decoupled-asynchronous-agent-loops-ai-town)
  - [10.1 The Arithmetic of Decoupling](#101-the-arithmetic-of-decoupling)
  - [10.2 Engine, Inputs, and the Transactional Backend](#102-engine-inputs-and-the-transactional-backend)
  - [10.3 Memory, Retrieval, and the Embedding Contract](#103-memory-retrieval-and-the-embedding-contract)
  - [10.4 The Rendering Half: PixiJS](#104-the-rendering-half-pixijs)
  - [10.5 Local Inference and Cost Control](#105-local-inference-and-cost-control)
  - [10.6 Running and Customizing an Instance](#106-running-and-customizing-an-instance)
- [11. The Historical Tier: Lineage, Not Recommendations](#11-the-historical-tier-lineage-not-recommendations)
  - [11.1 Maintained but CPU-Only: OpenSpiel and Melting Pot](#111-maintained-but-cpu-only-openspiel-and-melting-pot)
- [12. Three Integration Shapes, Compared](#12-three-integration-shapes-compared)
  - [12.1 Choosing a Shape](#121-choosing-a-shape)
- [Integrations](#integrations)
- [References](#references)

---

## 1. Scope: What Counts as "AI in the Game Loop"

This chapter covers frameworks that embed a learning or language-model agent into a *game* or *game engine* — something with a render loop, an entity model, and gameplay semantics. Three adjacent territories are deliberately excluded.

**General robotics simulators** are a different problem. MuJoCo, its JAX port MJX, and Gazebo optimize for contact-rich rigid-body dynamics with physical fidelity as the objective; their rendering produces camera observations and debug views, not a presented game. They are covered in Chapter 211a. The architectural *lesson* transfers — MJX and Craftax are the same idea applied to different state spaces — but the software is unrelated.

**NVIDIA's robotics-training stack** — Isaac Sim, Isaac Lab, and the GR00T foundation-model family — is the industrial-scale version of the same vectorized-simulation idea, with a proprietary renderer and a USD-based scene model. It is covered in Chapters 69 and 240.

**Classical game AI** — navmeshes, behaviour trees, GOAP, utility systems — used to sit outside this chapter's scope as deterministic authored logic with no new coupling to the render loop. It no longer does: enough of the "AI in games" conversation now runs directly through these systems, either as the thing being described (Section 2) or as the execution layer a learned or generative policy sits on top of (Section 3), that leaving them out would misrepresent the field as it stands in 2026.

What remains is the intersection this chapter is about: classical decision-making and pathfinding architectures on their own terms (Section 2), how modern learned and generative AI integrates with and increasingly subsumes parts of that classical stack (Section 3), the API standard that defines the environment contract (Section 4), three engine-integration bridges (Sections 5, 6, and 7), one fully-vectorized GPU-native game environment (Section 8), the wider ecosystem of GPU-native and vectorized environment frameworks it belongs to (Section 9), one LLM-driven social simulation (Section 10), and the historical projects that established the research agenda (Section 11).

---

## 2. Classical Game AI: Navmeshes, Behaviour Trees, GOAP, and Utility Systems

These four systems are the actual substrate of "game AI" as shipped in the overwhelming majority of released games — deterministic, authored, and designed to be debuggable by a designer rather than trained. They matter to this chapter for two reasons: they are the baseline every learned or LLM-driven approach in the rest of the chapter is implicitly compared against, and — as Section 3 covers — they are increasingly the execution layer that generative approaches sit on top of rather than replace.

### 2.1 Navmeshes and Pathfinding

A navigation mesh (navmesh) is a walkable-surface approximation of a level's geometry — a set of convex polygons an agent can legally stand on — searched with A* to produce a polygon corridor, then smoothed into a concrete path with a funnel algorithm. The de facto standard implementation is **Recast & Detour**: Recast voxelizes arbitrary level geometry and extracts the navmesh, Detour performs the runtime pathfinding and steering queries against it. As of this writing the repository carries **7,854 stars**, is Zlib-licensed, and was last pushed **2026-02-27** [[Source](https://github.com/recastnavigation/recastnavigation)]. Recast/Detour or a close derivative underpins navigation in Unity, Unreal Engine, O3DE, and CryEngine; Godot ships its own `NavigationServer` implementation rather than embedding Recast directly, but targets the same polygon-mesh-plus-funnel-algorithm model.

The reason navmeshes are stable ground rather than a contested research topic is that the problem is well-posed: level geometry is static (or piecewise-static, with runtime rebaking for doors and destructible cover), and A* over a polygon graph is cheap enough to run per-agent per-tick even for large crowds. Nothing in the rest of this section, or in Section 3, changes that — every behaviour tree, GOAP planner, or LLM-driven agent covered below still calls into a navmesh query to get from A to B; none of them reimplement pathfinding.

### 2.2 Behaviour Trees

A behaviour tree (BT) composes small, reusable action and condition nodes into a tree evaluated top-to-bottom every tick, using **selector** nodes (try children in order until one succeeds) and **sequence** nodes (run children in order until one fails) to express fallback and multi-step logic without a combinatorial explosion of hand-coded states. BTs displaced hand-rolled finite state machines as the default authored-AI architecture in the 2000s specifically because a designer can extend a tree by adding a branch, where extending an FSM means touching a transition table that already has quadratic-in-states edges.

**BehaviorTree.CPP** is the reference open-source implementation: a C++17 library with an XML-based tree definition format and Groot2, a companion visual editor, that has moved well beyond games into robotics — it is the planning backbone of ROS 2's Nav2 navigation stack. It carries **4,153 stars**, is MIT-licensed, and was last pushed **2026-08-05** [[Source](https://github.com/BehaviorTree/BehaviorTree.CPP)]. On the commercial side, Unreal Engine ships a built-in visual Behavior Tree editor (Blackboard-backed, node-graph authored) as one of its two first-party AI decision architectures — the other being State Tree, covered in §3.3 as its data-driven successor [[Source](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-state-tree-in-unreal-engine)]. Godot ships no first-party behaviour-tree editor at all, which is what makes **LimboAI** the de facto standard for the engine: a C++ GDExtension/module plugin providing both a behaviour-tree editor and a state-machine system, with GDScript-authored tasks and states and a built-in visual debugger. It carries **2,941 stars**, is MIT-licensed, and was last pushed **2026-08-04** [[Source](https://github.com/limbonaut/limboai)] — a maintenance profile closer to BehaviorTree.CPP's than to the smaller GOAP and utility-AI libraries below.

### 2.3 GOAP: Goal-Oriented Action Planning

Goal-Oriented Action Planning adapts STRIPS — the 1971 Stanford Research Institute Problem Solver — into a form cheap enough to run in a game loop. Each action is declared with preconditions and effects expressed as world-state facts; a planner (in practice, A* searching backward from the goal state to the current state, with per-action costs as the search heuristic) assembles a sequence of actions that satisfies a goal, rather than a designer hand-authoring every path to it. GOAP's canonical origin is Jeff Orkin's AI for Monolith's **F.E.A.R.** (2005), documented in his 2006 GDC talk "Three States and a Plan": the AI loop collapses to a three-state FSM (Go To, Animate, Use Smart Object) wrapped around a planner that re-plans in real time, so a soldier who loses a flanking route through a door the player slammed shut can replan through a window or a different entrance without any of that contingency being scripted in advance [[Source](https://www.gamedeveloper.com/design/building-the-ai-of-f-e-a-r-with-goal-oriented-action-planning)].

GOAP's appeal over a behaviour tree is exactly the flexibility a BT's fixed structure doesn't give you — new emergent action sequences the designer never explicitly authored — traded against a real cost: planning is a search, not a tree walk, so it is more expensive per tick and considerably harder to predict and debug than a BT's linear evaluation order. Open-source implementations exist mainly as small, single-purpose C/C++ libraries rather than large maintained frameworks: **cppGOAP** (105 stars, no clearly declared open-source license per GitHub's detector — *note: verify licensing before depending on it*) [[Source](https://github.com/cpowell/cppGOAP)] and **GPGOAP** (619 stars, no license file) [[Source](https://github.com/stolk/GPGOAP)] are both widely referenced minimal implementations, which is itself informative: unlike navmeshes or behaviour trees, GOAP has never produced a dominant, actively-maintained open-source library on the scale of Recast or BehaviorTree.CPP, and most shipped uses remain in-house engine code.

### 2.4 Utility AI Systems

A utility system replaces both the BT's fixed branching and GOAP's search with continuous scoring: every candidate action is run through one or more curve-based "considerations" (distance to target, health remaining, ammo count, threat level) that each produce a normalized score, those scores are combined, and the agent executes whichever action scores highest this tick. The most-cited formalization is Dave Mark's **Infinite Axis Utility System (IAUS)**, presented at GDC 2013 and grounded in his 2009 book *Behavioral Mathematics for Game AI*: an architecture built specifically to let modular "reasoners" combine an effectively unbounded number of independent considerations, avoiding the brittle nested-conditional logic utility systems were designed to replace [[Source](https://www.gameai.com/iaus.php)]. Utility scoring is the standard technique for large-cast simulation games — it is the documented architecture behind The Sims' agent decision-making, where an object's own "advertisement" of what needs it satisfies is scored against a Sim's current needs.

On the open-source side, the Rust/Bevy ecosystem has the clearest concrete example: **big-brain**, a data-driven utility AI library where a game only implements Scorers and Actions and the library handles selection, carries 1,300 stars and is Rust-licensed dual MIT/Apache, but its repository is **archived** as of its last push on 2025-10-07 [[Source](https://github.com/zkat/big-brain)]. The behaviour-tree alternative in the same ecosystem, **bevior_tree**, remains actively updated (last push 2026-08-02, 57 stars) [[Source](https://github.com/hyranno/bevior_tree)] — a small but telling data point that in Bevy's third-party crate ecosystem (Chapter 40, §8), behaviour-tree-style composition currently has more maintenance momentum than utility scoring.

| System | Core idea | Reference implementation | Stars | License | Status |
|---|---|---|---|---|---|
| Navmesh | Polygon graph + A* + funnel smoothing | Recast & Detour | 7,854 | Zlib | Active |
| Behaviour Tree | Selector/sequence node composition | BehaviorTree.CPP | 4,153 | MIT | Active |
| Behaviour Tree (Godot) | Same model, Godot-native editor + GDScript tasks | LimboAI | 2,941 | MIT | Active |
| GOAP | STRIPS-style backward A* over world-state facts | cppGOAP / GPGOAP | 105 / 619 | Unclear / none | Minimal, unmaintained-at-scale |
| Utility AI | Per-action curve scoring, highest wins | big-brain (Bevy) | 1,300 | MIT/Apache-2.0 | Archived |

*Star counts and push dates verified 2026-08-10.*

---

## 3. How Modern AI Integrates and Subsumes Classical Game AI

Section 2's four systems are not being replaced wholesale by learned or generative AI — they are being repositioned. The dominant pattern in current research and early commercial deployment is **LLM-as-planner over classical-executor**: a language model proposes a goal, a command, or a plan fragment, and a conventional behaviour tree, GOAP planner, or utility selector remains the thing that actually runs every tick, walks the navmesh, and drives animation and physics. That is a strict refinement of what §1 used to call an "ordinary gameplay system" — the LLM sits upstream of it, at design time or at a slow decision cadence, not inside the per-frame execution path Section 2.1's navmesh queries and Section 2.2–2.4's decision architectures still own.

### 3.1 The LLM Generates the Classical Structure

Four recent papers make the shape of this concrete, and they split evenly between two directions the coupling can run. In the first direction, the LLM's output *is* a classical structure. Ito and Takahashi's "Game Agent Driven by Free-Form Text Command" has an LLM translate a player's free-form natural-language command into a *behavior branch* — a knowledge representation built directly on behaviour trees — which the game then executes; the authors validate the approach with a Pokémon-style simulation played by multiple human participants [[Source](https://arxiv.org/abs/2402.07442)]. Ao et al.'s "LLM-as-BT-Planner" (ICRA 2025) targets robot task planning rather than games, but the technique transfers directly: an LLM, combined with in-context learning and fine-tuning, generates behaviour trees for assembly tasks, trading some autonomy for the interpretability and debuggability a hand-authored BT already has [[Source](https://arxiv.org/abs/2409.10444)]. Shan and Michel's "Generative AI with GOAP for Fast-Paced Dynamic Decision-Making in Game Environments" (IEEE CoG 2024) addresses the coupling's central practical problem head-on — an LLM is too slow to call every tick — by having GOAP handle real-time action selection while the LLM contributes a slower strategic layer above it [[Source](https://ieeexplore.ieee.org/document/10645549)].

### 3.2 The Classical Structure Disciplines the LLM

The second direction inverts the relationship: the classical structure is not generated by the LLM but imposed on it. Kelley's "Behavior Trees Enable Structured Programming of Language Model Agents" uses BT composition (sequences, selectors, decorators) as scaffolding that constrains an otherwise brittle language-model agent, implemented as **Dendron**, a BSD-3-Clause Python library (96 stars, last pushed 2025-02-06) demonstrated on a chat agent, a robotic inspection system, and a safety-constrained agent [[Source](https://arxiv.org/abs/2404.07439)] [[Source](https://github.com/RichardKelley/dendron)]. That direction matters for the same reason Section 10 (AI Town) matters elsewhere in this chapter — it is a way to keep an LLM's output bounded and predictable, not a way to make gameplay logic smarter.

### 3.3 State Trees: A Data-Driven Successor Already Absorbing Part of BT's Role

Not all of the pressure on behaviour trees comes from LLMs — some of it is classical AI evolving on its own terms in a direction that happens to make LLM integration easier too. Unreal Engine's **State Tree**, developed since UE5's launch as an alternative to its own Behavior Tree system, merges BT-style Selectors with explicit States and Transitions from state-machine design, and replaces the BT's shared Blackboard with per-state, per-task data bindings (`InstancedPropertyBag`) [[Source](https://dev.epicgames.com/documentation/en-us/unreal-engine/state-tree-in-unreal-engine)]. The explicit, typed data bindings that make State Tree more maintainable for designers are the same property that makes it an easier target for a code- or plan-generating LLM than a Blackboard's loosely-typed key/value store — a state's required inputs and produced outputs are declared, not implicit in whatever a task happens to read and write at runtime.

### 3.4 Commercial Productization

Both major commercial pushes in this space keep the classical execution layer, market the LLM as a layer above it, and — as of this writing — do not publicly document the seam between the two precisely enough to describe further than that. **NVIDIA's ACE Game Agent SDK**, released as an open-source, on-device C/C++ framework at Unreal Fest and published at `github.com/NVIDIA/game-agent-sdk` (Apache-2.0, 14 stars, last pushed 2026-06-16), exposes an Agent API that "owns chat history and autonomously drives multistep, tool-assisted reasoning," alongside Chat and RAG APIs, with Unreal Engine 5 plugin support for Blueprint and C++ integration; its shipped example, a *Total War: PHARAOH* advisor, queries a RAG index over 1,200-plus interlinked game data tables to produce recommendations [[Source](https://developer.nvidia.com/blog/build-on-device-ai-companions-with-the-nvidia-ace-game-agent-sdk-and-unreal-engine-5-plugins/)] [[Source](https://github.com/NVIDIA/game-agent-sdk)]. Neither NVIDIA's blog post nor its ACE documentation site specifies how an Agent API decision reaches a Behavior Tree, State Tree, or navmesh call inside the host engine. **Inworld** markets its Unreal and Unity SDKs on the explicit claim that its Character Engine handles NPC personality, memory, and behaviour without the studio scripting dialogue trees or behaviour trees by hand [[Source](https://www.unrealengine.com/marketplace/en-US/product/inworld-ai-characters-dialogue)] — a vendor positioning claim, not a documented architecture, and one this book cannot verify further without a primary technical source. *Note: needs verification if either vendor publishes the integration architecture in more detail.*

### 3.5 NPC SDKs: A Wider Commercial Landscape

NVIDIA ACE and Inworld are two entries in a broader, fast-moving market of commercial NPC dialogue-and-behaviour SDKs, all converging on the same architectural claim — a hosted or on-device LLM drives character behaviour while the game engine keeps rendering, animation, and movement — and differing mainly in how open their engine integration is and how much of "behaviour" beyond dialogue they actually own.

**Convai** targets the same on-device-perception niche as ACE from the opposite direction: instead of shipping a general agent framework, it ships engine plugins with **environment vision** built in, so a Convai-driven character can see objects and player actions in the Unreal or Unity scene and comment on them, alongside a WebRTC transport chosen specifically to push conversational latency below 200ms [[Source](https://convai.com/blog/quick-setup-guide-conversational-ai-unreal-engine-convai-fab-plugin)]. Its Unreal Engine plugin is published in the open at `github.com/Conv-AI/Convai-UnrealEngine-SDK` — Apache-2.0, 61 stars, last pushed 2026-07-06 [[Source](https://github.com/Conv-AI/Convai-UnrealEngine-SDK)].

**Charisma.ai** is the one vendor here that does not claim to eliminate dialogue trees — it explicitly hybridizes them: a structured story graph (branches, player-choice nodes, game-state variables) authored by a writer, with generative dialogue filling in delivery and minor variation rather than replacing the graph outright [[Source](https://docs.charisma.ai/)]. Its Unreal SDK is likewise open — `github.com/charisma-ai/charisma-sdk-unreal`, MIT, 20 stars, last pushed 2025-09-08 [[Source](https://github.com/charisma-ai/charisma-sdk-unreal)] — distributed as a code plugin with pre-built binaries and full source, installed per-engine-version and enabled per-project.

The open/closed split is itself the most concrete, verifiable fact this section can offer where the vendors' behavioural claims cannot be checked: ACE, Convai, and Charisma.ai all publish their Unreal integration as inspectable source under permissive licenses, while Inworld's engine SDKs ship only through the closed Unreal Marketplace and Unity Asset Store listings, with no first-party public repository found for either engine plugin at the time of writing. *Note: needs verification* — this reflects a point-in-time GitHub search, not confirmation that Inworld has no source-available option.

One more vendor is worth naming without a full table row: **Artificial Agency**, founded by former DeepMind researchers and backed by Microsoft, markets a "Backstory Engine" for adaptive NPC behaviour combining reinforcement learning with LLM-driven narrative, with a stated Unreal integration — but its public technical documentation is materially thinner than the four vendors below, consisting mostly of press coverage and funding announcements rather than API references or open engine plugins, so this book cannot verify its integration architecture closely enough to table it alongside them.

| Vendor | Engine plugins | Integration | Behaviour approach | Repo / License | Stars | Status |
|---|---|---|---|---|---|---|
| NVIDIA ACE (Game Agent SDK) | Unreal Engine 5 (Blueprint + C++) | On-device C/C++ Agent/Chat/RAG APIs | LLM agent owns chat history, multistep tool-assisted reasoning | `NVIDIA/game-agent-sdk`, Apache-2.0 | 14 | Active, pushed 2026-06-16 |
| Convai | Unreal (Beta), Unity | WebRTC streaming, sub-200ms target, environment vision | Conversational AI plus scene perception | `Conv-AI/Convai-UnrealEngine-SDK`, Apache-2.0 | 61 | Active, pushed 2026-07-06 |
| Charisma.ai | Unreal, Unity (Plug 'n' Play) | Code plugin, pre-built binaries + source | Structured story graph plus generative delivery (hybrid, not full replacement) | `charisma-ai/charisma-sdk-unreal`, MIT | 20 | Active, pushed 2025-09-08 |
| Inworld | Unreal, Unity (Marketplace only) | Closed-source Character Engine, no public engine-plugin repo found | Generative personality/memory/behaviour, marketed as eliminating dialogue/behaviour trees | Not publicly available | — | Closed |

*Star counts, license fields, and push dates verified 2026-08-10.*

What both directions in Section 3.1–3.2 and all four commercial SDKs in Sections 3.4–3.5 share is the one invariant this chapter's Scope (§1) is actually built around: none of them move navmesh queries, animation blending, or physics off the classical execution layer Section 2 describes. An LLM proposing "flee to cover," or emitting a freshly-generated behaviour-tree fragment, or a State Tree transition, still hands off to the same A* or funnel-algorithm navmesh code and the same steering system a hand-authored AI would use. The render-loop-facing part is unchanged; only what decides to call it has grown a language model somewhere upstream.

---

## 4. The Environment API Standard: Gymnasium and PettingZoo

Every framework in this chapter either implements or deliberately mimics the Gymnasium interface. Understanding it is a prerequisite, and understanding what it *does not* specify — namely anything about GPUs or rendering backends — explains why the three architectures in this chapter can differ so radically while all claiming Gymnasium compatibility.

Gymnasium is the maintained successor to OpenAI Gym, developed under the Farama Foundation. As of 2026-08-08 the repository carries **12,298 stars**, was last pushed **2026-08-05**, and is MIT-licensed [[Source](https://github.com/Farama-Foundation/Gymnasium)]. PettingZoo is its multi-agent sibling: **3,488 stars**, last pushed **2026-08-03**, also MIT [[Source](https://github.com/Farama-Foundation/PettingZoo)].

### 4.1 The Single-Agent Contract

The entire single-agent API is four methods and one canonical loop:

```python
observation, info = env.reset()

episode_over = False
while not episode_over:
    action = env.action_space.sample()
    observation, reward, terminated, truncated, info = env.step(action)
    episode_over = terminated or truncated

env.close()
```

[[Source](https://gymnasium.farama.org/introduction/basic_usage/)]

That is the whole surface an engine bridge has to satisfy: `reset()` returns an initial observation, `step(action)` advances one decision step and returns a five-tuple, `close()` releases resources. `action_space` and `observation_space` describe the shapes. Everything else — vectorization, frame stacking, reward normalization, pixel preprocessing — is layered on as wrappers.

The interface's most consequential property is what it omits. There is no `render_async`, no notion of a frame budget, no way to express "the simulation has advanced 4 frames since your last action," and no way to signal that an observation is stale. A `step()` call is assumed to be a blocking function that returns promptly. Sections 5 and 6 are, in effect, two different accounts of the engineering required to make a running game engine pretend to be that function.

### 4.2 `terminated` vs `truncated`

The five-tuple's split of episode-end into two booleans is the most important correctness detail in the API, and the most frequently botched in engine bridges. `terminated` means the episode ended **for a reason internal to the MDP** — the agent died, reached the goal, entered an absorbing state. `truncated` means it ended **for a reason external to the MDP** — a time limit, a wrapper cutting the episode, the harness shutting down [[Source](https://gymnasium.farama.org/introduction/basic_usage/)].

The distinction is not bookkeeping; it changes the bootstrap target. On `terminated` the successor state's value is genuinely zero, whereas on `truncated` it is nonzero but unobserved, so a correct implementation bootstraps from the final state's value estimate. Conflating the two biases every value estimate near a time limit toward zero.

This matters here because bridges written against the older single-`done` Gym API often plumb one boolean into both slots. Godot RL Agents does exactly that: its `step_recv` returns the same value twice, with an in-source acknowledgement that the API has not been migrated:

```python
return (
    obs,
    reward,
    done,
    done,  # TODO update API to term, trunc
    info,
)
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/207b6f476f5846f33d08b92c7a350147e7b78bf5/godot_rl/core/godot_env.py)]

Anyone building a Godot environment with a time limit should treat that as a live correctness caveat and enforce truncation in a Python-side wrapper, where the two conditions can still be told apart.

### 4.3 Rendering Is Not in the Core Dependency Set

Gymnasium's `render_mode` accepts `"human"` (open a window, present at a human-viewable rate), `"rgb_array"` (return a NumPy frame from `render()`), or `None` (no rendering) [[Source](https://gymnasium.farama.org/introduction/basic_usage/)]. The mode is fixed at construction, not per-call, because backends generally need to allocate a surface up front.

The revealing detail is where the rendering dependency lives. Gymnasium's core `dependencies` list is `numpy`, `cloudpickle`, `typing-extensions`, and `farama-notifications` — no graphics library at all. `pygame-ce >=2.1.3` appears only in the `classic-control`, `box2d`, and `toy-text` optional-dependency extras [[Source](https://github.com/Farama-Foundation/Gymnasium/blob/main/pyproject.toml)]. The reference environments that do draw use a CPU-side SDL wrapper, opt-in.

That has a direct consequence for the rest of the chapter: **Gymnasium's rendering is a debugging and demonstration facility, not a data path.** Nothing in the core library rasterizes on a GPU, and nothing in the API contemplates an observation that lives in device memory. A framework that wants GPU-resident observations must either accept a readback (Sections 5 and 6) or abandon the graphics API entirely and rasterize in the same compute framework as the policy (Section 8). The `pyproject.toml` does carry `jax`, `torch`, and `array-api-compat` extras, signalling a move toward array-API-generic internals; how completely the environment set exploits them is a moving target and not asserted here.

### 4.4 PettingZoo: AEC and Parallel

PettingZoo generalizes the contract to multiple agents, and it offers two distinct APIs rather than one, because two genuinely different classes of game exist.

**AEC (Agent Environment Cycle)** is sequential: agents act one at a time and the environment updates after each individual step. This is the natural model for turn-based games and makes credit assignment explicit, since each agent's reward attaches to its own action rather than to a simultaneous joint action [[Source](https://pettingzoo.farama.org/api/aec/)]. Its loop uses an agent iterator and a `last()` call rather than a return value from `step()`:

```python
for agent in env.agent_iter():
    observation, reward, termination, truncation, info = env.last()

    if termination or truncation:
        action = None
    else:
        action = env.action_space(agent).sample()

    env.step(action)
```

[[Source](https://pettingzoo.farama.org/api/aec/)]

Two things are worth noting. `action_space` is a *function of the agent*, not a property, because heterogeneous agents are the normal case. And a terminated agent is stepped with `action = None` — it stays in the iteration until the environment removes it, which keeps the iterator's bookkeeping uniform but surprises people who expect the agent to simply disappear.

**Parallel** is simultaneous: all agents submit actions together and observations and rewards are returned at the end of the cycle. This models a partially-observable stochastic game (POSG) and is the right choice for real-time games where agents genuinely act at the same instant [[Source](https://pettingzoo.farama.org/api/parallel/)].

The Parallel API's hazard is that simultaneity is a fiction the implementation has to maintain. When two agents submit mutually exclusive actions in the same cycle — both moving into the same tile, both grabbing the same item — the outcome depends on the order in which the environment's internal update resolves them, a race-condition class PettingZoo's documentation notes AEC structurally avoids [[Source](https://pettingzoo.farama.org/api/aec/)]. For an engine bridge this is the same ordering ambiguity ECS schedules introduce whenever two systems mutate the same component set without an explicit constraint, and it deserves deterministic tie-breaking in the engine rather than being left to schedule order.

---

## 5. Synchronous Lockstep I: Godot RL Agents

Godot RL Agents is the most complete open-source engine integration in this space and the best case study for the synchronous shape. It bridges Godot 4 — 2D or 3D — to Python, and wraps four training frameworks: Stable-Baselines3, Sample Factory, Ray RLlib, and CleanRL. It is MIT-licensed, carries **1,556 stars**, was last pushed **2026-07-10**, installs as `pip install godot-rl`, and is described in a workshop paper at arXiv:2112.03636 [[Source](https://github.com/edbeeching/godot_rl_agents)].

### 5.1 The Inverted Client/Server Polarity

The first thing to understand is counter-intuitive. **The Python trainer is the TCP server; the game connects to it as a client.**

```python
def _start_server(self):
    # Either launch a an exported Godot project or connect to a playing project
    print(f"waiting for remote GODOT connection on port {self.port}")
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    address = "0.0.0.0" if self.host_binding else "127.0.0.1"
    server_address = (address, self.port)
    sock.bind(server_address)
    sock.listen(1)
    sock.settimeout(GodotEnv.DEFAULT_TIMEOUT)
    connection, client_address = sock.accept()
    return connection
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/207b6f476f5846f33d08b92c7a350147e7b78bf5/godot_rl/core/godot_env.py)]

The reasoning is practical. The trainer owns the experiment's lifecycle — how many parallel environments, on which ports, and when to tear them down — so making it the listener lets it bind before any game process exists and then launch N games that each dial in, which is far more robust than launching games and polling for their sockets. It also lets the trainer attach to a game already running in the editor, the standard iterate-on-your-environment workflow: run `gdrl`, watch it print `waiting for remote GODOT connection on port 11008`, and press play in Godot.

Note `sock.listen(1)` — a backlog of one, one game per port. Parallel environments mean parallel ports, and the protocol version and default port are pinned as class constants:

```python
class GodotEnv:
    MAJOR_VERSION = "0"  # Versioning for the environment
    MINOR_VERSION = "7"
    DEFAULT_PORT = 11008  # Default port for communication with Godot Game
    DEFAULT_TIMEOUT = 60  # Default socket timeout TODO
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/207b6f476f5846f33d08b92c7a350147e7b78bf5/godot_rl/core/godot_env.py)]

The 60-second default timeout is generous for a reason: on the far side of the socket is an engine that may be loading a scene, and on this side a trainer that may be mid-gradient-update. The project's FAQ states plainly that "it's normal for the Godot env to periodically freeze while the model is updating" [[Source](https://github.com/edbeeching/godot_rl_agents)] — the synchronous shape's defining characteristic, stated as a support answer. The engine is not running a game; it is a subroutine.

### 5.2 Framing and Message Types

The wire format is 4-byte little-endian length-prefixed JSON:

```python
def _send_string(self, string):
    message = len(string).to_bytes(4, "little") + bytes(string.encode())
    self.connection.sendall(message)
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/207b6f476f5846f33d08b92c7a350147e7b78bf5/godot_rl/core/godot_env.py)]

Length-prefixing is the correct choice, since TCP offers no message boundaries and a newline-delimited protocol would break the moment an observation contained an escaped newline. JSON trades throughput for debuggability and for GDScript's fast native codec, so no binary serialization layer has to be written twice.

The message set is small: `handshake`, `env_info`, `action`, `reset`, `call`, and `close`. `handshake` exchanges the version constants above. `env_info` is the interesting one — it is how the engine, not the Python code, defines the spaces. The trainer asks the game what its observation and action spaces are and constructs Gymnasium space objects from the reply:

```python
spaces.Discrete(v["size"])
spaces.Box(low=-1.0, high=1.0, shape=(v["size"],))
spaces.Box(low=0, high=255, shape=v["size"], dtype=np.uint8)
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/207b6f476f5846f33d08b92c7a350147e7b78bf5/godot_rl/core/godot_env.py)]

This inversion — engine as the source of truth for space definitions — is the right call for a game bridge: the environment designer works in the Godot editor, and requiring a parallel Python space declaration to stay in sync would guarantee drift. The `uint8` box with bounds 0–255 is the pixel-observation case of Section 5.4. `call` is the escape hatch, dispatching an arbitrary engine-side method, which is how curricula and evaluation harnesses change level, difficulty, or spawn conditions without extending the protocol.

Headless operation is a launch-flag concern, and the default matters:

```python
if show_window is False:
    launch_cmd.append("--disable-render-loop")
    launch_cmd.append("--headless")
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/207b6f476f5846f33d08b92c7a350147e7b78bf5/godot_rl/core/godot_env.py)]

`--headless` skips window and audio driver creation; `--disable-render-loop` stops Godot from drawing frames while still running the physics and script ticks. Together they mean that **the standard training configuration renders nothing at all.** The graphics stack is engaged for exactly one reason in this shape — vision-based observations — and if the observations are vectors, the GPU is idle.

Platform handling is mundane but worth knowing for CI: the launcher appends `.x86_64` on `linux`, `.app` on `darwin`, `.exe` on `win32`, so the exported binary must follow Godot's default export naming.

### 5.3 The Engine-Side Contract: AIController

On the Godot side, an environment is defined by extending `AIController2D` or `AIController3D` and implementing five methods. The pattern is compact enough to show in full:

```gdscript
extends AIController3D

var move_action : float = 0.0

func get_obs() -> Dictionary:
    var ball_pos = to_local(_player.ball.global_position)
    var ball_vel = to_local(_player.ball.linear_velocity)
    var obs = [ball_pos.x, ball_pos.z, ball_vel.x/10.0, ball_vel.z/10.0]

    return {"obs":obs}

func get_reward() -> float:
    return reward

func get_action_space() -> Dictionary:
    return {
        "move_action" : {
            "size": 1,
            "action_type": "continuous"
        },
        }

func set_action(action) -> void:
    move_action = clamp(action["move_action"][0], -1.0, 1.0)
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/main/docs/CUSTOM_ENV.md)]

Three details in this fragment are load-bearing and generalize to any engine bridge. `to_local(...)` converts world-space positions into the agent's local frame, so the network is handed translation and rotation equivariance rather than having to learn it. The division by 10.0 is manual normalization into roughly [-1, 1], done in GDScript rather than a Python wrapper so that the scaling lives with the definition of what the numbers mean. And `clamp` in `set_action` enforces the action bound engine-side rather than trusting the policy — a Gaussian policy over a `Box(-1, 1)` space emits out-of-range samples routinely, and clamping at the point of use means the physics never sees a nonsense impulse.

Reset is cooperative rather than commanded. The controller raises a flag and the gameplay code services it at a safe point in its own tick:

```gdscript
if ai_controller.needs_reset:
    ai_controller.reset()
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/main/docs/CUSTOM_ENV.md)]

A reset fired from the socket-reading code would mutate scene state at an arbitrary point relative to the physics step; deferring it to a flag the owning node checks means the reset happens where the game already expects state discontinuities.

A `Sync` node placed in the scene owns the connection and the mode selection, with Control Mode offering **Training**, **Human**, and **ONNX Inference** [[Source](https://github.com/edbeeching/godot_rl_agents/blob/main/docs/NODE_REFERENCE.md)]. Human mode exercises the same `get_obs`/`get_reward` paths with a person supplying the actions, which is how you discover that your reward function rewards something other than what you meant; the Python side reaches it through a `heuristic == "human"` branch.

The plugin ships pre-built sensor nodes so common observation types need not be hand-rolled: **RayCastSensor2D/3D** (collision mask, optional boolean class mask, `n_rays_width`/`n_rays_height`, ray length, cone width and height), **GridSensor2D/3D** (detection mask, cell dimensions, grid size), and **CameraSensor** [[Source](https://github.com/edbeeching/godot_rl_agents/blob/main/docs/NODE_REFERENCE.md)]. The raycast sensor is the workhorse: a cone of rays returning distance-plus-class is a compact, egocentric, resolution-independent encoding of local geometry that costs a fraction of a pixel observation's bandwidth. Reaching for `CameraSensor` when a raycast fan would do is the most common self-inflicted performance wound in this framework.

### 5.4 Pixel Observations and the Hex-Encoding Readback

When observations *are* pixels, the data path becomes the story, and it is expensive at every hop.

```python
@staticmethod
def _decode_2d_obs_from_string(hex_string, shape):
    return np.frombuffer(bytes.fromhex(hex_string), dtype=np.uint8).reshape(shape)
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/207b6f476f5846f33d08b92c7a350147e7b78bf5/godot_rl/core/godot_env.py)]

Read that backwards to recover the full chain. The GPU rasterizes into a render target. Godot reads that target back to CPU memory — a synchronizing readback, which stalls the graphics pipeline until the frame is complete. The bytes are hex-encoded into an ASCII string, **doubling their size**. That string is embedded in a JSON document, which may escape further. The JSON is length-prefixed and pushed through a loopback TCP socket. Python parses the JSON, un-hexes the string back to bytes, and finally wraps it in a NumPy array.

`np.frombuffer` is at least the one efficient step — it creates a view over the decoded buffer with no copy. But the hex round-trip is a 2x bandwidth multiplier on the single largest object in the protocol, and `bytes.fromhex` on a multi-megabyte string is real CPU time inside the trainer's step, per environment, per step.

The design is defensible — it is a JSON protocol, JSON has no binary type, and base64 would have needed a GDScript-side encoder where a hex encoder is trivial — but the consequence should shape decisions. With vector observations the socket is not the bottleneck; with pixels, the readback plus the hex-in-JSON round-trip will dominate, and the mitigations are to shrink the observation and to lean on Action Repeat.

### 5.5 Amortizing the Round-Trip: Action Repeat and Speed Up

The `Sync` node exposes two properties that exist purely to fight the synchronous shape's overhead.

**Action Repeat** batches the observation/reward/done exchange so that it happens once every *n* engine steps rather than every step, with the last action held in between; the documentation notes that less data is sent and training runs faster [[Source](https://github.com/edbeeching/godot_rl_agents/blob/main/docs/NODE_REFERENCE.md)]. It is the standard frame-skip technique doing double duty — it lowers decision frequency *and* divides the IPC cost by *n* — and on a pixel environment it is the single most effective lever available.

**Speed Up** raises Godot's physics tick multiplier so simulated time advances faster than wall-clock. This is free throughput only up to the point where the physics step becomes CPU-bound, and it is unsafe for any environment whose logic depends on real elapsed time or on frame-rate-dependent numerical behaviour. It is exposed on the Python side too, as one of the kwargs that get forwarded to the game process as CLI arguments:

```python
StableBaselinesGodotEnv(
    env_path="...",
    port=11008,
    env_seed=42,
    speedup=1,
    starting_level="RainbowRoad",
)
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/main/docs/CUSTOM_ENV.md)]

The kwargs-to-CLI-args mechanism is a pattern worth noting: arbitrary keyword arguments become command-line flags on the launched binary for the GDScript side to read. `starting_level` is not a framework concept but a user-defined argument, and that is how curriculum learning is wired up without touching the protocol.

Multi-policy training uses a per-environment policy assignment, defaulting to a single shared policy:

```python
agent_policy_names = ["shared_policy"] * num_envs
```

[[Source](https://github.com/edbeeching/godot_rl_agents/blob/207b6f476f5846f33d08b92c7a350147e7b78bf5/godot_rl/core/godot_env.py)]

Naming the default `"shared_policy"` rather than leaving it implicit is a small thing that makes the parameter-sharing decision visible in logs and configs, which matters when the answer to "why do both teams behave identically" is that they were one network.

### 5.6 Escaping Python: ONNX In-Engine Inference

Training needs Python. Shipping must not. The framework's answer is ONNX export: training with `--onnx_export_path=GameModel.onnx` writes the policy in a portable graph format, and the `Sync` node's ONNX Inference control mode loads it and runs the policy inside the engine with no Python process and no socket at all [[Source](https://github.com/edbeeching/godot_rl_agents)].

This is the resolution of the whole architecture: the synchronous socket bridge is a *training-time* scaffold, dismantled before ship. At runtime the policy is an ordinary engine-side function call in the gameplay tick — no IPC, no readback for vector observations, no external dependency.

Two constraints are real deployment blockers if unnoticed. ONNX export and inference require **the mono/.NET build of the Godot editor and a `.csproj` file** [[Source](https://github.com/edbeeching/godot_rl_agents/blob/main/docs/NODE_REFERENCE.md)], because inference runs through a .NET ONNX runtime binding rather than GDScript, so a project that stayed on the plain GDScript build must migrate first. And the framework describes ONNX support as **experimental**, supported only with Stable-Baselines3, RLlib, and CleanRL — Sample Factory is absent from that list [[Source](https://github.com/edbeeching/godot_rl_agents)]. Choosing a trainer therefore constrains the deployment path, a coupling worth discovering before rather than after training a policy.

---

## 6. Synchronous Lockstep II: bevy_rl

`bevy_rl` is the Rust-ecosystem counterpart: a Bevy plugin exposing a Gym-style environment over a local REST API. Its adoption is small — **103 stars** as of 2026-08-08 — and it should be described as niche rather than as a peer of Godot RL Agents. It earns its place in this chapter for exactly one reason: it makes the *opposite* choice on socket ownership, which turns the synchronous shape from a single design into a genuine axis with two ends.

`Cargo.toml` is the authoritative metadata. The crate is at `version = "0.19.0-rc1"` — a release candidate, not a release — licensed `"MIT OR Apache-2.0"`, and it tracks `bevy = "0.19"`, `wgpu = "29.0.3"`, `gotham = "0.7.1"`, and `hyper = "0.14.20"` [[Source](https://github.com/stillonearth/bevy_rl/blob/204be2c46b3dd2c488e8614a3581cfedee216cd4/Cargo.toml)]. The `hyper` pin carries an in-file comment worth quoting because it is unusually candid about technical debt: `# version is old because gotham no longer in development`. The HTTP framework this crate is built on is unmaintained, and that pins a transitive dependency several major versions back.

One licensing discrepancy is worth recording: GitHub's automatic detector reports only `apache-2.0`, while `Cargo.toml` declares the standard Rust dual license `MIT OR Apache-2.0`. The manifest is what `cargo` and every downstream tool read, so `MIT OR Apache-2.0` is the operative answer.

### 6.1 Opposite Socket Ownership

In Godot RL Agents, Python listens and the game dials in. In `bevy_rl`, **the game hosts an HTTP server on `localhost:7878` and the trainer is the client.** The surface is four endpoints:

| Method | Verb | URL |
|---|---|---|
| Camera Pixels | GET | `http://localhost:7878/visual_observations` |
| State | GET | `http://localhost:7878/state` |
| Reset Environment | GET | `http://localhost:7878/reset` |
| Step | GET | `http://localhost:7878/step?payload=ACTION` |

[[Source](https://github.com/stillonearth/bevy_rl)]

The crate's own integration test shows the concrete wire format, which the README's table leaves implicit. Actions go out as a JSON array in a query parameter and results come back as a parallel array of per-agent outcomes:

```rust
let actions_json = serde_json::to_string(&actions).unwrap();
let response =
    reqwest::blocking::get(format!("http://localhost:7878/step?payload={actions_json}"))
        .unwrap()
        .text()
        .unwrap();

let expected_response = r#"[{"is_terminated":false,"reward":0.0},{"is_terminated":false,"reward":0.0},{"is_terminated":false,"reward":0.0},{"is_terminated":false,"reward":0.0},{"is_terminated":false,"reward":0.0}]"#;
assert!(response == expected_response);
```

[[Source](https://github.com/stillonearth/bevy_rl/blob/204be2c46b3dd2c488e8614a3581cfedee216cd4/tests/test_rest_api.rs)]

Two consequences follow from this polarity. Against it: mutating simulation state through HTTP `GET` on a query string is not defensible on the merits, since `GET` is specified as safe and idempotent while `/step` and `/reset` are neither, and query strings have practical length limits that a `POST` body would not impose. The exchange is also a fresh request-response per step, with header parsing and — absent keep-alive — potentially a fresh TCP handshake, against Godot RL Agents' single persistent socket carrying compact length-prefixed frames. For it: the game owning the server is far better for *interactive* use, because anything speaking HTTP can attach to a running simulation — `curl`, a browser, a notebook, a dashboard, a language with no RL library at all — with no launcher integration and no port coordination. That accessibility is worth real per-step cost for exploratory work and tooling, and is not worth it for throughput.

Note also that `bevy_rl` returns only `is_terminated` per agent. There is no truncation channel in the wire format at all, so the Section 4.2 caveat applies in a stronger form: time-limit handling must live entirely in the client.

### 6.2 The Plugin Surface and the Pause Handshake

Setup is a resource and a plugin, both generic over the user's action and state types:

```rust
let ai_gym_state = AIGymState::<Actions, EnvironmentState>::new(AIGymSettings {
    num_agents: num_agents as u32,
    render_to_buffer: false,
    pause_interval: 0.0001,
    ..default()
});
app.insert_resource(ai_gym_state)
    .add_plugins(AIGymPlugin::<Actions, EnvironmentState>::default());
```

[[Source](https://github.com/stillonearth/bevy_rl/blob/204be2c46b3dd2c488e8614a3581cfedee216cd4/tests/test_rest_api.rs)]

The generic parameters are the interesting part. `Actions` and `EnvironmentState` are the user's own types; `EnvironmentState` must be `Serialize` and a Bevy `Resource`, and it *is* the observation space — the `/state` endpoint returns `serde_json` of it directly. Making the observation space a serializable ECS resource rather than a flat float vector is idiomatic Rust and considerably more pleasant than declaring shapes: the type is the schema.

`render_to_buffer: false` in the crate's own test is the same signal as Godot's headless default. This shape does not render unless asked. The test goes further and registers only `MinimalPlugins` plus `StatesPlugin`, `WindowPlugin`, `AssetPlugin`, and `ImagePlugin` — **no `RenderPlugin`, no GPU involvement at all** [[Source](https://github.com/stillonearth/bevy_rl/blob/204be2c46b3dd2c488e8614a3581cfedee216cd4/tests/test_rest_api.rs)].

The lockstep handshake runs through Bevy events and a state machine. `pause_interval` sets how often the simulation halts to offer a decision point — the README's example value of `0.01` corresponds to 100 Hz [[Source](https://github.com/stillonearth/bevy_rl)]. On pause, the plugin emits an event, and the user's system snapshots the environment state into the shared `AIGymState`:

```rust
fn bevy_rl_pause_request(
    mut pause_event_reader: MessageReader<EventPause>,
    ai_gym_state: Res<AIGymState<Actions, EnvironmentState>>,
    env_state: Res<EnvironmentState>,
) {
    for _ in pause_event_reader.read() {
        let mut ai_gym_state = ai_gym_state.lock().unwrap();
        ai_gym_state.set_env_state(env_state.clone());
    }
}
```

[[Source](https://github.com/stillonearth/bevy_rl/blob/204be2c46b3dd2c488e8614a3581cfedee216cd4/tests/test_rest_api.rs)]

`AIGymState` is behind a mutex because the gotham HTTP handlers run on their own threads while Bevy's schedule runs on its own — the lock is the boundary between the web server and the ECS world.

Applying actions is the mirror image. The control-event system iterates the per-agent action list, mutates the world, and ends with the line that actually unblocks the simulation:

```rust
for control in pause_event_reader.read() {
    let unparsed_actions = &control.0;
    for i in 0..unparsed_actions.len() {
        if let Some(unparsed_action) = unparsed_actions[i].clone() {
            match unparsed_action.as_str() {
                "UP" => env_state.agents[i].location.1 += 1.0,
                // ... remaining directions
                _ => {}
            }
        }
    }

    simulation_state.set(SimulationState::Running);
}
```

[[Source](https://github.com/stillonearth/bevy_rl/blob/204be2c46b3dd2c488e8614a3581cfedee216cd4/tests/test_rest_api.rs)]

Advancing the state machine, rather than returning from a function, is what resumes the game — the trainer's action does not "return a value" so much as release a paused schedule. Actions arrive as `Option<String>` per agent, with `None` for an agent that submitted nothing, which is how the framework represents a missing or terminated agent. The payload is an unparsed string the environment must deserialize itself; `bevy_rl` deserializes only its internal `AgentAction` wrapper. Rewards and termination are written back through `set_reward` and `set_terminated` on the shared state, with `reset`, `set_env_state`, and `send_reset_result` completing the surface [[Source](https://github.com/stillonearth/bevy_rl)].

### 6.3 Render-to-Buffer and the wgpu Readback

Turning on `render_to_buffer` engages the pixel path, and the README describes the mechanism in one sentence that carries the entire performance story: render targets are **copied each frame from GPU memory to RAM buffers** so that they can be served over the REST API [[Source](https://github.com/stillonearth/bevy_rl)].

This is the same collision as Section 5.4, expressed in wgpu terms. A `Camera` renders to a `Image` render target; the plugin issues a buffer-to-buffer copy from the GPU texture into a mapped staging buffer; the CPU maps it and reads the bytes. `wgpu`'s buffer mapping is asynchronous by design precisely because that transfer is slow, and a per-frame readback forces a pipeline synchronization point that stalls until the frame's work has drained. It is the classic anti-pattern in GPU programming — and in vision-based RL on a conventional engine it is unavoidable, because the trainer genuinely needs those pixels in host memory.

The one hard constraint documented here is a good example of a leaky abstraction that will bite: **width and height must exceed 256, otherwise wgpu will panic** [[Source](https://github.com/stillonearth/bevy_rl)]. This is almost certainly the `COPY_BYTES_PER_ROW_ALIGNMENT` requirement surfacing — `wgpu` requires texture-to-buffer copies to have row strides aligned to 256 bytes — and it is a direct conflict with what vision-based RL actually wants, since standard practice is to downsample observations aggressively — small frames train faster and cost less bandwidth. Here the graphics API's alignment rule forbids the low resolutions the algorithm prefers, so pixels must be rendered large and downsampled on the CPU afterwards, paying full readback bandwidth for a frame that will immediately be thrown away. That is a real and non-obvious cost of routing observations through a general-purpose renderer.

### 6.4 An Honest Maturity Assessment

Three signals should temper any decision to depend on this crate.

The version is `0.19.0-rc1`. The HTTP layer is built on `gotham`, which the manifest itself acknowledges is no longer developed, forcing an old `hyper` pin. And the README's example code is stale relative to the crate's own dependency pin: it uses `Camera3dBundle`, a camera `priority` field, `EventReader::iter()`, and push/pop `State` transitions — all pre-0.12 Bevy idioms — while `Cargo.toml` tracks Bevy 0.19. The `tests/` directory, by contrast, uses the current API: `MessageReader` rather than `EventReader`, `NextState`/`set` rather than push/pop, and `..default()` on the settings struct. **Read the tests, not the README.**

None of this makes the crate wrong to use for exploration, and the REST-server polarity is genuinely the better choice for interactive tooling. It does mean that a reader adopting it should expect to maintain the integration themselves.

---

## 7. In-Engine Inference, External Training: Unreal Engine Learning Agents

Learning Agents is Epic's own answer to the same problem Sections 5 and 6 solve for Godot and Bevy: how to train NPCs with reinforcement and imitation learning without leaving the engine's own C++/Blueprint workflow. It first shipped in **Unreal Engine 5.3**, roughly five months after its initial announcement [[Source](https://80.lv/articles/unreal-engine-5-3-s-new-plug-in-lets-you-train-npcs-via-machine-learning)], and remains present through at least **5.8**, where Epic's own plugin index still marks it **experimental** with the explicit warning to "use caution when shipping with it" [[Source](https://dev.epicgames.com/documentation/unreal-engine/API/PluginIndex/LearningAgents?lang=en-US)]. Unlike its earliest form, which required building the `ue5-main` branch from source, current releases ship it as a standard enable-from-`Edit > Plugins` plugin restartable into any C++ or Blueprint project [[Source](https://github.com/XanderBert/Unreal-Engine-Learning-Agents-Learning-Environment)].

### 7.1 Status, Modules, and Dependencies

The plugin is four modules — `LearningAgents`, `LearningAgentsTraining`, `LearningAgentsTrainingEditor`, and `LearningAgentsReplay` — built on three dependencies: **Learning Core**, a **Python ML Package** bundled with the engine, and **NNERuntimeBasicCpu**, Unreal's own Neural Network Engine (NNE) CPU inference runtime [[Source](https://dev.epicgames.com/documentation/unreal-engine/API/PluginIndex/LearningAgents?lang=en-US)]. A C++ project wires in `LearningAgentsTraining` and `LearningAgents` as public dependencies and `LearningAgentsTraining` as private in its `Build.cs` [[Source](https://github.com/XanderBert/Unreal-Engine-Learning-Agents-Learning-Environment)]. That NNE dependency is the load-bearing architectural fact of this section: it means a trained policy can run **inference** entirely inside the engine process, in contrast to Godot RL Agents' every-tick round trip to an external Python server (Section 5.2).

### 7.2 The Interactor Contract: Observations and Actions

The observation/action boundary lives in `ULearningAgentsInteractor`, and the contract is split into two phases that Gymnasium's single `step()` (Section 4.1) collapses into one. `SpecifyAgentObservation` and `SpecifyAgentAction` declare the *schema* — the shape and type of the observation and action vectors — once, up front; `GatherAgentObservation` and `PerformAgentAction` are the per-tick callbacks that actually read game state into that schema and write an action back out to the world [[Source](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/LearningAgentsInteractor?application_version=5.4)]. Separating declaration from per-tick execution is what lets the plugin allocate fixed-size observation/action buffers once rather than re-deriving shapes every frame — the same fixed-shape discipline Craftax's `vmap` tracing needs for an unrelated reason (Section 8.2, one section over). UE 5.4 added **Structured Observations and Actions**, letting these schemas be composed from typed sub-elements (vectors, enums, nested structs) instead of one flat float array, which is the detail that makes hand-authoring a non-trivial observation space tractable in Blueprint as well as C++ [[Source](https://forums.unrealengine.com/t/course-learning-agents-5-4/1806974)].

### 7.3 The Trainer: Rewards, Completions, and the External Process

`ULearningAgentsTrainer` is where reinforcement learning happens, and its documentation states its role plainly: "the core class for reinforcement learning training." A game implements `GatherAgentRewards` and `GatherAgentCompletions` to define the reward signal and episode-termination conditions; the trainer's `begin_training()`/`run_training()`/`end_training()` drive the loop, and `process_experience()` is the method that moves gathered observations, actions, and rewards into an experience buffer and pulls back an **updated policy from an external training process** [[Source](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/LearningAgentsTrainer?application_version=5.4)]. That external process is a PyTorch program implementing PPO and behavioral cloning, launched against the copy of Python UE ships internally (`Engine/Binaries/ThirdParty/Python3`), and connected back to the engine over a `LearningAgentsCommunicator` built on shared memory or a socket bridge [[Source](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Utilities/Struct/MakeLearningAgentsCommunicator)]. A dedicated `has_training_failed()` accessor exists specifically to detect when that external-process link has died [[Source](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/LearningAgentsTrainer?application_version=5.4)] — an explicit acknowledgment that, unlike inference, training genuinely is a fragile cross-process dependency. `ULearningAgentsPolicy` is the artifact that ties inference and training together: it is described simply as "a policy that maps from observations to actions," runnable locally via NNE once trained, and re-trainable by handing its weights to the external process again [[Source](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/LearningAgentsPolicy?application_version=5.4)]. A parallel `ULearningAgentsImitationTrainer` class runs behavioral cloning against recorded human demonstrations instead of a reward signal, for the tasks — driving a lap in the "Learning to Drive" tutorial is the canonical example — where writing a reward function is harder than just recording someone playing [[Source](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/LearningAgentsImitationTrainer?application_version=5.4)].

The net effect is a third architectural shape distinct from both engine-integration bridges already covered. Godot RL Agents and bevy_rl (Sections 5, 6) round-trip to Python for *every action*, so the engine blocks or interpolates around Python's response on every tick. AI Town (Section 10) decouples *all* of cognition from the simulation loop, tolerating LLM latencies of up to two minutes. Learning Agents splits the difference: **inference is local and synchronous** — a trained `ULearningAgentsPolicy` runs on an in-process NNE runtime with no round trip at all — while **training is external and episodic**, invoked in batches via `process_experience()` rather than on every tick. That asymmetry is deliberate: the expensive, latency-tolerant half (gradient computation, since a training run already expects to take minutes to hours) is farmed out to a full PyTorch process with GPU access, while the latency-critical half (choosing this frame's action) never leaves the engine.

### 7.4 Composing with Classical AI: Neural Networks Inside a Behaviour Tree

Learning Agents does not replace the classical stack from Section 2; the community tutorials treat it as one more leaf a behaviour tree can call. A trained policy is deployed as a `ULearningAgentsManagerComponent` attached to an `ALearningAgentsManager` actor, queried each tick for an action vector, and that vector is fed into a Blueprint or C++ behaviour tree task the same way a hand-authored decision would be [[Source](https://github.com/XanderBert/Unreal-Engine-Learning-Agents-Learning-Environment)]. This is the "classical structure disciplines the LLM" pattern from Section 3.2, transposed to reinforcement learning: the neural network supplies a value (steer left, accelerate), and the behaviour tree still owns sequencing, interruption, and the parts of NPC logic where a hand-authored guarantee is worth more than a learned one.

### 7.5 An Honest Maturity Assessment

Every primary source in this section carries the same qualifier: **experimental**. Epic's own plugin documentation repeats the shipping caution across 5.6, 5.7, and 5.8 without change, and community write-ups as late as the 5.5 course cycle still describe the documentation as sparse relative to the API surface [[Source](https://dev.epicgames.com/documentation/unreal-engine/API/PluginIndex/LearningAgents?lang=en-US)]. Unlike Godot RL Agents' Gymnasium-standard wire protocol (Section 5.2) or bevy_rl's REST interface (Section 6.2), Learning Agents has no publicly documented wire format for its shared-memory communicator, no reference to a supported OS/architecture matrix, and no GitHub issue tracker independent of the main `UnrealEngine` repository, which is closed to non-licensees. *Note: needs verification* — Epic does not publish stars/activity metrics comparable to the open-source frameworks elsewhere in this chapter, since the plugin lives inside the (access-gated) engine source tree rather than a standalone public repository. Treat it as the right tool specifically when the target is already an Unreal project and the team can accept an experimental dependency; it is not a reason to adopt Unreal for a project that would otherwise use Godot or Bevy.

---

## 8. Vectorized Headless GPU Simulation: Craftax

Craftax is the architectural break. It is a JAX reimplementation of the Crafter benchmark — an open-world survival game with a crafting tree and a 2D tile map — extended with mechanics inspired by NetHack. Because the whole environment is written in JAX, it can be `jit`-compiled and `vmap`-ed, which means thousands of independent game instances advance inside a single set of GPU kernel launches.

It is MIT-licensed, carries **435 stars**, was last pushed **2026-06-20**, publishes to PyPI as version **1.6.1**, supports Python 3.9–3.12, and was presented as an ICML 2024 Spotlight with the paper at arXiv:2402.16801 [[Source](https://github.com/MichaelTMatthews/Craftax)].

The headline number is the point of the whole exercise: Craftax "runs up to 250x faster than the Python-native original," and a PPO run consuming **one billion environment interactions finishes in under an hour on a single GPU** [[Source](https://arxiv.org/abs/2402.16801)]. The paper's abstract specifies "a single GPU" without naming the model, so the figure should be read as an order-of-magnitude claim rather than a reproducible benchmark on known hardware. The two names in the project are worth keeping straight: **Craftax-Classic** is the faithful JAX rewrite of Crafter, and **Craftax** is the extended, harder benchmark built on it [[Source](https://arxiv.org/abs/2402.16801)].

The relationship to robotics is exact: Craftax is to Crafter what MJX is to MuJoCo. Both take a simulator whose step function was imperative CPU code, rewrite it as pure functional array transformations, and let XLA compile a batched version onto accelerators. The technique is domain-independent, and the constraint it imposes — no data-dependent Python control flow, no dynamic shapes, no mutation — is what makes it a rewrite rather than a port. Chapter 211a covers the MJX side.

### 8.1 The gymnax Functional Interface

Craftax conforms to the `gymnax` interface, which is Gymnasium's contract restated functionally. The difference is total and it is the source of all the performance:

```python
rng = jax.random.PRNGKey(0)
rng, _rng = jax.random.split(rng)
_rngs = jax.random.split(_rng, 3)

# Create environment
env = make_craftax_env_from_name("Craftax-Symbolic-v1", auto_reset=True)
env_params = env.default_params

# Get an initial state and observation
obs, state = env.reset(_rngs[0], env_params)

# Pick random action
action = env.action_space(env_params).sample(_rngs[1])

# Step environment
obs, state, reward, done, info = env.step(_rngs[2], state, action, env_params)
```

[[Source](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/README.md)]

Compare this to Section 4.1. Gymnasium's `env.step(action)` mutates hidden state inside the environment object and returns an observation. Craftax's `env.step(rng, state, action, params)` is a **pure function**: state goes in, new state comes out, and the environment object holds nothing that changes. Randomness is an explicit argument — a split PRNG key, threaded through by the caller — rather than a hidden global generator.

Purity is the enabling condition for everything else. A pure function of arrays can be traced into a computation graph and handed to XLA; it can be `vmap`-ed, which rewrites every operation to work on a leading batch axis, turning one game into ten thousand with no change to the environment code; and critically it can be **fused with the policy update into a single compiled program**. Where the synchronous shape's training loop is Python alternating between a socket and a GPU, here the environment step, policy forward pass, advantage computation, and gradient update all live inside one `jit` boundary — nothing returns to the Python interpreter between kernel launches, and observations never leave device memory. The `PureJaxRL` line of work is built on this property, and libraries including `JaxMARL`, `JaxUED`, and `minimax` extend it to multi-agent and unsupervised-environment-design settings. `gymnax` itself carries **912 stars** and is actively maintained as of 2026-08-08 [[Source](https://github.com/RobertTLange/gymnax)].

Note that `done` is a single boolean, not the `terminated`/`truncated` pair. The functional formulation makes this less dangerous than it looks, since the caller owns the state and implements time limits in its own scan loop where the reason for ending is unambiguous — but the Section 4.2 bootstrapping care remains the caller's responsibility.

### 8.2 Rasterization as Array Code

Craftax offers two observation modes, and the pixel one is where this section's graphics relevance lies. `make_craftax_env_from_name` dispatches on names including `"Craftax-Symbolic-v1"` and `"Craftax-Pixels-v1"`, each with `AutoReset` and `NoAutoReset` variants [[Source](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/craftax/craftax_env.py)]. Symbolic gives a structured vector; pixels gives a rendered image.

The important fact is *how* the pixels are produced. The pixel environment imports a renderer factory from Craftax's own module and calls it inside the observation function:

```python
from craftax.craftax.renderer import make_craftax_pixel_renderer

class CraftaxPixelsEnv(EnvironmentAutoReset):
    def __init__(self, static_env_params: Optional[StaticEnvParams] = None):
        ...
        self._render_fn = make_craftax_pixel_renderer(BLOCK_PIXEL_SIZE_AGENT)

    def get_obs(self, state: EnvState) -> jax.Array:
        pixels = self._render_fn(state) / 255.0
        return pixels
```

[[Source](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/craftax/craftax/envs/craftax_pixels_env.py)]

And `get_obs` is called from inside `step_env`, wrapped in `lax.stop_gradient`:

```python
def step_env(
    self, key: jax.Array, state: EnvState, action: int, params: EnvParams
) -> Tuple[jax.Array, EnvState, float, bool, dict]:

    state, reward = craftax_step(key, state, action, params, self.static_env_params)

    done = self.is_terminal(state, params)
    info = log_achievements_to_info(state, done)
    info["discount"] = self.discount(state, params)

    return (
        lax.stop_gradient(self.get_obs(state)),
        lax.stop_gradient(state),
        reward,
        done,
        info,
    )
```

[[Source](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/craftax/craftax/envs/craftax_pixels_env.py)]

That `lax.stop_gradient` around the render output is decisive evidence, not just suggestion: `stop_gradient` is a JAX transformation that can only be applied to a traced JAX value. The renderer's output is therefore part of the same traced computation as the game logic, which means **rasterization here is JAX array code — not a graphics API call.** There is no Vulkan, no OpenGL, no wgpu, no framebuffer, no swapchain, and above all no readback. The observation is produced by array indexing and blending operations that compile into the same XLA program as the physics and the policy, and it is already sitting in device memory where the convolutional encoder wants it. Tile-based 2D rendering is unusually amenable to this treatment: compositing a tile map is a gather from a texture atlas followed by masked blends, which is exactly what array operations express well.

The observation shape confirms the tile-grid construction — an agent-view grid plus an inventory strip, scaled by a per-block pixel size, three channels, normalized to [0, 1]:

```python
def observation_space(self, params: EnvParams) -> spaces.Box:
    return spaces.Box(
        0.0,
        1.0,
        (
            (OBS_DIM[0] + INVENTORY_OBS_HEIGHT) * BLOCK_PIXEL_SIZE_AGENT,
            OBS_DIM[1] * BLOCK_PIXEL_SIZE_AGENT,
            3,
        ),
        dtype=jnp.float32,
    )
```

[[Source](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/craftax/craftax/envs/craftax_pixels_env.py)]

This is the cleanest possible contrast with Sections 5 and 6. Same nominal capability — pixel observations for vision-based RL — and a completely disjoint implementation stack. Where `bevy_rl` pays a GPU-to-CPU copy every frame and cannot render below 256 pixels because of a copy-alignment rule, Craftax pays nothing, has no alignment constraint, and can batch ten thousand renders into one kernel launch. The price is that the renderer had to be written from scratch in JAX, it can only do what array code can do, and no existing engine content can feed it. That is the trade: **rewrite the renderer in your compute framework, or pay the readback.**

Note that `BLOCK_PIXEL_SIZE_AGENT` is distinct from the larger block size used for human play, so the agent's observation is a genuinely low-resolution render rather than a downsample of a display frame — the resolution choice is made before rasterization, not after.

### 8.3 Optimistic Resets and Why Branching Is Expensive

The `auto_reset` flag on `make_craftax_env_from_name` looks like a convenience and is actually the entry point to the deepest constraint of this architecture. When `auto_reset=False`, the README directs the user to wrap the environment in either `OptimisticResetVecEnvWrapper` or `AutoResetEnvWrapper` [[Source](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/README.md)].

Resets need special machinery because `vmap` erases divergent control flow: every one of the thousands of batched environments executes the same instruction stream, so a single finished environment cannot run the world-generation code by itself. Generating a fresh world for the entire batch and masking the results is correct but enormously wasteful when one episode in ten thousand ended. "Optimistic" reset instead generates a small pool of fresh worlds and distributes them, accepting a small probability of running short in a given step — a bounded, correctness-neutral inefficiency traded for a large constant-factor saving. The principle generalizes: under `vmap`, **anything that only some environments need to do costs what it would cost for all of them**, so per-environment special cases must be reformulated as masked uniform work.

The same constraint explains why Craftax is a rewrite and not a port. Ordinary `if` statements on game state have to become a `jnp.where` or a `lax.select` with both branches computed, and fixed-size arrays replace growable lists — entity count, inventory size, and map dimensions all become compile-time constants, which is what `StaticEnvParams` in the constructor signature above exists to carry. This is why the technique has spread to a handful of benchmark environments rather than to engines generally.

### 8.4 Compilation Latency as the New Startup Cost

The synchronous shape's characteristic annoyance is a socket timeout. This shape's is a compiler.

Craftax's documentation warns that the first frame render takes roughly **30 seconds** and the first action roughly **20 seconds**, purely because JAX is tracing and XLA is compiling [[Source](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/README.md)]. Every subsequent call reuses the compiled binary and is fast. The practical consequences are that iteration on environment code is slow in a way that has nothing to do with runtime performance, and that anything which changes an array shape triggers a full recompile — which is why `StaticEnvParams` is separated from `EnvParams`. Runtime-varying values live in `EnvParams` and are traced; shape-determining values live in `StaticEnvParams` and are baked in.

A second, more mundane startup cost is texture loading: `export CRAFTAX_RELOAD_TEXTURES=true` forces the texture cache to be regenerated [[Source](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/README.md)]. The textures become constant arrays inside the compiled renderer, so they are prepared once and cached; the variable exists for when that cache goes stale after an upgrade.

Two documentation details matter before comparing numbers with anyone. Maximum achievable reward is **226**, and the scoreboards show how far from solved the benchmark is — the best reported result on the Craftax-1B budget is PPO-GTrXL at **18.3%**, and on Craftax-1M **6.6%** [[Source](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/README.md)]. The repository also documents score-changing errata: a plant-ripeness bug before version 1.5.0 and reward-on-death bugs before 1.6.0, so **results are not comparable across those version boundaries**.

---

## 9. Beyond Craftax: The Wider GPU-Native Environment Ecosystem

Craftax is one instance of a pattern, not a singleton. Since `vmap`-and-`jit` turned "rewrite the environment as a pure array function" into a repeatable technique rather than a one-off research trick, a small ecosystem of environment libraries has adopted it for domains far from tile-based survival games — continuous-control physics, board games, and combinatorial optimization all now have GPU-native implementations built on the identical purity/tracing contract Section 8.1 describes. Two frameworks depart from that recipe in instructive ways: one hand-writes CUDA kernels instead of tracing JAX, and one is a real GPU-resident ECS game engine rather than array code dressed as a renderer. A sixth framework belongs here as the deliberate counter-example — proof that the GPU-native pattern does not subsume every environment, and that a sufficiently well-engineered CPU thread pool remains the right answer when the state-transition function cannot be expressed as data-parallel array code at all.

### 9.1 The `vmap` Family: Brax, Pgx, and Jumanji

**Brax** is Google's JAX-native physics engine for continuous control — the domain Craftax's tile-grid technique does not cover. It bills itself as doing "massively parallel rigidbody physics simulation on accelerator hardware" and its documentation repeats the same order-of-magnitude claim Craftax makes for game logic: environments simulate "at millions of physics steps per second" on TPU, with full training runs completing in minutes rather than hours [[Source](https://github.com/google/brax)]. It is Apache-2.0, carries **3,214 stars**, and was last pushed **2026-08-06** — an actively maintained project, not a research artifact.

**Pgx** applies the same technique to discrete, complete-information games: **27 environments** including Chess, Shogi, Go, Backgammon, Othello, Hex, 2048, Connect Four, Kuhn and Leduc poker, mahjong, bridge, and the five-game MinAtar suite, all executing as `vmap`-ed JAX step functions [[Source](https://github.com/sotetsuk/pgx)]. It is Apache-2.0 with **635 stars**, but its last push was **2025-03-06** — over a year stale as of this writing, which puts it closer to dormant-but-usable than to the actively-developed tier Brax and Jumanji occupy. Anyone depending on it should verify current activity before assuming it is a maintained target.

**Jumanji** targets a third domain the other two don't: combinatorial optimization and routing — bin-packing, job-shop scheduling, the travelling salesman and capacitated vehicle routing problems, Sudoku, Rubik's Cube, and multi-agent scenarios like level-based foraging and search-and-rescue, 22 environments in total [[Source](https://github.com/instadeepai/jumanji)]. Its API is a deliberate hybrid — it adopts Gymnasium's registry and `render()` conventions while using DeepMind's `TimeStep` structure for the step return, so it reads as familiar to practitioners from either lineage rather than inventing a third convention [[Source](https://github.com/instadeepai/jumanji)]. It is Apache-2.0, **855 stars**, and was last pushed **2026-08-04** — actively maintained.

All three inherit exactly the constraints Section 8.3 describes, because they are built on the same technique: fixed-size arrays in place of growable collections, `jnp.where`/`lax.select` in place of data-dependent branching, and reset machinery that distributes a small pool of fresh episodes rather than regenerating the whole batch. Nothing about that constraint is domain-specific — it is a property of `vmap` itself, and it recurs identically whether the array being traced represents a tile map, a rigid-body scene graph, or a chessboard.

### 9.2 Hand-Written CUDA Batching: WarpDrive

WarpDrive is the manual alternative to `vmap`'s automatic one. Rather than tracing a pure JAX function and letting XLA compile the batched kernel, WarpDrive has the environment author write the step function directly in **CUDA C via PyCUDA, or in Numba** for JIT-compiled GPU code, with a thin Python layer for the training loop on top [[Source](https://github.com/salesforce/warp-drive)]. The published claims are aggressive: "orders-of-magnitude faster RL compared to CPU simulation + GPU model implementations," at least **100x throughput over CPU-based counterparts**, roughly a 10x speedup against a 16-CPU node in multi-agent Tag benchmarks, and scaling to **10,000+ concurrent environment replicas on a single A100** [[Source](https://github.com/salesforce/warp-drive)]. It accompanied a JMLR 2022 paper on end-to-end multi-agent deep RL entirely on GPU.

The honest caveat belongs up front rather than as a footnote: WarpDrive's repository is **archived and read-only**, with its last push on **2025-05-01** [[Source](https://github.com/salesforce/warp-drive)]. It is BSD-3-Clause and carries 503 stars, but it is no longer a live dependency choice — it belongs in this section on technical merit, as the clearest example of the direct-kernel-authorship path, rather than as something to build on today. Treat it the way Section 11 treats the historical Minecraft-agent projects: valuable to read, not safe to adopt.

### 9.3 A GPU-Native ECS Engine: Madrona

Madrona is architecturally distinct from every other framework in this section, and the distinction matters specifically to a graphics-stack audience: it is not JAX array code wearing a renderer's clothes, the way Craftax's tile compositor is. It is a real **game engine** — described in its own SIGGRAPH paper as "the first fully-GPU accelerated ECS implementation that natively supports batch environment simulation" [[Source](https://madrona-engine.github.io/)], published as "An Extensible, Data-Oriented Architecture for High-Performance, Many-World Simulation" (Shacklett et al., ACM Transactions on Graphics, SIGGRAPH 2023).

The engine keeps its Entity Component System model but runs it on a CUDA GPU backend with unified-memory support (Linux only, Volta-or-newer, CUDA 12.4/12.8+), alongside a CPU backend that executes the identical code path for debugging and visualization [[Source](https://madrona-engine.github.io/)]. It exports ECS component state as PyTorch tensors directly from GPU memory, and it ships its own **high-throughput batch rasterizer** rather than delegating to Vulkan or a conventional engine's render graph — a genuine batch-rendering pipeline, not the array-blend trick Craftax's tile renderer performs. The performance claims are correspondingly framed against CPU game-engine baselines rather than against other JAX libraries: **100–300x over open-source CPU baselines and 5–33x over an optimized 32-thread CPU implementation**, with measured throughput of 2 million steps/second on an OpenAI Hide-and-Seek-style environment, 40 million steps/second on Overcooked-AI, and 20 million steps/second on Hanabi, all on a single RTX 4090 [[Source](https://madrona-engine.github.io/)]. Example projects distributed with the engine include `madrona_escape_room` and `madrona_puzzle_bench`. It is MIT-licensed, carries 513 stars, and was last pushed **2025-11-03**.

Madrona's most direct connection to the rest of this book is `madrona_mjx`, a bridge that hands MuJoCo MJX's physics state to Madrona's batch renderer so that vision-based policies can train against physically simulated scenes at "rendering FPS in the hundreds of thousands," with integrations into MuJoCo Playground and Brax training pipelines [[Source](https://github.com/shacklettbp/madrona_mjx)]. That bridge is itself now deprecated in favor of **MJWarp**, MuJoCo's own native high-throughput batch renderer — the ecosystem's center of gravity has moved from a separate GPU-ECS renderer bolted onto MJX toward a renderer built into MuJoCo's own project, which Chapter 211a's MJX coverage is the place to follow further [[Source](https://github.com/shacklettbp/madrona_mjx)].

### 9.4 The Deliberate Counter-Example: EnvPool

Every framework so far answers the question "how do I put the environment on the GPU." EnvPool answers a different one: what do you do when the environment fundamentally cannot be rewritten as array or ECS code at all? Atari's ALE emulator, VizDoom, Google Research Football, and a full StarCraft II binary are irreducibly branching C++ and cannot be traced, `vmap`-ed, or expressed as batched entity updates — there is no rewrite that turns an emulator into array math.

EnvPool's answer is to make the CPU path itself extremely well-engineered: a **C++ thread-pool-based vectorized execution engine**, exposed to Python through pybind11, supporting 16-plus environment families including Atari, MuJoCo, Classic Control, VizDoom, DeepMind Control Suite, Box2D, Google Research Football, Procgen, Minigrid, and MetaWorld [[Source](https://github.com/sail-sg/envpool)]. On an NVIDIA DGX-A100 with 256 CPU cores it reaches roughly **1 million frames per second on Atari and 3 million frames per second on MuJoCo** — throughput that rivals the GPU-native frameworks above for the class of environment that can accept it [[Source](https://github.com/sail-sg/envpool)]. It is Apache-2.0, carries **1,494 stars**, and was last pushed **2026-07-17** — actively maintained.

The lesson EnvPool teaches is the one this section opened with: the choice is not "rewrite in JAX or accept a slow Pythonic bottleneck" as a universal binary. It is "match the technique to whether the environment's state-transition function can be expressed as data-parallel code at all." Gymnasium's own reference environments (Section 4.3) mostly cannot — many wrap exactly the emulators and physics engines EnvPool targets — which is why EnvPool, not a `vmap` rewrite, is the answer for that tier of environment, while Craftax-style rewrites are the answer for environments simple enough to express as array or ECS state.

### 9.5 A Different Axis: PufferLib as Wrapper-Plus-Trainer

PufferLib does not fit cleanly into the "how is the environment itself parallelized" axis the rest of this section is organized around, because it operates one layer up: it is a vectorization wrapper and training stack that sits in front of whatever environments already exist, rather than a technique for writing new GPU-native ones. A game exposes a native PyTorch network and a short per-environment binding; PufferLib supplies fast shared-memory vectorization across worker processes, one-line compatibility wrappers for Gymnasium, PettingZoo, and other RL library conventions, and its own PPO implementation, **PuffeRL**, reporting up to 20,000,000 steps/second in an optimized ~5,000-line CUDA build — roughly 4x the throughput of the pure-PyTorch build, which tops out around 5,000,000 steps/second [[Source](https://puffer.ai/docs.html)]. It carries **6,240 stars** and is MIT-licensed — the highest star count of any framework in this section, with a push date (2026-08-04) within days of Brax's (2026-08-06).

PufferLib also ships its own native environment suite, **Ocean** — "20+ environments from simple arcade games to massively multiagent sims" [[Source](https://puffer.ai/docs.html)], and as of the 4.0 release the repository lists over 60 individual Ocean environments, including native C ports of Craftax, NetHack, and a scaled-down Neural MMO (`nmmo3`) [[Source](https://github.com/PufferAI/PufferLib/tree/4.0/ocean)]. Earlier PufferLib releases (through roughly 1.0) wrapped external environment families — Atari, Procgen, the original NetHack Learning Environment — through Gymnasium- and PettingZoo-compatible bindings, but the project's own FAQ states that generic third-party wrapping was dropped because it was "100x+ slower than PufferLib is now," with C/C++-level hooks for Atari and NetHack planned but not yet shipped [[Source](https://puffer.ai/docs.html)]. In its current form, then, PufferLib's contrast with the rest of this section is less about wrapping arbitrary existing environments and more about training throughput: Brax, Pgx, Jumanji, Madrona, and EnvPool each answer "how do I make this specific class of environment fast," while PufferLib answers "how do I get one fast, uniform training loop across a large native suite" — a narrower value proposition than a general-purpose wrapper, but one still worth reaching for when Ocean already covers the target game genre.

### 9.6 Choosing Among These Frameworks

The decision is less "which framework is fastest" — several report step rates within an order of magnitude of each other on incomparable hardware and environment classes — and more "which axis does my environment's state-transition function actually sit on":

- **The state-transition function can be written as pure array code.** Use the `vmap` family (§9.1): Brax for continuous-control physics, Jumanji for combinatorial/routing problems, Pgx only after confirming its current activity given its 2025-03-06 last push. This is the right choice when the environment is being designed or rewritten from scratch and the team is already JAX-fluent, matching Craftax's own trade in Section 8.
- **The state-transition function needs full engine features — rendering, physics, ECS composition — not just array math.** Use Madrona (§9.3), or its declining-relevance bridge to MuJoCo MJX if the target is physics-based visual RL specifically, noting MJWarp is now the maintained path for that combination.
- **The environment already exists as an external emulator or physics engine that cannot be rewritten** (Atari's ALE, VizDoom, a full StarCraft II binary, MuJoCo XML models). Use EnvPool (§9.4) if the environment family is in its supported list; if not, and CUDA authorship expertise is available in-house, WarpDrive's technique remains valid even though the reference implementation is archived — treat it as a pattern to reimplement, not a dependency to add.
- **The target game genre is already covered by PufferLib's own Ocean suite** (arcade-style games, small-scale multi-agent sims, a native Craftax port) **and the priority is training throughput and a single, well-tuned PPO implementation over authoring a new environment.** Use PufferLib (§9.5) directly rather than as a wrapper on top of the other four — as of the 4.0 release it is closer to a competing native-environment suite plus trainer than a universal adapter, since its earlier Gymnasium/PettingZoo wrapping of external libraries like Atari and NetHack was dropped for being too slow and has not yet been replaced.

| Framework | Technique | Domain | Stars | License | Status |
|---|---|---|---|---|---|
| Brax | JAX/XLA, `vmap` | Continuous-control physics | 3,214 | Apache-2.0 | Active |
| Pgx | JAX/XLA, `vmap` | Discrete board/card games | 635 | Apache-2.0 | Dormant (~17 mo) |
| Jumanji | JAX/XLA, `vmap` | Combinatorial optimization | 855 | Apache-2.0 | Active |
| WarpDrive | Hand-written CUDA/Numba | Multi-agent RL | 503 | BSD-3-Clause | Archived |
| Madrona | GPU-native ECS engine | General batch simulation | 513 | MIT | Active |
| EnvPool | CPU thread pool (C++) | Emulator-bound environments | 1,494 | Apache-2.0 | Active |
| PufferLib | Native C environments (Ocean) + CUDA trainer | Arcade/small multi-agent, native Craftax port | 6,240 | MIT | Active |

*Star counts and push dates verified 2026-08-10.*

---

## 10. Decoupled Asynchronous Agent Loops: AI Town

AI Town is the third shape. It is a maintained reimplementation of the "Generative Agents: Interactive Simulacra of Human Behavior" work (arXiv:2304.03442) — the "Smallville" simulation in which LLM-driven characters remember events, reflect on them, form plans, and hold conversations. It is MIT-licensed, carries **10,274 stars**, and was last pushed **2026-06-12** [[Source](https://github.com/a16z-infra/ai-town)].

Its stated secondary goal is explicit and explains its existence: to provide a JavaScript/TypeScript framework for this class of simulation, since most simulators in the space — including the original paper's — are written in Python [[Source](https://github.com/a16z-infra/ai-town)]. The stack is Convex for the game engine, database, and vector search; PixiJS for rendering; and a pluggable LLM backend.

### 10.1 The Arithmetic of Decoupling

The claim that the LLM loop is decoupled from the simulation tick does not need to be argued. It is visible in four constants in one file:

```ts
// The interval at which the engine updates the world state
export const TICK = 16;
export const STEP_INTERVAL = 1000;

// Don't run a turn of the agent more than once a second.
export const AGENT_WAKEUP_THRESHOLD = 1000;

export const ACTION_TIMEOUT = 120_000; // more time for local dev
```

[[Source](https://github.com/a16z-infra/ai-town/blob/main/convex/constants.ts)]

The simulation tick is **16 ms**. The timeout allowed for a single agent action — which is to say, for an LLM call — is **120,000 ms**. That is a factor of **7,500**. There is no engineering that puts a 120-second operation inside a 16-millisecond tick. The architecture is not decoupled as a design preference; it is decoupled because the alternative is arithmetically impossible.

`STEP_INTERVAL = 1000` and `AGENT_WAKEUP_THRESHOLD = 1000` establish the intermediate layer. The engine simulates in 16 ms increments but commits work in 1-second batches, and an agent's decision logic is rate-limited to at most once per second. So there are three clocks, not two: a 16 ms physics/movement tick, a ~1 s agent-decision tick, and an unbounded-up-to-120 s cognition tick. Movement and collision stay smooth while thinking happens on a clock three to four orders of magnitude slower.

The remaining timing constants describe how the system survives operations that slow:

```ts
export const IDLE_WORLD_TIMEOUT = 5 * 60 * 1000;
export const WORLD_HEARTBEAT_INTERVAL = 60 * 1000;
export const MAX_STEP = 10 * 60 * 1000;
export const ENGINE_ACTION_DURATION = 30000;
export const MAX_PATHFINDS_PER_STEP = 16;
export const INPUT_DELAY = 1000; // Wait for 1s after sending an input to the engine.
```

[[Source](https://github.com/a16z-infra/ai-town/blob/main/convex/constants.ts)]

`MAX_PATHFINDS_PER_STEP = 16` is a per-step budget on the most expensive non-LLM operation, which is the standard way a simulation keeps a variable-cost subsystem from blowing the frame — the same reasoning as a navmesh query budget in a conventional game. `INPUT_DELAY = 1000` is the acknowledgement that this is a distributed system: an input submitted to the engine is not immediately visible, and the code waits a second rather than assuming read-after-write. `IDLE_WORLD_TIMEOUT` and `WORLD_HEARTBEAT_INTERVAL` are cost control, discussed in Section 10.5.

The conversation constants show the same philosophy applied to LLM spend:

```ts
export const MAX_CONVERSATION_MESSAGES = 8;
export const MAX_CONVERSATION_DURATION = 10 * 60_000;
export const TYPING_TIMEOUT = 15 * 1000;
export const CONVERSATION_DISTANCE = 1.3;
export const INVITE_ACCEPT_PROBABILITY = 0.8;
```

[[Source](https://github.com/a16z-infra/ai-town/blob/main/convex/constants.ts)]

Every one of these is a bound on a process that would otherwise be unbounded. Conversations cap at 8 messages because each message is an LLM call and a two-agent conversation left alone will continue indefinitely. `CONVERSATION_DISTANCE = 1.3` ties cognition to spatial proximity, so agents only converse when adjacent — which is what keeps LLM calls proportional to interesting spatial events rather than to agent count squared. `INVITE_ACCEPT_PROBABILITY = 0.8` injects deliberate stochasticity so the social graph does not become deterministic. And a `TYPING_TIMEOUT` of 15 seconds is a UI affordance for latency: the typing indicator is how a multi-second generation is made to read as natural behaviour rather than as a hang. That is a genuinely transferable lesson — **the animation and UI layer is where LLM latency gets hidden**, and a game with LLM NPCs needs idle behaviours, thinking animations, and conversational filler as load-bearing architecture rather than polish. AI Town supplies exactly that in the form of an `ACTIVITIES` list — reading a book, daydreaming, gardening, 60 seconds each — that agents perform when they have nothing else to do.

### 10.2 Engine, Inputs, and the Transactional Backend

The Convex backend is not merely a database in this design; it is the game engine's execution substrate. The repository organizes it as `convex/engine/` for the generic simulation loop, `convex/aiTown/` for game-specific logic, and `convex/agent/` for cognition, alongside `schema.ts`, `crons.ts`, `http.ts`, and `constants.ts` [[Source](https://github.com/a16z-infra/ai-town)].

The property that makes this work is transactional. Convex is described as offering immediate consistency, serializable isolation, and automatic conflict resolution via optimistic multi-version concurrency control (OCC/MVCC) [[Source](https://github.com/a16z-infra/ai-town)]. Serializable isolation is a strong guarantee to have available inside a simulation loop, and it is what lets agent cognition run concurrently with the tick without explicit locking. An LLM action that started 40 seconds ago and is now writing its result commits as a serializable transaction; if the world moved underneath it, the OCC layer detects the conflict and the write retries rather than corrupting state. In the synchronous shape this problem cannot arise, because nothing is concurrent. Here, concurrency is the entire point, and the correctness burden is discharged by the database rather than by hand-written synchronization.

The `INPUT_DELAY` constant is the visible edge of this: the engine consumes *inputs* from a queue rather than accepting direct state mutation. An agent that decides to walk somewhere submits an input; the engine applies it on its next step. That indirection is what allows a slow, asynchronous decision-maker and a fast, synchronous simulator to share a world.

### 10.3 Memory, Retrieval, and the Embedding Contract

`convex/agent/` contains four files, and their names are the cognitive architecture: `conversation.ts`, `memory.ts`, `embeddingsCache.ts`, and `schema.ts` [[Source](https://github.com/a16z-infra/ai-town)].

Memory works by embedding-based retrieval, and the retrieval constant is annotated in a way that reveals the design:

```ts
// How many memories to get from the agent's memory.
// This is over-fetched by 10x so we can prioritize memories by more than relevance.
export const NUM_MEMORIES_TO_SEARCH = 3;
```

[[Source](https://github.com/a16z-infra/ai-town/blob/main/convex/constants.ts)]

Three memories reach the prompt, but thirty candidates are retrieved, because pure vector similarity is the wrong ranking function for a memory system. The generative-agents design scores retrieval on relevance *plus* recency *plus* importance, and you cannot apply recency or importance weighting to results a vector index has already truncated. Over-fetching by 10x and re-ranking in application code is the practical way to get a composite ranking out of a similarity index — a pattern worth stealing for any retrieval system whose true ranking function is not cosine distance.

`embeddingsCache.ts` exists because embedding calls cost money and latency, and the same text gets embedded repeatedly. Caching embeddings keyed by text content is the cheapest single optimization available in this class of system.

The one configuration hazard is dimensional. The default models are `llama3` for chat and `mxbai-embed-large` for embeddings, and `EMBEDDING_DIMENSION` must match whichever embedding model is configured [[Source](https://github.com/a16z-infra/ai-town)]. Change the embedding model without changing that constant and the vector index is silently wrong; change it after data exists and the previously-stored vectors are incomparable with new ones. Embedding dimension is effectively part of the schema, and it is not something to adjust casually.

`VACUUM_MAX_AGE` is set to two weeks, so old data is reaped rather than accumulating indefinitely [[Source](https://github.com/a16z-infra/ai-town/blob/main/convex/constants.ts)] — memory growth being the other unbounded process in a long-running agent simulation.

### 10.4 The Rendering Half: PixiJS

The client is where this chapter's usual concerns reappear. All interactions, background music, and rendering happen on a `<Game/>` component powered by **PixiJS** [[Source](https://github.com/a16z-infra/ai-town)].

PixiJS is a 2D WebGL/WebGPU renderer built around batched sprite drawing, and it is the correct tool for this workload for reasons that have nothing to do with AI. A tile-map town with a few dozen animated characters is thousands of quads sharing a handful of textures; the entire performance question is how few draw calls those quads can be issued in. Pixi's sprite batcher coalesces same-texture sprites into single draw calls, and with a packed atlas the whole scene can be a small number of GPU submissions.

The division of labour is clean and is the shape's main structural advantage. The server owns simulation and cognition; the client owns rendering and interpolates between the discrete positions the server publishes. Because the server commits in 1-second batches while the client renders at display rate, client-side interpolation is doing real work — the smooth walking a viewer sees is a rendering-layer reconstruction of a coarsely-sampled server trajectory. This is ordinary networked-game architecture, and its presence here is the reason the LLM latency is invisible: the render loop was never in the same process as the thinking, so it cannot be blocked by it.

Contrast with Sections 5 and 6 one more time. There, the AI blocks the engine — the engine is documented as freezing during model updates. Here the AI *cannot* block the renderer, because they are separated by a network boundary and a database. For any application where a human is watching, that is the architecture to want.

### 10.5 Local Inference and Cost Control

AI Town runs against Ollama for local inference by default, with Together.ai or any OpenAI-compatible API as alternatives, configured through Convex environment variables such as `npx convex env set OLLAMA_HOST` [[Source](https://github.com/a16z-infra/ai-town)]. Music generation uses Replicate's MusicGen. The self-hosted Docker layout exposes the frontend on 5173, the backend on 3210, the HTTP API on 3211, and a dashboard on 6791.

The local-inference option is not a nicety. A simulation of a few dozen agents holding conversations generates a continuous stream of completions, and against a metered API that is an open-ended bill for something no user may be watching. Local inference converts that to a fixed hardware cost. Chapter 124 covers the serving stack — quantization, KV-cache management, batching, GPU memory pressure — that determines how many concurrent agents a given machine can actually sustain, and that capacity is the real constraint on simulation size here. It also means the LLM and the renderer are competing for the same GPU, which is a resource-contention problem the synchronous and vectorized shapes do not have in this form.

The cost-control mechanisms are explicit in the constants and the cron schedule. `IDLE_WORLD_TIMEOUT = 5 * 60 * 1000` means the simulation pauses after five minutes with an idle window, and a "stop inactive worlds" cron in `convex/crons.ts` reaps abandoned simulations [[Source](https://github.com/a16z-infra/ai-town)]. `WORLD_HEARTBEAT_INTERVAL = 60 * 1000` is the liveness signal that keeps an observed world running. Together they implement the rule that **cognition is only paid for when someone is watching** — a design constraint with no analogue in RL training, where the entire point is to run unobserved for as long as possible, and a good illustration of how differently the two use cases are shaped. `MAX_HUMAN_PLAYERS = 8` bounds the interactive side.

### 10.6 Running and Customizing an Instance

The two supported paths diverge on where Convex, the transactional backend from Section 10.2, actually runs. The standard path (`git clone`, `npm install`, `npm run dev`) requires a free Convex cloud account and serves the frontend at `http://localhost:5173`; the self-hosted path (`docker compose up --build -d`) runs Convex itself in a container alongside the frontend (5173), backend RPC (3210), HTTP API (3211), and a dashboard (6791) [[Source](https://github.com/a16z-infra/ai-town/blob/main/README.md)]. The self-hosted path needs one extra step the cloud path does not: generating an admin key with `docker compose exec backend ./generate_admin_key.sh` and placing it in `.env.local` as `CONVEX_SELF_HOSTED_ADMIN_KEY`, regenerated every time the containers are torn down and brought back up [[Source](https://github.com/a16z-infra/ai-town/blob/main/README.md)]. Choose the cloud path for a quick trial or for the smoothest iteration loop; choose Docker Compose when the deployment target is a machine without a Convex account, or when the LLM backend is also local and everything should run air-gapped except optional model downloads.

Wiring up an LLM is a second, independent step from deployment. For the default Ollama backend: install Ollama, run `ollama serve`, then `ollama pull llama3` to fetch the default chat model — `mxbai-embed-large` is the paired default embedding model. When self-hosting with Docker, Convex cannot reach a host-installed Ollama by its usual `localhost` address, so it must be pointed at the Docker host explicitly: `npx convex env set OLLAMA_HOST http://host.docker.internal:11434` [[Source](https://github.com/a16z-infra/ai-town/blob/main/README.md)]. Switching to OpenAI, Together.ai, or any other OpenAI-compatible endpoint (Anthropic, Groq, Azure) is a matter of setting `OPENAI_API_KEY`/`TOGETHER_API_KEY`/`LLM_API_KEY` via `npx convex env set` and changing one line in `convex/util/llm.ts` — `EMBEDDING_DIMENSION` must be reassigned to match whichever provider's embedding model is active, echoing the dimensional hazard from Section 10.3. Changing provider or embedding model after a simulation has accumulated memories requires wiping the database (`npx convex run testing:wipeAllTables`) and starting over, since old vectors are not comparable to new ones [[Source](https://github.com/a16z-infra/ai-town/blob/main/README.md)].

Building a custom simulation rather than running the stock one touches three files. Characters, their descriptions, and their initial plans live in `data/characters.ts`; editing them requires a full `wipeAllTables` and re-run of `npm run dev` because character data is only uploaded to Convex on initial load, not read live [[Source](https://github.com/a16z-infra/ai-town/blob/main/README.md)]. Sprite sheets are wired per-character in the same file via a `spritesheetData` reference to a per-character definition file. The map itself is authored externally in the [Tiled](https://www.mapeditor.org/) editor, exported as a two-layer (`bgtiles`, `objmap`) JSON file, and converted with the repository's `data/convertMap.js` script into the same format `convex/init.ts` already loads for the stock `gentle.js` map. None of this touches the cognition or engine code from Sections 9.1–9.3 — the simulation loop, memory retrieval, and timing constants are orthogonal to world content.

For an operator tuning an already-running instance, three knobs matter more than the rest, in the order most deployments should touch them. First, `IDLE_WORLD_TIMEOUT` and the "stop inactive worlds" cron in `convex/crons.ts` are the primary spend control — leave them as-is for a public demo, but comment out the cron entirely for a long-running unattended simulation meant to be observed later rather than live [[Source](https://github.com/a16z-infra/ai-town/blob/main/README.md)]. Second, `NUM_MEMORIES_TO_SEARCH` (§10.3) trades conversation quality for prompt size and latency — the README's own recommendation for a slow local model is to drop it from 3 to 1 [[Source](https://github.com/a16z-infra/ai-town/blob/main/README.md)]. Third, `npx convex run testing:stop` and `testing:resume` are the emergency brake for runaway activity or cost, halting the engine and all agents without tearing down the deployment, which is the operational analogue of the `IDLE_WORLD_TIMEOUT` auto-pause but under manual control [[Source](https://github.com/a16z-infra/ai-town/blob/main/README.md)].

Whether to reach for AI Town's architecture at all is a separate question from how to run it. A simpler LLM-agent loop — one Python or TypeScript process that ticks a handful of NPCs synchronously, calling out to an LLM and blocking on the result — is the right choice when there are only a few agents, no human is watching in real time, or the game already has its own authoritative simulation loop that a second, Convex-owned one would conflict with; it is also far less operational surface area than standing up Convex, a vector index, and a cron-based reaper. AI Town's decoupled shape earns its complexity specifically when the three constants from Section 10.1 all apply at once: dozens of agents rather than a handful, a human watching a live render at display rate (Section 10.4) that must never stall on an LLM call, and conversations or memories that need to persist and stay consistent across a session longer than any single process's lifetime. Section 12 frames this as one of three integration shapes; choosing it should follow from needing genuine decoupling, not from AI Town being the most fully-featured option on offer.

---

## 11. The Historical Tier: Lineage, Not Recommendations

The projects below established the research agenda that Sections 8 through 10 build on. Most are no longer maintained. They are listed with last-commit dates so the maintenance status is a fact rather than an impression. Star counts and dates verified 2026-08-08.

| Project | Stars | Last commit | Contribution |
|---|---|---|---|
| `joonspk-research/generative_agents` | 21,899 | **2023-08-11** | Reference code for the Smallville paper; the memory/reflection/planning architecture AI Town reimplements |
| `MineDojo/Voyager` | 7,121 | **2023-07-27** | LLM-driven open-ended Minecraft agent; iterative code generation as the action space with a growing skill library |
| `MineDojo/MineDojo` | 2,244 | **2023-08-29** | Minecraft task suite with a large internet-scale knowledge base for open-ended agent evaluation |
| `minerllabs/minerl` (`dev`) | 969 | **2025-01-22** | Minecraft RL environment and human-demonstration dataset; the NeurIPS MineRL competition platform |
| `danijar/crafter` | 577 | **2023-12-13** | The original Python survival benchmark that Craftax reimplements in JAX |
| `mindagent/mindagent` | 102 | **2024-06-12** | LLM multi-agent gaming-coordination research artifact |

The cluster of 2023 dates around the Minecraft-and-LLM-agents projects marks the end of a specific research moment. These repositories remain valuable as reading — Voyager's use of *generated code* as the action space, rather than a fixed discrete set, is still one of the more interesting ideas in LLM agents, and MineRL's human-demonstration dataset was foundational for imitation learning at scale — but they are pinned to dependency versions that were current three years ago. Standing a Python 3.8-era Minecraft RL stack up today is an archaeology project. Treat them as papers with source attached.

Two specific cautions. MineRL's most recent activity is on a `dev` branch rather than a release, and at roughly eighteen months dormant it is best described as inactive rather than merely stale. MindAgent's repository has **no license field at all** and appears to be essentially README-only — it is a research artifact, not a usable dependency, and without a license it is not safely reusable regardless of maintenance state. Licenses for the projects in this table are not asserted here; anyone considering reuse should read the repository's own license file, since several are not the standard permissive licenses one might assume.

Unity ML-Agents (`Unity-Technologies/ml-agents`, **19,612 stars**, actively pushed) belongs in this discussion as the commercially-backed precedent for the synchronous engine-bridge shape that Godot RL Agents occupies in the open-source world. Its license is reported inconsistently by automated tooling and is not asserted here — *Note: needs verification* against the repository's own `LICENSE.md` before any reuse decision.

### 11.1 Maintained but CPU-Only: OpenSpiel and Melting Pot

Two DeepMind libraries are genuinely maintained and deserve separating from the table above.

**OpenSpiel** (**5,395 stars**) is a framework for research in games, with a large collection of game implementations and algorithms spanning perfect and imperfect information, sequential and simultaneous move, and cooperative and zero-sum settings. It is a C++ core with Python bindings.

**Melting Pot** (**860 stars**) is a suite of multi-agent scenarios for evaluating social behaviour — cooperation, competition, deception, reciprocity, free-riding — with an emphasis on generalization to unfamiliar co-players rather than performance against training partners.

Both are CPU-only in the sense that matters for this chapter: their environments execute on the host, and neither offers a `vmap`-style GPU-native batched step of the kind Section 8 describes. Scaling means process-level parallelism across cores, not kernel-level parallelism across a device. For OpenSpiel this is entirely appropriate — the state of a card game is a small tagged union, not an array, and the branching is irreducible. Melting Pot's grid worlds are closer to the class of thing a JAX rewrite could target, and the fact that a maintained, well-designed multi-agent social benchmark exists on CPU while Craftax demonstrates 250x from a rewrite marks one of the clearer open opportunities in this space. *Note: needs verification* — licenses for both are commonly given as Apache-2.0 but were not re-confirmed against the repositories in this pass.

---

## 12. Three Integration Shapes, Compared

The chapter's argument, stated compactly.

| | **Synchronous lockstep** | **Vectorized headless** | **Decoupled async** |
|---|---|---|---|
| Examples | Godot RL Agents, `bevy_rl` | Craftax (and MJX, Ch211a) | AI Town |
| Coupling | Separate processes, blocking IPC | One compiled program, no IPC | Network + transactional DB |
| Who owns the socket | Python (Godot RL); game (`bevy_rl`) | No socket | HTTP + DB |
| Rendering during training | Off by default (`--headless`, `render_to_buffer: false`) | No graphics API at all | Always on, client-side |
| Pixel observation path | GPU→CPU readback, then IPC encode | Stays in device memory | N/A (human viewer, not agent) |
| Parallelism | One process per env | `vmap` over thousands per device | One world per simulation |
| Characteristic latency | 16 ms tick, blocked on gradient step | ~30 s compile, then microseconds | 16 ms tick / 1 s agent / 120 s LLM |
| Existing engine content | Reusable | Must be rewritten | Reusable |
| Dominant cost | IPC and readback | Compilation, batch-uniform work | LLM inference |

Three cross-cutting observations.

**"Step-and-render" is a misnomer for the synchronous shape.** Both frameworks default to not rendering: Godot RL Agents appends `--headless --disable-render-loop` whenever `show_window is False`, and `bevy_rl`'s own test sets `render_to_buffer: false` and registers `MinimalPlugins` with no `RenderPlugin`. Rendering is engaged only for vision-based observations, through `CameraSensor` or a render-target image. Calling it step-and-*simulate* is accurate and clarifies where the graphics stack actually enters.

**The readback is the real boundary between shapes one and two.** When a conventional engine produces pixel observations, the frame must cross from device to host memory, and that crossing is a pipeline synchronization plus a bandwidth cost plus, in Godot's case, a 2x hex-encoding penalty inside a JSON document. `wgpu`'s 256-byte row-alignment requirement even forbids the small frames that vision-based RL prefers, forcing full-size renders that are immediately downsampled. Craftax's answer is not to optimize the readback but to delete it, by reimplementing rasterization as JAX array code inside the traced program. That is a rewrite of the renderer, and it buys the elimination of an entire class of cost.

**Socket ownership is a genuine axis, not an implementation detail.** Godot RL Agents makes Python the server because the trainer owns the experiment lifecycle and must bind ports before games exist. `bevy_rl` makes the game the server because that lets anything speaking HTTP inspect and drive a live simulation. The first is better for throughput and orchestration; the second is better for interactivity and tooling. Both are correct for their priority, and knowing which priority you have determines which polarity you want.

### 12.1 Choosing a Shape

The choice is largely forced by what you already have and what you need. An existing game with existing content admits only **synchronous lockstep**, since it is the one shape that reuses the engine, the assets, and the gameplay code — the cost is IPC, mitigated by preferring structured observations over pixels and by Action Repeat. A benchmark that must absorb billions of steps justifies **vectorized headless**, whose price is a full rewrite in a functional array framework with every data-dependent branch reformulated as masked uniform work. Agents that reason or converse in language have no option but **decoupled async**: a 120-second cognition step cannot be made synchronous with a 16 ms frame, so the three clocks and the transactional store between them are structural requirements rather than design choices.

---

## Integrations

**Chapter 40 — Bevy and wgpu** establishes the ECS architecture and the `wgpu` render-graph model that `bevy_rl` plugs into: `AIGymState` is an ordinary Bevy `Resource` behind a mutex, the pause/control handshake is built from Bevy's buffered-message system and `NextState` transitions, and the `render_to_buffer` path is a `wgpu` texture-to-buffer copy whose 256-byte row-alignment requirement is what produces the crate's documented "width and height must exceed 256" constraint. Section 6.3's readback analysis is a direct application of that chapter's discussion of GPU-to-host transfer costs and mapped staging buffers.

**Chapter 41 — Godot 4 RenderingDevice** covers the rendering architecture that Godot RL Agents' engine-side plugin sits on top of. The `--headless --disable-render-loop` pair in Section 5.2 is precisely a decision not to instantiate that chapter's rendering pipeline while keeping the physics and script ticks alive, and `CameraSensor`'s pixel observations are a viewport-texture readback through the same `RenderingDevice` abstraction — which is why Section 5.4's data path begins with a synchronizing GPU read before the hex-in-JSON encoding ever happens.

**Chapter 69 — Omniverse, PhysX, and GR00T** describes the industrial-scale version of Section 8's vectorized-simulation argument. Isaac Lab applies the same insight — thousands of environment copies advancing in lockstep on one device, with the policy update fused into the same program — to robot learning with PhysX on a proprietary renderer and a USD scene model, where Craftax applies it to a 2D tile game with a hand-written JAX rasterizer. Reading them together separates the architectural idea from its implementation.

**Chapter 211a — MuJoCo and MJX** is the closest structural analogue to Section 8 and the best companion to it. MJX is MuJoCo's step function rewritten as pure JAX so that XLA can compile and `vmap` thousands of parallel physics environments onto a single GPU; Craftax does the identical transformation to game logic and tile rendering. The constraints transfer exactly — fixed-size arrays, no data-dependent Python branching, masked uniform work instead of per-environment special cases — which is why both projects needed reset machinery of the kind Section 8.3 describes rather than a simple conditional. Section 9.1's Brax is built by the same organization on the same technique for a different physics domain, and Section 9.3's Madrona connects to MJX directly through the now-deprecated `madrona_mjx` bridge, superseded by MuJoCo's own MJWarp batch renderer.

**Chapter 240 — Isaac Sim and GR00T Synthetic Data Generation** covers the case where the *renderer's output is the product* rather than an observation to be consumed by a policy in the same process. That inverts this chapter's economics: Sections 5.4 and 6.3 treat GPU-to-CPU readback as pure overhead to be minimized or deleted, whereas a synthetic-data pipeline exists precisely to write rendered frames out at scale, making high-fidelity rasterization and domain randomization the goal rather than a tax.

**Chapter 124 — Local LLM Inference** is the serving stack beneath Section 10. AI Town's default configuration points at Ollama with `llama3` for chat and `mxbai-embed-large` for embeddings, and the practical size of a simulation is set by that chapter's concerns — quantization choices, KV-cache management, continuous batching, and GPU memory pressure — since concurrent agent conversations are concurrent generation requests. It also explains the contention this shape uniquely suffers: the LLM and the PixiJS client can end up competing for the same device.

---

## References

- [Ito, Takahashi — "Game Agent Driven by Free-Form Text Command: Using LLM-based Code Generation and Behavior Branch"](https://arxiv.org/abs/2402.07442) — LLM translates free-form natural-language commands into behaviour-tree-based "behavior branches"; validated on a Pokémon-style simulation (§1.1)
- [Ao, Wu, Wu, Swikir, Haddadin — "LLM-as-BT-Planner: Leveraging LLMs for Behavior Tree Generation in Robot Task Planning"](https://arxiv.org/abs/2409.10444) — ICRA 2025; LLM plus in-context learning and fine-tuning generates behaviour trees for robot assembly planning (§1.1)
- [Shan, Michel — "Generative AI with GOAP for Fast-Paced Dynamic Decision-Making in Game Environments"](https://ieeexplore.ieee.org/document/10645549) — IEEE CoG 2024; GOAP handles real-time action selection while an LLM contributes a slower strategic layer, addressing LLM call latency in a fast-paced game loop (§1.1)
- [Kelley — "Behavior Trees Enable Structured Programming of Language Model Agents"](https://arxiv.org/abs/2404.07439) — behaviour-tree composition (sequences, selectors, decorators) as scaffolding that constrains an LLM agent, implemented as the Dendron library (§1.1)
- [Recast & Detour](https://github.com/recastnavigation/recastnavigation) — de facto standard navmesh generation (Recast) and runtime pathfinding/steering (Detour) library; Zlib, 7,854 stars, last pushed 2026-02-27 (§2.1)
- [BehaviorTree.CPP](https://github.com/BehaviorTree/BehaviorTree.CPP) — reference open-source C++17 behaviour-tree library with XML tree definitions and the Groot2 visual editor, also the planning backbone of ROS 2's Nav2; MIT, 4,153 stars, last pushed 2026-08-05 (§2.2)
- [Unreal Engine — Behavior Tree overview (State Tree comparison)](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-state-tree-in-unreal-engine) — first-party Blackboard-backed, node-graph Behavior Tree editor positioned against its State Tree successor (§2.2)
- [LimboAI](https://github.com/limbonaut/limboai) — de facto behaviour-tree and state-machine standard for Godot, a C++ GDExtension/module plugin with GDScript-authored tasks/states and a built-in visual debugger; MIT, 2,941 stars, last pushed 2026-08-04 (§2.2)
- [Orkin — "Building the AI of F.E.A.R. with Goal-Oriented Action Planning"](https://www.gamedeveloper.com/design/building-the-ai-of-f-e-a-r-with-goal-oriented-action-planning) — GOAP's canonical origin; the three-state FSM wrapped around a real-time STRIPS-style planner (§2.3)
- [cppGOAP](https://github.com/cpowell/cppGOAP) — minimal open-source GOAP implementation; 105 stars, license undeclared per GitHub's detector (§2.3)
- [GPGOAP](https://github.com/stolk/GPGOAP) — minimal open-source GOAP implementation; 619 stars, no license file (§2.3)
- [Mark — Infinite Axis Utility System](https://www.gameai.com/iaus.php) — GDC 2013 formalization of utility AI as modular reasoners combining an unbounded number of scored considerations (§2.4)
- [big-brain](https://github.com/zkat/big-brain) — Bevy/Rust data-driven utility AI library (Scorers and Actions); dual MIT/Apache, 1,300 stars, **archived**, last pushed 2025-10-07 (§2.4)
- [bevior_tree](https://github.com/hyranno/bevior_tree) — Bevy/Rust behaviour-tree crate; 57 stars, last pushed 2026-08-02 (§2.4)
- [Unreal Engine — State Tree in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/state-tree-in-unreal-engine) — State Tree's Selector/State/Transition model and `InstancedPropertyBag` per-state data bindings, replacing the Behavior Tree's shared Blackboard (§3.3)
- [NVIDIA — "Build On-Device AI Companions with the NVIDIA ACE Game Agent SDK and Unreal Engine 5 Plugins"](https://developer.nvidia.com/blog/build-on-device-ai-companions-with-the-nvidia-ace-game-agent-sdk-and-unreal-engine-5-plugins/) — Agent/Chat/RAG API overview and the *Total War: PHARAOH* advisor example (§3.4)
- [NVIDIA ACE Game Agent SDK](https://github.com/NVIDIA/game-agent-sdk) — open-source, on-device C/C++ agent framework with Unreal Engine 5 plugin support; Apache-2.0, 14 stars, last pushed 2026-06-16 (§3.4)
- [Inworld AI Characters & Dialogue — Unreal Marketplace listing](https://www.unrealengine.com/marketplace/en-US/product/inworld-ai-characters-dialogue) — vendor positioning for the closed-source Character Engine (personality, memory, behaviour) (§3.4)
- [Convai — "Quick Setup Guide: Conversational AI in Unreal Engine (Convai Fab Plugin)"](https://convai.com/blog/quick-setup-guide-conversational-ai-unreal-engine-convai-fab-plugin) — environment-vision-equipped character plugin and its sub-200ms WebRTC transport target (§3.5)
- [Convai — Unreal Engine SDK](https://github.com/Conv-AI/Convai-UnrealEngine-SDK) — open-source Unreal plugin; Apache-2.0, 61 stars, last pushed 2026-07-06 (§3.5)
- [Charisma.ai — Documentation](https://docs.charisma.ai/) — hybrid structured story-graph plus generative-delivery dialogue architecture (§3.5)
- [Charisma.ai — Unreal SDK](https://github.com/charisma-ai/charisma-sdk-unreal) — open-source Unreal code plugin distributed with pre-built binaries and full source; MIT, 20 stars, last pushed 2025-09-08 (§3.5)
- [Gymnasium](https://github.com/Farama-Foundation/Gymnasium) — Farama Foundation single-agent RL environment API; MIT, 12,298 stars, last pushed 2026-08-05 (§4)
- [Gymnasium `pyproject.toml`](https://github.com/Farama-Foundation/Gymnasium/blob/main/pyproject.toml) — core dependency set contains no graphics library; `pygame-ce` appears only in `classic-control`, `box2d`, and `toy-text` extras (§4.3)
- [Gymnasium — Basic Usage](https://gymnasium.farama.org/introduction/basic_usage/) — canonical `reset`/`step`/`close` loop, `terminated` vs `truncated`, and the `render_mode` values (§4.1–4.3)
- [PettingZoo](https://github.com/Farama-Foundation/PettingZoo) — multi-agent RL API; MIT, 3,488 stars, last pushed 2026-08-03 (§4.4)
- [PettingZoo — AEC API](https://pettingzoo.farama.org/api/aec/) — sequential agent-environment-cycle model, `agent_iter`/`last`, and the race conditions AEC avoids (§4.4)
- [PettingZoo — Parallel API](https://pettingzoo.farama.org/api/parallel/) — simultaneous-action POSG formulation (§4.4)
- [Godot RL Agents](https://github.com/edbeeching/godot_rl_agents) — Godot 4 ↔ Python RL bridge; MIT, 1,556 stars, last pushed 2026-07-10; four framework wrappers, ONNX export, arXiv:2112.03636 (§5)
- [`godot_rl/core/godot_env.py`](https://github.com/edbeeching/godot_rl_agents/blob/207b6f476f5846f33d08b92c7a350147e7b78bf5/godot_rl/core/godot_env.py) — version constants, `_start_server`, 4-byte little-endian length-prefixed JSON framing, hex pixel decoding, headless launch flags, and the `done, done  # TODO update API to term, trunc` return (§5.1–§5.6)
- [Godot RL Agents — `docs/CUSTOM_ENV.md`](https://github.com/edbeeching/godot_rl_agents/blob/main/docs/CUSTOM_ENV.md) — `AIController3D` skeleton, cooperative `needs_reset` pattern, and kwargs-to-CLI-args environment configuration (§5.3, §5.5)
- [Godot RL Agents — `docs/NODE_REFERENCE.md`](https://github.com/edbeeching/godot_rl_agents/blob/main/docs/NODE_REFERENCE.md) — `Sync` node control modes, Action Repeat, Speed Up, ONNX model path and its .NET requirement, and the RayCast/Grid/Camera sensor nodes (§5.3, §5.5, §5.6)
- [bevy_rl](https://github.com/stillonearth/bevy_rl) — Bevy plugin exposing a Gym-style environment over REST; 103 stars, last pushed 2026-06-22; REST endpoint table, `AIGymSettings`, and the "width and height should exceed 256" wgpu constraint (§6)
- [bevy_rl `Cargo.toml`](https://github.com/stillonearth/bevy_rl/blob/204be2c46b3dd2c488e8614a3581cfedee216cd4/Cargo.toml) — `0.19.0-rc1`, `MIT OR Apache-2.0`, bevy 0.19 / wgpu 29.0.3 / gotham 0.7.1, and the `# version is old because gotham no longer in development` comment (§6, §6.4)
- [bevy_rl `tests/test_rest_api.rs`](https://github.com/stillonearth/bevy_rl/blob/204be2c46b3dd2c488e8614a3581cfedee216cd4/tests/test_rest_api.rs) — current-API plugin setup with `MinimalPlugins` and `render_to_buffer: false`, the `MessageReader<EventPause>`/`EventControl` handshake, and the `/step?payload=` and `/state` wire formats (§6.1–§6.2)
- [80.lv — "Unreal Engine 5.3's New Plug-in Lets You Train NPCs via Machine Learning"](https://80.lv/articles/unreal-engine-5-3-s-new-plug-in-lets-you-train-npcs-via-machine-learning) — Learning Agents' initial announcement and its UE 5.3 ship date (§7)
- [Unreal Engine — Learning Agents Plugin Index](https://dev.epicgames.com/documentation/unreal-engine/API/PluginIndex/LearningAgents?lang=en-US) — module list (`LearningAgents`, `LearningAgentsTraining`, `LearningAgentsTrainingEditor`, `LearningAgentsReplay`), the Learning Core/Python ML Package/NNERuntimeBasicCpu dependencies, and the persistent "experimental... use caution when shipping" status through UE 5.8 (§7, §7.1, §7.5)
- [XanderBert — Unreal Engine Learning Agents Learning Environment](https://github.com/XanderBert/Unreal-Engine-Learning-Agents-Learning-Environment) — community write-up and sample project covering `Build.cs` module wiring and the manager/component actor architecture (§7, §7.1, §7.4)
- [Unreal Engine — `ULearningAgentsInteractor` Python API](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/LearningAgentsInteractor?application_version=5.4) — `SpecifyAgentObservation`/`SpecifyAgentAction` schema declaration vs. `GatherAgentObservation`/`PerformAgentAction` per-tick execution (§7.2)
- [Unreal Engine Forums — "Course: Learning Agents 5.4"](https://forums.unrealengine.com/t/course-learning-agents-5-4/1806974) — UE 5.4's Structured Observations and Actions feature (§7.2)
- [Unreal Engine — `ULearningAgentsTrainer` Python API](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/LearningAgentsTrainer?application_version=5.4) — `GatherAgentRewards`/`GatherAgentCompletions`, `process_experience()`, and `has_training_failed()` (§7.3)
- [Unreal Engine — `MakeLearningAgentsCommunicator`](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Utilities/Struct/MakeLearningAgentsCommunicator) — the shared-memory/socket bridge connecting the engine to the external PyTorch training process (§7.3)
- [Unreal Engine — `ULearningAgentsPolicy` Python API](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/LearningAgentsPolicy?application_version=5.4) — the observation-to-action mapping runnable locally via NNE once trained (§7.3)
- [Unreal Engine — `ULearningAgentsImitationTrainer` Python API](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/LearningAgentsImitationTrainer?application_version=5.4) — behavioral-cloning trainer used in the "Learning to Drive" tutorial (§7.3)
- [Craftax](https://github.com/MichaelTMatthews/Craftax) — JAX reimplementation of Crafter extended with NetHack-inspired mechanics; MIT, 435 stars, last pushed 2026-06-20, PyPI 1.6.1 (§8)
- [Craftax `README.md`](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/README.md) — gymnax-interface usage snippet, optimistic-reset wrappers, `CRAFTAX_RELOAD_TEXTURES`, JIT compile latencies, maximum reward 226, scoreboards, and pre-1.5.0/pre-1.6.0 errata (§8.1, §8.3, §8.4)
- [Craftax `craftax_env.py`](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/craftax/craftax_env.py) — environment-name dispatch confirming `Craftax-Symbolic-v1` and `Craftax-Pixels-v1` with AutoReset/NoAutoReset variants (§8.2)
- [Craftax `craftax_pixels_env.py`](https://github.com/MichaelTMatthews/Craftax/blob/c3c2e0d038c4e641f9481320c158f457f30c28f3/craftax/craftax/envs/craftax_pixels_env.py) — `make_craftax_pixel_renderer` called from `get_obs` inside `step_env` under `lax.stop_gradient`, proving rasterization is traced JAX array code rather than a graphics-API call; plus the observation-space shape (§8.2)
- [Craftax paper — arXiv:2402.16801](https://arxiv.org/abs/2402.16801) — ICML 2024; "up to 250x faster than the Python-native original" and "1 billion environment interactions finishes in under an hour using only a single GPU"; Craftax-Classic vs Craftax (§8)
- [gymnax](https://github.com/RobertTLange/gymnax) — the functional JAX environment interface Craftax conforms to; 912 stars, active (§8.1)
- [Brax](https://github.com/google/brax) — Google's JAX-native massively-parallel rigidbody physics engine; Apache-2.0, 3,214 stars, last pushed 2026-08-06; "millions of physics steps per second" on TPU (§9.1)
- [Pgx](https://github.com/sotetsuk/pgx) — vectorized JAX board/card-game environments; Apache-2.0, 635 stars, last pushed 2025-03-06; 27 games including Chess, Shogi, Go, Backgammon, poker variants, mahjong, bridge, and MinAtar (§9.1)
- [Jumanji](https://github.com/instadeepai/jumanji) — InstaDeep's JAX suite of combinatorial-optimization and routing environments; Apache-2.0, 855 stars, last pushed 2026-08-04; 22 environments, hybrid Gym-registry/`dm_env`-`TimeStep` API (§9.1)
- [WarpDrive](https://github.com/salesforce/warp-drive) — Salesforce's hand-written CUDA/Numba multi-agent RL framework; BSD-3-Clause, 503 stars, **archived**, last pushed 2025-05-01; 100x+ CPU throughput claim, 10,000+ environments on one A100, JMLR 2022 (§9.2)
- [Madrona](https://madrona-engine.github.io/) — GPU-native ECS batch-simulation game engine with its own batch rasterizer and PyTorch tensor export; MIT, 513 stars, last pushed 2025-11-03; SIGGRAPH 2023 (Shacklett et al., "An Extensible, Data-Oriented Architecture for High-Performance, Many-World Simulation"), 100–300x over CPU baselines (§9.3)
- [`madrona_mjx`](https://github.com/shacklettbp/madrona_mjx) — bridge from Madrona's batch renderer to MuJoCo MJX physics for vision-based RL; 163 stars, deprecated in favor of MJWarp; hundreds of thousands of rendering FPS, integrates with MuJoCo Playground and Brax (§9.3)
- [EnvPool](https://github.com/sail-sg/envpool) — C++ thread-pool-based vectorized execution engine for emulator-bound environments (Atari, VizDoom, DeepMind Control Suite, Google Research Football, and more); Apache-2.0, 1,494 stars, last pushed 2026-07-17; ~1M FPS Atari / ~3M FPS MuJoCo on a DGX-A100 (§9.4)
- [PufferLib — Documentation](https://puffer.ai/docs.html) — PuffeRL throughput figures (20M steps/s CUDA, 5M steps/s PyTorch), the Ocean native environment suite, and the FAQ explaining why generic third-party environment wrapping was dropped (§9.5)
- [PufferLib — `ocean/` (4.0 branch)](https://github.com/PufferAI/PufferLib/tree/4.0/ocean) — the 60+-environment native C Ocean suite, including Craftax, NetHack, and `nmmo3` ports (§9.5)
- [AI Town](https://github.com/a16z-infra/ai-town) — maintained TypeScript reimplementation of the generative-agents simulation; MIT, 10,274 stars, last pushed 2026-06-12; Convex backend, PixiJS rendering, Ollama/OpenAI-compatible LLM backends, Docker port layout, `EMBEDDING_DIMENSION` requirement (§10)
- [AI Town `convex/constants.ts`](https://github.com/a16z-infra/ai-town/blob/main/convex/constants.ts) — `TICK = 16`, `STEP_INTERVAL = 1000`, `AGENT_WAKEUP_THRESHOLD = 1000`, `ACTION_TIMEOUT = 120_000`, conversation and pathfinding budgets, `NUM_MEMORIES_TO_SEARCH` 10x over-fetch comment, `VACUUM_MAX_AGE` (§10.1–§10.5)
- [AI Town — `README.md`](https://github.com/a16z-infra/ai-town/blob/main/README.md) — cloud vs. self-hosted Docker Compose setup, admin-key generation, Ollama/OpenAI-compatible LLM wiring, `EMBEDDING_DIMENSION` and `wipeAllTables` requirements, `data/characters.ts` and Tiled map customization, and the `IDLE_WORLD_TIMEOUT`/`NUM_MEMORIES_TO_SEARCH`/`testing:stop` operational knobs (§10.6)
- [Tiled Map Editor](https://www.mapeditor.org/) — the external map-authoring tool AI Town's `data/convertMap.js` converts from (§10.6)
- [Generative Agents — arXiv:2304.03442](https://arxiv.org/abs/2304.03442) — "Interactive Simulacra of Human Behavior"; the memory/reflection/planning architecture AI Town reimplements (§10, §11)
- [generative_agents](https://github.com/joonspk-research/generative_agents) — reference implementation of the above; 21,899 stars, last commit 2023-08-11 (§11)
- [Voyager](https://github.com/MineDojo/Voyager) — LLM-driven open-ended Minecraft agent using generated code as the action space; 7,121 stars, last commit 2023-07-27 (§11)
- [MineDojo](https://github.com/MineDojo/MineDojo) — Minecraft task suite with internet-scale knowledge base; 2,244 stars, last commit 2023-08-29 (§11)
- [MineRL](https://github.com/minerllabs/minerl) — Minecraft RL environment and human-demonstration dataset; 969 stars, last commit 2025-01-22 on `dev` (§11)
- [Crafter](https://github.com/danijar/crafter) — the original Python survival benchmark Craftax reimplements; 577 stars, last commit 2023-12-13 (§11)
- [MindAgent](https://github.com/mindagent/mindagent) — LLM multi-agent gaming-coordination research artifact; 102 stars, last commit 2024-06-12, no license field (§11)
- [Unity ML-Agents](https://github.com/Unity-Technologies/ml-agents) — commercially-backed precedent for the synchronous engine-bridge architecture; 19,612 stars, active; license not verified in this pass (§11)
- [OpenSpiel](https://github.com/google-deepmind/open_spiel) — framework for research in games spanning perfect/imperfect information and cooperative/zero-sum settings; 5,395 stars, active, CPU-only (§11.1)
- [Melting Pot](https://github.com/google-deepmind/meltingpot) — multi-agent social-behaviour evaluation suite emphasising generalization to unfamiliar co-players; 860 stars, active, CPU-only (§11.1)
