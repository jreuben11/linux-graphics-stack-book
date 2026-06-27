# Chapter 191 — LiteRT and MediaPipe: On-Device ML Inference on the Android Graphics Stack

> **Audiences:** Graphics and GPU developers who want to understand how ML inference shares `AHardwareBuffer` and GPU compute infrastructure with the Android rendering pipeline; Android application developers integrating on-device ML with camera and display workflows; and systems engineers bridging the ML and GPU subsystems on Android, including those evaluating how the delegate and hardware-accelerator model maps onto the kernel DRM/DMA-BUF and HAL abstractions described in earlier parts of this book.

---

## Table of Contents

1. [Why ML Inference Belongs in a Graphics Stack Book](#1-why-ml-inference-belongs-in-a-graphics-stack-book)
2. [LiteRT Architecture](#2-litert-architecture)
3. [Delegate Architecture: Routing Inference to Hardware](#3-delegate-architecture-routing-inference-to-hardware)
   - [3.1 NNAPI: Historical Context (API 27–34)](#31-nnapi-historical-context-api-2734) — HAL evolution, LiteRT delegate, why deprecated
   - [3.3 What Replaced NNAPI: Vendor Delegates](#33-what-replaced-nnapi-vendor-delegates) — QNN/HTP, Tensor NPU, MediaTek APU, discovery chain, op coverage
   - [3.5 XNNPack: CPU Inference with SIMD and Thread Parallelism](#35-xnnpack-cpu-inference-with-simd-and-thread-parallelism)
4. [AHardwareBuffer Tensor Interop](#4-ahardwarebuffer-tensor-interop)
5. [Model Formats and Quantisation](#5-model-formats-and-quantisation)
   - [5.6 ONNX: The Open Neural Network Exchange Format](#56-onnx-the-open-neural-network-exchange-format)
   - [5.7 ONNX Runtime Mobile on Android](#57-onnx-runtime-mobile-on-android)
   - [5.8 ExecuTorch: PyTorch's Native Mobile Runtime](#58-executorch-pytorchs-native-mobile-runtime)
   - [5.9 JAX and StableHLO: Export Path to LiteRT](#59-jax-and-stablehlo-export-path-to-litert)
6. [Model Conversion and Quantization Workflow](#6-model-conversion-and-quantization-workflow)
7. [MediaPipe Framework Architecture](#7-mediapipe-framework-architecture)
8. [MediaPipe Tasks API](#8-mediapipe-tasks-api)
   - [8.5 Audio Processing with MediaPipe Tasks](#85-audio-processing-with-mediapipe-tasks)
9. [MediaPipe Model Maker: Custom Task Training](#9-mediapipe-model-maker-custom-task-training)
10. [Camera2 → MediaPipe Zero-Copy Pipeline](#10-camera2--mediapipe-zero-copy-pipeline)
11. [ARCore + MediaPipe Composition](#11-arcore--mediapipe-composition)
12. [Performance Profiling and Tuning](#12-performance-profiling-and-tuning)
   - [12.7 Benchmarking with benchmark_model](#127-benchmarking-with-benchmark_model)
13. [Model Security: Encryption, Signing, and Integrity Protection](#13-model-security-encryption-signing-and-integrity-protection)
14. [Privacy by Design: Data Protection for On-Device ML](#14-privacy-by-design-data-protection-for-on-device-ml)
15. [On-Device Training and Federated Learning](#15-on-device-training-and-federated-learning)
16. [Gemini Nano: On-Device Generative AI](#16-gemini-nano-on-device-generative-ai)
    - [16.1 AICore System Architecture](#161-aicore-system-architecture)
    - [16.2 ML Kit GenAI API](#162-ml-kit-genai-api)
    - [16.3 Streaming Inference and Token Budget](#163-streaming-inference-and-token-budget)
    - [16.4 Hardware Requirements and Memory Footprint](#164-hardware-requirements-and-memory-footprint)
    - [16.5 Capability Limits and Task Fit](#165-capability-limits-and-task-fit)
    - [16.6 Gemini Nano vs LiteRT-LM + Open Weights](#166-gemini-nano-vs-litert-lm--open-weights)
17. [Roadmap](#17-roadmap)
   - [17.10 GGUF and llama.cpp on Android](#1710-gguf-and-llamacpp-on-android)
18. [Integrations](#18-integrations)


---

## 1. Why ML Inference Belongs in a Graphics Stack Book

A natural question arises when encountering LiteRT and MediaPipe in a book whose previous chapters have traced the path from the Linux DRM subsystem through Mesa, SurfaceFlinger, and Vulkan: why does on-device machine learning inference belong here at all? The answer is that it does not live *beside* the Android graphics stack — it lives *inside* it, sharing the same GPU command queues, the same `AHardwareBuffer` memory objects, the same DMA-BUF backing, and the same EGL context infrastructure described in Ch85 and Ch86.

When an Android application runs pose-landmark detection at 30 frames per second on a phone camera stream, the path is approximately as follows. The Camera HAL produces frames as Gralloc-backed `AHardwareBuffer` objects (Ch85). Those buffers are imported into an OpenGL ES 3.1 compute pipeline — exactly the same command-submission path that GLES rendering uses. The inference result may then flow, still as a GPU-resident tensor, into SurfaceFlinger for overlay composition (Ch85) or into an ARCore pass for world-anchored landmark projection (Ch87). No CPU copy need occur from the moment the camera image leaves the sensor ISP until the landmark overlay is composited to the display framebuffer.

This architecture, pioneered in LiteRT's GPU delegate and systematised by the MediaPipe framework, makes on-device ML inference a *first-class user of GPU resources* on Android. Understanding the buffer lifecycle, delegate dispatch, and synchronisation primitives is therefore essential knowledge for any engineer working at the intersection of Android graphics and ML — and increasingly, that intersection is most of the stack.

A secondary motivation is the rise of LiteRT-LM (announced May 2026), which brings multi-turn large-language-model inference to the same GPU compute pipeline, targeting 52 tokens/second on Android GPU and 56 tokens/second on Apple Metal. [Source: Google Developers Blog, LiteRT-LM](https://developers.googleblog.com/blazing-fast-on-device-genai-with-litert-lm/) On-device LLM inference is not a different technology layer — it is the GPU compute pipeline operating on larger models.

---

## 2. LiteRT Architecture

**LiteRT** is the name Google adopted in 2024 for the inference runtime previously known as TensorFlow Lite (TFLite). The rebrand reflects a strategic broadening: LiteRT is positioned as a universal on-device inference runtime independent of the TensorFlow training ecosystem, competing directly with ONNX Runtime Mobile and CoreML. [Source: ai.google.dev/edge/litert](https://ai.google.dev/edge/litert)

### 2.1 The `.tflite` FlatBuffer Model Format

LiteRT models are stored as [FlatBuffers](https://flatbuffers.dev/), a schema-based binary serialisation format with zero-copy read access. The schema lives in the LiteRT repository at `tflite/schema/schema.fbs`. [Source: LiteRT schema](https://github.com/google-ai-edge/LiteRT/blob/main/tflite/schema/schema.fbs)

A `.tflite` file contains:
- One or more **subgraphs** (usually one for single-entry-point models; multiple for multi-signature models). Each subgraph has an ordered list of **operators** (encoded as operator codes referencing a built-in or custom op), a list of **tensors** (shape, element type, quantisation parameters, name, and an index into the buffer table), and lists of input and output tensor indices.
- A **buffer table**: an array of byte arrays holding constant data. Weight tensors point into this table; activation tensors point to a null entry (allocated at runtime).
- **Metadata**: key-value pairs that attach schema-versioned blobs — used by the `SignatureDef` mechanism and by model-card metadata.

The FlatBuffer layout enables the interpreter to memory-map the file and read tensor weights with zero copy — important on memory-constrained devices where a full deserialization would double peak RSS.

### 2.2 Built-in Op Coverage and Custom Ops

LiteRT ships with a `BuiltinOpResolver` that covers the standard TensorFlow Lite operator set: convolution variants (`CONV_2D`, `DEPTHWISE_CONV_2D`, `TRANSPOSE_CONV`), pooling, activations, element-wise arithmetic, reshape, concatenation, attention-related ops (`BATCH_MATMUL`, `SPLIT`), and quantisation/dequantisation ops. The resolver can be extended with custom ops:

```cpp
tflite::MutableOpResolver resolver;
resolver.AddBuiltin(tflite::BuiltinOperator_CONV_2D,
                    tflite::ops::builtin::Register_CONV_2D());
// Register a custom C++ kernel
resolver.AddCustom("MyCustomLayer", RegisterMyCustomLayer());
```

Custom ops that the GPU delegate does not know about automatically fall back to the CPU partition. If a critical custom op must run on GPU, it can be implemented as a `DelegateKernelInterface` registered with the delegate at creation time — a more advanced pattern used by MediaPipe's `TfLiteInferenceCalculator` for pre/post-processing fusion.

### 2.3 The Interpreter API

The C++ Interpreter API follows a construction-then-invoke pattern:

```cpp
#include "tflite/interpreter_builder.h"
#include "tflite/kernels/register.h"
#include "tflite/model_builder.h"

// Load model (memory-maps the file, no copy)
auto model = tflite::FlatBufferModel::BuildFromFile("model.tflite");

// Create the op resolver (built-in ops) and build an Interpreter
tflite::ops::builtin::BuiltinOpResolver resolver;
std::unique_ptr<tflite::Interpreter> interpreter;
tflite::InterpreterBuilder(*model, resolver)(&interpreter);

// Allocate activation tensors (malloc/mmap based on planner)
interpreter->AllocateTensors();

// Access the input tensor and populate it
float* input = interpreter->typed_input_tensor<float>(0);
// ... populate input buffer, e.g. from a camera frame ...

// Run inference
interpreter->Invoke();

// Read the output tensor
float* output = interpreter->typed_output_tensor<float>(0);
```
[Source: LiteRT C++ inference guide](https://ai.google.dev/edge/litert/inference)

The `TfLiteTensor` struct, accessible via `interpreter->input_tensor(i)` and `interpreter->output_tensor(i)`, exposes the layout:

```c
typedef struct TfLiteTensor {
    TfLiteType type;           // kTfLiteFloat32, kTfLiteInt8, etc.
    TfLiteIntArray* dims;      // shape: dims->data[0..rank-1]
    TfLiteQuantizationParams params; // zero_point, scale for INT8
    void* data;                // raw pointer to tensor memory
    size_t bytes;              // byte size of the buffer
    TfLiteAllocationType allocation_type; // kTfLiteArenaRw, kTfLiteMmapRo, etc.
    // ... additional fields omitted for brevity
} TfLiteTensor;
```

After `AllocateTensors()`, weight tensors point into the memory-mapped model file (`kTfLiteMmapRo`) while activation tensors point into a contiguous arena (`kTfLiteArenaRw`). When a GPU or NNAPI delegate takes ownership of a subgraph, its tensors may switch to a delegate-managed allocation type.

### 2.4 Python API

The Python runtime uses the `tflite-runtime` package (a thin wrapper around the C++ library, separate from the full `tensorflow` package):

```python
import numpy as np
from tflite_runtime.interpreter import Interpreter

interpreter = Interpreter(model_path="model.tflite")
interpreter.allocate_tensors()

input_details = interpreter.get_input_details()
output_details = interpreter.get_output_details()

# Populate input
interpreter.set_tensor(input_details[0]['index'],
                       np.expand_dims(image_array, axis=0))
interpreter.invoke()

output = interpreter.get_tensor(output_details[0]['index'])
```
[Source: LiteRT Python quickstart](https://ai.google.dev/edge/litert/microcontrollers/python)

The Python API is primarily used for desktop development, rapid prototyping, and the benchmark/evaluation pipeline before deploying models to Android NDK.

---

## 3. Delegate Architecture: Routing Inference to Hardware

The delegate system is LiteRT's mechanism for offloading operator subgraphs to hardware accelerators. After `InterpreterBuilder` constructs the interpreter with a CPU kernel map, delegates inspect the graph, claim the subset of operators they can accelerate, and replace those nodes with an opaque "delegate node" whose `Invoke()` method dispatches to hardware. The remaining operators run on CPU. This partitioning is called **delegation**.

### 3.0 Graph Partitioning

When `interpreter->ModifyGraphWithDelegate(delegate)` is called, LiteRT's graph partitioner runs a DFS over the operator DAG. For each operator, it queries `TfLiteDelegate::flags` and calls the delegate's `IsNodeSupportedFn` callback to determine which nodes the delegate can handle. Contiguous runs of supported nodes form **delegate partitions** — opaque super-nodes in the graph. The remaining unsupported nodes stay on CPU. `max_delegated_partitions` bounds the number of partitions the delegate may claim; setting it to 1 forces all accelerated ops into a single contiguous subgraph, which is often preferable because partitioned graphs incur CPU↔GPU synchronisation overhead at each partition boundary.

After partitioning, each delegate partition registers a `TfLiteNode` whose `invoke` callback calls into the delegate's `DelegateKernelInterface::Invoke()`. From the scheduler's perspective, a delegate partition is just another op with GPU memory inputs and outputs.

### 3.1 NNAPI: Historical Context (API 27–34)

> **Status: deprecated in Android 15 (API 35).** NNAPI remains functional on Android 8.1–14 devices (the majority of the installed base in 2026), but no new features are planned. New code targeting API 35+ should use vendor NPU delegates (§3.3) or the GPU delegate. [Source: NNAPI migration guide](https://developer.android.com/ndk/guides/neuralnetworks/migration-guide)

**What NNAPI was:** A universal portability abstraction introduced in Android 8.1 (API 27) that let apps target any Android NPU/DSP/GPU via a single C API (`NeuralNetworks.h`) without knowing the vendor hardware. `libneuralnetworks.so` forwarded inference requests over Binder IPC to the `neuralnetworks_service` system daemon, which loaded the OEM's HIDL/AIDL HAL implementation (targeting Hexagon DSP, APU, or GPU) in the vendor partition.

**HAL version milestones:**

| Version | Android | API | Key additions |
|---|---|---|---|
| v1.0 | 8.1 Oreo | 27 | 29 ops (Conv2D, FC, MaxPool, ReLU, Softmax…); FP32 only |
| v1.1 | 9 Pie | 28 | 8 more ops; FP16 relaxed compute option |
| v1.2 | 10 Q | 29 | 60+ ops; device enumeration; AHardwareBuffer memory domains; sync-fence execution; burst execution |
| v1.3 | 11 R | 30 | Signed INT8; memory descriptor API; compilation priority |
| AIDL v1–v2 | 12–13 S–T | 31–33 | HAL migrated from HIDL to AIDL; NNAPI Shim for APEX delivery |
| Deprecated | 15 V | 35 | Marked `@Deprecated`; no new features |

**The LiteRT NNAPI delegate** (`tflite/delegates/nnapi/nnapi_delegate.h`) wraps this API transparently. It partitions the TFLite graph via `ModifyGraphWithDelegate()`, translates supported ops to `ANeuralNetworksModel` operands, compiles via `ANeuralNetworksCompilation_finish()`, and submits execution via `ANeuralNetworksExecution_compute()` (synchronous) or `startComputeWithDependencies()` (sync-fence, for camera pipelines). Compilation caching (`setCaching()`) reduces cold-start from 1–5 s to ~30 ms.

```cpp
#include "tflite/delegates/nnapi/nnapi_delegate.h"

StatefulNnApiDelegate::Options options;
options.execution_preference = StatefulNnApiDelegate::Options::kSustainedSpeed;
options.allow_fp16 = true;
options.max_number_delegated_partitions = 3;
options.cache_dir   = "/data/data/com.example.app/nnapi_cache";
options.model_token = "mymodel_v1";
// options.accelerator_name = "qti-dsp";  // target a specific device (API 29+)

auto delegate = std::make_unique<StatefulNnApiDelegate>(options);
interpreter->ModifyGraphWithDelegate(delegate.get());
// Ops unsupported by hardware fall back to CPU automatically.
```
[Source: LiteRT NNAPI delegate](https://ai.google.dev/edge/litert/android/delegates/nnapi)

**Why it was deprecated:** (1) HAL version fragmentation — many shipping devices were stuck on v1.0/v1.1, preventing apps from relying on v1.2 features. (2) Vendor quality inconsistency — numerical results diverged from the reference CPU implementation beyond tolerable bounds on several HAL implementations. (3) Binder IPC overhead — ~0.5–1 ms round-trip per inference call. (4) New NPU architectures (Pixel Tensor G4, Snapdragon X Elite, Dimensity 9400 APU) expose hardware features that don't map to the NNAPI op vocabulary. The replacement (§3.3) drops the universal HAL in favour of in-process vendor delegate `.so` files.

### 3.2 GPU Delegate

The GPU delegate runs inference as **OpenGL ES 3.1 compute shaders** (the default on most Android devices) or **Vulkan compute pipelines** (when `TFLITE_GPU_EXPERIMENTAL_FLAGS_ENABLE_VULKAN_INFERENCE` is set in `experimental_flags`). The delegate translates the LiteRT operator graph into a series of GLSL compute shaders (or SPIR-V compute pipelines), compiling each op into one or more dispatch calls. [Source: LiteRT GPU delegate](https://ai.google.dev/edge/litert/performance/gpu)

The options struct, defined in `tflite/delegates/gpu/delegate_options.h`, controls behaviour:

```cpp
#include "tflite/delegates/gpu/delegate.h"

TfLiteGpuDelegateOptionsV2 options = TfLiteGpuDelegateOptionsV2Default();

// For sustained inference (live camera stream) — maximise throughput:
options.inference_preference =
    TFLITE_GPU_INFERENCE_PREFERENCE_SUSTAINED_SPEED;

// Priority ordering: latency first, then precision, then memory:
options.inference_priority1 = TFLITE_GPU_INFERENCE_PRIORITY_MIN_LATENCY;
options.inference_priority2 = TFLITE_GPU_INFERENCE_PRIORITY_MAX_PRECISION;
options.inference_priority3 = TFLITE_GPU_INFERENCE_PRIORITY_MIN_MEMORY_USAGE;

// Serialisation: cache compiled shaders to avoid recompilation on restart:
options.serialization_dir = "/data/local/tmp/litert_gpu_cache";
options.model_token = "pose_landmark_lite_v1";

TfLiteDelegate* gpu_delegate = TfLiteGpuDelegateV2Create(&options);
interpreter->ModifyGraphWithDelegate(gpu_delegate);
```
[Source: TfLiteGpuDelegateOptionsV2 header](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/lite/delegates/gpu/delegate_options.h)

Key fields in `TfLiteGpuDelegateOptionsV2`:

| Field | Type | Purpose |
|---|---|---|
| `inference_preference` | `int32_t` | `FAST_SINGLE_ANSWER` (0) or `SUSTAINED_SPEED` (1) or `BALANCED` (2) |
| `inference_priority1/2/3` | `int32_t` | Ordered priorities: AUTO, MAX_PRECISION, MIN_LATENCY, MIN_MEMORY_USAGE |
| `experimental_flags` | `int64_t` | Bitmask for Vulkan backend, etc. |
| `max_delegated_partitions` | `int32_t` | Limit graph partitioning (default 1) |
| `serialization_dir` | `const char*` | Path for compiled kernel cache |
| `model_token` | `const char*` | Unique identifier for cache namespace |

**Thread-safety note:** When calling `Interpreter::ModifyGraphWithDelegate()` or `Interpreter::Invoke()`, the calling thread must have an EGLContext active. If one does not exist, the delegate creates one internally, but then all subsequent `Invoke()` calls must originate from the same thread. This is a common integration pitfall in Android JNI code. [Source: LiteRT GPU C/C++ guide](https://ai.google.dev/edge/litert/android/gpu_native)

Shader serialisation (`serialization_dir` + `model_token`) persists compiled GLSL/SPIR-V kernels to disk. On subsequent runs, the delegate loads the binary directly, reducing first-inference latency from ~500 ms to ~10 ms on a mid-range Adreno GPU.

### 3.3 What Replaced NNAPI: Vendor Delegates

NNAPI's core promise was a **universal portability abstraction**: write your model once, target any Android NPU/DSP/GPU via a single API, without knowing the vendor hardware. Its deprecation explicitly abandons that abstraction in favour of a **direct-dispatch model** where chip vendors ship LiteRT delegate libraries as in-process `.so` files, and apps load them at runtime. Portability becomes the app's concern — the hardware vendor no longer participates in a common HAL.

#### The Architectural Shift

```
NNAPI model (deprecated):
App ──[NeuralNetworks.h]──► libneuralnetworks.so
                                    │ Binder IPC
                            neuralnetworks_service
                                    │ HIDL/AIDL HAL
                            vendor HAL .so (vendor partition)
                                    │
                            Hexagon DSP / NPU hardware

Vendor delegate model (replacement):
App ──[LiteRT Interpreter]──► StatefulNnApiDelegate  [gone]
                           ──► TfLiteGpuDelegateV2   [GLES compute]
                           ──► QnnTFLiteDelegate      [in-process .so]
                                    │  direct IPC / kernel driver ioctl
                            Hexagon HTP / NPU hardware
```

The key differences:
- **No daemon**: vendor delegates run in the app process; there is no cross-process Binder IPC for each inference call
- **No HAL version matrix**: the delegate `.so` ships with the model/app; the hardware is accessed via direct kernel driver ioctls (e.g., `/dev/fastrpc-cdsp` for Hexagon, or `/dev/dsp` on MediaTek)
- **No universal fallback**: if a vendor delegate is not present on the device, the app must fall back to the GPU delegate or CPU manually
- **Hardware-specific features**: vendor delegates can expose op extensions (e.g., Qualcomm QNN's `QNN_OP_MULTI_HEAD_ATTENTION`) that have no NNAPI equivalent

#### 3.3.1 Qualcomm AI Engine Direct (QNN) Delegate

The Qualcomm Neural Network (QNN) delegate is the most widely deployed vendor delegate as of 2026, targeting Snapdragon SoCs (8 Gen 1–4, X Elite). It routes inference to the **Hexagon Tensor Processor (HTP)** — the dedicated NPU that replaced the older Hexagon DSP vector extensions. [Source: Google Developers Blog, Qualcomm NPU LiteRT](https://developers.googleblog.com/unlocking-peak-performance-on-qualcomm-npu-with-litert/)

**Setup via Maven:**

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("com.google.ai.edge.litert:litert-gpu:1.0.1")
    // QNN delegate — pulls libQnnTFLiteDelegate.so from Maven Central:
    implementation("com.qualcomm.qnn:qnn-litert-delegate:2.26.0")
}
```

**C++ usage:**

```cpp
#include "QnnTFLiteDelegate.h"

TfLiteQnnDelegateOptions qnn_options = TfLiteQnnDelegateOptionsDefault();

// Target backend: HTP (Hexagon Tensor Processor) for NPU acceleration.
// Other backends: DSP (older Hexagon vector), GPU, CPU (reference).
qnn_options.backend_type = kHtpBackend;

// HTP performance mode — maps to Hexagon clock domain selection:
// kHtpDefault, kHtpSustainedHighPerformance, kHtpBurst, kHtpHighPerformance,
// kHtpPowerSaver, kHtpLowPower, kHtpLowBalanced
qnn_options.htp_options.performance_mode = kHtpHighPerformance;

// Weight sharing: cache the compiled QNN graph in /data/local/tmp
// to avoid recompilation on subsequent cold starts (~3-8 s first run).
qnn_options.htp_options.use_conv_hmx = 1;     // use HMX (matrix multiply units)
qnn_options.htp_options.use_fold_relu = 1;    // fuse ReLU into preceding op

qnn_options.cache_dir = "/data/data/com.example.app/qnn_cache";
qnn_options.model_prefix = "mobilenet_v3_large";

TfLiteDelegate* qnn_delegate = TfLiteQnnDelegateCreate(&qnn_options);
interpreter->ModifyGraphWithDelegate(qnn_delegate);
```
[Source: Qualcomm QNN LiteRT delegate docs](https://docs.qualcomm.com/bundle/publicresource/topics/80-63442-50/litert-delegate.html)

**QNN vs NNAPI on Snapdragon:**

| Metric | NNAPI (Hexagon DSP, API 29–34) | QNN Delegate (HTP, API 29+) |
|---|---|---|
| Per-inference IPC overhead | ~0.5–1 ms (Binder round-trip) | ~0.05 ms (in-process ioctl) |
| First-run compilation | 2–8 s (no cache) | 3–10 s (no cache) |
| Cached startup | ~30 ms | ~15 ms |
| MobileNetV3 INT8 latency (8 Gen 3) | ~1.8 ms | ~0.9 ms |
| Supported op count | NNAPI v1.3 op set (~150 ops) | QNN op set (~300+ ops incl. attention) |
| Requires device pairing | No (works on any Snapdragon ≥ 8.1) | No (`.so` ships with app) |

**Model quantisation for HTP**: The HTP operates natively in INT8 and INT16. For best performance, quantise the model to INT8 with per-channel scales using `ai_edge_quantizer` or `tf.lite.TFLiteConverter` with `QUANTIZE` optimisation. FP32 models run on the HTP via software quantisation, but at 3–5× the latency of a properly quantised model.

#### 3.3.2 Google Tensor NPU Delegate (Pixel)

Pixel 6–9 devices include the **Google Tensor** SoC with a dedicated **TPU (Tensor Processing Unit)** designed at Google specifically for on-device ML. The Tensor NPU delegate targets this hardware via Google's internal `libedgetpu_ml_platform.so`, delivered as part of the Pixel system image.

Unlike QNN, the Tensor NPU delegate is **not available on Maven Central** and cannot be bundled with third-party apps — it is only accessible through the **AICore system service** (`android.ai.app.aicore`) and (for some use cases) via the **ML Kit GenAI API** and the **Tensor ML Platform SDK** (restricted access):

```kotlin
// For Pixel 6+ via AICore — model must be pre-loaded by Google:
val genAiOptions = GenerativeModel.Options.builder()
    .setModel("gemini-nano")
    .build()
val genAiModel = GenerativeModel(genAiOptions)   // routes through Tensor NPU via AICore
```

For general LiteRT models on Pixel devices, use the **GPU delegate** (which also benefits from Pixel's Arm Mali/Immortalis GPU) or the **NNAPI qti-hta / nnapi-reference path** on Pixel 6 (which includes a Tensor G1 with NNAPI HAL v1.3 support). The Tensor G4 on Pixel 9 no longer exposes an NNAPI HAL surface — it is accessed exclusively via AICore.

[Source: Android ML Platform](https://developer.android.com/ml/platform)

#### 3.3.3 MediaTek APU Delegate (NeuroPilot)

MediaTek Dimensity SoCs (9000, 9200, 9300, 9400) include an **AI Processing Unit (APU)** running NeuroPilot firmware. The LiteRT delegate ships via the **MediaTek NeuroPilot SDK**:

```kotlin
// build.gradle.kts — MediaTek SDK (not on Maven Central; integrate via AAR)
// Download from developer.mediatek.com/NeuroPilot
```

```cpp
#include "neuron_delegate.h"

NeuronDelegateOptions neuron_opts = NeuronDelegateOptions{
    .execution_preference = kNeuronPreferSustainedSpeed,
    .allow_fp16 = true,
    .use_ahwb = true,   // import AHardwareBuffer tensors directly (API 29+)
    .cache_dir = "/data/data/com.example.app/neuron_cache",
    .model_token = "mymodel_v1",
};
TfLiteDelegate* neuron_delegate = NeuronDelegate(neuron_opts);
interpreter->ModifyGraphWithDelegate(neuron_delegate);
```

The NeuroPilot APU supports ~200 ops in the TFLite op set and adds INT4 weight quantisation (Dimensity 9400+). Like QNN, it caches compiled APU binaries to disk to avoid recompilation.

#### 3.3.4 Runtime Discovery and Graceful Fallback

The core migration challenge is that vendor delegates are device-specific: a QNN delegate binary is meaningless on a MediaTek device, and vice versa. Apps must implement a **priority chain**:

```cpp
// Preferred: query for Qualcomm HTP availability
bool qnn_available = false;
{
    TfLiteQnnDelegateOptions opts = TfLiteQnnDelegateOptionsDefault();
    opts.backend_type = kHtpBackend;
    TfLiteDelegate* d = TfLiteQnnDelegateCreate(&opts);
    if (d) {
        TfLiteStatus s = interpreter->ModifyGraphWithDelegate(d);
        qnn_available = (s == kTfLiteOk);
        if (!qnn_available) TfLiteQnnDelegateDelete(d);
    }
}

if (!qnn_available) {
    // Try GPU delegate (works on any OpenGL ES 3.1 / Vulkan 1.1 device):
    TfLiteGpuDelegateOptionsV2 gpu_opts = TfLiteGpuDelegateOptionsV2Default();
    gpu_opts.inference_preference = TFLITE_GPU_INFERENCE_PREFERENCE_SUSTAINED_SPEED;
    gpu_opts.serialization_dir = cache_dir.c_str();
    gpu_opts.model_token = "mymodel_gpu_v1";
    TfLiteDelegate* gpu_delegate = TfLiteGpuDelegateV2Create(&gpu_opts);
    if (interpreter->ModifyGraphWithDelegate(gpu_delegate) != kTfLiteOk) {
        // Ultimate fallback: CPU (XNNPACK) — always works
        // XNNPack delegate is added automatically by InterpreterBuilder
        // when no other delegate succeeds.
    }
}
```

Google's **LiteRT Acceleration Service** (part of Google Play Services, formerly `AccelerationService`) automates this chain: it benchmarks candidate delegates on-device on first run and caches the best choice, transparently handling the fallback. [Source: LiteRT Acceleration Service](https://ai.google.dev/edge/litert/android/play_services)

```kotlin
// Kotlin — using LiteRT Acceleration Service via Play Services:
val accelerationConfig = CustomValidationConfig.builder()
    .setAccelerationConfig(
        CpuAccelerationConfig.builder().setNumThreads(4).build()
    )
    .build()

AccelerationService.create(context).validatedAccelerationConfigFor(
    model_bytes,
    accelerationConfig
).addOnSuccessListener { config ->
    // config contains the best validated delegate for this device
    val interpreter = Interpreter(model_bytes, config.interpreterOptions)
}
```

#### 3.3.5 Vendor Delegate Op Coverage Comparison

| Op Category | CPU (XNNPack) | GPU (GLES) | QNN HTP | Neuron APU | Tensor NPU (AICore) |
|---|---|---|---|---|---|
| Conv2D / DepthwiseConv2D | ✅ | ✅ | ✅ HMX | ✅ | ✅ |
| FullyConnected (FP32) | ✅ | ✅ | ✅ | ✅ | ✅ |
| FullyConnected (INT4 weights) | ✅ | ❌ | ✅ | ✅ (D9400) | ✅ |
| LSTM / GRU | ✅ | ✅ | ✅ | ✅ | ✅ |
| Multi-Head Attention (fused) | ❌ | ❌ | ✅ QNN extension | ❌ | ✅ |
| Custom ops | ✅ C++ kernel | ❌ | ❌ (CPU fallback) | ❌ | ❌ |
| Dynamic shapes | ✅ | Partial | Partial | Partial | ❌ |
| FP16 tensors | ✅ (NEON) | ✅ | ✅ | ✅ | ✅ |
| INT8 asymmetric | ✅ | ✅ | ✅ | ✅ | ✅ |
| INT4 weight-only quant | ✅ (LiteRT-LM) | ❌ | ✅ (QNN 2.22+) | ✅ (D9400) | ✅ |

### 3.4 Edge TPU Delegate

Coral hardware (USB, M.2, and PCIe Edge TPU accelerators) uses a separate delegate:

```cpp
#include "edgetpu.h"

TfLiteDelegate* edgetpu_delegate = edgetpu_create_delegate(
    EDGETPU_APEX_USB, /*device=*/nullptr, /*options=*/nullptr, 0);
interpreter->ModifyGraphWithDelegate(edgetpu_delegate);
```

Edge TPU models must be compiled with the [Edge TPU compiler](https://coral.ai/docs/edgetpu/compiler/) ahead of time, which quantises ops to INT8 and maps them to the TPU's SIMD arrays. Ops that the compiler cannot map remain on CPU.

### 3.5 XNNPack: CPU Inference with SIMD and Thread Parallelism

XNNPack is the default CPU inference backend in LiteRT and deserves more than a footnote. Unlike the vendor delegates in §3.3, XNNPack is not a "delegate" in the traditional sense — it does not claim a subgraph via `ModifyGraphWithDelegate()` and replace nodes with an opaque partition. Instead, it is integrated directly into the `BuiltinOpResolver` as the preferred implementation of each supported operator. When `InterpreterBuilder` constructs an interpreter from a `BuiltinOpResolver`, XNNPack-optimised kernels are automatically registered for all ops it covers. No explicit delegate creation is required for the default path; the fallback to the original reference CPU kernels only occurs for ops XNNPack does not yet support. [Source: XNNPack GitHub](https://github.com/google/XNNPACK) [Source: LiteRT best practices](https://ai.google.dev/edge/litert/performance/best-practices)

That said, XNNPack *can* also be invoked explicitly as a delegate — this is useful when you need to control thread count or enable experimental flags while still relying on the CPU path:

```cpp
#include "tflite/delegates/xnnpack/xnnpack_delegate.h"

TfLiteXNNPackDelegateOptions xnnpack_opts =
    TfLiteXNNPackDelegateOptionsDefault();
xnnpack_opts.num_threads = 4;  // use 4 threads for compute
TfLiteDelegate* xnnpack_delegate = TfLiteXNNPackDelegateCreate(&xnnpack_opts);
interpreter->ModifyGraphWithDelegate(xnnpack_delegate);
// Release when done: TfLiteXNNPackDelegateDelete(xnnpack_delegate);
```

#### Microkernel Architecture

XNNPack's design philosophy is to compile per-operator microkernels for every ISA it encounters, selecting at runtime the highest-performance path available on the current CPU. The ISA targets as of 2026 are:

- **NEON** (Armv7, Armv8): baseline for all Android ARM devices
- **NEON+FP16** (Armv8.2+): FP16 arithmetic for the compute path, reducing bandwidth on mid-to-high-end chips (Cortex-A55+, Cortex-X1+)
- **SVE / SVE2** (Armv8.2+ with SVE extensions, Cortex-X1+, Cortex-X4): scalable vector extensions; XNNPack SVE kernels adapt to the CPU's vector length at runtime, covering 128-bit through 512-bit SIMD widths
- **SSE4.1 / AVX / AVX-512** (x86): covers x86-64 Linux and Chrome OS ML inference
- **WebAssembly SIMD**: the WASM target uses the 128-bit `v128` type, enabling browser-side LiteRT to benefit from the same microkernel investment

CPU feature detection at startup determines which microkernel set is loaded. The selection is done once at `InterpreterBuilder` time via CPUID (x86) or `getauxval(AT_HWCAP)` (ARM), storing the result in a global capability bitmask. Each XNNPack operator then dispatches through a function pointer set to the best available implementation.

#### Thread Pool and Parallelism

`XNNPackDelegate` creates a `pthreadpool` thread pool with `num_threads` workers. When `num_threads` is left at its default of 0, XNNPack queries the number of physical (not hyperthreaded) CPU cores and creates that many workers. On a Snapdragon 8 Gen 3 with 1 prime + 3 performance + 4 efficiency cores, this yields 8 workers; XNNPack's tiling strategy partitions the output tensor into horizontal strips, assigning one strip per thread. The `pthreadpool` library uses futex-based barriers — threads block in the kernel between operator invocations rather than spinning, which is critical for power efficiency on battery-constrained devices.

Setting `num_threads` above the physical core count does not help and typically hurts: hyper-thread siblings share L1 cache bandwidth with the primary sibling, and the thread synchronisation overhead at strip boundaries adds latency without throughput benefit. For power-sensitive workloads, setting `num_threads = 2` or `num_threads = 4` and pinning to the performance core cluster (via `pthread_setaffinity_np`) is a common tuning choice.

#### Weight Caching and Subgraph Reshaping

For quantised INT8 models, XNNPack pre-packs weight tensors into an optimised memory layout on first use — for example, NEON matrix-multiply kernels prefer a packed, block-transposed column-major layout rather than the row-major weight storage in the `.tflite` FlatBuffer. This pre-packing occurs during `AllocateTensors()` and is cached in heap memory for the lifetime of the interpreter.

When the input tensor shape changes at runtime (e.g., the application switches from 640×480 to 1280×720 camera resolution), the default behaviour is to re-run graph partitioning and re-allocate activation tensors, which can discard the pre-packed weights. Enabling subgraph reshaping preserves the weight cache across `ResizeInputTensor()` calls:

```cpp
xnnpack_opts.flags |= TFLITE_XNNPACK_DELEGATE_FLAG_ENABLE_SUBGRAPH_RESHAPING;
```

This flag is important for applications that dynamically adjust resolution based on thermal state or frame rate targets — without it, each resize triggers a costly re-pack of all weight tensors.

#### When XNNPack CPU Beats the GPU Delegate

The received wisdom is that GPU is always faster than CPU for neural network inference, but this is not universally true on Android. The GPU delegate's pipeline involves shader compilation (amortised with caching), GPU command submission overhead, and CPU↔GPU synchronisation at the delegate boundary. For models with fewer than approximately 1 million parameters, these fixed overheads can dominate the actual compute time.

Concrete guidance for typical models (benchmarked on Cortex-A78, 4 threads):

| Precision | Latency (4 threads) | Latency (1 thread) |
|---|---|---|
| FP32 | ~18 ms | ~65 ms |
| FP16 (Armv8.2+) | ~11 ms | ~40 ms |
| INT8 (quantised) | ~6 ms | ~21 ms |

For comparison, the GPU delegate on a mid-range Adreno 640 achieves ~5 ms for the same MobileNetV3-Small model — comparable to XNNPack INT8 at 4 threads. For MobileNetV2-Small and all MediaPipe "lite" model variants (pose landmark lite, hand landmark lite — all under 5 MB), benchmarking frequently shows XNNPack INT8 matching or beating the GPU delegate, particularly when shader serialisation is not configured (cold-start GPU path adds 200–500 ms on first run). Always benchmark both paths on your target device before assuming GPU is the right choice.

#### Operator Coverage

XNNPack covers the majority of vision-model operators: `CONV_2D`, `DEPTHWISE_CONV_2D`, `TRANSPOSE_CONV`, `FULLY_CONNECTED`, all pooling variants, element-wise arithmetic (`ADD`, `MUL`, `SUB`), activations (`RELU`, `RELU6`, `SIGMOID`, `HARD_SWISH`), `CONCATENATION`, `RESHAPE`, `PAD`, `MEAN`, and `SOFTMAX`. It does not accelerate `LSTM` or `UNIDIRECTIONAL_SEQUENCE_LSTM` — for recurrent models, use the GPU delegate's `LSTM` path or the platform's `BATCH_MATMUL`-based reformulation. The full supported op set can be queried at runtime via `TfLiteXNNPackDelegateGetFlags()`, which returns a bitmask of enabled capabilities.

---

## 4. AHardwareBuffer Tensor Interop

This section covers the primary integration point with Ch85 (SurfaceFlinger and `AHardwareBuffer`) and Ch86 (Vulkan on Android). The GPU delegate can bind `AHardwareBuffer`-backed memory directly as tensor storage, enabling a zero-copy path from the Android Camera HAL to the inference engine.

### 4.1 Memory Model

An `AHardwareBuffer` wraps a Gralloc-allocated buffer — a DMA-BUF file descriptor managed by the kernel's GEM/DMA heap subsystem. On import into OpenGL ES, the buffer becomes an **SSBO** (shader storage buffer object) or an **EGLImage** depending on its format. The GPU delegate imports it as an OpenGL ES SSBO when the format is `AHARDWAREBUFFER_FORMAT_BLOB` (a raw byte array suitable for tensor data), or as an `EGLImage` (via `eglCreateImageKHR` with `EGL_ANDROID_image_native_buffer`) when the format is a pixel format.

### 4.2 Usage Flags

`AHardwareBuffer_Desc.usage` encodes the access pattern. For tensor data:

```c
AHARDWAREBUFFER_USAGE_GPU_DATA_BUFFER   // Generic GPU-readable/writable blob
AHARDWAREBUFFER_USAGE_CPU_WRITE_OFTEN   // CPU fills the buffer from camera/sensor
AHARDWAREBUFFER_USAGE_GPU_SAMPLED_IMAGE // GPU reads it as a texture (pixel inputs)
```

[Source: AHardwareBuffer NDK docs](https://developer.android.com/ndk/reference/group/a-hardware-buffer)

### 4.3 Allocating and Binding a Tensor Buffer

The following pattern allocates an `AHardwareBuffer` for use as a 1-D float tensor of `N` elements, then binds it to the GPU delegate:

```c
#include <android/hardware_buffer.h>
#include "tflite/delegates/gpu/delegate.h"

// Step 1: Describe the buffer
AHardwareBuffer_Desc desc = {
    .width  = N,           // number of floats (blob: width = total bytes / 4)
    .height = 1,
    .layers = 1,
    .format = AHARDWAREBUFFER_FORMAT_BLOB,
    .usage  = AHARDWAREBUFFER_USAGE_GPU_DATA_BUFFER |
              AHARDWAREBUFFER_USAGE_CPU_WRITE_OFTEN,
};

// Step 2: Allocate the buffer (requires API level 26 / Android 8.0+)
AHardwareBuffer* ahwb = nullptr;
AHardwareBuffer_allocate(&desc, &ahwb);

// Step 3: Import the AHardwareBuffer into an OpenGL ES buffer object.
// Use EGL_ANDROID_image_native_buffer + GL_OES_EGL_image to get a GL
// buffer handle, then bind it to the LiteRT GPU delegate.
// TfLiteGpuDelegateAssignGLBufferToTensor() binds an existing GL buffer
// object (obtained via eglCreateImageKHR + glEGLImageTargetTexture2DOES
// or via GL_EXT_memory_object_fd for DMA-BUF import) to a tensor index.
//
// Note: The exact binding function name varies by LiteRT version. As of
// LiteRT 1.x, the API is TfLiteGpuDelegateAssignGLBufferToTensor();
// check the installed header tflite/delegates/gpu/gl/gl_buffer.h for
// the current signature. The function requires the GPU delegate to be
// created with experimental_flags enabling the advanced buffer API.
//
// Pseudocode — verify against installed headers before use:
GLuint gl_buffer = ImportAHardwareBufferAsGLBuffer(ahwb); // platform-specific
TfLiteGpuDelegateAssignGLBufferToTensor(gpu_delegate, tensor_index, gl_buffer);
```

**Zero-copy flow:** Camera2 `ImageReader` → `Image.getHardwareBuffer()` → `AHardwareBuffer` → GPU delegate input tensor (imported as GL SSBO) → GLSL compute dispatch → output tensor (GPU-resident) → SurfaceFlinger via `ASurfaceTransaction_setBuffer()` or Vulkan via `VK_ANDROID_external_memory_android_hardware_buffer`.

The critical detail is that once the `AHardwareBuffer` is bound to the GPU delegate's input tensor, `Interpreter::Invoke()` initiates a compute dispatch that reads the buffer directly from the GPU L2 cache, never involving a CPU data transfer. The caller must ensure the Camera2 acquisition fence has been signalled before invoking — use Android's `Sync` fence API (`sync_wait()` or `eglClientWaitSyncKHR()`) to block until the HAL has finished writing. [Source: Ch85, SurfaceFlinger sync fence discussion]

### 4.4 Pixel Tensor Inputs (Image Format)

For image-format inputs (e.g., RGB float textures for vision models), the GPU delegate can read from an `EGLImage` backed by an `AHardwareBuffer` with `AHARDWAREBUFFER_FORMAT_R8G8B8A8_UNORM`. The `TfLiteGpuDelegateOptionsV2` must have quantisation inference enabled if the model expects normalised `uint8` inputs. A GLSL preprocessing shader scales `[0, 255]` → `[-1.0, 1.0]` before the first convolution layer. This preprocessing pass executes on-GPU alongside the rest of inference, adding negligible overhead relative to a CPU-side normalisation loop.

---

## 5. Model Formats and Quantisation

### 5.1 FlatBuffer Schema

The `.tflite` schema (`tflite/schema/schema.fbs`, current schema version 3) encodes quantisation parameters at the per-tensor level using `QuantizationParameters`: an array of `scale` values and `zero_point` offsets, supporting per-channel quantisation for weights and per-tensor quantisation for activations. [Source: LiteRT schema FBS](https://github.com/google-ai-edge/LiteRT/blob/main/tflite/schema/schema.fbs)

### 5.2 INT8 Post-Training Quantisation (PTQ)

INT8 PTQ reduces a float32 model's weight and activation tensors to 8-bit signed integers. The primary benefits are 4× weight memory reduction and faster arithmetic on DSP/NPU hardware (which often has native INT8 MACs but no float32 MAC units). The typical conversion workflow uses TensorFlow's `tf.lite.TFLiteConverter`:

```python
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model("saved_model/")
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# Provide a representative dataset for activation range calibration
def representative_data_gen():
    for image in calibration_images[:100]:
        yield [image[np.newaxis, :, :, :].astype(np.float32)]

converter.representative_dataset = representative_data_gen
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type  = tf.int8
converter.inference_output_type = tf.int8

tflite_model = converter.convert()
```

INT8 quantisation typically achieves 2–4× latency speedup on NNAPI/NPU and 1.5–2× on the GPU delegate, with <1% accuracy loss on most vision models when a good calibration dataset is used.

### 5.3 INT4 Quantisation for On-Device LLMs

LiteRT 2.x supports block-wise INT4 weight quantisation targeting large language models. INT4 stores two weight values per byte, achieving 8× compression relative to float32. This enables Gemma 2B (2 billion parameters) to reside in approximately 1 GB of device RAM, within reach of mid-range Android devices. The quantisation scheme is group-wise (block size 32 or 64 weights), storing a float16 scale per group. Activations remain in float16 at runtime.

```python
# ai-edge-torch export with INT4 quantization
import ai_edge_torch
import torch

model = MyGemmaModel()
sample_inputs = (torch.randint(0, 32000, (1, 128)),)  # token ids

# INT4 weight-only quantization via ai_edge_torch
edge_model = ai_edge_torch.convert(
    model.eval(), sample_inputs,
    quant_config=ai_edge_torch.quantize.quant_config.QuantConfig(
        generative_weights=ai_edge_torch.quantize.QuantizationRecipe.INT4_WEIGHT_ONLY
    )
)
edge_model.export("gemma2b_int4.tflite")
```
[Source: ai-edge-torch](https://github.com/google-ai-edge/ai-edge-torch)

### 5.4 Dynamic Range Quantisation

Dynamic range quantisation quantises weights to INT8 at conversion time but dequantises them to float32 at runtime before MAC operations. This yields ~2× weight-memory reduction with minimal accuracy impact, and requires no calibration dataset. Activations remain in float32, so it is less effective on NPUs that prefer fully-quantised INT8 graphs.

### 5.5 Multi-Signature Models

`SignatureDef` allows multiple named inference entry points in a single `.tflite` file. Call `interpreter->GetSignatureRunner("encode")` and `interpreter->GetSignatureRunner("decode")` to switch between subgraphs. This pattern is used by LLM tokeniser+model combos and by multi-task models that share a backbone but expose separate task heads.

### 5.6 ONNX: The Open Neural Network Exchange Format

**ONNX** (Open Neural Network Exchange) is an open standard for representing machine learning models, originally developed by Microsoft and Facebook in 2017 and now governed by the Linux Foundation's LF AI & Data. It defines a common intermediate representation that allows models trained in one framework (PyTorch, TensorFlow, JAX, scikit-learn) to be loaded and executed by any compatible runtime. [Source: ONNX specification](https://onnx.ai/onnx/intro/concepts.html)

#### Format Structure

An ONNX model is a Protobuf-serialised `ModelProto` containing:
- A **graph** (`GraphProto`) of nodes, each representing a single operator (Conv, MatMul, Relu, etc.)
- **Initializers**: constant weight tensors stored inline in the file
- **Inputs / outputs**: typed tensor descriptors with optional shape information (static or symbolic dynamic shapes)
- **Opset imports**: the operator set version(s) the model uses

```python
import onnx
model = onnx.load("resnet50.onnx")
print(f"Opset: {model.opset_import[0].version}")   # e.g. 17
print(f"IR version: {model.ir_version}")            # e.g. 8
onnx.checker.check_model(model)                     # validate graph integrity
```

#### Opset Versions

ONNX operators are versioned independently of the file format via **opsets**. Each opset increment may add new operators, change operator semantics, or deprecate old ones. Runtimes declare which opsets they support; a model targeting opset 17 will not run on a runtime that only supports opset 13.

| Opset | ONNX version | Notable additions |
|---|---|---|
| 13 | 1.10 (2021) | `Squeeze`/`Unsqueeze` shape-input; quantised ops |
| 17 | 1.13 (2022) | `LayerNormalization`, `BlackmanWindow`, STFT ops |
| 18 | 1.14 (2023) | `CenterCropPad`, grouped `Conv` improvements |
| 19 | 1.15 (2023) | `AveragePool` dilations; `Identity` extended |
| 20 | 1.16 (2024) | `ConstantOfShape` extensions; `IsInf` / `IsNaN` updates |
| 21 | 1.17 (2024) | `StringConcat`, `RegexFullMatch`; `RoiAlign` extensions |

Mobile runtimes typically support up to opset 17–19. Models exported from modern PyTorch (`torch.onnx.export` with `opset_version=17`) are broadly compatible. Opset 20+ features may not be supported by ONNX Runtime Mobile or LiteRT's ONNX ingestion path.

#### Exporting to ONNX

**From PyTorch** (the most common source):
```python
import torch
import torch.onnx

model = MyModel().eval()
dummy_input = torch.randn(1, 3, 224, 224)

torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    opset_version=17,
    input_names=["input"],
    output_names=["output"],
    dynamic_axes={"input": {0: "batch"}, "output": {0: "batch"}},  # dynamic batch
    export_params=True,   # embed weights
)
```

**From TensorFlow / Keras** via `tf2onnx`:
```bash
pip install tf2onnx
python -m tf2onnx.convert \
    --saved-model ./saved_model_dir \
    --output model.onnx \
    --opset 17
```

**Verification and inspection:**
```python
import onnx
from onnx import shape_inference

model = onnx.load("model.onnx")
onnx.checker.check_model(model)                    # raises if graph is invalid
inferred = shape_inference.infer_shapes(model)     # propagate tensor shapes
onnx.save(inferred, "model_with_shapes.onnx")

# Visualise with Netron: https://netron.app — drag and drop .onnx file
```

#### ONNX on Android: Conversion vs Direct Execution

There are two strategies for using an ONNX model on Android:

1. **Convert to `.tflite`** (LiteRT path): use `ai-edge-torch` (for PyTorch-origin models) or `onnx-tf` + `tf.lite.TFLiteConverter` (for any ONNX model) to produce a `.tflite` artefact, then run with the LiteRT engine and its full delegate ecosystem (GPU, QNN, XNNPack). This is the recommended path when the model's operators map cleanly to LiteRT's op set.

2. **Run ONNX natively** with ONNX Runtime Mobile (§5.7): keep the `.onnx` file and execute it with ORT Mobile. This avoids the conversion step and is preferred when the model uses opsets or operators that LiteRT's converter does not support (e.g., attention layers with complex reshape patterns, newer ONNX operator versions).

| Approach | Keeps `.onnx` | LiteRT delegates | Conversion required | Operator coverage |
|---|---|---|---|---|
| Convert → `.tflite` | No | Full (QNN, GPU, XNNPack) | Yes | Limited by converter |
| ORT Mobile | Yes | Partial (NNAPI EP deprecated; QNN EP available) | No | Broader ONNX opset support |

**ONNX Model Zoo.** Microsoft maintains a curated set of pre-trained ONNX models at [github.com/onnx/models](https://github.com/onnx/models), including ResNet, EfficientNet, BERT, YOLOv8, and Whisper. These are ready-to-use `.onnx` files suitable for direct ORT Mobile execution or conversion to `.tflite`.

### 5.7 ONNX Runtime Mobile on Android

**ONNX Runtime Mobile** (ORT Mobile) is Microsoft's on-device inference engine for Android and iOS. It runs `.onnx` models (or the pre-packaged `.ort` format) without requiring conversion to a platform-native format. ORT Mobile shares its core with the server-side ONNX Runtime but is compiled with a reduced operator set and optimised for mobile memory and latency constraints. [Source: ORT Mobile documentation](https://onnxruntime.ai/docs/tutorials/mobile/)

#### Android Integration

ORT Mobile is distributed as a Maven AAR:

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.microsoft.onnxruntime:onnxruntime-android:1.19.2")
    // or the GPU-enabled variant:
    // implementation("com.microsoft.onnxruntime:onnxruntime-gpu-android:1.19.2")
}
```

The `onnxruntime-android` AAR contains the ORT native library (`libonnxruntime.so`) for `arm64-v8a` and `armeabi-v7a`, the Java/Kotlin API classes, and the default CPU Execution Provider. The `onnxruntime-gpu-android` variant additionally includes the NNAPI and GPU Execution Providers.

#### Kotlin Inference API

```kotlin
import ai.onnxruntime.OrtEnvironment
import ai.onnxruntime.OrtSession
import ai.onnxruntime.OnnxTensor
import java.nio.FloatBuffer

// 1. Create environment (one per process):
val env = OrtEnvironment.getEnvironment()

// 2. Create session from assets:
val modelBytes = context.assets.open("model.onnx").readBytes()
val sessionOptions = OrtSession.SessionOptions().apply {
    setIntraOpNumThreads(4)           // parallelism within a single op
    setInterOpNumThreads(1)           // parallelism between independent ops
    setOptimizationLevel(OrtSession.SessionOptions.OptLevel.ALL_OPT)
    // Append NNAPI EP if available (API 27+, deprecated API 35+):
    // addNnapi()
}
val session = env.createSession(modelBytes, sessionOptions)

// 3. Inspect model I/O:
val inputName = session.inputNames.iterator().next()  // e.g. "input"
println("Output names: ${session.outputNames}")

// 4. Prepare input tensor (1×3×224×224 float32):
val inputData = FloatBuffer.allocate(1 * 3 * 224 * 224)
// ... fill inputData with normalised pixel values ...
val inputShape = longArrayOf(1, 3, 224, 224)
val inputTensor = OnnxTensor.createTensor(env, inputData, inputShape)

// 5. Run inference:
val results = session.run(mapOf(inputName to inputTensor))

// 6. Extract output:
val outputTensor = results["output"]?.value as Array<FloatArray>
inputTensor.close()
results.close()
```

#### Execution Providers on Android

ORT Mobile dispatches computation through **Execution Providers (EPs)**, each targeting a different hardware backend. EPs are evaluated in priority order; ops not supported by a higher-priority EP fall back to the CPU EP automatically.

| Execution Provider | Android support | Status (2026) | Notes |
|---|---|---|---|
| CPU EP | All devices | Stable; default | XNNPACK-accelerated for supported ops |
| NNAPI EP | API 27–34 | Deprecated (API 35+) | Mirrors NNAPI fate (§3.1); avoid for new projects |
| QNN EP | Snapdragon (API 29+) | Active development | Qualcomm Hexagon via QNN SDK; best on Snapdragon 8 Gen 2+ |
| CoreML EP | iOS only | N/A | |
| XNNPACK EP | All devices | Stable | CPU SIMD kernels; similar to LiteRT XNNPack delegate |
| DirectML EP | Windows only | N/A | |

**Enabling the QNN Execution Provider:**
```kotlin
val sessionOptions = OrtSession.SessionOptions().apply {
    // QNN EP: requires libQnnHtp.so on device (ships with Snapdragon devices)
    val qnnOptions = mapOf(
        "backend_path" to "/vendor/lib64/libQnnHtp.so",
        "htp_performance_mode" to "burst",   // sustained high perf
        "enable_htp_fp16_precision" to "1",  // FP16 on HTP
    )
    addQnn(qnnOptions)
}
```

The QNN EP on ORT Mobile follows the same Hexagon HTP path as LiteRT's QNN delegate (§3.3) — both ultimately call into `libQnnHtp.so`. The practical difference is the host runtime: ORT dispatches ONNX graph nodes to QNN, while LiteRT dispatches `.tflite` ops.

#### The `.ort` Pre-Optimised Format

For production deployment, convert the `.onnx` model to ORT's **pre-optimised `.ort` format** offline. This strips the graph optimisation and partitioning cost from the first inference, reducing cold-start session creation time from seconds to tens of milliseconds on a typical classification model:

```bash
# Install ORT tools:
pip install onnxruntime

# Convert .onnx → .ort with graph optimisation applied offline:
python -c "
import onnxruntime as ort
sess_options = ort.SessionOptions()
sess_options.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL
sess_options.optimized_model_filepath = 'model.ort'
ort.InferenceSession('model.onnx', sess_options)
"
```

The `.ort` file is a FlatBuffer (analogous to `.tflite`'s FlatBuffer) containing the pre-optimised graph, pre-partitioned EP assignment, and serialised weight tensors. It is typically 10–30% smaller than the source `.onnx` file because constant-folding and op fusion have already been applied.

#### Quantisation with ORT Mobile

ORT Mobile supports INT8 quantisation via the `onnxruntime.quantization` Python package:

```python
from onnxruntime.quantization import quantize_dynamic, QuantType

quantize_dynamic(
    model_input="model.onnx",
    model_output="model_int8.onnx",
    weight_type=QuantType.QInt8,      # INT8 weights, FP32 activations
    # For full static INT8 (activations too), use quantize_static() with calibration data
)
```

Dynamic quantisation compresses weight tensors to INT8 and dequantises them to FP32 at inference time — reducing model size by ~4× with minimal accuracy loss for transformer-based models. Static quantisation (INT8 weights + INT8 activations) requires a calibration dataset but achieves better latency on CPU EP and is required for QNN EP acceleration.

#### ORT Mobile vs LiteRT: When to Choose Each

| Decision factor | Prefer LiteRT | Prefer ORT Mobile |
|---|---|---|
| Model origin | TensorFlow, Keras, JAX | PyTorch, HuggingFace Transformers |
| Hardware acceleration | Full delegate ecosystem (QNN, GPU, XNNPack) | QNN EP available; GPU EP less mature |
| Operator coverage | Strong for vision/audio ops | Broader ONNX opset; better transformer support |
| Model format | `.tflite` required by existing pipeline | Can ship `.onnx`/`.ort` directly |
| Ecosystem integration | MediaPipe Tasks, LiteRT-LM, benchmark_model | ORT GenAI for on-device LLMs |
| File size overhead | ~1 MB runtime | ~5–8 MB AAR (larger runtime) |
| Google Play ML Kit | Supported | Not supported |

**ORT GenAI** (2024+) is Microsoft's extension for generative AI on ORT Mobile, supporting Phi-3-mini, Mistral-7B (quantised), and ONNX-format LLMs via the same `OrtSession` API. It competes with LiteRT-LM (§17.1) for on-device LLM inference, with ORT GenAI having an advantage for PyTorch-origin transformer models that export cleanly to ONNX but lose fidelity through the LiteRT conversion path. [Source: ORT GenAI](https://github.com/microsoft/onnxruntime-genai)

**Runtime comparison:**

| Runtime | Format | Primary Backend | Android Delegation |
|---|---|---|---|
| LiteRT | `.tflite` | GPU (GLES/Vulkan), NPU | GPU delegate, QNN, XNNPack |
| ORT Mobile | `.onnx` / `.ort` | CPU / QNN EP | QNN EP (Snapdragon); NNAPI EP deprecated |
| CoreML (iOS) | `.mlpackage` | ANE | N/A on Android |
| OpenVINO Mobile | `.xml`/`.bin` | CPU / Intel NPU | LiteRT CompiledModel (Linux/x86) |

---

### 5.8 ExecuTorch: PyTorch's Native Mobile Runtime

**ExecuTorch** is Meta's official on-device inference runtime for PyTorch models, announced at PyTorch Conference 2023 and shipped in production in WhatsApp and Instagram. It replaces the deprecated **PyTorch Mobile** (TorchScript-based, maintenance-only since 2023) with a fundamentally different architecture: rather than shipping a general interpreter, ExecuTorch compiles the model graph to a portable binary (`.pte`) containing the execution plan, constants, and operator kernels needed — nothing more. [Source: ExecuTorch documentation](https://pytorch.org/executorch/stable/)

#### 5.8.1 The Export Pipeline

The ExecuTorch export pipeline has three stages:

```python
import torch
from executorch.exir import to_edge
from torch.export import export

# Stage 1: torch.export — capture the full computation graph
model = MobileNetV3(pretrained=True).eval()
example_input = (torch.randn(1, 3, 224, 224),)
exported = export(model, example_input)

# Stage 2: to_edge — lower to ExecuTorch's Edge IR (subset of ATen ops)
edge_program = to_edge(exported)

# Stage 3: to_executorch — partition for delegates, serialise to .pte
et_program = edge_program.to_executorch()
with open("mobilenet_v3.pte", "wb") as f:
    f.write(et_program.buffer)
```

**Key design difference from TorchScript**: `torch.export.export()` performs *ahead-of-time* full-graph capture with symbolic shapes rather than tracing. It produces a strict `ExportedProgram` with no Python fallbacks, ensuring the resulting `.pte` contains only statically-resolved operators that the runtime can execute without an interpreter loop. [Source: torch.export documentation](https://pytorch.org/docs/stable/export.html)

**Edge IR** is an ATen-dialect IR (a subset of ~180 core ATen operators) that sits below `torch.export`'s full ATen IR. Lowering to Edge IR eliminates training-only operators and resolves dtype/layout constraints that would be ambiguous on mobile hardware. [Source: ExecuTorch EXIR spec](https://pytorch.org/executorch/stable/ir-exir.html)

#### 5.8.2 Android Integration

ExecuTorch ships as an **Android AAR** available from Maven Central:

```kotlin
// build.gradle.kts (app)
dependencies {
    implementation("org.pytorch:executorch-android:0.4.0")
    // Optional delegates
    implementation("org.pytorch:executorch-android-xnnpack:0.4.0")
    implementation("org.pytorch:executorch-android-vulkan:0.4.0")
}
```

The runtime API is minimal by design — the execution plan is pre-computed in the `.pte`:

```kotlin
import org.pytorch.executorch.EValue
import org.pytorch.executorch.Module
import org.pytorch.executorch.Tensor

val module = Module.load(assetFilePath(context, "mobilenet_v3.pte"))

val inputData = FloatArray(1 * 3 * 224 * 224) { /* ... */ }
val inputTensor = Tensor.fromBlob(inputData, longArrayOf(1, 3, 224, 224))

val outputs: Array<EValue> = module.forward(EValue.from(inputTensor))
val outputTensor = outputs[0].toTensor()
val scores = outputTensor.dataAsFloatArray
```

The `Module.load()` call memory-maps the `.pte` — no deserialization copy — and the `forward()` call dispatches directly to the pre-planned execution sequence. Cold-start latency is substantially lower than ORT Mobile or LiteRT because there is no graph optimization pass at load time. [Source: ExecuTorch Android tutorial](https://pytorch.org/executorch/stable/demo-apps-android.html)

#### 5.8.3 Delegate Backends

ExecuTorch's partitioner-delegate architecture allows selective offload:

| Delegate | Target | Notes |
|---|---|---|
| **XNNPACK** | CPU NEON/SVE | Default; Meta production path; covers most float/quant8 ops |
| **Vulkan** | Adreno/Mali GPU | Compute shader backend; covers linear/conv/ReLU |
| **QNN** | Snapdragon NPU | Qualcomm AI Engine Direct; requires Qualcomm SDK |
| **CoreML** | Apple ANE (iOS only) | N/A on Android |
| **Custom** | Any | Implement `BackendInterface` in C++ |

The **XNNPACK delegate** is Meta's primary production backend for Android, covering the float32 and per-channel int8 operator set used in WhatsApp's ML features. The **Vulkan delegate** is newer and covers a subset of operators; for full coverage, operators not claimed by Vulkan fall back to the XNNPACK kernel.

Partitioning is performed at export time:

```python
from executorch.backends.xnnpack.partition.xnnpack_partitioner import XnnpackPartitioner

edge_program = to_edge(exported)
# Partition ops that XNNPACK can handle; remainder stays on ATen CPU
edge_program = edge_program.to_backend(XnnpackPartitioner())
et_program = edge_program.to_executorch()
```

[Source: ExecuTorch XNNPACK backend](https://pytorch.org/executorch/stable/tutorial-xnnpack-delegate-lowering.html)

#### 5.8.4 Quantization

ExecuTorch uses **PyTorch 2.0 Export Quantization** (`torch.ao.quantization`) rather than LiteRT's post-training quantization API. The flow applies quantization at the `torch.export` stage:

```python
from torch.ao.quantization.quantize_pt2e import convert_pt2e, prepare_pt2e
from torch.ao.quantization.quantizer.xnnpack_quantizer import (
    XNNPACKQuantizer, get_symmetric_quantization_config
)

quantizer = XNNPACKQuantizer()
quantizer.set_global(get_symmetric_quantization_config(is_per_channel=True))

# Calibrate
prepared = prepare_pt2e(exported, quantizer)
for sample in calibration_data:
    prepared(sample)

# Convert to int8
quantized = convert_pt2e(prepared)
edge_program = to_edge(quantized)
et_program = edge_program.to_executorch()
```

INT8 symmetric per-channel quantization (the XNNPACK default) typically achieves <1% accuracy loss on classification models. INT4 support is available via the `Int4WeightOnlyQuantizer` for LLM weight compression. [Source: ExecuTorch quantization tutorial](https://pytorch.org/executorch/stable/quantization-overview.html)

#### 5.8.5 ExecuTorch vs LiteRT vs ORT Mobile

| Factor | ExecuTorch | LiteRT | ORT Mobile |
|---|---|---|---|
| Native model origin | PyTorch | TensorFlow/Keras/JAX | PyTorch/HuggingFace |
| Format | `.pte` | `.tflite` | `.onnx` / `.ort` |
| Runtime size (AAR) | ~2–4 MB (modular) | ~1–2 MB | ~5–8 MB |
| Cold-start overhead | Very low (pre-planned) | Low (load-time optimization) | Medium (graph optimization at load) |
| GPU acceleration | Vulkan delegate (partial) | GPU delegate (full, GLES+Vulkan) | Vulkan EP (experimental) |
| NPU acceleration | QNN delegate | QNN delegate, GPU delegate | QNN EP |
| LLM support | `executorch-llm` (Llama 3) | LiteRT-LM | ORT GenAI |
| Maintenance status | Active (Meta production) | Active (Google production) | Active (Microsoft) |
| PyTorch ecosystem fit | Native | Requires ONNX or ai-edge-torch | Requires ONNX export |

ExecuTorch is the correct choice when: (1) the model is trained in PyTorch and must remain on the PyTorch operator set without format conversion, (2) Meta-compatible licensing and tooling is acceptable, or (3) the team already uses `torch.compile` / `torch.export` in the training pipeline.

---

### 5.9 JAX and StableHLO: Export Path to LiteRT

**JAX** (developed by Google DeepMind) is a NumPy-compatible array framework with XLA-based JIT compilation. Unlike PyTorch, JAX has no dedicated mobile inference runtime — instead, JAX models export to **StableHLO** (a stable subset of HLO/MHLO), which then converts to `.tflite` via the `ai-edge-torch` toolchain. [Source: JAX documentation](https://jax.readthedocs.io/en/latest/) [Source: StableHLO spec](https://openxla.org/stablehlo)

#### 5.9.1 The StableHLO Ecosystem

**StableHLO** is an open standard for ML computation defined by the OpenXLA project (Google, Meta, AMD, Nvidia, Intel). It provides:

- A stable opset with forward/backward compatibility guarantees (unlike HLO, which changes with XLA releases)
- A textual IR (`func.func @main(%arg0: tensor<1x224x224x3xf32>`) based on MLIR
- Serialization to `stablehlo.mlirbc` (MLIR bytecode)
- Consumption by LiteRT, IREE, TensorFlow, and JAX itself

StableHLO sits at the *same abstraction level* as ONNX — it's a portability interchange format, not a runtime. Both XLA (via JAX/TensorFlow) and PyTorch 2.0 (via `torch-mlir`) can produce StableHLO. [Source: OpenXLA StableHLO repository](https://github.com/openxla/stablehlo)

#### 5.9.2 JAX Export Flow

```python
import jax
import jax.numpy as jnp
import numpy as np

# Define a simple CNN inference function
def predict(params, x):
    # ... JAX functional model
    return output

# JIT-compile and export to StableHLO
predict_jit = jax.jit(predict, static_argnames=())
example_input = jnp.ones((1, 224, 224, 3), dtype=jnp.float32)

# JAX 0.4.25+: jax.export produces a StableHLO artifact
exported = jax.export.export(
    predict_jit,
    platforms=['cpu']  # or 'tpu', 'cuda'
)(jax.ShapeDtypeStruct((1, 224, 224, 3), jnp.float32))

stablehlo_text = exported.mlir_module()
```

[Source: jax.export API](https://jax.readthedocs.io/en/latest/export/)

#### 5.9.3 StableHLO to LiteRT: ai-edge-torch and ai-edge-litert

Google provides the **`ai-edge-torch`** library to convert PyTorch and StableHLO models to `.tflite`. For JAX models, the path is:

```bash
pip install ai-edge-litert-nightly
```

```python
import ai_edge_litert
from ai_edge_litert.compiler import LiteRTCompiler

# Convert StableHLO bytecode to .tflite
compiler = LiteRTCompiler()
litert_model = compiler.compile_from_stablehlo(
    exported.mlir_module_serialized,   # bytes: MLIR bytecode
    input_shapes=[(1, 224, 224, 3)],
    input_dtypes=["float32"],
)
litert_model.export("jax_model.tflite")
```

Alternatively, for **Flax** (the common JAX neural network library) models trained with Orbax checkpoints, Google provides a reference conversion path via `jax2tf` (stable) then TFLiteConverter:

```python
import jax
import tensorflow as tf
from jax.experimental import jax2tf

# Legacy path (stable): JAX → TF SavedModel → TFLite
tf_predict = tf.function(
    jax2tf.convert(predict, enable_xla=False),
    input_signature=[tf.TensorSpec((1, 224, 224, 3), tf.float32)]
)

converter = tf.lite.TFLiteConverter.from_concrete_functions(
    [tf_predict.get_concrete_function()]
)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_model = converter.convert()
with open("jax_model.tflite", "wb") as f:
    f.write(tflite_model)
```

The `enable_xla=False` flag forces `jax2tf` to use only TensorFlow ops (no XLA-specific ops), which is required for TFLiteConverter compatibility. The XLA path (`enable_xla=True`) produces ops that the TFLite runtime cannot execute. [Source: jax2tf documentation](https://github.com/google/jax/tree/main/jax/experimental/jax2tf)

#### 5.9.4 Flax/Orbax Checkpoint Integration

For production JAX/Flax models, the typical Android deployment path:

```python
import orbax.checkpoint as ocp
import flax.linen as nn

class MobileModel(nn.Module):
    @nn.compact
    def __call__(self, x):
        x = nn.Conv(32, (3, 3))(x)
        x = nn.relu(x)
        x = nn.avg_pool(x, window_shape=(2, 2), strides=(2, 2))
        x = x.reshape((x.shape[0], -1))
        x = nn.Dense(10)(x)
        return x

# Restore from Orbax checkpoint
checkpointer = ocp.StandardCheckpointer()
params = checkpointer.restore("checkpoint_dir/", target=None)

# Bind params and export
model = MobileModel()
bound_predict = lambda x: model.apply({"params": params["params"]}, x)

# → jax2tf → TFLiteConverter pipeline as above
```

[Source: Orbax documentation](https://orbax.readthedocs.io/)

#### 5.9.5 Quantization for JAX → LiteRT

The `jax2tf` → TFLiteConverter path supports the same post-training quantization options as native TF models:

```python
converter = tf.lite.TFLiteConverter.from_concrete_functions([cf])
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# INT8 full quantization requires a representative dataset
def representative_data_gen():
    for _ in range(100):
        yield [np.random.randn(1, 224, 224, 3).astype(np.float32)]

converter.representative_dataset = representative_data_gen
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.int8
converter.inference_output_type = tf.int8

int8_model = converter.convert()
```

JAX's native **quantization-aware training (QAT)** uses `aqt` (Accurate Quantized Training) from Google, but `aqt`-quantized models require conversion via the StableHLO path (not `jax2tf`) to preserve quantization annotations in the resulting `.tflite`. The `ai-edge-litert` compiler handles this correctly while `jax2tf` loses the annotations. [Source: AQT library](https://github.com/google/aqt)

#### 5.9.6 JAX vs PyTorch Mobile Paths: Summary

| Aspect | JAX → LiteRT | PyTorch → ExecuTorch | PyTorch → ORT Mobile |
|---|---|---|---|
| Export mechanism | `jax.export` / `jax2tf` | `torch.export.export()` | `torch.onnx.export()` |
| Intermediate format | StableHLO / TF SavedModel | ExecuTorch Edge IR | ONNX |
| Mobile runtime | LiteRT (Google) | ExecuTorch (Meta) | ORT Mobile (Microsoft) |
| Native mobile format | `.tflite` | `.pte` | `.onnx` / `.ort` |
| Hardware delegation | Full LiteRT delegate stack | XNNPACK, Vulkan, QNN | QNN EP, CPU |
| QAT support | AQT → StableHLO path | PT2E quantization | ORT quantization |
| LLM path | LiteRT-LM / Gemini Nano | executorch-llm (Llama 3) | ORT GenAI |
| Production status | Google (internal) / research | Meta production (2023+) | Microsoft production |

**Key insight**: JAX has no dedicated mobile runtime. The Android deployment story for JAX models is always mediated by LiteRT — either via the `jax2tf` + TFLiteConverter legacy path (stable, widely supported) or the `jax.export` + StableHLO + `ai-edge-litert` path (newer, required for QAT models). Teams training in JAX/Flax who need Android inference should plan the LiteRT conversion step into their model pipeline from the start, as it constrains op choice (particularly XLA-specific ops that have no TFLite equivalent).

---

## 6. Model Conversion and Quantization Workflow

Producing a production-ready `.tflite` model involves more than a single API call. This section covers the full toolchain from training framework to deployed INT4/INT8/FP16 artefact, including the key decision points around quantisation strategy. The quantisation *theory* (scale/zero-point encoding, schema representation) is covered in §5; this section focuses on the *process and tooling*. [Source: LiteRT model conversion guide](https://ai.google.dev/edge/litert/models/convert/convert_models) [Source: LiteRT optimisation guide](https://ai.google.dev/edge/litert/models/optimize/)

### 6.1 Conversion Pipeline Overview

The general path from a trained model to a deployed LiteRT artefact is:

```
PyTorch / JAX / Keras
    │ ai-edge-torch / tf.lite.TFLiteConverter
    ▼
.tflite FlatBuffer (fp32)
    │ post-training quantization or QAT
    ▼
.tflite (INT8 / INT4 / FP16)
    │ ai-edge-quantizer (advanced) or TFLiteConverter
    ▼
Deploy to LiteRT
```

Choosing the entry point depends on the training framework. TensorFlow/Keras users go directly through `tf.lite.TFLiteConverter`. PyTorch users use `ai-edge-torch`, which reached stable release in 2024 and is now the preferred path for PyTorch-originating models. JAX users can export via `orbax-export` to SavedModel format, then apply `TFLiteConverter`. Each path produces semantically identical FlatBuffer artefacts.

### 6.2 TFLiteConverter for TensorFlow and Keras Models

`tf.lite.TFLiteConverter` is the canonical tool for TensorFlow SavedModel and Keras model formats. The following converts a SavedModel to a fully integer-quantised INT8 model suitable for NPU deployment:

```python
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model("saved_model_dir")
converter.optimizations = [tf.lite.Optimize.DEFAULT]  # enables PTQ

# Provide a representative dataset for INT8 calibration
def representative_dataset():
    for image, _ in val_dataset.take(100):
        yield [tf.cast(image, tf.float32)]

converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.int8
converter.inference_output_type = tf.int8
tflite_model = converter.convert()

with open("model_int8.tflite", "wb") as f:
    f.write(tflite_model)
```

Setting `inference_input_type = tf.int8` means the model expects pre-quantised INT8 input — the application is responsible for converting the raw float image to INT8 before calling `set_tensor()`. If this is inconvenient, omit both `inference_input_type` and `inference_output_type`; the model will accept float32 input and output, with quantisation/dequantisation ops inserted automatically at the graph boundary.

### 6.3 ai-edge-torch for PyTorch Models

`ai-edge-torch` converts a `torch.nn.Module` directly to a `.tflite` FlatBuffer without going through TensorFlow. It uses `torch.export` (PyTorch 2.x) to capture a static computation graph and then lowers it to LiteRT operators:

```python
import ai_edge_torch
import torch

model = MobileNetV3().eval()
sample_input = (torch.randn(1, 3, 224, 224),)

edge_model = ai_edge_torch.convert(model, sample_input)
edge_model.export("mobilenet_v3.tflite")
```

For quantised export, `ai-edge-torch` integrates with PyTorch's `torch.ao.quantization` pipeline:

```python
from ai_edge_torch.quantize import pt2e_quantizer, quant_config

quantizer = pt2e_quantizer.PT2EQuantizer().set_global(
    quant_config.get_symmetric_quantization_config(is_per_channel=True)
)
edge_model = ai_edge_torch.convert(model, sample_input, quant_config=quantizer)
edge_model.export("mobilenet_v3_int8.tflite")
```

[Source: ai-edge-torch GitHub](https://github.com/google-ai-edge/ai-edge-torch)

### 6.4 PTQ versus QAT: When to Use Each

**Post-Training Quantisation (PTQ)** requires no retraining. The converter runs a calibration pass over the representative dataset (100–500 samples is usually sufficient) to determine activation value ranges, then selects the per-layer scale and zero-point that minimises quantisation error. PTQ is fast — typically minutes on a laptop GPU — and for most convolutional vision models achieves accuracy loss below 1% on standard metrics (top-1 ImageNet, COCO mAP).

**Quantization-Aware Training (QAT)** inserts fake-quantisation nodes into the model's forward pass during training, so the model learns to compensate for the rounding noise introduced by quantisation. The resulting model is typically within 0.5% of the FP32 baseline. QAT requires a training infrastructure, access to the training data, and training time (hours to days), so it is reserved for cases where PTQ accuracy degradation exceeds the acceptable threshold.

A practical decision rule: run PTQ first and measure accuracy on your target evaluation set. If the degradation exceeds 2% on the primary metric, switch to QAT. Common cases where PTQ struggles include:
- Object detection models with complex feature pyramid networks (FPN)
- Transformer-based models with attention mechanisms (activation distributions are bimodal, hard to calibrate with a single scale)
- LSTM/GRU-based sequence models (recurrent activations have high dynamic range)
- Models with a small number of output classes where per-class accuracy matters more than top-1

### 6.5 Representative Dataset Selection

The quality of INT8 PTQ depends critically on the representative dataset. The calibration samples determine the min/max (or percentile) of each activation tensor's range, which in turn sets the quantisation scale. Biased calibration — using images from a single class, or using training images that differ in distribution from deployment — will produce sub-optimal scales that hurt accuracy in deployment.

Guidelines: use 100–500 samples drawn uniformly from the *validation* set (not training); ensure the samples cover the full range of expected deployment inputs (all lighting conditions, all relevant object categories); avoid duplicate frames from video sequences, which inflate the apparent frequency of a particular activation range. If the deployment domain differs from the training domain (e.g., industrial camera with different white balance), use samples from the deployment domain for calibration.

### 6.6 Per-Channel vs Per-Tensor Quantisation

LiteRT supports two quantisation granularities for weight tensors:

**Per-tensor quantisation** assigns a single scale and zero-point to the entire weight tensor. It is simpler to implement in hardware and produces smaller model metadata, but results in ~0.5–1% more accuracy loss than per-channel on deep convolutional networks, because different output channels in a convolution layer typically have very different weight magnitude ranges.

**Per-channel quantisation** (also called per-output-channel) assigns a separate scale to each output filter. This is the LiteRT default when `target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]` is set. It is particularly important for `DEPTHWISE_CONV_2D`, where channel-to-channel weight magnitude variation is high and per-tensor quantisation often causes >2% accuracy loss.

Activations are always quantised per-tensor in LiteRT's current implementation, since per-channel activation quantisation is rarely supported in hardware and adds significant complexity to the runtime.

### 6.7 ai-edge-quantizer for Advanced Scenarios

For production deployments requiring selective quantisation or INT4 weight-only quantisation, the `ai-edge-quantizer` package provides more control than `TFLiteConverter`:

```python
from ai_edge_quantizer import quantizer, algorithm_manager

quantization_config = quantizer.QuantizationConfig(
    algorithm_key=algorithm_manager.AlgorithmName.MIN_MAX_UNIFORM_QUANTIZE,
    op_selection_config=quantizer.OpSelectionConfig(
        # Skip the first and last layers, which are sensitivity to quantisation:
        skip_ops_with_output_layer_count=[0, -1]
    )
)
qt = quantizer.Quantizer("model.tflite", quantization_config)
qt.load_representative_dataset(representative_dataset_gen)
result = qt.quantize()
result.export_model("model_quantized.tflite")
```

`ai-edge-quantizer` supports selective quantisation (skip sensitive layers), INT4 weight-only quantisation (2× weight compression vs INT8), and clustering quantisation (grouping weights into centroid clusters to preserve accuracy). [Source: ai-edge-quantizer](https://github.com/google-ai-edge/ai-edge-quantizer)

### 6.8 INT4 Weight-Only Quantisation for LLMs

For transformer decoder models (LLMs), the dominant compute pattern is weight-dequantise-then-matrix-multiply (GeMM-bound). INT4 weight-only quantisation stores weights in 4-bit groups of 32 or 64 elements, each group with a FP16 scale. Activations remain in FP16 at runtime; only weights are stored compressed. This achieves 2× memory reduction versus INT8, enabling Gemma 2B-class models (~2 billion parameters) to reside in approximately 1 GB of GPU high-bandwidth memory on mid-range Android devices.

```python
converter = tf.lite.TFLiteConverter.from_saved_model("gemma_2b_saved_model")
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.target_spec.supported_types = [tf.float16, tf.int4]
tflite_model = converter.convert()
```

Note: INT4 weight-only is supported by LiteRT-LM's GPU delegate (OpenCL backend) and by the Qualcomm QNN delegate (v2.22+) and MediaTek NeuroPilot (Dimensity 9400+). The standard GPU delegate (`TfLiteGpuDelegateV2`) does not yet support INT4.

### 6.9 Measuring Accuracy Regression

After conversion and quantisation, always compare output on a held-out test set before and after:

```python
import numpy as np
from tflite_runtime.interpreter import Interpreter

interp_fp32 = Interpreter("model_fp32.tflite")
interp_int8 = Interpreter("model_int8.tflite")

for interp in [interp_fp32, interp_int8]:
    interp.allocate_tensors()

correct_fp32, correct_int8 = 0, 0
for image, label in test_dataset:
    for interp, counter in [(interp_fp32, "fp32"), (interp_int8, "int8")]:
        interp.set_tensor(interp.get_input_details()[0]["index"], image)
        interp.invoke()
        pred = np.argmax(interp.get_tensor(
            interp.get_output_details()[0]["index"]))
        if pred == label:
            if counter == "fp32":
                correct_fp32 += 1
            else:
                correct_int8 += 1

print(f"FP32 accuracy: {correct_fp32/len(test_dataset):.3f}")
print(f"INT8 accuracy: {correct_int8/len(test_dataset):.3f}")
print(f"Accuracy delta: {(correct_fp32 - correct_int8)/len(test_dataset):.3f}")
```

A delta exceeding 2% on top-1 accuracy for classification, 1.5 mAP for detection, or 2% WER for speech recognition is a signal to switch from PTQ to QAT or to increase the bit width of the sensitive layers.

---

## 7. MediaPipe Framework Architecture

**MediaPipe** is Google's cross-platform framework for building live, streaming ML pipelines from modular, composable processing nodes. It is developed in the [google-ai-edge/mediapipe](https://github.com/google-ai-edge/mediapipe) repository and supports Android, iOS, Linux, macOS, and Web (via WASM). [Source: MediaPipe framework](https://developers.google.com/mediapipe/framework)

### 7.1 CalculatorGraph

The central abstraction is the `CalculatorGraph`: a directed acyclic graph (DAG) of `Calculator` nodes connected by typed, timestamped **Packet** streams. Graphs are defined as Protocol Buffer text (`CalculatorGraphConfig`) and compiled into a validated execution graph at runtime.

```proto
// Minimal pose inference graph (text proto)
input_stream: "input_video"
output_stream: "pose_landmarks"

node {
  calculator: "ImageToTensorCalculator"
  input_stream: "IMAGE:input_video"
  output_stream: "TENSORS:input_tensors"
  options { [mediapipe.ImageToTensorCalculatorOptions.ext] {
    output_tensor_width: 256
    output_tensor_height: 256
    keep_aspect_ratio: false
    output_tensor_float_range { min: -1.0 max: 1.0 }
    gpu_origin: TOP_LEFT
  }}
}
node {
  calculator: "TfLiteInferenceCalculator"
  input_stream: "TENSORS:input_tensors"
  output_stream: "TENSORS:output_tensors"
  options { [mediapipe.TfLiteInferenceCalculatorOptions.ext] {
    model_path: "pose_landmark_lite.tflite"
    delegate { gpu {} }
  }}
}
node {
  calculator: "TfLiteTensorsToLandmarksCalculator"
  input_stream: "TENSORS:output_tensors"
  output_stream: "NORM_LANDMARKS:pose_landmarks"
}
```
[Source: MediaPipe framework graph documentation](https://developers.google.com/mediapipe/framework/framework_concepts/calculators)

### 7.2 Calculator Lifecycle

Every `Calculator` subclass implements:

```cpp
class MyCalculator : public mediapipe::CalculatorBase {
 public:
  // Declares input/output stream types and side-packet types
  static absl::Status GetContract(CalculatorContract* cc);

  // Called once before the first Process(); open resources
  absl::Status Open(CalculatorContext* cc) override;

  // Called once per packet; do computation here
  absl::Status Process(CalculatorContext* cc) override;

  // Called once after all packets are processed; release resources
  absl::Status Close(CalculatorContext* cc) override;
};
```

`GetContract()` is a static method that describes the calculator's interface to the graph validation engine. Input and output streams carry typed packets; the type is encoded as a C++ type tag (e.g., `mediapipe::ImageFrame`, `mediapipe::GpuBuffer`, `std::vector<mediapipe::Tensor>`).

### 7.3 Packets and Timestamps

A `Packet` is an immutable, typed, reference-counted data container with a `Timestamp`:

```cpp
// Wrapping data in a Packet
auto packet = mediapipe::MakePacket<mediapipe::GpuBuffer>(gpu_buffer)
                  .At(mediapipe::Timestamp(frame_timestamp_us));

// Extracting data from a Packet
const mediapipe::GpuBuffer& buffer = packet.Get<mediapipe::GpuBuffer>();
```

Timestamps are in microseconds and flow monotonically through the graph. The graph scheduler dispatches `Process()` in timestamp order, enabling correct synchronisation when multiple input streams must be aligned (e.g., combining camera frames with depth frames that arrive at different rates).

### 7.4 GlCalculatorHelper and GPU Context Management

For calculators that execute GLES commands, `GlCalculatorHelper` manages the EGL context and provides safe context-switch operations:

```cpp
class MyGlCalculator : public mediapipe::CalculatorBase {
  mediapipe::GlCalculatorHelper gpu_helper_;

  absl::Status Open(CalculatorContext* cc) override {
    // Acquires an EGL context; all GPU operations must be wrapped in
    // RunInGlContext to ensure correct context-switch semantics
    return gpu_helper_.Open(cc);
  }

  absl::Status Process(CalculatorContext* cc) override {
    return gpu_helper_.RunInGlContext([&]() -> absl::Status {
      auto src = gpu_helper_.CreateSourceTexture(
          cc->Inputs().Tag("IMAGE").Get<mediapipe::GpuBuffer>());
      gpu_helper_.BindFramebuffer(dst_texture_);
      // ... issue GLES draw calls ...
      return absl::OkStatus();
    });
  }
};
```
[Source: GlCalculatorHelper header](https://github.com/google-ai-edge/mediapipe/blob/master/mediapipe/gpu/gl_calculator_helper.h)

`RunInGlContext()` switches to the helper's EGL context, executes the lambda, and restores the previous context. This abstraction means that multiple graph threads can share a single EGL context without race conditions.

### 7.5 GpuBuffer, GlTexture, and AHardwareBuffer Interop

`mediapipe::GpuBuffer` is a platform-neutral container for GPU-resident image data. On Android it wraps an `AHardwareBuffer` via `GpuBufferStorageAhwb` (when available), falling back to `GpuBufferStorageEglImage` for older devices. [Source: MediaPipe GPU docs](https://developers.google.com/mediapipe/framework/framework_concepts/gpu)

The interop chain:

```
AHardwareBuffer (Gralloc, DMA-BUF)
    ↓ eglCreateImageKHR(EGL_ANDROID_image_native_buffer)
EGLImageKHR
    ↓ glEGLImageTargetTexture2DOES(GL_TEXTURE_EXTERNAL_OES, ...)
GL_TEXTURE_EXTERNAL_OES (samplerExternalOES in GLSL)
    ↓ GlTexture / GpuBuffer::GetGlTextureBufferSharedPtr()
mediapipe::GpuBuffer
    ↓ MakePacket<GpuBuffer>()
MediaPipe Packet
```

`AHardwareBufferView` on Android 29+ enables direct CPU access to the buffer with automatic GPU-CPU synchronisation, ensuring that GPU writes have completed before the CPU reads the data. This is used by `GpuBuffer::GetReadView<AHardwareBufferView>()` in newer MediaPipe versions.

### 7.6 InputStreamHandlers

`CalculatorGraph` supports multiple scheduling modes:

- `DefaultInputStreamHandler`: Synchronises packets across all input streams by timestamp. A calculator's `Process()` is called only when all inputs have a packet at the same timestamp. Appropriate for cameras + depth-sensor fusion.
- `ImmediateInputStreamHandler`: Calls `Process()` whenever any new packet arrives on any input stream, without waiting for other streams. Appropriate for low-latency single-stream inference.
- `FixedSizeInputStreamHandler`: Bounds queue depth to prevent backpressure accumulation during transient slowdowns.

---

## 8. MediaPipe Tasks API

The **Tasks API**, introduced in 2022, wraps `CalculatorGraph` behind strongly-typed, high-level task interfaces that hide graph configuration entirely. It is the recommended entry point for application developers. [Source: MediaPipe Tasks API](https://developers.google.com/mediapipe/solutions/guide)

### 8.1 BaseOptions and RunningMode

All Tasks share a common `BaseOptions` structure:

```kotlin
val baseOptions = BaseOptions.builder()
    .setModelAssetPath("pose_landmarker_lite.task")
    .setDelegate(Delegate.GPU)          // CPU, GPU, or NNAPI (deprecated API 35+)
    .build()
```

`RunningMode` controls the processing contract:
- `IMAGE`: Single still image input, synchronous result.
- `VIDEO`: Timestamped video frames; the task uses inter-frame state (e.g., tracking).
- `LIVE_STREAM`: Asynchronous streaming mode; results are delivered via a registered callback on a MediaPipe-managed thread. Callers must not block in the callback.

### 8.2 Available Vision Tasks

| Task | Output Type | Key Output Fields |
|---|---|---|
| `ObjectDetector` | `ObjectDetectionResult` | Bounding boxes, category labels, scores |
| `PoseLandmarker` | `PoseLandmarkerResult` | 33 normalised landmarks + 33 world landmarks (metres) per person |
| `HandLandmarker` | `HandLandmarkerResult` | 21 landmarks per hand, handedness (Left/Right + score) |
| `FaceLandmarker` | `FaceLandmarkerResult` | 478 mesh landmarks, 52 blend shape coefficients |
| `ImageClassifier` | `ClassificationResult` | Category label, score |
| `ImageSegmenter` | `ImageSegmenterResult` | Per-pixel mask tensors (confidence per category) |
| `GestureRecognizer` | `GestureRecognitionResult` | Hand gesture category, landmarks |

[Source: MediaPipe Solutions guide](https://developers.google.com/mediapipe/solutions/vision/pose_landmarker)

### 8.3 PoseLandmarker — Kotlin Live-Stream Example

```kotlin
import com.google.mediapipe.tasks.core.BaseOptions
import com.google.mediapipe.tasks.core.Delegate
import com.google.mediapipe.tasks.vision.core.RunningMode
import com.google.mediapipe.tasks.vision.poselandmarker.PoseLandmarker
import com.google.mediapipe.tasks.vision.poselandmarker.PoseLandmarker.PoseLandmarkerOptions
import com.google.mediapipe.tasks.vision.poselandmarker.PoseLandmarkerResult
import com.google.mediapipe.framework.image.MPImage

val options = PoseLandmarkerOptions.builder()
    .setBaseOptions(
        BaseOptions.builder()
            .setModelAssetPath("pose_landmarker_lite.task")
            .setDelegate(Delegate.GPU)
            .build()
    )
    .setRunningMode(RunningMode.LIVE_STREAM)
    .setNumPoses(1)
    .setMinPoseDetectionConfidence(0.5f)
    .setMinPosePresenceConfidence(0.5f)
    .setMinTrackingConfidence(0.5f)
    .setResultListener { result: PoseLandmarkerResult, input: MPImage ->
        renderLandmarks(result)
    }
    .setErrorListener { error -> Log.e(TAG, "Pose error", error) }
    .build()

val poseLandmarker: PoseLandmarker =
    PoseLandmarker.createFromOptions(context, options)

// Per camera frame (called from CameraX ImageAnalysis.Analyzer):
val mpImage = BitmapImageBuilder(bitmap).build()  // or MediaImageBuilder
poseLandmarker.detectAsync(mpImage, SystemClock.uptimeMillis())
```
[Source: MediaPipe PoseLandmarker Android guide](https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker/android)

The `detectAsync()` call is non-blocking. The Tasks runtime enqueues the frame on MediaPipe's internal `CalculatorGraph`, and the `setResultListener` callback fires on a background thread when inference completes. The 33 normalised landmarks are in `[0, 1]` image coordinates; the 33 world landmarks are in metres relative to the hip midpoint.

### 8.4 LLM Inference Task

MediaPipe's `LlmInference` task (Android/iOS) has been superseded by **LiteRT-LM** (announced May 2026), which provides a dedicated orchestration layer for autoregressive LLM inference including KV-cache management, INT4/INT8 weight dispatch, and NPU routing. The MediaPipe LLM Inference API is now in maintenance-only mode; new projects should use the LiteRT-LM Android/iOS SDK. [Source: LiteRT-LM announcement](https://developers.googleblog.com/blazing-fast-on-device-genai-with-litert-lm/)

### 8.5 Audio Processing with MediaPipe Tasks

MediaPipe Tasks extends beyond vision to include audio inference, using `AUDIO_CLIPS` and `AUDIO_STREAM` running modes that mirror the `IMAGE` and `LIVE_STREAM` modes familiar from vision tasks. Audio tasks operate on short audio windows (typically 0.975–2 s) and can be chained with vision tasks for multi-modal applications. [Source: MediaPipe audio classifier](https://ai.google.dev/edge/mediapipe/solutions/audio/audio_classifier)

#### AudioClassifier

`AudioClassifier` classifies environmental sounds, music genres, or speech commands using models like YAMNet (521 AudioSet classes: car horn, glass break, baby cry, applause, music genre, etc.) or custom models fine-tuned with MediaPipe Model Maker (§9). YAMNet processes 0.975-second audio windows (15600 samples at 16 kHz), classifying each window into 521 AudioSet categories.

```kotlin
import com.google.mediapipe.tasks.audio.audioclassifier.AudioClassifier
import com.google.mediapipe.tasks.audio.audioclassifier.AudioClassifierOptions
import com.google.mediapipe.tasks.audio.core.RunningMode
import com.google.mediapipe.tasks.components.containers.AudioData

val options = AudioClassifierOptions.builder()
    .setBaseOptions(
        BaseOptions.builder()
            .setModelAssetPath("yamnet.tflite")
            .setDelegate(Delegate.GPU)
            .build()
    )
    .setRunningMode(RunningMode.AUDIO_STREAM)
    .setMaxResults(5)
    .setScoreThreshold(0.05f)
    .setResultListener { result, timestamp ->
        result.classificationResults().firstOrNull()
            ?.categories()?.take(3)
            ?.forEach { cat ->
                Log.d("Audio", "${cat.categoryName()}: ${cat.score()}")
            }
    }
    .build()

val classifier = AudioClassifier.createFromOptions(context, options)

// For live microphone input via AudioRecord:
val recorder = classifier.createAudioRecord(
    /* channelConfig = */ AudioFormat.CHANNEL_IN_MONO,
    /* sampleRateHz = */ 16000,
    /* bufferSizeInSeconds = */ 2.0f
)
recorder.startRecording()

// Feed audio windows in a background thread:
val bufferSize = classifier.requiredInputBufferSize()
val audioBuffer = ShortArray(bufferSize)

while (isRecording) {
    val samplesRead = recorder.read(audioBuffer, 0, bufferSize)
    if (samplesRead > 0) {
        val audioData = AudioData.create(
            AudioData.AudioDataFormat.create(
                /* numChannels = */ 1,
                /* sampleRate = */ 16000
            ),
            bufferSize
        )
        audioData.load(audioBuffer)
        classifier.classifyAsync(audioData, SystemClock.uptimeMillis())
    }
}
```

#### On-Device Performance

YAMNet at INT8 quantisation runs at under 5 ms inference latency on Pixel 7's GPU delegate, well within the 975 ms audio window it processes. This makes always-on audio event detection feasible with negligible CPU and battery impact — the GPU returns to idle between windows. Example applications include smart home sound monitors (glass break detection, smoke alarm), industrial machinery anomaly detection, and accessibility features (alert deaf users to environmental sounds).

#### Audio Buffer Management

The critical engineering challenge in audio inference is buffer management. `AudioRecord` delivers PCM at the model's required sample rate (16 kHz for YAMNet, 22050 Hz for some music models). The `classifier.requiredInputBufferSize()` method returns the exact frame count the model expects — for YAMNet, 15600 samples (0.975 s at 16 kHz). The caller must collect enough PCM samples before passing to `classifyAsync()`.

For continuous classification, the recommended pattern is a **circular ring buffer**: maintain a `FloatArray` of size `bufferSize * 2`, append new audio as it arrives from `AudioRecord`, and slide the inference window forward by `bufferSize / 2` samples (50% overlap) to avoid missing events at window boundaries. Overlapping windows at 50% doubles the classification rate — two classifications per second rather than one — and catches events that straddle a window boundary.

#### Multi-Modal Composition

MediaPipe Tasks does not yet provide an official fused audio+video task runner (as of mid-2026). For applications requiring both audio classification and video inference simultaneously (e.g., a dashboard camera detecting both dangerous sounds and visual hazards), compose the two task runners manually: run `AudioClassifier` on the audio track in a dedicated background thread and `ImageClassifier` or `ObjectDetector` on video frames in the CameraX analyser thread. Fuse results in a post-processing step keyed on timestamp — match the audio classification timestamp to the nearest video frame timestamp within a 500 ms window.

#### Latency Constraints and Model Selection

The 0.975-second window of YAMNet introduces fundamental latency for event detection: a sound event that begins at window start is not classified until the window ends, adding up to ~1 s of detection delay. For applications requiring lower latency (speech commands with <200 ms response), two options exist: (1) use a custom model trained on shorter windows (e.g., 0.25 s / 4000 samples at 16 kHz) created with MediaPipe Model Maker (§9); (2) use Google's `SpeechCommandsClassifier` model which processes 1-second windows but is optimised for onset detection and begins scoring partway through the window.

---

## 9. MediaPipe Model Maker: Custom Task Training

MediaPipe Model Maker (`mediapipe-model-maker` Python package) provides transfer learning pipelines for MediaPipe task types, enabling developers to fine-tune pre-trained TFLite backbones (MobileNetV2, EfficientDet-Lite, EfficientNet, etc.) on custom datasets without writing training infrastructure from scratch. The resulting models are exported directly to `.tflite` format and can be loaded by the Tasks API described in §8. [Source: MediaPipe Model Maker](https://ai.google.dev/edge/mediapipe/solutions/model_maker)

### 9.1 Supported Task Types

As of mid-2026, Model Maker supports fine-tuning for:

| Task | Pre-trained backbone | Dataset format |
|---|---|---|
| Image classification | MobileNetV2, EfficientNet-Lite0–4 | Directory tree: `class_name/image.jpg` |
| Object detection | EfficientDet-Lite0–4 | Pascal VOC XML or COCO JSON |
| Image segmentation | DeepLab v3 (MobileNetV3 backbone) | PNG mask per image |
| Text classification | MobileBERT, BERT-Base | CSV with text + label columns |
| Audio classification | YAMNet | WAV files in directory tree |
| Gesture recognition | MediaPipe hand pose → class | Video clips or labeled frame sequences |
| Face stylisation | U-Net style transfer | Paired image dataset |

Model Maker is appropriate when the task type matches a standard MediaPipe task, the dataset is under 100k samples, and the base model already covers the general domain (e.g., a YAMNet fine-tuned for factory sounds still benefits from pre-training on 521 AudioSet categories). For custom architectures, larger datasets, or cross-domain transfer, use the full TFLite conversion pipeline described in §6.

### 9.2 Image Classification Example

The most common Model Maker use case is adapting an image classification model to a custom set of categories:

```python
from mediapipe_model_maker import image_classifier

# Dataset loaded from directory structure:
# dataset/
#   cats/image1.jpg image2.jpg ...
#   dogs/image1.jpg image2.jpg ...
data = image_classifier.Dataset.from_folder("/path/to/dataset")
train_data, test_data = data.split(0.9)

# Configure training with MobileNetV2 backbone:
options = image_classifier.ImageClassifierOptions(
    supported_model=image_classifier.SupportedModels.MOBILENET_V2,
    hparams=image_classifier.HParams(
        epochs=10,
        batch_size=8,
        learning_rate=0.001,
        export_dir="/tmp/classifier_export"
    )
)

# Create and train the classifier:
classifier = image_classifier.ImageClassifier.create(
    train_data=train_data,
    validation_data=test_data,
    options=options,
)

# Evaluate and inspect metrics:
loss, accuracy = classifier.evaluate(test_data)
print(f"Loss: {loss:.4f}, Accuracy: {accuracy:.4f}")

# Export to .tflite (saved to options.export_dir/model.tflite):
classifier.export_model()
```

The `create()` method freezes all backbone layers, replaces the classification head with a new `Dense(num_classes)` layer, and trains only the head for the first half of epochs. In the second half, it unfreezes the top few backbone blocks and fine-tunes end-to-end with a lower learning rate. This two-phase training is automatic and generally achieves good accuracy without requiring the user to manage layer freezing manually.

### 9.3 Object Detection Example

Object detection fine-tuning uses EfficientDet-Lite as the backbone, which is specifically designed for mobile inference and produces `.tflite` models that run on the GPU delegate and Qualcomm QNN:

```python
from mediapipe_model_maker import object_detector

# Dataset in Pascal VOC format (XML annotation per image):
data = object_detector.Dataset.from_pascal_voc_folder(
    "/path/to/voc_dataset",
    data_pipeline_options=object_detector.DataPipelineOptions(
        prefetch_buffer_size=10
    )
)
train_data, test_data = data.split(0.9)

# EfficientDet-Lite1 gives a good accuracy/latency trade-off:
options = object_detector.ObjectDetectorOptions(
    supported_model=object_detector.SupportedModels.MOBILENET_MULTI_AVG,
    hparams=object_detector.HParams(
        epochs=50,
        batch_size=8,
        learning_rate=0.01
    ),
    model_options=object_detector.ModelOptions(
        l2_weight_decay=4e-5
    )
)

detector = object_detector.ObjectDetector.create(
    train_data=train_data,
    validation_data=test_data,
    options=options
)

# Export as INT8-quantised model for NPU deployment:
from mediapipe_model_maker.python.vision.object_detector import (
    object_detector as od
)
detector.export_model(
    quantization_config=od.QuantizationConfig.for_int8(
        representative_data=test_data
    )
)
```

### 9.4 Quantisation Export

Model Maker integrates quantisation into the export step. Calling `export_model(quantization_config=QuantizationConfig.for_int8(representative_data))` runs PTQ using the test dataset as the representative calibration set and exports a fully INT8-quantised `.tflite` ready for NPU deployment. For FP16 export (suitable for GPU delegate without quantisation accuracy impact):

```python
from mediapipe_model_maker.python.core.utils.quantization import QuantizationConfig
classifier.export_model(
    quantization_config=QuantizationConfig.for_float16()
)
```

INT4 weight-only export is not yet available through Model Maker's public API as of mid-2026; for INT4 export, use `ai-edge-quantizer` directly on the exported FP32 `.tflite`.

### 9.5 Google Colab Workflow and Practical Considerations

Model Maker runs on Google Colab's free tier (T4 GPU). A typical fine-tuning session for MobileNetV2 on 1000 images takes approximately 10 minutes on a T4. The recommended workflow:

1. Upload the dataset to Google Drive.
2. Mount Drive in Colab: `from google.colab import drive; drive.mount('/content/drive')`.
3. Run `pip install mediapipe-model-maker` (Colab already has TF installed).
4. Train, evaluate, export.
5. Download the `.tflite` file from Drive to local machine for APK packaging.

**Choosing Model Maker versus the full TFLite pipeline:** Model Maker constrains the architecture to pre-approved backbones and heads. This is the right choice when the task type matches exactly and dataset size is modest. When you need a custom model architecture (e.g., a two-stage detector with a custom feature pyramid, or a multi-head model that simultaneously outputs classification and depth), the full `ai-edge-torch` or `TFLiteConverter` pipeline described in §6 is necessary. Model Maker's `export_model()` output is compatible with the Tasks API without any modification — the exported `.tflite` includes the metadata (`TFLiteModelMetadataPopulator`) and label file required by `ImageClassifier.createFromFile()` or `ObjectDetector.createFromFile()`.

---

## 10. Camera2 → MediaPipe Zero-Copy Pipeline

The primary production use-case for MediaPipe on Android is live inference on camera frames. The full zero-copy path avoids any CPU-side pixel copy between the camera HAL and the inference GPU.

### 10.1 Camera2 SurfaceTexture Path

```
┌─────────────────────────────────────────────────────┐
│ Camera HAL3                                          │
│ process_capture_result() → AHardwareBuffer           │
└───────────────────────────┬─────────────────────────┘
                            │ Gralloc / DMA-BUF fd
                            ▼
┌─────────────────────────────────────────────────────┐
│ SurfaceTexture (android.graphics.SurfaceTexture)    │
│ updateTexImage() → GL_TEXTURE_EXTERNAL_OES           │
└───────────────────────────┬─────────────────────────┘
                            │ EGLImage (EGL_ANDROID_image_native_buffer)
                            ▼
┌─────────────────────────────────────────────────────┐
│ MediaPipe GpuBuffer (GpuBufferStorageEglImage)       │
│ MakePacket<GpuBuffer>()                              │
└───────────────────────────┬─────────────────────────┘
                            │ Packet on "input_video" stream
                            ▼
┌─────────────────────────────────────────────────────┐
│ ImageToTensorCalculator                              │
│ Normalise + resize → std::vector<Tensor>             │
└───────────────────────────┬─────────────────────────┘
                            │ Packets on "input_tensors" stream
                            ▼
┌─────────────────────────────────────────────────────┐
│ TfLiteInferenceCalculator (GPU delegate)             │
│ GLSL compute dispatch                                │
└───────────────────────────┬─────────────────────────┘
                            │ Packets on "output_tensors" stream
                            ▼
┌─────────────────────────────────────────────────────┐
│ LandmarksToRenderDataCalculator                      │
│ OverlayRenderer → AHardwareBuffer output frame       │
└─────────────────────────────────────────────────────┘
```

### 10.2 CameraX Integration

MediaPipe's Android SDK integrates with CameraX via `CameraXPreviewHelper` and `CameraXSourceCalculator`. The recommended integration in 2025/2026 is via CameraX's `ImageAnalysis` use-case:

```kotlin
val imageAnalyzer = ImageAnalysis.Builder()
    .setTargetResolution(Size(1280, 720))
    .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
    .setOutputImageFormat(ImageAnalysis.OUTPUT_IMAGE_FORMAT_RGBA_8888)
    .build()
    .also { analysis ->
        analysis.setAnalyzer(cameraExecutor) { imageProxy ->
            // Convert ImageProxy to MPImage and feed to the Task
            val mpImage = MediaImageBuilder(imageProxy.image!!).build()
            poseLandmarker.detectAsync(mpImage, imageProxy.imageInfo.timestamp / 1000L)
            imageProxy.close()
        }
    }
```

`ImageProxy.image` is a `Media.Image` backed by a `PRIVATE` format `ImageReader`, which on Adreno and Mali GPUs allocates a GPU-tiled `AHardwareBuffer`. `MediaImageBuilder` wraps it as an `MPImage` without copying. [Source: CameraX image analysis](https://developer.android.com/training/camerax/analyze)

### 10.3 Timestamp Synchronisation

Camera2 timestamps are in nanoseconds since boot (`Image.getTimestamp()`, type `long`). MediaPipe timestamps are in microseconds. The conversion is:

```kotlin
val mediapipeTimestampUs = imageProxy.imageInfo.timestamp / 1_000L
poseLandmarker.detectAsync(mpImage, mediapipeTimestampUs)
```

Monotonic alignment is guaranteed since both use `CLOCK_BOOTTIME`. Failure to convert correctly manifests as the scheduler rejecting out-of-order packets — a common integration bug.

### 10.4 AHardwareBuffer Direct Path (API 29+)

On devices with Android 10+ (API 29+), `Image.getHardwareBuffer()` returns the underlying `AHardwareBuffer` directly, enabling MediaPipe's `GpuBufferStorageAhwb` path:

```kotlin
val hardwareBuffer: HardwareBuffer? = imageProxy.image?.hardwareBuffer
if (hardwareBuffer != null) {
    // MediaPipe can wrap this directly without creating an EGLImage
    val mpImage = BitmapImageBuilder(
        Bitmap.wrapHardwareBuffer(hardwareBuffer, null)!!
    ).build()
}
```

On hardware that supports Vulkan memory import (`VK_ANDROID_external_memory_android_hardware_buffer`), the GPU delegate's Vulkan backend can import the buffer via `vkGetAndroidHardwareBufferPropertiesANDROID()` and dispatch compute directly — the same import mechanism described for ARCore in Ch86.

---

## 11. ARCore + MediaPipe Composition

Combining ARCore (Ch87) with MediaPipe enables spatial-aware on-device inference: ARCore provides world geometry, camera pose, and the background camera texture; MediaPipe runs ML inference on the same camera image simultaneously.

### 11.1 Shared GL Context and Texture

ARCore and MediaPipe can operate within the same EGLContext. ARCore's `ArSession` owns the camera texture handle — a `GL_TEXTURE_EXTERNAL_OES` texture populated by the Camera HAL at each `ArSession_update()` call. MediaPipe's `GlCalculatorHelper` is initialised with the existing EGL context rather than creating a new one:

```cpp
// In JNI / NDK layer after ArSession_create()
mediapipe::GlCalculatorHelper gpu_helper;
// Pass existing EGLContext from ARCore session
gpu_helper.InitializeForTest(ar_gl_context);

// Wrap ARCore's camera texture as a MediaPipe GpuBuffer
mediapipe::GpuBuffer camera_gpu_buffer =
    mediapipe::WrapGpuBufferFromExternalOES(ar_camera_texture_id,
                                            frame_width, frame_height);
```

Sharing the context eliminates the need for EGL sync objects at the ARCore/MediaPipe boundary — both AR rendering and ML inference read from the same texture object.

### 11.2 Timestamp Synchronisation

`ArFrame_getTimestamp()` returns nanoseconds since boot (`int64_t`), identical to Camera2's `Image.getTimestamp()`. Convert to MediaPipe microseconds:

```c
int64_t ar_timestamp_ns;
ArFrame_getTimestamp(ar_session, ar_frame, &ar_timestamp_ns);
mediapipe::Timestamp mp_timestamp(ar_timestamp_ns / 1000LL);
```

### 11.3 Person Segmentation Overlay on AR Scene

A practical pattern for AR applications:

1. ARCore provides the `GL_TEXTURE_EXTERNAL_OES` camera image and the `ArCamera_getViewMatrix()` / `ArCamera_getProjectionMatrix()` transforms.
2. MediaPipe `SelfieSegmentationCalculator` processes the camera texture in GLES compute, producing a single-channel `GpuBuffer` mask (confidence that each pixel is foreground person).
3. The mask is uploaded to a GLES texture and blended in the AR fragment shader:

```glsl
// Fragment shader: blend AR scene with real-world background
// based on segmentation mask
uniform sampler2D u_ArSceneTexture;
uniform sampler2D u_SegmentationMask;
uniform samplerExternalOES u_CameraTexture;

void main() {
    float mask = texture2D(u_SegmentationMask, vTexCoord).r;
    vec4 ar_scene = texture2D(u_ArSceneTexture, vTexCoord);
    vec4 camera   = texture2D(u_CameraTexture,  vTexCoord);
    gl_FragColor  = mix(camera, ar_scene, mask);
}
```

### 11.4 Pose Landmarks to AR World Space

MediaPipe 3D pose landmarks are in **normalised image space**. To project them into ARCore's world coordinate space:

1. Obtain camera intrinsics via `ArCamera_getImageIntrinsics()` (`focal_length_x/y`, `principal_point_x/y`).
2. Use the depth value from ARCore's Depth API (Ch87 §6) at the landmark's `(x, y)` pixel position to reconstruct the 3D point: `X = (x - cx) * Z / fx`, `Y = (y - cy) * Z / fy`.
3. Transform from camera space to world space using `ArCamera_getViewMatrix()` inverse.

This produces world-anchored pose landmarks that remain stable as the device moves, enabling AR overlays such as skeleton visualisations anchored to a person walking through a scene.

---

## 12. Performance Profiling and Tuning

### 12.1 LiteRT Profiler

LiteRT's built-in profiler reports per-operator execution time:

```cpp
#include "tflite/profiling/buffered_profiler.h"

auto profiler = std::make_unique<tflite::profiling::BufferedProfiler>(
    /*max_num_entries=*/1024);
interpreter->SetProfiler(profiler.get());
profiler->StartProfiling();
interpreter->Invoke();
profiler->StopProfiling();

auto profile_events = profiler->GetProfileEvents();
for (const auto* event : profile_events) {
    LOG(INFO) << "Op " << event->tag << ": "
              << event->end_timestamp_us - event->begin_timestamp_us
              << " us";
}
```
[Source: LiteRT profiling](https://ai.google.dev/edge/litert/performance/measurement)

This exposes bottleneck operators — commonly `DEPTHWISE_CONV_2D` or `FULLY_CONNECTED` — that may benefit from quantisation or from switching delegate.

### 12.2 GPU Delegate Latency Breakdown

The GPU delegate's latency decomposes into:
- **CPU→GPU transfer**: Eliminated entirely with `AHardwareBuffer` binding (§4).
- **GLSL kernel compilation** (cold start only): Eliminated by shader serialisation (`serialization_dir`).
- **Kernel dispatch**: The main runtime cost; scales with model FLOPs and GPU frequency.
- **GPU→CPU readback**: Eliminated if the output tensor remains GPU-resident (e.g., feeding into a GLES overlay shader).

For a typical 256×256 pose landmark model on an Adreno 740 (Snapdragon 8 Gen 3), the fully warm path achieves ~2 ms inference latency, comfortably within a 33 ms frame budget at 30 fps.

### 12.3 Adreno and Mali GPU Monitoring

```bash
# Adreno GPU frequency and utilisation (Qualcomm devices)
adb shell cat /sys/class/kgsl/kgsl-3d0/gpuclk
adb shell cat /sys/class/kgsl/kgsl-3d0/gpu_busy_percentage

# Mali GPU utilisation (ARM devices)
adb shell cat /sys/devices/platform/mali.0/utilisation

# General GPU info via dumpsys
adb shell dumpsys gpu
```

### 12.4 Systrace / Perfetto End-to-End Tracing

Use Perfetto to visualise the full camera→inference→render pipeline:

```bash
adb shell perfetto \
    -c - --txt \
    -o /data/misc/perfetto-traces/trace.pftrace \
<<EOF
buffers: { size_kb: 65536 }
data_sources: { config { name: "track_event" } }
data_sources: { config { name: "android.surfaceflinger.frame" } }
duration_ms: 5000
EOF
adb pull /data/misc/perfetto-traces/trace.pftrace .
```

In the resulting trace, look for `tflite::Interpreter::Invoke` slices on the inference thread and `GlCalculatorHelper::RunInGlContext` slices on the MediaPipe GPU thread. The gap between Camera2 timestamp and MediaPipe's `Process()` call reveals buffering latency.

### 12.5 Thermal Throttling

Sustained 30 fps inference at full GPU frequency triggers thermal limits within 3–5 minutes on most phones. Mitigation strategies:

- Register `PowerManager.OnThermalStatusChangedListener` and reduce inference resolution or frame rate when `THERMAL_STATUS_SEVERE` is reached.
- Switch from GPU delegate to NPU delegate (Qualcomm QNN) at elevated temperatures — NPUs are typically more power-efficient than the GPU for quantised INT8 models.
- Use `TFLITE_GPU_INFERENCE_PREFERENCE_FAST_SINGLE_ANSWER` instead of `SUSTAINED_SPEED` when thermal state is elevated; the GPU may lower frequency between frames.
- Enable adaptive frame skipping: skip inference on every other camera frame when `SystemClock.uptimeMillis()` shows the previous frame took more than 40 ms.

### 12.6 Quantisation Impact Summary

| Configuration | Typical Latency | Memory | Notes |
|---|---|---|---|
| Float32, CPU | 100–200 ms | 4× base | Baseline, no delegate |
| Float32, GPU delegate | 5–15 ms | 4× base | GLSL compute shaders |
| INT8, GPU delegate | 3–10 ms | 1× base | ~1.5–2× speedup |
| INT8, NNAPI/NPU (pre-API 35) | 2–8 ms | 1× base | 2–4× speedup vs GPU |
| INT8, Qualcomm QNN delegate | 1–5 ms | 1× base | Hexagon DSP, lowest power |

### 12.7 Benchmarking with benchmark_model

`benchmark_model` is the standard LiteRT performance measurement tool, providing reproducible, hardware-isolated latency measurements that are not affected by application-level overhead (JNI marshalling, Kotlin object allocation, UI thread jank). It is built from `tflite/tools/benchmark/` in the LiteRT repository and is also available as a prebuilt binary in the Android NDK tools package. [Source: LiteRT performance measurement](https://ai.google.dev/edge/litert/performance/measurement)

#### Deploying and Running

The binary runs as a shell command directly on the device, bypassing the Android framework entirely:

```bash
# Push the binary to a writable location
adb push benchmark_model /data/local/tmp/
adb shell chmod +x /data/local/tmp/benchmark_model

# Push the model
adb push model.tflite /data/local/tmp/

# Run with GPU delegate, 50 inference runs, 10 warmup runs:
adb shell /data/local/tmp/benchmark_model \
    --graph=/data/local/tmp/model.tflite \
    --num_threads=4 \
    --num_runs=50 \
    --warmup_runs=10 \
    --use_gpu=true \
    --gpu_precision_loss_allowed=true \
    --enable_op_profiling=true \
    --report_peak_memory_footprint=true
```

Key flags:
- `--num_threads`: CPU thread count, passed to XNNPack; ignored when `--use_gpu=true`
- `--warmup_runs`: Number of inferences to run before timing begins; critical for GPU delegate, where the first 1–3 runs may include JIT compilation even with shader serialisation
- `--use_xnnpack=true`: Force XNNPack CPU delegate (normally auto-applied)
- `--use_nnapi=false`: Disable NNAPI delegate
- `--external_delegate_path=/data/local/tmp/libQnnTFLiteDelegate.so`: Load a vendor delegate `.so` directly, useful for testing QNN or NeuroPilot without a full app

#### Interpreting the Output

A typical output for a GPU-delegate run looks like:

```
Inference timings in us: Init: 8234, First inference: 43210,
Warmup (avg): 12340 us, Inference (avg): 11890 us
Peak memory footprint: 47.2 MB
Note: as the benchmark was running in a separate thread, the timings
above may be slightly higher than if the benchmark was run in the main thread.
```

The four timing fields have distinct meanings:

- **Init**: Model loading time including memory mapping, FlatBuffer parsing, and delegate setup. For the GPU delegate, this includes EGL context creation and delegate registration. This is a fixed cost paid once at app startup.
- **First inference**: The first `Invoke()` call after `AllocateTensors()`. Even with shader serialisation, the first inference may load compiled kernels from disk and bind GPU buffers, so this is typically 2–5× the steady-state latency.
- **Warmup avg**: Average over `--warmup_runs` invocations. After the first inference, the GPU command queue is pre-populated and shader bindings are cached, so warmup latency represents the cache-warm path.
- **Inference avg**: Average over `--num_runs` invocations after warmup. This is the number to use for frame-budget planning. For a 30 fps pipeline, the inference avg must be below 33 ms; for 60 fps, below 16 ms.

#### Per-Op Profiling

With `--enable_op_profiling=true`, `benchmark_model` prints a per-operator breakdown after the main benchmark loop:

```
Op[0]: CONV_2D                  2340 us (19.7%)
Op[1]: DEPTHWISE_CONV_2D         890 us  (7.5%)
Op[2]: ADD                        45 us  (0.4%)
...
Op[34]: FULLY_CONNECTED          3120 us (26.2%)
DELEGATE[GPU]:          9870 us total for 35 ops
CPU fallback ops:        320 us for 2 ops (CUSTOM_OP_X, RESHAPE)
```

This breakdown identifies which operators are consuming the most inference time and, critically, which operators fell back to CPU (the "CPU fallback ops" line). CPU fallback within a GPU-delegated graph introduces synchronisation overhead: the runtime must finish the GPU compute, copy results to CPU, run the CPU op, copy back to GPU, and resume — typically adding 2–5 ms per CPU fallback boundary. When `benchmark_model` shows CPU fallback ops, the fix is either to implement those ops in the GPU delegate (if they are custom ops) or to restructure the model to avoid the unsupported operation.

#### Latency Percentiles and Regression Detection

For real-time applications, average latency is insufficient. A model with 10 ms average inference but 80 ms p99 will cause visible stutter at 30 fps. Use `--print_preinvoke_state=true` combined with CSV output (`--profiling_output_csv_file=/data/local/tmp/profile.csv`) to capture per-run latency, then compute percentiles offline:

```bash
adb pull /data/local/tmp/profile.csv .
python3 -c "
import csv, numpy as np
with open('profile.csv') as f:
    latencies = [float(row['inference_time_us']) for row in csv.DictReader(f)]
print(f'p50={np.percentile(latencies,50):.0f}us '
      f'p90={np.percentile(latencies,90):.0f}us '
      f'p99={np.percentile(latencies,99):.0f}us')
"
```

For CI regression detection, wrap `benchmark_model` in a shell script that parses the `Inference (avg)` field from stdout and compares it against a stored baseline. A latency increase of more than 10% on the same reference device is a signal to investigate — common causes include a kernel update changing GPU governor behaviour, a new TF op that falls back to CPU, or a model change that added a non-delegatable op.

---

## 13. Model Security: Encryption, Signing, and Integrity Protection

On-device ML models represent significant intellectual property. A `.tflite` model bundled in the APK's `assets/` directory is trivially extractable: `apktool d app.apk` unpacks the APK and exposes the model file with no obfuscation. An attacker with the extracted `.tflite` can inspect the model architecture using `flatc --json schema.fbs model.tflite`, infer training data characteristics, or fine-tune the model on their own dataset. For proprietary models with commercial value — medical diagnosis, premium recommendation engines, or specialised vision capabilities — encryption and integrity protection are essential. [Source: Android KeyStore](https://developer.android.com/privacy-and-security/keystore) [Source: Google Play Integrity](https://developer.android.com/google/play/integrity)

### 13.1 AES-256-GCM Encryption

The recommended approach is AES-256-GCM authenticated encryption. The model is encrypted offline before APK packaging; the decryption key is stored in the Android KeyStore (never in the APK); and the model is decrypted in memory at runtime, never written to disk in plaintext.

**Offline encryption** (CI/CD step, not in the APK):

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

# 256-bit key — generated once, stored securely in your build system's
# secret manager (GitHub Actions Secrets, Google Cloud Secret Manager, etc.)
# NOT hardcoded in source and NOT bundled in the APK.
key = os.urandom(32)
aesgcm = AESGCM(key)

# 96-bit nonce — unique per encryption, safe to store with ciphertext
nonce = os.urandom(12)

with open("model.tflite", "rb") as f:
    plaintext = f.read()

# GCM produces ciphertext + 128-bit authentication tag (appended automatically)
ciphertext = aesgcm.encrypt(nonce, plaintext, associated_data=None)

# Store nonce prepended to ciphertext for runtime decryption
with open("model.enc", "wb") as f:
    f.write(nonce + ciphertext)
```

**Runtime decryption** using a key provisioned into the Android KeyStore:

```kotlin
import android.security.keystore.KeyGenParameterSpec
import android.security.keystore.KeyProperties
import javax.crypto.Cipher
import javax.crypto.KeyGenerator
import javax.crypto.SecretKey
import javax.crypto.spec.GCMParameterSpec

// --- Key provisioning (done once at app first-run or from server) ---
// This example shows generating a key in the KeyStore; in production,
// the key would be fetched from a server and imported via ECDH key agreement.
fun generateKeyInKeyStore(alias: String) {
    val keyGenerator = KeyGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore"
    )
    keyGenerator.init(
        KeyGenParameterSpec.Builder(alias,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setKeySize(256)
            // Key is not exportable from the KeyStore:
            .setUnlockedDeviceRequired(true)
            .build()
    )
    keyGenerator.generateKey()
}

// --- Runtime decryption ---
fun decryptModel(context: Context, encryptedAssetName: String): ByteArray {
    val keyStore = java.security.KeyStore.getInstance("AndroidKeyStore")
    keyStore.load(null)
    val secretKey = keyStore.getKey("model_key", null) as SecretKey

    val encBytes = context.assets.open(encryptedAssetName).readBytes()
    val nonce = encBytes.copyOfRange(0, 12)
    val ciphertext = encBytes.copyOfRange(12, encBytes.size)

    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.DECRYPT_MODE, secretKey, GCMParameterSpec(128, nonce))
    // GCM authentication tag is verified automatically; throws AEADBadTagException
    // if the ciphertext has been tampered with.
    return cipher.doFinal(ciphertext)
}

// Load the decrypted model into LiteRT:
val modelBytes = decryptModel(context, "model.enc")
val interpreter = org.tensorflow.lite.Interpreter(
    java.nio.ByteBuffer.wrap(modelBytes)
)
```

### 13.2 Android KeyStore and Hardware Security

Keys created in the Android KeyStore with the `AndroidKeyStore` provider are backed by hardware security: on Pixel 6+ and other devices with Arm TrustZone or a discrete Strongbox secure element, the AES key material is stored inside tamper-resistant hardware and **never leaves that hardware in plaintext**. Even with root access to the Android OS, the key cannot be extracted. The `Cipher` object is handed to the hardware for decryption, and the decrypted bytes are returned directly to the calling process.

Key protection options worth configuring in production:

- `setUnlockedDeviceRequired(true)`: Key is only accessible when the device is unlocked; prevents offline attack with a stolen device.
- `setUserAuthenticationRequired(true)` with `setUserAuthenticationValidityDurationSeconds(30)`: Requires biometric or PIN authentication before the key is accessible; appropriate for models containing sensitive healthcare or financial data.
- `setIsStrongBoxBacked(true)`: Force use of the Strongbox secure element (API 28+, Pixel 3+); slower than TrustZone but provides the highest resistance to side-channel attacks.

### 13.3 HMAC Signing for Integrity Without Encryption

In cases where the model is not proprietary but tamper-resistance is still important (e.g., a safety-critical model where a modified model could produce unsafe outputs), HMAC-SHA256 signing is a lightweight alternative to full encryption:

```python
import hmac, hashlib

signing_key = b"..."  # 32-byte key from your signing system
with open("model.tflite", "rb") as f:
    model_bytes = f.read()

# Compute HMAC-SHA256 over the entire model file
signature = hmac.new(signing_key, model_bytes, hashlib.sha256).digest()

# Store signature alongside the model (e.g., model.tflite.sig)
with open("model.tflite.sig", "wb") as f:
    f.write(signature)
```

On device, verify before loading:

```kotlin
fun verifyModelSignature(modelBytes: ByteArray, signatureBytes: ByteArray,
                          signingKey: ByteArray): Boolean {
    val mac = javax.crypto.Mac.getInstance("HmacSHA256")
    mac.init(javax.crypto.spec.SecretKeySpec(signingKey, "HmacSHA256"))
    val computed = mac.doFinal(modelBytes)
    return computed.contentEquals(signatureBytes)
}

if (!verifyModelSignature(modelBytes, sigBytes, key)) {
    throw SecurityException("Model integrity check failed — file may be tampered")
}
```

The signing key itself must come from the Android KeyStore or from a remote attestation service — storing it in the APK defeats the purpose.

### 13.4 Google Play Asset Delivery for Key Management

An alternative to managing your own key provisioning is **Google Play Secure Delivery** for on-demand or install-time asset packs. The Play Store manages APK signing; for models, you can deliver an encrypted model via an `installTime` asset pack where the decryption key is bound to the user's Play license. This means the key cannot be obtained without a valid Play Store license — preventing model extraction via a repackaged APK that strips license checks. The downside is tight coupling to Play infrastructure and the added complexity of asset pack delivery.

### 13.5 Play Integrity API for Runtime Attestation

Even with model encryption and KeyStore-backed decryption, a sophisticated attacker can repackage the APK (adding instrumentation code) and sign it with their own certificate, intercepting the decrypted model bytes from process memory after `cipher.doFinal()`. The **Play Integrity API** addresses this by verifying that the running app is genuine and unmodified:

```kotlin
val integrityManager = IntegrityManagerFactory.create(context)
val request = IntegrityTokenRequest.newBuilder()
    .setNonce(generateNonce())  // server-issued nonce to prevent replay
    .build()

integrityManager.requestIntegrityToken(request)
    .addOnSuccessListener { response ->
        val token = response.token()
        // Send token to your server; server decrypts and verifies:
        // - MEETS_DEVICE_INTEGRITY
        // - MEETS_BASIC_INTEGRITY
        // - Package name matches expected
        // Only after server confirms integrity, send the decryption key
        fetchDecryptionKeyFromServer(token)
    }
```

This architecture ensures that the decryption key is only issued to a genuine, unmodified install of your app. An attacker with a repackaged APK receives a token that fails server-side integrity verification and never gets the key. [Source: Play Integrity API](https://developer.android.com/google/play/integrity)

---

## 14. Privacy by Design: Data Protection for On-Device ML

The fundamental privacy advantage of LiteRT and MediaPipe is that inference runs locally — camera frames, audio, health sensor data, and text never leave the device. This is a concrete GDPR and CCPA compliance advantage, but it requires careful design to realise: the on-device inference guarantee can be undermined by incidental data collection elsewhere in the application, by telemetry SDKs, or by logging of inference outputs. [Source: Android privacy best practices](https://developer.android.com/privacy/best-practices)

### 14.1 What Stays On-Device

When using LiteRT directly (no cloud services integrated), the following never leave the device:
- Input tensors (camera frames, audio buffers, text tokens)
- Intermediate activation tensors during inference
- Model weight tensors
- Output tensors (landmarks, classification scores, detection boxes)
- The `.tflite` model file itself (once loaded into memory)

The Android OS does not receive inference inputs or outputs from the LiteRT runtime. There are no telemetry hooks in the LiteRT C++ runtime itself — it is a pure in-process library. Google Play Services *does* receive crash reports from apps, but stack traces do not include tensor contents.

### 14.2 What May Leave the Device

Developers must audit their application carefully for data paths that could inadvertently exfiltrate inference inputs or outputs:

- **Firebase Performance Monitoring / Crashlytics**: If the app logs inference latency, model file path, or input tensor shapes as custom attributes in Firebase events, those attributes are uploaded to Google servers. Log only latency numbers, not tensor contents.
- **Cloud Anchor queries (ARCore)**: If ARCore Visual Positioning System (VPS) is used alongside MediaPipe, anonymised feature point descriptors extracted from camera frames are uploaded to Google for localisation (as documented in Ch87 §5). This is independent of MediaPipe inference.
- **Crash reports containing model path in stack traces**: If `Interpreter::Invoke()` crashes, the stack trace may include the model file path as a string argument. Ensure the model file path does not include user-identifying information.
- **Any explicit app-level logging**: Inference outputs themselves — landmark coordinates, detected object labels, classification scores — are the developer's responsibility. If the app logs or uploads these, that is the app's data processing decision, not LiteRT's.

### 14.3 GDPR Article 9 and Biometric Special Category Data

Face landmarks (from `FaceLandmarker`), iris coordinates, hand geometry (from `HandLandmarker`), and voice embeddings are **special category biometric data** under GDPR Article 9. Processing this data requires explicit prior consent from the data subject (Article 9(2)(a)), or that processing is necessary for the data subject's own vital interests (Article 9(2)(c)) — a narrow exception. Apps using any of these MediaPipe tasks that operate on EU users must:

1. Present a clear, specific consent dialog before activating the feature — generic "we use your camera" consent in the privacy policy is insufficient for GDPR Article 9.
2. Log the consent event with a timestamp and the specific biometric data type consented to.
3. Provide a mechanism to withdraw consent and delete associated data (e.g., derived embeddings stored in a local database).
4. Document the processing in the Records of Processing Activities (RoPA) required under GDPR Article 30.

Even though the raw pixel data never leaves the device, the *derived biometric features* (landmark coordinates, embeddings) may be stored locally or transmitted if the application design requires it, and these are subject to GDPR Article 9 regardless of where they are stored.

### 14.4 Data Minimisation at Inference Time

The camera pipeline described in §10 handles `ImageProxy` objects backed by hardware-accelerated `AHardwareBuffer` allocations. The privacy-correct pattern is to close the image immediately after passing to MediaPipe, preventing the buffer from accumulating in a queue where it could be accessed by other application components:

```kotlin
analysis.setAnalyzer(cameraExecutor) { imageProxy ->
    val image = imageProxy.image ?: run {
        imageProxy.close()  // close before returning on error
        return@setAnalyzer
    }
    val mpImage = BitmapImageBuilder(
        image.toBitmap()  // or MediaImageBuilder(image)
    ).build()

    // Pass to MediaPipe and close immediately — do NOT store mpImage
    poseLandmarker.detectAsync(mpImage, imageProxy.imageInfo.timestamp / 1_000L)

    imageProxy.close()  // close immediately; don't buffer frames
}
```

Do not maintain a ring buffer of recent `ImageProxy` frames unless the application requires it for temporal smoothing. If temporal smoothing is needed, buffer only the already-processed inference outputs (landmark coordinates, confidence scores), not the raw camera frames.

### 14.5 Differential Privacy for On-Device Training

If the application fine-tunes a model on device (§15), differential privacy (DP) prevents the trained model update from revealing individual training samples. The TensorFlow Privacy library provides `DPKerasAdamOptimizer`, which adds calibrated Gaussian noise to gradients before they are applied:

```python
from tensorflow_privacy import DPKerasAdamOptimizer
from tensorflow_privacy.privacy.analysis import compute_dp_sgd_privacy

optimizer = DPKerasAdamOptimizer(
    l2_norm_clip=1.0,           # gradient clipping bound
    noise_multiplier=1.1,       # noise scale (higher = more privacy)
    num_microbatches=None,      # vectorised per-example gradients
    learning_rate=0.001
)
model.compile(optimizer=optimizer, loss='sparse_categorical_crossentropy',
              metrics=['accuracy'])
```

The privacy budget (ε, δ) can be computed after training to quantify the privacy guarantee:

```python
eps, delta = compute_dp_sgd_privacy.compute_dp_sgd_privacy(
    n=len(train_dataset),
    batch_size=32,
    noise_multiplier=1.1,
    epochs=10,
    delta=1e-5
)
print(f"DP guarantee: ε={eps:.2f}, δ={delta}")
# Typical output: ε=2.34, δ=1e-5
```

DP training is relevant when inference results are used to fine-tune local models that are then aggregated (Federated Learning, §15). For pure inference without local training, DP is not applicable.

### 14.6 Privacy Policy Disclosure Checklist

The following data processing activities require explicit disclosure in the app's privacy policy and, where applicable, an in-app consent flow:

- **ARCore + VPS**: Disclose that anonymised camera feature points are uploaded to Google for visual localisation (required by ARCore Terms of Service §4).
- **FaceLandmarker / iris tracking**: Disclose biometric data processing, purpose (e.g., "face filter rendering"), retention period (e.g., "not stored"), and GDPR Article 9 legal basis.
- **HandLandmarker**: Disclose hand geometry processing if the derived data is stored or transmitted.
- **AudioClassifier**: If audio windows are retained beyond the single inference call (e.g., for debug logging), disclose; 16 kHz PCM retained even briefly constitutes audio recording under many privacy frameworks.
- **On-device LLM inference**: If the user's text inputs to the LLM are used to fine-tune the local model (via on-device training, §15), disclose this to the user — even though the data stays on device, the user's input is affecting model behaviour, which many users consider a privacy-relevant action.

---

## 15. On-Device Training and Federated Learning

On-device inference — passing a camera frame through a fixed model — is the dominant pattern in LiteRT deployments today. But a growing class of applications requires on-device *training*: personalising a model to a specific user's behaviour, adapting to domain shift (a factory camera's lighting is different from the training dataset), or fine-tuning a base model on locally collected labelled data. This section covers LiteRT's experimental on-device training API and the Federated Learning infrastructure that allows privacy-preserving model updates across millions of devices. [Source: On-Device Personalization](https://developer.android.com/on-device-personalization) [Source: TensorFlow Federated](https://www.tensorflow.org/federated)

### 15.1 On-Device Training Use Cases

The cases where on-device training makes sense are distinct from those where it does not:

**Good fit for on-device training:**
- Typed word completion or next-word prediction personalised to a user's vocabulary and style
- Wake-word detection adapted to the user's specific voice and accent
- Health anomaly detection tuned to an individual's physiological baseline (heart rate variability, gait pattern)
- Gesture recognition customised to a user's specific hand size and motion patterns

**Poor fit for on-device training:**
- General-purpose model improvement (use server-side training with federated updates)
- Cases where >10k labelled examples are needed (data collection on-device is too slow)
- Models requiring architecture changes (on-device training only fine-tunes weights, not architecture)

### 15.2 LiteRT On-Device Training API

LiteRT's on-device training API is experimental as of 2026. It requires a model converted with training mode enabled, which preserves the gradient computation subgraph alongside the inference subgraph in the `.tflite` FlatBuffer:

```python
# Export in training mode (TF/Keras)
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.experimental_enable_resource_variables = True
# Note: training-mode export requires the experimental TrainingSession API;
# verify the exact converter flags against the current LiteRT release.
tflite_train_model = converter.convert()
```

On-device, `TfLiteTrainingSession` runs forward and backward passes:

```cpp
// Note: LiteRT on-device training API is experimental;
// API names and header locations may change between releases.
// Verify against the installed tflite/experimental/learning/ headers.
#include "tflite/experimental/learning/training_session.h"

TfLiteTrainingSessionOptions opts;
opts.max_gradient_norm = 1.0f;   // gradient clipping
TfLiteTrainingSession* session =
    TfLiteTrainingSessionCreate(interpreter, &opts);

// Run one training step (forward + backward pass + weight update):
TfLiteStatus status = TfLiteTrainingSessionRun(
    session,
    /*inputs=*/input_tensors,
    /*labels=*/label_tensors,
    /*outputs=*/output_tensors
);
// Updated weights are written back into the interpreter's weight tensors.
TfLiteTrainingSessionDelete(session);
```

The on-device training API is deliberately limited to fine-tuning the last few layers (classification head, adapter layers) rather than the full model. Training the full backbone on-device requires too much memory for the gradient tensors — a MobileNetV2 backbone produces ~14 MB of gradient tensors even at INT8, exceeding available RAM on mid-range devices when combined with the inference activations.

### 15.3 Federated Learning Architecture

**Federated Learning (FL)** is the privacy-preserving approach to aggregating on-device training across many users. No raw data leaves any device. Each device trains on its local data, computes a gradient update (the change to the model weights), and uploads only the update — not the training data — to a central server. The server aggregates updates from many devices using **FedAvg** (Federated Averaging) and sends back an updated global model.

```
Device A: local training → gradient update ΔW_A
Device B: local training → gradient update ΔW_B
Device C: local training → gradient update ΔW_C
        ...
Server: W_global += α × average(ΔW_A, ΔW_B, ΔW_C, ...)
Server → sends updated W_global back to all devices
```

The privacy guarantee is that the server sees only gradient updates, not raw training data. This is substantially stronger than "data stays on device" — the gradient update itself must be further protected with differential privacy (§14.5) because gradients can leak information about individual training examples through gradient inversion attacks.

### 15.4 TensorFlow Federated

**TensorFlow Federated (TFF)** is Google's Python framework for FL simulation and production deployment. The core building block is `tff.learning.algorithms.build_weighted_fed_avg()`, which produces a `tff.templates.IterativeProcess` implementing FedAvg:

```python
import tensorflow_federated as tff

# Define the model via a model_fn:
def model_fn():
    keras_model = create_keras_model()
    return tff.learning.models.from_keras_model(
        keras_model,
        input_spec=train_data[0].element_spec,
        loss=tf.keras.losses.SparseCategoricalCrossentropy(),
        metrics=[tf.keras.metrics.SparseCategoricalAccuracy()]
    )

# Build the FedAvg algorithm:
training_process = tff.learning.algorithms.build_weighted_fed_avg(
    model_fn=model_fn,
    client_optimizer_fn=tff.learning.optimizers.build_sgdm(learning_rate=0.1),
    server_optimizer_fn=tff.learning.optimizers.build_adam(learning_rate=0.001)
)

# Simulate a training round with 10 participating devices:
state = training_process.initialize()
for round_num in range(100):
    sampled_clients = np.random.choice(len(train_data), size=10, replace=False)
    client_datasets = [train_data[i] for i in sampled_clients]
    result = training_process.next(state, client_datasets)
    state = result.state
    print(f"Round {round_num}, loss: {result.metrics['client_work']['train']['loss']:.4f}")
```

TFF runs in simulation mode on a desktop/server, allowing you to prototype and evaluate the FL algorithm before deploying. The same algorithm is used for production deployment via the Android **On-Device Personalization (ODP)** service.

### 15.5 Android On-Device Personalization Service

Android 14+ (API 34+) introduced the **On-Device Personalization (ODP)** framework, which provides a secure, sandboxed execution environment for FL training tasks. It is implemented as a system service (`OnDevicePersonalizationService`) with strict isolation guarantees:

- FL training tasks run in an `IsolatedProcess`, a sandboxed process that cannot make arbitrary network requests
- Raw training data (keyboard history, app usage, health data) never leaves the `IsolatedProcess`
- Gradient updates are processed by the **Federated Compute service** before any aggregation; differential privacy noise is added automatically
- The aggregated model update is only uploaded when the device is idle, charging, and on Wi-Fi
- The ODP service enforces a privacy budget across all FL tasks on the device; if a task exceeds its allotted (ε, δ) budget, future updates are suppressed

The typical FL round on Android proceeds as follows:

1. The ODP service receives a task from the FL server (model file + training hyperparameters + privacy budget allocation).
2. The `IsolatedProcess` is launched and the global model is fetched.
3. Local SGD runs on on-device data using the isolated environment's access to keyboard history, app usage patterns, and other permitted data sources.
4. The ODP service applies DP noise to the gradient update (Gaussian mechanism, calibrated to the allocated ε, δ budget).
5. The privacy-noised update is compressed (top-k gradient sparsification retaining only the k largest-magnitude gradients) and uploaded to the FL server.
6. The FL server runs FedAvg across updates from N devices and produces a new global model.

### 15.6 Communication Efficiency

The gradient update itself can be large — for a 5 million parameter model (MobileNetV2), a FP32 gradient update is 20 MB. Standard FL communication efficiency techniques reduce this:

- **Top-k sparsification**: Upload only the k% of gradient components with largest absolute value. At k=0.1% (top-0.1% of gradients), the upload is 2000× smaller with typically <1% accuracy impact on the final model. Sparsification is applied by the ODP service automatically.
- **Quantised gradients**: Compress gradient values to INT8 before upload; 4× bandwidth reduction with negligible accuracy impact.
- **Federated dropout**: Randomly drop a fraction of neurons during local training; the resulting gradient is inherently sparse, reducing update size without separate sparsification.
- **Encoded aggregation**: TFF's `tff.aggregators.EncodedSumFactory` applies the same compression on the simulation side, allowing you to evaluate the accuracy/bandwidth trade-off before production deployment.

---

## 16. Gemini Nano: On-Device Generative AI

Gemini Nano is Google's on-device large language model — the smallest member of the Gemini family, optimised for inference on Android smartphones without a network connection. Unlike LiteRT models that apps load directly, Gemini Nano is delivered and managed by the Android operating system through the **AICore** system service, and accessed by applications through the **ML Kit GenAI API**. This section covers the full stack: AICore architecture, the developer API, hardware requirements, capability boundaries, and the strategic trade-offs between Gemini Nano and deploying open-weight models via LiteRT-LM. [Source: Google AI Edge — Gemini Nano](https://ai.google.dev/gemini-api/docs/android)

### 16.1 AICore System Architecture

AICore (`android.ai.app.aicore`) is an Android system service introduced on Pixel 8 series devices and expanded to Pixel 9+ and selected Samsung Galaxy S25 devices. It runs as a persistent system daemon with higher permissions than a standard app, giving it exclusive access to the SoC's AI-dedicated hardware (Google Tensor G4 NPU, Samsung Exynos NPU) that is deliberately not exposed to the general NNAPI or LiteRT delegate surfaces.

```
Application
    │ ML Kit GenAI API (Java/Kotlin, API 34+)
    ▼
com.google.android.gms.ai (Google Play Services module)
    │ Binder IPC
    ▼
AICore system service (persistent daemon, UID=system)
    │ proprietary HAL
    ▼
Gemini Nano model weights (encrypted, /data/data/com.google.android.aicore/)
    │ vendor runtime
    ▼
Tensor G4 / Exynos NPU  ←→  GPU (fallback compute)
```

The key architectural distinction from LiteRT: **Gemini Nano weights are never accessible to apps.** The model is provisioned by Google as a system update (via `com.google.android.aicore` APK), verified with hardware attestation, and decrypted only inside the AICore daemon's isolated process. Applications never receive the raw FlatBuffer or weight tensors — they interact exclusively through the ML Kit API, receiving only text token outputs.

Gemini Nano is updated over-the-air independently of Android OS updates, via the `android.ai.app.aicore` Google Play Services module. This allows Google to update the model quarterly without requiring a full Android system update. [Source: AICore release notes](https://developer.android.com/ml/on-device/aicore)

### 16.2 ML Kit GenAI API

The ML Kit GenAI API (`com.google.mlkit:genai-common` + `com.google.mlkit:genai-tasks-generate-text`) is the developer-facing interface to Gemini Nano. Available on API 34+, it wraps the AICore Binder IPC in a familiar Tasks-style async API.

**Gradle dependencies:**
```kotlin
// app/build.gradle.kts
dependencies {
    implementation("com.google.mlkit:genai-common:0.0.1-beta1")
    implementation("com.google.mlkit:genai-tasks-generate-text:0.0.1-beta1")
}
```

**Availability check (required before any inference):**

Not all devices with Pixel 9 hardware actually have the Gemini Nano model downloaded — the model is a ~1.8 GB download that may not be present on first boot. Apps must check availability and trigger a download if needed:

```kotlin
import com.google.mlkit.genai.common.DownloadCallback
import com.google.mlkit.genai.common.GenAiException
import com.google.mlkit.genai.text.generation.TextGeneration
import com.google.mlkit.genai.text.generation.TextGenerationParams

val textGenerator = TextGeneration.getClient(
    TextGenerationParams.builder()
        .setTemperature(0.7f)
        .setTopK(40)
        .setMaxOutputTokens(256)
        .build()
)

// Check if Gemini Nano is available on this device:
textGenerator.checkFeatureStatus()
    .addOnSuccessListener { status ->
        when (status) {
            FeatureStatus.DOWNLOADABLE -> {
                // Model available but not yet downloaded; trigger download:
                textGenerator.downloadFeature(object : DownloadCallback {
                    override fun onDownloadStarted(bytesToDownload: Long) { }
                    override fun onDownloadProgress(totalBytesDownloaded: Long) { }
                    override fun onDownloadCompleted() { /* ready to use */ }
                    override fun onDownloadFailed(e: GenAiException) { /* handle */ }
                })
            }
            FeatureStatus.DOWNLOADING -> { /* show progress UI */ }
            FeatureStatus.AVAILABLE   -> { /* ready now */ }
            FeatureStatus.NOT_SUPPORTED -> { /* fall back to cloud or LiteRT-LM */ }
        }
    }
```

**Text generation (non-streaming):**

```kotlin
val request = TextGenerationRequest.builder()
    .addSystemInstructions("You are a concise assistant.")
    .addContent(contentFromText("user", "Summarise this review in one sentence: $reviewText"))
    .build()

textGenerator.generateText(request)
    .addOnSuccessListener { result ->
        val outputText = result.text   // full generated string
        Log.d("Gemini", outputText)
    }
    .addOnFailureListener { e ->
        Log.e("Gemini", "Error: ${e.message}")
    }
```

Multi-turn conversation is supported by accumulating `Content` objects with alternating `"user"` and `"model"` roles, then passing the full list to `generateText`. The AICore daemon maintains no session state between calls — the app is responsible for conversation history management.

### 16.3 Streaming Inference and Token Budget

For interactive use cases (chat, real-time completion), Gemini Nano supports streaming: tokens are delivered to the app as they are generated, rather than waiting for the full response. This reduces perceived latency from ~8–15 seconds (full response for 256 tokens) to first-token latency of ~400–800 ms on Pixel 9.

```kotlin
import com.google.mlkit.genai.text.generation.StreamingTextGenerationCallback

val streamingRequest = TextGenerationRequest.builder()
    .addContent(contentFromText("user", userMessage))
    .build()

textGenerator.generateTextStream(streamingRequest,
    object : StreamingTextGenerationCallback {
        override fun onPartialResult(partialResult: TextGenerationResult) {
            // Called on main thread for each new token chunk:
            appendToUI(partialResult.text)
        }
        override fun onComplete(finalResult: TextGenerationResult) {
            val totalTokensUsed = finalResult.totalTokenCount
            Log.d("Gemini", "Done. Tokens: $totalTokensUsed")
        }
        override fun onFailure(e: GenAiException) {
            Log.e("Gemini", "Stream failed: ${e.message}")
        }
    }
)
```

**Token counting (pre-flight check):**

Before sending a long prompt, count tokens to avoid exceeding the context window:

```kotlin
textGenerator.countTokens(request)
    .addOnSuccessListener { countResult ->
        Log.d("Gemini", "Prompt tokens: ${countResult.totalTokenCount}")
        // Gemini Nano context window: ~8192 tokens (input + output combined)
    }
```

The context window is approximately 8192 tokens on Pixel 9 hardware (subject to change with model updates). Input tokens + output tokens must not exceed this limit; the AICore daemon returns `GenAiException` with `ERROR_CONTEXT_LIMIT_EXCEEDED` if the limit is breached.

### 16.4 Hardware Requirements and Memory Footprint

Gemini Nano imposes strict hardware requirements due to the model's size and the need for a hardware-backed secure enclave to protect the weights:

| Device | SoC | Gemini Nano variant | RAM min | Context window |
|---|---|---|---|---|
| Pixel 8 / 8 Pro | Tensor G3 | Nano 1 (original, smaller) | 8 GB | ~4096 tokens |
| Pixel 9 / 9 Pro / 9 Pro XL / 9 Pro Fold | Tensor G4 | Nano 2 (improved) | 8 GB | ~8192 tokens |
| Samsung Galaxy S25 / S25+ / S25 Ultra | Exynos 2500 / Snapdragon 8 Elite | via Google AICore | 8 GB | ~8192 tokens |
| Other Android phones | — | Not supported | — | — |

**Memory footprint**: Gemini Nano 2 weights occupy ~1.8 GB on-disk (INT4 quantised). At inference time, the AICore daemon loads weights into a combination of NPU SRAM and system DRAM. The occupied DRAM appears in `/proc/meminfo` as `KernelMalloc` or mapped to the AICore process — it is not reclaimed between AICore calls, but may be evicted under severe memory pressure (LMK). After eviction, the next call incurs a cold-load penalty of ~3–8 seconds.

**Battery impact**: Gemini Nano inference at 256 output tokens consumes approximately 150–300 mJ on Pixel 9, depending on whether the Tensor G4 NPU (more efficient) or the Arm Mali GPU (fallback) handles the compute. For sustained use in a chat application, thermal throttling becomes relevant after ~5–10 minutes of continuous generation; combine with `AThermal_getThermalHeadroom()` (§17 of ch161) to detect imminent throttling and pause generation.

### 16.5 Capability Limits and Task Fit

Gemini Nano 2 as exposed through the ML Kit GenAI API (mid-2026) has the following limitations compared to full Gemini cloud models:

| Capability | Gemini Nano 2 (on-device) | Gemini Pro / Flash (cloud) |
|---|---|---|
| Text input | Yes | Yes |
| Vision input (images) | Not in public API | Yes |
| Audio input | Not in public API | Yes |
| Function calling | Not in public API | Yes |
| System instructions | Yes | Yes |
| Multi-turn conversation | Yes (app-managed history) | Yes |
| Context window | ~8192 tokens | 1M tokens |
| Output token limit | 256 (configurable, max ~1024) | 8192+ |
| Latency (first token) | 400–800 ms | 200–500 ms (network) |
| Throughput | ~15–25 tokens/s | ~100+ tokens/s |
| Offline operation | Yes | No |
| Privacy (data stays on device) | Yes | No |

**Best-fit tasks for Gemini Nano**:
- **Smart reply generation**: suggest 3 short replies to an incoming message; low token count, latency-tolerant
- **On-device summarisation**: summarise a document or email without sending content to cloud; privacy-critical use case
- **Text classification and routing**: classify user intent or sentiment; small output, fast
- **Proofreading and rewriting**: short passages; offline-capable grammar correction
- **On-device RAG** (Retrieval-Augmented Generation): embed a small local document corpus with a lightweight embedding model, retrieve top-k chunks, feed to Gemini Nano for answer synthesis

**Tasks where Gemini Nano is insufficient**:
- Long document processing (>8K tokens)
- Vision-language tasks (describe image, OCR + reason)
- Complex multi-step function calling
- Code generation (model quality below GPT-4-class for complex algorithms)

### 16.6 Gemini Nano vs LiteRT-LM + Open Weights

The on-device generative AI landscape now offers two distinct paths. Choosing between them involves quality, control, and hardware trade-offs:

| Dimension | Gemini Nano via ML Kit | LiteRT-LM + Gemma/PaliGemma |
|---|---|---|
| Model quality | Higher (proprietary, Google-trained) | Lower (open-weight, smaller parameter count) |
| Hardware requirement | Pixel 8+ / Galaxy S25 only | Any Android with ≥4 GB RAM |
| Model customisation | None (black box) | Full: fine-tune, LoRA, quantise |
| Weight access | None (encrypted in AICore) | Full FlatBuffer access |
| Privacy | On-device; AICore attestation | On-device; app-controlled |
| Cold start | 3–8 s if weights evicted | 1–3 s (smaller model) |
| Throughput | 15–25 tok/s | 25–56 tok/s (Gemma 2B INT4 on GPU) |
| Integration complexity | ML Kit dependency; Play Services required | Add `litert-lm` AAR; no Play Services |
| LoRA adapter support | No | Yes (LiteRT-LM adapter loading) |
| Multimodal (vision) | Not yet in public API | PaliGemma, Gemma 3 (text+image) |

**Decision rule**:
- Use **Gemini Nano** when: (1) the app targets Pixel 9+ as the primary device, (2) model quality matters more than broad device coverage, (3) the use case is text-in/text-out and privacy is paramount
- Use **LiteRT-LM + open weights** when: (1) the app needs to run on a wide range of Android devices, (2) fine-tuning or domain adaptation is needed, (3) vision input is required, or (4) no dependency on Google Play Services is acceptable

The two paths can coexist in a single app: check `FeatureStatus.AVAILABLE` for Gemini Nano; if not supported, fall back to LiteRT-LM with a smaller open model. This is the recommended production pattern for apps targeting both high-end and mid-range Android devices. [Source: Google AI Edge decision guide](https://ai.google.dev/edge/litert/models/gemma/android)

---

## 17. Roadmap

### 17.1 LiteRT-LM (Shipping, May 2026)

LiteRT-LM is the production orchestration layer for on-device LLM inference, superseding MediaPipe's `LlmInference` task. It achieves 52 tokens/second on Android GPU (OpenCL backend) and 56 tokens/second on Apple Metal, with multimodal support (text + vision + audio). Backends: CPU (XNNPack), GPU (OpenCL on Android, Metal on iOS, WebGPU in browser), NPU (Android only, via vendor delegates). [Source: LiteRT-LM announcement](https://developers.googleblog.com/blazing-fast-on-device-genai-with-litert-lm/)

KV-cache management stores Key and Value tensors in GPU high-bandwidth memory between decode steps, reducing per-token latency from O(context_length²) to O(context_length) via incremental attention computation.

### 17.2 Vulkan Compute Backend for GPU Delegate

The current GPU delegate defaults to OpenGL ES 3.1 compute shaders. The Vulkan compute backend (`TFLITE_GPU_EXPERIMENTAL_FLAGS_ENABLE_VULKAN_INFERENCE`) targets devices with Vulkan 1.1+. Advantages over GLES include lower driver overhead (no OpenGL state machine), explicit memory management (`VkDeviceMemory` from imported `AHardwareBuffer` via `VK_ANDROID_external_memory_android_hardware_buffer`), and access to Vulkan subgroup operations for more efficient SIMD reductions. Expect Vulkan backend to exit experimental status by late 2026.

### 17.3 LiteRT on Linux

The `ai-edge-litert` package is expanding to x86-64 and ARM64 Linux (targeting Raspberry Pi 5 and NVIDIA Jetson). The GPU delegate runs via OpenGL ES on Mesa (panfrost, v3d, freedreno). This is the convergence path between Android on-device ML and Linux embedded ML described in Ch88 and Ch124. [Source: LiteRT universal framework](https://developers.googleblog.com/litert-the-universal-framework-on-device-ai/)

### 17.4 WebAssembly / WebGPU

LiteRT-LM's Web SDK runs via WebGPU (WASM + JavaScript API) in Chrome and other browsers, with 2026 parity between mobile-native and browser inference performance on the same GPU. This connects to the WebGPU stack described in Ch35 (Dawn and WebGPU).

### 17.5 Vendor NPU Convergence

The post-NNAPI model (API 35+) pushes hardware acceleration into vendor-supplied LiteRT delegate libraries loaded at runtime. The long-term expectation is that all major Android SoC vendors (Qualcomm, MediaTek, Samsung Exynos, Google Tensor) ship production LiteRT delegates, converging on a uniform ABI where the delegate `.so` is the accelerator contract — analogous to how ICD `.so` files are the Vulkan driver contract.

### 17.6 On-Device Foundation Models

Vision-language models in the PaliGemma 3B class (3 billion parameters, vision + text) are feasible on 2026 flagship hardware at INT4 with LiteRT-LM. The GPU serves as the primary compute engine; the NPU handles INT8 attention layers. The camera→inference→display pipeline described in §10 of this chapter becomes the execution path for camera-grounded VLM queries ("What is this object?") running entirely on-device.

### 17.7 MediaPipe Tasks SDK Evolution

MediaPipe Tasks (§8) is transitioning from a community-maintained SDK to a Google-supported production API:
- **New modalities (2026–2027)**: Audio classification, face stylisation, and gesture customisation tasks are in preview. The Tasks API schema is being extended to support multi-modal tasks that accept both image and text inputs in a single `TaskRunner.process()` call, enabling on-device vision-language queries without the separate LiteRT-LM layer.
- **Stable ABI**: The current Tasks API targets Kotlin/Java and Python. A stable C API (`mediapipe/tasks/c/`) is under active development to allow C++ and Rust callers to consume Tasks results without going through the JNI boundary — eliminating the 0.5–1 ms JNI overhead on high-frequency (30 Hz) inference pipelines.
- **Calculator graph persistence**: The multi-calculator MediaPipe graph (§7.1) currently reconstructs packet routing on each `TaskRunner` call. A session-mode API keeps the `CalculatorGraph` resident in memory between frames, amortising graph construction cost across inference runs — critical for 60 Hz real-time use cases like AR overlay (§11).

### 17.8 Gemini Nano + LiteRT Convergence

Gemini Nano (the on-device Gemini variant) is currently deployed via AICore (`android.ai.app.aicore.AICore`) as a system-level service on Pixel 9+ devices. The convergence trajectory with LiteRT-LM:

- **Phase 1 (2026)**: Gemini Nano remains an AICore-brokered black box; apps call `GenerativeModel("gemini-nano")` via ML Kit GenAI API. LiteRT-LM serves open-weight models (Gemma 2B, PaliGemma). No shared inference path.
- **Phase 2 (2027+)**: Google is expected to expose Gemini Nano's internal transformer as a LiteRT-LM model, allowing apps to run the same Gemini model weights with the LiteRT-LM engine (rather than through AICore). This would allow fine-tuning, custom LoRA adapters, and private on-device inference without AICore telemetry.
- **Multimodal cache sharing**: LiteRT-LM and Gemini Nano both need KV-cache GPU memory. A shared KV-cache arbiter (analogous to how SurfaceFlinger arbitrates display memory) is the long-horizon solution for devices running multiple concurrent LLM sessions (e.g., ARCore scene description + background document assistant).

### 17.9 Competitive Landscape: GGML/llama.cpp vs LiteRT-LM

The on-device LLM ecosystem is splitting into two camps with different optimisation targets:

| Dimension | LiteRT-LM (Google) | llama.cpp / GGML |
|---|---|---|
| **Primary target** | Mobile (Android/iOS), browser | Desktop Linux, macOS, embedded |
| **GPU backend** | OpenCL, Metal, WebGPU | Vulkan, CUDA, Metal, ROCm |
| **Quantisation** | INT4 NF4, INT8 via `dynamic_range_quant` | GGUF Q2–Q8, FP16, BF16 |
| **Android integration** | First-class: NNAPI delegates, AHardwareBuffer, Thermal API | Via JNI wrapper (`llama-android`); no AHardwareBuffer |
| **Model ecosystem** | TFLite/LiteRT format; Keras export | GGUF format; llama3/Gemma/Mistral natively |
| **Latency target** | Real-time mobile (30–60 Hz inference) | High-throughput batch |
| **ARCore integration** | Via `ArFrame_acquireCameraImage()` → LiteRT pipeline | Not integrated |

The two ecosystems are converging at the GGUF level: Google has published a GGUF→LiteRT-LM converter, and `llama.cpp` has added experimental Android OpenCL support that targets the same Adreno/Mali GPU path as LiteRT-LM's GPU delegate.

### 17.10 GGUF and llama.cpp on Android

**GGUF** (GGML Unified Format) is the binary model file format used by `llama.cpp` and the broader GGML ecosystem. Introduced in August 2023 as a replacement for the older GGML binary format, GGUF is a self-describing container that stores model weights, tokeniser vocabulary, metadata, and architecture hyperparameters in a single file. It has become the dominant distribution format for open-weight LLMs on consumer hardware — the majority of quantised Llama 3, Mistral, Gemma, Phi-3, and Qwen models on HuggingFace are distributed as GGUF. [Source: GGUF specification](https://github.com/ggerganov/ggml/blob/master/docs/gguf.md)

#### GGUF File Structure

A GGUF file consists of a header followed by key-value metadata and tensor data blocks:

```
GGUF magic (4 bytes: "GGUF")
Version (uint32: 3)
Tensor count (uint64)
Metadata KV count (uint64)
─── Metadata key-value pairs ───
  general.architecture = "llama"
  general.name = "Llama-3-8B-Instruct"
  llama.context_length = 8192
  llama.embedding_length = 4096
  llama.block_count = 32
  llama.attention.head_count = 32
  tokenizer.ggml.model = "llama"
  tokenizer.ggml.tokens = [... vocab ...]
─── Tensor descriptors ───
  name, n_dims, shape, type (Q4_K_M / Q8_0 / F16 / ...)
─── Tensor data (page-aligned) ───
```

All metadata and tokeniser data is embedded directly — no separate vocabulary files or config JSON. This self-contained design makes GGUF models trivially distributable as single files, which is the primary reason for their adoption on Android where the APK asset model favours bundling.

#### GGUF Quantisation Schemes

GGUF defines its own quantisation types, distinct from LiteRT's INT8/INT4 schemes. The naming convention is `Q<bits>_<variant>`:

| Type | Bits/weight | Description | Size (7B model) | Quality loss |
|---|---|---|---|---|
| `F16` | 16 | Half-precision float; no quantisation | ~14 GB | None |
| `Q8_0` | 8 | Simple 8-bit; high quality | ~7.7 GB | Minimal |
| `Q6_K` | 6.5 | K-quant 6-bit; near-lossless | ~5.9 GB | Very low |
| `Q5_K_M` | 5.5 | K-quant 5-bit medium; recommended for quality | ~5.0 GB | Low |
| `Q4_K_M` | 4.5 | K-quant 4-bit medium; **best quality/size ratio** | ~4.1 GB | Low–medium |
| `Q4_K_S` | 4.4 | K-quant 4-bit small; slightly smaller than M | ~3.9 GB | Medium |
| `Q3_K_M` | 3.9 | K-quant 3-bit medium; aggressive compression | ~3.3 GB | Medium–high |
| `Q2_K` | 3.35 | 2-bit with 4-bit scale blocks | ~2.9 GB | High |
| `IQ4_NL` | 4.5 | Importance-aware 4-bit; often better than Q4_K_M | ~4.1 GB | Low |

**K-quants** (the `_K_` variants, introduced in GGUF) quantise weights in blocks of 256, storing a separate FP16 scale and minimum per block rather than per-tensor, which recovers ~0.5–1.0 perplexity point over the simpler `Q4_0` format at the same bit depth.

**`Q4_K_M` is the practical default for Android.** At ~4 GB for a 7B-parameter model, it fits in RAM on devices with 8 GB (leaving headroom for the OS and other apps), delivers 15–25 tokens/second on Snapdragon 8 Gen 2 CPU, and the quality degradation versus F16 is typically <5% on standard benchmarks.

#### llama.cpp on Android

`llama.cpp` builds natively for Android via the NDK. The official Android demo app (`examples/llama.android/`) provides a complete reference integration.

**Building from source:**

```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp

# Build for arm64-v8a with NEON optimisations:
cmake -B build-android \
    -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=arm64-v8a \
    -DANDROID_PLATFORM=android-26 \
    -DLLAMA_ANDROID_ENABLE_NEON=ON \
    -DLLAMA_VULKAN=ON \          # enable Vulkan compute backend
    -DCMAKE_BUILD_TYPE=Release

cmake --build build-android --config Release -j$(nproc)
# Produces: build-android/bin/llama-cli (executable for adb shell testing)
#           build-android/src/libllama.so (library for app integration)
```

**Vulkan compute backend on Android.** When `DLLAMA_VULKAN=ON`, llama.cpp uses Vulkan compute shaders for matrix-vector multiplication (the dominant operation in transformer inference). On Adreno 740+ and Mali G715+ the Vulkan path delivers 2–3× higher throughput than the NEON CPU path for Q4_K_M 7B models:

| Backend | Snapdragon 8 Gen 2 | Samsung Exynos 2400 |
|---|---|---|
| CPU NEON (4 threads) | ~15 tok/s (Q4_K_M 7B) | ~12 tok/s |
| Vulkan compute | ~35–45 tok/s | ~28 tok/s |
| OpenCL (experimental) | ~30–40 tok/s | ~25 tok/s |

**Integrating `libllama.so` into an Android app via JNI:**

The official `llama.android` example wraps `llama.cpp` in a thin JNI layer. The essential pattern:

```cpp
// File: app/src/main/cpp/llama-android.cpp
#include "llama.h"
#include <jni.h>
#include <android/log.h>

static llama_model* g_model = nullptr;
static llama_context* g_ctx = nullptr;

extern "C" JNIEXPORT void JNICALL
Java_com_example_llama_LlamaAndroid_loadModel(
        JNIEnv* env, jobject, jstring model_path_j) {
    const char* path = env->GetStringUTFChars(model_path_j, nullptr);

    llama_model_params mparams = llama_model_default_params();
    mparams.n_gpu_layers = 99;   // offload all layers to GPU (Vulkan)

    g_model = llama_load_model_from_file(path, mparams);
    env->ReleaseStringUTFChars(model_path_j, path);

    llama_context_params cparams = llama_context_default_params();
    cparams.n_ctx    = 4096;    // context window
    cparams.n_batch  = 512;     // prompt processing batch size
    cparams.n_threads = 4;      // CPU threads for non-GPU layers

    g_ctx = llama_new_context_with_model(g_model, cparams);
    __android_log_print(ANDROID_LOG_INFO, "llama", "Model loaded: %s", path);
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_llama_LlamaAndroid_generate(
        JNIEnv* env, jobject, jstring prompt_j) {
    const char* prompt = env->GetStringUTFChars(prompt_j, nullptr);

    // Tokenise prompt:
    std::vector<llama_token> tokens(llama_n_ctx(g_ctx));
    int n_tokens = llama_tokenize(g_model, prompt,
        strlen(prompt), tokens.data(), tokens.size(), true, false);
    tokens.resize(n_tokens);
    env->ReleaseStringUTFChars(prompt_j, prompt);

    // Evaluate prompt batch:
    llama_batch batch = llama_batch_get_one(tokens.data(), tokens.size());
    llama_decode(g_ctx, batch);

    // Greedy decode loop (simplified):
    std::string result;
    for (int i = 0; i < 256; ++i) {
        llama_token id = llama_sampler_sample(/* sampler */, g_ctx, -1);
        if (id == llama_token_eos(g_model)) break;
        result += llama_token_to_piece(g_model, id);
        batch = llama_batch_get_one(&id, 1);
        llama_decode(g_ctx, batch);
    }
    return env->NewStringUTF(result.c_str());
}
```

**Model delivery on Android.** A Q4_K_M GGUF for a 7B model is ~4 GB — far too large for the 150 MB APK limit. Delivery options:

1. **Play Asset Delivery** (on-demand pack): declare a `fast-follow` or `on-demand` asset pack containing the GGUF file; downloaded after install via `AssetPackManager` (ch161 §PAD). Max 512 MB per pack — requires splitting a 7B GGUF across two packs or using a smaller 1B–3B model (Phi-3-mini at Q4_K_M is ~2 GB).
2. **Runtime download**: download from a CDN (S3, Cloudflare R2) at first launch; store in `getFilesDir()` or `getExternalFilesDir()`. Simpler but requires network on first use.
3. **Small models only in APK**: 1B models (TinyLlama-1.1B Q4_K_M ~670 MB, Gemma-2-2B Q4_K_M ~1.4 GB) can fit within the 150 MB APK limit only with aggressive quantisation (Q2_K) — generally not recommended.

#### GGUF ↔ LiteRT-LM Conversion

Google's `ai-edge-litert` toolchain now includes a GGUF→LiteRT converter, allowing GGUF-format models to be run on LiteRT-LM's GPU delegate rather than llama.cpp's Vulkan backend:

```bash
pip install ai-edge-litert-nightly
python -m ai_edge_litert.tools.convert_gguf \
    --input_gguf llama-3-8b-instruct.Q4_K_M.gguf \
    --output_tflite llama-3-8b.tflite
```

The converted `.tflite` can then use LiteRT-LM's first-class Android integration (AHardwareBuffer memory, ADPF thermal hints via `AThermal_getThermalHeadroom`, Play Store delivery infrastructure) while retaining the GGUF quantised weights. The conversion is lossy at the quantisation level — GGUF Q4_K_M uses k-quant block scaling that does not map 1:1 to LiteRT's NF4/INT4 scheme; expect ~1–3% perplexity change after conversion.

#### MLC-LLM: TVM-Compiled Models on Android

**MLC-LLM** (Machine Learning Compilation for LLMs) is an alternative to both llama.cpp and LiteRT-LM. It compiles HuggingFace-format or GGUF models via Apache TVM's ML compiler to optimised Vulkan compute shaders or OpenCL kernels tuned specifically for the target GPU microarchitecture:

```bash
# Install MLC-LLM toolchain:
pip install mlc-llm mlc-ai-nightly

# Compile Llama-3-8B for Android Vulkan (Adreno target):
mlc_llm convert_weight ./Llama-3-8B-Instruct/ \
    --quantization q4f16_1 \
    --output ./llama3-8b-q4f16-MLC/

mlc_llm gen_config ./Llama-3-8B-Instruct/ \
    --quantization q4f16_1 \
    --conv-template llama-3 \
    --output ./llama3-8b-q4f16-MLC/

# Build Android app with compiled model:
mlc_llm package --task android \
    --model-list llama3-8b-q4f16-MLC
```

MLC-LLM's advantage is GPU-architecture-specific kernel autotuning — TVM's AutoScheduler finds the optimal tile sizes and memory layouts for a specific Adreno or Mali GPU model, achieving better throughput than llama.cpp's generic Vulkan shaders. The trade-off is a longer compile step (minutes per model per GPU target) and a less mature Android ecosystem compared to llama.cpp.

| Runtime | Format | Android GPU | Relative throughput (7B Q4, Adreno 740) |
|---|---|---|---|
| llama.cpp | GGUF | Vulkan generic | 35–45 tok/s |
| MLC-LLM | TVM compiled | Vulkan tuned | 50–65 tok/s |
| LiteRT-LM | `.tflite` INT4 | OpenCL | 45–56 tok/s |
| ORT GenAI | ONNX | CPU+QNN EP | 20–30 tok/s (QNN) |

[Source: llama.cpp GitHub](https://github.com/ggerganov/llama.cpp), [GGUF spec](https://github.com/ggerganov/ggml/blob/master/docs/gguf.md), [MLC-LLM Android](https://llm.mlc.ai/docs/deploy/android.html)

---

## 18. Integrations

**Ch85 — Android Compositor: SurfaceFlinger, HardwareBuffer, and the Buffer Pipeline**
`AHardwareBuffer` is the central memory object shared between the Camera HAL, the LiteRT GPU delegate, and SurfaceFlinger. The buffer lifecycle (allocate → acquire → GPU write → release → compositor acquire) described in Ch85 applies directly to the inference pipeline §§4 and 10.

**Ch86 — Vulkan on Android: Drivers, ANGLE, and Mobile GPU Performance**
The GPU delegate's Vulkan backend imports `AHardwareBuffer` via `VK_ANDROID_external_memory_android_hardware_buffer` (Ch86 §3). The same `VkSamplerYcbcrConversion` used for YUV camera textures in Ch86 is needed when feeding YCbCr camera frames to LiteRT vision models. ANGLE's GLES-on-Vulkan translation layer means that the GLES compute delegate runs on Vulkan on Chrome-backed Android targets.

**Ch87 — Android AR: ARCore Architecture, Camera HAL Integration, and the Android XR Platform**
§11 of this chapter describes the ARCore + MediaPipe composition pattern. ARCore provides world geometry; MediaPipe provides ML inference on the same camera texture. The shared GL context model, timestamp synchronisation (`ArFrame_getTimestamp()` → MediaPipe `Timestamp`), and the `GL_TEXTURE_EXTERNAL_OES` sharing pattern all build on the ARCore architecture in Ch87.

**Ch88 — NPU and AI Accelerator Integration on Linux**
Ch88 covers Linux NPU kernel drivers (Intel ivpu, AMD XDNA) and the OpenVINO/ROCm compute stack on desktop Linux. This chapter (§3.1) covers the Android NNAPI C API and HAL architecture in depth — the two chapters are complementary: Ch88 is the Linux NPU driver story; this chapter is the Android HAL-to-hardware path. Post-NNAPI, vendor LiteRT delegates (Qualcomm QNN, §3.3) access the Hexagon DSP via a direct in-process `.so` rather than the NNAPI HAL Binder IPC.

**Ch108 — ROCm and HIP — AMD's GPU Compute Stack**
On desktop Linux (x86-64), ROCm provides the ML compute stack via HIP/MIOpen. LiteRT on Linux uses OpenGL ES via Mesa instead. Ch108 provides the contrast: ROCm is a GPGPU runtime with batch-optimised kernels; LiteRT is a latency-optimised runtime designed for single-inference-at-a-time execution at mobile frame rates.

**Ch124 — Local LLM Inference on Linux GPUs**
LiteRT-LM (§11.1) converges on-device Android LLM inference with the desktop Linux LLM stack (llama.cpp, Ollama, vLLM) via the `ai-edge-litert` Linux package. Ch124 covers the llama.cpp/Vulkan compute path; this chapter covers LiteRT-LM's GPU delegate path, which targets the same GPU hardware on ARM64 Linux.

**Ch166 — Android AR: ARCore Architecture, Camera HAL Integration, and Android XR (Expanded)**
Ch166 expands the Android XR platform coverage (Samsung Galaxy XR / Project Moohan, Jetpack XR SDK). On Android XR headsets, MediaPipe's hand-landmark and pose-landmark tasks are the primary input recognition mechanisms, operating on passthrough camera streams at the full GPU bandwidth available to an XR-class SoC.

**Ch48 — ROCm and Machine Learning on Linux GPUs** *(optional)*
For context on how desktop-class AMD GPUs handle the same model families covered here, see Ch48's discussion of ROCm's MIOpen operator libraries — the functional equivalent of LiteRT's GPU delegate GLSL compute kernels at datacenter scale.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
