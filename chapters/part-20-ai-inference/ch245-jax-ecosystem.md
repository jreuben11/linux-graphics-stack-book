# Chapter 245: The JAX Ecosystem

**Target audiences:** Graphics application developers and researchers building differentiable
rendering, simulation, or ML training pipelines on Linux GPUs; systems developers who need
to understand how a `jax.jit`-compiled workload reaches the driver. This chapter is a guide
to JAX's programming model and the library ecosystem built on top of it — `jax.numpy`, the
core function transformations, and the surrounding stack of Flax, Optax, Orbax, and related
tooling. It deliberately does not explain *how* `jax.jit` tracing or XLA compilation work
internally: that mechanism, and its comparison with PyTorch's `torch.compile`, is covered in
Chapter 246 ("JAX and PyTorch Internals"). The XLA/HLO/StableHLO compiler pipeline that
`jax.jit` lowers into is covered in Chapter 91 §7. This chapter's job is the layer a JAX user
actually writes code against.

---

## Table of Contents

1. [Core Programming Model](#1-core-programming-model)
   - [1.1 jax.numpy and jax.Array](#11-jaxnumpy-and-jaxarray)
   - [1.2 Functional Purity and In-Place Updates](#12-functional-purity-and-in-place-updates)
2. [Core Function Transformations](#2-core-function-transformations)
   - [2.1 jax.jit](#21-jaxjit)
   - [2.2 jax.grad and jax.value_and_grad](#22-jaxgrad-and-jaxvalue_and_grad)
   - [2.3 jax.vmap](#23-jaxvmap)
   - [2.4 Composing Transformations](#24-composing-transformations)
   - [2.5 Custom Derivative Rules](#25-custom-derivative-rules)
3. [The PRNG Model](#3-the-prng-model)
4. [Pytrees](#4-pytrees)
5. [Multi-Device and Sharding](#5-multi-device-and-sharding)
   - [5.1 jax.Array, Mesh, NamedSharding, PartitionSpec](#51-jaxarray-mesh-namedsharding-partitionspec)
   - [5.2 shard_map and the pmap Transition](#52-shard_map-and-the-pmap-transition)
6. [Flax: Neural Network Modules](#6-flax-neural-network-modules)
7. [Optax: Functional Optimizers](#7-optax-functional-optimizers)
8. [Orbax: Checkpointing](#8-orbax-checkpointing)
9. [Testing and Typing Aids: chex and jaxtyping](#9-testing-and-typing-aids-chex-and-jaxtyping)
10. [Equinox: Models as Pytrees](#10-equinox-models-as-pytrees)
11. [Haiku: Legacy Status](#11-haiku-legacy-status)
12. [Installation and Backends on Linux](#12-installation-and-backends-on-linux)
13. [Interoperability](#13-interoperability)
14. [Library Comparison](#14-library-comparison)
15. [Integrations](#15-integrations)

---

## 1. Core Programming Model

### 1.1 jax.numpy and jax.Array

JAX exposes a NumPy-compatible array API through `jax.numpy` (conventionally imported as
`jnp`). Almost anything expressible with `numpy` is expressible with `jax.numpy`, which
makes JAX approachable to anyone coming from NumPy or SciPy:

```python
import jax.numpy as jnp

x = jnp.linspace(0, 10, 1000)
y = 2 * jnp.sin(x) * jnp.cos(x)
```

[Source: JAX "Quickstart: How to think in JAX"](https://docs.jax.dev/en/latest/notebooks/thinking_in_jax.html)

Every value produced by `jax.numpy` (and by JAX's transformations generally) is a
`jax.Array` — the single unified array type that represents both single-device and
sharded, multi-device values (§5). Earlier JAX releases exposed a separate
`DeviceArray`/`ShardedDeviceArray` type hierarchy; `jax.Array` unified these into one
type, so code and documentation written against `jax.Array` covers both the single-device
and distributed cases without a separate code path.
[Source: jax.Array API reference](https://docs.jax.dev/en/latest/_autosummary/jax.Array.html)

### 1.2 Functional Purity and In-Place Updates

`jax.Array` values are immutable — unlike NumPy, direct item assignment raises an error:

```python
x = jnp.arange(10)
x[0] = 10  # TypeError: JAX arrays are immutable
```

In-place-looking updates are instead expressed functionally through the `.at[...]`
indexing helper, which returns an updated *copy* rather than mutating the original:

```python
y = x.at[0].set(10)
print(x)  # [0 1 2 3 4 5 6 7 8 9]  (unchanged)
print(y)  # [10 1 2 3 4 5 6 7 8 9]
```

[Source: JAX "Quickstart: How to think in JAX"](https://docs.jax.dev/en/latest/notebooks/thinking_in_jax.html)

Immutability is not an incidental restriction — it is what makes a JAX function safe to
trace, cache, and re-execute under `jit`, `vmap`, and `grad` without the aliasing and
ordering hazards that in-place mutation would introduce into a traced program. What
"tracing" actually means mechanically — how a Python function becomes a `jaxpr` — is
covered in Chapter 246 §1; here it is enough to know that JAX transformations expect
*pure* functions: no in-place mutation of arguments, no reliance on external mutable
state, and no Python-level side effects (`print` inside a jitted function only fires
during the trace, not on every call).

## 2. Core Function Transformations

JAX's transformations are higher-order functions: each takes a Python function and
returns a new function with different behavior — compiled, differentiated, or
vectorized. They compose arbitrarily with each other.
[Source: JAX "Quickstart: How to think in JAX"](https://docs.jax.dev/en/latest/notebooks/thinking_in_jax.html)

### 2.1 jax.jit

`jax.jit` compiles a function via XLA the first time it is called with a given
combination of input shapes and dtypes, and reuses the compiled executable on
subsequent calls with matching shapes/dtypes:

```python
from jax import jit

def norm(X):
    X = X - X.mean(0)
    return X / X.std(0)

norm_compiled = jit(norm)
```

`jit` requires every array in the traced computation to have a static shape — a
function that indexes with a boolean mask of data-dependent length, for example, will
fail to trace:

```python
def get_negatives(x):
    return x[x < 0]

jit(get_negatives)(x)  # NonConcreteBooleanIndexError
```

Arguments that must vary the compiled program's control flow rather than just its data
(Python bools, small integers used in `if` statements, etc.) are marked with
`static_argnums`/`static_argnames`, which causes `jit` to re-trace and recompile a new
executable whenever their value changes, rather than treating them as an array input.
[Source: JAX "Quickstart: How to think in JAX"](https://docs.jax.dev/en/latest/notebooks/thinking_in_jax.html)

What actually happens between the Python call and the compiled executable — tracer
objects, `jaxpr` construction, and lowering to StableHLO/XLA — is Chapter 246's subject,
not this chapter's; Chapter 91 §7 covers the XLA/HLO/StableHLO pipeline itself.

### 2.2 jax.grad and jax.value_and_grad

`jax.grad` returns a function that computes the gradient of a scalar-valued function
with respect to its first argument by default:

```python
from jax import grad

def sum_logistic(x):
    return jnp.sum(1.0 / (1.0 + jnp.exp(-x)))

derivative_fn = grad(sum_logistic)
derivative_fn(jnp.arange(3.0))
# [0.25       0.19661197 0.10499357]
```

`jax.grad` and `jax.jit` compose and can be mixed arbitrarily — `jit(grad(f))` and
`grad(jit(f))` are both valid and common.
[Source: JAX "Quickstart: How to think in JAX"](https://docs.jax.dev/en/latest/notebooks/thinking_in_jax.html)

`jax.value_and_grad` is the common variant that returns both the function's value and
its gradient in a single traced computation, avoiding a duplicate forward pass when
both are needed — as they typically are in a training loop that logs the loss.
[Source: jax.value_and_grad API reference](https://docs.jax.dev/en/latest/_autosummary/jax.value_and_grad.html)

### 2.3 jax.vmap

`jax.vmap` traces a function the way `jit` does, but instead of compiling it as-is it
adds a batch dimension and pushes the batching down into the primitive operations
(vectorized `matmul` instead of a Python loop of `matmul` calls), so a function written
for a single example can be applied to a batch without being rewritten:

```python
from jax import vmap

def apply_matrix(x):
    return jnp.dot(mat, x)

@jit
def batched_apply_matrix(batched_x):
    return vmap(apply_matrix)(batched_x)
```

`in_axes`/`out_axes` control which axis of each argument/output is treated as the batch
axis (defaulting to axis 0), and can be set to `None` for arguments that should be
broadcast unbatched to every call.
[Source: JAX "Quickstart: How to think in JAX"](https://docs.jax.dev/en/latest/notebooks/thinking_in_jax.html)

### 2.4 Composing Transformations

Because `jit`, `grad`, and `vmap` are ordinary higher-order functions over the same
tracing machinery, they nest freely: `vmap(grad(loss_fn))` computes per-example
gradients over a batch in one traced program, and wrapping the whole thing in `jit`
compiles that combination once. This composability is the core ergonomic argument for
JAX's functional style — there is no separate "batched" or "differentiable" variant of
an op to reach for; the same function is reused under every transformation.

### 2.5 Custom Derivative Rules

For functions where JAX's automatic reverse-mode differentiation of the traced
implementation is undesirable — numerically unstable, or wrapping a non-differentiable
external call — `jax.custom_vjp` and `jax.custom_jvp` let a function supply its own
vector-Jacobian-product or Jacobian-vector-product rule as a decorator, which the
corresponding differentiation transform (`jax.grad`, `jax.jvp`) uses instead of tracing
into the function body.
[Source: jax.custom_vjp API reference](https://docs.jax.dev/en/latest/_autosummary/jax.custom_vjp.html) ·
[Source: jax.custom_jvp API reference](https://docs.jax.dev/en/latest/_autosummary/jax.custom_jvp.html)

## 3. The PRNG Model

JAX's random number generation is explicit and stateless where NumPy's is implicit and
stateful. Instead of a global RNG object that mutates on every call, every random
function takes a **key** as an explicit argument, and a key must be split before reuse:

```python
import jax

key = jax.random.key(0)          # new-style typed key
key1, key2 = jax.random.split(key)  # defaults to producing 2 new keys

x = jax.random.normal(key1, (3,))
y = jax.random.normal(key2, (3,))
```

`jax.random.key()` produces the current typed-key representation, which treats a key as
an opaque scalar array element rather than exposing its underlying `uint32` buffer
layout; `jax.random.PRNGKey()` produces the older, untyped `uint32`-array key format that
the newer API grew out of. `jax.random.key_data()`/`jax.random.wrap_key_data()` convert
between the two representations where a codebase needs to interoperate with both.
[Source: JEP 9263 — Typed keys and pluggable RNGs](https://docs.jax.dev/en/latest/jep/9263-typed-keys.html) ·
[Source: jax.random.split API reference](https://docs.jax.dev/en/latest/_autosummary/jax.random.split.html)

Explicit, splittable keys exist because a global mutable RNG state is exactly the kind
of side effect §1.2 rules out: a function that silently advances a global generator
produces different results depending on trace order and call count, which is
incompatible with `vmap`ping or `pmap`ping the same function over many parallel
instances and expecting reproducible, independent randomness in each.

## 4. Pytrees

A **pytree** is JAX's name for a nested Python container of arrays — lists, tuples, and
dicts nested arbitrarily, with array (or other non-container) values at the leaves. Every
JAX transformation operates over pytrees, not just flat arrays, which is what lets a
`grad` call return a gradient with the same nested structure as a model's parameter
dictionary. `jax.tree_util` (aliased as `jax.tree`) provides the operations:

```python
import jax

leaves = jax.tree.leaves(params)          # flat list of all array leaves
jax.tree.map(lambda x: x * 2, params)     # apply a function to every leaf, same structure back
jax.tree_util.tree_structure(params)      # inspect the container shape without the leaf values
```

`jax.tree.map` requires its inputs' structures to match exactly — lists must have the
same length, dicts the same keys — when called with more than one pytree argument (as in
an SGD update that combines a parameter tree with a gradient tree of the same shape).
Custom container classes participate in this machinery once registered with
`jax.tree_util.register_pytree_node`, which is how Flax's and Equinox's own module types
(§6, §10) become directly usable as arguments to `jit`, `grad`, and `vmap`.
[Source: JAX pytrees documentation](https://docs.jax.dev/en/latest/pytrees.html)

## 5. Multi-Device and Sharding

### 5.1 jax.Array, Mesh, NamedSharding, PartitionSpec

A `jax.Array` (§1.1) can be sharded across multiple devices while still presenting a
single logical array to user code — indexing, slicing, and passing it to `jit`-compiled
functions works the same whether the array lives on one device or is split across
sixty-four. The sharding is described by three objects:

- **`Mesh`** — a multidimensional array of JAX devices where each axis carries a name
  (e.g. `'data'`, `'model'`).
- **`PartitionSpec`** — a tuple describing, per array dimension, which mesh axis (or
  `None` for replication) that dimension is partitioned across. `PartitionSpec('x', 'y')`
  shards an array's first dimension along mesh axis `x` and its second along axis `y`.
- **`NamedSharding`** — the pairing of a `Mesh` and a `PartitionSpec` that JAX actually
  attaches to an array to describe its layout.

```python
from jax.sharding import Mesh, NamedSharding, PartitionSpec as P
import numpy as np

mesh = Mesh(np.array(jax.devices()).reshape(2, 4), axis_names=('data', 'model'))
sharding = NamedSharding(mesh, P('data', 'model'))
```

[Source: jax.sharding module documentation](https://docs.jax.dev/en/latest/jax.sharding.html)

When a `jit`-compiled function is called on inputs carrying a `NamedSharding`, XLA's
GSPMD auto-partitioner decides how to shard every intermediate value and insert the
necessary cross-device collectives automatically — the user writes single-device-looking
code and lets the compiler partition it. This automatic path, and the compiler mechanics
behind it, are covered from the compiler side in Chapter 246 §6.

### 5.2 shard_map and the pmap Transition

`jax.experimental.shard_map` (`shard_map`) is the explicit alternative to auto-partitioned
`jit`: the function body you write is executed *per device*, operating on that device's
local shard, and any cross-device communication (a `psum`, an all-gather) must be written
explicitly using the mesh's named axes — nothing is inferred by the compiler.
`in_specs`/`out_specs` (both `PartitionSpec`s) describe how each argument's dimensions map
onto mesh axes, with unmentioned axis names meaning the argument is replicated across
that axis rather than split.
[Source: "shmap (shard_map) for simple per-device code" JEP](https://docs.jax.dev/en/latest/jep/14273-shard-map.html) ·
[Source: "Manual parallelism with shard_map" tutorial](https://docs.jax.dev/en/latest/notebooks/shard_map.html)

`jax.pmap`, the original multi-device SPMD transform, is not deprecated, but as of JAX
0.8.0 its default implementation is itself built on top of `jax.jit` and `shard_map`
rather than being a separate code path — and the migration guide is explicit that this
new implementation is *not* a perfect drop-in replacement for the original `pmap`
semantics in every case. JAX's own guidance for new code is to migrate directly to
`jax.jit(jax.shard_map(...))` with an explicit `Mesh`/`PartitionSpec` rather than writing
new `pmap` call sites.
[Source: "Migrating to the new jax.pmap"](https://docs.jax.dev/en/latest/migrate_pmap.html)

Practically, this means: greenfield multi-device JAX code should reach for `NamedSharding`
+ `jit` (§5.1) when the partitioning is straightforward enough for GSPMD to infer, and for
explicit `shard_map` when the communication pattern needs to be pinned down exactly (a
custom collective, a pipeline-parallel stage boundary); `pmap` mainly persists for
existing code that has not yet migrated.

## 6. Flax: Neural Network Modules

Flax ([github.com/google/flax](https://github.com/google/flax), Apache-2.0) is Google's
neural-network library built on JAX. It currently ships two module APIs. **Flax NNX** is
the API new users are encouraged to reach for — modules are ordinary, mutable Python
objects rather than the immutable functional structures of Flax's older API, which makes
model code read closer to idiomatic PyTorch:

```python
from flax import nnx

class Model(nnx.Module):
    def __init__(self, din, dmid, dout, rngs: nnx.Rngs):
        self.linear = nnx.Linear(din, dmid, rngs=rngs)
        self.bn = nnx.BatchNorm(dmid, rngs=rngs)
        self.dropout = nnx.Dropout(0.2)
        self.linear_out = nnx.Linear(dmid, dout, rngs=rngs)

    def __call__(self, x, rngs):
        x = nnx.relu(self.dropout(self.bn(self.linear(x)), rngs=rngs))
        return self.linear_out(x)
```

**Flax Linen** (`flax.linen`, usually imported as `nn`) is the older, functional-style
API — a `nn.Module` subclass declares layers and a `__call__`, but parameters are not
stored on the module instance; instead `module.init(key, x)` returns a separate parameter
pytree, and `module.apply(params, x)` runs the forward pass against that external state.
Flax's own project documentation is explicit that Linen is *not* being deprecated in the
near term, since most existing Flax users and codebases still depend on it — the
recommendation to use NNX applies to new projects, not as an end-of-life notice for
Linen.
[Source: Flax documentation](https://flax.readthedocs.io/en/latest/)

Training loops built on either API typically wrap the parameter pytree, optimizer state,
and step count in a `TrainState`-style container, and hand the parameter pytree to Optax
(§7) for the actual update rule and to Orbax (§8) for checkpointing.

## 7. Optax: Functional Optimizers

Optax ([github.com/google-deepmind/optax](https://github.com/google-deepmind/optax),
Apache-2.0) is JAX's optimizer library. Every optimizer is a `GradientTransformation` — a
pair of pure functions, `init` (builds the optimizer's state, e.g. Adam's running moment
estimates, from a parameter pytree) and `update` (consumes gradients and the current
state, returns parameter updates and new state):

```python
import optax

optimizer = optax.adam(learning_rate)
params = {'w': jnp.ones((num_weights,))}
opt_state = optimizer.init(params)

grads = jax.grad(loss_fn)(params, xs, ys)
updates, opt_state = optimizer.update(grads, opt_state)
params = optax.apply_updates(params, updates)
```

[Source: Optax repository](https://github.com/google-deepmind/optax)

Because a `GradientTransformation` is just a pair of functions over pytrees, optimizers
compose: `optax.chain` combines several transformations (e.g. gradient clipping, then a
learning-rate schedule, then Adam's moment update) into a single `GradientTransformation`
that behaves as one optimizer, and Optax's schedule utilities (cosine decay, warmup,
piecewise-constant) plug into that chain as ordinary transformations rather than as a
special case bolted onto the optimizer.

## 8. Orbax: Checkpointing

Orbax ([github.com/google/orbax](https://github.com/google/orbax), Apache-2.0) is the
checkpointing library used across the JAX ecosystem, including by Flax's own training
utilities. It targets the requirements large-scale JAX training actually has: async
checkpoint writes that don't block the training step, sharding-aware saves/restores that
match a checkpoint's on-disk layout to the running job's device mesh, and support for
arbitrary custom pytree types alongside plain arrays:

```python
import jax
from orbax.checkpoint import v1 as ocp

state = {'params': params, 'opt_state': opt_state, 'step': step}
ocp.save('/tmp/my_checkpoint', state)
restored_state = ocp.load('/tmp/my_checkpoint')
```

[Source: Orbax repository](https://github.com/google/orbax)

A `TrainState` built from Flax parameters and an Optax optimizer state (§6, §7) is an
ordinary pytree, so it serializes through Orbax without any Flax- or Optax-specific
checkpoint format — the same async, sharded checkpoint machinery that saves a plain
parameter dict handles a full training state.

## 9. Testing and Typing Aids: chex and jaxtyping

**chex** ([github.com/google-deepmind/chex](https://github.com/google-deepmind/chex),
Apache-2.0) is a library of utilities for writing reliable JAX code: shape/rank/dtype
assertions (`chex.assert_shape(x, (2, 3))`, `chex.assert_rank(x, 0)`,
`chex.assert_equal_shape([x, y, z])`, `chex.assert_tree_all_finite(tree)`), and a
`@chex.variants` test decorator that runs a test body under multiple JAX execution modes
(`with_jit=True, without_jit=True`) so a numerical bug that only appears under tracing
doesn't hide behind a non-jitted test suite.
[Source: chex repository](https://github.com/google-deepmind/chex)

**jaxtyping** ([github.com/patrick-kidger/jaxtyping](https://github.com/patrick-kidger/jaxtyping))
provides type annotations that encode an array's expected shape and dtype directly in a
function signature:

```python
from jaxtyping import Float, Array

def matrix_multiply(
    x: Float[Array, "dim1 dim2"],
    y: Float[Array, "dim2 dim3"],
) -> Float[Array, "dim1 dim3"]:
    ...
```

The annotations are inert on their own (ordinary Python type hints), but jaxtyping is
designed to be checked at runtime by a general-purpose type checker such as `beartype`,
which validates a random sample of a large tree's shapes/dtypes at each call rather than
exhaustively checking every value, keeping the overhead low enough to leave enabled during
development.
[Source: jaxtyping repository](https://github.com/patrick-kidger/jaxtyping)

## 10. Equinox: Models as Pytrees

Equinox ([github.com/patrick-kidger/equinox](https://github.com/patrick-kidger/equinox),
Apache-2.0) is a lighter-weight alternative to Flax: rather than splitting a model into a
separate `Module` definition and an external parameter pytree (Linen's approach) or a
stateful mutable object with its own transformation-filtering rules (NNX's approach),
Equinox modules *are* pytrees directly — an `eqx.Module` subclass declares its parameters
as dataclass-style fields, and the resulting instance can be passed straight into
`jax.jit`, `jax.grad`, or `jax.vmap` like any other pytree:

```python
import equinox as eqx
import jax

class Linear(eqx.Module):
    weight: jax.Array
    bias: jax.Array

    def __init__(self, in_size, out_size, key):
        wkey, bkey = jax.random.split(key)
        self.weight = jax.random.normal(wkey, (out_size, in_size))
        self.bias = jax.random.normal(bkey, (out_size,))

    def __call__(self, x):
        return self.weight @ x + self.bias
```

Because plain `jax.grad` would try to differentiate with respect to every leaf of a
pytree — including non-array fields such as a stored activation function or a boolean
flag — Equinox provides *filtered* transformations, `eqx.filter_jit` and
`eqx.filter_grad`, which apply the underlying JAX transformation only to the array
leaves of a pytree and pass everything else through unchanged.
[Source: Equinox repository](https://github.com/patrick-kidger/equinox)

## 11. Haiku: Legacy Status

Haiku ([github.com/google-deepmind/dm-haiku](https://github.com/google-deepmind/dm-haiku),
Apache-2.0) was DeepMind's original JAX neural-network library, predating both Flax NNX
and, in large part, the ecosystem-standard status Flax now has. DeepMind's own
announcement states that Haiku entered maintenance mode — bug fixes and compatibility
updates for new JAX releases, but no new features and no new-feature pull requests — with
new projects directed toward Flax instead. Google DeepMind has stated it will continue to
support Haiku indefinitely given the library's substantial internal usage, but for a new
project choosing a module system in 2026, Haiku is a maintenance-mode option rather than
the actively developed one.
[Source: dm-haiku repository](https://github.com/google-deepmind/dm-haiku)

## 12. Installation and Backends on Linux

JAX ships separate CPU and accelerator wheels. The GPU wheel installs CUDA-specific
plugin and PJRT packages alongside the framework:

```bash
# CUDA 12 wheels (JAX is migrating its primary support target to CUDA 13;
# CUDA 12 wheels remain available but are expected to be dropped eventually):
pip install --upgrade "jax[cuda12]"

# Using a CUDA toolkit already installed on the system rather than a
# pip-bundled one:
pip install --upgrade "jax[cuda12-local]"
```

Pre-built CUDA wheels are provided only for Linux x86_64 and Linux aarch64; other
platform/accelerator combinations require building from source.
[Source: JAX installation documentation](https://docs.jax.dev/en/latest/installation.html)

`jax[rocm]` is AMD's equivalent GPU wheel; the install matrix, supported ROCm versions,
and current JAX/ROCm version pairing are covered in Chapter 48's "JAX on ROCm" section
and Chapter 108 — this chapter does not restate that matrix. TPU wheels exist but are out
of scope for a book focused on the Linux desktop/workstation GPU stack.

Regardless of backend, `jax.devices()` and `jax.default_backend()` are the standard
runtime introspection calls for confirming which accelerator(s) JAX has actually attached
to at process start.

## 13. Interoperability

Two interop paths let JAX arrays and models cross into other frameworks without a full
data copy:

- **DLPack.** `jax.dlpack.from_dlpack` constructs a `jax.Array` view over an external
  DLPack-capable tensor (a PyTorch or NumPy tensor, for example) sharing the same
  underlying device memory when no device transfer is needed. Because `jax.Array` is
  immutable while a DLPack buffer is not, external code that mutates the source buffer
  in place after the JAX-side array has been constructed can produce undefined behavior
  — the immutability guarantee from §1.2 only holds as long as nothing outside JAX writes
  into the shared buffer.
  [Source: jax.dlpack.from_dlpack API reference](https://docs.jax.dev/en/latest/_autosummary/jax.dlpack.from_dlpack.html)
- **jax2tf.** JAX ships a `jax2tf` conversion path that lowers a JAX function into a
  TensorFlow graph, primarily for deployment into TensorFlow-based serving
  infrastructure (TF Serving, TFLite) that has no native JAX runtime of its own.

## 14. Library Comparison

| Library | Role | Module style | License |
|---|---|---|---|
| Flax NNX | Neural network modules | Mutable Python objects | Apache-2.0 |
| Flax Linen | Neural network modules (established) | Immutable, `init`/`apply` split | Apache-2.0 |
| Haiku | Neural network modules (maintenance mode) | `hk.transform`, functional | Apache-2.0 |
| Equinox | Neural network modules, general pytree utilities | Modules *are* pytrees directly | Apache-2.0 |
| Optax | Optimizers | `GradientTransformation` (`init`/`update`) | Apache-2.0 |
| Orbax | Checkpointing | Async, sharding-aware save/load | Apache-2.0 |
| chex | Testing/assertions | Shape/dtype asserts, `@variants` | Apache-2.0 |
| jaxtyping | Static + runtime shape/dtype typing | Type annotations + `beartype` hook | — |

## 15. Integrations

- **Chapter 91 §7 (MLIR and the Emerging GPU Compiler Infrastructure — XLA)**: the
  XLA/HLO/StableHLO compiler pipeline that `jax.jit` (§2.1) and `jax.jit`-with-sharding
  (§5.1) lower into is covered there in full; this chapter treats that pipeline as a
  black box behind the transformation API.
- **Chapter 246 (JAX and PyTorch Internals — Tracing, Autodiff, and Compilation)**: the
  tracing mechanism behind `jax.jit`/`jax.grad`/`jax.vmap` (jaxpr construction, the
  `Tracer` abstraction), and a side-by-side comparison against PyTorch's TorchDynamo/
  AOTAutograd/TorchInductor pipeline, live in that chapter rather than here.
- **Chapter 48 (ROCm and Machine Learning on Linux GPUs) and Chapter 108 (ROCm and
  HIP)**: the `jax[rocm]` install matrix and current AMD backend support status
  referenced in §12.
- **Chapter 212 (Python 3D ML Libraries — Open3D, PyTorch3D, and Kaolin)**: the
  PyTorch-centric sibling ecosystem for 3D deep learning, contrasted with the JAX-native
  libraries in this chapter.
- **Chapter 199 (Jupyter Internals)**: the typical interactive environment JAX research
  and training code in this chapter's examples is developed and run in.

## Roadmap

### Near-term (6-12 months)
- **`pmap`'s implementation migration continues to settle.** `jax.pmap`'s default
  implementation shifted onto `jax.jit`/`shard_map` foundations in JAX 0.8.0 (§5.2), with
  a temporary legacy-behavior fallback; expect continued documentation and bug-fix
  activity around edge cases where the new implementation's semantics diverge from the
  original as the fallback window closes and more codebases migrate off direct `pmap`
  usage.
  [Source: "Migrating to the new jax.pmap"](https://docs.jax.dev/en/latest/migrate_pmap.html)
- **CUDA 13 becomes JAX's primary GPU target.** JAX's installation guidance already
  points new CUDA 12 installs toward eventual migration to CUDA 13 wheels (§12);
  expect the CUDA 12 wheel to move from "supported" to "legacy" over this window, which
  will affect the ROCm/CUDA version matrices tracked in Chapter 48 and Chapter 108 as
  well.
  [Source: JAX installation documentation](https://docs.jax.dev/en/latest/installation.html)

### Medium-term (1-3 years)
- **Flax NNX consolidates as the default entry point for new projects while Linen's
  installed base keeps it alive.** Flax's own documentation already steers new users to
  NNX (§6) while explicitly declining to set an end-of-life date for Linen; the practical
  trajectory is a widening gap between "what new tutorials teach" (NNX) and "what large
  existing training codebases run" (Linen), rather than a hard cutover.
  [Source: Flax documentation](https://flax.readthedocs.io/en/latest/)
- **Haiku's role narrows to legacy maintenance rather than active adoption.** With
  Haiku already in maintenance mode and DeepMind's own new work built on Flax, expect the
  library's presence in new tutorials, papers, and libraries to keep shrinking even as
  the project itself continues receiving compatibility fixes.
  [Source: dm-haiku repository](https://github.com/google-deepmind/dm-haiku)

### Long-term
- **Explicit sharding APIs (Mesh/NamedSharding/shard_map) keep displacing implicit,
  device-order-dependent parallelism primitives.** The `pmap`→`shard_map` transition
  (§5.2) is one instance of a broader direction in JAX's distributed-array design:
  pushing toward explicit, inspectable sharding descriptions rather than parallelism
  APIs whose behavior depends on an implicit device ordering — a trajectory `jax.Array`'s
  original unification of single- and multi-device arrays (§1.1) already set in motion.
  [Source: jax.sharding module documentation](https://docs.jax.dev/en/latest/jax.sharding.html)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
