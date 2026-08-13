# Chapter 205f: Artificial Life on the GPU — Cellular Automata, Lenia, and Digital-Organism Simulators

> **Part**: Part XI — Engines and Creative Tools
> **Audience**: Graphics application developers who want a compact, self-contained catalogue of GPGPU parallelization patterns, and systems developers interested in how a compute-only simulation engine binds to a rendering backend. Artificial-life simulators are unusually good teaching material for compute-shader architecture because their rules are trivial to state and their data layouts are forced almost entirely by GPU memory-access constraints rather than by domain complexity.
> **Status**: First draft — 2026-08-10

Artificial life (ALife) is the study of systems that exhibit lifelike behaviour — growth, self-maintenance, reproduction, evolution — from mechanical local rules. From a graphics-programming standpoint the interesting property is that almost every classical ALife model is a *uniform update over a large array of identical elements with a bounded neighbourhood*. That description is also, almost word for word, the workload a GPU compute unit is built to execute. The result is that ALife implementations converge on a small number of GPU architecture patterns, and the choice between them is dictated by the shape of the rule rather than by the biology being modelled.

This chapter treats ALife systems as compute-shader workloads. It identifies three parallelization patterns — grid ping-pong, spectral (FFT) convolution, and agent-based multi-pass buffers — shows each in an upstream implementation whose licence permits quotation, and then examines ALIEN, a production simulator that combines all three ideas with CUDA Graphs and CUDA–OpenGL interoperability, and which has recently grown an AMD ROCm/HIP backend. It closes with the historical digital-organism simulators, which are the interesting negative case: systems whose rules are so control-flow-divergent that the GPU has never been the right machine for them.

Throughout, code is quoted only from repositories whose licensing permits it under this book's CC BY 4.0 terms. Several widely cited ALife repositories ship no licence file at all, or use a NonCommercial licence; those are described in prose with file-and-line citations instead. Section 8.4 sets out that rule explicitly, because for this particular corner of the ecosystem it materially constrains what a derivative work can do.

---

## Table of Contents

- [1. Why ALife Maps Onto Compute Shaders](#1-why-alife-maps-onto-compute-shaders)
  - [1.1 The Shape of a Cellular Rule](#11-the-shape-of-a-cellular-rule)
  - [1.2 Three Patterns](#12-three-patterns)
- [2. Pattern 1: Grid Ping-Pong](#2-pattern-1-grid-ping-pong)
  - [2.1 The Double-Buffered Storage Formulation](#21-the-double-buffered-storage-formulation)
  - [2.2 Toroidal Wrapping by Unsigned Modulo](#22-toroidal-wrapping-by-unsigned-modulo)
  - [2.3 The In-Place Alternative and Why It Races](#23-the-in-place-alternative-and-why-it-races)
  - [2.4 Texture Storage Versus Storage Buffers](#24-texture-storage-versus-storage-buffers)
- [3. The Algorithm That Refuses to Parallelize: HashLife](#3-the-algorithm-that-refuses-to-parallelize-hashlife)
  - [3.1 Canonicalized Quadtrees](#31-canonicalized-quadtrees)
  - [3.2 Memoized Lookahead and Superlinear Speedup](#32-memoized-lookahead-and-superlinear-speedup)
  - [3.3 Why It Does Not Port](#33-why-it-does-not-port)
  - [3.4 Computational Irreducibility: Why the Memo Table Sometimes Never Fills](#34-computational-irreducibility-why-the-memo-table-sometimes-never-fills)
- [4. Pattern 2: Spectral Convolution and Lenia](#4-pattern-2-spectral-convolution-and-lenia)
  - [4.1 From Discrete Neighbour Counts to Continuous Kernels](#41-from-discrete-neighbour-counts-to-continuous-kernels)
  - [4.2 The Convolution Theorem as an Architecture Decision](#42-the-convolution-theorem-as-an-architecture-decision)
  - [4.3 The reikna Backend and Its Round-Trip Cost](#43-the-reikna-backend-and-its-round-trip-cost)
  - [4.4 The JAX/XLA Reformulation](#44-the-jaxxla-reformulation)
- [5. Pattern 3: Agent-Based Multi-Pass Pipelines](#5-pattern-3-agent-based-multi-pass-pipelines)
  - [5.1 The Physarum Model](#51-the-physarum-model)
  - [5.2 A Four-Stage WebGPU Pipeline](#52-a-four-stage-webgpu-pipeline)
  - [5.3 Agent Ping-Pong at Per-Agent Granularity](#53-agent-ping-pong-at-per-agent-granularity)
  - [5.4 Turning Scatter Into Gather](#54-turning-scatter-into-gather)
  - [5.5 Field Decay, Diffusion, and Storage-Texture Precision](#55-field-decay-diffusion-and-storage-texture-precision)
  - [5.6 The CPU Baseline: Particle Life](#56-the-cpu-baseline-particle-life)
- [6. ALIEN: A Production GPU-Resident ALife Engine](#6-alien-a-production-gpu-resident-alife-engine)
  - [6.1 Engine Shape and Requirements](#61-engine-shape-and-requirements)
  - [6.2 The Kernel-Per-Phase Timestep Pipeline](#62-the-kernel-per-phase-timestep-pipeline)
  - [6.3 CUDA Graphs and the Configuration-Keyed Cache](#63-cuda-graphs-and-the-configuration-keyed-cache)
  - [6.4 CUDA–OpenGL Interop as a Pointer Swap](#64-cudaopengl-interop-as-a-pointer-swap)
  - [6.5 The ROCm/HIP Backend as an Aliasing Shim](#65-the-rocmhip-backend-as-an-aliasing-shim)
- [7. Digital Organisms: The Divergent-Control-Flow Case](#7-digital-organisms-the-divergent-control-flow-case)
  - [7.1 Avida](#71-avida)
  - [7.2 Polyworld](#72-polyworld)
  - [7.3 Framsticks](#73-framsticks)
  - [7.4 Why These Stayed on the CPU](#74-why-these-stayed-on-the-cpu)
- [8. Comparison and Portability](#8-comparison-and-portability)
  - [8.1 Pattern Comparison](#81-pattern-comparison)
  - [8.2 Choosing a Pattern](#82-choosing-a-pattern)
  - [8.3 WebGPU as the Portable ALife Substrate](#83-webgpu-as-the-portable-alife-substrate)
  - [8.4 Licensing in the ALife Ecosystem](#84-licensing-in-the-alife-ecosystem)
- [9. AI-Driven Search and Evolution in ALife](#9-ai-driven-search-and-evolution-in-alife)
  - [9.1 Neural Cellular Automata: The Update Rule as a Trained Network](#91-neural-cellular-automata-the-update-rule-as-a-trained-network)
  - [9.2 GPU-Native Evolutionary Computation: evosax, QDax, EvoJAX, and Brax](#92-gpu-native-evolutionary-computation-evosax-qdax-evojax-and-brax)
  - [9.3 JaxLife and the Fixed-Architecture Resolution to §7.4](#93-jaxlife-and-the-fixed-architecture-resolution-to-74)
  - [9.4 LLM-Driven Program Evolution: Digital Red Queen and the Avida Lineage](#94-llm-driven-program-evolution-digital-red-queen-and-the-avida-lineage)
- [Integrations](#integrations)
- [References](#references)
- [Roadmap](#roadmap)

---

## 1. Why ALife Maps Onto Compute Shaders

### 1.1 The Shape of a Cellular Rule

Conway's Game of Life is defined by a rule with four properties that together make it an ideal compute-shader workload:

- **Uniformity.** Every cell runs the same rule. There is no per-element branch on element identity, so a SIMD lane group executes without divergence.
- **Bounded locality.** A cell's next state depends only on its eight immediate neighbours. The working set per output element is nine reads, all within a small spatial window, which is exactly what a texture cache or a coalesced buffer load serves well.
- **Statelessness across steps.** Generation *N+1* depends only on generation *N*. There is no accumulated history to carry, so the only state is the grid itself.
- **Embarrassing parallelism within a step.** Given generation *N*, every cell of generation *N+1* can be computed independently, in any order.

The fourth property is the one that matters architecturally, and it is also the one most easily broken by an implementation mistake — see §2.3. It holds only if the reads of generation *N* and the writes of generation *N+1* target *different* memory.

Continuous and agent-based ALife models weaken these properties in specific, informative ways. Lenia (§4) keeps uniformity and statelessness but replaces the 3×3 neighbourhood with a large radially symmetric kernel, destroying bounded locality and forcing a different algorithm entirely. Physarum-style agent simulations (§5) keep locality but replace the fixed grid with a mobile agent population, so the parallel domain is no longer the output array. Digital-organism systems (§7) break uniformity outright: each organism executes its own program.

### 1.2 Three Patterns

The rest of the chapter is organised around three implementation patterns, each matched to one of those rule shapes:

| Pattern | Parallel domain | Data structure | Typical rule class |
|---|---|---|---|
| Grid ping-pong | One thread per cell | Two grids (buffers or textures), swapped | Local-neighbourhood cellular automata |
| Spectral convolution | One thread per frequency bin, then per cell | Complex-valued frequency-domain arrays | Large-kernel continuous automata |
| Agent multi-pass | One thread per agent, plus one thread per cell | Agent array plus a field texture | Stigmergic / agent-and-trail models |

These are not mutually exclusive. ALIEN (§6) uses a grid for its spatial-hash acceleration structure, agent-style per-object kernels for its cell physics, and neither FFTs nor ping-pong for the parts where in-place updates are provably safe.

---

## 2. Pattern 1: Grid Ping-Pong

### 2.1 The Double-Buffered Storage Formulation

The WebGPU samples repository ships a Game of Life sample whose compute shader is the canonical minimal statement of the pattern. The repository is BSD-3-Clause licensed, so it is quotable directly ([webgpu-samples LICENSE](https://github.com/webgpu/webgpu-samples/blob/main/LICENSE.txt)).

```wgsl
// sample/gameOfLife/compute.wgsl, webgpu/webgpu-samples (BSD-3-Clause)
@binding(0) @group(0) var<storage, read> size: vec2u;
@binding(1) @group(0) var<storage, read> current: array<u32>;
@binding(2) @group(0) var<storage, read_write> next: array<u32>;

override blockSize = 8;

fn getIndex(x: u32, y: u32) -> u32 {
  let h = size.y;
  let w = size.x;

  return (y % h) * w + (x % w);
}

fn getCell(x: u32, y: u32) -> u32 {
  return current[getIndex(x, y)];
}

fn countNeighbors(x: u32, y: u32) -> u32 {
  return getCell(x - 1, y - 1) + getCell(x, y - 1) + getCell(x + 1, y - 1) + 
         getCell(x - 1, y) +                         getCell(x + 1, y) + 
         getCell(x - 1, y + 1) + getCell(x, y + 1) + getCell(x + 1, y + 1);
}

@compute @workgroup_size(blockSize, blockSize)
fn main(@builtin(global_invocation_id) grid: vec3u) {
  let x = grid.x;
  let y = grid.y;
  let n = countNeighbors(x, y);
  next[getIndex(x, y)] = select(u32(n == 3u), u32(n == 2u || n == 3u), getCell(x, y) == 1u); 
}
```

Four architectural decisions are visible in twenty-five lines.

**The address-space split is the correctness mechanism.** `current` is declared `var<storage, read>` and `next` is `var<storage, read_write>`. Every read in the shader goes through `getCell`, which reads only `current`; the single write goes only to `next`. Because the two are distinct buffer bindings, no invocation can observe another invocation's output within the dispatch, and the "any order" property of §1.1 is preserved by construction rather than by scheduling luck. The host swaps the two bindings between dispatches — this is the "ping-pong".

**The grid dimensions are data, not constants.** `size` is itself a storage buffer rather than a uniform or an override, which means the grid can be resized without recompiling the pipeline.

**`override blockSize = 8` is a pipeline-overridable constant.** WGSL `override` declarations are resolved at pipeline-creation time, not at shader-module-creation time, so a single compiled module can be instantiated with several workgroup sizes — useful for tuning against a specific GPU's subgroup width without a shader-source rebuild. This is the WGSL analogue of Vulkan specialization constants, and it is used here directly as the `@workgroup_size` argument.

**The state transition is branchless.** `select(f, t, cond)` returns `t` when `cond` is true. The expression encodes both Life rules at once: a live cell survives on two or three neighbours, a dead cell is born on exactly three. Writing this as a `select` rather than an `if` chain matters less on modern hardware than it once did — compilers flatten short branches — but it is a habit worth keeping for shaders where the two sides of a branch would otherwise both issue memory traffic.

### 2.2 Toroidal Wrapping by Unsigned Modulo

`getIndex` performs `(y % h) * w + (x % w)`, which wraps the grid into a torus. The subtlety is what happens at `x == 0`, where `countNeighbors` evaluates `getCell(x - 1, ...)`. In WGSL, `x` is `u32`, so `0u - 1u` wraps to `0xFFFFFFFF`. That is not undefined behaviour — WGSL specifies unsigned integer arithmetic as wrapping modulo 2³² ([WGSL specification, arithmetic expressions](https://www.w3.org/TR/WGSL/#arithmetic-expr)) — and `0xFFFFFFFF % w` yields the correct left-wrapped column for any `w` that divides 2³² evenly, and a *nearly* correct one otherwise.

This is worth stating precisely because it is a real constraint the sample does not spell out. `0xFFFFFFFF % w == w - 1` holds if and only if `2³² ≡ 0 (mod w)`, that is, when `w` is a power of two. For a non-power-of-two width the wrap lands on `(2³² - 1) mod w`, which is some other column — the simulation still runs, and still looks plausible, but the left and bottom edges no longer join the right and top edges correctly. Implementations that need exact toroidal topology at arbitrary widths should compute `(x + w - 1) % w` instead, which is correct for all `w` at the cost of one add.

### 2.3 The In-Place Alternative and Why It Races

The natural "optimisation" is to drop the second buffer and update in place, halving memory. A widely referenced Unity implementation does exactly this: its `NewGeneration` compute kernel binds a single `RWTexture2D<float4> _Result`, reads the 3×3 neighbourhood from `_Result`, and writes the centre cell back to `_Result` within the same dispatch, with `[numthreads(8, 8, 1)]` and no second surface anywhere in the pipeline. The host side allocates one `RenderTexture` with `enableRandomWrite = true`, dispatches `Mathf.CeilToInt(Screen.width / 8.0f)` groups, and blits the result to the screen. That repository ships no licence file, so the code is described here rather than quoted; the files are `Assets/Game Of Life.compute` and `Assets/Life.cs` in [GarrettGunnell/Compute-Game-Of-Life](https://github.com/GarrettGunnell/Compute-Game-Of-Life).

The in-place formulation contains an unsynchronized read-write hazard, and it is worth being precise about why, because the reasoning generalises to every stencil kernel.

Within a single workgroup, invocations can be ordered with `workgroupBarrier()` (WGSL) or `GroupMemoryBarrierWithGroupSync()` (HLSL). *Across* workgroups there is no such facility: neither Vulkan, D3D12, Metal, nor WebGPU provides a device-wide barrier inside a dispatch, because workgroups are not guaranteed to be co-resident — a GPU is free to run workgroup 0 to completion before workgroup 4000 is even scheduled. The only device-wide ordering point is the dispatch boundary itself.

So consider a cell at the right edge of workgroup *A*, whose right-hand neighbours belong to workgroup *B*. If *B* has already executed, those neighbours hold generation *N+1* values. If *B* has not, they hold generation *N*. If *B* is executing concurrently, the read may observe either, and nothing in any of the four APIs constrains which. The kernel therefore computes each cell from a mixture of two generations, with the mixture determined by the hardware scheduler. The output is not a Conway step; it is a plausible-looking cellular automaton with no fixed rule, and it is not reproducible across GPUs or even across runs.

The lesson is not that the implementation is defective — it produces attractive output, which for a visual demo is the goal. The lesson is that **ping-pong double buffering is not a memory-management convenience; it is the synchronization mechanism**. The second buffer exists precisely because the dispatch boundary is the only global barrier available, and a stencil kernel needs a global barrier between the read of generation *N* and the write of generation *N+1*. Paying one extra grid of memory buys that barrier for free.

There is one legitimate in-place case: when the update is *element-local*, reading only the cell it writes. Decay, thresholding, and colour-space conversion passes can safely run in place. The moment a neighbour is read, the second buffer becomes mandatory.

### 2.4 Texture Storage Versus Storage Buffers

The two examples above differ in a second respect: the WebGPU sample stores the grid in a `array<u32>` storage buffer, the Unity one in an `RWTexture2D<float4>`. The trade-offs are worth naming, because they recur in every pattern in this chapter.

A storage buffer gives exact control over element size. Life needs one bit per cell; a `u32` array wastes 31 of them, but a bit-packed layout is straightforward and yields a 32× memory reduction, at the cost of shifting and masking on every neighbour read. A texture's format is chosen from a fixed enumeration, so `rgba32float` for a one-bit state wastes 127 bits per cell.

A texture, in exchange, gives hardware address handling. Out-of-bounds coordinates are resolved by the sampler's address mode — `repeat` implements toroidal wrapping with no arithmetic at all, replacing §2.2's modulo and its power-of-two caveat. Textures also route reads through the texture cache, which is optimised for 2D spatial locality, whereas a linear buffer read at `(y-1)*w + x` is a full row-stride away from the read at `y*w + x` and may miss.

For a small stencil on a large grid the texture path usually wins on bandwidth; for bit-packed state or non-power-of-two element sizes the buffer path wins on footprint. Section 5 shows an implementation that uses both simultaneously, for exactly these reasons.

---

## 3. The Algorithm That Refuses to Parallelize: HashLife

The GPU is not the fastest way to run Conway's Game of Life. For structured patterns it is not remotely competitive, and understanding why is the sharpest available illustration of when to *not* reach for a compute shader.

Golly is the reference Life explorer. Its canonical home is SourceForge; note that the frequently guessed GitHub path `GollyGang/golly` does not exist. An actively synced unofficial mirror is [AlephAlpha/golly](https://github.com/AlephAlpha/golly), which tracks the SourceForge repository and carries no GitHub licence metadata of its own. Golly itself is distributed under the **GNU GPL version 2 or (at your option) any later version**, per [`docs/License.html`](https://github.com/AlephAlpha/golly/blob/master/docs/License.html) in the source tree. The HashLife implementation lives in `gollybase/hlifealgo.h` and `gollybase/hlifealgo.cpp`, and the header's opening comment block is an unusually clear description of the algorithm. It is paraphrased here rather than quoted, given the licence.

### 3.1 Canonicalized Quadtrees

The first mechanism is a symbolic representation of 2D space, explicitly analogised in the source to a binary decision diagram. Space is divided into 8×8 squares, and those squares are *canonicalized*: all empty 8×8 squares are represented by a single shared instance, all squares with only the upper-left cell set by another single instance, and so on. A grid position then stores a pointer to a shared instance rather than the 64 bits of cell data, which is already a compression for any repetitive pattern.

Four 8-squares are grouped into a 16-square, represented by four pointers and itself canonicalized; four 16-squares into a 32-square; and upward without limit. The header notes that twenty levels of nodes describe a universe roughly 4 × 2²⁰ cells on a side, and a hundred levels roughly 4 × 2¹⁰⁰ — and that patterns expanding beyond 10⁵⁰ cells on a side have been run. Crucially, the representation stores no coordinates anywhere, so there is no coordinate type to overflow and no multi-precision address arithmetic in the inner loop.

### 3.2 Memoized Lookahead and Superlinear Speedup

The second mechanism is the one that produces the algorithm's characteristic behaviour. Each canonical node caches the *result* of running Life on the region it represents — and the lookahead distance scales with the node's level, exactly as the node's spatial extent does. A node at level *k* caches a result 2^(k-2) generations into the future, not one generation.

The consequence is that advancing a pattern by 2^k generations can cost a single cache lookup, if the relevant node has been seen before. For patterns with spatial or temporal regularity — which is most interesting Life patterns, since gliders, guns, and replicators are all periodic — the effective cost per generation *falls* as the simulation runs, because the memo table warms up. Golly can advance such patterns by 2⁶⁴ generations effectively instantly.

### 3.3 Why It Does Not Port

HashLife has no property a GPU rewards and every property a GPU punishes:

- **Pointer chasing.** Traversal follows four pointers per node down an irregular tree. The memory access pattern is data-dependent and uncoalesced, which is the worst case for a wide SIMD load.
- **A global mutable hash table.** Canonicalization requires a hash lookup on every node construction, against a table shared by all of the computation. Making that concurrent on a GPU means either heavy atomics or partitioning that destroys the sharing the algorithm depends on.
- **No fixed parallel domain.** The number of nodes to process is discovered during traversal, so there is nothing to size a dispatch against.
- **Divergent recursion.** Sibling subtrees terminate at wildly different depths, so neighbouring lanes in a warp do wildly different amounts of work.

The result is a clean split by workload. For a dense pseudorandom soup with no repeating structure, the memo table never hits, HashLife degenerates, and the GPU stencil kernel of §2.1 wins by a wide margin. For structured patterns the asymptotics are not close: a single CPU core doing memoized quadtree lookups beats any number of GPU cores doing per-cell arithmetic, because it is not doing per-cell arithmetic at all. Golly ships both, exposing HashLife alongside a conventional per-cell algorithm in `gollybase/qlifealgo.cpp`, and the choice belongs to the user.

The general form of this lesson: a compute shader parallelizes the *work*, but an algorithmic change can delete the work. Check for the second before investing in the first.

### 3.4 Computational Irreducibility: Why the Memo Table Sometimes Never Fills

§3.3 described HashLife's failure mode empirically: a dense pseudorandom soup defeats the memo table because no node repeats. Wolfram's theoretical framework for cellular automata gives a name and a general argument for why this happens, independent of HashLife's specific data structure.

Wolfram's *Principle of Computational Equivalence* holds that almost any process whose behaviour is not obviously simple is computationally equivalent in sophistication to any other, including the step-by-step evolution of the process itself. A direct consequence is *computational irreducibility*: for such a process there is, in general, no procedure that predicts its state after *n* steps in less work than actually carrying out those *n* steps ([*A New Kind of Science*, Ch. 12 §6, "Computational Irreducibility"](https://www.wolframscience.com/nks/p737--computational-irreducibility/)). Historically, mathematical physics succeeded by finding closed-form shortcuts — a formula for a planet's position that skips over tracing every intermediate instant — and irreducibility is Wolfram's account of why that strategy runs out for most cellular automata: no such shortcut exists to find, not merely one that has not yet been found. A more restricted, formally stronger version of the same obstruction appears in Wolfram's earlier work on undecidability, which shows that some questions about cellular automaton evolution are not merely hard but formally undecidable ([Wolfram, "Undecidability and Intractability in Theoretical Physics," *Physical Review Letters* 54, 735 (1985)](https://doi.org/10.1103/PhysRevLett.54.735)).

HashLife's memoized quadtree is exactly a search for closed-form shortcuts, restricted to shortcuts of one specific shape: "this spatial pattern's future is identical to a pattern already computed." That restriction is what makes the algorithm's measured behaviour track the irreducibility argument so cleanly. A glider, a still life, a small oscillator — anything periodic or self-similar in space or time — is by construction the kind of pattern for which such a shortcut exists, and HashLife finds it, which is why Golly can advance a stable glider gun by 2⁶⁴ generations from a single cache hit. A pseudorandom soup has no repeating substructure at any quadtree level, so it is, for the purposes of this particular algorithm, the irreducible case: nothing in the memo table matches, every node is constructed exactly once, and the amortized cost per generation degenerates to roughly what §2.1's per-cell GPU kernel already pays directly, without the pointer-chasing and hash-table overhead on top. HashLife is therefore not a general Life accelerator; it is a shortcut-finder whose hit rate is an empirical measurement of how reducible a given pattern happens to be, and the GPU-versus-HashLife split of §3.3 is that measurement made concrete at two ends of the reducibility spectrum.

---

## 4. Pattern 2: Spectral Convolution and Lenia

### 4.1 From Discrete Neighbour Counts to Continuous Kernels

Lenia generalises Life along three axes simultaneously: state becomes continuous (a cell holds a real value in [0,1] rather than a bit), time becomes continuous (a small `dt` increment replaces a discrete step), and space becomes continuous in effect, because the 3×3 neighbour count is replaced by a convolution against a large, smooth, radially symmetric kernel ([Chakazul/Lenia](https://github.com/Chakazul/Lenia), MIT).

An update step becomes:

1. **Potential.** Convolve the world *A* with kernel *K* to obtain a potential field *U = K \* A*.
2. **Growth.** Map the potential through a growth function *G* — typically a Gaussian centred on μ with width σ — to obtain a field of signed growth rates.
3. **Integrate.** `A ← clip(A + dt · G(U), 0, 1)`.

Step 1 is the entire architectural problem. Where Life reads 8 neighbours, a Lenia kernel of radius *R* reads on the order of πR² cells per output cell. At the radii that produce the model's characteristic self-organising structures — commonly 13 to 20 or more — that is several hundred to well over a thousand reads per cell. A direct spatial convolution is *O(N·R²)* for an *N*-cell grid, and becomes the dominant cost long before the grid gets large.

### 4.2 The Convolution Theorem as an Architecture Decision

The convolution theorem states that convolution in the spatial domain is pointwise multiplication in the frequency domain. Applied here:

*U = K \* A = F⁻¹( F(K) · F(A) )*

This replaces an *O(N·R²)* operation with two transforms and a pointwise multiply — *O(N log N)*, with no dependence on *R* at all. Above a modest kernel radius the FFT path wins, and it keeps winning as *R* grows, which is precisely the regime Lenia lives in.

The reference Python implementation exploits a further property. The kernel does not change between steps, so its transform can be computed once at initialisation and reused. In `Python/LeniaNDK.py` the kernel bank is pre-transformed in a single comprehension:

```python
# Python/LeniaNDK.py, Chakazul/Lenia (MIT)
self.kernel_FFT = [self.fftn(kernel_norm[k]) for k in KERNEL]
```

so that the per-step work reduces to one forward transform of the world, one complex multiply per kernel, and one inverse transform:

```python
# Python/LeniaNDK.py, Chakazul/Lenia (MIT) — calc_once
self.world_FFT = self.fftn(A)
...
self.potential_FFT = self.kernel_FFT[k] * self.world_FFT
self.potential = self.fftshift(np.real(self.ifftn(self.potential_FFT)))
self.field = gfunc(self.potential, m, s)
...
D += dt * self.field
```

The `fftshift` after the inverse transform recentres the result, because an FFT-based convolution computes a *circular* convolution with the kernel's origin at index zero. This is not a workaround — it is the same toroidal topology that §2.2 implemented with a modulo, arriving for free as a property of the transform. For Lenia, whose worlds are conventionally toroidal, that is exactly the wanted behaviour.

### 4.3 The reikna Backend and Its Round-Trip Cost

The GPU acceleration is supplied by `reikna`, a GPGPU algorithm library layered over PyOpenCL or PyCUDA, pinned at `reikna==0.7.5` in `Python/requirements.txt` ([reikna documentation](https://reikna.publicfields.net/en/latest/)). Compilation selects whichever backend is present via `reikna.cluda.any_api()`, creates a thread, and compiles three computations: a 1-D FFT, an N-D FFT, and an FFT shift. If anything in that sequence raises, the implementation sets `has_gpu = False` and falls back to NumPy silently.

The dispatch wrapper is where the architectural cost sits:

```python
# Python/LeniaNDK.py, Chakazul/Lenia (MIT) — run_gpu
def run_gpu(self, A, cpu_func, gpu_func, dtype, **kwargs):
    if self.is_gpu and self.gpu_thr and gpu_func:
        op_dev = self.gpu_thr.to_device(A.astype(dtype))
        gpu_func(op_dev, op_dev, **kwargs)
        return op_dev.get()
    else:
        return cpu_func(A)
```

The kernel itself is in-place — `gpu_func(op_dev, op_dev)` reads and writes the same device array — but the wrapper uploads with `to_device` and downloads with `.get()` on *every invocation*. Since `fftn`, `ifftn`, and `fftshift` all route through `run_gpu`, a single simulation step performs a host-to-device transfer and a device-to-host transfer for each of them. The intermediate `potential_FFT` multiply happens in NumPy on the host.

This is the classic accelerated-library integration failure, and it is instructive precisely because the library is used correctly and the result is still bandwidth-bound. Over PCIe, a full round trip per transform on a grid large enough to matter costs more than the transform saves. The fix is not a faster FFT; it is keeping the data resident on the device across the whole step — which is what §4.4 and §6 do. Note also that the 1-D `fft1` path in this implementation is CPU-only, so any code path using it does not touch the GPU at all.

### 4.4 The JAX/XLA Reformulation

The later Lenia derivatives reformulate the whole simulation, not just the transform, in JAX. This eliminates the round trip structurally: JAX arrays live on the accelerator, `jax.jit` fuses the transform, the multiply, the growth function, and the integration into a single compiled XLA computation, and data crosses the PCIe boundary only when the host explicitly asks for a result.

Flow Lenia extends the model with mass conservation via reintegration tracking. Its `flowlenia/flowlenia.py` uses `jnp.fft` directly — `fA = jnp.fft.fft2(A, axes=(0,1))` at line 82 and `U = jnp.real(jnp.fft.ifft2(state.fK * fAk, axes=(0,1)))` at line 86 — with the kernel transform `fK` computed once in `initialize` and carried in the state, the same pre-transform optimisation as §4.2. The module is built on `equinox` and annotated with `jaxtyping`, and exposes `rollout` / `rollout_` helpers that wrap a `_step` closure, so an entire trajectory is one JIT-compiled call rather than a Python loop issuing one dispatch per step ([erwanplantec/FlowLenia](https://github.com/erwanplantec/FlowLenia); [Flow Lenia paper, arXiv:2212.07906](https://arxiv.org/abs/2212.07906)).

Leniax takes the same approach for a different purpose — it describes itself as a search and rendering engine for Lenia, built for quality-diversity search over parameter space rather than for interactive display. Its `leniax/core.py` implements `get_potential_fft`, computing `fft_cells = jnp.fft.fftn(state, axes=world_dims)` at line 81 and `conv_out = jnp.real(jnp.fft.ifftn(fft_out, axes=world_dims))` at line 87, generalised over arbitrary world dimensionality and a channel axis ([morgangiraud/leniax](https://github.com/morgangiraud/leniax)).

**Licensing note:** neither FlowLenia nor leniax ships a `LICENSE` file, and neither declares a licence in packaging metadata; GitHub reports no licence for either repository. Their code is therefore described above with file-and-line citations but not quoted, and readers intending to reuse either should seek clarification from upstream rather than assume permissive terms. This is stated explicitly rather than guessed at.

The deeper payoff of the JAX formulation is differentiability. Because every operation in the step — including the FFTs — has a defined derivative, the whole rollout can be differentiated end to end, making kernel and growth parameters directly optimisable by gradient descent rather than only by evolutionary search. That is a capability the reikna formulation cannot offer at any level of optimisation, and it is the strongest argument for treating an ALife simulator as a differentiable program rather than as a hand-written shader.

---

## 5. Pattern 3: Agent-Based Multi-Pass Pipelines

### 5.1 The Physarum Model

The third pattern drops the fixed grid as the primary parallel domain. In a Physarum simulation — modelled on the slime mold *Physarum polycephalum* — the state is a population of mobile agents, each with a position and a heading, plus a scalar "chemoattractant" field over the plane. Each step, every agent samples the field ahead-left, ahead, and ahead-right, rotates toward the strongest signal, steps forward, and deposits into the field; the field then diffuses and decays. The agents do not interact directly at all: all coupling is mediated by the field. This is *stigmergy*, and it is the mechanism that produces the model's characteristic transport networks.

The canonical algorithmic description is Jones (2010), *Characteristics of pattern formation and evolution in approximations of Physarum transport networks*, Artificial Life 16(2) ([UWE repository](https://uwe-repository.worktribe.com/output/980579/characteristics-of-pattern-formation-and-evolution-in-approximations-of-physarum-transport-networks)); implementations commonly also cite [sagejenson.com/physarum](https://sagejenson.com/physarum) for the parameter space.

### 5.2 A Four-Stage WebGPU Pipeline

A compact MIT-licensed Rust + wgpu implementation, [tom-strowger/physarum-rust](https://github.com/tom-strowger/physarum-rust), lays the pipeline out explicitly in its README, and the structure is worth reproducing because it generalises to essentially every agent-and-field model:

| Stage | Type | Inputs | Output |
|---|---|---|---|
| 1. Draw positions | Render | Agents[A] | New-dots texture |
| 2. Deposit | Compute | New dots, Chemo[0] | Chemo[1] |
| 3. Diffuse | Compute | Chemo[1] | Chemo[0] |
| 4. Update agents | Compute | Agents[A], Chemo[0] | Agents[B] |

with A and B swapped each frame. Two independent ping-pongs are running concurrently: one over the agent storage buffers, one over the chemo textures. Note that stages 2 and 3 ping-pong the chemo texture *within* a single frame — `Chemo[0] → Chemo[1] → Chemo[0]` — which returns the field to its original binding by the end of the frame, so only the agent buffers need a host-side swap.

Each of the four stages is a separate dispatch, and that is not an implementation detail. As established in §2.3, the dispatch boundary is the only device-wide ordering point available, so every place where one stage must observe the complete output of the previous one *has* to be a separate dispatch. The pipeline's stage count is dictated by its data dependencies.

### 5.3 Agent Ping-Pong at Per-Agent Granularity

The agent update shader shows the pattern at agent rather than cell granularity:

```wgsl
// src/compute.wgsl, tom-strowger/physarum-rust (MIT)
@group(0) @binding(0) var<uniform> params : SimParams;
@group(0) @binding(1) var<storage, read> agents_in : array<Particle>;
@group(0) @binding(2) var chemo_texture : texture_2d<f32>;
@group(0) @binding(3) var<storage, read_write> agents_out : array<Particle>;
@group(0) @binding(4) var texture_sampler: sampler;
@group(0) @binding(5) var control_texture : texture_2d<f32>;

@compute
@workgroup_size(128)
fn main(@builtin(global_invocation_id) global_invocation_id: vec3<u32>) {
  let total = arrayLength(&agents_in);
  let index = global_invocation_id.x;
  if (index >= total) {
    return;
  }
  ...
}
```

Three things differ from §2.1 despite the identical read/read_write split:

**The dispatch is one-dimensional and sized by agent count**, with `@workgroup_size(128)` rather than a 2-D 8×8 tile. There is no spatial locality among agent indices to exploit — agent *i* and agent *i+1* may be anywhere on the plane — so a 1-D layout maximising coalesced access to the `Particle` array is the right shape.

**The bounds guard uses `arrayLength(&agents_in)`.** Because the dispatch size must be rounded up to a whole number of 128-thread workgroups, the final workgroup contains invocations with no agent. WGSL's `arrayLength` reads the runtime-sized binding's actual length, so the guard adapts to a resized population without a pipeline rebuild — the same "dimensions are data" principle as §2.1's `size` buffer.

**The agent struct carries explicit padding.** The declaration is `pos : vec2<f32>, heading : f32, padding : f32` — 16 bytes. WGSL's storage address space applies alignment rules under which a `vec2<f32>` requires 8-byte alignment; making the struct a clean 16 bytes keeps every element aligned to a 16-byte boundary and makes the array stride a power of two, which matters for coalescing. Padding an agent struct to 16 or 32 bytes is close to universal practice in this pattern.

The agent's sensing logic reads the chemo field through `textureSampleLevel` at three offset positions, then rotates toward the strongest. Where all three sensors disagree symmetrically, the shader picks a direction using a hash-based pseudo-random function of position — a stateless PRNG, since carrying per-agent RNG state across the ping-pong would cost another buffer.

### 5.4 Turning Scatter Into Gather

The most interesting decision in the pipeline is stage 1. Agents deposit into the field, which is naturally a *scatter*: each agent writes to whatever texel it happens to occupy, and several agents may occupy the same texel. The obvious implementation is an atomic add into a storage texture — but atomics on storage textures are not available in the base WebGPU feature set, and even where they are available, contention when thousands of agents converge on the same texel (which is exactly what this simulation does, since agents form dense trails) makes them slow.

The implementation avoids the problem entirely by making stage 1 a *render* pass that draws a dot per agent into a separate `new_dots` texture, letting the fixed-function blending hardware resolve overlaps. Stage 2 then reads that texture per-texel:

```wgsl
// src/deposit.wgsl, tom-strowger/physarum-rust (MIT)
@group(0) @binding(0) var<uniform> params : SimParams;
@group(0) @binding(1) var chemo_in : texture_2d<f32>;
@group(0) @binding(2) var new_dots : texture_2d<f32>;
@group(0) @binding(3) var chemo_out: texture_storage_2d<rgba16float, write>;

@compute
@workgroup_size(8,8)
fn main(@builtin(global_invocation_id) global_invocation_id: vec3<u32>) {
  let index = vec2<u32>(global_invocation_id.x, global_invocation_id.y);
  let deposit_amount = params.deposit_chemo / params.max_chemo;

  var chemo = textureLoad(chemo_in, index, 0).rgb;
  chemo += textureLoad(new_dots, index, 0).rgb * vec3<f32>( deposit_amount, deposit_amount, deposit_amount );
  chemo = min( chemo, vec3<f32>( 65504.0, 65504.0, 65504.0 ) );

  textureStore(chemo_out, index, vec4<f32>(chemo.r, chemo.g, chemo.b, 1.0));
}
```

Stage 2 is a pure gather: one thread per texel, two loads, one store, no contention, and a 2-D 8×8 workgroup that matches the texture's spatial locality. **The scatter has been converted into a gather by routing it through the rasterizer** — a general technique worth remembering whenever a compute pass wants to accumulate from a mobile population into a fixed grid.

Note also that this stage runs at *cell* granularity while stage 4 runs at *agent* granularity, within the same simulation. Mixing dispatch shapes across passes of one pipeline is normal and correct; each pass should be sized by its own output domain.

### 5.5 Field Decay, Diffusion, and Storage-Texture Precision

Stage 3 combines diffusion and decay. Diffusion is a 9-tap Gaussian blur applied twice — once horizontally, once vertically — with the two results averaged, followed by a multiplicative decay of `1.0 - params.decay_chemo`. Because diffusion reads a neighbourhood, this stage cannot run in place, which is why the pipeline needs the second chemo texture even though stages 2 and 3 could otherwise have been fused.

The chemo textures are declared `texture_storage_2d<rgba16float, write>` on the write side and `texture_2d<f32>` on the read side. That asymmetry is a WebGPU rule rather than a stylistic choice: a storage texture binding is declared with an explicit access mode, and `read_write` storage textures are not in the base feature set, so a texture that is read in one pass and written in another is bound as a sampled texture in the first and a write-only storage texture in the second. This is the texture-domain equivalent of §2.1's `read` / `read_write` buffer split, and it enforces the ping-pong just as firmly.

The `min(chemo, 65504.0)` clamp in the deposit shader is a direct consequence of the `rgba16float` format: 65504 is the largest finite IEEE-754 half-precision value. Without the clamp, a sufficiently dense agent cluster would push a texel to `+Inf`, which then propagates through the blur in stage 3 and destroys a growing region of the field. Half-precision halves the field's bandwidth relative to `rgba32float`, which for a pass that reads ten texels per output is a substantial saving — but it moves the overflow boundary into the range the simulation actually reaches, so the clamp is required rather than defensive.

### 5.6 The CPU Baseline: Particle Life

For contrast, [hunar4321/particle-life](https://github.com/hunar4321/particle-life) (MIT) implements a related class of model — coloured particle groups with asymmetric attraction/repulsion rules — entirely on the CPU, in openFrameworks. Its core is a nested loop over pairs in `particle_life/src/ofApp.cpp`:

```cpp
// particle_life/src/ofApp.cpp, hunar4321/particle-life (MIT) — ofApp::interaction
	//#pragma omp parallel
	for (size_t i = 0; i < Group1.pos.size(); i++)
	{
		float fx = 0.0F;	// force on x
		float fy = 0.0F;	// force on y

		for (size_t j = 0; j < Group2.pos.size(); j++)
		{
			if (Group1.pos[i] != Group2.pos[j])
			{
				const float distance_squared = ((Group1.pos[i].x - Group2.pos[j].x) * (Group1.pos[i].x - Group2.pos[j].x))
					+ ((Group1.pos[i].y - Group2.pos[j].y) * (Group1.pos[i].y - Group2.pos[j].y));

				const float force = distance_squared < radius* radius ? 1.0F / std::sqrtf(distance_squared) : 0.0F;
				fx += ((Group1.pos[i].x - Group2.pos[j].x) * force);
				fy += ((Group1.pos[i].y - Group2.pos[j].y) * force);
			}
```

The README notes that "the core algorithm is the first 100 lines of code" in that file. It is *O(n²)* per step, and notably the `#pragma omp parallel` directive is commented out, as is an `#include "oneapi/tbb.h"` — so the shipped implementation is not merely un-GPU-accelerated, it is single-threaded.

This makes it an unusually clean porting exercise, and the mapping to §5.3 is direct. One thread per particle, `@workgroup_size(128)`, a read/read_write particle buffer pair; the inner loop becomes a loop inside the shader over the read-side buffer. The interesting optimisation is the next step: the *O(n²)* all-pairs loop is only necessary because the force has unbounded range. Since the force falls off with distance, binning particles into a spatial grid — a uniform grid hash, built in its own dispatch — reduces the inner loop to the neighbouring bins, at which point the algorithm becomes *O(n)* with a constant factor set by the bin occupancy. That is the same acceleration structure ALIEN uses (§6.2), and it is the standard answer for any particle system whose interactions are range-limited.

---

## 6. ALIEN: A Production GPU-Resident ALife Engine

### 6.1 Engine Shape and Requirements

ALIEN (Artificial Life Environment) is the most architecturally complete open-source ALife simulator in current development, and the useful thing about it for this chapter is that it is not a demo. It won the ALIFE 2024 Virtual Creatures Competition, and its README states plainly that "the simulation code is written entirely in CUDA" and that "rendering and post-processing [is done] via OpenGL using CUDA-OpenGL interoperability" ([chrxh/alien](https://github.com/chrxh/alien), BSD-3-Clause).

The model is a particle engine in which each particle ("cell") carries energy, can bond to neighbours with spring-like connections, and executes one of a fixed set of cell functions — constructor, attacker, sensor, muscle, memory, and others. Multicellular organisms are graphs of bonded cells, and their construction is encoded in a genome that a constructor cell interprets. Evolution is not scripted; it emerges from replication with mutation under energy competition.

The build requirements set the hardware floor: NVIDIA compute capability 6.0 or higher and CUDA Toolkit 11.2 or newer, built with CMake, vcpkg (vendored as a submodule), Ninja, and `nvcc`. The default architecture list in `CMakeLists.txt` is `set(CMAKE_CUDA_ARCHITECTURES 75 120)` — Turing through the newest supported generation.

### 6.2 The Kernel-Per-Phase Timestep Pipeline

`source/EngineKernels/SimulationKernels.cu` declares roughly 46 `__global__` kernels named `cudaNextTimestep_*`. A single simulation timestep launches them in a fixed sequence, and the sequence is the physical model written out as a dependency chain:

- `prepare`, then `physics_init` and `physics_fillMaps` — the latter building the spatial hash that makes neighbour queries *O(1)*, the same acceleration structure §5.6 argued for.
- Fluid-dynamics force calculation, then `physics_applyForces`.
- Verlet integration split into position and velocity phases, with connection (spring) forces applied between them.
- Signal calculation and propagation along cell connections, then `energyFlow`.
- Cell-state substeps, then one kernel per cell function: `cellType_constructor`, `cellType_attacker`, `cellType_sensor`, `cellType_muscle`, `cellType_injector`, `cellType_detonator`, `cellType_digestor`, `cellType_memory`, `cellType_communicator`, and others.
- Friction, five `structuralOperations_substep` kernels handling bond creation and destruction, and `incTimestep`.

The design decision worth extracting is **why this is dozens of small kernels rather than one large one**. Two reasons, both already established in this chapter. First, §2.3: each phase must observe the complete output of the previous phase across the entire simulation, and the kernel boundary is the only device-wide barrier. Second, divergence: dispatching one kernel per cell function means that within `cellType_attacker` every active lane is running attacker logic. A single monolithic kernel with a `switch` on cell type would serialise all fourteen branches within every warp containing a mix of types. **Splitting by phase buys global ordering; splitting by cell type buys warp coherence.**

Separate kernels for the cluster-detection pass (`cudaInitClusterData`, `cudaFindClusterIteration`, `cudaFindClusterBoundaries`, `cudaAccumulateClusterPosAndVel`, `cudaAccumulateClusterAngularProp`, `cudaApplyClusterData`) implement an iterative connected-components search over the bond graph — an inherently multi-pass algorithm for the same barrier reason.

### 6.3 CUDA Graphs and the Configuration-Keyed Cache

Dozens of kernel launches per timestep, at interactive frame rates, is enough launch overhead to matter: each launch costs microseconds of CPU-side driver work, and at 46 launches per step and hundreds of steps per second the CPU becomes the bottleneck before the GPU does.

`source/EngineImpl/SimulationKernelsService.cu` solves this with CUDA Graphs. The launch sequence is captured once and replayed as a single object:

```cpp
// source/EngineImpl/SimulationKernelsService.cu, chrxh/alien (BSD-3-Clause)
cudaGraph_t graph;

CHECK_FOR_DEVICE_ERRORS(cudaStreamBeginCapture(_stream, cudaStreamCaptureModeGlobal));

launchTimestepKernels(config, settings, data, statistics);

CHECK_FOR_DEVICE_ERRORS(cudaStreamEndCapture(_stream, &graph));

cudaGraphExec_t graphExec;
CHECK_FOR_DEVICE_ERRORS(cudaGraphInstantiate(&graphExec, graph, nullptr, nullptr, 0));
CHECK_FOR_DEVICE_ERRORS(cudaGraphDestroy(graph));

_graphCache[config] = graphExec;
return graphExec;
```

`cudaGraphDestroy` immediately after instantiation is correct and worth noting: the `cudaGraph_t` is only the *template*: once `cudaGraphInstantiate` has produced an executable `cudaGraphExec_t`, the template is no longer needed and only the executable is cached.

and subsequent timesteps with the same configuration replay it with a single `cudaGraphLaunch(graphExec, _stream)`. Capture mode records the launches into a graph instead of executing them; instantiation resolves the graph into an executable form with dependencies already computed, so the driver work for the whole timestep is paid once. Chapter 66 covers the stream and graph API in general terms; this is a representative production use of it.

The subtlety is that a graph is *static* — the captured launch sequence cannot be changed without re-instantiating. But ALIEN's timestep is not identical every step: some phases run only on certain steps, and block sizes depend on simulation parameters. The engine handles this with a cache keyed on a `CudaGraphConfig` struct built by `buildGraphConfig()`, whose fields are exactly the things that alter the launch sequence:

- `timestepMod3` — `timestep % 3`, for phases on a three-step cycle.
- `executeCellFunction` — whether `timestep % TIMESTEPS_PER_CELL_FUNCTION == 0`.
- `hasLayers`, `rigidityEnabled` — feature toggles that add or remove kernels.
- `fluidKernelThreads`, `fluidBoundaryKernelThreads`, `numBlocks` — launch geometry.

`_graphCache` maps this key to a `cudaGraphExec_t`. A step looks up its config; on a hit it launches the cached graph, on a miss it captures a new one and caches it. Because the config space is small and quickly exhausted, steady-state execution is essentially all cache hits. **This is the general pattern for applying CUDA Graphs to a non-uniform pipeline: enumerate the discrete variation axes, key a cache on them, and accept a one-time capture per combination.**

The block size for the fluid kernel is derived from the physics rather than tuned by hand — `calcOptimalThreadsForFluidKernel` computes the SPH scan rectangle from the smoothing length and squares it, so the thread count matches the number of grid cells a particle's kernel support actually covers:

```cpp
// source/EngineImpl/SimulationKernelsService.cu, chrxh/alien (BSD-3-Clause)
int SimulationKernelsService::calcOptimalThreadsForFluidKernel(SimulationParameters const& parameters) const
{
    auto scanRectLength = ceilf(parameters.smoothingLength.value * 2) * 2 + 1;
    return static_cast<int>(scanRectLength * scanRectLength);
}
```

A companion function for the fluid *boundary* kernel uses `* 4` rather than `* 2`, because fluid particles use twice the base smoothing length and the kernel support extends to twice that again. Both are pure functions of a simulation parameter, which is exactly why `fluidKernelThreads` and `fluidBoundaryKernelThreads` have to appear in the graph cache key — changing the smoothing length changes the launch geometry and therefore invalidates the captured graph.

### 6.4 CUDA–OpenGL Interop as a Pointer Swap

ALIEN's rendering path is the cleanest available illustration of graphics-compute interop. Registration happens once, in `source/EngineKernels/CudaGeometryBuffers.cu`:

```cpp
// source/EngineKernels/CudaGeometryBuffers.cu, chrxh/alien (BSD-3-Clause)
cudaGraphicsResource* registerBufferResource(GLuint buffer)
{
    cudaGraphicsResource* result = nullptr;
    CHECK_FOR_DEVICE_ERRORS(cudaGraphicsGLRegisterBuffer(&result, buffer, cudaGraphicsMapFlagsWriteDiscard));
    return result;
}
```

Note `cudaGraphicsGLRegisterBuffer`, not `...RegisterImage`: ALIEN registers OpenGL *buffer objects* — VBOs for objects, fluid particles, locations, selected objects, selected connections, attack events and detonation events, plus EBOs for line and triangle indices, nine in total. The simulation produces geometry, not a framebuffer image; shading happens afterwards in a GLSL post-processing chain of roughly 66 shader stages under `source/Shaders/` (metaballs, blur, subsurface scattering, tone mapping, and so on).

`cudaGraphicsMapFlagsWriteDiscard` is the correct hint here and is worth noting: it tells the driver that CUDA will overwrite the buffer entirely and that its previous contents need not be preserved, allowing the driver to skip any migration of the existing data.

Per frame, `source/EngineImpl/GeometryKernelsService.cu` maps, extracts, and unmaps:

```cpp
// source/EngineImpl/GeometryKernelsService.cu, chrxh/alien (BSD-3-Clause)
CHECK_FOR_DEVICE_ERRORS(cudaGraphicsMapResources(1, &renderingData.vertexBuffer));
ObjectVertexData* mappedCellBuffer;
size_t bufferSize;
CHECK_FOR_DEVICE_ERRORS(cudaGraphicsResourceGetMappedPointer(reinterpret_cast<void**>(&mappedCellBuffer), &bufferSize, renderingData.vertexBuffer));
setValueToDevice(_numObjects, static_cast<uint64_t>(0));
KERNEL_CALL(cudaExtractObjectData, data, mappedCellBuffer, _numObjects, context);
CHECK_FOR_DEVICE_ERRORS(cudaGraphicsUnmapResources(1, &renderingData.vertexBuffer));
```

`cudaGraphicsResourceGetMappedPointer` yields a device pointer *into the OpenGL buffer's own storage*. The extraction kernel writes vertices straight there. Nothing is copied: the simulation's output lands in the memory OpenGL will read at draw time, and the CPU never sees a vertex.

What makes this exemplary is the fallback. `extractObjectData` takes a `bool useInterop`, and its `else` branch launches **the same kernels** against plain device buffers:

```cpp
// source/EngineImpl/GeometryKernelsService.cu, chrxh/alien (BSD-3-Clause) — no-interop branch
setValueToDevice(_numObjects, static_cast<uint64_t>(0));
KERNEL_CALL(cudaExtractObjectData, data, renderingData.deviceObjectBuffer, _numObjects, context);
```

Only the destination pointer differs. The no-interop path then pays a device-to-host `cudaMemcpy` per buffer in `CudaGeometryBuffers::copyToOpenGL`, followed by an upload through the ordinary GL API. **Interop is therefore not a structural property of the engine but a choice about where one pointer comes from** — which is why the same binary can run headless via `source/Cli` with no GL context at all. Structuring interop as a swappable destination pointer, rather than threading GL resources through the simulation code, is the design lesson.

### 6.5 The ROCm/HIP Backend as an Aliasing Shim

ALIEN has added an AMD backend, enabled with `-DUSE_HIP=ON` and requiring ROCm 7.2 or newer. The README documents `CMAKE_HIP_ARCHITECTURES` values of `gfx90a` (CDNA2 / MI200) and `gfx1100` (RDNA3), and notes that `-DCMAKE_PREFIX_PATH=/opt/rocm` is needed so that `find_package(hip)` resolves under the vcpkg toolchain.

The interesting part is the mechanism. Rather than porting the CUDA source, the project ships a single 100-line header, `external/hip/cuda_to_hip.h`, that aliases the CUDA API surface onto HIP, and force-includes it into every translation unit via a compiler flag:

```cmake
# CMakeLists.txt, chrxh/alien (BSD-3-Clause)
set(_HIP_FORCE_INCLUDE "-include${CMAKE_CURRENT_SOURCE_DIR}/external/hip/cuda_to_hip.h")
set(_CXX_FORCE_INCLUDE "-include${CMAKE_CURRENT_SOURCE_DIR}/external/hip/cuda_to_hip.h")
...
add_compile_options($<$<COMPILE_LANGUAGE:HIP>:${_HIP_FORCE_INCLUDE}>)
...
  add_compile_options($<$<COMPILE_LANGUAGE:CXX>:${_CXX_FORCE_INCLUDE}>)
```

Both the HIP and the host C++ compilation get the shim, through separate generator expressions guarded so that neither fires on the CUDA path.

The header covers error handling, device management, memory, streams, CUDA graphs, OpenGL interop, and device intrinsics. Its own comments record two design constraints that are the whole reason the approach works.

First, on the NVIDIA path the header is not compiled at all, so the CUDA build is byte-for-byte unchanged. The portability layer imposes zero risk on the primary target — which is what makes it acceptable to force-include something into every file.

Second, the header deliberately does *not* alias project identifiers that merely look like CUDA names. `cudaSimulationParameters`, `cudaSettings`, the `cudaTO*` transfer-object types, and the `cudaNextTimestep_*` kernels are ALIEN's own symbols that happen to share the `cuda` prefix. A naive `sed`-style rename would have mangled them. The shim's exclusion list is the difference between a working aliasing layer and a broken one, and any project attempting the same trick will need an equivalent list.

The intrinsics section carries the one genuine semantic difference. CUDA's block-scoped atomics — `atomicAdd_block`, `atomicExch_block` — have no HIP equivalent with the same scope qualifier, so the shim maps them onto the plain `atomicAdd` and `atomicExch`. That is *correct but potentially slower*: a block-scoped atomic on NVIDIA hardware can be resolved within the SM without going to a device-wide coherence point, whereas the unqualified form must be device-visible. The build also sets `CMAKE_HIP_SEPARABLE_COMPILATION ON` (the HIP equivalent of `-fgpu-rdc`) to match the CUDA build's use of device-side linking, and a nightly CI workflow (`.github/workflows/hip-nightly-ci.yml`) guards against drift between the two paths.

The generalisable claim: for a codebase that uses the CUDA runtime API rather than driver-level or vendor-specific features, HIP portability can be a header and a build flag rather than a port. The cost is confined to intrinsics with no cross-vendor equivalent, and those are exactly what to audit first when evaluating the approach.

---

## 7. Digital Organisms: The Divergent-Control-Flow Case

The systems in this section predate practical GPGPU and remain CPU-bound. They are included because the reason they stayed there is architecturally instructive rather than merely historical.

### 7.1 Avida

Avida is a digital-evolution platform in which each organism is a self-replicating program written in an assembly-like instruction set, executing on its own virtual CPU in a shared population. Organisms compete for CPU cycles, replicate with mutation, and can be rewarded for performing logic tasks, which drives the evolution of computational capability. It is a C++ codebase built with CMake, shipping an ncurses viewer, and organised as a top-level distribution over multiple submodules ([devosoft/avida](https://github.com/devosoft/avida)).

**A licence correction:** this book's outline recorded Avida as LGPL-2.1. The upstream `avida-core/COPYING` file states that Avida is distributed under the GNU Lesser General Public License "either version 3 of the License or (at your option) any later version" — that is, **LGPL-3.0-or-later**. GitHub's API reports no detected licence for the repository, so the `COPYING` file is the authority here.

### 7.2 Polyworld

Polyworld is an artificial-life system built as an approach to artificial intelligence, in which agents controlled by evolved neural networks live, forage, mate, and compete in a 2D world, with the network topology and learning parameters themselves under evolutionary control ([polyworld/polyworld](https://github.com/polyworld/polyworld)).

**Licence flag:** Polyworld is distributed under the **Apple Public Source License version 2.0**, per its `LICENSE.txt`. This is an unusual choice for a scientific codebase and is worth flagging explicitly to anyone considering reuse. APSL 2.0 is a weak-copyleft, file-based licence. The FSF classifies it as a free software licence that is nonetheless **incompatible with the GPL** ([FSF licence list](https://www.gnu.org/licenses/license-list.html#apsl2)), and it is [OSI-approved](https://opensource.org/license/apsl-2-0). Its terms differ materially from BSD/MIT in ways that matter for reuse — notably obligations attaching to "Externally Deployed" modifications, and a patent-termination clause — so the licence text itself should be read rather than inferred from the badge. Code from it is therefore described here rather than quoted, and integration into a differently licensed project needs legal review rather than a licence-badge check.

### 7.3 Framsticks

Framsticks is a long-running 3D artificial-life simulator in which agents have evolvable morphologies as well as evolvable neural controllers. It is documented at [framsticks.com](https://www.framsticks.com/), and it is the third member of the historical trio alongside Avida and Polyworld.

**Needs verification:** the current licensing status, source-availability terms, and any GPU-acceleration support for Framsticks were not confirmed against a primary source during the preparation of this chapter. It is mentioned here for historical completeness only; no claim is made about its implementation architecture, and readers should consult the project site directly before relying on any specific characterisation of it.

### 7.4 Why These Stayed on the CPU

Each of the three violates §1.1's uniformity assumption at the deepest level, and the specific violation differs in an informative way:

- **Avida.** Every organism runs a *different program*. Within a warp, sixteen organisms will be at sixteen different instruction pointers executing sixteen different opcodes. A GPU must serialise every distinct instruction across the warp, so a 32-wide warp running 32 different programs achieves roughly 1/32 of peak. Worse, organisms have unbounded and unequal lifetimes and replicate at unpredictable moments, so the population size and per-organism work are both dynamic — there is no fixed domain to dispatch against.
- **Polyworld.** Each agent has a *different network topology*, evolved rather than designed. The neural evaluation that would be a dense matrix multiply for a fixed architecture — the one thing GPUs do best — becomes a sparse, per-agent, irregular graph traversal, with a different sparsity pattern per agent and no shared structure to batch.
- **Physical-morphology simulators generally.** Evolving body plans means every agent has a different number of joints, segments, and contact points, so the physics solve is a differently shaped problem per agent.

The common thread: **these systems evolve their own structure, and structure is exactly what a GPU needs fixed in advance.** Evolving parameters within a fixed architecture is GPU-friendly, and modern neuroevolution work that targets GPUs takes precisely that route — fix the topology, batch the population into a tensor dimension, and evolve weights. Evolving the architecture itself is not.

ALIEN is the interesting counterexample and shows where the boundary actually lies. Its organisms genuinely evolve their morphology — the genome encodes a cell graph built at runtime by constructor cells. But it stays GPU-resident because the *simulation substrate* is uniform: whatever an organism's shape, it is made of cells drawn from a fixed set of types, and each type is processed by its own kernel over all cells of that type in the world (§6.2). The variable structure lives in the data — bond graphs, genomes — while the code executed per element remains fixed and homogeneous. That inversion is the design move that makes open-ended evolution tractable on a GPU, and it is the single most transferable idea in this chapter.

It is worth being precise about what "GPU-hostile" does and does not mean here, because it is tempting to conflate computational sophistication with GPU-hostility, and elementary cellular automaton Rule 110 is the cleanest available proof that the two are independent. Rule 110 is one-dimensional, two-state, radius-1 — its entire rule table has eight entries — so its update is uniform, stateless, and embarrassingly parallel in exactly the sense of §1.1. Despite that, Rule 110 is computationally universal: Matthew Cook proved it capable of simulating an arbitrary Turing machine, by constructing the necessary logic gates and signal-carrying structures entirely out of colliding gliders in its own evolution ([Cook, "Universality in Elementary Cellular Automata," *Complex Systems* 15(1), 1–40 (2004)](https://wpmedia.wolfram.com/uploads/sites/13/2018/02/15-1-1.pdf); discussed at length in [*A New Kind of Science*, Ch. 11 §8–10](https://www.wolframscience.com/nks/chap-11--the-rule-110-cellular-automaton/)). A Rule 110 simulation running a universal computation is, from a compute-shader's point of view, indistinguishable from one doing anything else: every cell still runs the identical eight-entry lookup, so §2.1's grid ping-pong kernel ports it unmodified. What breaks GPU-friendliness in Avida, Polyworld, and body-plan evolution generally is not the sophistication of the computation each organism performs — it is that *different organisms run different rules*, which Rule 110 never does. Heterogeneity across elements is the obstacle; universal computation within a uniform element is not.

Treating a cellular automaton as a large-scale parallel-hardware workload, rather than as a mathematical curiosity run on one machine, predates GPGPU by decades. Wolfram's own "Cellular Automaton Supercomputing" — whose Figure 1 illustrates Rule 110 as one of four one-dimensional rules generating "considerable complexity" from a single-site or random initial condition — reports a 2D fluid-dynamics cellular automaton run on a Connection Machine with 65,536 processors, updating lattices as large as 4096×8192 at roughly 10⁹ site updates per second, and attributes the result explicitly to "the readily scalable architecture of the Connection Machine computer" ([Wolfram, "Cellular Automaton Supercomputing," in *High-Speed Computing: Scientific Applications and Algorithm Design*, University of Illinois Press, 1988; NASA Technical Reports Server 19870017109](https://ntrs.nasa.gov/citations/19870017109)). The properties that made a CA rule suit a 65,536-processor SIMD machine in 1987 — bounded locality and embarrassing per-element parallelism — are exactly §1.1's properties, and they are the same properties a compute shader rewards forty years later. The mapping this chapter draws between CA rules and GPU dispatches is not a new observation about the hardware; it is the same observation about the *rule*, applied to newer hardware built on the same principle.

---

## 8. Comparison and Portability

### 8.1 Pattern Comparison

| | Grid ping-pong | Spectral convolution | Agent multi-pass |
|---|---|---|---|
| **Parallel domain** | One thread per cell | One thread per frequency bin, then per cell | One thread per agent (agent passes); one per texel (field passes) |
| **Dispatch shape** | 2-D, e.g. 8×8 | Library-determined (FFT) | 1-D over agents; 2-D over field |
| **Memory access** | Small stencil; spatially coherent | Strided/butterfly; global per transform | Agent reads coalesced; field reads scattered by agent position |
| **Cost per step** | *O(N)*, constant per cell | *O(N log N)*, independent of kernel radius | *O(A)* + *O(N)* over field |
| **Synchronization** | Buffer swap at dispatch boundary | Dispatch boundary between transform stages | Dispatch boundary between all four stages |
| **Extra memory** | 2× grid | Complex arrays + pre-transformed kernels | 2× agent array + 2× field + dots texture |
| **Scatter handling** | None needed | None needed | Rasterizer, or atomics |
| **Best for** | Small fixed neighbourhoods | Large smooth kernels | Mobile populations coupled via a field |
| **Breaks down when** | Neighbourhood radius grows | Kernel is small, or non-shift-invariant | Agents interact directly, not via a field |

### 8.2 Choosing a Pattern

The decision reduces to three questions asked in order:

1. **Is the parallel domain the output grid, or a population?** A population means pattern 3, and probably two ping-pongs rather than one.
2. **How large is the neighbourhood?** Below roughly a 5×5 or 7×7 stencil, a direct gather (pattern 1) beats an FFT, because the transform's constant factor dominates. Above that, and certainly above radius 10, the *O(N log N)* independence from radius wins decisively (pattern 2). The crossover is hardware- and grid-size-dependent and should be measured, not assumed.
3. **Is the rule shift-invariant?** The FFT approach requires it — the convolution theorem does not apply to a position-dependent kernel. A model with spatially varying rules is back to a direct gather regardless of radius.

Two cross-cutting rules apply to all three, and are the most common sources of both bugs and wasted bandwidth:

- **Any pass that reads a neighbourhood needs separate read and write resources.** This is §2.3, and it is not negotiable within a dispatch. Only element-local passes may run in place.
- **Keep data resident on the device across the whole step.** This is §4.3's failure mode, and it survives every generation of API. A pipeline of ten well-optimised device-side passes is faster than three optimal passes with host round trips between them.

### 8.3 WebGPU as the Portable ALife Substrate

Two of the three worked examples in this chapter — the Game of Life sample and the Physarum implementation — target WebGPU, and the reason is worth stating: WebGPU is currently the only compute API that runs a single unmodified shader across Vulkan, Metal, D3D12, and the browser.

On Linux specifically, a WGSL compute shader submitted through Dawn or wgpu is translated to SPIR-V and executed by a Vulkan driver — RADV, ANV, NVK, or a proprietary driver — so the ALife kernel ultimately runs through the same stack as any native Vulkan compute workload, with the browser's GPU-process boundary in front of it. Chapter 35 traces that path in detail.

For ALife specifically, this matters more than for most GPU workloads, because these are *artefacts to be shared*: an interesting Lenia creature or a Physarum parameter set is a result someone wants to show to other people. A WebGPU implementation can be published as a URL and run on an unmodified machine, which is a fundamentally different distribution story from "clone this, install CUDA 11.2 or newer, build with vcpkg and Ninja."

The costs are real and should be stated. Base WebGPU has no subgroup operations (they exist as an extension with limited availability), no storage-texture atomics, no `read_write` storage textures in the core feature set, a default workgroup-storage limit of 16 KiB, and default buffer-size limits well below what a large simulation wants. §5.4's rasterizer-based scatter is a direct consequence of the atomics gap, and §5.5's read/write texture split is a direct consequence of the storage-texture access rules. A CUDA implementation like ALIEN's has none of those constraints, and uses that freedom for CUDA Graphs, block-scoped atomics, and zero-copy GL interop — none of which have a WebGPU equivalent today.

The honest summary is that WebGPU is the right target for a shareable ALife artefact, and CUDA or native Vulkan is the right target for a simulation that needs to be as large or as fast as the hardware allows. Chapter 98 covers shipping the compiled-language half of such a system to the browser via WebAssembly, which is how the Rust Physarum implementation reaches a URL at all.

### 8.4 Licensing in the ALife Ecosystem

A practical observation for anyone building on this ecosystem: ALife repositories are unusually likely to carry no licence, or an unexpected one. Within the small set surveyed for this chapter:

- **Permissively licensed and safely reusable:** webgpu-samples (BSD-3-Clause), Chakazul/Lenia (MIT), tom-strowger/physarum-rust (MIT), hunar4321/particle-life (MIT), chrxh/alien (BSD-3-Clause), fogleman/physarum (MIT).
- **Copyleft or unusual:** Golly (GPL-2.0-or-later per `docs/License.html`), Avida (LGPL-3.0-or-later per `avida-core/COPYING`, not the LGPL-2.1 sometimes cited), Polyworld (APSL 2.0 — see §7.2).
- **Non-commercial:** at least one widely referenced WebGPU slime-mold implementation is licensed CC BY-NC-SA 4.0, whose NonCommercial term makes it incompatible with permissive redistribution and with CC BY 4.0 works such as this book.
- **No licence file at all:** GarrettGunnell/Compute-Game-Of-Life, erwanplantec/FlowLenia, morgangiraud/leniax.

The last category is the one most often mishandled. Absent an explicit grant, code is under exclusive copyright by default; a public repository is not a licence. An unlicensed repository is therefore legally more restrictive than a NonCommercial-licensed one, even though the latter looks more forbidding. Every unlicensed repository in this chapter is therefore described with file-and-line citations rather than quoted, and the correct action before reuse is to open an issue asking upstream to add a licence.

---

## 9. AI-Driven Search and Evolution in ALife

Everything so far has treated the *rule* as hand-designed and the GPU's job as executing it fast. A separate line of work treats the rule itself — or the search over rules, or the organisms subject to evolution — as the thing to be learned or automated, and it is worth a closing section because it changes what "the workload" even is. Three developments matter for the architectural questions this chapter asks: the update rule can itself be a trained neural network rather than a hand-written formula (§9.1); the evolutionary search loops that ALife has always used can themselves be reformulated as GPU-batched tensor programs, the same move §4.4 made for Lenia's simulation step (§9.2); and at least one recent simulator resolves §7.4's "structure must be fixed for GPU residency" thesis by fixing *architecture* while leaving *behaviour* open-ended (§9.3). A fourth development attacks §7.1's Avida problem from a different angle entirely, using large language models rather than mutation-and-selection to evolve organism-level programs (§9.4).

### 9.1 Neural Cellular Automata: The Update Rule as a Trained Network

A Neural Cellular Automaton (NCA) keeps every structural property §1.1 lists — uniformity, bounded locality, statelessness across steps, embarrassing parallelism within a step — and replaces the hand-written rule with a small neural network applied identically at every cell. The founding result grew a target image from a single seed pixel purely through local update rules learned by backpropagating through many simulation steps ([Mordvintsev, Randazzo, Niklasson, Levin, "Growing Neural Cellular Automata," *Distill* (2020), doi:10.23915/distill.00023](https://distill.pub/2020/growing-ca/)). Because the trained rule is still uniform and local, an NCA is exactly as GPU-friendly as Conway's Life — the only change from §2.1 is that `countNeighbors` becomes a small learned convolutional stack instead of a sum.

A follow-up from the same research group generalises the idea to texture synthesis and ships as ordinary framework code rather than only a notebook, which makes it directly quotable here. `self_organising_systems/texture_ca/ca.py` in the Apache-2.0-licensed [google-research/self-organising-systems](https://github.com/google-research/self-organising-systems) repository — the companion codebase to the Distill article above and to the closely related "Texture Generation with Neural Cellular Automata" ([Mordvintsev, Niklasson, arXiv:2105.07299](https://arxiv.org/abs/2105.07299)) — implements the update rule as a perception step followed by a tiny dense network:

```python
# self_organising_systems/texture_ca/ca.py, google-research/self-organising-systems (Apache-2.0)
@tf.function
def perceive(x, angle=0.0, repeat=True):
  chn = tf.shape(x)[-1]
  identify = np.outer([0, 1, 0], [0, 1, 0])
  dx = np.outer([1, 2, 1], [-1, 0, 1]) / 8.0  # Sobel filter
  dy = dx.T
  laplacian = np.outer([1, 2, 1], [1, 2, 1]) / 8.0
  laplacian[1, 1] -= 2.0
  c, s = tf.cos(angle), tf.sin(angle)
  kernel = tf.stack([identify, c*dx-s*dy, s*dx+c*dy, laplacian], -1)[:, :, None, :]
  kernel = tf.repeat(kernel, chn, 2)
  ...
  y = tf.nn.depthwise_conv2d(x, kernel, [1, 1, 1, 1], pad_mode)
  return y
```

`perceive` is the neighbourhood-read step of §1.1 made explicit: a depthwise convolution with a fixed Sobel/Laplacian kernel bank, one 3×3 gather per channel, run over the whole grid as a single dispatch. The result feeds two dense layers implemented as 1×1 convolutions — `tf.nn.conv2d(x, w, 1, 'VALID')` rather than `tf.matmul`, because, per the source's own comment, "TFjs matMul doesn't work with non-2d tensors" — and the update is applied stochastically:

```python
# self_organising_systems/texture_ca/ca.py, google-research/self-organising-systems (Apache-2.0) — CAModel.embody
update_mask = tf.random.uniform(tf.shape(x[:, :, :, :1])) <= fire_rate
x += dx * tf.cast(update_mask, tf.float32)
```

The `fire_rate` mask — every cell independently decides whether to apply its own computed delta this step — is not a training convenience; it is the mechanism that makes the trained rule asynchronous-tolerant, so that the resulting pattern is stable under the same GPU-scheduling non-determinism §2.3 describes for a naive in-place stencil, rather than fragile to it. `perceive` is invoked with `pad_repeat` (toroidal wrap by explicit concatenation, the same topology §2.2 gets from unsigned modulo and §4.2 gets from the FFT for free), so the entire model — perception, two dense layers, stochastic update — is a per-cell function with no branch on cell identity, and it drops into §2.1's ping-pong dispatch exactly like Life's rule does.

### 9.2 GPU-Native Evolutionary Computation: evosax, QDax, EvoJAX, and Brax

ALife has always leaned on evolutionary search — evolving Lenia parameters, Physarum sensing angles, or (§7) whole organism genomes. §4.4 already showed one search system, ASAL, built entirely in JAX; ASAL's own search algorithm is Sep-CMA-ES supplied by [evosax](https://github.com/RobertTLange/evosax), which generalises the point beyond one search strategy. A small cluster of Apache-2.0/MIT JAX libraries reformulates the evolutionary-computation outer loop itself — not just the fitness evaluation — as a batched tensor program, applying the same architectural move §4.4 made for Lenia's simulation step to the search loop wrapped around it.

`evosax` provides evolution strategies behind a uniform `ask`/`tell` interface, and the whole loop is `jax.jit`-compiled with the population handled by `jax.vmap` rather than a Python loop over individuals ([Lange, "evosax: JAX-based Evolution Strategies," arXiv:2212.04180](https://arxiv.org/abs/2212.04180)):

```python
# evosax/algorithms/base.py, RobertTLange/evosax (Apache-2.0)
@partial(jax.jit, static_argnames=("self",))
def ask(
    self,
    key: jax.Array,
    state: State,
    params: Params,
) -> tuple[Population, State]:
    """Ask evolutionary algorithm for new candidate solutions to evaluate."""
    # Generate population
    population, state = self._ask(key, state, params)

    # Unravel population
    population = jax.vmap(self._unravel_solution)(population)

    return population, state
```

This is the population equivalent of §5.3's per-agent dispatch: one array axis is "which individual," `vmap` maps the per-individual solution-unraveling function across it with no Python-level iteration, and `jax.jit` fuses generation, evaluation, and the strategy's parameter update into one compiled call. Where a classical genetic algorithm's inner loop is `for individual in population: evaluate(individual)`, the JAX formulation makes population size a tensor dimension the accelerator batches over directly.

QDax applies the same move to *quality-diversity* search, where the goal is not one best solution but an archive of diverse high-performing ones spanning a behaviour-descriptor space ([Lim et al., "Accelerated Quality-Diversity through Massive Parallelism," arXiv:2202.01258](https://arxiv.org/abs/2202.01258)). Its `MAPElites.update` composes the same three-step ask/score/tell shape, with the scoring step itself vmapped over the batch of candidate genotypes:

```python
# qdax/core/map_elites.py, adaptive-intelligent-robotics/QDax (MIT) — MAPElites.update
# generate offsprings with the emitter
key, subkey = jax.random.split(key)
genotypes, extra_info = self.ask(repertoire, emitter_state, subkey)

# scores the offsprings
key, subkey = jax.random.split(key)
(fitnesses, descriptors, extra_scores) = self._scoring_function(
    genotypes, subkey
)

repertoire, emitter_state, metrics = self.tell(...)
```

QDax couples routinely to [Brax](https://github.com/google/brax), a differentiable rigid-body physics engine whose entire step function is itself JAX-traceable ([Freeman et al., "Brax — A Differentiable Physics Engine for Large Scale Rigid Body Simulation," arXiv:2106.13281](https://arxiv.org/abs/2106.13281)), so that `_scoring_function` above can be thousands of Brax environments stepped in parallel rather than a CPU-side simulator called once per genotype. [EvoJAX](https://github.com/google/evojax) targets the same niche with a lighter-weight `ask`/`tell` core aimed at neuroevolution specifically ([Tang et al., "EvoJAX: Hardware-Accelerated Neuroevolution," arXiv:2202.05008](https://arxiv.org/abs/2202.05008)). None of these four libraries is an ALife simulator on its own; they are the GPU-native answer to the outer evolutionary loop that wraps whatever simulator — Lenia, Physarum, a neural cellular automaton, or a JaxLife-style agent population (§9.3) — sits inside it.

### 9.3 JaxLife and the Fixed-Architecture Resolution to §7.4

§7.4 concluded that Avida-style systems stay off the GPU because *structure* — which instructions an organism executes, which network topology it evolved — is exactly what a GPU needs fixed in advance, and that ALIEN's answer is to keep structure in the data (bond graphs, genomes) while keeping the code executed per element fixed. JaxLife applies the same inversion to agent-based ALife directly: agents are parametrised by a recurrent neural network with a *fixed architecture*, evolved via natural selection that mutates weights, not topology, while the *behaviour* those weights encode — communication, tool use, rudimentary agriculture — is left fully open-ended ([Lu, Beukman, Matthews, Foerster, "JaxLife: An Open-Ended Agentic Simulator," arXiv:2409.00853, ALIFE 2024](https://arxiv.org/abs/2409.00853)). Because every agent runs the identical network forward pass, the population evaluation is a batched matrix-multiply problem — the one thing a GPU does best, and precisely the case §7.4 contrasts against Polyworld's per-agent, differently-shaped sparse graph.

The simulation goes further than a fixed neural controller: agents can program *robot* entities in the world, and those robots are capable of Turing-complete computation — the project's own repository illustrates this with a rendered Rule 110 pattern, the same automaton discussed in §7.4 as the proof that universal computation and GPU-uniform execution are compatible. JaxLife is, in that sense, a direct existence proof of §7.4's closing claim applied to open-ended agent behaviour rather than to ALIEN's cell physics: unbounded behavioural and cultural complexity coexists with a GPU-resident, batch-uniform simulation substrate, as long as the thing left free to vary is *what the fixed network computes*, not *what code runs*.

**Licensing note:** the [luchris429/JaxLife](https://github.com/luchris429/JaxLife) repository ships no `LICENSE` file and GitHub reports no detected licence, so — consistent with §8.4's rule — it is described here rather than quoted.

### 9.4 LLM-Driven Program Evolution: Digital Red Queen and the Avida Lineage

§7.1 and §7.4 identified the specific obstacle that has kept Avida off the GPU for its entire history: organisms are independent programs in a custom instruction set, and a population of them is a population of divergent control-flow paths with no shared structure to batch. Three decades on, that obstacle is unchanged for Avida itself (see the Roadmap), but a recent project attacks the same problem class — evolving competing self-replicating assembly-level programs — from a different angle, using a large language model as the mutation-and-recombination operator instead of a genetic algorithm.

Digital Red Queen (DRQ) evolves "warriors," self-replicating assembly-like programs in Core War, a Turing-complete virtual-machine game with a decades-long history as an ALife testbed in its own right. Rather than random mutation and fitness-proportional selection, an LLM is prompted to write a new warrior specifically to defeat every warrior evolved so far, and the process repeats — an explicit implementation of Red Queen dynamics, where continued adaptation is driven by a constantly shifting competitive target rather than a fixed fitness function. Independent runs, started from unrelated initial conditions, were reported to converge toward similar general-purpose strategies ([Kumar et al., "Digital Red Queen: Adversarial Program Evolution in Core War with LLMs," arXiv:2601.03335, GECCO 2026](https://arxiv.org/abs/2601.03335); project page [sakana.ai/drq](https://sakana.ai/drq/)).

This does not resolve §7.4's architectural problem — DRQ's warriors still execute as divergent per-organism programs, and the project runs that execution on ordinary CPU-hosted Core War virtual machines, not on a GPU. What has changed is where the *search* runs: the LLM proposing new warriors is a single large, batchable, GPU-resident forward pass, even though the artefact it produces — a heterogeneous population of assembly programs — remains exactly as GPU-hostile to *execute* as Avida's organisms always were. The distinction is the same one §9.2 draws for evolutionary search generally: the outer loop that finds structure can be GPU-native even when the structure it finds is not.

---

## Integrations

**Chapter 35 — Dawn and WebGPU** traces the path a WebGPU call takes from JavaScript through Chrome's GPU-process boundary into Mesa's Vulkan drivers. Every WGSL compute shader quoted in §2.1 and §5.3–5.5 travels that exact path: the `@compute` entry point is compiled by Tint to SPIR-V, the `@group`/`@binding` declarations become a Vulkan descriptor set layout, and the dispatch becomes a `vkCmdDispatch` on a compute queue. The constraints that shape §5.4's rasterizer-based deposit and §5.5's read/write texture split are WebGPU validation rules imposed above that translation layer rather than hardware limits below it — which is why the same simulation written in native Vulkan would be free to use storage-image atomics that the WebGPU version must design around.

**Chapter 98 — WebAssembly and WebGPU as a Deployment Target** covers the compilation and packaging model that makes §5's Rust implementation reachable as a URL. The simulation logic compiles to `wasm32`, the WGSL shaders travel as strings, and the WebGPU device is acquired through the browser rather than through a loader — meaning the ALife artefact ships as a page with no driver installation. That chapter's deployment story is the direct answer to §8.3's argument about why WebGPU is the right target for a *shareable* simulation, and the direct contrast with ALIEN's CUDA Toolkit 11.2 / vcpkg / Ninja build requirement in §6.1.

**Chapter 66 — CUDA Runtime, Streams, and NVRTC** documents the API layer that §6 exercises in production. The stream-capture and graph-instantiation sequence in §6.3 is the runtime's graph API applied to a real pipeline whose launch sequence varies by configuration, and ALIEN's configuration-keyed `cudaGraphExec_t` cache is a concrete answer to the practical question that chapter raises — what to do when a graph must be static but the workload is not. The zero-copy interop in §6.4 similarly extends that chapter's device-memory model with the graphics-resource registration path, where the device pointer originates in an OpenGL buffer object rather than in `cudaMalloc`.

**Chapter 205e — FOSS Simulation-Game Frameworks** examines simulation systems from the opposite architectural end: data models, rules engines, and modding surfaces, with rendering deliberately out of scope. Placing the two chapters side by side isolates a real design fork. A rules-engine simulation puts its complexity in *heterogeneous per-entity behaviour*, which is the property §7.4 identifies as fundamentally GPU-hostile. A GPU-resident simulation puts its complexity in *homogeneous per-element update rules applied to variable data*, which is ALIEN's inversion in §7.4. The same domain — an evolving population in a shared world — admits both architectures, and the choice determines almost everything else about the implementation. §9.3 shows a third point on that spectrum: JaxLife fixes agent *architecture* — a neural-network policy evaluated identically for every agent — while leaving *behaviour* fully open-ended, which is a different way of satisfying the same GPU constraint than ALIEN's fixed-cell-type inversion.

**Chapter 205c — Open-Source 2D Simulation-Game Engines** traces the CPU-to-GPU migration path in shipped 2D codebases, which is the same journey §5.6 sets up as an exercise. Particle Life's commented-out `#pragma omp parallel` is the exact state that chapter's engines began from, and the progression it documents — CPU loop, then threaded, then batched onto the GPU — is the concrete roadmap for the port sketched in §5.6, including the spatial-binning step that turns an *O(n²)* interaction into an *O(n)* one.

**Chapter 205d — Modding Architectures** covers scripting and sandboxing for user-supplied extension code. It is worth reading against §7.1's Avida, which is in effect a modding architecture taken to its limit: the "mods" are self-replicating programs in a custom instruction set, written by evolution rather than by users, executing on a virtual CPU per organism. The isolation properties that chapter demands of a plugin sandbox — bounded memory, no ambient authority, preemptible execution — are exactly the properties a digital-organism virtual CPU must provide, and for the same reason. The divergence analysis in §7.4 then explains why that architecture cannot be moved to a GPU: a sandbox per entity is a distinct instruction stream per entity.

---

## References

- [GitHub — webgpu/webgpu-samples](https://github.com/webgpu/webgpu-samples) — BSD-3-Clause; `sample/gameOfLife/compute.wgsl` double-buffered storage-buffer ping-pong, `override` workgroup size, branchless `select` transition (§2.1–2.2)
- [W3C — WebGPU Shading Language (WGSL) specification](https://www.w3.org/TR/WGSL/) — `override` declarations, storage address-space access modes, `arrayLength`, unsigned wrapping arithmetic, structure alignment rules (§2.1–2.2, §5.3)
- [W3C — WebGPU specification](https://www.w3.org/TR/webgpu/) — Storage-texture access modes, default limits, absence of a device-wide barrier within a dispatch (§2.3, §5.5, §8.3)
- [GitHub — GarrettGunnell/Compute-Game-Of-Life](https://github.com/GarrettGunnell/Compute-Game-Of-Life) — **No licence file**; described not quoted. `Assets/Game Of Life.compute` single-`RWTexture2D` in-place stencil update; `Assets/Life.cs` host dispatch and blit (§2.3)
- [Golly — SourceForge project page](https://sourceforge.net/projects/golly/) — Canonical upstream for the Life explorer; GPL-2.0-or-later per `docs/License.html` (§3)
- [GitHub — AlephAlpha/golly](https://github.com/AlephAlpha/golly) — Actively synced unofficial mirror of the SourceForge repository; `gollybase/hlifealgo.h` quadtree canonicalization and memoized lookahead, `gollybase/qlifealgo.cpp` conventional algorithm. Note that `GollyGang/golly` does not exist (§3.1–3.3)
- [GitHub — Chakazul/Lenia](https://github.com/Chakazul/Lenia) — MIT; `Python/LeniaNDK.py` reikna backend, pre-transformed kernel bank, `run_gpu` host round trip; `Python/requirements.txt` reikna 0.7.5 pin (§4.1–4.3)
- [reikna documentation](https://reikna.publicfields.net/en/latest/) — PyOpenCL/PyCUDA-backed GPGPU algorithms; `cluda.any_api()`, `fft.FFT`, `fft.FFTShift` (§4.3)
- [GitHub — erwanplantec/FlowLenia](https://github.com/erwanplantec/FlowLenia) — **Licence unconfirmed**; described not quoted. `flowlenia/flowlenia.py:82,86` `jnp.fft.fft2`/`ifft2`, kernel transform carried in state, `rollout` over a JIT-compiled `_step` (§4.4)
- [Flow Lenia: Mass conservation for the study of virtual creatures in continuous cellular automata (arXiv:2212.07906)](https://arxiv.org/abs/2212.07906) — Model definition and reintegration tracking (§4.4)
- [GitHub — morgangiraud/leniax](https://github.com/morgangiraud/leniax) — **Licence unconfirmed**; described not quoted. `leniax/core.py:81,87` `get_potential_fft` over arbitrary world dimensionality; JAX/QD search engine framing (§4.4)
- [Jones (2010), Characteristics of pattern formation and evolution in approximations of Physarum transport networks, Artificial Life 16(2)](https://uwe-repository.worktribe.com/output/980579/characteristics-of-pattern-formation-and-evolution-in-approximations-of-physarum-transport-networks) — Canonical agent-and-trail algorithm description (§5.1)
- [sagejenson.com — physarum](https://sagejenson.com/physarum) — Parameter-space exploration of the model, cited by most implementations (§5.1)
- [GitHub — tom-strowger/physarum-rust](https://github.com/tom-strowger/physarum-rust) — MIT; four-stage wgpu pipeline with dual agent/field ping-pong, `src/compute.wgsl` per-agent dispatch, `src/deposit.wgsl` rasterizer-to-gather deposit, `src/diffuse.wgsl` blur-and-decay with `rgba16float` clamp (§5.2–5.5)
- [GitHub — fogleman/physarum](https://github.com/fogleman/physarum) — MIT; Go CPU reference implementation of the same model (§5.1, §8.4)
- [GitHub — hunar4321/particle-life](https://github.com/hunar4321/particle-life) — MIT; `particle_life/src/ofApp.cpp` single-threaded *O(n²)* interaction loop with OpenMP and TBB both commented out (§5.6)
- [GitHub — chrxh/alien](https://github.com/chrxh/alien) — BSD-3-Clause; CUDA-only simulation, compute capability 6.0+ / CUDA 11.2+, ALIFE 2024 Virtual Creatures Competition winner, ROCm 7.2+ HIP backend and `CMAKE_HIP_ARCHITECTURES` guidance (§6.1, §6.5)
- [ALIEN `source/EngineKernels/SimulationKernels.cu`](https://github.com/chrxh/alien/blob/develop/source/EngineKernels/SimulationKernels.cu) — ~46 `cudaNextTimestep_*` kernels; per-phase and per-cell-type split; cluster connected-components passes (§6.2)
- [ALIEN `source/EngineImpl/SimulationKernelsService.cu`](https://github.com/chrxh/alien/blob/develop/source/EngineImpl/SimulationKernelsService.cu) — `cudaStreamBeginCapture`/`cudaGraphInstantiate`/`cudaGraphLaunch`, `CudaGraphConfig`-keyed `_graphCache`, `calcOptimalThreadsForFluidKernel` (§6.3)
- [ALIEN `source/EngineKernels/CudaGeometryBuffers.cu`](https://github.com/chrxh/alien/blob/develop/source/EngineKernels/CudaGeometryBuffers.cu) — `cudaGraphicsGLRegisterBuffer` with `cudaGraphicsMapFlagsWriteDiscard` over nine GL buffers; `allocateBuffersForNoInterop` and `copyToOpenGL` fallback path (§6.4)
- [ALIEN `source/EngineImpl/GeometryKernelsService.cu`](https://github.com/chrxh/alien/blob/develop/source/EngineImpl/GeometryKernelsService.cu) — `extractObjectData(..., bool useInterop)`; map/get-mapped-pointer/unmap around the same extraction kernels used by the no-interop branch (§6.4)
- [ALIEN `external/hip/cuda_to_hip.h`](https://github.com/chrxh/alien/blob/develop/external/hip/cuda_to_hip.h) — 100-line CUDA→HIP aliasing shim; force-included via `-include`; excludes project identifiers with `cuda` prefixes; `atomicAdd_block`→`atomicAdd` scope loss (§6.5)
- [NVIDIA CUDA C++ Programming Guide — Graph API](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#cuda-graphs) — Stream capture, instantiation, and launch semantics (§6.3)
- [NVIDIA CUDA C++ Programming Guide — OpenGL Interoperability](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#opengl-interoperability) — Resource registration, map flags, mapped-pointer retrieval (§6.4)
- [AMD ROCm — HIP Programming Guide](https://rocm.docs.amd.com/projects/HIP/en/latest/) — CUDA-to-HIP API correspondence, `--offload-arch`, separable compilation (§6.5)
- [GitHub — devosoft/avida](https://github.com/devosoft/avida) — Digital-evolution platform; `avida-core/COPYING` states LGPL v3 or later, correcting the LGPL-2.1 attribution sometimes cited (§7.1, §8.4)
- [GitHub — polyworld/polyworld](https://github.com/polyworld/polyworld) — Evolved-neural-network ALife system; `LICENSE.txt` is **Apple Public Source License 2.0** (§7.2, §8.4)
- [Framsticks project site](https://www.framsticks.com/) — 3D ALife simulator with evolvable morphology and control. **Licensing, source terms, and GPU support not verified for this chapter** (§7.3)
- [ALIEN `RELEASE-NOTES.md`](https://github.com/chrxh/alien/blob/develop/RELEASE-NOTES.md) — Per-release engine changelog on the `develop` branch; newest entry `[4.12.2] - 2025-05-26`, Blackwell support via CUDA Toolkit 12.8 (Roadmap)
- [ALIEN `.github/workflows/windows-nightly-deploy.yml`](https://github.com/chrxh/alien/blob/develop/.github/workflows/windows-nightly-deploy.yml) — Nightly CUDA + HIP build; `HIP_ARCHS: gfx10-3-generic;gfx11-generic;gfx12-generic` packaged as one AMD executable (§6.5, Roadmap)
- [MaCE: General Mass Conserving Dynamics for Cellular Automata (arXiv:2507.12306)](https://arxiv.org/abs/2507.12306) — Generalizes Flow-Lenia's mass conservation to arbitrary CAs, including discrete and neural CAs (§4.4, Roadmap)
- [Flow-Lenia: Emergent evolutionary dynamics in mass conservative continuous cellular automata (arXiv:2506.08569)](https://arxiv.org/abs/2506.08569) — Journal version; parameter embedding in the state field enabling multispecies simulation (§4.4, Roadmap)
- [Automating the Search for Artificial Life with Foundation Models (arXiv:2412.17799)](https://arxiv.org/abs/2412.17799) — ASAL; foundation-model-driven search across Boids, Particle Life, Game of Life, Lenia, and neural CAs (§4.4, §8.2, Roadmap)
- [GitHub — SakanaAI/asal](https://github.com/SakanaAI/asal) — Apache-2.0; ASAL reference implementation, JAX throughout and end-to-end jittable (§4.4, Roadmap)
- [Exploring Flow-Lenia Universes with a Curiosity-driven AI Scientist: Discovering Diverse Ecosystem Dynamics (arXiv:2505.15998)](https://arxiv.org/abs/2505.15998) — IMGEP-based automated discovery in Lenia's parameter space (§4.4, Roadmap)
- [Guiding Evolution of Artificial Life Using Vision-Language Models (arXiv:2509.22447)](https://arxiv.org/abs/2509.22447) — VLM-guided evolutionary search over ALife substrates (Roadmap)
- [Expedition & Expansion: Leveraging Semantic Representations for Goal-Directed Exploration in Continuous Cellular Automata (arXiv:2509.03863)](https://arxiv.org/abs/2509.03863) — Semantics-guided goal-directed exploration in continuous CAs (Roadmap)
- [Visualizing the Structure of Lenia Parameter Space (arXiv:2601.01932)](https://arxiv.org/abs/2601.01932) — Parameter-space cartography with an interactive explorer at <https://lenia-explorer.vercel.app/> (§4.1, Roadmap)
- [Chrome Platform Status — WebGPU Subgroups](https://chromestatus.com/feature/5126409856221184) — Enabled by default since Chrome 134 (§8.3, Roadmap)
- [Chrome for Developers — What's New in WebGPU (Chrome 142)](https://developer.chrome.com/blog/new-in-webgpu-142) — `texture-formats-tier1` and `texture-formats-tier2` shipping; tier2 grants `"read-write"` storage access to `rgba16float` (§5.5, §8.3, Roadmap)
- [Framsticks release history (`files/apps/history.xml`)](https://www.framsticks.com/files/apps/history.xml) — Dated per-version changelog; 5.5 on 23 April 2026 with a 5.6 entry open; no GPU or CUDA entries (§7.3, Roadmap)
- [Wolfram, *A New Kind of Science*, Chapter 12 §6 — The Notion of Computational Irreducibility](https://www.wolframscience.com/nks/p737--the-notion-of-computational-irreducibility/) — Definition and rationale for computational irreducibility and the Principle of Computational Equivalence (§3.4)
- [Wolfram, Undecidability and Intractability in Theoretical Physics, *Physical Review Letters* 54, 735 (1985)](https://www.stephenwolfram.com/publications/undecidability-intractability-theoretical-physics/) — Formal undecidability result underlying computational irreducibility (§3.4)
- [Wolfram, *A New Kind of Science*, Chapter 11 §8–10 — Universality in Cellular Automata](https://www.wolframscience.com/nks/chapter-11-the-notion-of-computation/) — Rule 110 and the class of universal one-dimensional cellular automata (§3.4, §7.4)
- [Cook, A Concrete View of Rule 110 Computation, *Complex Systems* 15(1), 1 (2004)](https://wpmedia.wolfram.com/uploads/sites/13/2018/02/15-1-1.pdf) — Formal proof of Rule 110's Turing completeness (§7.4)
- [Wolfram, Cellular Automaton Supercomputing (NASA NTRS 19870017109, 1987/1988)](https://ntrs.nasa.gov/citations/19870017109) — Connection Machine CA hardware: 65,536 processors, 4096×8192 lattices, ~10⁹ site updates/sec; Figure 1 depicts Rule 110 (§7.4)
- [Mordvintsev, Randazzo, Niklasson, Levin, Growing Neural Cellular Automata, *Distill* (2020), doi:10.23915/distill.00023](https://distill.pub/2020/growing-ca/) — Founding NCA result: a target image grown from one seed pixel via a trained local update rule (§9.1)
- [Mordvintsev, Niklasson, Texture Generation with Neural Cellular Automata (arXiv:2105.07299)](https://arxiv.org/abs/2105.07299) — NCA generalised to texture synthesis (§9.1)
- [GitHub — google-research/self-organising-systems](https://github.com/google-research/self-organising-systems) — Apache-2.0; `self_organising_systems/texture_ca/ca.py` Sobel/Laplacian `perceive`, `DenseLayer`, fire-rate-masked stochastic update (§9.1)
- [Lange, evosax: JAX-based Evolution Strategies (arXiv:2212.04180)](https://arxiv.org/abs/2212.04180) — JIT-compiled, `vmap`-batched evolution-strategy library (§9.2)
- [GitHub — RobertTLange/evosax](https://github.com/RobertTLange/evosax) — Apache-2.0; `evosax/algorithms/base.py` `ask`/`tell` interface, `jax.vmap`-batched population handling; used by ASAL's Sep-CMA-ES (§4.4, §9.2)
- [Lim et al., Accelerated Quality-Diversity through Massive Parallelism (arXiv:2202.01258)](https://arxiv.org/abs/2202.01258) — QDax; MAP-Elites reformulated as a batched JAX ask/score/tell loop (§9.2)
- [GitHub — adaptive-intelligent-robotics/QDax](https://github.com/adaptive-intelligent-robotics/QDax) — MIT; `qdax/core/map_elites.py` `MAPElites.update` (§9.2)
- [Tang et al., EvoJAX: Hardware-Accelerated Neuroevolution (arXiv:2202.05008)](https://arxiv.org/abs/2202.05008) — `ask`/`tell` neuroevolution library targeting the same JAX ecosystem (§9.2)
- [GitHub — google/evojax](https://github.com/google/evojax) — Neuroevolution library referenced alongside evosax and QDax (§9.2)
- [Freeman et al., Brax — A Differentiable Physics Engine for Large Scale Rigid Body Simulation (arXiv:2106.13281)](https://arxiv.org/abs/2106.13281) — JAX-traceable rigid-body physics, commonly coupled to QDax's scoring function (§9.2)
- [GitHub — google/brax](https://github.com/google/brax) — Differentiable physics engine referenced for QDax-scale parallel scoring (§9.2)
- [Lu, Beukman, Matthews, Foerster, JaxLife: An Open-Ended Agentic Simulator (arXiv:2409.00853), ALIFE 2024](https://arxiv.org/abs/2409.00853) — Fixed recurrent-network agent architecture with open-ended evolved behaviour and Turing-complete robot programs (§9.3)
- [GitHub — luchris429/JaxLife](https://github.com/luchris429/JaxLife) — **No licence file**; described not quoted. `pics/rule110.gif` illustrates robot-level Turing-complete computation (§9.3)
- [Kumar et al., Digital Red Queen: Adversarial Program Evolution in Core War with LLMs (arXiv:2601.03335), GECCO 2026, doi:10.1145/3795095.3805116](https://arxiv.org/abs/2601.03335) — LLM-driven, Red-Queen-style evolution of self-replicating Core War warriors (§9.4)
- [Sakana AI — Digital Red Queen project page](https://sakana.ai/drq/) — Project overview and convergent-strategy results across independent runs (§9.4)

---

## Roadmap

### Near-term (6–12 months)

- **ALIEN's AMD backend moves from a source-only build to a shipped multi-architecture binary.** The project publishes no GitHub milestones and no roadmap issue, so its direction has to be read from the repository itself. The nightly deploy workflow on `develop` now builds the CUDA and HIP backends in the same run and packages them into one archive, with `HIP_ARCHS: gfx10-3-generic;gfx11-generic;gfx12-generic` so that a single AMD executable covers RDNA2, RDNA3, and RDNA4. That turns §6.5's HIP path from a per-architecture source build into a prebuilt binary, at the cost of the fat-binary size the generic targets imply. [Source](https://github.com/chrxh/alien/blob/develop/.github/workflows/windows-nightly-deploy.yml) (§6.5)
- **A large accumulated ALIEN engine delta sits unreleased.** The most recent tagged release, `v4.12.3`, points at a merge commit dated 2025-09-08, while `develop` has continued to take engine and UI commits through August 2026. The changelog's newest entry is `[4.12.2] - 2025-05-26`, which records Blackwell support via CUDA Toolkit 12.8 — so Blackwell compatibility did ship, but the HIP backend of §6.5 appears in no tag, and readers who build from a release tarball rather than `develop` will not have it. No next-version number has been announced. [Source](https://github.com/chrxh/alien/blob/develop/RELEASE-NOTES.md) (§6.1, §6.5)
- **WebGPU subgroups are no longer an extension with limited availability.** §8.3 describes subgroup operations that way; as of the August 2026 Candidate Recommendation Draft, `"subgroups"` and `"subgroup-size-control"` are standard `GPUFeatureName` entries [Source](https://www.w3.org/TR/webgpu/), and Chrome has enabled subgroups by default since version 134 [Source](https://chromestatus.com/feature/5126409856221184). Cross-lane reductions are therefore available to portable ALife kernels wherever the feature is reported, narrowing one of the gaps §8.3 charges against WebGPU. (§8.3)
- **Extended storage-texture formats reach shipping browsers.** `texture-formats-tier1` and `texture-formats-tier2` shipped in Chrome 142; tier2 grants `"read-write"` `GPUStorageTextureAccess` to `rgba16float` among other formats. [Source](https://developer.chrome.com/blog/new-in-webgpu-142) This qualifies §8.3 in two ways: `read-write` is a core access mode, already supported without any feature flag by `r32uint`, `r32sint`, and `r32float`, and the extended format list now reaches the `rgba16float` trail field of §5.5. The ping-pong of §2.3 is nonetheless still required for any pass that reads its neighbours, because a read-write storage texture does not supply a device-wide barrier within a dispatch. (§5.5, §8.3)

### Medium-term (1–3 years)

- **Mass conservation generalizes beyond Lenia.** MaCE recasts the reintegration tracking that §4.4 describes for Flow-Lenia as a scheme applicable to arbitrary cellular automata, including discrete CAs and neural CAs. [Source](https://arxiv.org/abs/2507.12306) The journal version of Flow-Lenia meanwhile embeds per-cell parameters in the state field, so that multiple species coexist in one simulation without changing the convolution structure the GPU kernel implements. [Source](https://arxiv.org/abs/2506.08569) Both keep the spectral-convolution pattern of §4.3 intact and change only what is transported between steps. (§4.3–4.4)
- **Automated search displaces hand-tuning of ALife parameters.** ASAL uses vision-language foundation models to search for target, open-ended, and illuminated behaviour, and is implemented entirely in JAX so that rollouts are end-to-end jittable and GPU-resident. [Source](https://arxiv.org/abs/2412.17799) [Source](https://github.com/SakanaAI/asal) It spans Boids, Particle Life, Game of Life, Lenia, and neural CAs — that is, all three of the parallelization patterns this chapter separates — which makes the JAX reformulation of §4.4 a prerequisite for the search rather than merely a faster way to run one simulation. Related work extends the loop with VLM-guided evolution [Source](https://arxiv.org/abs/2509.22447) and expedition/expansion search strategies [Source](https://arxiv.org/abs/2509.03863), and IMGEP-based curiosity-driven exploration attacks the same problem without foundation models [Source](https://arxiv.org/abs/2505.15998). (§4.4, §8.2)
- **New Lenia results arrive as new JAX codebases, not as updates to the repositories cited here.** The three implementations §4.1–§4.4 draw on are dormant: `Chakazul/Lenia` was last pushed 2024-07-19, `erwanplantec/FlowLenia` 2024-02-08, and `morgangiraud/leniax` 2024-04-03. The research has not slowed correspondingly — parameter-space cartography with a browser-based interactive explorer is a 2026 result. [Source](https://arxiv.org/abs/2601.01932) Treat the chapter's code citations as pinned references to a specific formulation rather than as the currently maintained tooling. (§4.1–§4.4)

### Long-term

- **No GPU path is announced for any of the digital-organism simulators.** Avida's most recent commit is dated 2025-01-27 and is maintenance work — a population action and CI repair — with nothing touching execution strategy [Source](https://github.com/devosoft/avida). Polyworld's last substantive code commit is 2019-05-14; its later repository activity is a data-file upload [Source](https://github.com/polyworld/polyworld). Framsticks, by contrast, ships regularly — 5.5 on 23 April 2026, with a 5.6 entry already open in the changelog — but its published change history covers rigid-body collisions, UI, mobile platforms, and scripting, with no GPU or CUDA entries [Source](https://www.framsticks.com/files/apps/history.xml). This is the empirical form of §7.4's argument: the divergent, per-organism control flow these systems execute has not attracted a port, and none is planned in public. The §7.3 verification caveat on Framsticks licensing and source terms still stands. (§7.1–§7.4)
- **Storage-texture atomics remain absent from WebGPU, so §5.4's workaround is durable.** The specification still provides atomics only in the storage and workgroup address spaces; no texture-atomic operation exists in WGSL or in any listed feature. [Source](https://www.w3.org/TR/webgpu/) The rasterizer-based scatter-to-gather deposit that §5.4 describes therefore remains the portable way to accumulate agent contributions into a shared field, independent of how the format tiers evolve. (§5.4, §8.3)
- **The dispatch boundary stays the only device-wide barrier.** Nothing in the current specification or in the shipped Chrome features adds intra-dispatch global synchronization, and the roadmap items above extend formats and lane-level operations rather than the memory model. The multi-pass decomposition that §2.3, §5.2, and §6.2 all arrive at independently — a separate dispatch per phase, with double buffering as the synchronization mechanism rather than a memory convenience — is a property of the execution model, not a temporary gap. [Source](https://www.w3.org/TR/webgpu/) (§2.3, §5.2, §6.2, §8.1)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
