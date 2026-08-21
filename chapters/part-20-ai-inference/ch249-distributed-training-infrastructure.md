# Chapter 249: Distributed Training Infrastructure — NCCL/RCCL, Parallelism Strategies, and GPU Data Loading

**Target audiences:** AI/ML systems engineers building or operating multi-node training
clusters on Linux GPUs; systems developers who own the InfiniBand/RoCE fabric and Kubernetes
scheduling layer beneath a training job; anyone who has read Chapter 246's account of how a
single compiled training step reaches a GPU and wants to know what happens once that step has
to run identically, in lockstep, across hundreds of GPUs. Chapter 246 §6 covered DDP, FSDP,
and GSPMD as *framework mechanisms* — hooks and compiler passes that emit collectives. This
chapter covers what those collectives run on top of: the communication library, the strategies
for deciding what to shard and when, the data pipeline that has to keep every GPU fed, and the
cluster infrastructure that makes a multi-node job survivable. Chapter 66 §18 already documents
NCCL's core API, ring/tree AllReduce, and per-communicator transport selection in detail; this
chapter does not restate that material — it starts where Chapter 66 §18 stops, at multi-node
topology and the parallelism strategies built on top of NCCL's primitives.

---

## Table of Contents

1. [Scope: What This Chapter Assumes](#1-scope-what-this-chapter-assumes)
2. [Collective Communication at Multi-Node Scale](#2-collective-communication-at-multi-node-scale)
   - [2.1 Double Binary Tree: NCCL's Latency-Optimized Algorithm](#21-double-binary-tree-nccls-latency-optimized-algorithm)
   - [2.2 Topology-Aware Selection: NCCL_TOPO_FILE and Multi-Node Rings](#22-topology-aware-selection-nccl_topo_file-and-multi-node-rings)
   - [2.3 RCCL at Multi-Node Scale](#23-rccl-at-multi-node-scale)
3. [Parallelism Strategies](#3-parallelism-strategies)
   - [3.1 Data Parallelism: Scaling DDP/FSDP Across Nodes](#31-data-parallelism-scaling-ddpfsdp-across-nodes)
   - [3.2 Tensor Parallelism: Intra-Layer Sharding](#32-tensor-parallelism-intra-layer-sharding)
   - [3.3 Pipeline Parallelism: Inter-Layer Sharding and the Bubble](#33-pipeline-parallelism-inter-layer-sharding-and-the-bubble)
   - [3.4 3D and Hybrid Parallelism: DeepSpeed ZeRO and Megatron-LM](#34-3d-and-hybrid-parallelism-deepspeed-zero-and-megatron-lm)
   - [3.5 JAX's Alternative: GSPMD and shard_map](#35-jaxs-alternative-gspmd-and-shard_map)
4. [GPU-Accelerated Data Loading](#4-gpu-accelerated-data-loading)
   - [4.1 The CPU Data-Loading Bottleneck](#41-the-cpu-data-loading-bottleneck)
   - [4.2 NVIDIA DALI: A GPU-Resident Pipeline](#42-nvidia-dali-a-gpu-resident-pipeline)
   - [4.3 PyTorch DataLoader: Worker-Process Prefetching](#43-pytorch-dataloader-worker-process-prefetching)
   - [4.4 JAX Input Pipelines: tf.data and Grain](#44-jax-input-pipelines-tfdata-and-grain)
5. [Cluster Topology and Scheduling](#5-cluster-topology-and-scheduling)
   - [5.1 InfiniBand and RoCE Fabric Considerations](#51-infiniband-and-roce-fabric-considerations)
   - [5.2 Kubernetes GPU Scheduling for Training Jobs](#52-kubernetes-gpu-scheduling-for-training-jobs)
   - [5.3 Elastic and Fault-Tolerant Restart](#53-elastic-and-fault-tolerant-restart)
   - [5.4 Checkpointing at Scale](#54-checkpointing-at-scale)
6. [Integrations](#6-integrations)
7. [Roadmap](#7-roadmap)

---

## 1. Scope: What This Chapter Assumes

This chapter assumes Chapter 66 §18's coverage of NCCL's core API (`ncclCommInitRank`,
`ncclAllReduce`/`ncclReduceScatter`/`ncclAllGather`, `ncclGroupStart`/`ncclGroupEnd`),
single-communicator ring- and tree-AllReduce mechanics, and per-communicator transport
selection (NVLink vs. PCIe vs. InfiniBand) as already covered — this chapter does not
re-derive any of it. It also assumes Chapter 246 §6's account of *how* PyTorch DDP/FSDP and
JAX GSPMD/`shard_map` turn framework-level sharding decisions into collective calls; this
chapter is about what happens once those collectives have to run across more than one node,
and about the strategic question DDP/FSDP/GSPMD leave open — *which* parallelism strategy, or
combination of strategies, a given model and cluster size actually calls for. It also assumes
the NVLink/NVSwitch peer-to-peer DMA primitives from Chapters 4 and 49, and the single-node
multi-GPU partitioning pattern already introduced for rendering workloads in Chapter 69.

## 2. Collective Communication at Multi-Node Scale

### 2.1 Double Binary Tree: NCCL's Latency-Optimized Algorithm

Chapter 66 §18.3 describes NCCL's tree-AllReduce as a latency-optimized alternative to the
ring algorithm for small tensors, without detailing its internal structure. The specific
algorithm NCCL uses, from NCCL 2.4 onward, is a **double binary tree**. A single binary tree
used for reduction is bandwidth-inefficient because roughly half of all ranks are leaves that
each handle only one link's worth of traffic, while interior nodes carry disproportionately
more; NCCL's double binary tree exploits the fact that in a binary tree with N ranks, at most
half of the ranks are interior nodes and at least half are leaves, so a second, complementary
binary tree can be constructed in which every leaf of the first tree becomes an interior node
of the second and vice versa. Running an AllReduce over both trees simultaneously — half of
each rank's data on each tree — means every rank does both interior-node and leaf work, and
the two trees together give full bandwidth utilization at logarithmic latency, rather than the
linear latency ring-AllReduce pays as node count grows.
[Source: NCCL 2.4 double binary tree design](https://github.com/NVIDIA/nccl/issues/272)

The practical consequence is that NCCL's algorithm choice is not simply "ring for large
tensors, tree for small ones" as a fixed cutoff; NCCL builds both ring and (double binary)
tree topologies over the detected hardware graph during communicator initialization and
selects per-call based on message size and the cost model computed from the detected topology,
with `NCCL_ALGO` (accepting `Ring`, `Tree`, or a comma-separated set) available to force one
explicitly for benchmarking or debugging. At multi-node scale, the tree topology is
particularly important because it is the mechanism that bounds AllReduce latency growth to
`O(log N)` in the number of participating GPUs, which matters directly for the small,
frequent, latency-sensitive collectives that tensor parallelism (§3.2) issues inside every
transformer layer, as distinct from the large, infrequent, bandwidth-bound AllReduce a
data-parallel gradient sync issues once per step.

### 2.2 Topology-Aware Selection: NCCL_TOPO_FILE and Multi-Node Rings

Chapter 66 §18.4 covers NCCL's per-communicator transport probing (NVLink, PCIe, InfiniBand)
at communicator init time. At cluster scale, that probing is driven by an explicit topology
model rather than only live hardware queries. NCCL builds an internal topology graph — GPUs,
NICs, CPUs/NUMA nodes, and the PCIe/NVLink/network links between them — by combining what it
discovers via `nvidia-smi`-equivalent queries and `/sys` PCIe topology with, optionally, a
pre-supplied description. **`NCCL_TOPO_FILE`** points NCCL at an XML file describing this
topology in place of (or supplementing) live auto-detection, which is useful in environments
where NCCL's live probing does not see the true fabric layout — virtualized/containerized
GPU environments where PCIe topology is obscured, or clusters where an operator wants to pin
a known-good topology rather than rely on re-detection on every job launch. NCCL defaults to
loading a system-provided topology file at `/var/run/nvidia-topologyd/virtualTopology.xml` if
one is present, and **`NCCL_TOPO_DUMP_FILE`** writes NCCL's own detected topology back out to
XML, which is the standard way to capture a known-good topology description for later reuse
via `NCCL_TOPO_FILE`.
[Source: NCCL environment variables reference](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)

This topology graph is what NCCL's channel-search algorithm consumes to build its rings and
(double binary) trees across an entire multi-node job rather than a single node's GPUs: rings
and trees are constructed to traverse the fastest available path between adjacent members —
NVLink/NVSwitch within a node, then GPUDirect RDMA over InfiniBand or RoCE between nodes — and
**`NCCL_CROSS_NIC`** controls whether a given ring or tree is allowed to switch which physical
NIC it uses partway through, trading strict same-NIC-per-channel locality (avoiding crossing
network rails, `NCCL_CROSS_NIC=0`) against the flexibility to route around a congested or
unavailable path (`NCCL_CROSS_NIC=1`, or NCCL's topology-dependent default of `2`).
[Source: NCCL environment variables reference](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)
This is the mechanism that connects the fabric design decisions in §5.1 (rail-optimized
topology, which NIC a given GPU is wired to) to the collective algorithm NCCL actually runs: a
rail-optimized fabric is specifically designed so that the GPU-to-NIC mapping NCCL's topology
graph discovers lines up with the ring/tree structure NCCL wants to build, minimizing the
cross-rail hops that `NCCL_CROSS_NIC` would otherwise have to negotiate.

### 2.3 RCCL at Multi-Node Scale

Chapter 66 §18.6 and Chapter 48's RCCL section cover RCCL as an ABI-compatible NCCL
replacement for AMD GPUs, using Infinity Fabric/XGMI intra-node and InfiniBand (via
UCX/RDMA) or RoCE inter-node. The topology-graph and channel-search model described above is
architecturally the same on RCCL's side — RCCL performs its own topology detection via
`rocm_agent_enumerator` and constructs ring/tree channels across the detected XGMI and network
links (Chapter 48's RCCL section) — but the concrete transport bandwidths differ: an 8-GPU
MI300X node's fully-connected XGMI mesh gives every GPU direct peer-to-peer DMA to every other
GPU in the node (Chapter 48), which changes the intra-node portion of the topology graph RCCL
builds relative to a partially-connected NVLink/NVSwitch topology, while the inter-node
portion converges on the same InfiniBand/RoCE fabric considerations covered in §5.1 regardless
of GPU vendor. AMD's ROCm roadmap has signaled a new RDMA transport layer, referred to as ROCm
Optiq, intended to eventually replace RCCL's current libfabric-based network backend for
scale-out InfiniBand clusters (noted in Chapter 48's roadmap); its production maturity was not
independently verified for this chapter. *Note: needs verification against current ROCm
release notes before citing Optiq as shipping.*

## 3. Parallelism Strategies

The strategies in this section answer a question Chapter 246 §6 deliberately left open:
DDP/FSDP and GSPMD/`shard_map` are *mechanisms* for expressing a sharding decision as
collectives, but they do not by themselves decide *what* to shard — the whole model
replicated (data parallelism), individual layers' weights (tensor parallelism), or the layer
sequence itself (pipeline parallelism). Real large-model training combines several of these
simultaneously, and the combination is usually driven by a framework purpose-built for it —
DeepSpeed or Megatron-LM on the PyTorch side, GSPMD's auto-partitioner on the JAX side — rather
than composed by hand.

### 3.1 Data Parallelism: Scaling DDP/FSDP Across Nodes

Data parallelism — the strategy Chapter 246 §6.1–§6.2 already covers in mechanism — replicates
(DDP) or shards (FSDP) the model across ranks and partitions the *data*, not the model, so
each rank processes a different micro-batch and synchronizes gradients (DDP's bucketed
AllReduce) or parameters/gradients/optimizer state (FSDP's all-gather/reduce-scatter pair)
after each step. At multi-node scale, the collectives Chapter 246 describes simply run over a
communicator that spans multiple nodes instead of one, which is exactly the scenario §2.1–§2.2
address: the tree algorithm's logarithmic latency and the topology-aware ring/tree construction
matter more as node count grows, because a purely bandwidth-optimal ring's latency term
(proportional to the number of ranks) starts to dominate at scale. Data parallelism alone does
not reduce per-GPU memory pressure the way FSDP's sharding or the strategies below do when a
single layer, or the model as a whole, no longer fits in one GPU's memory even with FSDP's
full-parameter sharding — which is the scenario tensor and pipeline parallelism exist for.

### 3.2 Tensor Parallelism: Intra-Layer Sharding

Tensor parallelism (also called intra-layer or "model" parallelism) splits the weight matrices
*inside* individual layers across GPUs, rather than splitting which layers each GPU owns.
Megatron-LM's approach to a transformer MLP block is the canonical example: the first linear
layer's weight matrix is partitioned column-wise across GPUs (each GPU computes a
disjoint slice of the hidden activations, requiring no communication for that matmul alone,
since each GPU has all the input activations already), and the second linear layer's weight
matrix is partitioned row-wise, so that each GPU's partial output must be summed across GPUs
to produce the correct final result — an **AllReduce** placed at the boundary between the two
linear layers. Self-attention is partitioned analogously, splitting attention heads across
GPUs so each GPU computes a subset of heads independently, with an AllReduce again needed
where the per-head outputs are concatenated and projected back down. This means every
transformer layer contributes a small number of AllReduces (typically two: one after
attention, one after the MLP) rather than one large AllReduce per training step the way data
parallelism does — many more, smaller, latency-sensitive collectives, which is exactly the
workload §2.1's tree algorithm is optimized for and which is why tensor parallelism is
conventionally confined to GPUs connected by the highest-bandwidth, lowest-latency link
available (NVLink/NVSwitch within a node) rather than spanning nodes over InfiniBand.
[Source: Megatron-LM, "Efficient Large-Scale Language Model Training on GPU Clusters"](https://arxiv.org/pdf/2104.04473)

### 3.3 Pipeline Parallelism: Inter-Layer Sharding and the Bubble

Pipeline parallelism instead partitions the model *sequentially*: each GPU (or group of GPUs)
owns a contiguous subset of the model's layers as one pipeline **stage**, and a training batch
is split into smaller **micro-batches** that flow through the stages like an assembly line —
stage `i` finishes its forward pass on micro-batch `k` and hands the activations to stage
`i+1`, while stage `i` immediately begins its forward pass on micro-batch `k+1`. The
unavoidable cost is the **pipeline bubble**: at the start of a batch, later stages sit idle
waiting for the first micro-batch to arrive through earlier stages, and symmetrically at the
end of a batch, earlier stages sit idle waiting to receive the corresponding backward pass;
with `p` pipeline stages and `m` micro-batches per batch, the bubble time fraction is
`(p-1)/m` under the naive fill-drain (GPipe-style) schedule, meaning more micro-batches per
batch amortizes the fixed fill/drain cost, at the expense of more activation memory held
simultaneously (one set per in-flight micro-batch).
[Source: Megatron-LM, "Efficient Large-Scale Language Model Training on GPU Clusters"](https://arxiv.org/pdf/2104.04473)
Megatron-LM's default schedule, **1F1B** ("one-forward-one-backward"), interleaves forward and
backward micro-batch execution per stage rather than running all forwards before any backward,
which bounds the number of in-flight activation sets to the pipeline depth rather than the
micro-batch count, reducing peak activation memory relative to the naive schedule at the same
bubble fraction; an **interleaved** variant assigns each GPU multiple smaller, non-contiguous
stages rather than one contiguous block, further shrinking the bubble fraction at the cost of
more frequent, smaller point-to-point transfers between stages.
[Source: Megatron-LM, "Efficient Large-Scale Language Model Training on GPU Clusters"](https://arxiv.org/pdf/2104.04473)
Communication between pipeline stages is point-to-point (activations forward, gradients
backward across the stage boundary), not a collective in the AllReduce sense — a materially
different communication pattern from both data parallelism's periodic AllReduce and tensor
parallelism's per-layer AllReduce, which is part of why pipeline-parallel stage boundaries are
the dimension most tolerant of being placed across the *slower* inter-node link in a hybrid
deployment (§3.4), rather than being confined to intra-node NVLink the way tensor parallelism
usually is.

### 3.4 3D and Hybrid Parallelism: DeepSpeed ZeRO and Megatron-LM

Real large-model training combines data, tensor, and pipeline parallelism — commonly called
**3D parallelism** — because each strategy alone hits a different wall: data parallelism does
not shrink per-GPU model memory (mitigated by FSDP/ZeRO sharding), tensor parallelism's
per-layer AllReduce traffic makes it impractical across the slower inter-node fabric, and
pipeline parallelism's bubble overhead grows as pipeline depth increases relative to available
micro-batches. The standard placement, followed by both Megatron-LM and DeepSpeed's 3D
configurations, nests the strategies by communication cost: tensor-parallel groups confined to
GPUs within one node (highest-bandwidth NVLink/NVSwitch link, absorbing the most frequent
small AllReduces), pipeline-parallel stages spanning across nodes (point-to-point-only
traffic, tolerant of higher inter-node latency), and data parallelism as the outermost
dimension replicating the whole tensor+pipeline-parallel group across as many such groups as
the cluster has capacity for.

**DeepSpeed's ZeRO** (Zero Redundancy Optimizer) approaches the same memory problem from the
data-parallelism side rather than by introducing model-parallel dimensions: instead of every
data-parallel rank holding a full replica of optimizer state, gradients, and parameters, ZeRO
partitions them across the data-parallel group in three increasing stages. **ZeRO-1**
partitions only the optimizer states (for Adam, the 32-bit master weights and first/second
moment estimates), cutting optimizer-state memory from `12Φ` to `12Φ/N` for `N`-way data
parallelism (`Φ` = parameter count) with no additional communication beyond the DDP-style
gradient AllReduce. **ZeRO-2** additionally partitions gradients — each rank retains only the
reduced gradient shard corresponding to its optimizer-state shard, using a reduce-scatter in
place of an AllReduce, dropping gradient memory from `2Φ` to `2Φ/N`. **ZeRO-3** additionally
partitions the parameters themselves, gathering each layer's full parameter shard via
all-gather immediately before it is needed for compute and releasing it again afterward,
bringing total per-rank model-state memory down to `16Φ/N` (mixed-precision Adam) — the same
all-gather/reduce-scatter decomposition Chapter 246 §6.2 describes for PyTorch FSDP, since
FSDP was built following the same partitioning strategy ZeRO-3 introduced.
[Source: DeepSpeed, "Zero Redundancy Optimizer" tutorial](https://www.deepspeed.ai/tutorials/zero/)
**ZeRO-Offload** and its successor **ZeRO-Infinity** extend ZeRO-3 by moving the partitioned
optimizer state and, for ZeRO-Infinity, parameters as well, out to CPU memory or even NVMe
storage between uses, trading additional PCIe/NVMe transfer latency for the ability to train
models whose full state would not fit in aggregate GPU memory at all.
[Source: DeepSpeed, "Zero Redundancy Optimizer" tutorial](https://www.deepspeed.ai/tutorials/zero/)
DeepSpeed and Megatron-LM are commonly composed directly — DeepSpeed's ZeRO data-parallel
sharding as the outer dimension, Megatron-LM's tensor- and pipeline-parallel implementation as
the inner dimensions — which is the origin of "Megatron-DeepSpeed" as a combined training
stack for very large models.
[Source: "Using DeepSpeed and Megatron to Train Megatron-Turing NLG 530B"](https://arxiv.org/pdf/2201.11990)

### 3.5 JAX's Alternative: GSPMD and shard_map

Chapter 246 §6.3 describes GSPMD and `shard_map` (the user-facing API for which is Chapter 245
§5's `Mesh`/`NamedSharding`/`PartitionSpec`) as compiler-mechanical alternatives to PyTorch's
explicitly-scheduled DDP/FSDP hooks: rather than a framework wiring specific collectives into
specific backward-pass hooks, sharding annotations on a `jax.jit`-ed function's arguments
propagate through the traced HLO graph, and XLA's GSPMD partitioner rewrites the whole-array
program into a single per-device SPMD program with the necessary collectives inserted
automatically. The strategies in §3.1–§3.4 are not PyTorch-specific concepts that JAX lacks —
they are expressible as different `PartitionSpec`/`Mesh` configurations over the same GSPMD
mechanism: a `Mesh` axis used to shard the batch dimension expresses data parallelism: an axis
used to shard a weight matrix's output-feature dimension expresses tensor parallelism; pipeline
parallelism is expressible but is less commonly auto-partitioned end-to-end than data/tensor
sharding, since inter-stage scheduling (§3.3's bubble-minimizing 1F1B ordering) is a scheduling
decision GSPMD's data-flow-driven sharding propagation does not natively express, and is more
commonly handled by explicit `shard_map`-based control flow or a dedicated pipelining library
layered on top of JAX rather than by GSPMD auto-partitioning alone. The practical distinction
Chapter 246 §6.3 draws — PyTorch's collectives scheduled explicitly by the framework versus
JAX's collectives inserted by the compiler as a consequence of sharding-constraint propagation
— applies identically whether the sharding in question expresses data, tensor, or a hybrid
strategy; this chapter's parallelism *taxonomy* is the same across both frameworks, only the
mechanism generating the resulting NCCL calls differs.

## 4. GPU-Accelerated Data Loading

### 4.1 The CPU Data-Loading Bottleneck

A multi-node training job with the collective and parallelism infrastructure of §2–§3 correctly
configured can still be bottlenecked by something upstream of any GPU-to-GPU communication: the
per-step cost of getting the next batch of raw data — decoded, augmented, and laid out in
device memory — ready before the GPU finishes the previous step's compute. Traditional data
pipelines decode and augment on the CPU (JPEG/video decode, resize, crop, color-space
conversion, normalization), and as GPU compute throughput has scaled faster than
single-core CPU throughput, that CPU-side pipeline increasingly cannot keep pace with the GPU's
appetite for the next batch, leaving the GPU idle waiting on data rather than saturated on
compute — a bottleneck that gets strictly worse in multi-node training, where more GPUs need
feeding from the same per-node CPU core count.
[Source: NVIDIA DALI developer page](https://developer.nvidia.com/dali)

### 4.2 NVIDIA DALI: A GPU-Resident Pipeline

**DALI** (Data Loading Library) addresses this by moving the decode-and-augment pipeline onto
the GPU itself rather than only optimizing the CPU-side implementation. A DALI pipeline is
defined as a directed graph of operators — built with the `@pipeline_def` decorator in current
DALI releases — where each operator runs on a **CPU**, **GPU**, or **Mixed** backend; a
canonical pipeline reads raw compressed bytes on the CPU (I/O-bound, not worth moving to the
GPU), decodes them via a `mixed`-backend operator that accepts CPU input and produces
GPU-resident output — for JPEG, backed by NVIDIA's hardware **nvJPEG** decoder, including
support for the dedicated JPEG decode hardware block introduced on NVIDIA A100 — and performs
the remaining resize/crop/normalize augmentation steps as GPU operators, so that from decode
onward, no data crosses back to host memory before reaching the training step.
[Source: NVIDIA DALI documentation](https://docs.nvidia.com/deeplearning/dali/user-guide/docs/index.html);
[Source: "Loading Data Fast with DALI and the New Hardware JPEG Decoder in NVIDIA A100 GPUs"](https://developer.nvidia.com/blog/loading-data-fast-with-dali-and-new-jpeg-decoder-in-a100/)
DALI is explicitly framework-portable — the same pipeline definition connects to PyTorch,
TensorFlow, and JAX via framework-specific iterator classes rather than being reimplemented per
framework. The PyTorch integration (`DALIGenericIterator`) yields PyTorch tensors directly from
pipeline outputs; the JAX integration provides an analogous iterator that yields `jax.Array`s
and, notably, accepts a `jax.sharding.Sharding`-compatible object so that DALI-produced batches
land pre-sharded across a `Mesh` in a form compatible with `jax.jit`-ed, GSPMD-partitioned
training steps (§3.5) without an extra host-side reshaping/redistribution pass. DALI also
supports embedding arbitrary JAX functions as pipeline operators via `nvidia.dali.plugin.jax.fn`,
including `jax.jit`-compiled and `shard_map`-based multi-GPU augmentation operators, letting a
custom augmentation step benefit from JAX's own compilation while remaining part of the same
DALI graph.
[Source: NVIDIA DALI JAX plugin documentation](https://docs.nvidia.com/deeplearning/dali/user-guide/docs/plugins/jax_tutorials.html)

### 4.3 PyTorch DataLoader: Worker-Process Prefetching

PyTorch's default `torch.utils.data.DataLoader` takes the opposite architectural approach from
DALI: rather than moving decode/augmentation onto the GPU, it parallelizes the CPU-side
pipeline across processes. Setting `num_workers > 0` spawns that many worker subprocesses, each
holding its own copy of the `Dataset` object, pulling and transforming samples independently
and queuing completed batches back to the main process; `prefetch_factor` controls how many
batches *each* worker is allowed to prepare ahead of when they are actually consumed, so the
total amount of look-ahead buffering scales with `num_workers × prefetch_factor`.
`persistent_workers=True` keeps the worker pool alive across epochs rather than respawning it,
and `pin_memory=True` allocates the returned batch in page-locked host memory so the
host-to-device copy that follows can use an asynchronous DMA transfer instead of a slower
pageable-memory copy.
[Source: PyTorch, "Data Loading Optimization in PyTorch" tutorial](https://docs.pytorch.org/tutorials/intermediate/intermediate_data_loading_tutorial.html)
This design has a structural ceiling DALI's GPU-resident approach does not share: because each
worker is a separate Python process, batches returned to the main process must be serialized
and deserialized across the multiprocessing boundary, and that serialization overhead — not
just raw decode/augment compute — can prevent `num_workers`/`prefetch_factor` increases from
scaling throughput linearly, particularly for larger tensors (e.g. video, high-resolution
images) where per-batch serialization cost itself becomes significant.
[Source: PyTorch GitHub issue, "num_worker and prefetch_factor in DataLoader do not scale"](https://github.com/pytorch/pytorch/issues/81412)
`DataLoader` and DALI are not mutually exclusive choices at the framework-integration level —
DALI's `DALIGenericIterator` is designed to be dropped in wherever a `DataLoader`-produced
iterator was previously used — but they represent different answers to where the CPU bottleneck
described in §4.1 should be relieved: more CPU parallelism versus moving the work off the CPU
entirely.

### 4.4 JAX Input Pipelines: tf.data and Grain

JAX does not ship its own built-in data-loading library analogous to `DataLoader`, by design —
consistent with JAX's narrower core-library scope described in Chapter 245, input pipelines are
left to companion libraries rather than built into `jax` itself. The two most common choices in
the JAX ecosystem are **`tf.data`**, TensorFlow's data pipeline API used purely for its
input-pipeline capabilities (decode, shuffle, batch, prefetch) independent of any TensorFlow
model code, and **Grain**, a JAX-ecosystem data-loading library built specifically around
deterministic, checkpointable input pipelines — reproducing the exact same data order after a
restart from a checkpoint, which matters for the elastic/fault-tolerant restart scenarios in
§5.3, where a training job may resume mid-epoch on a different set of hosts than it was running
on when it checkpointed. Both integrate with JAX's `jax.Array`/`Mesh` sharding model the same
way DALI's JAX iterator does (§4.2): the input pipeline is responsible for producing
already-sharded batches (or batches that JAX's own device-put/sharding machinery distributes)
rather than JAX itself owning any decode or augmentation logic.

## 5. Cluster Topology and Scheduling

### 5.1 InfiniBand and RoCE Fabric Considerations

Multi-node collectives (§2) are only as fast as the fabric carrying them, and the two dominant
fabric choices for GPU training clusters — InfiniBand and RoCE (RDMA over Converged Ethernet,
specifically RoCEv2) — both provide the GPUDirect RDMA path NCCL/RCCL use to move data directly
between GPU memory and the NIC without a CPU-side copy, but differ in consistency and typical
latency: a well-tuned InfiniBand fabric commonly achieves single-digit-microsecond latency for
small messages with strong throughput consistency under contention, while RoCEv2 over Ethernet
is more sensitive to congestion and switch configuration (priority flow control, ECN tuning)
to reach comparable latency and loss behavior, though a correctly tuned RoCEv2 fabric is used at
very large scale in production — Meta's, Microsoft Azure's, and AWS's largest GPU training
clusters are built on RoCEv2 over Ethernet rather than InfiniBand.
[Source: "GPU Cluster Networking: InfiniBand vs. RoCE for Large-Scale AI Training"](https://inflect.com/blog/gpu-cluster-networking-infiniband-vs.-roce-for-large-scale-ai-training)
The dominant physical topology for both fabric choices in large training clusters is
**rail-optimized**: each GPU in a node is wired to a specific, consistent NIC ("rail"), and
same-rail NICs across every node in the cluster connect to the same leaf switch, so that a
collective's cross-node hops stay within one rail's switch layer rather than crossing between
rails, minimizing the number of switch hops — and therefore latency and blocking-probability —
an AllReduce's inter-node traffic has to traverse. This is the physical layout that
`NCCL_CROSS_NIC` (§2.2) is tuned against: a correctly rail-aligned fabric lets NCCL's default
same-NIC-per-channel behavior hold without penalty, while a fabric where rail alignment breaks
down (oversubscribed rails, heterogeneous node wiring) is where allowing NCCL to cross NICs
becomes a meaningful lever.
[Source: "GPU Cluster Network Topology Design"](https://introl.com/blog/gpu-cluster-network-topology-fat-tree-dragonfly-rail-optimized-2025)

### 5.2 Kubernetes GPU Scheduling for Training Jobs

Chapter 240 §7.1 documents the standard Kubernetes GPU-scheduling stack — the NVIDIA device
plugin exposing GPUs as the schedulable `nvidia.com/gpu` resource, the NVIDIA GPU Operator
automating driver/toolkit/monitoring deployment across a cluster, and GPU Feature Discovery
(GFD) labeling nodes with GPU model, memory, and MIG capability — in the context of Physical AI
orchestration (OSMO, Omniverse Farm). The same stack, and specifically the same node labels
GFD produces, is what a training-focused scheduler consumes to make **topology-aware**
placement decisions: because tensor parallelism's per-layer AllReduces (§3.2) are latency-
sensitive enough to require the fastest available link, a training job's tensor-parallel group
needs every one of its ranks placed on GPUs within one node (or one NVSwitch domain), and a
generic Kubernetes scheduler that only tracks GPU *count* per node, without topology awareness,
can scatter a job's pods across nodes in a way that silently forces tensor-parallel
communication onto the slower inter-node fabric. Kubernetes' **Topology Manager**, together
with node-selector/affinity rules driven by GFD's labels, and — on newer clusters — **Dynamic
Resource Allocation (DRA)** exposing richer GPU attributes (including interconnect topology)
directly to the scheduler, are the mechanisms that close this gap, letting a scheduler co-locate
a tensor-parallel group's pods on NVLink-connected GPUs specifically rather than any GPUs with
free capacity.
[Source: "How to Configure Kubernetes Topology-Aware GPU Scheduling"](https://oneuptime.com/blog/post/2026-02-09-topology-aware-gpu-scheduling-nvlink/view)

### 5.3 Elastic and Fault-Tolerant Restart

At the scale multi-node training runs at, some fraction of nodes failing mid-job — a GPU Xid
error, a NIC flapping, a host becoming unreachable — is a statistical certainty rather than an
edge case, and the collective operations in §2 make a hard failure mode explicit: an AllReduce
is a synchronizing operation across every rank in the communicator, so one rank dying leaves
every surviving rank blocked indefinitely waiting on a collective that will never complete,
unless the training infrastructure detects the failure and either restarts the failed rank or
reforms the communicator around the survivors. PyTorch's answer is **`torchrun`**'s elastic
mode, built on **TorchElastic**: an **elastic agent** runs on each node, and agents coordinate
membership and consensus on when to start or resume training through a **rendezvous** backend
(the recommended `c10d` backend, or an `etcd`-based backend for larger or Kubernetes-orchestrated
deployments) rather than through a static, pre-agreed rank list. On a membership change — a node
failing, or, in genuinely elastic deployments, a node joining — `torchrun` terminates and
respawns the affected processes, re-running rendezvous with the new membership and typically
resuming training from the most recent checkpoint (§5.4) rather than restarting the whole job
from scratch.
[Source: PyTorch, "Fault-tolerant Distributed Training with torchrun"](https://docs.pytorch.org/tutorials/beginner/ddp_series_fault_tolerance.html)
A newer PyTorch-native mechanism, **TorchFT**, aims at a stronger property than checkpoint-based
restart — recovering from a failed rank without necessarily reloading a checkpoint at all,
integrated with the TorchTitan training framework for large-scale runs; its production maturity
and adoption breadth outside of the specific deployments that have reported using it were not
independently verified for this chapter. *Note: needs verification against current TorchFT
project status.*

### 5.4 Checkpointing at Scale

Whether restart is triggered by an elastic-agent-detected failure (§5.3) or a routine
preemption, resuming correctly requires a checkpoint recent enough that the recomputation cost
of the lost steps is acceptable — and at the model sizes 3D parallelism (§3.4) targets, writing
that checkpoint is itself an expensive, synchronizing operation that can stall training for a
meaningful fraction of wall-clock time if done naively. PyTorch's **`torch.distributed.checkpoint`**
(DCP) is built specifically for this: it writes a **sharded** checkpoint format, where each
rank persists only the shard of model/optimizer state it locally holds (consistent with FSDP's
or ZeRO's own sharding, §3.4) rather than gathering a full unsharded copy onto one rank first,
and it supports **asynchronous** checkpointing, in which the GPU-resident state is first staged
into pinned host memory (a fast, blocking copy) and the actual, slower write to persistent
storage then proceeds on a background thread while training continues on the GPU, so the
training loop's stall is bounded by the staging copy rather than by total storage I/O time.
[Source: PyTorch, "Asynchronous Saving with Distributed Checkpoint (DCP)"](https://docs.pytorch.org/tutorials/recipes/distributed_async_checkpoint_recipe.html)
A further wrinkle specific to elastic restart is **reshard-on-load**: a job that checkpointed
under one parallelism configuration (a given tensor-parallel/pipeline-parallel/data-parallel
degree, tied to how many nodes were available at checkpoint time) may need to resume under a
*different* configuration if the available node count changed — DCP's sharded format is
designed to be re-partitioned across a different rank/shard layout on load rather than requiring
an exact match to the configuration that wrote it, which is what makes elastic restart onto a
differently-sized cluster tractable rather than requiring a full-precision, fully-gathered
checkpoint as an intermediate step. JAX's equivalent, Orbax (introduced in Chapter 245 §8),
provides the analogous sharded-checkpoint and cross-mesh-reshaping capability for
GSPMD/`shard_map`-partitioned JAX training state, and is not re-described here.

## 6. Integrations

- **Chapter 66 (CUDA Runtime, Streams, and NVRTC), §18**: owns NCCL's core API, single-
  communicator ring/tree AllReduce mechanics, and per-communicator NVLink/PCIe/InfiniBand
  transport selection; this chapter's §2 builds on that material rather than restating it,
  covering the double-binary-tree algorithm's internal structure and multi-node topology-file-
  driven channel construction specifically.
- **Chapter 246 (JAX and PyTorch Internals), §6**: owns the mechanism by which PyTorch
  DDP/FSDP and JAX GSPMD/`shard_map` turn a sharding decision into scheduled or compiler-
  inserted collectives; this chapter's §3 covers the strategic layer above that mechanism —
  which sharding strategy (data/tensor/pipeline/hybrid) a given model and cluster size calls
  for — and §3.5 maps that same taxonomy onto JAX's compiler-driven sharding.
- **Chapter 245 (The JAX Ecosystem), §5 and §8**: owns the `Mesh`/`NamedSharding`/
  `PartitionSpec`/`shard_map` user-facing sharding API this chapter's §3.5 assumes, and the
  Orbax checkpointing library this chapter's §5.4 cross-references rather than re-describes.
- **Chapter 48 (ROCm and Machine Learning on Linux GPUs)**: owns RCCL's topology detection via
  `rocm_agent_enumerator` and XGMI/Infinity Fabric intra-node transport detail that this
  chapter's §2.3 builds on for multi-node RCCL behavior.
- **Chapter 4 (GPU Memory Management) and Chapter 49 (Multi-GPU and PRIME Render Offload)**:
  own the kernel-level GEM/DMA-BUF/PRIME peer-to-peer DMA primitives that NVLink/XGMI
  transports (§2.1–§2.3) and GPUDirect RDMA (§5.1) are built on.
- **Chapter 69 (Omniverse, USD, and the LIVERPS Stack)**: introduced single-node multi-GPU
  workload partitioning in a rendering context; this chapter's §3 is the training-specific,
  multi-node generalization of that same underlying hardware capability.
- **Chapter 240 (NVIDIA Cosmos, OSMO, and Omniverse Farm), §7.1**: owns the Kubernetes GPU
  Operator/device-plugin/GPU Feature Discovery stack this chapter's §5.2 builds on for
  topology-aware training-job placement specifically, as distinct from Chapter 240's Physical
  AI orchestration use case.

## 7. Roadmap

### Near-term (6–12 months)

- **PyTorch's DTensor/native-sharding work continues narrowing the API gap with JAX's
  `Mesh`/`PartitionSpec` model** (Chapter 246 §6.3's roadmap already flags this convergence at
  the mechanism level); the parallelism-strategy taxonomy in this chapter's §3 is expected to
  remain framework-agnostic even as the specific PyTorch APIs expressing it keep changing.
- **Reshard-on-load checkpointing (§5.4) is an active area of continued optimization** across
  both `torch.distributed.checkpoint` and Orbax, as elastic training makes "resume on a
  differently-shaped cluster" progressively more of a routine operational requirement rather
  than an edge case.

### Medium-term (1–3 years)

- **AMD's ROCm Optiq transport (§2.3), if it matures as signaled**, would give RCCL a network
  backend developed specifically for scale-out InfiniBand clusters rather than inherited
  libfabric plumbing, narrowing one of the remaining multi-node-scale gaps between the NCCL and
  RCCL ecosystems; its actual production timeline should be verified independently before
  relying on it.
- **TorchFT-style failure recovery without a full checkpoint reload (§5.3)**, if it matures
  beyond its current early integrations, would change the cost model for §5.3/§5.4 together —
  reducing how often the expensive sharded-checkpoint write path needs to be on a training job's
  critical path for fault tolerance specifically, as distinct from routine periodic
  checkpointing for other reasons (evaluation, publishing intermediate artifacts).

### Long-term

- **Rail-optimized InfiniBand/RoCE fabric design (§5.1) and topology-aware Kubernetes GPU
  scheduling (§5.2) are likely to keep converging into a single, more automated placement
  layer**, where the physical fabric topology a cluster operator designs and the
  logical placement decisions a training-job scheduler makes are derived from the same
  topology description rather than being configured and reasoned about separately, reducing
  the operational risk of the mismatch §5.1 and §2.2 describe (a scheduler placing a
  tensor-parallel group across nodes a rail-optimized fabric was specifically designed to keep
  it off of).

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
