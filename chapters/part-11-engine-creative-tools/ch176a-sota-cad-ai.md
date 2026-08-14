# Chapter 176a: State-of-the-Art CAD AI — Generative Models, Reverse Engineering, and Agentic Design

**Target audiences:** Graphics application developers evaluating AI-assisted CAD tooling; engineers building or integrating text-to-CAD and image-to-CAD pipelines; researchers and tooling engineers who need a representation-level map of the current AI-CAD literature rather than a single tool's documentation.

---

## Table of Contents

1. [Scope: Why CAD Generation Is a Different Problem](#1-scope-why-cad-generation-is-a-different-problem)
2. [Foundational Datasets](#2-foundational-datasets)
   - 2.1 [ABC: A Big CAD Model Dataset](#21-abc-a-big-cad-model-dataset)
   - 2.2 [Fusion 360 Gallery](#22-fusion-360-gallery)
   - 2.3 [The DeepCAD Dataset and Its Descendants](#23-the-deepcad-dataset-and-its-descendants)
3. [Command-Sequence Generative Models](#3-command-sequence-generative-models)
   - 3.1 [DeepCAD: CAD as a Language](#31-deepcad-cad-as-a-language)
   - 3.2 [SkexGen: Disentangled Codebooks](#32-skexgen-disentangled-codebooks)
   - 3.3 [SketchGen and Vitruvion: Constrained Sketches](#33-sketchgen-and-vitruvion-constrained-sketches)
4. [Cutting-Edge B-Rep Frameworks: Generation and Representation Learning](#4-cutting-edge-b-rep-frameworks-generation-and-representation-learning)
   - 4.1 [BrepGen and BrepDiff: Diffusion-Based Direct Generation](#41-brepgen-and-brepdiff-diffusion-based-direct-generation)
   - 4.2 [BRepCLIP and BRep-BERT: B-Rep Representation Learning](#42-brepclip-and-brep-bert-b-rep-representation-learning)
5. [Text-to-CAD: Two Competing Philosophies](#5-text-to-cad-two-competing-philosophies)
   - 5.1 [Generate the Command Sequence Directly](#51-generate-the-command-sequence-directly)
   - 5.2 [Generate Code in an Existing CAD Language](#52-generate-code-in-an-existing-cad-language)
6. [Reverse Engineering: Point Clouds, Meshes, and Images to CAD](#6-reverse-engineering-point-clouds-meshes-and-images-to-cad)
   - 6.1 [ComplexGen: B-Rep as a Chain Complex](#61-complexgen-b-rep-as-a-chain-complex)
   - 6.2 [CAD-Recode: Point Clouds to Executable Code](#62-cad-recode-point-clouds-to-executable-code)
   - 6.3 [CAD-Coder: Images to Executable Code](#63-cad-coder-images-to-executable-code)
   - 6.4 [Point2Cyl and CSGNet: Differentiable and Program-Synthesis Primitive Fitting](#64-point2cyl-and-csgnet-differentiable-and-program-synthesis-primitive-fitting)
7. [Multimodal and Unified Models: CAD-MLLM](#7-multimodal-and-unified-models-cad-mllm)
8. [Agentic and Tool-Verified Design](#8-agentic-and-tool-verified-design)
9. [Benchmarks and the Evaluation Problem](#9-benchmarks-and-the-evaluation-problem)
   - 9.1 [CADBench: Fragmentation Made Comparable](#91-cadbench-fragmentation-made-comparable)
   - 9.2 [MUSE: Beyond Geometric Similarity](#92-muse-beyond-geometric-similarity)
10. [Commercial Landscape](#10-commercial-landscape)
11. [Open Problems](#11-open-problems)
12. [Integrations](#integrations)

---

## 1. Scope: Why CAD Generation Is a Different Problem

The broader field of 3D generative AI — text-to-3D, image-to-3D — almost always targets a mesh, a point cloud, or an implicit field: a representation a renderer can consume directly, with no requirement that it correspond to anything a human designer would have built. CAD generation targets something narrower and stricter: a model that is either a valid **parametric construction history** (a sequence of sketch-and-extrude-style operations a CAD kernel can replay) or a valid **boundary representation** (watertight, manifold NURBS topology a kernel like OCCT can accept for further editing, Boolean operations, or STEP export). A mesh that merely looks like the target object is not a CAD model; a CAD-generation system that emits invalid topology has produced nothing usable, regardless of how close its silhouette is.

This validity constraint is why CAD-AI research reads differently from image- or mesh-generation research: papers spend as much space on execution/validity rates (does the generated program run at all? is the resulting solid watertight?) as they do on visual or geometric similarity to a target. It is also why the field has split along a representation axis that recurs across every task this chapter covers — generate a **command sequence** in a fixed CAD-operation vocabulary, generate **code** in an existing scripting CAD language (CadQuery, OpenSCAD-style), or generate a **B-rep** directly as geometry/topology — a split this book's coverage of Chapter 176 §9.6 already surfaces for the narrower case of OCCT-based Text-to-CAD tooling. This chapter surveys that landscape at the representation level: what each family of approach actually outputs, what training data it depends on, and how the field currently measures whether any of it works.

## 2. Foundational Datasets

Every model in this chapter is trained or evaluated against one of a small number of shared datasets, so understanding them first clarifies what "state of the art" is actually state-of-the-art *on*.

### 2.1 ABC: A Big CAD Model Dataset

The ABC dataset assembled roughly one million real-world CAD models — collections of explicitly parametrized curves and surfaces sourced from public CAD repositories — specifically to give geometric deep learning methods ground-truth differential quantities, patch segmentation, and geometric feature labels that are otherwise expensive to produce by hand. [Source](https://arxiv.org/abs/1812.06216) It predates the generative-CAD wave by several years and functions today mainly as a geometry-processing benchmark (surface normal estimation, patch segmentation) and as raw material other datasets subsample or re-derive from, rather than as a generative-model training set in its own right.

### 2.2 Fusion 360 Gallery

Autodesk Research's Fusion 360 Gallery dataset is built from real, human-authored Fusion 360 design histories rather than static geometry, and ships as three components: a **Reconstruction** dataset of 8,625 human-designed sketch-and-extrude CAD sequences; a **Segmentation** dataset of 35,680 parts with each face labeled by the modeling operation most responsible for creating it; and tooling built on the Fusion 360 API for traversing B-rep data and converting between formats. [Source](https://arxiv.org/abs/2010.02392) [Source](https://github.com/AutodeskAILab/Fusion360GalleryDataset) Because it captures actual design *sequences* rather than final shapes, it is the dataset of choice for reconstruction and reverse-engineering methods that need to reproduce not just a shape but a plausible modeling history for it.

### 2.3 The DeepCAD Dataset and Its Descendants

DeepCAD (§3.1) introduced its own 178,238-model dataset of CAD construction sequences, derived from ABC. [Source](https://arxiv.org/abs/2105.09492) This DeepCAD dataset has since become the substrate for most text-to-CAD work: Text2CAD's annotation pipeline (§5.1) runs open-source LLMs and VLMs over the DeepCAD dataset to attach 660K text prompts across four description-complexity levels to its 170K models, rather than collecting a new geometry corpus from scratch. [Source](https://arxiv.org/abs/2409.17106) CAD-MLLM's Omni-CAD dataset (§7) similarly builds its ~450K multimodal instances on top of the same underlying CAD-sequence corpus rather than an independent one. [Source](https://arxiv.org/abs/2411.04954) The practical consequence: nearly every model in this chapter is ultimately trained on some derivative of a few thousand real Fusion 360 histories and one large ABC-derived corpus — a data-scarcity point this chapter returns to in §11.

## 3. Command-Sequence Generative Models

### 3.1 DeepCAD: CAD as a Language

DeepCAD was the first generative model to target CAD construction sequences directly rather than a discrete shape representation like voxels or point clouds, framing the problem as sequence generation over a fixed vocabulary of CAD commands — an explicit analogy to natural-language generation. [Source](https://arxiv.org/abs/2105.09492) The vocabulary is deliberately small: six command types — `Line`, `Arc`, `Circle`, `SOL` (start-of-loop), `Extrude`, and `EOS` (end-of-sequence) — each carrying quantized parameters (endpoint coordinates for a line; sweep angle and direction flag for an arc; center and radius for a circle; sketch-plane orientation, origin, scale, extrude distances, Boolean type, and direction for an extrude). [Source](https://arxiv.org/abs/2105.09492) A Transformer autoencoder learns to reconstruct and randomly sample sequences in this vocabulary, demonstrating both shape autoencoding and unconditional random shape generation. [Source](https://arxiv.org/abs/2105.09492) Because every generated sequence is, by construction, a valid sketch-and-extrude history, DeepCAD's outputs are trivially replayable in a real CAD kernel — the tradeoff is that the fixed six-command vocabulary caps expressiveness at prismatic, sketch-extrude geometry; it cannot represent a filleted edge, a lofted surface, or a swept profile.

**Command parametrization.** Each command carries a different subset of a shared 16-slot parameter vector — a design the reference implementation encodes directly as a fixed-size argument mask rather than a variable-length schema per command:

| Command | Parameters |
|---|---|
| `⟨SOL⟩` | none (marks the start of a new sketch loop) |
| `Line` | `x, y` — line endpoint |
| `Arc` | `x, y, α, f` — endpoint, sweep angle, counter-clockwise flag |
| `Circle` | `x, y, r` — center, radius |
| `Extrude` | `θ, φ, γ` (sketch-plane orientation) + `px, py, pz, s` (plane origin, sketch scale) + `e1, e2, b, u` (extrude distances, Boolean type, extent type) |
| `⟨EOS⟩` | none (marks end of sequence) |

[Source](https://arxiv.org/abs/2105.09492) The reference implementation's `cadlib/macro.py` defines this same 16-slot layout and the per-command argument mask that zeroes out the slots a given command doesn't use:

```python
# DeepCAD/cadlib/macro.py:16-22,27-32 (github.com/ChrisWu1997/DeepCAD)
# (lines 23-26, the SOL_VEC/EOS_VEC padding vectors, elided)
PAD_VAL = -1
N_ARGS_SKETCH = 5  # sketch parameters: x, y, alpha, f, r
N_ARGS_PLANE = 3   # sketch plane orientation: theta, phi, gamma
N_ARGS_TRANS = 4   # sketch plane origin + sketch bbox size: p_x, p_y, p_z, s
N_ARGS_EXT_PARAM = 4  # extrusion parameters: e1, e2, b, u
N_ARGS_EXT = N_ARGS_PLANE + N_ARGS_TRANS + N_ARGS_EXT_PARAM
N_ARGS = N_ARGS_SKETCH + N_ARGS_EXT

CMD_ARGS_MASK = np.array([[1, 1, 0, 0, 0, *[0]*N_ARGS_EXT],  # line
                          [1, 1, 1, 1, 0, *[0]*N_ARGS_EXT],  # arc
                          [1, 1, 0, 0, 1, *[0]*N_ARGS_EXT],  # circle
                          [0, 0, 0, 0, 0, *[0]*N_ARGS_EXT],  # EOS
                          [0, 0, 0, 0, 0, *[0]*N_ARGS_EXT],  # SOL
                          [*[0]*N_ARGS_SKETCH, *[1]*N_ARGS_EXT]])  # Extrude
```

Every continuous parameter — coordinates, angles, distances — is normalized into a bounded range and then uniformly quantized to 256 levels, so the entire command sequence becomes a sequence of discrete tokens a Transformer can predict with a softmax over a fixed vocabulary rather than regress as continuous values. [Source](https://arxiv.org/abs/2105.09492) Training minimizes a loss that sums a cross-entropy term over the predicted command type and a separately-weighted cross-entropy term over each command's quantized parameters:

$$\mathcal{L} = \sum_{i=1}^{N_c} \ell(\hat{t}_i, t_i) \;+\; \beta \sum_{i=1}^{N_c} \sum_{j=1}^{N_p} \ell(\hat{p}_{i,j}, p_{i,j})$$

with $\beta = 2$ weighting the parameter loss against the command-type loss, summed over $N_c$ commands and $N_p = 16$ parameter slots per command. [Source](https://arxiv.org/abs/2105.09492) A parameter is scored correct at evaluation time if it falls within a tolerance of $\eta = 3$ quantization levels (out of 256) of the ground truth — the accuracy metric later benchmark papers in §9 inherit. [Source](https://arxiv.org/abs/2105.09492)

### 3.2 SkexGen: Disentangled Codebooks

SkexGen (ICML 2022) targets the same sketch-and-extrude command-sequence space as DeepCAD but restructures the generative model around three separate Transformer-encoded **codebooks** — disentangling a CAD sequence's topology, geometry, and extrusion variation into independently sampleable latent codes rather than one entangled latent vector. [Source](https://arxiv.org/abs/2207.04632) Autoregressive Transformer decoders then generate sequences conditioned on combinations of codebook vectors, which gives a user (or a downstream search process) explicit control: hold the topology code fixed and vary geometry to get shapes with the same construction structure but different proportions, or mix codes from two different models to explore a design space between them. [Source](https://arxiv.org/abs/2207.04632) This disentanglement — separating *what kind of shape this is* from *what its specific dimensions are* — is a recurring idea in later CAD-generation work, echoing the parametric-vs-instance distinction Chapter 176 §9.1's CodeCAD discussion draws for hand-written code-CAD models.

**Codebook mechanics.** Each codebook is a discrete dictionary of learned vectors — 500 entries for topology, 1,000 each for geometry and extrusion — and an encoder output is mapped onto the codebook by nearest-neighbor lookup in Euclidean distance, the same vector-quantization step VQ-VAE popularized for discrete image tokenization. [Source](https://arxiv.org/abs/2207.04632) Because that nearest-neighbor lookup is non-differentiable, gradients are passed straight through the quantization step, and training combines three terms — sequence reconstruction, a codebook loss that pulls the dictionary toward encoder outputs, and a commitment loss weighted by $\beta = 0.25$ that pulls the encoder toward the dictionary instead of drifting arbitrarily:

$$\mathcal{L} = \sum_K \mathrm{CE}(h^{out}_K, h^{gt}_K) \;+\; \lVert \mathrm{sg}[Z^e] - b \rVert_2^2 \;+\; \beta \lVert Z^e - \mathrm{sg}[b] \rVert_2^2$$

where $\mathrm{sg}[\cdot]$ denotes a stop-gradient. [Source](https://arxiv.org/abs/2207.04632) The autoregressive decoders are trained with teacher forcing — conditioned on ground-truth prior tokens rather than their own predictions — which is standard practice for sequence Transformers but means SkexGen's reported sample quality, like DeepCAD's, is measured under exposure to ground truth at training time that is unavailable at generation time. [Source](https://arxiv.org/abs/2207.04632)

### 3.3 SketchGen and Vitruvion: Constrained Sketches

Every model in §3.1-§3.2 generates *unconstrained* curves: a DeepCAD or SkexGen line or arc is a set of coordinates, with no explicit record that two edges are meant to be parallel, tangent, or share an endpoint exactly. A real parametric sketch in Fusion 360 or SolidWorks is not just geometry — it is geometry plus a **constraint graph** (coincident points, parallel or perpendicular lines, tangency, symmetry, equal-length relations) that is what makes the sketch re-editable: drag one dimension and the constraint solver keeps every relation intact. SketchGen (NeurIPS 2021) targets this gap directly, using a Transformer to autoregressively generate both primitives and the constraints between them in one sequential language, with primitive and constraint types distinguished by tagged tokens so the model can share parameters across related entities; the generated primitive-plus-constraint graph can then be handed to a real geometric constraint solver for regularization, rather than being accepted as-is. [Source](https://arxiv.org/abs/2106.02711) Vitruvion (ICLR 2022) takes a similar two-stage autoregressive approach — sample primitives, then sample constraints conditioned on them — and additionally conditions generation on a raster image of a hand-drawn sketch, targeting autocompletion and constraint inference for an already-partially-specified design rather than only unconditional generation. [Source](https://arxiv.org/abs/2109.14124) Neither model is evaluated in this chapter's later benchmarks (§9), but the capability they add — an explicit, solver-checkable constraint graph — is precisely the property command-sequence models without constraints lack, and directly informs the editability leg of the three-way tradeoff §11 returns to.

## 4. Cutting-Edge B-Rep Frameworks: Generation and Representation Learning

The models in §3 all generate a *construction history* — a script that, when replayed, produces geometry. A separate and more recent line of work instead operates on the **B-rep itself**, either generating it directly (§4.1) or learning general-purpose embeddings of it for retrieval and evaluation (§4.2) — treating boundary representation the way image and language models treat pixels and tokens, as a native substrate for representation learning rather than an output format to be reached only via a command sequence.

### 4.1 BrepGen and BrepDiff: Diffusion-Based Direct Generation

BrepGen generates a B-rep as a diffusion process over a structured latent representation: a hierarchical tree whose root represents the whole solid and whose descendants represent each face, edge, and vertex, with geometry attached to each node as a bounding box plus a local shape latent code — a face node carries a bounding box $F_p \in \mathbb{R}^6$ and a $4{\times}4{\times}3$ latent shape grid $F_z$, an edge node carries its own bounding box plus a joint edge-vertex latent $E_{zv}$, and a vertex node carries raw $(x,y,z)$ coordinates. [Source](https://arxiv.org/abs/2401.15563) Each stage follows the standard DDPM forward process — Gaussian noise added over $T$ steps, $q(\mathbf{x}_t \mid \mathbf{x}_0) = \mathcal{N}(\mathbf{x}_t; \sqrt{\bar\alpha_t}\,\mathbf{x}_0, (1-\bar\alpha_t)\mathbf{I})$ with $\bar\alpha_t = \prod_{i=1}^t (1-\beta_i)$ under a linear noise schedule — and a network trained to predict the injected noise under an L2 regression loss, $\mathcal{L} = \mathbb{E}_{t,\mathbf{x}_0,\epsilon_t}\big[\lVert \epsilon_t - \epsilon_\theta(\sqrt{\bar\alpha_t}\mathbf{x}_0 + \sqrt{1-\bar\alpha_t}\epsilon_t,\, t) \rVert^2\big]$, exactly as in DDPM. [Source](https://arxiv.org/abs/2401.15563) What makes this a *cascade* rather than one diffusion run is the factorization the four Transformer-based denoisers are trained to respect — face position first, then face geometry conditioned on it, then edge position conditioned on the finished faces, then joint edge-vertex geometry conditioned on edge position: $p(F,E,V) = p(E_{zv} \mid E_p, F)\,p(E_p \mid F)\,p(F_z \mid F_p)\,p(F_p \mid \varnothing)$. [Source](https://arxiv.org/abs/2401.15563) At inference, the face- and edge-position denoisers run full DDPM sampling ($T=250$ to $0$) for precise bounding boxes while the two geometry denoisers use the faster PNDM sampler (200 forward passes), until the result is a watertight solid. [Source](https://arxiv.org/abs/2401.15563) The significance is in what this unlocks versus the command-sequence family: prior CAD-generation methods were limited to simple prismatic shapes (the sketch-extrude vocabulary's ceiling), while BrepGen is presented as the first CAD-generative model to produce free-form and doubly curved surfaces — geometry no fixed sketch-and-extrude command set can express — validated in part on a newly collected furniture dataset chosen to showcase non-prismatic complexity. [Source](https://arxiv.org/abs/2401.15563) The cost is that BrepGen's outputs are geometry, not an editable feature history — a diffusion-generated B-rep does not carry the "which sketch, which extrude" provenance a command-sequence model's output does, which matters for a downstream user who wants to *edit* the result rather than just consume the shape.

BrepDiff (SIGGRAPH 2025) targets the same direct-B-rep-diffusion problem but collapses BrepGen's cascade of dependent stages into a single-stage diffusion transformer, denoising a **masked UV-grid representation** — structured point samples across each face, carried with per-sample visibility masks — as its token set, rather than denoising topology and per-entity geometry as separate cascaded stages. [Source](https://brepdiff.github.io/) An asynchronous, shifted noise schedule lets the model learn a good distribution over plausible UV grids without ever explicitly encoding topology; a valid solid B-rep is instead recovered afterward by a post-processing pass that extends faces and resolves their intersections. [Source](https://brepdiff.github.io/) [Source](https://dl.acm.org/doi/10.1145/3721238.3730698) Because the UV-grid representation is explicit rather than buried inside a multi-stage latent pipeline, BrepDiff reports supporting direct geometry manipulation — shape autocompletion, merging two B-reps, and interpolating between them — operations BrepGen's cascaded, topology-first pipeline makes harder to expose. [Source](https://brepdiff.github.io/)

### 4.2 BRepCLIP and BRep-BERT: B-Rep Representation Learning

Generation is not the only task a B-rep foundation model can target: BRepCLIP and BRep-BERT instead learn general-purpose B-rep *embeddings*, useful for retrieval, classification, and evaluation rather than generation itself.

BRepCLIP tokenizes a CAD model as a sequence of face and edge tokens, drawn from separate discrete vocabularies for surface geometry (planar, cylindrical, toroidal, NURBS, ...) and curve geometry (line, arc, B-spline, ...) augmented with spatial descriptors, and aggregates these tokens with a Transformer encoder into a single global B-rep embedding aligned — via a CLIP-style contrastive objective — with the text and image embedding spaces of a pretrained CLIP model. [Source](https://arxiv.org/abs/2606.05515) Concretely, the vocabularies come from two separately-trained discrete autoencoders — a face dVAE with an 8,192-entry codebook and an edge dVAE with a 2,048-entry codebook, each with 256-dimensional latents, trained with a reconstruction-plus-regularization loss combining Chamfer distance against the sampled surface/curve and a KL term. [Source](https://arxiv.org/abs/2606.05515) The alignment objective is a symmetric InfoNCE loss over $\ell_2$-normalized embeddings with a learnable temperature $\tau$:

$$\mathcal{L}_{bt} = -\frac{1}{2N}\sum_i \left[\log\frac{\exp(Z_i^B \cdot Z_i^T / \tau)}{\sum_j \exp(Z_i^B \cdot Z_j^T / \tau)} + \log\frac{\exp(Z_i^T \cdot Z_i^B / \tau)}{\sum_j \exp(Z_i^T \cdot Z_j^B / \tau)}\right]$$

with an identically-structured $\mathcal{L}_{bi}$ for the image branch and a total objective $\mathcal{L} = \mathcal{L}_{bt} + \mathcal{L}_{bi}$ — the same batch-contrastive formulation CLIP itself uses, just with a B-rep encoder standing in for CLIP's image tower. [Source](https://arxiv.org/abs/2606.05515) On text-to-CAD retrieval the authors report BRepCLIP outperforming point-cloud-based baselines by roughly 40%, 22%, and 24% on the ABC, CADParser, and Automate benchmarks respectively, and on zero-shot classification it transfers to the unseen FabWave dataset without any fine-tuning, again ahead of point-cloud counterparts. [Source](https://arxiv.org/abs/2606.05515) The paper's second contribution, **BRepCLIP-Score**, is arguably the more consequential one for this chapter's §9: a geometry-aware similarity metric for scoring text- or image-conditioned CAD generation, defined simply as $\mathrm{BRepCLIP\text{-}Score}(t, x) = \cos\big(f_{\mathrm{text}}(t),\, f_{\mathrm{3D}}(x)\big)$ — cosine similarity in the same aligned embedding space the contrastive loss above trained — that the authors report correlates with human expert judgment more reliably than either raw CLIP score or Chamfer distance, both of which operate on rendered pixels or sampled points rather than on B-rep structure directly. [Source](https://arxiv.org/abs/2606.05515)

BRep-BERT applies the same masked-pretraining idea BERT popularized for text directly to B-rep graphs: a GNN tokenizer first assigns each B-rep entity (face, edge, vertex) a discrete token carrying its geometric and structural semantics, an entity sequence is then built from the graph's structural relationships, and the model is pretrained via Masked Entity Modeling — predicting a proportion of masked entity tokens from their surrounding context, exactly as BERT predicts masked words. [Source](https://dl.acm.org/doi/10.1145/3583780.3614795) A learnable relative-position encoding folded into the attention module addresses the attention-sparsity problem large B-rep graphs otherwise create. [Source](https://dl.acm.org/doi/10.1145/3583780.3614795) The motivation given for pretraining on this scale is the same one that motivates masked language-model pretraining generally: labeled B-rep datasets for any single downstream task are small, so a self-supervised objective on unlabeled B-rep graphs lets the model learn general structural priors before fine-tuning on whatever specific, label-scarce task it will be evaluated on. [Source](https://dl.acm.org/doi/10.1145/3583780.3614795)

Read together, §4.1 and §4.2 mark a genuine shift from the rest of this chapter: every other model surveyed here treats a CAD-command sequence as the interchange format between language and geometry. BrepDiff, BRepCLIP, and BRep-BERT treat the B-rep graph itself as that interchange format — closer in spirit to how vision and language foundation models treat pixels and tokens as a native substrate, rather than routing everything through an intermediate scripting language.

## 5. Text-to-CAD: Two Competing Philosophies

Once a base representation and dataset exist, the natural next step is conditioning generation on a user's text prompt — and the field has settled into two genuinely different strategies for doing so, which do not compete on the same axis so much as trade one constraint for another.

### 5.1 Generate the Command Sequence Directly

Text2CAD (NeurIPS 2024 Spotlight) trains an end-to-end Transformer to autoregressively generate a full DeepCAD-style construction-command sequence directly from a natural-language prompt, spanning prompts from beginner-level abstract descriptions ("generate two concentric cylinders") to expert-level fully dimensioned instructions. [Source](https://sadilkhan.github.io/text2cad-project/) [Source](https://arxiv.org/abs/2409.17106) Its main contribution is arguably not the generation model itself but the **annotation pipeline** that produced training data for it: because no large corpus of (CAD model, text description) pairs existed, the authors used open-source LLMs (Mistral) and VLMs (LLaVA-NeXT) to auto-annotate the existing DeepCAD dataset with multi-level text prompts, producing roughly 660K prompts across 170K models. [Source](https://arxiv.org/abs/2409.17106) This is the same strategy — generate a fixed-vocabulary command sequence directly — as DeepCAD and SkexGen, just now conditioned on language instead of sampled unconditionally.

Related work in this same family includes CAD-Llama, which fine-tunes an LLM to emit parametric 3D CAD construction sequences directly rather than free-text code [Source](https://arxiv.org/abs/2505.04481), and CAD-GPT, which frames CAD-sequence synthesis as a spatial-reasoning task for a multimodal LLM. [Source](https://arxiv.org/pdf/2412.19663) Every model in this sub-family shares command-sequence generation's structural guarantee (the output is always a valid, replayable sketch-extrude history) and its structural ceiling (no fillets, lofts, or sweeps beyond what the fixed command vocabulary encodes).

### 5.2 Generate Code in an Existing CAD Language

The alternative strategy, already covered for the OCCT ecosystem specifically in Chapter 176 §9.6, sidesteps the fixed-vocabulary ceiling entirely: instead of generating tokens in a bespoke CAD-command language, have the model write source code in an existing, general-purpose code-CAD language — CadQuery Python being the dominant target — and execute that code against a real kernel to obtain the geometry. Text-to-CadQuery fine-tunes LLMs on 170K paired (text description, CadQuery script) examples, with output quality improving consistently as model scale increases. [Source](https://arxiv.org/abs/2505.06507) CADmium takes the same code-generation strategy but fine-tunes a code-specialized language model specifically for text-driven sequential CAD design rather than a general LLM. [Source](https://arxiv.org/html/2507.09792v3)

Concretely, this family's *output* looks like an ordinary human-written CadQuery script, chaining a fluent `Workplane` API rather than emitting a token per geometric parameter — this is the shape of program the code-generation strategy targets, though a given model (§6.2, §6.3) may be trained to emit only a restricted subset of the full fluent API rather than every operation shown below:

```python
# cadquery/examples/Ex003_Pillow_Block_With_Counterbored_Holes.py:28-41
# (github.com/CadQuery/cadquery)
result = (
    cq.Workplane("XY")
    .box(length, width, thickness)
    .faces(">Z")
    .workplane()
    .hole(center_hole_dia)
    .faces(">Z")
    .workplane()
    .rect(length - cbore_inset, width - cbore_inset, forConstruction=True)
    .vertices()
    .cboreHole(cbore_hole_diameter, cbore_diameter, cbore_depth)
    .edges("|Z")
    .fillet(2.0)
)
```

Every operation here — `.hole()`, `.cboreHole()`, `.fillet()` — has no equivalent token in DeepCAD's six-command vocabulary (§3.1); this is precisely the expressiveness the code-generation strategy buys, at the cost that a generated script this long has many more places to fail to parse or execute than a fixed-length command sequence does.

This code-generation strategy trades DeepCAD-family's guaranteed-valid-but-limited output for the full expressiveness of whatever CAD-scripting language it targets (fillets, lofts, sweeps, and every other operation the underlying kernel exposes) — at the cost of validity no longer being guaranteed by construction: generated code can fail to parse, fail to execute, or execute into a non-manifold or non-watertight solid, which is precisely the failure mode the evaluation methodologies in §9 and the tool-verification pattern in §8 exist to catch.

## 6. Reverse Engineering: Point Clouds, Meshes, and Images to CAD

A distinct but related task takes an *existing* 3D observation — a scanned point cloud, a mesh, a photograph — and reconstructs a parametric or B-rep CAD model that reproduces it, rather than generating a novel shape from a text description.

### 6.1 ComplexGen: B-Rep as a Chain Complex

ComplexGen reframes B-rep reconstruction from a point cloud as detecting geometric primitives of three different orders — vertices, edges, and surface patches — together with the correspondence relationships between them, modeling the whole structure holistically as a **chain complex** rather than reconstructing each primitive type independently. [Source](https://arxiv.org/abs/2205.14573) Formally, a B-rep is a graded structure $\mathcal{C} = (V, E, F, \partial, \mathcal{P})$ — 0th-order vertices $V$, 1st-order curved edges $E$, 2nd-order curved faces $F$, geometric primitives $\mathcal{P}$ attached to each, and boundary operators $\partial_2 : F \to E$ (each face's bounding edges) and $\partial_1 : E \to V$ (each edge's endpoint vertices), forming the chain

$$F \xrightarrow{\partial_2} E \xrightarrow{\partial_1} V$$

subject to $\partial_1 \circ \partial_2 = 0$ — the algebraic-topology condition that a face's edges must close into loops, since applying $\partial_2$ then $\partial_1$ to a face must return to where it started rather than land on a dangling vertex. [Source](https://arxiv.org/abs/2205.14573) Validity is checked with counting constraints derived from this structure: every edge borders exactly two faces, every open (closed) edge has exactly two (zero) endpoints, and the face/edge/vertex incidence counts satisfy $\mathbf{FE} \times \mathbf{EV} = 2\,\mathbf{FV}$. [Source](https://arxiv.org/abs/2205.14573) The pipeline is two-stage: a sparse-CNN point-cloud encoder paired with a tri-path Transformer decoder first predicts primitives and their probabilistic mutual relationships, and then a global optimization pass recovers a definite, structurally valid B-rep chain complex by maximizing likelihood under these validity constraints and applying geometric refinement. [Source](https://arxiv.org/abs/2205.14573) The chain-complex framing is the paper's central claim: modeling vertex/edge/face relationships jointly, rather than per-primitive, produces more complete and topologically consistent reconstructions than prior per-primitive-independent methods.

### 6.2 CAD-Recode: Point Clouds to Executable Code

CAD-Recode takes the code-generation philosophy from §5.2 and applies it to reverse engineering: rather than predicting geometric primitives directly, it translates a point cloud into Python code that, when executed, reconstructs the CAD model — combining a relatively small pretrained LLM decoder with a lightweight point-cloud projector that feeds point-cloud features into the LLM's input space. [Source](https://arxiv.org/abs/2412.14042) The authors report the approach significantly outperforms prior point-cloud-to-CAD methods across the DeepCAD, Fusion360, and a real-world CC3D dataset. [Source](https://arxiv.org/abs/2412.14042) The architectural choice — small LLM plus point-cloud projector, rather than a purpose-built point-cloud-to-sequence network — mirrors vision-language model design generally, and reflects the same code-target strategy CAD-Coder (below) applies to images instead of point clouds.

### 6.3 CAD-Coder: Images to Executable Code

CAD-Coder is an open-source vision-language model fine-tuned specifically to generate editable CadQuery Python code directly from an image, rather than from a point cloud or text prompt. [Source](https://arxiv.org/abs/2505.14646) It was trained on GenCAD-Code, a purpose-built dataset of over 163K paired CAD-model images and CadQuery code, and is reported to outperform general-purpose VLM baselines (including GPT-4.5 and Qwen2.5-VL-72B) on syntax validity and 3D solid similarity, reaching a 100% valid-syntax rate on its evaluation set. [Source](https://arxiv.org/abs/2505.14646) The authors also report signs of generalization beyond the training distribution: the model successfully generates CAD code from real-world photographs and reproduces CAD operations not seen during fine-tuning. [Source](https://arxiv.org/abs/2505.14646) Img2CADSeq extends the image-conditioning idea to sequence-based diffusion rather than direct code generation, another point on the same representation axis this chapter keeps returning to. [Source](https://arxiv.org/pdf/2605.13293)

### 6.4 Point2Cyl and CSGNet: Differentiable and Program-Synthesis Primitive Fitting

§6.1-§6.3 cover two reverse-engineering strategies — holistic chain-complex detection and LLM code generation — but a third, older family fits an explicit *program* to the input geometry without training an LLM at all. Point2Cyl (CVPR 2022) targets CAD's extrusion-cylinder structure directly: a point cloud is first segmented per-point into base/barrel labels and normals with a supervised network, and the extrusion parameters (axis, sketch profile, distance) are then recovered from that segmentation via closed-form, differentiable formulas rather than a second learned decoder — so the geometric estimation step is exact given a correct segmentation, not merely learned to approximate it. [Source](https://arxiv.org/abs/2112.09329) CSGNet is the older program-synthesis end of this same family: a CNN encoder and RNN decoder parse a 2D or 3D shape top-down into a Constructive Solid Geometry program — a tree of Boolean operations over primitives — and, notably, can be trained via policy-gradient reinforcement learning (REINFORCE) directly against a rendering-similarity reward when no ground-truth program annotations exist at all, since a CSG program's discrete structure is not differentiable end-to-end. [Source](https://arxiv.org/abs/1712.08290) Both stand apart from §6.2-§6.3's LLM-code and §6.1's chain-complex strategies in the same way: correctness comes from geometric or combinatorial search grounded in the input, not from a language model's learned prior over what CAD code usually looks like — a useful check on the code-generation family's failure mode of producing code that is syntactically fluent but geometrically wrong.

## 7. Multimodal and Unified Models: CAD-MLLM

Rather than building a separate model per input modality (text, image, point cloud), CAD-MLLM unifies all three into a single multimodal LLM that generates a CAD command sequence conditioned on any one of them, or a combination. [Source](https://arxiv.org/abs/2411.04954) It aligns the feature space of diverse input modalities with CAD command-sequence representations inside an LLM, and was trained and evaluated on a purpose-built dataset, Omni-CAD, containing roughly 450K instances — each paired with a text description, multi-view images, points, and its CAD construction sequence. [Source](https://arxiv.org/abs/2411.04954) To evaluate topology quality and surface-enclosure extent (properties generic image/mesh similarity metrics do not capture), the authors introduce dedicated metrics rather than relying solely on shape-similarity scores borrowed from mesh-generation research. [Source](https://arxiv.org/abs/2411.04954) CAD-MLLM's significance is less any single-modality result and more the demonstration that a command-sequence CAD-generation target (§3) can share one model and one training run across text, image, and point-cloud conditioning — collapsing what would otherwise be three separate research lines (§5, §6) into one architecture.

## 8. Agentic and Tool-Verified Design

Every model surveyed so far is fundamentally one-shot: prompt in, CAD output out, with no verification step in between. A separate line of work instead wraps a generation model in an agent loop that inspects its own output and iterates — the same shift Chapter 176 §9.6 documents for MCP-based tooling around CadQuery, build123d, and FreeCAD, where wrapping a generation model with tool calls that render, measure, and validate geometry measurably improves benchmark scores over blind one-shot code generation.

CADDesigner is an LLM-powered agent for *conceptual* CAD design that accepts both text descriptions and hand-drawn sketches, engages in clarifying dialogue with the user before committing to a design, and uses iterative visual feedback during model creation — rendering intermediate results and checking them before proceeding — plus a structured knowledge base of completed designs that the agent can draw on for future requests. [Source](https://arxiv.org/abs/2508.01031) "From Idea to CAD" takes a more explicitly multi-agent structure, mirroring an industrial design team: a Requirements Engineering agent handles initial specification, a CAD Engineering agent performs the actual modeling, and a Vision-Based Quality Assurance agent validates the result, iterating with user feedback in a loop. [Source](https://arxiv.org/abs/2503.04417) "Clarify Before You Draw" targets a specific, common failure mode of one-shot text-to-CAD systems directly: an underspecified or contradictory prompt has many valid interpretations, and a one-shot model must guess. Its proposed system, **ProCAD**, pairs a clarifying agent that audits the prompt and asks a targeted follow-up question only when necessary with a separate coding agent that turns the (now-disambiguated) specification into executable CadQuery code — reported to cut invalid-output rate from 4.8% to 0.9% against a Claude Sonnet baseline. [Source](https://arxiv.org/abs/2602.03045)

The common thread across all three: CAD-generation quality gates on validity and design intent in ways image generation does not, so an agent that can render its own output, measure it, or ask a clarifying question closes a feedback loop a pure one-shot generator cannot — the same lesson Chapter 176 §9.6 draws from `build123d-mcp`'s measured effect on CADGenBench scores.

## 9. Benchmarks and the Evaluation Problem

The datasets in §2 were built to *train* models; a separate and more recent wave of work exists specifically to *evaluate* them, and its own history is instructive: early evaluation was largely ad hoc geometric-similarity comparison on whatever subset of DeepCAD or Fusion360 a given paper happened to hold out, which made cross-paper comparison close to impossible. Chapter 176 §9.6 already covers two of the resulting standardized benchmarks in the OCCT/code-generation context — CADGenBench (geometric accuracy, topology correctness, and CAD validity, with a mandatory watertight/manifold check) and Text2CAD-Bench (600 human-curated parametric-CAD examples across four complexity levels) — this section covers two more general benchmarks that evaluate across representations and modalities rather than a single toolchain.

### 9.1 CADBench: Fragmentation Made Comparable

CADBench addresses exactly the fragmentation problem above directly: existing CAD-generation evaluation was scattered across different datasets, input modalities, and metrics, making direct comparison between published methods difficult even when the underlying task was the same. [Source](https://arxiv.org/abs/2605.10873) It assembles 18,000 evaluation samples spanning six benchmark families derived from DeepCAD, Fusion 360, ABC, MCB, and Objaverse, across five input modalities (clean meshes, noisy meshes, single-view renders, photorealistic renders, and multi-view renders), scored on six metrics spanning three categories:

| Category | Metric | Definition |
|---|---|---|
| Geometric fidelity | Volumetric IoU | $\lvert V(\hat S) \cap V(S) \rvert / \lvert V(\hat S) \cup V(S) \rvert$ — voxelized-occupancy overlap |
| Geometric fidelity | Chamfer Distance | $\frac{1}{\lvert \hat P \rvert}\sum \min \lVert x - y \rVert^2 + \frac{1}{\lvert P \rvert}\sum \min \lVert y - x \rVert^2$ — bidirectional point-to-surface error |
| Geometric fidelity | Surface IoU | thresholded bidirectional surface-point coverage (matched within 1% of the bounding-box diagonal) |
| Executability | Valid Shape Rate | fraction of generated programs that execute and yield a valid solid |
| Program compactness | Token Count | number of generated tokens |
| Program compactness | Operation Count | number of CAD operations in the generated program |

[Source](https://arxiv.org/abs/2605.10873) Running eleven CAD-specialized and general-purpose vision-language systems through this unified harness (over 1.4 million generated CAD programs in total) produced a finding that cuts against the code-generation optimism of §5.2 and §6.3: under idealized clean-input conditions, specialized mesh-to-CAD reconstruction models substantially outperform code-generating VLMs, which remain far from reliable CAD-program reconstruction, with quality degrading further as geometric complexity increases and model rankings shifting depending on which of the six metrics is applied. [Source](https://arxiv.org/abs/2605.10873)

### 9.2 MUSE: Beyond Geometric Similarity

MUSE argues that geometric similarity — however it is measured — is the wrong ceiling for evaluating industrial text-to-CAD, and instead evaluates three engineering-specific dimensions: whether a generated design is **manufacturable**, **functional**, and **assemblable**. [Source](https://arxiv.org/abs/2605.28579) Its evaluation protocol is staged rather than a single score, and each stage is a hard gate on the next. A **Code Check** executes the generated CadQuery script in a sandbox and confirms it constructs a solid and exports a valid STEP file — anything that fails to execute scores zero on every later stage. A **Geometric Check** then runs four binary tests against the exported STEP geometry: watertight (no open boundaries or naked edges), manifold (valid manifold topology throughout), self-intersection-free, and overlap-free (no interpenetrating solid components) — failing any one of the four blocks advancement. Only geometry that clears all four reaches **Design-Intent Alignment**, where a rubric-based VLM judge (validated against human annotation) scores six binary sub-criteria grouped into three pillars — functional and robust (functionality); well-toleranced and manufacturable (manufacturability); assembly-ready and connectable (assemblability) — with the final score the mean across the three pillars. [Source](https://arxiv.org/abs/2605.28579) The paper's central empirical finding is a **failure cascade**: models that reliably clear the Code Check fail at markedly higher rates once evaluated for valid geometry, and fail at higher rates still once evaluated against engineering-ready design specifications — meaning most of the "success rate" numbers models report against weaker benchmarks describe executability, not usable engineering output. [Source](https://arxiv.org/abs/2605.28579)

## 10. Commercial Landscape

Chapter 176 §9.6 already covers the OCCT-adjacent AI ecosystem specifically — Text-to-CadQuery-style code generation, and MCP servers (`build123d-mcp`, `cadquery-mcp-server`, FreeCAD MCP variants) that wrap existing OCCT-based scripting APIs as agent tools — plus Zoo (formerly KittyCAD) as the deliberate counterexample: a commercial Text-to-CAD product built on its own GPU-native geometry engine and a custom scripting language (KCL), not on OCCT at all. That split — wrap an existing kernel's scripting surface with agent tooling, versus build (or train) a foundation model against a proprietary geometry engine — recurs at the largest commercial CAD vendors. Autodesk is developing an agentic assistant across Fusion and its other products, with a stated roadmap goal of generating **editable B-rep geometry** from a single prompt using Autodesk's own foundation models for manufacturing — output a user can continue editing as ordinary parametric CAD, not a static mesh. [Source](https://adsknews.autodesk.com/en/news/new-investments-in-fusion-bring-ai-powered-transformation-to-manufacturing/) The distinction Autodesk draws in its own framing — prompt-to-mesh versus prompt-to-*editable BRep* — is precisely the validity/editability boundary §1 opens this chapter with: a system that cannot hand back an editable feature history has not, by the standard this chapter's research literature applies, solved CAD generation, however good the resulting shape looks.

## 11. Open Problems

Three limitations recur across nearly every system this chapter covers, and are worth stating plainly rather than leaving implicit in each section's caveats:

**Data scarcity for real engineering parts.** Every dataset in §2 traces back to either ABC (scraped public CAD repositories, not curated for engineering realism) or a few thousand Fusion 360 hobbyist/consumer design histories. None of the datasets this chapter surveys are drawn from tolerance-critical aerospace, medical, or precision-manufacturing parts, and MUSE's failure-cascade result (§9.2) suggests the gap between "executes" and "manufacturable" is exactly where that data scarcity shows up.

**The representation ceiling is still unresolved.** Command-sequence models (§3, §5.1) guarantee valid, editable output but cap expressiveness at their fixed vocabulary; code-generation models (§5.2, §6.2, §6.3) gain full kernel expressiveness but lose the validity guarantee; diffusion-over-B-rep models (§4.1) gain free-form geometry but lose feature-history editability. No approach surveyed here has all three properties (guaranteed valid, fully expressive, and editable) simultaneously — a system architect choosing among them today is choosing which of the three to give up. Constraint-graph sketch models (§3.3) narrow this gap only at the sketch level, not the full-solid level: SketchGen and Vitruvion make a *2D sketch* re-editable via a solver-checkable constraint graph, but nothing surveyed in this chapter extends that same explicit-constraint property to a full 3D B-rep or feature history.

**Evaluation is still immature relative to the pace of new models.** CADBench and MUSE (§9) are recent enough that most of the generative models in §3-§8 predate them and were never evaluated against their stricter, multi-stage protocols — meaning many of this chapter's headline results were measured against benchmarks the field's own subsequent research has since argued undercount real failure modes.

---

## Integrations

- **Ch176 (OpenCASCADE Technology):** §9.6's OCCT-specific AI-CAD coverage (Text-to-CadQuery code generation, CADGenBench, Text2CAD-Bench, and the `build123d-mcp`/`cadquery-mcp-server`/FreeCAD MCP servers) is the OCCT-toolchain-specific instance of the representation-level landscape this chapter surveys; §8's agentic/tool-verified pattern here is the same one §9.6 documents concretely via `build123d-mcp`'s measured score improvement on CADGenBench.

- **Ch124 (Local LLM Inference on Linux):** Every model in this chapter that runs as an inference-time LLM or VLM (Text2CAD, CAD-Recode, CAD-Coder, CADDesigner, and the agentic systems in §8) is model-agnostic at the inference layer — nothing about the CAD-generation task requires a cloud-hosted model, and a locally served model along the lines Ch124 describes is a valid backend for any of the open-weights systems this chapter covers.

- **Ch205 (AI-Driven 3D Creation — Blender MCP, Claude Code, and Generative Tools):** Ch205's Blender MCP bridge and this chapter's agentic CAD systems (§8) share the same underlying pattern — wrap an existing scripting API (`bpy` vs. CadQuery/build123d/FreeCAD) behind agent-callable tools rather than training a model to emit final geometry directly — applied to mesh-based DCC tooling in Ch205's case and parametric/B-rep CAD tooling here.

- **Ch40 (Bevy and wgpu) and Ch98 (WebAssembly and WebGPU as a Deployment Target):** Any of this chapter's generated or reconstructed CAD models still needs a rendering path once produced; §9.5's `opencascade.js`/CascadeStudio browser stack and §9.2's Rust-native kernels (both in Ch176) are the visualization layers a generative or reverse-engineering pipeline from this chapter would plausibly render through.
