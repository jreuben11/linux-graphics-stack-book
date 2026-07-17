# Chapter 168: WebNN — The Web Neural Network API

**Part X — The Browser Rendering Stack**

**Audiences targeted:**
- **Browser and web platform engineers** — need to understand how machine-learning inference is plumbed through Chrome's multi-process architecture
- **ML and AI application developers** — building models that run in the browser without a native runtime
- **Chromium and Firefox contributors** — interested in the service-layer implementation
- **Linux graphics stack engineers** — curious about how the same Mesa/Vulkan drivers used for WebGPU relate to on-device inference

---

## Table of Contents

1. [Introduction: Why WebNN Exists](#1-introduction-why-webnn-exists)
2. [The WebNN API Surface](#2-the-webnn-api-surface)
   - 2.1 [ML Interface and Context Creation](#21-ml-interface-and-context-creation)
   - 2.2 [MLGraphBuilder and the Computation Graph Model](#22-mlgraphbuilder-and-the-computation-graph-model)
   - 2.3 [Operator Coverage](#23-operator-coverage)
   - 2.4 [MLTensor and Buffer Management](#24-mltensor-and-buffer-management)
   - 2.5 [dispatch() and the Execution Timeline](#25-dispatch-and-the-execution-timeline)
   - 2.6 [WebGPU Interoperability](#26-webgpu-interoperability)
3. [Chromium Implementation](#3-chromium-implementation)
   - 3.1 [Process Layout and services/webnn/ Directory Structure](#31-process-layout-and-serviceswebnn-directory-structure)
   - 3.2 [Backend Selection and MLContextOptions](#32-backend-selection-and-mlcontextoptions)
   - 3.3 [CPU Backend: TFLite and XNNPACK on Linux](#33-cpu-backend-tflite-and-xnnpack-on-linux)
   - 3.4 [The Mojo IPC Path](#34-the-mojo-ipc-path)
   - 3.5 [MLTensor and SharedImage Buffer Sharing](#35-mltensor-and-sharedimage-buffer-sharing)
4. [The Linux GPU Inference Situation](#4-the-linux-gpu-inference-situation)
   - 4.1 [What the GPU Backend Actually Means on Linux](#41-what-the-gpu-backend-actually-means-on-linux)
   - 4.2 [WebGPU Context Interop: createContext(GPUDevice)](#42-webgpu-context-interop-createcontextgpudevice)
   - 4.3 [Relation to Dawn and Mesa Vulkan](#43-relation-to-dawn-and-mesa-vulkan)
5. [Firefox Implementation](#5-firefox-implementation)
6. [NPU Backend and Linux](#6-npu-backend-and-linux)
7. [WebNN vs. WebGPU Compute for ML](#7-webnn-vs-webgpu-compute-for-ml)
8. [Developer Workflow](#8-developer-workflow)
   - 8.1 [Feature Detection](#81-feature-detection)
   - 8.2 [ONNX Runtime Web Integration](#82-onnx-runtime-web-integration)
   - 8.3 [Polyfills and Tooling](#83-polyfills-and-tooling)
   - 8.4 [Chrome Flags and Enablement Status](#84-chrome-flags-and-enablement-status)
9. [Integrations](#9-integrations)
10. [References](#10-references)

---

## 1. Introduction: Why WebNN Exists

The browser already has two avenues for GPU computation: **WebGL**, which exposes a fragment-shader execution model well-suited to matrix multiplications through texture sampling tricks, and **WebGPU**, which provides explicit compute shaders that can implement any parallelisable algorithm. Developers running neural-network inference in the browser have used both. A flourishing ecosystem of JavaScript ML frameworks — **TensorFlow.js**, **ONNX Runtime Web**, **Transformers.js** — compiles model graphs to WebGL fragment shaders or WebGPU WGSL kernels. The results are impressive, but the approach carries a deep structural cost: every such framework must ship its own compute-shader library, its own quantisation kernels, its own operator implementations, and its own shape-inference logic. For a typical quantised transformer model the total download including the WASM runtime plus the model weights sits at 50–200 MB, and the first-inference latency while the browser JIT-compiles hundreds of WGSL kernels often exceeds one second.

There is a deeper problem. WebGPU compute shaders are, by design, portable — they know nothing about the underlying hardware beyond the abstract wgpu memory and execution model. But modern systems have hardware that goes further than GPUs: Intel Core Ultra processors have a built-in NPU (the Intel AI Boost), Qualcomm Snapdragon X laptops include the Hexagon Tensor Processor, and Apple Silicon M-series SoCs include the Apple Neural Engine. These devices are invisible to WebGPU compute shaders. Reaching them requires a different abstraction: one that expresses computation at the graph level (operators and tensors rather than WGSL kernels) and allows the browser to choose, at runtime, whether a conv2d + relu + maxpool chain executes on the NPU with on-device fusion, or falls back to a WGSL compute dispatch, or runs through a CPU SIMD library.

**WebNN** (Web Neural Network API) was designed to fill exactly this gap. Rather than exposing a shader execution model, it exposes a **computational graph** model: the developer describes operations (matrix multiplication, 2D convolution, softmax, layer normalisation) at a logical level, and the browser translates that graph to whatever native ML runtime the platform supports — DirectML on Windows, Core ML on macOS and iOS, NNAPI on Android, TensorFlow Lite with XNNPACK on Linux and cross-platform fallback paths, and eventually hardware NPU stacks as they mature.

### Historical Context

The W3C Machine Learning Working Group (MLWG) was chartered in April 2021 ([charter](https://www.w3.org/2021/04/web-machine-learning-charter.html)) specifically to develop WebNN as a first-class web standard, separate from the earlier Web Machine Learning Community Group. The first Working Draft appeared in late 2021. The spec reached **Candidate Recommendation Snapshot** status on 11 April 2024 and was further updated to a **Candidate Recommendation Draft** (CRD) on 22 January 2026 incorporating over 100 significant changes — including a third wave of operators targeting transformer architectures, the `MLTensor` buffer-sharing API, and a redesigned device-selection mechanism. The most recent published version as of this writing is dated 21 May 2026 ([W3C TR](https://www.w3.org/TR/webnn/)). The editor's draft lives at `https://webmachinelearning.github.io/webnn/`. Once two independent implementations pass the W3C Web Platform Test suite for WebNN, the specification can advance to full Recommendation.

Chromium's implementation is the most complete to date. It is a collaborative effort from engineers at Google, Intel, and Microsoft. Chromium has supported WebNN behind a flag since approximately Chrome 112 and entered an **origin trial** in Chrome 146. As of mid-2026 WebNN has not yet reached the Chrome stable channel without a flag; developers who want to ship against it today must either use the origin trial token or instruct users to enable `#web-machine-learning-neural-network` in `chrome://flags`.

WebNN is designed as a low-level building block, not a user-facing framework. It sits beneath TensorFlow.js, ONNX Runtime Web, and similar libraries as an execution provider, in the same way that WebGL or WebGPU sits beneath Three.js.

---

## 2. The WebNN API Surface

### 2.1 ML Interface and Context Creation

WebNN is accessed via a new property on `Navigator` and `WorkerNavigator` named `ml`:

```webidl
// From the WebNN specification (CRD 21 May 2026)
// https://www.w3.org/TR/webnn/#api-ml
interface mixin NavigatorML {
  [SecureContext, SameObject] readonly attribute ML ml;
};
Navigator includes NavigatorML;
WorkerNavigator includes NavigatorML;
```

The `ML` interface exposes two overloaded `createContext()` methods:

```webidl
// https://www.w3.org/TR/webnn/#api-ml
interface ML {
  Promise<MLContext> createContext(optional MLContextOptions options = {});
  Promise<MLContext> createContext(GPUDevice gpuDevice);
};

dictionary MLContextOptions {
  MLPowerPreference powerPreference = "default";
  boolean accelerated = true;
};

enum MLPowerPreference {
  "default",
  "high-performance",
  "low-power",
};
```

Two points deserve emphasis for engineers coming from earlier WebNN documentation or from the ONNX Runtime Web API surface.

**`deviceType` was removed from `MLContextOptions`.** Earlier drafts of the WebNN specification included an `MLDeviceType` enum with values `"cpu"`, `"gpu"`, and `"npu"`. This was removed from the web-facing API in favour of the current `accelerated` / `powerPreference` model ([device-selection-explainer](https://github.com/webmachinelearning/webnn/blob/main/device-selection-explainer.md)). The rationale is that implementations and operating systems better understand system state than web applications: exposing a direct `"gpu"` vs. `"npu"` choice requires the browser to promise semantics it cannot guarantee portably. Instead, the developer expresses a preference:

- `accelerated: true` (the default) — the implementation should prefer parallel compute hardware (GPU or NPU) if available.
- `accelerated: false` — CPU-only execution is acceptable.
- `powerPreference: "high-performance"` — typically maps to GPU.
- `powerPreference: "low-power"` — typically maps to NPU (where available) or CPU.

The `accelerated` attribute on the returned `MLContext` object reflects whether the implementation considers this context to be hardware-accelerated.

Note: ONNX Runtime Web's WebNN execution provider still accepts a `deviceType` parameter in its own configuration interface (Section 8.2), but that is a framework-level option that the ORT library maps to the `accelerated` / `powerPreference` combination internally — it is not part of the `MLContextOptions` dictionary.

The second `createContext()` overload accepts an existing `GPUDevice`. This is the WebGPU interoperability path discussed in Section 2.6.

### 2.2 MLGraphBuilder and the Computation Graph Model

All WebNN computation is expressed as a **directed acyclic graph (DAG)** of ML operators. Unlike WebGPU, where the developer writes executable shader code, WebNN developers describe the graph structure and let the browser compile it to the optimal native representation. The `MLGraphBuilder` interface is the factory for graph nodes:

```javascript
// Feature detection and context creation
if (!('ml' in navigator)) {
  throw new Error('WebNN not supported in this browser');
}

const context = await navigator.ml.createContext({
  powerPreference: 'high-performance',
  accelerated: true,
});

const builder = new MLGraphBuilder(context);
```

`MLGraphBuilder` methods return `MLOperand` objects representing the outputs of operations. These operands are composable: the output of one operation becomes the input of the next, building up a computation graph lazily.

Once the graph is fully described, `builder.build()` compiles it asynchronously, returning a `Promise<MLGraph>`:

```javascript
// The compiled graph is an immutable object ready for repeated dispatch
const graph = await builder.build({ output: outputOperand });
```

`MLGraph` is immutable after build and should be reused across inference calls.

### 2.3 Operator Coverage

The WebNN specification defines 95 operators organised into functional categories. As of the CRD dated 21 May 2026 ([W3C TR](https://www.w3.org/TR/webnn/#api-mlgraphbuilder)), the full `MLGraphBuilder` operator set includes:

**Tensor manipulation:** `concat`, `expand`, `gather`, `gatherElements`, `gatherND`, `scatterElements`, `scatterND`, `where`, `pad`, `reshape`, `slice`, `split`, `transpose`, `resample2d`, `reverse`, `tile`, `triangular`

**Casting and quantisation:** `cast`, `quantizeLinear`, `dequantizeLinear`

**Element-wise binary:** `add`, `sub`, `mul`, `div`, `max`, `min`, `pow`

**Element-wise unary:** `abs`, `ceil`, `cos`, `erf`, `exp`, `floor`, `identity`, `log`, `neg`, `reciprocal`, `sign`, `sin`, `sqrt`, `tan`

**Logical operations:** `equal`, `notEqual`, `greater`, `greaterOrEqual`, `lesser`, `lesserOrEqual`, `logicalNot`, `logicalAnd`, `logicalOr`, `logicalXor`

**Linear algebra:** `matmul`, `gemm`

**Convolution:** `conv2d`, `convTranspose2d`

**Pooling:** `averagePool2d`, `l2Pool2d`, `maxPool2d`

**Activation functions:** `clamp`, `elu`, `gelu`, `hardSigmoid`, `hardSwish`, `leakyRelu`, `linear`, `prelu`, `relu`, `sigmoid`, `softmax`, `softplus`, `softsign`, `tanh`

**Normalisation:** `batchNormalization`, `instanceNormalization`, `layerNormalization`

**Reduction:** `argMin`, `argMax`, `cumulativeSum`, `reduceL1`, `reduceL2`, `reduceLogSum`, `reduceLogSumExp`, `reduceMax`, `reduceMean`, `reduceMin`, `reduceProduct`, `reduceSum`, `reduceSumSquare`

**Recurrent:** `gru`, `gruCell`, `lstm`, `lstmCell`

A practical example — building a two-layer MLP with `matmul`, bias-`add`, and `relu`:

```javascript
// Shape: [batch=1, features=784]
const input = builder.input('input', { dataType: 'float32', shape: [1, 784] });

// Layer 1: 784 → 256
const w1 = builder.constant(
  { dataType: 'float32', shape: [784, 256] },
  new Float32Array(layer1Weights),    // pre-loaded weights
);
const b1 = builder.constant(
  { dataType: 'float32', shape: [256] },
  new Float32Array(layer1Bias),
);
const hidden = builder.relu(
  builder.add(builder.matmul(input, w1), b1)
);

// Layer 2: 256 → 10
const w2 = builder.constant(
  { dataType: 'float32', shape: [256, 10] },
  new Float32Array(layer2Weights),
);
const b2 = builder.constant(
  { dataType: 'float32', shape: [10] },
  new Float32Array(layer2Bias),
);
const logits = builder.add(builder.matmul(hidden, w2), b2);
const output = builder.softmax(logits, 1);

const graph = await builder.build({ probabilities: output });
```

The constants (weights and biases) are embedded in the compiled `MLGraph` object and need not be passed at inference time.

### 2.4 MLTensor and Buffer Management

The `MLTensor` interface, added in Chrome M124 and now specified in the CRD, addresses the main performance bottleneck of the earlier `compute()` API: the need to copy input data from JavaScript `ArrayBuffer` objects into the hardware accelerator on every inference call.

`MLTensor` is an opaque device-specific allocation managed by the browser. Data moves in and out through explicit asynchronous operations:

```javascript
// Allocate a reusable input tensor on the device
const inputTensor = await context.createTensor({
  dataType: 'float32',
  shape: [1, 784],
  writable: true,   // JavaScript can upload data
  readable: false,  // JavaScript does not need to read it back
});

// Allocate a reusable output tensor on the device
const outputTensor = await context.createTensor({
  dataType: 'float32',
  shape: [1, 10],
  writable: false,
  readable: true,   // JavaScript will read inference results
});
```

Uploading a batch of input data:

```javascript
// writeTensor is fire-and-forget; it enqueues the upload
// on the context's internal timeline without blocking JavaScript
const pixelData = new Float32Array(784);
// ... fill pixelData from <canvas>, video frame, etc. ...
context.writeTensor(inputTensor, pixelData);
```

Reading results back:

```javascript
// readTensor returns a Promise that resolves when
// all preceding dispatches have completed and data is available
const results = await context.readTensor(outputTensor);
const probabilities = new Float32Array(results);
console.log('Predicted class:', probabilities.indexOf(Math.max(...probabilities)));
```

Tensors should be explicitly destroyed when no longer needed:

```javascript
inputTensor.destroy();
outputTensor.destroy();
graph.destroy();
```

### 2.5 dispatch() and the Execution Timeline

The older `compute()` method (present before September 2024) accepted `ArrayBuffer`-backed inputs and outputs directly and returned a `Promise` that resolved when inference completed. It has been **replaced by `dispatch()`** ([spec change September 2024](https://www.w3.org/TR/webnn/)), which works with `MLTensor` objects rather than raw buffers.

`dispatch()` is synchronous from JavaScript's perspective — it enqueues work on the context's internal GPU/device timeline and returns immediately, without waiting for the operation to complete:

```javascript
// Enqueue inference — returns void, does not await
context.dispatch(graph,
  { input: inputTensor },      // named inputs
  { probabilities: outputTensor } // named outputs
);

// You can enqueue multiple dispatches; they form a pipeline
context.dispatch(postProcessGraph,
  { logits: outputTensor },
  { classes: finalTensor }
);

// Only readTensor actually waits for the timeline to drain
const rawOutput = await context.readTensor(finalTensor);
```

This pipelining behaviour is essential for high-throughput applications such as real-time video frame processing, where inference on frame N can overlap with pre-processing of frame N+1.

The `MLContext` interface also exposes `opSupportLimits()` for querying what data types and shapes each operator supports on this context, enabling applications to adapt gracefully to hardware capability differences. The `lost` attribute is a `Promise<MLContextLostInfo>` that resolves if the hardware context is lost (analogous to `webglcontextlost`).

### 2.6 WebGPU Interoperability

The second `createContext()` overload accepts a `GPUDevice`:

```javascript
// Prerequisite: a WebGPU device is already initialised
const adapter = await navigator.gpu.requestAdapter();
const gpuDevice = await adapter.requestDevice();

// Create a WebNN context sharing the same underlying GPU device
const mlContext = await navigator.ml.createContext(gpuDevice);
const builder = new MLGraphBuilder(mlContext);
```

When a `GPUDevice` is passed, the resulting `MLContext` is bound to that GPU. This enables zero-copy interoperability: `MLTensor` objects can be created with `exportableToGPU: true`, and the browser can export them to WebGPU as `GPUBuffer` objects without any data copy across the PCIe bus:

```javascript
// Create a tensor whose backing storage can be shared with WebGPU
const sharedTensor = await mlContext.createTensor({
  dataType: 'float32',
  shape: [1, 3, 224, 224],
  writable: true,
  readable: false,
  exportableToGPU: true,
});

// After inference, export to a GPUBuffer for WebGPU post-processing
const gpuBuffer = await mlContext.exportToGPU(sharedTensor);

// Use the GPUBuffer in a WebGPU render or compute pass
const commandEncoder = gpuDevice.createCommandEncoder();
// ... bind gpuBuffer as a storage buffer ...
gpuDevice.queue.submit([commandEncoder.finish()]);

// Return the buffer to WebNN ownership before the next dispatch
gpuBuffer.destroy();
```

**Note: `exportToGPU()` support is not yet available on all platforms.** It has been prototyped and is tracked in the specification; the `exportableToGPU` descriptor flag is the web-facing signal that the application requests this capability. On Linux, whether this path is available depends on whether the GPU backend is wired up for the target hardware (see Section 4).

---

## 3. Chromium Implementation

### 3.1 Process Layout and services/webnn/ Directory Structure

Chromium's WebNN implementation follows the same process-isolation model used for WebGPU and the GPU command buffer (Chapter 33). The implementation is spread across three source areas:

- **`third_party/blink/renderer/modules/ml/`** — Blink-side JavaScript bindings. These implement the `navigator.ml` property and translate JavaScript calls into Mojo messages sent to the browser process.
- **`services/webnn/`** — The WebNN service, which runs in the GPU process. This is where backend selection, graph compilation, tensor management, and actual inference dispatch live.
- **`services/webnn/public/mojom/`** — The Mojo interface definition language files that describe the typed IPC channel between the renderer and the WebNN service.

The `services/webnn/` directory structure as of Chromium HEAD ([source.chromium.org](https://chromium.googlesource.com/chromium/src/+/HEAD/services/webnn/)):

```
services/webnn/
├── coreml/          # Apple Core ML backend (macOS/iOS)
├── host/            # Common host-side infrastructure
├── ort/             # ONNX Runtime (Windows, Windows 11 24H2+)
├── public/
│   └── mojom/       # Mojo interface definitions
│       ├── webnn_context.mojom
│       ├── webnn_context_provider.mojom
│       ├── webnn_graph.mojom
│       ├── webnn_graph_builder.mojom
│       ├── webnn_tensor.mojom
│       └── ... (15 .mojom files total)
├── tflite/          # TFLite/LiteRT backend (Linux, ChromeOS, Android, Windows fallback)
├── webnn_context_impl.cc        # MLContext service implementation
├── webnn_context_provider_impl.cc  # Backend selection logic
├── webnn_graph_builder_impl.cc  # MLGraphBuilder service implementation
├── webnn_graph_impl.cc          # MLGraph compilation and dispatch
├── webnn_tensor_impl.cc         # MLTensor service implementation
├── gpu_task_scheduler.cc        # GPU-side task scheduling
└── README.md
```

There is **no Dawn or WebGPU backend directory** in `services/webnn/`. On Linux, the TFLite backend (`services/webnn/tflite/`) is the primary inference engine. The WebGPU relationship is through the `GPUDevice`-based `createContext()` overload at the API level (Section 2.6), not through a shared Dawn code path.

### 3.2 Backend Selection and MLContextOptions

The backend selection logic lives in `services/webnn/webnn_context_provider_impl.cc` ([source.chromium.org](https://chromium.googlesource.com/chromium/src/+/HEAD/services/webnn/webnn_context_provider_impl.cc)). The implementation uses a `mojom::Device` enum internally (`kCpu`, `kGpu`, `kNpu`) that corresponds to the user-visible `powerPreference` / `accelerated` option pair, even though `deviceType` no longer appears in the web API.

The backend selection order, derived from the source:

| Platform | Backend preference (in order) |
|---|---|
| **Windows** | ONNX Runtime (Windows 11 24H2+ with appropriate EP) → DirectML → TFLite/XNNPACK fallback |
| **macOS / iOS** | Core ML (if enabled and not incognito) → TFLite/XNNPACK fallback |
| **Android** | NNAPI → TFLite/XNNPACK fallback |
| **Linux / ChromeOS** | TFLite/XNNPACK (CPU) |

The relevant code pattern for context creation:

```cpp
// Simplified from services/webnn/webnn_context_provider_impl.cc
// https://source.chromium.org/chromium/chromium/src/+/HEAD:services/webnn/webnn_context_provider_impl.cc

void WebNNContextProviderImpl::CreateWebNNContext(
    mojom::CreateContextOptionsPtr options,
    CreateWebNNContextCallback callback) {

  // Platform-specific backend selection
#if BUILDFLAG(IS_WIN)
  if (ShouldUseOrtBackend(options->device)) {
    gpu_host_->EnsureWebNNExecutionProvidersReady(
        base::BindOnce(&CreateOrtContext, ...));
    return;
  }
#elif BUILDFLAG(IS_APPLE)
  if (base::FeatureList::IsEnabled(features::kWebNNCoreML) &&
      !params.is_incognito) {
    CreateCoreMLContext(std::move(options), std::move(callback));
    return;
  }
#endif

  // Fallback: TFLite backend (enabled on Linux, ChromeOS, Android, Windows)
  if (base::FeatureList::IsEnabled(features::kWebNNUseTFLite)) {
    CreateTFLiteContext(std::move(options), std::move(callback));
    return;
  }

  // Last resort: fall back to in-renderer XNNPACK
  std::move(callback).Run(mojom::CreateContextResult::kFallbackToInProcess,
                           mojo::NullRemote());
}
```

The `kFallbackToInProcess` path (`webnn_context_provider_in_renderer.cc`) is a sandboxed in-renderer XNNPACK path that avoids the GPU process IPC hop. On Linux, this path requires `cpuinfo` to be initialised before the renderer's `seccomp-BPF` sandbox is installed, because XNNPACK uses `/proc/cpuinfo` to detect SIMD features (AVX2, AVX-512, etc.) — access that is blocked inside the sandbox.

### 3.3 CPU Backend: TFLite and XNNPACK on Linux

The Linux inference path goes through `services/webnn/tflite/`, which implements a complete WebNN backend on top of **TensorFlow Lite** (now branded **LiteRT**) with the **XNNPACK** delegate.

The TFLite backend has two main files at its core:

- **`graph_builder_tflite.cc`** — Translates the WebNN operator graph (received from the Blink renderer as a Mojo struct) into a TFLite `FlatBuffer` model. It performs quantisation op fusion: when it encounters a dequantize → op → quantize pattern in the graph, it fuses these into a single quantised TFLite operator where the TFLite op supports quantised types. This fusion is important for INT8 quantised models (common for efficient on-device inference) and is handled entirely at the TFLite graph-building stage.

- **`graph_impl_tflite.cc`** — Holds the compiled TFLite `Interpreter` and handles inference dispatch. If a quantised operator in the source graph lacks matching quantisation parameters (a valid but degenerate case), the implementation falls back to dequantising operands before dispatch.

XNNPACK is the TFLite default CPU inference delegate. It is a highly optimised library from Google that implements neural network operators using hand-tuned SIMD kernels for ARM NEON, x86 AVX2, and AVX-512. On a modern x86-64 Linux workstation, XNNPACK typically achieves within a factor of 2–3× of a native PyTorch CPU forward pass for the same quantised model.

TFLite also supports **GPU delegate** and **NNAPI delegate** paths that the WebNN backend can in principle enable for GPU-accelerated inference. On Linux, however, these delegates rely on OpenGL ES (the GPU delegate) or on the Android NNAPI HAL (unavailable on desktop Linux), so the effective path on desktop Linux is XNNPACK CPU inference. **There is no GPU-accelerated WebNN inference path on desktop Linux as of mid-2026** through the TFLite backend.

The `context_impl_tflite.cc` and `context_impl_litert.cc` files manage the LiteRT interpreter lifecycle — the "LiteRT" variants correspond to the rebranded TFLite as Google transitions from the TensorFlow umbrella to the LiteRT standalone SDK.

### 3.4 The Mojo IPC Path

The flow of a WebNN inference call through Chromium's process model:

```
JavaScript (renderer process)
   │
   │  navigator.ml.createContext()
   ▼
Blink MLContext (third_party/blink/renderer/modules/ml/)
   │
   │  Mojo: mojom::WebNNContextProvider::CreateWebNNContext()
   │  (pipe: renderer → GPU process via browser process broker)
   ▼
WebNNContextProviderImpl (services/webnn/, GPU process)
   │  backend selection → TFLite on Linux
   │
   │  Returns: mojo::Remote<mojom::WebNNContext>
   ▼
WebNNContextImpl (services/webnn/webnn_context_impl.cc, GPU process)
   │
   │  mojom::WebNNGraphBuilder::CreateGraph() (graph definition)
   │  → WebNNGraphBuilderImpl → graph_builder_tflite.cc
   │  → TFLite FlatBuffer compilation
   │
   │  Returns: mojo::Remote<mojom::WebNNGraph>
   ▼
WebNNGraphImpl (services/webnn/webnn_graph_impl.cc, GPU process)
   │
   │  mojom::WebNNContext::Dispatch()
   │  (input MLTensor handles passed by Mojo message)
   │  → graph_impl_tflite.cc → TFLite Interpreter.Invoke()
   │  → XNNPACK delegate → CPU SIMD inference
   │
   │  Result available in output MLTensor
   ▼
Blink (renderer process)
   │  context.readTensor(outputTensor) → ArrayBuffer
```

Tensor data passes between the renderer and the GPU process via one of two mechanisms selected by the feature flag `kWebNNUseDataPipe`:

- **Mojo Data Pipes** (`write_tensor_consumer_` / `read_tensor_producer_`) — for large tensor payloads, avoiding copies through the pipe itself.
- **BigBuffer** — inline Mojo buffer for smaller tensors, with the threshold set to avoid per-message overhead on small activations.

The IPC interfaces (`webnn_context.mojom`, `webnn_graph.mojom`, `webnn_tensor.mojom`) are defined in `services/webnn/public/mojom/`. The full list of `.mojom` files includes 15 interface definitions covering the context provider, context, graph builder, graph, tensor, compiler service, model loader, device, error types, and service introspection.

### 3.5 MLTensor and SharedImage Buffer Sharing

When a WebNN context is created from a `GPUDevice` (Section 2.6), `MLTensor` objects can be backed by **`gpu::SharedImage`** — the same cross-process GPU texture and buffer abstraction used by ANGLE, Dawn, and the Viz compositor (Chapter 36).

The Chromium implementation for this path uses `CreateTensorFromMailbox()` in `webnn_context_impl.cc`:

```cpp
// From services/webnn/webnn_context_impl.cc (simplified)
// Establishes a SharedImage-backed MLTensor binding
void WebNNContextImpl::CreateTensorFromMailbox(
    const gpu::Mailbox& mailbox,
    CreateTensorFromMailboxCallback callback) {
  // Requires GPU process SharedImageManager
  // Produces a WebNN tensor backed by the SharedImage
  auto tensor = ProduceWebNNTensor(mailbox, ...);
  // Graph constants cannot use interop tensors (read-only restriction)
  ...
}
```

The mailbox-based path allows a `GPUBuffer` allocation made by Dawn (for WebGPU) to be exposed as an `MLTensor` input without copying data back through the Mojo pipe to the renderer and then re-uploading it. This is the zero-copy GPU pipeline envisioned for real-time video inference: a WebGPU compute shader preprocesses a video frame into a `GPUBuffer`, which is then exported to WebNN as an `MLTensor` input for the inference pass.

---

## 4. The Linux GPU Inference Situation

### 4.1 What the GPU Backend Actually Means on Linux

The distinction between `accelerated: true` and `accelerated: false` in `MLContextOptions` maps to different hardware on different platforms. On Linux desktop:

| `MLContextOptions` | Effective Linux backend | Notes |
|---|---|---|
| `{ accelerated: true, powerPreference: 'high-performance' }` | TFLite/XNNPACK (CPU) | No GPU delegate wired on Linux |
| `{ accelerated: true, powerPreference: 'low-power' }` | TFLite/XNNPACK (CPU) | NPU path also falls back to CPU |
| `{ accelerated: false }` | TFLite/XNNPACK (CPU) | Same result |

As the `webnn.io` compatibility matrix states explicitly: "No native GPU backend today; execution remains on CPU via XNNPACK" for Linux, and NPU support "Not supported; falls back to CPU" ([webnn.io/en/api-reference/browser-compatibility/api](https://webnn.io/en/api-reference/browser-compatibility/api)).

This is a significant practical limitation compared to the Windows and macOS paths, where DirectML and Core ML respectively provide GPU and NPU access. Linux developers using WebNN today get XNNPACK CPU inference. This is still useful — XNNPACK is highly optimised and consistently outperforms naively written WebGPU compute shaders for CPU-bound inference — but it does not compete with the GPU-accelerated paths available on other platforms.

**Note: needs verification.** The situation may change as the LiteRT GPU delegate stabilises on Linux and as Chromium developers wire it up in `services/webnn/tflite/`. Watch the [webmachinelearning/webnn-status](https://webmachinelearning.github.io/webnn-status/) tracker for updates.

### 4.2 WebGPU Context Interop: createContext(GPUDevice)

The `createContext(GPUDevice)` path (Section 2.6) is the real connection between WebNN and Dawn/Vulkan on Linux. When a developer creates a WebNN context from a `GPUDevice`, the resulting `MLContext` is conceptually bound to the same GPU device that Dawn uses for WebGPU dispatch. This means:

1. The browser can in principle schedule WebNN inference and WebGPU compute passes on the same Vulkan queue, sharing synchronisation primitives.
2. `MLTensor` objects marked `exportableToGPU: true` can be backed by Vulkan buffer memory allocated by Dawn's memory management layer.
3. Zero-copy pipelines (WebGPU preprocessing → WebNN inference → WebGPU postprocessing) become possible without touching CPU memory.

Whether the actual inference work (the WebNN graph execution) dispatches through GPU compute shaders or still falls back to TFLite/XNNPACK on the CPU depends on the platform. On Linux, even with a `GPUDevice`-derived context, the inference today still executes on the CPU through TFLite, with only the buffer storage potentially shared through `SharedImage`. The `GPUDevice` context primarily enables the buffer-sharing interop; it does not automatically route inference through Vulkan compute. **Note: needs verification** for future Chromium releases.

### 4.3 Relation to Dawn and Mesa Vulkan

Dawn (Chapter 35) has **no WebNN backend or WebNN adapter**. A search of the Dawn repository (`dawn.googlesource.com/dawn`) finds no WebNN-specific code paths. Dawn implements the WebGPU API; WebNN is implemented in Chromium's `services/webnn/` service and uses its own backend chain.

The relationship between WebNN and Dawn on Linux is indirect:

1. When `createContext(GPUDevice)` is used, the `GPUDevice` is a Dawn-managed Vulkan device wrapping a Mesa Vulkan driver (RADV for AMD, ANV for Intel, NVK for NVIDIA).
2. `MLTensor` objects backed by `gpu::SharedImage` share the same underlying Vulkan memory allocator and synchronisation infrastructure that Dawn uses.
3. The actual inference work for WebNN on Linux does not go through Dawn's WGSL → SPIR-V → Mesa Vulkan path. It goes through TFLite/XNNPACK.

Putting it plainly for Linux graphics stack engineers: **WebNN and WebGPU share a GPU device at the resource level on Linux, but not a compute dispatch path**. There is no "WebNN dispatches WGSL compute shaders through Dawn" scenario.

### 4.4 NVIDIA: Windows Acceleration vs Linux Gap

NVIDIA's position in the WebNN ecosystem is sharply split by platform.

#### NVIDIA on Windows (TensorRT-RTX via Windows ML)

On Windows, NVIDIA has the best-performing WebNN path available in any browser. The chain is:

```
Browser WebNN API
  → Chromium services/webnn/ort/  (Windows 11 24H2+)
  → Windows ML / ONNX Runtime system component
  → TensorRT-RTX Execution Provider  (RTX 30 / Ampere+)
  → CUDA + TensorRT graph compiler
  → NVIDIA GPU
```

Microsoft delivered this path via **KB5096142** (Windows Update, 2025), which ships the NVIDIA TensorRT-RTX ONNX Runtime execution provider (v2.2605.1.0) to Windows 11 24H2 and 25H2 systems as a system component — meaning it is available to any application using Windows ML/ORT without requiring a separate NVIDIA SDK install. [Source](https://onnxruntime.ai/docs/execution-providers/TensorRTRTX-ExecutionProvider.html)

TensorRT-RTX delivers approximately **50% higher throughput** compared to the DirectML execution provider on NVIDIA RTX GPUs, because TensorRT's ahead-of-time graph compilation fuses operators and selects CUDA kernels at a level that DirectML's shader-based approach cannot match. Supported GPU generation: Ampere (RTX 30xx) and later. TensorRT-RTX also integrates with Windows ML's just-in-time compilation for deployment to end-user devices where the exact model structure may not be known in advance.

On Windows, this means an Transformers.js or ORT Web application using `device: 'webnn-gpu'` on an RTX 3070 laptop will transparently use CUDA-accelerated TensorRT inference without any NVIDIA SDK visible to the web developer.

#### NVIDIA on Linux (no WebNN path)

On Linux, NVIDIA has no contribution to the WebNN stack:

- **No VA-API in Chrome** (Chapter 147, §11): Chrome's video decode falls back to software on NVIDIA Linux; hardware video uses Firefox + `nvidia-vaapi-driver`
- **No WebNN GPU/NPU backend**: the `services/webnn/tflite/` backend runs on CPU via XNNPACK regardless of GPU vendor. There is no CUDA or TensorRT backend for Chrome's WebNN service on Linux.
- **No ONNX Runtime TensorRT-RTX on Linux**: TensorRT-RTX is a Windows-only execution provider; the Linux ONNX Runtime build uses CUDA EP or TensorRT EP for native inference, but these are not wired into Chrome's sandboxed WebNN service.
- **WebGPU (not WebNN)** is the only GPU-accelerated ML inference path for NVIDIA on Linux in the browser — via Vulkan compute shaders through NVK (open-source) or the proprietary NVIDIA Vulkan driver. This goes through the WebGPU API and WGSL compute shaders, entirely bypassing WebNN.

The practical consequence: an RTX 4090 on Linux running Chrome gets the same WebNN inference performance as an Intel Celeron N4000 — both execute on XNNPACK. For GPU-accelerated ML inference on NVIDIA Linux today, the correct API is **WebGPU**, using ONNX Runtime Web's WebGPU execution provider or Transformers.js with `device: 'webgpu'`.

#### Why NVIDIA Has Not Prioritised a Linux WebNN Path

Five interlocking reasons explain the gap:

**1. The partnership is with Microsoft, not with Chromium.**
NVIDIA's browser-ML investment went into TensorRT-RTX as a [Windows ML execution provider](https://developer.nvidia.com/blog/deploy-ai-models-faster-with-windows-ml-on-rtx-pcs/). The [RTX Spark / Windows AI PC initiative](https://nvidianews.nvidia.com/news/nvidia-microsoft-windows-pcs-agents-rtx-spark) is a bilateral Microsoft–NVIDIA deal: Microsoft owns the Windows ML stack; NVIDIA contributed an EP optimised for its hardware. There is no Linux operating-system vendor in an analogous position — no entity that owns a "browser ML service layer" for Linux as Microsoft does for Windows.

**2. WebNN on Linux is a Chromium engineering problem, not an NVIDIA driver problem.**
On Windows the chain is `services/webnn/ → Windows ML API → TensorRT-RTX`. The OS API glues the pieces together. On Linux, building an equivalent path would require writing a `services/webnn/ → CUDA/TensorRT` backend inside Chromium's sandboxed GPU service process. That is the work of Chromium contributors, not of NVIDIA driver engineers. NVIDIA could supply a sandboxable inference library, but neither the Chromium project nor NVIDIA has committed to building that integration.

**3. NVIDIA's Linux answer for browser ML is WebGPU, which now works.**
Chrome 147–148 [added Linux NVIDIA WebGPU support](https://www.izendestudioweb.com/articles/2026/04/28/whats-new-in-webgpu-in-chrome-147-148-wgsl-linear-indexing-and-linux-nvidia-support/) via Vulkan, using NVIDIA's proprietary Vulkan driver. ONNX Runtime Web's WebGPU execution provider and Transformers.js `device: 'webgpu'` both run CUDA-like compute through WGSL shaders on RTX hardware today. A WebNN GPU backend for Linux would be a second GPU path to maintain on top of an already-working WebGPU path, with marginal incremental benefit for browser developers.

**4. Market math does not justify the investment.**
Linux desktop holds roughly 3–4 % of the PC market. The sub-segment of "NVIDIA GPU + browser ML" on Linux is a small fraction of that. NVIDIA's RTX revenue comes from Windows gamers and from enterprise Linux *CUDA* (servers, HPC, AI training) — neither population relies on in-browser WebNN. The RTX AI PC programme targets hundreds of millions of Windows laptops; Linux desktop is outside that addressable market.

**5. The same pattern is consistent across the Linux browser stack.**
NVIDIA does not provide a VA-API driver for Chrome on Linux (Chapter 147, §11 — the community wrote `nvidia-vaapi-driver` as a NVDEC→VA-API shim for Firefox). NVIDIA does not publish an open Mesa Vulkan driver analogous to Intel ANV or AMD RADV (NVK exists but is community-driven). The common thread: NVIDIA invests in Linux support where CUDA/enterprise compute is the use case, and defers browser standards integration unless a partner fills the gap.

| | NVIDIA Windows (RTX 30+) | NVIDIA Linux |
|---|---|---|
| WebNN GPU | **Yes** via TensorRT-RTX + Windows ML | **No** (CPU/XNNPACK only) |
| WebNN NPU | No | No |
| WebGPU ML | Yes (Vulkan/D3D12) | **Yes** (Vulkan, best current option) |
| Video decode (Chrome) | DirectX Video, DXVA | **No** (software fallback) |
| Video decode (Firefox) | DirectX Video | Yes — native Vulkan Video (Firefox 153, July 2026); previously nvidia-vaapi-driver shim |

---

## 5. Firefox Implementation

Firefox does not have a production WebNN implementation in Gecko as of mid-2026. The MDN compatibility data and the `webnn.io` browser compatibility matrix list no Firefox WebNN support.

An active open-source project, **rustnn**, is building a Rust implementation of the W3C WebNN specification with Firefox integration as a goal ([blog post, December 2025](https://blog.ziade.org/2025/12/17/building-rustnn-webnn-implementation-rust/)). The rustnn approach follows a three-layer architecture:

1. **Platform-independent graph representation** — WebNN graphs compile once into an internal format.
2. **Backend converters** — Transform the graph into ONNX format (for ONNX Runtime) or CoreML format.
3. **Executors** — Run the converted graph on the native runtime.

Rustnn achieves coverage of all 95 WebNN operators and validates against the W3C Web Platform Tests. As a proof of concept, the author demonstrated an image classification model running through a rustnn-backed WebNN implementation in a Firefox build. However, as the author explicitly notes, the current integration runs inference in the **content process** (the Gecko equivalent of the Blink renderer process) rather than in a dedicated sandboxed GPU process — "a big security hole" that would need to be addressed before shipping. The intended final architecture involves a dedicated AI GPU service process being designed by Paul Adenot; the WebNN implementation will use it once the process exists. An IPC layer between C++ and the Rust bridge does not yet exist in the proof-of-concept patches.

There is no `dom/webnn/` directory in the main Firefox source tree as of this writing. **Note: needs verification** — check `searchfox.org/mozilla-central` for the current state of Gecko WebNN work.

#### rustnn Executors and NVIDIA Support

rustnn's executor layer currently supports three backends:

| Executor | Platform | Status |
|---|---|---|
| ONNX Runtime | Cross-platform | Stable |
| CoreML | macOS | Stable |
| TensorRT-RTX | Windows (NVIDIA RTX 30+) | Work in progress |

The TensorRT-RTX executor uses a new Rust binding crate (`trtx-rs`) written by the rustnn author because the existing NVIDIA TensorRT Rust binding was five years old and unmaintained. Development is slower because it requires a separate Windows machine.

**For NVIDIA Linux specifically:** rustnn has no TensorRT or CUDA executor for Linux. However, Firefox's architecture provides a more direct future path than Chrome's: Firefox's wgpu-based WebGPU implementation uses Vulkan as its Linux backend, and Vulkan works natively with NVIDIA's proprietary Linux driver (unlike Chrome's WebNN, which is tied to OS-level API stacks — Windows ML on Windows, CoreML on macOS). When Firefox's sandboxed AI GPU process exists and rustnn (or a native Gecko implementation) is wired into it, a `createContext(GPUDevice)` path through wgpu → Vulkan → NVIDIA driver would give NVIDIA Linux users GPU-accelerated WebNN inference — something Chrome's architecture makes significantly harder to deliver.

#### Firefox 153: Vulkan Video Decode for NVIDIA Linux

Separately from WebNN, Firefox 153 (scheduled July 21, 2026) adds [native Vulkan Video decode for NVIDIA GPUs on Linux](https://goodtech.info/firefox-153-integration-decodage-video-vulkan-linux-nvidia/), covering H.264, H.265, AV1, and VP9 via the Vulkan backend within Firefox's RDD (Remote Data Decoder) process. This eliminates the `nvidia-vaapi-driver` shim that Chapter 147 §11 documents as the current workaround. The code was merged to Firefox Nightly as of June 2026.

This is not WebNN, but it signals a consistent direction: Firefox is investing in direct Vulkan-based NVIDIA Linux acceleration rather than relying on translation layers. The same architectural investment (wgpu, Vulkan process isolation) that enables Vulkan Video is the foundation a future NVIDIA-accelerated Firefox WebNN would build on.

Firefox's **wgpu**-based WebGPU implementation (Chapter 52) provides the GPU context that a future Firefox WebNN could use for the `createContext(GPUDevice)` interop path. The alignment here is architectural: just as Chromium's WebNN can share a Dawn `GPUDevice`, a future Firefox WebNN could share a wgpu `GPUDevice` with Gecko's WebGPU implementation.

---

## 6. NPU Backend and Linux

### 6.1 The deviceType Abstraction at Runtime

Although `MLContextOptions` no longer exposes `deviceType` on the web-facing API, Chromium's internal `mojom::Device` enum still has a `kNpu` variant. When a user passes `powerPreference: 'low-power'`, the implementation maps this to a preference for the NPU. On platforms where an NPU backend is implemented — currently Windows via ONNX Runtime's DirectML-NPU execution provider on Intel Core Ultra hardware — the NPU path is activated. On Linux, the mapping falls through to TFLite/XNNPACK.

### 6.2 The Linux NPU Landscape

Native NPU support on Linux is an active but immature area. The principal driver stacks in various states of Linux readiness as of mid-2026:

**Intel OpenVINO (NPU plugin):** Intel has upstreamed the `intel_vpu` kernel driver to the mainline Linux kernel for the NPU in Meteor Lake and later processors. The user-space OpenVINO runtime ([openvinotoolkit.org](https://github.com/openvinotoolkit/openvino), stable release 2026.1.0) includes a Linux NPU plugin. The plugin can run inference on Intel's AI Boost NPU directly from Linux user space. However, this path is not yet exposed through WebNN — it would require a Chromium backend analogous to the Windows ORT execution provider, which does not currently exist.

**AMD XDNA (Ryzen AI NPU):** AMD has merged the `amdxdna` accelerator driver into the mainline Linux kernel for Ryzen AI (XDNA and XDNA2) NPUs. AMD has also released the Peano open-source LLVM compiler targeting XDNA. User-space tooling (the Lemonade server with XDNA LLM support, ROCm AI compute) is beginning to mature ([Phoronix, 2025](https://www.phoronix.com/news/AMD-Ryzen-AI-NPUs-Linux-LLMs)). A WebNN backend targeting AMD XDNA on Linux does not exist.

**Qualcomm Hexagon (SNPE):** The Snapdragon X Elite and related processors include the Hexagon Tensor Processor. Qualcomm's SNPE (Snapdragon Neural Processing Engine) SDK supports Linux on Snapdragon hardware. Driver maturity for web-facing access is still limited.

### 6.3 Why the NPU Path Remains Proprietary-Driver-Dependent

WebNN's architecture requires a Chromium backend module (analogous to `services/webnn/ort/` on Windows or `services/webnn/coreml/` on macOS) that maps the WebNN Mojo graph to the native NPU runtime. On Windows, Microsoft's direct involvement in the ONNX Runtime and DirectML NPU paths made this feasible. On Linux, each NPU vendor has a different user-space stack, different security model, and different Linux kernel interface. Building sandboxed browser backends for each would require the same effort that went into the DirectML and CoreML backends — effort that has not yet been invested for Linux NPU targets.

The practical consequence for developers targeting Linux NPU via WebNN is: **it is not possible today**. The fallback to TFLite/XNNPACK means inference runs on the CPU regardless of NPU hardware present. See Chapter 88 (NPU and AI Accelerator Integration) for the native Linux NPU stack independent of the browser.

---

## 7. WebNN vs. WebGPU Compute for ML

Both WebNN and WebGPU are viable targets for in-browser ML inference. The right choice depends on the workload and the deployment target.

### 7.1 Abstraction Level

WebGPU gives the developer a **compute shader** execution model. To implement matrix multiplication, the developer writes a WGSL kernel that tiles the matrices, manages shared memory, and dispatches across a grid of workgroups. The benefit is total control over the kernel; the cost is that the developer must re-implement what every ML framework has already done.

WebNN gives the developer a **graph operator** model. The developer calls `builder.matmul(a, b)`, and the browser decides how to implement that on the current hardware. The browser can fuse `conv2d → relu → maxPool2d` into a single kernel, use INT8 quantised SIMD paths through XNNPACK, delegate to a CoreML accelerator with on-chip memory, or dispatch to a native NPU. The developer writes no shader code.

### 7.2 Portability and Operator Coverage

A WebGPU compute shader is portable across all platforms that implement WebGPU, but only at the execution-model level — a kernel that is fast on an Intel GPU may be slow on an Arm Mali GPU due to different tile sizes and warp widths. WebNN operators are portable at the graph level: the developer expresses intent, and the platform optimises.

As of mid-2026, WebNN covers 95 operators ([webmachinelearning.github.io/webnn-status/](https://webmachinelearning.github.io/webnn-status/)). This is sufficient for standard CNN architectures (ResNet, EfficientNet, MobileNet), transformers (BERT, ViT, Whisper), and most production-grade models, but gap-fills exist. Operations that fall outside the 95-operator set require a custom WebGPU dispatch fallback.

### 7.3 Performance Considerations

For standard operator chains, WebNN has two systematic advantages over hand-written WebGPU kernels:

1. **Operator fusion.** A conv2d followed by relu followed by max pooling, expressed as three separate WebGPU dispatches, incurs three kernel launches, three synchronisation points, and three rounds of memory traffic between the compute units and memory. A WebNN backend can fuse these into a single kernel, reducing memory round-trips by up to 3× for typical CNN blocks.

2. **SIMD paths on CPU.** On Linux, where WebNN currently falls back to XNNPACK, XNNPACK's hand-tuned AVX2 / AVX-512 kernels outperform WebGPU WGSL shaders running on the CPU path (SwiftShader or WASM SIMD) for the same operation. If the user's GPU is unavailable or blacklisted, WebNN's CPU fallback is typically faster than WebGPU's CPU fallback.

For **large language model inference** (attention-heavy, large matmuls, long context), the balance shifts. WebGPU gives complete control of the WGSL attention kernel, enabling FlashAttention-style fusion and KV-cache management patterns that WebNN does not yet express. Projects like **WebLLM** use WebGPU for this reason ([arxiv.org/abs/2412.15803](https://arxiv.org/abs/2412.15803)).

### 7.4 The "Best of Both" Pattern

A common production pattern combines both APIs:

1. **WebGPU** for pre-processing: decode a video frame, apply colour correction and normalisation using a WGSL compute shader.
2. **WebNN** (via `createContext(GPUDevice)`) for inference: run the detection or segmentation model.
3. **WebGPU** for post-processing: overlay bounding boxes, apply the segmentation mask as a texture.

This pipeline stays entirely on the GPU (or at least on the same device timeline) on platforms where the GPU buffer interop path is supported, avoiding the expensive GPU→CPU→GPU round-trip that would be required if the WebGPU and WebNN contexts were independent.

---

## 8. Developer Workflow

### 8.1 Feature Detection

The canonical WebNN feature detection check is:

```javascript
if ('ml' in navigator) {
  // WebNN is available in this browser
  const context = await navigator.ml.createContext();
  console.log('WebNN accelerated:', context.accelerated);
} else {
  // Fall back to WebGPU compute or WASM
  console.warn('WebNN not available, using fallback');
}
```

For production applications, also check `context.accelerated` to determine whether GPU/NPU acceleration is actually engaged, and use `context.opSupportLimits()` to verify that the operators your model requires are supported on this hardware:

```javascript
const context = await navigator.ml.createContext({ accelerated: true });
const limits = context.opSupportLimits();

// Example: check conv2d INT8 support
const conv2dLimits = limits.conv2d;
const supportsInt8 = conv2dLimits.input.dataTypes.includes('int8');
if (!supportsInt8) {
  console.warn('INT8 quantised conv2d not supported; using float32 fallback');
}
```

### 8.2 ONNX Runtime Web Integration

The most practical path for deploying real models through WebNN is **ONNX Runtime Web** (ORT Web). ORT Web provides a WebNN **execution provider** (EP) that translates the ONNX model graph into WebNN operator calls:

```javascript
import * as ort from 'onnxruntime-web/all';

// Configure WebNN as the execution provider
const sessionOptions = {
  executionProviders: [{
    name: 'webnn',
    // Note: this is an ORT-level configuration, not MLContextOptions.
    // ORT maps deviceType to the accelerated/powerPreference pair internally.
    deviceType: 'gpu',        // 'cpu' | 'gpu' | 'npu'
    powerPreference: 'default',
  }],
};

// Load and run the model
const session = await ort.InferenceSession.create(
  './model.onnx',
  sessionOptions,
);

const inputTensor = new ort.Tensor('float32', inputData, [1, 3, 224, 224]);
const results = await session.run({ input: inputTensor });
```

For IO-binding (keeping tensors on device across multiple inference calls):

```javascript
// Pre-create an MLContext and MLTensor for IO binding
// Use accelerated + powerPreference — deviceType is not part of MLContextOptions
const mlContext = await navigator.ml.createContext({
  accelerated: true,
  powerPreference: 'high-performance',
});
const inputMLTensor = await mlContext.createTensor({
  dataType: 'float32',
  shape: [1, 3, 224, 224],
  writable: true,
});

// Wrap in ORT Tensor for passing to the session
const inputTensor = ort.Tensor.fromMLTensor(inputMLTensor, {
  dataType: 'float32',
  dims: [1, 3, 224, 224],
});

// Request outputs remain as MLTensors (avoids readback to CPU)
const sessionOptions = {
  executionProviders: [{ name: 'webnn', context: mlContext }],
  preferredOutputLocation: 'ml-tensor',
};
```

The ORT Web WebNN EP requires the `ort-wasm-simd-threaded.jsep.wasm` file to be served alongside the application, since it relies on the JavaScript Execution Provider (JSEP) WASM module.

### 8.3 Polyfills and Tooling

For browsers that do not support WebNN (Firefox, older Chrome), the **`@webnn/polyfill`** package provides a pure-JavaScript implementation backed by WebGL or WebGPU dispatch:

```javascript
// In environments without native WebNN, install the polyfill
import '@webnn/polyfill';
// After this import, navigator.ml is available (backed by WebGPU/WebGL)
const context = await navigator.ml.createContext();
```

Note that polyfilled WebNN does not access NPU hardware or benefit from backend-level operator fusion — it simply maps WebNN calls back to WebGPU compute dispatches. The polyfill is a compatibility shim, not a performance shortcut.

For offline/Node.js development and server-side model validation, **`@webnn/node`** provides a Node.js binding to the WebNN C++ implementation, allowing models to be tested in a Node environment before deploying to the browser.

### 8.4 Chrome Flags and Enablement Status

As of mid-2026, WebNN is behind the `#web-machine-learning-neural-network` flag in `chrome://flags/`. The flag expires at Chrome Milestone 150, indicating it is under active development. To enable WebNN without the flag for a limited production deployment, developers can enrol in the WebNN **origin trial** (launched in Chrome 146).

To enable via the command line:

```bash
google-chrome --enable-features=WebMachineLearningNeuralNetwork
```

Backend-specific flags for testing non-default backends:

```
#webnn-onnxruntime       — ONNX Runtime backend (Windows)
#webnn-directml          — DirectML backend (Windows)
#webnn-coreml            — Core ML backend (macOS)
```

On Linux, these additional flags are not applicable; the TFLite/XNNPACK backend is used unconditionally.

The `chrome://gpu` page includes WebNN feature status in its output, and `about://gpu` in Edge shows analogous information. WebNN context errors and NPU driver issues can also appear in the log at the bottom of `about://gpu`.

**Chrome DevTools WebNN panel:** A dedicated DevTools panel for WebNN graph inspection has been prototyped but was not in stable DevTools as of this writing. **Note: needs verification.** Check the Chrome DevTools changelog for updates.

For a live view of browser support across Chrome, Firefox, and other browsers, the **Web Platform Status** tracker at [webstatus.dev/features/webnn](https://webstatus.dev/features/webnn) is the authoritative source.

### 8.5 Framework WebNN Support: TensorFlow.js, Transformers.js, ONNX Runtime Web

The W3C MLWG's stated architectural goal is for WebNN to sit beneath all JavaScript ML frameworks as a unified hardware-acceleration layer. The degree to which each major framework has implemented this varies significantly as of mid-2026.

#### ONNX Runtime Web (most mature)

As covered in §8.2, ORT Web has the most complete WebNN integration. The `executionProviders: ['webnn']` path is actively maintained, supports `MLTensor` IO binding to avoid GPU readback, and is the reference implementation that other frameworks build on top of. On Windows 11 24H2+, ORT Web's WebNN EP routes through the Windows ML ONNX Runtime system component, which on NVIDIA RTX hardware uses the TensorRT-RTX execution provider (§8.5 NVIDIA below).

#### Transformers.js (shipped since v3)

Transformers.js ([huggingface.co/docs/transformers.js](https://huggingface.co/docs/transformers.js)) uses ONNX Runtime Web as its inference backend, inheriting ORT Web's WebNN EP. WebNN support shipped in **Transformers.js v3** and carries forward in **v4** (released March 2025, which rewrote the WebGPU runtime but retained the WebNN path). [Source](https://huggingface.co/blog/transformersjs-v4)

```javascript
import { pipeline } from '@huggingface/transformers';

// WebNN with default device selection (browser chooses GPU/NPU/CPU)
const classifier = await pipeline('image-classification', 'model-id', {
  device: 'webnn',
});

// Explicit device preference (maps to powerPreference internally)
const classifierNpu = await pipeline('image-classification', 'model-id', {
  device: 'webnn-npu',   // prefers NPU (Copilot+ PC, Apple Silicon)
});
const classifierGpu = await pipeline('image-classification', 'model-id', {
  device: 'webnn-gpu',   // prefers GPU
});
const classifierCpu = await pipeline('image-classification', 'model-id', {
  device: 'webnn-cpu',   // forces CPU (XNNPACK on all platforms)
});
```

**Critical limitation — static shapes only**: WebNN does not currently support dynamic tensor shapes. Transformers.js requires `free_dimension_overrides` to be set for models with variable sequence lengths or batch sizes. In practice this means fixing the input shape at session creation time:

```javascript
const session = await ort.InferenceSession.create('./model.onnx', {
  executionProviders: ['webnn'],
  freeDimensionOverrides: {
    batch_size: 1,
    sequence_length: 128,   // must match actual input at inference time
  },
});
```

This is a significant ergonomic constraint for NLP models (variable sequence lengths) and detection models (variable number of detections). The WebNN spec working group is tracking dynamic shape support; it is not in the current Candidate Recommendation Draft.

On Linux, `device: 'webnn-gpu'` and `device: 'webnn-npu'` both fall back to TFLite/XNNPACK CPU inference (see §4.1). `device: 'webnn-cpu'` behaves identically. Transformers.js users on Linux should prefer `device: 'webgpu'` for GPU-accelerated inference until WebNN's Linux GPU backend matures.

#### TensorFlow.js (no WebNN backend)

TensorFlow.js ships four backend packages: `@tensorflow/tfjs-backend-webgl` (default), `@tensorflow/tfjs-backend-webgpu`, `@tensorflow/tfjs-backend-wasm`, and `@tensorflow/tfjs-node`. There is no `@tensorflow/tfjs-backend-webnn` package as of mid-2026, and no public roadmap entry for one has been found in the TF.js GitHub issue tracker or ROADMAP.md. [Source](https://github.com/tensorflow/tfjs)

The W3C MLWG architecture document describes TF.js as a future WebNN consumer, but the TF.js project itself has not committed to this. The practical route for TF.js users wanting WebNN acceleration is to **convert models to ONNX format and use ONNX Runtime Web** — TF.js model → `tensorflowjs_converter` → ONNX → ORT Web WebNN EP. This conversion is lossy for some ops and requires the SavedModel or keras format as input.

TF.js's continued investment in its WebGPU backend (via `@tensorflow/tfjs-backend-webgpu`) suggests the team's GPU acceleration strategy remains WebGPU-centric rather than WebNN-centric.

#### Framework support summary

| Framework | WebNN status | Implementation path | Dynamic shapes | Linux GPU |
|-----------|-------------|---------------------|---------------|-----------|
| ONNX Runtime Web | **Shipped** (production preview) | Native WebNN EP | No (static only) | CPU/XNNPACK fallback |
| Transformers.js v3+ | **Shipped** (via ORT Web EP) | ORT Web WebNN EP | No | CPU/XNNPACK fallback |
| TensorFlow.js | **Not implemented** | — | — | — |
| MediaPipe | **Partial** (via WASM + XNNPACK) | No WebNN EP; uses own WASM runtime | N/A | No |
| WebNN polyfill | **Full coverage** (software only) | WebGPU/WebGL dispatch | Yes | Via WebGPU |

The bottom line: as of mid-2026, WebNN in the browser is **production-ready only on Windows** (DirectML/TensorRT-RTX/Core ML on macOS) and specifically through ORT Web or Transformers.js. On Linux, it is a CPU-only path offering XNNPACK performance rather than GPU or NPU acceleration. The recommendation for cross-platform production deployment is to feature-detect WebNN and fall back to WebGPU compute:

```javascript
const hasWebNN = 'ml' in navigator;
const device = hasWebNN ? 'webnn' : 'webgpu';
const model = await pipeline('task', 'model', { device });
```

---

## Roadmap

### Near-term (6–12 months)

- **WebNN stable channel launch in Chrome:** The `#web-machine-learning-neural-network` flag expires at Chrome Milestone 150; Google has signalled intent to ship WebNN without a flag once the origin trial (Chrome 146–149) collects sufficient compatibility data. Two independent implementations passing the W3C Web Platform Test suite are the remaining gate for advancing the spec to full Recommendation. [Source](https://groups.google.com/a/chromium.org/g/blink-dev/c/5CWKSChYo98/m/xMw0U5NkAAAJ)
- **Chrome DevTools WebNN panel:** A dedicated DevTools panel for WebNN graph inspection has been prototyped, allowing developers to visualise the compiled operator graph and inspect tensor shapes at each node. Stabilisation and inclusion in a DevTools stable release is expected in the near term. [Source](https://chromium.googlesource.com/chromium/src/+/HEAD/services/webnn/)
- **Firefox WebNN implementation:** Mozilla's `rustnn` project (a Rust-based WebNN implementation) is under active development as the basis for Firefox's WebNN backend, with the `wgpu`-backed `GPUDevice` interop path being a focus. [Source](https://blog.ziade.org/2025/12/17/building-rustnn-webnn-implementation-rust/)
- **Dynamic shapes and tensor binding:** The W3C MLWG identified at TPAC 2025 that dynamic input shapes and a `bindInput()` / re-dispatch API are needed for transformer-based autoregressive generation (which changes sequence length at each decode step). Specification changes and Chromium implementation are expected in this window. [Source](https://www.w3.org/blog/2025/ai-at-tpac-2025/)
- **Expanded operator coverage for transformers:** The CRD of 21 May 2026 added a third wave of operators targeting transformer architectures; remaining gaps (RoPE embedding, grouped-query attention) are tracked in the core operator set issue and targeted for near-term closure. [Source](https://github.com/webmachinelearning/webnn/issues/573)

### Medium-term (1–3 years)

- **Linux NPU backend in Chromium:** The current Linux WebNN path uses TFLite/XNNPACK (CPU) unconditionally. As Intel OpenVINO (`intel_vpu` kernel driver) and AMD XDNA user-space runtimes mature on Linux, a `services/webnn/` Linux NPU backend analogous to the existing DirectML (Windows) and Core ML (macOS) backends is a stated goal. Note: needs verification on the exact timeline.
- **W3C WebNN full Recommendation:** Once two conformant implementations exist (Chromium on all major platforms, Firefox), the MLWG can advance to full W3C Recommendation, unlocking mandatory browser support and enabling frameworks like ONNX Runtime Web and TensorFlow.js to advertise WebNN as a stable execution provider. [Source](https://progosling.com/en/dev-digest/2026-02/webnn-candidate-recommendation)
- **Small Language Model (SLM) acceleration:** The MLWG is specifically optimising WebNN for in-browser SLMs (quantised 1–7B-parameter models). This involves graph-level quantisation metadata, `int4` / `int8` weight types in the buffer management API, and model-weight caching across page navigations. [Source](https://www.w3.org/blog/2025/ai-at-tpac-2025/)
- **WebMCP and agent integration:** The emerging WebMCP proposal, discussed alongside WebNN at W3C, would allow browser-based ML agents to call local model inference via WebNN as a typed capability channel — tying WebNN to the broader browser-native agentic computing story. [Source](https://speakerdeck.com/christianliebel/webnn-built-in-ai-webmcp-whats-new-in-web-ai)
- **Cross-browser Web Platform Test suite hardening:** Passing the WPT suite for all operator semantics (numerical tolerances, edge-case handling, async execution ordering) is a prerequisite for the W3C gate; this represents a sustained medium-term engineering effort across Google, Intel, Microsoft, and Mozilla. [Source](https://webstatus.dev/features/webnn)

### Long-term

- **Universal NPU support across Linux hardware:** The long-term vision is for any Linux device with a discrete NPU (Intel AI Boost, Qualcomm Hexagon, AMD XDNA) to be reachable from WebNN via a standardised kernel interface — analogous to how any V4L2-compliant camera is accessible from `getUserMedia`. This depends on kernel-driver and user-space-runtime standardisation that is years away. Note: needs verification.
- **WebNN as the canonical ML execution layer for browsers:** The architectural goal of the W3C MLWG is for WebNN to sit beneath all JavaScript ML frameworks (TensorFlow.js, ONNX Runtime Web, Transformers.js, MediaPipe) as a unified hardware-acceleration abstraction, replacing ad-hoc WebGPU compute shader libraries with a single API that the browser vendor optimises per platform. [Source](https://webmachinelearning.github.io/webnn-intro/)
- **Tight integration with WebGPU's roadmap:** As WebGPU adds subgroup operations and cooperative matrix operations (targeting ML workloads), the boundary between WebNN operator dispatch and WebGPU compute dispatch will become a design question. Long-term, the two APIs may share a common scheduling and memory model, allowing seamless operator-level mixing of WebNN-accelerated primitives with custom WebGPU kernels.
- **Standardised model-weight caching and offline inference:** Long-term, browser vendors are expected to add WebNN-aware model-weight caching (keyed by model hash) that survives page reload — reducing first-inference latency for recurring models without requiring a Service Worker workaround. Note: needs verification.

---

## 9. Integrations

- **Chapter 35 (Dawn and WebGPU)** — Dawn is Chrome's WebGPU implementation; it manages the `VkDevice` that a WebNN context created via `createContext(GPUDevice)` shares. Buffer interop between WebNN `MLTensor` objects and WebGPU `GPUBuffer` objects goes through the same `gpu::SharedImage` mailbox infrastructure that Dawn uses for WebGPU texture management. Dawn does not contain a WebNN implementation or WebNN backend, but it is the foundational GPU resource manager that makes zero-copy WebNN/WebGPU pipelines possible.

- **Chapter 33 (Chromium GPU Architecture)** — WebNN lives in the same GPU process as WebGPU, the command buffer, and Viz. `WebNNContextProviderImpl` is initialised alongside `GpuServiceImpl`. The Mojo IPC path, process sandbox model, and SharedImage system described in Chapter 33 are the shared infrastructure on which `services/webnn/` runs.

- **Chapter 36 (Chromium Compositor and Viz)** — The `gpu::SharedImage` backing store for interop `MLTensor` objects is the same SharedImage system used by Viz to manage compositor surfaces, ANGLE textures, and Dawn-allocated WebGPU buffers. Understanding the mailbox and synchronisation token model from Chapter 36 is a prerequisite for the WebNN buffer interop path.

- **Chapter 88 (NPU and AI Accelerator Integration)** — The native Linux NPU stacks (Intel OpenVINO, AMD XDNA, Qualcomm SNPE) described in Chapter 88 are the same stacks that a future Chromium Linux NPU backend for WebNN would sit on top of. Chapter 88 covers the kernel driver (`intel_vpu`, `amdxdna`) and user-space runtime layers in depth. Understanding those layers is prerequisite for anyone seeking to build a Linux NPU backend for `services/webnn/`.

- **Chapter 98 (WebAssembly and WebGPU as a Deployment Target)** — ONNX Runtime Web's WASM execution provider and its WebNN execution provider are complementary deployment targets. Chapter 98 covers the WASM+WebGPU path (the current workhorse for in-browser ML); this chapter covers the WebNN path (the emerging standard for hardware-portable inference). The two paths can be combined in a single application with a runtime feature check.

- **Chapter 52 (Firefox and WebRender)** — Firefox's wgpu-based WebGPU implementation could serve as the `GPUDevice` for a future Firefox WebNN `createContext(GPUDevice)` call. The architectural parallel between Chromium's Dawn-based WebGPU and a future Firefox WebNN integration is direct.

---

## 10. References

- [Web Neural Network API — W3C Candidate Recommendation Draft, 21 May 2026](https://www.w3.org/TR/webnn/)
- [WebNN Editor's Draft — webmachinelearning.github.io](https://webmachinelearning.github.io/webnn/)
- [WebNN Device Selection Explainer](https://github.com/webmachinelearning/webnn/blob/main/device-selection-explainer.md)
- [MLTensor Explainer](https://github.com/webmachinelearning/webnn/blob/main/mltensor-explainer.md)
- [Implementation Status of WebNN Operations](https://webmachinelearning.github.io/webnn-status/)
- [Web Platform Status: WebNN](https://webstatus.dev/features/webnn)
- [Chromium `services/webnn/` source directory](https://chromium.googlesource.com/chromium/src/+/HEAD/services/webnn/)
- [Chromium `webnn_context_provider_impl.cc`](https://chromium.googlesource.com/chromium/src/+/HEAD/services/webnn/webnn_context_provider_impl.cc)
- [Chromium WebNN Development notes (bbernhar)](https://gist.github.com/bbernhar/df6697aca3ad75f23148d7bef524f4c5)
- [WebNN Chromium Development: Intent to Experiment (Chrome 146)](https://groups.google.com/a/chromium.org/g/blink-dev/c/5CWKSChYo98/m/xMw0U5NkAAAJ)
- [WebNN Flags in Chromium-based Browsers](https://webnn.io/en/api-reference/browser-compatibility/chrome-flags)
- [WebNN Browser Compatibility Matrix](https://webnn.io/en/api-reference/browser-compatibility/api)
- [Chrome 146 Beta with WebNN Origin Trial — Phoronix](https://www.phoronix.com/news/Chrome-146-Beta)
- [WebNN Overview — Microsoft Learn](https://learn.microsoft.com/en-us/windows/ai/directml/webnn-overview)
- [Using WebNN with ONNX Runtime Web](https://onnxruntime.ai/docs/tutorials/web/ep-webnn.html)
- [rustnn: A Rust WebNN Implementation for Firefox — Tarek Ziadé](https://blog.ziade.org/2025/12/17/building-rustnn-webnn-implementation-rust/)
- [WebNN is the Future of Browser AI — Tarek Ziadé](https://blog.ziade.org/2025/11/21/why-webnn-is-the-future-of-ai-in-browsers/)
- [W3C Web Machine Learning Working Group Charter](https://www.w3.org/2023/04/web-machine-learning-charter.html)
- [WebLLM: High-Performance In-Browser LLM Inference (arXiv:2412.15803)](https://arxiv.org/abs/2412.15803)
- [AMD Ryzen AI NPUs on Linux — Phoronix](https://www.phoronix.com/news/AMD-Ryzen-AI-NPUs-Linux-LLMs)
- [Characterising WebGPU Dispatch Overhead for LLM Inference (arXiv:2604.02344)](https://arxiv.org/html/2604.02344)
- [Introduction to Web Neural Network API — webmachinelearning.github.io](https://webmachinelearning.github.io/get-started/2024/05/16/introduction-to-web-neural-network-api.html)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
