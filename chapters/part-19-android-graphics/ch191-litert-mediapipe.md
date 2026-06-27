# Chapter 191 — LiteRT and MediaPipe: On-Device ML Inference on the Android Graphics Stack

> **Audiences:** Graphics and GPU developers who want to understand how ML inference shares `AHardwareBuffer` and GPU compute infrastructure with the Android rendering pipeline; Android application developers integrating on-device ML with camera and display workflows; and systems engineers bridging the ML and GPU subsystems on Android, including those evaluating how the delegate and hardware-accelerator model maps onto the kernel DRM/DMA-BUF and HAL abstractions described in earlier parts of this book.

---

## Table of Contents

1. [Why ML Inference Belongs in a Graphics Stack Book](#1-why-ml-inference-belongs-in-a-graphics-stack-book)
2. [LiteRT Architecture](#2-litert-architecture)
3. [Delegate Architecture: Routing Inference to Hardware](#3-delegate-architecture-routing-inference-to-hardware)
   - [3.1 NNAPI In Depth](#31-nnapi-in-depth): architecture, HAL history, model building, compilation, burst execution, AHardwareBuffer memory domains, device enumeration, LiteRT delegate internals, deprecation
4. [AHardwareBuffer Tensor Interop](#4-ahardwarebuffer-tensor-interop)
5. [Model Formats and Quantisation](#5-model-formats-and-quantisation)
6. [MediaPipe Framework Architecture](#6-mediapipe-framework-architecture)
7. [MediaPipe Tasks API](#7-mediapipe-tasks-api)
8. [Camera2 → MediaPipe Zero-Copy Pipeline](#8-camera2--mediapipe-zero-copy-pipeline)
9. [ARCore + MediaPipe Composition](#9-arcore--mediapipe-composition)
10. [Performance Profiling and Tuning](#10-performance-profiling-and-tuning)
11. [Roadmap](#11-roadmap)
12. [Integrations](#12-integrations)

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

### 3.1 NNAPI In Depth

> **Status:** NNAPI was deprecated in Android 15 (API level 35, 2024). For new development targeting API 35+, use vendor NPU delegates (§3.3) or the GPU delegate. For devices on Android 8.1–14 (API 27–34) — still the large majority of the installed base in 2026 — NNAPI remains the primary hardware acceleration path. This section covers the API and architecture in depth because NNAPI-era devices dominate the field population and the deprecation does not remove existing functionality. [Source: Android NNAPI migration guide](https://developer.android.com/ndk/guides/neuralnetworks/migration-guide)

#### 3.1.1 Architecture: From C API to Hardware

NNAPI is a layered system spanning four components:

```
App / LiteRT NNAPI delegate
         │  NeuralNetworks.h C API  (libneuralnetworks.so)
         ▼
    NNAPI Runtime (system process: neuralnetworks_service)
         │  android.hardware.neuralnetworks HAL
         ▼
  OEM HAL implementation (.so loaded by hwservicemanager)
         │  vendor-specific command submission
         ▼
  Hardware: DSP (Hexagon), NPU, GPU
```

The `libneuralnetworks.so` library (part of AOSP, lives in `packages/modules/NeuralNetworks/`) exports the public C API. It links against a runtime that communicates with the NNAPI HAL service across the HIDL/AIDL interface boundary, using a Binder IPC. The HAL implementation (`android.hardware.neuralnetworks@1.x` or AIDL `android.hardware.neuralnetworks`) lives in vendor partition and loads firmware (e.g., Hexagon DSP firmware on Snapdragon) independently.

From Android 13+, an **NNAPI Shim** (`INeuralNetworksShim`) enables hardware vendors to ship NNAPI-compatible implementations as APEX modules — the same mainline-modularity mechanism used for ART (Ch164 §1.3). This decouples the NNAPI HAL version from the system image OTA and allows faster hardware bring-up. [Source: AOSP NNAPI Shim](https://source.android.com/docs/core/ml/nnapi/nnapi-shim)

#### 3.1.2 HAL Version History

| HAL Version | Android Release | API Level | Key Additions |
|---|---|---|---|
| v1.0 | Android 8.1 Oreo | 27 | 29 ops: Conv2D, DepthwiseConv2D, FullyConnected, MaxPool2D, AveragePool2D, RELU, Softmax, Reshape, CONCATENATION; FP32 only |
| v1.1 | Android 9 Pie | 28 | 8 new ops: BATCH_TO_SPACE_ND, DIV, MEAN, PAD, SPACE_TO_BATCH_ND, SQUEEZE, STRIDED_SLICE, SUB; relaxed FP16 compute option |
| v1.2 | Android 10 Q | 29 | 60+ new ops including LSTM variants, ARGMAX, TOPK_V2, ELU, PRELU; **device enumeration** (`ANeuralNetworksDevice`); **AHardwareBuffer memory domains** (`ANeuralNetworksMemory_createFromAHardwareBuffer`); **sync fence execution** (`ANeuralNetworksExecution_startComputeWithDependencies`); **burst execution** (`ANeuralNetworksBurst_create`); performance hints via `ANeuralNetworksExecution_setTimeout` |
| v1.3 | Android 11 R | 30 | Signed INT8 quantisation (vs unsigned INT8 in v1.2); memory descriptor API (`ANeuralNetworksMemoryDesc`); fenced memory copies; quality of service API (`ANeuralNetworksCompilation_setPriority`) |
| AIDL v1 | Android 12 S | 31 | HAL migrated from HIDL to AIDL; Shim API for APEX delivery |
| AIDL v2 | Android 13 T | 33 | NNAPI Shim GA; fenced execution improvements |
| Deprecated | Android 15 V | 35 | `@Deprecated` annotation; API remains functional but no new features |

[Source: AOSP NeuralNetworks HAL evolution](https://source.android.com/docs/core/ml/nnapi)

#### 3.1.3 Model Building: `ANeuralNetworksModel`

Building an NNAPI model is explicit and low-level — there is no graph tracing or `torch.jit`. Each operand (tensor or scalar) is added individually, then operations are wired by operand index:

```c
#include <android/NeuralNetworks.h>

ANeuralNetworksModel* model = NULL;
ANeuralNetworksModel_create(&model);

// 1. Add operands (tensors and scalars)
// Input tensor: {1, 224, 224, 3} FLOAT32
ANeuralNetworksOperandType input_type = {
    .type = ANEURALNETWORKS_TENSOR_FLOAT32,
    .dimensionCount = 4,
    .dimensions = (uint32_t[]){1, 224, 224, 3},
    .scale = 0.0f,
    .zeroPoint = 0,
};
ANeuralNetworksModel_addOperand(model, &input_type);  // operand index 0

// Filter tensor: {32, 3, 3, 3} FLOAT32 (32 filters, 3×3 kernel, 3 input channels)
ANeuralNetworksOperandType filter_type = {
    .type = ANEURALNETWORKS_TENSOR_FLOAT32,
    .dimensionCount = 4,
    .dimensions = (uint32_t[]){32, 3, 3, 3},
    .scale = 0.0f, .zeroPoint = 0,
};
ANeuralNetworksModel_addOperand(model, &filter_type);  // operand index 1

// Bias: {32} FLOAT32
ANeuralNetworksOperandType bias_type = {
    .type = ANEURALNETWORKS_TENSOR_FLOAT32,
    .dimensionCount = 1,
    .dimensions = (uint32_t[]){32},
    .scale = 0.0f, .zeroPoint = 0,
};
ANeuralNetworksModel_addOperand(model, &bias_type);    // operand index 2

// Scalar operands for Conv2D padding, strides, activation
// ANEURALNETWORKS_INT32 scalars for: padding_mode, stride_w, stride_h, dilation_w, dilation_h, activation_fn
for (int i = 0; i < 5; i++) {
    ANeuralNetworksOperandType scalar = {
        .type = ANEURALNETWORKS_INT32, .dimensionCount = 0,
    };
    ANeuralNetworksModel_addOperand(model, &scalar);   // indices 3-7
}

// Output tensor: {1, 222, 222, 32} FLOAT32
ANeuralNetworksOperandType output_type = {
    .type = ANEURALNETWORKS_TENSOR_FLOAT32,
    .dimensionCount = 4,
    .dimensions = (uint32_t[]){1, 222, 222, 32},
    .scale = 0.0f, .zeroPoint = 0,
};
ANeuralNetworksModel_addOperand(model, &output_type);  // operand index 8

// 2. Set constant values for filter, bias, and scalar operands
ANeuralNetworksModel_setOperandValue(model, 1, filter_weights, filter_bytes);
ANeuralNetworksModel_setOperandValue(model, 2, bias_data, bias_bytes);
int32_t padding     = ANEURALNETWORKS_PADDING_VALID;   // operand 3
int32_t stride_w    = 1;   // operand 4
int32_t stride_h    = 1;   // operand 5
int32_t dilation_w  = 1;   // operand 6
int32_t dilation_h  = 1;   // operand 7 (v1.2+)
// (setOperandValue calls for scalars omitted for brevity)

// 3. Add the Conv2D operation
uint32_t conv_inputs[]  = {0, 1, 2, 3, 4, 5, 6, 7};
uint32_t conv_outputs[] = {8};
ANeuralNetworksModel_addOperation(model,
    ANEURALNETWORKS_CONV_2D,
    8, conv_inputs,
    1, conv_outputs);

// 4. Identify graph inputs and outputs
ANeuralNetworksModel_identifyInputsAndOutputs(model, 1, (uint32_t[]){0}, 1, (uint32_t[]){8});

// 5. Finish: validates the graph and makes it immutable
ANeuralNetworksModel_finish(model);
```
[Source: AOSP NeuralNetworks.h](https://developer.android.com/ndk/reference/group/neural-networks)

**Quantised operands** use `ANEURALNETWORKS_TENSOR_QUANT8_ASYMM` (v1.0+) or `ANEURALNETWORKS_TENSOR_QUANT8_ASYMM_SIGNED` (v1.3+). The `scale` and `zeroPoint` fields in `ANeuralNetworksOperandType` encode the quantisation parameters (`value = (q - zeroPoint) * scale`). The transition from unsigned `QUANT8_ASYMM` (zeroPoint 0–255) to signed `QUANT8_ASYMM_SIGNED` (zeroPoint −128 to 127) in v1.3 aligns with TFLite's default INT8 quantisation scheme.

#### 3.1.4 Compilation: Preferences, Caching, and Priority

After the model is finalised, `ANeuralNetworksCompilation` translates the abstract graph into hardware-specific kernel code. Compilation is the most expensive step — potentially 1–10 seconds on first run:

```c
ANeuralNetworksCompilation* compilation = NULL;
// Basic compilation (targets default accelerator):
ANeuralNetworksCompilation_create(model, &compilation);

// Or target a specific device (API 29+):
// ANeuralNetworksCompilation_createForDevices(model, devices, num_devices, &compilation);

// Execution preference — affects scheduling and power:
// ANEURALNETWORKS_PREFER_LOW_POWER        — battery-sensitive workloads
// ANEURALNETWORKS_PREFER_FAST_SINGLE_ANSWER — lowest latency, ignore power
// ANEURALNETWORKS_PREFER_SUSTAINED_SPEED  — throughput (camera stream inference)
ANeuralNetworksCompilation_setPreference(compilation,
    ANEURALNETWORKS_PREFER_SUSTAINED_SPEED);

// Priority (API 30+): HIGH, MEDIUM (default), LOW
// HIGH processes the execution before MEDIUM/LOW in the hardware queue.
ANeuralNetworksCompilation_setPriority(compilation,
    ANEURALNETWORKS_PRIORITY_HIGH);

// Compilation caching (API 29+):
// Without caching, compilation runs on every app cold start.
// With caching, the compiled binary is stored at cache_dir/model_token.{nb,nc}
// and loaded directly on subsequent runs — reducing first-inference latency
// from ~3 seconds to ~30 ms on a Snapdragon 888.
ANeuralNetworksCompilation_setCaching(compilation,
    "/data/data/com.example.app/nnapi_cache",
    (uint8_t[32]){"mymodel_v1_sha256_truncated"});

// Finalise compilation (blocks until hardware compilation completes):
ANeuralNetworksCompilation_finish(compilation);
```

**Cache invalidation:** The model token is an opaque 32-byte key. Apps must change the token when the model weights change; NNAPI does not verify cache freshness by content-hashing weights. A common pattern is to embed the model file's SHA-256 (truncated to 32 bytes) as the token.

#### 3.1.5 Execution: Synchronous, Asynchronous, and Burst

NNAPI provides three execution modes:

**Synchronous (API 30+, preferred for single-shot queries):**
```c
ANeuralNetworksExecution* exec = NULL;
ANeuralNetworksExecution_create(compilation, &exec);

// Set input/output buffers:
ANeuralNetworksExecution_setInput(exec, 0, NULL, input_buffer, input_bytes);
ANeuralNetworksExecution_setOutput(exec, 0, NULL, output_buffer, output_bytes);

// Blocking call — returns when inference completes:
ANeuralNetworksExecution_compute(exec);   // API 29+; blocks caller thread

ANeuralNetworksExecution_free(exec);
```

**Asynchronous with sync fence (API 29+ for fences, recommended for camera pipelines):**
```c
ANeuralNetworksExecution_create(compilation, &exec);
ANeuralNetworksExecution_setInput(exec, 0, NULL, input_buffer, input_bytes);
ANeuralNetworksExecution_setOutput(exec, 0, NULL, output_buffer, output_bytes);

// Optional: wait for input to be ready (e.g., fence from Camera HAL or GPU):
// ANeuralNetworksExecution_startComputeWithDependencies takes an array of
// ANeuralNetworksEvent* from prior operations as dependency fences.
ANeuralNetworksEvent* input_ready_event = /* ... from GPU render or Camera */ NULL;
ANeuralNetworksEvent* compute_event = NULL;

ANeuralNetworksExecution_startComputeWithDependencies(
    exec,
    &input_ready_event, 1,   // wait for these fences first
    0,                        // deadline (0 = no deadline, API 30+)
    &compute_event);

// compute_event is signalled when NNAPI finishes.
// You can pass it as input_ready_event to downstream operations.
ANeuralNetworksEvent_wait(compute_event);   // block if needed
ANeuralNetworksEvent_free(compute_event);
```

**Burst execution (API 29+, for high-frequency streams):**

Burst objects amortise the per-execution overhead of IPC between the app process and the NNAPI runtime daemon. A burst keeps a shared memory channel open across repeated executions on the same compilation:

```c
ANeuralNetworksBurst* burst = NULL;
ANeuralNetworksBurst_create(compilation, &burst);

// Inference loop at 30 Hz:
while (running) {
    ANeuralNetworksExecution* exec = NULL;
    ANeuralNetworksExecution_create(compilation, &exec);
    ANeuralNetworksExecution_setInput(exec, 0, NULL, frame_buffer, frame_bytes);
    ANeuralNetworksExecution_setOutput(exec, 0, NULL, result_buffer, result_bytes);

    // burstCompute uses the persistent burst channel — lower IPC overhead
    // than ANeuralNetworksExecution_compute() per call:
    ANeuralNetworksExecution_burstCompute(exec, burst);
    ANeuralNetworksExecution_free(exec);

    process_result(result_buffer);
}

ANeuralNetworksBurst_free(burst);
```

On a Snapdragon 888 Hexagon DSP, burst execution reduces per-inference IPC latency from ~0.8 ms (non-burst) to ~0.15 ms by avoiding repeated Binder transaction setup.

#### 3.1.6 Memory Domains: Zero-Copy with AHardwareBuffer

NNAPI v1.2 (API 29) introduced **memory domains** — a mechanism for passing `AHardwareBuffer`-backed tensors directly to NNAPI without CPU copies. This is the same `AHardwareBuffer` that SurfaceFlinger and the Camera HAL exchange, meaning the camera→inference path can be zero-copy end-to-end:

```c
// AHardwareBuffer received from Camera2 ImageReader or from GPU render target:
AHardwareBuffer* ahb = /* ... */ NULL;

// Wrap the AHardwareBuffer as an NNAPI memory object:
ANeuralNetworksMemory* nnapi_memory = NULL;
ANeuralNetworksMemory_createFromAHardwareBuffer(ahb, &nnapi_memory);

// Set the input tensor to point into the AHardwareBuffer memory:
ANeuralNetworksExecution_setInputFromMemory(
    exec,
    0,          // input index
    NULL,       // use full tensor type from compilation
    nnapi_memory,
    0,          // byte offset within the AHardwareBuffer
    input_bytes);

ANeuralNetworksExecution_compute(exec);

ANeuralNetworksMemory_free(nnapi_memory);
// AHardwareBuffer release is the caller's responsibility
```

**Memory descriptor (API 30+)** provides finer-grained control over which hardware-specific memory pools the NNAPI driver allocates:

```c
// Ask NNAPI to allocate a memory object in a format the DSP can access natively:
ANeuralNetworksMemoryDesc* desc = NULL;
ANeuralNetworksMemoryDesc_create(&desc);

// Declare usage for input operand 0 of the compiled model:
ANeuralNetworksMemoryDesc_addInputRole(desc, compilation, 0, 1.0f);
ANeuralNetworksMemoryDesc_finish(desc);

ANeuralNetworksMemory* dsp_memory = NULL;
ANeuralNetworksMemory_createFromDesc(desc, &dsp_memory);
ANeuralNetworksMemoryDesc_free(desc);

// Now copy host data in once:
ANeuralNetworksMemory_copy(host_nnapi_memory, dsp_memory);

// Use dsp_memory directly in execution — no host↔DSP copy at inference time:
ANeuralNetworksExecution_setInputFromMemory(exec, 0, NULL, dsp_memory, 0, input_bytes);
```

[Source: AOSP ANeuralNetworksMemory](https://developer.android.com/ndk/reference/group/neural-networks#aneuralnetworksmemory_createfromdesc)

#### 3.1.7 Device Enumeration and Capability Querying

NNAPI v1.2 (API 29) added `ANeuralNetworksDevice`, allowing apps to enumerate available accelerators and select a specific one for compilation:

```c
uint32_t num_devices = 0;
ANeuralNetworks_getDeviceCount(&num_devices);

for (uint32_t i = 0; i < num_devices; i++) {
    ANeuralNetworksDevice* device = NULL;
    ANeuralNetworks_getDevice(i, &device);

    const char* name = NULL;
    ANeuralNetworksDevice_getName(device, &name);

    // Device type: CPU (reference), GPU, ACCELERATOR (DSP/NPU), UNKNOWN
    int32_t type = 0;
    ANeuralNetworksDevice_getType(device, &type);

    const char* version = NULL;
    ANeuralNetworksDevice_getVersion(device, &version);

    // Feature level corresponds to HAL version:
    // 27 = v1.0, 28 = v1.1, 29 = v1.2, 30 = v1.3
    int64_t feature_level = 0;
    ANeuralNetworksDevice_getFeatureLevel(device, &feature_level);

    // Check whether the model can be compiled on this device:
    bool supported = false;
    ANeuralNetworksModel_getSupportedOperationsForDevices(
        model, &device, 1, &supported);

    printf("Device %u: %s (type=%d, HAL level=%lld, model_supported=%d)\n",
        i, name, type, (long long)feature_level, (int)supported);
}
```

Typical output on a Snapdragon 8 Gen 3 device:
```
Device 0: nnapi-reference (type=ANEURALNETWORKS_DEVICE_TYPE_CPU, HAL level=30, model_supported=1)
Device 1: qti-hta (type=ANEURALNETWORKS_DEVICE_TYPE_ACCELERATOR, HAL level=30, model_supported=1)
Device 2: qti-gpu (type=ANEURALNETWORKS_DEVICE_TYPE_GPU, HAL level=29, model_supported=1)
Device 3: qti-dsp (type=ANEURALNETWORKS_DEVICE_TYPE_ACCELERATOR, HAL level=30, model_supported=1)
```

`nnapi-reference` is always present — it is the AOSP CPU reference implementation and is the fallback for ops unsupported by hardware accelerators. `qti-dsp` is the Hexagon DSP; `qti-hta` is Qualcomm's Hexagon Tensor Accelerator.

**Performance measurement (API 29+):** Call `ANeuralNetworksDevice_getPerformanceInfo()` to retrieve the hardware's stated FP32, FP16, and quantised INT8 performance figures (in ops/second) and the relative power consumption vs the `nnapi-reference` CPU implementation. These are vendor-self-reported figures, not benchmarks, but they are useful for selecting between multiple accelerators.

#### 3.1.8 LiteRT NNAPI Delegate Internals

The LiteRT NNAPI delegate (`tflite/delegates/nnapi/nnapi_delegate.cc`) bridges the LiteRT graph format to the NNAPI C API. Its internal flow on `ModifyGraphWithDelegate()`:

1. **Op support query**: For each node in the claimed partition, the delegate calls `ANeuralNetworksModel_getSupportedOperationsForDevices()` to ask the target device which ops it supports. Unsupported ops remain on CPU.

2. **Graph translation**: Each supported TFLite op maps to an NNAPI op code. The mapping handles:
   - **Activation fusion**: TFLite encodes activations inline in Conv/FC ops (via `activation` attribute). NNAPI v1.0 supports fused activation in Conv2D directly; for ops that don't support fusion, the delegate inserts a separate `ANEURALNETWORKS_RELU` op.
   - **Padding mode translation**: TFLite uses `SAME`/`VALID` string attributes; NNAPI uses `ANEURALNETWORKS_PADDING_SAME` / `ANEURALNETWORKS_PADDING_VALID` integer operands.
   - **Dynamic shapes**: LiteRT supports dynamic batch/spatial dimensions; NNAPI v1.0–v1.2 requires fixed shapes. The delegate resizes the NNAPI model when `interpreter->ResizeInputTensor()` is called.

3. **FP16 relaxation**: When `options.allow_fp16 = true`, the delegate calls `ANeuralNetworksModel_relaxComputationFloat32toFloat16(model, true)`. This instructs the hardware runtime that FP32 tensor accumulations may be performed in FP16 — a significant speedup on DSPs and mobile GPUs at the cost of slight precision reduction.

4. **Compilation preference mapping**:
   | LiteRT `execution_preference` | NNAPI preference |
   |---|---|
   | `kFastSingleAnswer` | `ANEURALNETWORKS_PREFER_FAST_SINGLE_ANSWER` |
   | `kSustainedSpeed` | `ANEURALNETWORKS_PREFER_SUSTAINED_SPEED` |
   | `kLowPower` | `ANEURALNETWORKS_PREFER_LOW_POWER` |
   | `kUndecided` | `ANEURALNETWORKS_PREFER_FAST_SINGLE_ANSWER` |

5. **Execution submission**: The delegate uses burst execution (if the target device supports API 29+) or falls back to synchronous `ANeuralNetworksExecution_compute()`. Sync-fence-based execution (`startComputeWithDependencies`) is used when the input tensor is backed by an `AHardwareBuffer` and the caller provides a GPU or camera fence.

**Using the delegate:**

```cpp
#include "tflite/delegates/nnapi/nnapi_delegate.h"

StatefulNnApiDelegate::Options options;
options.execution_preference =
    StatefulNnApiDelegate::Options::kSustainedSpeed;
options.allow_fp16 = true;
options.allow_dynamic_dimensions = true;   // API 29+ only
options.max_number_delegated_partitions = 3;
options.cache_dir = cache_dir.c_str();
options.model_token = model_token.c_str();

// Optionally target a specific named device:
options.accelerator_name = "qti-dsp";

auto nnapi_delegate = std::make_unique<StatefulNnApiDelegate>(options);
interpreter->ModifyGraphWithDelegate(nnapi_delegate.get());
```

[Source: LiteRT NNAPI delegate options](https://ai.google.dev/edge/litert/android/delegates/nnapi)

#### 3.1.9 Supported Op Coverage and Fallback Behaviour

Not all TFLite ops have NNAPI equivalents, and hardware implementations vary in which ops they support:

| Category | Representative TFLite ops | NNAPI v1.0 | NNAPI v1.2 | Hardware Reality |
|---|---|---|---|---|
| Convolution | Conv2D, DepthwiseConv2D, TransposeConv | ✅ | ✅ | Full DSP support |
| Activation | ReLU, ReLU6, Sigmoid, Tanh | ✅ | ✅ | Full DSP support |
| Pooling | MaxPool2D, AveragePool2D | ✅ | ✅ | Full DSP support |
| FC / Dense | FullyConnected | ✅ | ✅ | Full DSP support |
| Normalisation | BatchNorm (as mul+add), L2Norm | ✅ | ✅ | Often CPU fallback |
| Recurrent | LSTM, RNN, GRU | ❌ | ✅ | Patchy hardware support |
| Element-wise | Add, Mul, Sub, Div | ✅ (add/mul) | ✅ | Full support |
| Reshape/slice | Reshape, Squeeze, StridedSlice | ❌ | ✅ | Usually fused |
| Reduction | Mean, ArgMax, TopK | ❌ | ✅ | Often CPU fallback |
| Custom ops | any `@custom_op` | ❌ | ❌ | CPU only |

When a hardware driver rejects an op, the NNAPI delegate automatically partitions that op back to the CPU partition — this is "automatic fallback". The cost is that data must cross from GPU/DSP memory to CPU memory at each partition boundary, which can eliminate the speedup from hardware acceleration if many ops fall back. Calling `ANeuralNetworksModel_getSupportedOperationsForDevices()` beforehand lets you audit coverage before deciding whether to delegate.

**Measuring actual delegation:** LiteRT's `Interpreter::execution_plan()` and `GetDelegateNodeAndRegistrations()` APIs enumerate which nodes are covered by which delegate at runtime. The [`benchmark_model`](https://ai.google.dev/edge/litert/performance/measurement) tool's `--enable_op_profiling` flag reports per-op latency to identify which ops are running on hardware vs CPU fallback.

#### 3.1.10 Why NNAPI Was Deprecated

Google deprecated NNAPI at API 35 for several compounding reasons:

1. **HAL version fragmentation**: A significant fraction of Android 14 devices shipped with NNAPI HAL v1.0 or v1.1 (Android 8.1–9 era hardware without firmware updates), meaning apps could not rely on burst execution, AHardwareBuffer memory domains, or sync fences being available — forcing complex runtime version-checking.

2. **Vendor quality variability**: The NNAPI specification left enough implementation latitude that op results on some vendors' hardware differed numerically from the reference CPU implementation beyond acceptable tolerances. Google documented many such issues in the [NNAPI conformance suite](https://source.android.com/docs/core/ml/nnapi/conformance), but could not enforce fixes without breaking the vendor contract.

3. **Compilation time**: NNAPI compilation (especially without caching) could take multiple seconds. The caching mechanism helped but required apps to manage cache invalidation manually.

4. **Indirect dispatch overhead**: The HAL Binder IPC introduced ~0.5–1 ms round-trip overhead per inference that direct in-process delegates (§3.3) avoid entirely.

5. **New accelerator architectures**: Newer NPU designs (Pixel Tensor G4, Snapdragon X Elite NPU, Dimensity 9400 APU) expose capabilities that don't map cleanly to the NNAPI op vocabulary — requiring vendor-specific extensions that undermined the portability goal.

The replacement model (§3.3) is vendor delegates loaded directly into the app process as `.so` files, with the LiteRT delegate ABI as the only contract. This is a tighter API with lower overhead and allows vendors to expose hardware-specific features without the HAL versioning constraint.

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

### 3.3 NPU / Vendor Delegates

Following NNAPI's deprecation, chip vendors ship LiteRT delegate libraries directly:

- **Qualcomm AI Engine Direct Delegate** (`libQnnTFLiteDelegate.so`): Routes inference to the Hexagon DSP via the Qualcomm Neural Network API (QNN). Available on Maven Central. [Source: Google Developers Blog, Qualcomm NPU LiteRT](https://developers.googleblog.com/unlocking-peak-performance-on-qualcomm-npu-with-litert/)
- **Google Tensor NPU delegate**: Experimental Tensor ML SDK targeting Pixel 9's Tensor G4 TPU; expected general availability in late 2026.
- **Intel OpenVINO delegate**: Targets Intel NPU on x86 platforms via LiteRT's `CompiledModel` API.

### 3.4 Edge TPU Delegate

Coral hardware (USB, M.2, and PCIe Edge TPU accelerators) uses a separate delegate:

```cpp
#include "edgetpu.h"

TfLiteDelegate* edgetpu_delegate = edgetpu_create_delegate(
    EDGETPU_APEX_USB, /*device=*/nullptr, /*options=*/nullptr, 0);
interpreter->ModifyGraphWithDelegate(edgetpu_delegate);
```

Edge TPU models must be compiled with the [Edge TPU compiler](https://coral.ai/docs/edgetpu/compiler/) ahead of time, which quantises ops to INT8 and maps them to the TPU's SIMD arrays. Ops that the compiler cannot map remain on CPU.

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

### 5.6 ONNX and PyTorch Conversion

The `ai-edge-torch` package provides a PyTorch-native export path:

```python
import ai_edge_torch

# Convert directly from a PyTorch nn.Module
edge_model = ai_edge_torch.convert(torch_model.eval(), sample_inputs)
edge_model.export("model.tflite")
```

The resulting `.tflite` is semantically identical to one produced from TensorFlow SavedModel. `onnx-tf` followed by `tf.lite.TFLiteConverter` remains an option for ONNX sources, though `ai-edge-torch` is now preferred for PyTorch-originating models. [Source: ai-edge-torch GitHub](https://github.com/google-ai-edge/ai-edge-torch)

| Runtime | Format | Primary Backend | Android Delegation |
|---|---|---|---|
| LiteRT | `.tflite` | GPU (GLES/Vulkan), NPU | GPU delegate, Qualcomm QNN |
| ONNX Runtime Mobile | `.onnx` / `.ort` | CPU / NNAPI | NNAPI EP (deprecated API 35+) |
| CoreML (iOS) | `.mlpackage` | ANE | N/A |
| OpenVINO Mobile | `.xml`/`.bin` | CPU / Intel NPU | LiteRT CompiledModel |

---

## 6. MediaPipe Framework Architecture

**MediaPipe** is Google's cross-platform framework for building live, streaming ML pipelines from modular, composable processing nodes. It is developed in the [google-ai-edge/mediapipe](https://github.com/google-ai-edge/mediapipe) repository and supports Android, iOS, Linux, macOS, and Web (via WASM). [Source: MediaPipe framework](https://developers.google.com/mediapipe/framework)

### 6.1 CalculatorGraph

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

### 6.2 Calculator Lifecycle

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

### 6.3 Packets and Timestamps

A `Packet` is an immutable, typed, reference-counted data container with a `Timestamp`:

```cpp
// Wrapping data in a Packet
auto packet = mediapipe::MakePacket<mediapipe::GpuBuffer>(gpu_buffer)
                  .At(mediapipe::Timestamp(frame_timestamp_us));

// Extracting data from a Packet
const mediapipe::GpuBuffer& buffer = packet.Get<mediapipe::GpuBuffer>();
```

Timestamps are in microseconds and flow monotonically through the graph. The graph scheduler dispatches `Process()` in timestamp order, enabling correct synchronisation when multiple input streams must be aligned (e.g., combining camera frames with depth frames that arrive at different rates).

### 6.4 GlCalculatorHelper and GPU Context Management

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

### 6.5 GpuBuffer, GlTexture, and AHardwareBuffer Interop

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

### 6.6 InputStreamHandlers

`CalculatorGraph` supports multiple scheduling modes:

- `DefaultInputStreamHandler`: Synchronises packets across all input streams by timestamp. A calculator's `Process()` is called only when all inputs have a packet at the same timestamp. Appropriate for cameras + depth-sensor fusion.
- `ImmediateInputStreamHandler`: Calls `Process()` whenever any new packet arrives on any input stream, without waiting for other streams. Appropriate for low-latency single-stream inference.
- `FixedSizeInputStreamHandler`: Bounds queue depth to prevent backpressure accumulation during transient slowdowns.

---

## 7. MediaPipe Tasks API

The **Tasks API**, introduced in 2022, wraps `CalculatorGraph` behind strongly-typed, high-level task interfaces that hide graph configuration entirely. It is the recommended entry point for application developers. [Source: MediaPipe Tasks API](https://developers.google.com/mediapipe/solutions/guide)

### 7.1 BaseOptions and RunningMode

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

### 7.2 Available Vision Tasks

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

### 7.3 PoseLandmarker — Kotlin Live-Stream Example

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

### 7.4 LLM Inference Task

MediaPipe's `LlmInference` task (Android/iOS) has been superseded by **LiteRT-LM** (announced May 2026), which provides a dedicated orchestration layer for autoregressive LLM inference including KV-cache management, INT4/INT8 weight dispatch, and NPU routing. The MediaPipe LLM Inference API is now in maintenance-only mode; new projects should use the LiteRT-LM Android/iOS SDK. [Source: LiteRT-LM announcement](https://developers.googleblog.com/blazing-fast-on-device-genai-with-litert-lm/)

---

## 8. Camera2 → MediaPipe Zero-Copy Pipeline

The primary production use-case for MediaPipe on Android is live inference on camera frames. The full zero-copy path avoids any CPU-side pixel copy between the camera HAL and the inference GPU.

### 8.1 Camera2 SurfaceTexture Path

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

### 8.2 CameraX Integration

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

### 8.3 Timestamp Synchronisation

Camera2 timestamps are in nanoseconds since boot (`Image.getTimestamp()`, type `long`). MediaPipe timestamps are in microseconds. The conversion is:

```kotlin
val mediapipeTimestampUs = imageProxy.imageInfo.timestamp / 1_000L
poseLandmarker.detectAsync(mpImage, mediapipeTimestampUs)
```

Monotonic alignment is guaranteed since both use `CLOCK_BOOTTIME`. Failure to convert correctly manifests as the scheduler rejecting out-of-order packets — a common integration bug.

### 8.4 AHardwareBuffer Direct Path (API 29+)

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

## 9. ARCore + MediaPipe Composition

Combining ARCore (Ch87) with MediaPipe enables spatial-aware on-device inference: ARCore provides world geometry, camera pose, and the background camera texture; MediaPipe runs ML inference on the same camera image simultaneously.

### 9.1 Shared GL Context and Texture

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

### 9.2 Timestamp Synchronisation

`ArFrame_getTimestamp()` returns nanoseconds since boot (`int64_t`), identical to Camera2's `Image.getTimestamp()`. Convert to MediaPipe microseconds:

```c
int64_t ar_timestamp_ns;
ArFrame_getTimestamp(ar_session, ar_frame, &ar_timestamp_ns);
mediapipe::Timestamp mp_timestamp(ar_timestamp_ns / 1000LL);
```

### 9.3 Person Segmentation Overlay on AR Scene

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

### 9.4 Pose Landmarks to AR World Space

MediaPipe 3D pose landmarks are in **normalised image space**. To project them into ARCore's world coordinate space:

1. Obtain camera intrinsics via `ArCamera_getImageIntrinsics()` (`focal_length_x/y`, `principal_point_x/y`).
2. Use the depth value from ARCore's Depth API (Ch87 §6) at the landmark's `(x, y)` pixel position to reconstruct the 3D point: `X = (x - cx) * Z / fx`, `Y = (y - cy) * Z / fy`.
3. Transform from camera space to world space using `ArCamera_getViewMatrix()` inverse.

This produces world-anchored pose landmarks that remain stable as the device moves, enabling AR overlays such as skeleton visualisations anchored to a person walking through a scene.

---

## 10. Performance Profiling and Tuning

### 10.1 LiteRT Profiler

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

### 10.2 GPU Delegate Latency Breakdown

The GPU delegate's latency decomposes into:
- **CPU→GPU transfer**: Eliminated entirely with `AHardwareBuffer` binding (§4).
- **GLSL kernel compilation** (cold start only): Eliminated by shader serialisation (`serialization_dir`).
- **Kernel dispatch**: The main runtime cost; scales with model FLOPs and GPU frequency.
- **GPU→CPU readback**: Eliminated if the output tensor remains GPU-resident (e.g., feeding into a GLES overlay shader).

For a typical 256×256 pose landmark model on an Adreno 740 (Snapdragon 8 Gen 3), the fully warm path achieves ~2 ms inference latency, comfortably within a 33 ms frame budget at 30 fps.

### 10.3 Adreno and Mali GPU Monitoring

```bash
# Adreno GPU frequency and utilisation (Qualcomm devices)
adb shell cat /sys/class/kgsl/kgsl-3d0/gpuclk
adb shell cat /sys/class/kgsl/kgsl-3d0/gpu_busy_percentage

# Mali GPU utilisation (ARM devices)
adb shell cat /sys/devices/platform/mali.0/utilisation

# General GPU info via dumpsys
adb shell dumpsys gpu
```

### 10.4 Systrace / Perfetto End-to-End Tracing

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

### 10.5 Thermal Throttling

Sustained 30 fps inference at full GPU frequency triggers thermal limits within 3–5 minutes on most phones. Mitigation strategies:

- Register `PowerManager.OnThermalStatusChangedListener` and reduce inference resolution or frame rate when `THERMAL_STATUS_SEVERE` is reached.
- Switch from GPU delegate to NPU delegate (Qualcomm QNN) at elevated temperatures — NPUs are typically more power-efficient than the GPU for quantised INT8 models.
- Use `TFLITE_GPU_INFERENCE_PREFERENCE_FAST_SINGLE_ANSWER` instead of `SUSTAINED_SPEED` when thermal state is elevated; the GPU may lower frequency between frames.
- Enable adaptive frame skipping: skip inference on every other camera frame when `SystemClock.uptimeMillis()` shows the previous frame took more than 40 ms.

### 10.6 Quantisation Impact Summary

| Configuration | Typical Latency | Memory | Notes |
|---|---|---|---|
| Float32, CPU | 100–200 ms | 4× base | Baseline, no delegate |
| Float32, GPU delegate | 5–15 ms | 4× base | GLSL compute shaders |
| INT8, GPU delegate | 3–10 ms | 1× base | ~1.5–2× speedup |
| INT8, NNAPI/NPU (pre-API 35) | 2–8 ms | 1× base | 2–4× speedup vs GPU |
| INT8, Qualcomm QNN delegate | 1–5 ms | 1× base | Hexagon DSP, lowest power |

---

## 11. Roadmap

### 11.1 LiteRT-LM (Shipping, May 2026)

LiteRT-LM is the production orchestration layer for on-device LLM inference, superseding MediaPipe's `LlmInference` task. It achieves 52 tokens/second on Android GPU (OpenCL backend) and 56 tokens/second on Apple Metal, with multimodal support (text + vision + audio). Backends: CPU (XNNPack), GPU (OpenCL on Android, Metal on iOS, WebGPU in browser), NPU (Android only, via vendor delegates). [Source: LiteRT-LM announcement](https://developers.googleblog.com/blazing-fast-on-device-genai-with-litert-lm/)

KV-cache management stores Key and Value tensors in GPU high-bandwidth memory between decode steps, reducing per-token latency from O(context_length²) to O(context_length) via incremental attention computation.

### 11.2 Vulkan Compute Backend for GPU Delegate

The current GPU delegate defaults to OpenGL ES 3.1 compute shaders. The Vulkan compute backend (`TFLITE_GPU_EXPERIMENTAL_FLAGS_ENABLE_VULKAN_INFERENCE`) targets devices with Vulkan 1.1+. Advantages over GLES include lower driver overhead (no OpenGL state machine), explicit memory management (`VkDeviceMemory` from imported `AHardwareBuffer` via `VK_ANDROID_external_memory_android_hardware_buffer`), and access to Vulkan subgroup operations for more efficient SIMD reductions. Expect Vulkan backend to exit experimental status by late 2026.

### 11.3 LiteRT on Linux

The `ai-edge-litert` package is expanding to x86-64 and ARM64 Linux (targeting Raspberry Pi 5 and NVIDIA Jetson). The GPU delegate runs via OpenGL ES on Mesa (panfrost, v3d, freedreno). This is the convergence path between Android on-device ML and Linux embedded ML described in Ch88 and Ch124. [Source: LiteRT universal framework](https://developers.googleblog.com/litert-the-universal-framework-on-device-ai/)

### 11.4 WebAssembly / WebGPU

LiteRT-LM's Web SDK runs via WebGPU (WASM + JavaScript API) in Chrome and other browsers, with 2026 parity between mobile-native and browser inference performance on the same GPU. This connects to the WebGPU stack described in Ch35 (Dawn and WebGPU).

### 11.5 Vendor NPU Convergence

The post-NNAPI model (API 35+) pushes hardware acceleration into vendor-supplied LiteRT delegate libraries loaded at runtime. The long-term expectation is that all major Android SoC vendors (Qualcomm, MediaTek, Samsung Exynos, Google Tensor) ship production LiteRT delegates, converging on a uniform ABI where the delegate `.so` is the accelerator contract — analogous to how ICD `.so` files are the Vulkan driver contract.

### 11.6 On-Device Foundation Models

Vision-language models in the PaliGemma 3B class (3 billion parameters, vision + text) are feasible on 2026 flagship hardware at INT4 with LiteRT-LM. The GPU serves as the primary compute engine; the NPU handles INT8 attention layers. The camera→inference→display pipeline described in §8 of this chapter becomes the execution path for camera-grounded VLM queries ("What is this object?") running entirely on-device.

### 11.7 MediaPipe Tasks SDK Evolution

MediaPipe Tasks (§7) is transitioning from a community-maintained SDK to a Google-supported production API:
- **New modalities (2026–2027)**: Audio classification, face stylisation, and gesture customisation tasks are in preview. The Tasks API schema is being extended to support multi-modal tasks that accept both image and text inputs in a single `TaskRunner.process()` call, enabling on-device vision-language queries without the separate LiteRT-LM layer.
- **Stable ABI**: The current Tasks API targets Kotlin/Java and Python. A stable C API (`mediapipe/tasks/c/`) is under active development to allow C++ and Rust callers to consume Tasks results without going through the JNI boundary — eliminating the 0.5–1 ms JNI overhead on high-frequency (30 Hz) inference pipelines.
- **Calculator graph persistence**: The multi-calculator MediaPipe graph (§6.1) currently reconstructs packet routing on each `TaskRunner` call. A session-mode API keeps the `CalculatorGraph` resident in memory between frames, amortising graph construction cost across inference runs — critical for 60 Hz real-time use cases like AR overlay (§9).

### 11.8 Gemini Nano + LiteRT Convergence

Gemini Nano (the on-device Gemini variant) is currently deployed via AICore (`android.ai.app.aicore.AICore`) as a system-level service on Pixel 9+ devices. The convergence trajectory with LiteRT-LM:

- **Phase 1 (2026)**: Gemini Nano remains an AICore-brokered black box; apps call `GenerativeModel("gemini-nano")` via ML Kit GenAI API. LiteRT-LM serves open-weight models (Gemma 2B, PaliGemma). No shared inference path.
- **Phase 2 (2027+)**: Google is expected to expose Gemini Nano's internal transformer as a LiteRT-LM model, allowing apps to run the same Gemini model weights with the LiteRT-LM engine (rather than through AICore). This would allow fine-tuning, custom LoRA adapters, and private on-device inference without AICore telemetry.
- **Multimodal cache sharing**: LiteRT-LM and Gemini Nano both need KV-cache GPU memory. A shared KV-cache arbiter (analogous to how SurfaceFlinger arbitrates display memory) is the long-horizon solution for devices running multiple concurrent LLM sessions (e.g., ARCore scene description + background document assistant).

### 11.9 Competitive Landscape: GGML/llama.cpp vs LiteRT-LM

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

---

## 12. Integrations

**Ch85 — Android Compositor: SurfaceFlinger, HardwareBuffer, and the Buffer Pipeline**
`AHardwareBuffer` is the central memory object shared between the Camera HAL, the LiteRT GPU delegate, and SurfaceFlinger. The buffer lifecycle (allocate → acquire → GPU write → release → compositor acquire) described in Ch85 applies directly to the inference pipeline §§4 and 8.

**Ch86 — Vulkan on Android: Drivers, ANGLE, and Mobile GPU Performance**
The GPU delegate's Vulkan backend imports `AHardwareBuffer` via `VK_ANDROID_external_memory_android_hardware_buffer` (Ch86 §3). The same `VkSamplerYcbcrConversion` used for YUV camera textures in Ch86 is needed when feeding YCbCr camera frames to LiteRT vision models. ANGLE's GLES-on-Vulkan translation layer means that the GLES compute delegate runs on Vulkan on Chrome-backed Android targets.

**Ch87 — Android AR: ARCore Architecture, Camera HAL Integration, and the Android XR Platform**
§9 of this chapter describes the ARCore + MediaPipe composition pattern. ARCore provides world geometry; MediaPipe provides ML inference on the same camera texture. The shared GL context model, timestamp synchronisation (`ArFrame_getTimestamp()` → MediaPipe `Timestamp`), and the `GL_TEXTURE_EXTERNAL_OES` sharing pattern all build on the ARCore architecture in Ch87.

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
