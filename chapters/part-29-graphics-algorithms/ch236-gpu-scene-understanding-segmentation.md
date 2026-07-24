# Chapter 236: GPU 3D Scene Understanding and Semantic Segmentation (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers integrating semantic understanding into AR/XR pipelines; systems developers building GPU-accelerated vision workloads on Linux; robotics engineers consuming semantic maps from GPU inference and feeding them into planning and mapping systems.

---

## Table of Contents

1. [Scene Understanding as a GPU Algorithm Domain](#1-scene-understanding-as-a-gpu-algorithm-domain)
2. [2D Semantic Segmentation on GPU](#2-2d-semantic-segmentation-on-gpu)
3. [2D Instance and Panoptic Segmentation](#3-2d-instance-and-panoptic-segmentation)
4. [SAM — Segment Anything Model](#4-sam--segment-anything-model)
5. [3D Point Cloud Semantic Segmentation](#5-3d-point-cloud-semantic-segmentation)
6. [Open-Vocabulary 3D Scene Understanding](#6-open-vocabulary-3d-scene-understanding)
7. [GPU Occupancy Grids and Voxel Semantic Mapping](#7-gpu-occupancy-grids-and-voxel-semantic-mapping)
8. [3D Scene Graph Generation](#8-3d-scene-graph-generation)
9. [Neural Radiance Field Scene Understanding](#9-neural-radiance-field-scene-understanding)
10. [GPU Deployment and Runtime on Linux](#10-gpu-deployment-and-runtime-on-linux)
11. [Real-Time Scene Understanding Pipelines](#11-real-time-scene-understanding-pipelines)
12. [Integration with the Graphics Stack](#12-integration-with-the-graphics-stack)
13. [Integrations](#13-integrations)

---

## 1. Scene Understanding as a GPU Algorithm Domain

Scene understanding encompasses a family of perception tasks that transform raw sensor data — camera images, LiDAR point clouds, depth maps — into structured semantic representations: per-pixel class labels, object instance masks, occupancy grids, and relational scene graphs. These representations drive downstream applications in augmented reality (object occlusion and surface estimation), robotics (navigable-space identification, manipulation planning), and autonomous vehicles (driveable surface and pedestrian detection).

### 1.1 2D vs. 3D Understanding

**2D understanding** operates on camera images at pixel resolution. Semantic segmentation assigns each pixel a category label; instance segmentation additionally distinguishes individual object instances; panoptic segmentation unifies both into a single map covering every pixel. The primary GPU workloads are convolutional encoder–decoder forward passes plus attention mechanisms, which map cleanly to GEMM and grouped convolution kernels.

**3D understanding** ingests either point clouds (raw LiDAR returns or depth-unprojected RGB-D data) or volumetric voxel grids. The compute pattern differs significantly: point clouds are unstructured and sparse, requiring specialised indexing kernels (farthest-point sampling, radius search); voxel grids are structured but very large, requiring sparse convolution to avoid computing on empty space. 3D understanding feeds semantic labelling of the environment directly into occupancy maps and scene graphs used by path planners.

### 1.2 The GPU Inference Pipeline from Sensor to Semantic Label

A typical pipeline on Linux:

1. **Capture.** Camera frame via V4L2 (or depth frame via librealsense over USB3/MIPI). LiDAR scan via a UDP packet driver exposing raw PCL `PointCloud2` on ROS 2.
2. **Preprocessing.** Colour-space conversion, mean subtraction, normalisation — executed as GPU compute kernels to avoid a round-trip through host memory.
3. **Backbone inference.** Convolutional or transformer backbone extracts multi-scale feature maps.
4. **Segmentation head.** Pixel decoder + classifier head produces per-pixel logits.
5. **Argmax + remapping.** Parallel reduction across class channels to produce the label tensor.
6. **Semantic map update.** Labels merged into an occupancy or scene graph structure via GPU atomic operations.

The entire pipeline can be pipelined across GPU streams so preprocessing of frame N+1 overlaps with inference on frame N.

### 1.3 Linux GPU Deployment Stack

Three principal runtimes handle GPU scene-understanding inference on Linux:

| Runtime | GPU backend | Primary use |
|---|---|---|
| ONNX Runtime | MIGraphX EP (ROCm), CUDA EP, OpenVINO EP | Cross-framework model deployment |
| TensorRT | CUDA only | NVIDIA-specific optimised INT8/FP16 |
| OpenVINO | Intel GPU (Level Zero), CPU | Intel iGPU/dGPU edge inference |

For AMD hardware, the ONNX Runtime **MIGraphX Execution Provider** compiles ONNX operator graphs into MIGraphX's internal IR, applies operator fusion, and dispatches onto ROCm via HIP. [Source: ONNX Runtime MIGraphX EP docs, https://onnxruntime.ai/docs/execution-providers/MIGraphX-ExecutionProvider.html]

Vulkan compute is the cross-vendor fallback, used in frameworks such as `llama.cpp` (for ViT-based vision encoders in multimodal models) and emerging direct-inference shaders; it requires hand-authoring SPIR-V or GLSL compute shaders for attention and convolution.

---

## 2. 2D Semantic Segmentation on GPU

### 2.1 Encoder–Decoder Architecture: FCN and DeepLabV3+

Semantic segmentation maps pixels to class labels using an encoder that compresses spatial resolution while increasing channel depth (a CNN backbone such as ResNet or MobileNet), and a decoder that recovers full-resolution predictions.

**FCN (Fully Convolutional Networks)** replaced classification head dense layers with 1×1 convolutions, enabling arbitrary input resolution and producing spatial output via bilinear upsampling. The GPU memory layout is straightforward: each layer is an `NCHW` tensor of activations passed through `cudnnConvolutionForward` or `rocblas_gemm`-lowered im2col convolution.

**DeepLabV3+** introduces **Atrous Spatial Pyramid Pooling (ASPP)**: four parallel branches of dilated convolutions with rates {1, 6, 12, 18} plus an image-level global average pooling branch, all concatenated and projected. Dilated convolutions with rate `r` sample input at stride `r`, expanding the receptive field without increasing parameters. On GPU, dilated conv is implemented as a strided memory gather in the im2col step, with a stride equal to the dilation rate. The five ASPP branches are independent and can run concurrently on separate CUDA streams.

[Source: "Encoder-Decoder with Atrous Separable Convolution for Semantic Image Segmentation," Chen et al., ECCV 2018, https://arxiv.org/abs/1802.02611]

### 2.2 Mask2Former: Pixel Decoder and Masked Cross-Attention

Mask2Former (Cheng et al., CVPR 2022) unifies semantic, instance, and panoptic segmentation under a single architecture. [Source: "Masked-attention Mask Transformer for Universal Image Segmentation," Cheng et al., CVPR 2022, https://arxiv.org/abs/2112.01527]

**Pixel decoder** — a Feature Pyramid Network (FPN)-style multi-scale deformable attention module that takes the backbone's multi-scale feature maps {C₃, C₄, C₅} and produces per-pixel features at 1/4 resolution. Multi-scale deformable attention samples a small fixed number of keys (4–8) per query from all scale levels, making complexity O(N) in pixel count rather than O(N²).

**Transformer decoder** — 9 layers of cross-attention between N learnable mask queries and the pixel-level features, but with a **masked cross-attention** variant: each query attends only to pixels within its predicted mask region from the previous layer, preventing early-layer queries from attending to irrelevant background. The GPU implementation fuses the attention mask application into the softmax step via `additive masking` (−∞ for pixels outside the mask).

```python
# Masked cross-attention: apply binary mask before softmax
# attn_weights: [batch, num_heads, num_queries, H*W]
# mask: [batch, num_queries, H*W] — True where query cannot attend
attn_weights = attn_weights.masked_fill(mask.unsqueeze(1), float('-inf'))
attn_weights = torch.softmax(attn_weights, dim=-1)
```

The output N mask predictions are matched to ground truth via bipartite matching at training time; at inference time, predictions are thresholded and argmax-assigned to produce the final segmentation map.

### 2.3 Real-Time Segmentation: SegFormer-B0 and MobileViT

For embedded Linux (Jetson, RK3588, i.MX 8), throughput-constrained targets require lightweight models:

**SegFormer-B0** (Xie et al., NeurIPS 2021) uses a hierarchical Mix Transformer encoder with overlapping patch embeddings and a lightweight all-MLP decoder that concatenates multi-scale features and projects to class logits. The absence of positional encoding (replaced by zero-padded 3×3 depth-wise convolution) enables arbitrary input resolution without interpolating embeddings. SegFormer-B0 is designed for real-time inference on embedded targets; its low parameter count (~3.7M) and all-MLP decoder minimise memory bandwidth. [Source: "SegFormer: Simple and Efficient Design for Semantic Segmentation with Transformers," Xie et al., NeurIPS 2021, https://arxiv.org/abs/2105.15203]

**MobileViT** (Mehta & Rastegari, ICLR 2022) combines MobileNetV2 blocks with local ViT blocks that unfold the feature map into non-overlapping patches, apply global self-attention within each patch, and fold back. The attention cost is O(P²) where P is patch size (2×2 or 4×4), far below O(HW)² of full attention. [Source: "MobileViT: Light-weight, General-purpose, and Mobile-friendly Vision Transformer," Mehta & Rastegari, ICLR 2022, https://arxiv.org/abs/2110.02178]

ONNX export and deployment on MIGraphX:

```bash
# Export SegFormer-B0 from HuggingFace transformers to ONNX
python -c "
from transformers import SegformerForSemanticSegmentation
import torch
model = SegformerForSemanticSegmentation.from_pretrained('nvidia/segformer-b0-finetuned-ade-512-512')
model.eval()
dummy = torch.randn(1, 3, 512, 512)
torch.onnx.export(model, dummy, 'segformer_b0.onnx',
    opset_version=17, input_names=['pixel_values'],
    output_names=['logits'],
    dynamic_axes={'pixel_values': {0: 'batch', 2: 'height', 3: 'width'}})
"

# Run with ONNX Runtime MIGraphX EP on ROCm
python -c "
import onnxruntime as ort
sess = ort.InferenceSession('segformer_b0.onnx',
    providers=['MIGraphXExecutionProvider'])
"
```

---

## 3. 2D Instance and Panoptic Segmentation

### 3.1 Mask R-CNN GPU Inference Pipeline

Mask R-CNN (He et al., 2017) extends Faster R-CNN with a parallel mask prediction branch. The GPU inference pipeline: [Source: "Mask R-CNN," He et al., ICCV 2017, https://arxiv.org/abs/1703.06870]

1. **FPN feature extraction.** ResNet-50/101 backbone extracts {C₂…C₅}; FPN lateral connections produce {P₂…P₆} multi-scale feature maps. Each FPN level is a standard conv2d kernel dispatch.

2. **Region Proposal Network (RPN).** Slides a 3×3 conv window across each FPN level, generating objectness scores and box deltas for K anchors per location. The anchor generation is a GPU kernel that computes anchor coordinates from a stride and scale array — fully parallel across spatial positions.

3. **RoIAlign.** For each proposed region of interest, samples a fixed 7×7 (or 14×14 for masks) grid of bilinear-interpolated feature values from the corresponding FPN level. Bilinear interpolation at non-integer coordinates `(x, y)`:

   ```
   f(x,y) = f(⌊x⌋,⌊y⌋)·(1-dx)·(1-dy) + f(⌈x⌉,⌊y⌋)·dx·(1-dy)
           + f(⌊x⌋,⌈y⌋)·(1-dx)·dy   + f(⌈x⌉,⌈y⌉)·dx·dy
   ```

   Each RoI sample point is independent; the GPU kernel launches one thread per (RoI, grid point, channel).

4. **Box head and mask head.** Box head: 2 FC layers → class logits + box delta. Mask head: 4×conv2d + bilinear upsample → 28×28 binary mask per class. The mask head runs an FCN independently for each detected instance.

### 3.2 GPU Non-Maximum Suppression

NMS is the primary post-processing bottleneck. The naive O(N²) algorithm is amenable to GPU parallelism:

```cuda
// Parallel class-wise NMS — each thread checks one pair (i,j)
__global__ void nms_kernel(const float* boxes, const float* scores,
                           int* keep, int N, float iou_thresh) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    int j = blockIdx.y * blockDim.y + threadIdx.y;
    if (i >= N || j >= N || i <= j) return;
    // Write suppression bit into shared bitmask
    if (scores[i] < scores[j] && iou(boxes + i*4, boxes + j*4) > iou_thresh)
        atomicOr(&keep[i / 32], 1u << (i % 32));  // suppress box i
}
```

Batched NMS across all image classes is implemented in torchvision's `batched_nms` using a class-offset trick: shift boxes by `class_id * max_box_dim` so that inter-class IoU is always zero, then run single-class NMS. [Source: torchvision `ops/boxes.py`, https://github.com/pytorch/vision/blob/main/torchvision/ops/boxes.py]

### 3.3 RT-DETR Panoptic Head and DETR Bipartite Matching

**RT-DETR** (Zhao et al., CVPR 2024) is a real-time detection transformer that replaces the slow Hungarian matching of DETR with an IoU-based assignment during training, enabling >100 fps inference. [Source: "DETRs Beat YOLOs on Real-time Object Detection," Zhao et al., CVPR 2024, https://arxiv.org/abs/2304.08069]

For panoptic segmentation, RT-DETR's detection outputs (instance boxes + class scores) are fused with a lightweight pixel decoder producing per-pixel stuff (amorphous regions like sky and road) and thing (countable instances) predictions. **Panoptic fusion on GPU:**

1. Stuff predictions: argmax of per-pixel semantic logits for stuff classes.
2. Thing predictions: for each detected instance, paste its predicted mask into the panoptic canvas at the instance's location (overlap resolved by confidence ordering).
3. Merge: GPU kernel writes thing labels (instance_id × num_classes + class_id) into pixels, then fills remaining unlabelled pixels with stuff predictions.

The Hungarian algorithm for DETR bipartite matching (matching N predicted boxes to M ground truth during training) is not run at inference time — predictions are made for all N queries simultaneously and thresholded. At training time, GPU-accelerated optimal transport or the `scipy.optimize.linear_sum_assignment` CPU fallback handles the matching; the cost matrix is GPU-computed (class cost + L1 box cost + GIoU cost) and then transferred to CPU for the linear assignment.

---

## 4. SAM — Segment Anything Model

### 4.1 Architecture

SAM (Kirillov et al., 2023) decomposes image segmentation into three GPU-executed modules: [Source: "Segment Anything," Kirillov et al., ICCV 2023, https://arxiv.org/abs/2304.02643]

**Image encoder.** A Vision Transformer (ViT-H, ViT-L, or ViT-B) processes a 1024×1024 image. ViT-H uses 32 transformer blocks, 16 attention heads, and window-based local attention in most layers, with four global attention layers interspersed. The output is a 64×64 image embedding (downsampled 16× from input). The dominant GPU cost is GEMM for multi-head attention: ViT-H at batch 1 is compute-dominated by attention GEMM across all transformer blocks, making image encoding the most latency-critical step of the pipeline by a wide margin.

**Prompt encoder.** Encodes sparse prompts (clicked points and bounding boxes) via positional encoding, and dense prompts (mask inputs) via a convolutional encoding. Points are encoded as a sum of a learnable type embedding and a 2D Fourier positional encoding. This module is lightweight and runs in microseconds.

**Mask decoder.** A two-way transformer decoder with two layers. Query tokens include: N learnable output tokens (N=3 for the three predicted masks) plus the prompt token embeddings. Keys and values come from the image embedding augmented with per-position 2D positional encoding. The two-way design runs attention in both directions — output tokens attend to image features, and image features attend to output tokens — allowing prompt context to propagate back into the image feature space.

After the transformer, two prediction heads run in parallel:
- **Upscaling convolutional layers**: 2× transpose convolution + GELU + 2× transpose conv, producing a 256×256 mask feature map.
- **IoU prediction head**: MLP mapping each output token to a scalar quality score.

The final mask is produced by a dot product between the upscaled feature map and each output token embedding, followed by sigmoid.

### 4.2 SAM GPU Inference Path

```python
from segment_anything import sam_model_registry, SamPredictor
import torch

device = "cuda"  # or "rocm" if using ROCm-patched SAM
sam = sam_model_registry["vit_h"](checkpoint="sam_vit_h_4b8939.pth")
sam.to(device)
predictor = SamPredictor(sam)

# Image encoding is the expensive step (GEMM-heavy ViT forward pass, sub-second on a data-centre GPU)
predictor.set_image(image_rgb)

# Mask prediction from point prompt is cheap (millisecond-class)
masks, scores, logits = predictor.predict(
    point_coords=np.array([[500, 375]]),
    point_labels=np.array([1]),  # foreground
    multimask_output=True
)
```

[Source: segment-anything repository, https://github.com/facebookresearch/segment-anything]

### 4.3 SAM 2 — Video Segmentation

SAM 2 (Ravi et al., Meta AI, 2024) extends SAM to video with a streaming memory architecture. [Source: "SAM 2: Segment Anything in Images and Videos," Ravi et al., https://arxiv.org/abs/2408.00714]

The core additions over SAM:

- **Memory encoder**: a lightweight convolutional module that compresses the predicted mask and a downsampled image feature into a fixed-size memory tensor, appended to the memory bank.
- **Memory bank**: a FIFO of recent frame memories plus a set of user-prompted "object memories." Stored on GPU as a stack of feature tensors.
- **Memory attention**: before the image encoder's output enters the mask decoder, cross-attention is applied between the current frame's image features (queries) and the memory bank entries (keys and values). This propagates temporal context — enabling the model to track an object across frames without re-prompting.

For real-time performance, **MobileSAM** replaces ViT-H with a distilled TinyViT image encoder, dramatically reducing image encoding latency and making the model suitable for edge deployment. [Source: "Faster Segment Anything: Towards Lightweight SAM for Mobile Applications," Zhang et al., https://arxiv.org/abs/2306.14289]

**ROCm deployment note.** As of mid-2026, SAM and SAM 2 run via PyTorch's ROCm backend (`torch.device("cuda")` on a ROCm-configured install maps to HIP); full MIGraphX-compiled ONNX export requires operator coverage verification for the two-way transformer decoder. [Note: needs verification for SAM 2 ONNX MIGraphX end-to-end.]

---

## 5. 3D Point Cloud Semantic Segmentation

### 5.1 PointNet++ Set Abstraction on GPU

PointNet++ (Qi et al., NeurIPS 2017) processes unstructured point clouds through hierarchical **set abstraction** layers. [Source: "PointNet++: Deep Hierarchical Feature Learning on Point Sets in a Metric Space," Qi et al., NeurIPS 2017, https://arxiv.org/abs/1706.02413]

Each set abstraction layer performs three GPU operations:

**Farthest Point Sampling (FPS).** Selects M centroids from N input points such that each selected point maximises the minimum distance to all previously selected points. Implemented as M iterations of a parallel max-reduction: each iteration computes each point's distance to the current set of selected points (O(N × current_set_size)), then takes the argmax.

```cuda
// One FPS iteration: find point farthest from already-selected set
__global__ void fps_kernel(const float* points, const float* min_dists,
                           int* selected_idx, int N) {
    extern __shared__ float sdata[];
    int tid = threadIdx.x;
    int gid = blockIdx.x * blockDim.x + tid;
    // Load min distance for this point
    float val = (gid < N) ? min_dists[gid] : -1.f;
    sdata[tid] = val;
    // Parallel max reduction
    __syncthreads();
    for (int stride = blockDim.x/2; stride > 0; stride >>= 1) {
        if (tid < stride && sdata[tid+stride] > sdata[tid])
            sdata[tid] = sdata[tid+stride];
        __syncthreads();
    }
    if (tid == 0) atomicMax(selected_idx, /* recover argmax */0);
}
```

**Ball Query (Radius Search).** For each centroid, finds all points within radius `r`. The GPU kernel launches one thread-block per centroid; each thread checks one candidate point's Euclidean distance against `r`. Returns a fixed-size `[M, K]` neighbour index matrix (padded with the centroid index if fewer than K neighbours found).

**Grouping and mini-PointNet MLP.** Gathers the K neighbour features for each centroid into `[M, K, C]` tensor, subtracts centroid positions to get local coordinates, concatenates with features, applies shared MLP, and max-pools over K to get the output feature per centroid `[M, C']`.

### 5.2 RandLA-Net: Local Feature Aggregation

RandLA-Net (Hu et al., CVPR 2020) trades FPS (O(N²) iterations) for random point sampling (O(1)), and compensates with a richer **local feature aggregation** mechanism. [Source: "RandLA-Net: Efficient Semantic Segmentation of Large-Scale Point Clouds," Hu et al., CVPR 2020, https://arxiv.org/abs/1911.11236]

The aggregation block applies:

1. **Local Spatial Encoding (LocSE)**: for each point and its KNN neighbours, concatenates relative position, absolute position, Euclidean distance, and per-point feature into an augmented representation `[N, K, C_aug]`.
2. **Attentive Pooling**: learn a weight per (point, neighbour) pair via a shared MLP + softmax, weighted-sum over K neighbours. This is a K-way softmax attention with features, implemented as batched GEMM + softmax + weighted sum — the same structure as dot-product attention but over spatial neighbours rather than sequence positions.

For outdoor LiDAR scenes (e.g., SemanticKITTI: 100k–130k points per frame), random sampling enables RandLA-Net to process full scans at manageable GPU memory cost by using a 4× decimation at each stage.

### 5.3 Sparse Convolution for Voxelised Point Clouds

Voxelising a point cloud and applying 3D convolutions would waste computation on empty voxels (typically >98% of 3D space is empty for outdoor scenes). **Sparse convolution** computes outputs only at non-empty locations.

**MinkowskiEngine** (Choy et al., CVPR 2019) implements sparse convolutions using a `SparseTensor` that stores coordinates and features separately. [Source: "4D Spatio-Temporal ConvNets: Minkowski Convolutional Neural Networks," Choy et al., CVPR 2019, https://arxiv.org/abs/1904.08755; MinkowskiEngine, https://github.com/NVIDIA/MinkowskiEngine]

```python
import MinkowskiEngine as ME

# Create sparse tensor from point coordinates and features
coords = torch.IntTensor([[0, 0, 0, 0], [0, 1, 2, 3]])  # [batch, x, y, z]
feats  = torch.FloatTensor([[0.5, 0.3], [0.1, 0.8]])    # C-dim features
x = ME.SparseTensor(features=feats, coordinates=coords)

# Sparse 3D convolution
conv = ME.MinkowskiConvolution(in_channels=2, out_channels=64,
                                kernel_size=3, stride=1, dimension=3)
y = conv(x)  # output SparseTensor: only at active voxel locations
```

Internally, MinkowskiEngine maintains a **Coordinate Manager** that maps coordinate sets to GPU hash tables, enabling O(1) lookup of active voxel neighbours for kernel generation.

**spconv** (used in OpenPCDet) uses cuSPARSE-style sparse matrix–vector products to compute submanifold and regular sparse convolutions. [Source: spconv, https://github.com/traveller59/spconv] **TorchSparse** (MIT) implements hash-based sparse convolution with a fused gather-scatter kernel. [Source: TorchSparse, https://github.com/mit-han-lab/torchsparse]

**Cylinder3D** applies cylindrical voxelisation for outdoor LiDAR: rather than Cartesian (x,y,z) voxels, the point cloud is mapped to cylindrical coordinates (ρ,φ,z), producing more uniform voxel occupancy since LiDAR returns are denser near the sensor origin in Cartesian space. The sparse convolution then operates on the cylindrical grid. [Source: "Cylindrical and Asymmetrical 3D Convolution Networks for LiDAR Segmentation," Zhu et al., CVPR 2021, https://arxiv.org/abs/2011.10033]

---

## 6. Open-Vocabulary 3D Scene Understanding

### 6.1 CLIP Feature Distillation into 3D Gaussian Splatting

**Gaussian Grouping** (Ye et al., 2024) attaches identity features to each 3D Gaussian in a Gaussian Splatting scene. The pipeline fuses CLIP image features and SAM mask features: SAM tracks object masks across training views; CLIP encodes each masked region; the per-view CLIP embeddings are projected into a compact identity feature dimension and stored per Gaussian. [Source: "Gaussian Grouping: Segment and Edit Anything in 3D Scenes," Ye et al., https://arxiv.org/abs/2312.00732]

At inference, a language query is encoded by CLIP's text encoder; per-Gaussian identity features are compared by cosine similarity and differentiable rasterisation produces a relevancy heatmap over the scene.

**LangSplat** (Qin et al., 2024) embeds multi-scale CLIP language features into each Gaussian, then trains a per-scene autoencoder to compress the 512-dimensional CLIP features to a 3-dimensional latent code stored per Gaussian, reducing memory overhead by ~170×. Relevancy rendering renders the autoencoder's decoded features like colour channels through the standard alpha-compositing splatting formula. [Source: "LangSplat: 3D Language Gaussian Splatting," Qin et al., CVPR 2024, https://arxiv.org/abs/2312.16084]

### 6.2 LERF — Language Embedded Radiance Fields

LERF (Kerr et al., 2023) conditions a NeRF-style volume on language by training a secondary "language field" alongside the standard density+colour field. [Source: "LERF: Language Embedded Radiance Fields," Kerr et al., ICCV 2023, https://arxiv.org/abs/2303.09553]

During training, CLIP features are extracted at multiple scales (using different crop sizes of each training image) and supervised at each 3D point via volumetric rendering. The scale dimension allows the network to represent language at object, part, and material granularity.

At inference, a natural-language query is encoded by CLIP's text encoder to produce a language embedding `q`. For each pixel, the relevancy score is computed as the softmax-normalised dot product between the rendered language feature and `q` against a set of canonical phrase embeddings (used to normalise):

```
relevancy(p) = softmax([ CLIP_sim(f(p), q) / τ,
                          CLIP_sim(f(p), phrase_i) / τ, ... ])[0]
```

The GPU pipeline: NeRF ray marching → at each sample point, evaluate both colour MLP and language MLP → alpha-composite separately → get per-pixel relevancy heatmap. The language MLP evaluation is a small (3 hidden layers, 256 units) network evaluated in parallel for all sample points across all rays.

---

## 7. GPU Occupancy Grids and Voxel Semantic Mapping

### 7.1 3D Occupancy Prediction: TPVFormer and OccNet

**TPVFormer** (Huang et al., CVPR 2023) predicts 3D semantic occupancy from multi-camera images by lifting features to three orthogonal planes: XY (bird's-eye view), XZ (front view), YZ (side view). Cross-view attention exchanges information between planes. [Source: "Tri-Perspective View for Vision-Based 3D Semantic Occupancy Prediction," Huang et al., CVPR 2023, https://arxiv.org/abs/2302.07817]

Each plane is represented as a 2D feature grid (e.g., 200×200 for XY). Cross-plane attention is a standard multi-head attention with queries from one plane and keys/values from another. After cross-view attention, each plane feature is queried at 3D voxel locations via bilinear interpolation, and the three plane features are summed and fed into a lightweight MLP to predict per-voxel class logits.

**OccNet** (Tian et al., 2023) uses multi-scale image features from a transformer backbone and predicts occupancy via a voxel query-based cross-attention over the frustum volume. [Source: "OccNet: Scene as Occupancy," Tian et al., ICCV 2023, https://arxiv.org/abs/2306.02851]

### 7.2 GPU Voxel Grid Update: Log-Odds and Ray Casting

For sensor-fusion-based semantic mapping (when an inference network predicts per-frame occupancy rather than a trained 3D occupancy network), voxel grid states are updated incrementally using **log-odds**:

```
l(s | z₁:t) = l(s | z₁:t₋₁) + l(s | zₜ) − l(s)
```

where `l(s)` is the log-odds prior, `l(s|zₜ)` is the sensor log-odds update. GPU implementation: for each new point observation, compute the occupied voxel index and apply `atomicAdd` on the log-odds buffer.

**3D DDA ray casting** identifies all voxels a ray passes through before hitting an occupied voxel:

```cuda
__device__ void dda_traverse(float3 origin, float3 dir,
                              float3 voxel_size, int3 grid_dims,
                              int* voxel_ids, int* count) {
    // Initialise step directions and tMax per axis
    float3 tDelta = voxel_size / fabs(dir);
    float3 tMax;
    int3 step, voxel = world_to_voxel(origin, voxel_size);
    // ... standard DDA traversal, writing voxel indices until hit
}
```

Each ray is processed by one GPU thread; for a depth image of 640×480, 307k rays can be cast in parallel.

### 7.3 Semantic Octree Operations

OctoMap (Hornung et al., 2013) represents the environment as a probabilistic octree of occupied/free voxels. GPU-accelerated variants maintain the octree nodes in a flat GPU buffer, using a Morton (Z-order) code as the key for spatial indexing. Semantic labels can be stored per leaf node as a class probability vector. [Note: GPU-accelerated OctoMap is a research area; production octree traversal on GPU requires careful handling of divergent threads; needs verification for specific library implementations.]

---

## 8. 3D Scene Graph Generation

### 8.1 Object-Level Node Extraction

3D scene graph nodes correspond to individual object instances. Starting from a 3D instance segmentation result (e.g., from PointNet++ with an instance head or from a lifted 2D instance mask via depth unprojection):

1. **Point cluster isolation**: GPU scatter operation maps each point to its instance ID, grouping points into per-instance buffers.
2. **Oriented Bounding Box (OBB) fitting via PCA**: for each cluster, compute the 3×3 covariance matrix on GPU (parallel reduce over points), then find the dominant eigenvectors. The OBB axes are the three eigenvectors; extents are the max projections along each axis.

```python
# GPU PCA for OBB fitting (per instance, batched)
# points: [B, N, 3]
center = points.mean(dim=1, keepdim=True)
centered = points - center
cov = torch.bmm(centered.transpose(1, 2), centered) / N  # [B, 3, 3]
eigvals, eigvecs = torch.linalg.eigh(cov)  # sorted ascending
# eigvecs columns are OBB axes; extents from projection range
```

### 8.2 Relation Prediction via GPU GNN Inference

Scene graph edges encode spatial relations (on, in, next-to, supported-by) and semantic relations (is-a, part-of). A **Graph Neural Network** (GNN) trained on 3D scene graph datasets (3RScan, ScanScribe) predicts edge labels.

GNN inference on GPU: the scene graph has O(N²) candidate edges for N objects. Edge features are computed from relative pose (translation vector, relative orientation from OBB axes), contact area (IOU of base planes for "on" relations), and appearance feature similarity. Edge feature computation is parallelisable over all (i,j) pairs.

Each GNN layer:
1. **Message passing**: for each node, aggregate incoming edge features via a learned MLP + pooling.
2. **Node update**: apply MLP to aggregated messages + current node feature.

This maps to batched GEMMs across nodes and edges. For Ch228's sparse graph GEMM formulation applied to scene graphs, see the integration note below.

### 8.3 Spatial Relation Types from Geometric Heuristics

Before GNN inference, geometric heuristics on GPU precompute candidate relation types to prune edge search:

- **"on"**: centroid of A is above centroid of B (Δz > 0) AND base plane of A intersects top plane of B.
- **"next-to"**: horizontal distance between centroids < threshold AND vertical overlap > threshold.
- **"supported-by"**: static contact between A and B — requires A's base face vertices to be within ε of B's top face.

These tests are GPU-parallelisable as pairwise distance and intersection tests across all object pairs.

---

## 9. Neural Radiance Field Scene Understanding

### 9.1 Semantic NeRF

Semantic NeRF (Zhi et al., ICCV 2021) extends the NeRF MLP with a semantic prediction head: alongside colour (r,g,b) and density (σ), the network outputs a C-dimensional logit vector at each sample point along each ray. [Source: "In-Place Scene Labelling and Understanding with Implicit Scene Representation," Zhi et al., ICCV 2021, https://arxiv.org/abs/2103.15875]

The semantic output is alpha-composited along the ray identically to colour:

```
S(r) = Σᵢ Tᵢ αᵢ sᵢ
```

where `sᵢ` is the semantic logit vector at sample `i`, and Tᵢ, αᵢ are the standard transmittance and occupancy weights. The rendered semantic vector `S(r)` is supervised with cross-entropy against 2D semantic labels (which can be sourced from a 2D segmentation network rather than manual annotation).

GPU inference: the only addition over standard NeRF is the extra C-output head in the MLP evaluation kernel. For C=40 classes and 64 sample points per ray at 640×480 resolution, this adds roughly 40×64×307k = ~800M multiply-adds per frame — a ~5% overhead over colour NeRF.

### 9.2 3D Feature Field Distillation

**N3F (Neural Feature Fusion Fields)**, Tschernezki et al., 2022, distils DINO self-supervised features (rather than CLIP language features) into a NeRF field, enabling part-level semantic clustering without class labels. [Source: "Neural Feature Fusion Fields: 3D Distillation of Self-Supervised 2D Image Representations," Tschernezki et al., 3DV 2022, https://arxiv.org/abs/2209.03494]

The distillation loss supervises the feature field to match the 2D DINO features when rendered; at inference, PCA or k-means clustering of the rendered feature field reveals semantically coherent 3D regions.

### 9.3 Semantic 3D Gaussian Splatting

Per-Gaussian semantic labels can be rendered identically to colour. Each Gaussian carries either a one-hot label vector (for closed-set segmentation) or a compact feature vector (for open-vocabulary, as in §6). The **semantic rendering pass** computes:

```
s(p) = Σᵢ sᵢ αᵢ ∏ⱼ<ᵢ (1 − αⱼ)
```

with the same alpha-compositing GPU kernel used for colour — just replacing the `[R,G,B]` output with an `[C]` output. Because the Gaussian splat is differentiable with respect to Gaussian positions, features, and opacities, semantic labels can be fine-tuned end-to-end.

---

## 10. GPU Deployment and Runtime on Linux

### 10.1 ONNX Runtime and Execution Providers

The **ONNX Runtime MIGraphX Execution Provider** compiles ONNX graphs to MIGraphX IR, performs operator fusion (Conv+BN+ReLU, Attention+softmax), and executes on AMD GPUs via HIP. Model export:

```python
# Export Mask2Former to ONNX for MIGraphX EP deployment
import torch
from transformers import Mask2FormerForUniversalSegmentation

model = Mask2FormerForUniversalSegmentation.from_pretrained(
    "facebook/mask2former-swin-large-coco-panoptic")
model.eval()

torch.onnx.export(
    model,
    torch.randn(1, 3, 800, 1333),
    "mask2former.onnx",
    opset_version=17,
    input_names=["pixel_values"],
    output_names=["masks_queries_logits", "class_queries_logits"]
)
```

Runtime session with MIGraphX:

```python
import onnxruntime as ort
sess_options = ort.SessionOptions()
sess_options.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL
session = ort.InferenceSession(
    "mask2former.onnx",
    sess_options=sess_options,
    providers=["MIGraphXExecutionProvider", "CPUExecutionProvider"]
)
```

[Source: https://onnxruntime.ai/docs/execution-providers/MIGraphX-ExecutionProvider.html]

### 10.2 Memory Layout and DMA-BUF Zero-Copy

For camera-to-inference pipelines on Linux, **DMA-BUF** eliminates the host-memory round-trip. A V4L2 capture buffer allocated with `V4L2_MEMORY_MMAP` can be exported as a DMA-BUF file descriptor. The GPU driver's import path (ROCm's `hipImportExternalMemory`, CUDA's `cuImportExternalMemory`) maps the DMA-BUF into GPU virtual address space, making the camera frame directly accessible as a GPU tensor with zero host copies.

The preprocessing kernel (colour conversion, normalisation) reads from the DMA-BUF-backed input and writes into the GPU tensor that feeds the segmentation model's first layer. This eliminates one `cudaMemcpy` / `hipMemcpy` call per frame — at 4K30, a 4096×2160 NV12 frame is ~13 MB; zero-copy saves ~390 MB/s of PCIe bandwidth compared to a staged host-copy approach.

For runtimes that do not natively support DMA-BUF import, the V4L2 buffer must be memcpy'd through host memory. GStreamer's `v4l2src ! video/x-raw ! appsink` pipeline handles this automatically; for zero-copy on Intel, GStreamer's `vaapisink` and `vaapidecodebin` use DRM PRIME DMA-BUF handles throughout.

### 10.3 Cross-Platform Inference: OpenVINO and Vulkan

**OpenVINO** (Intel) targets Intel iGPU (Xe graphics) and dGPU (Arc) via the Level Zero backend, converting ONNX models to OpenVINO IR (XML + binary weights) with `mo` or the new 2.x `openvino.convert_model` API. INT8 quantisation via the Neural Network Compression Framework reduces SegFormer-B0 from FP32 to INT8 at <1% mAP loss on ADE20k. [Source: OpenVINO Model Optimization Guide, https://docs.openvino.ai/2024/openvino-workflow/model-optimization-guide.html]

For cross-vendor inference where neither CUDA nor ROCm is available, **Vulkan compute** runs segmentation via custom SPIR-V shaders. The architecture matches the `llama.cpp` Vulkan inference pattern (Ch124): each operator (GEMM, softmax, conv) is a separate GLSL compute shader compiled to SPIR-V at first run and cached. For deep segmentation models, the shader dispatch overhead makes Vulkan competitive with CPU-based ONNX Runtime for batch-1 inference on small GPUs.

---

## 11. Real-Time Scene Understanding Pipelines

### 11.1 Latency Budget and Pipeline Parallelism

A real-time AR semantic segmentation pipeline targeting 30 Hz must complete a full camera-to-semantic-map cycle in 33ms. A typical budget:

| Stage | Time (GPU) |
|---|---|
| V4L2 capture + DMA-BUF map | 2–4 ms |
| Colour convert + normalise (GPU kernel) | 0.5 ms |
| SegFormer-B0 encoder forward pass | 8–12 ms |
| Decoder + argmax | 1–2 ms |
| Semantic map update (GPU atomic) | 0.5 ms |
| Total | 12–19 ms |

With double-buffering: while the GPU processes frame N, frame N+1 is captured into the other DMA-BUF. CUDA/HIP stream priorities ensure the segmentation stream does not contend with the compositor's render stream.

### 11.2 GStreamer Pipeline Integration

```bash
# GStreamer pipeline: capture → GPU inference → semantic overlay → display
gst-launch-1.0 \
    v4l2src device=/dev/video0 ! \
    video/x-raw,format=NV12,width=1280,height=720,framerate=30/1 ! \
    videoconvert ! video/x-raw,format=RGB ! \
    appsink name=inference_sink sync=false drop=true max-buffers=1
```

The Python/C++ inference process pulls frames from `inference_sink`, runs the segmentation model, and pushes semantic overlay frames into an `appsrc` feeding the compositor. For low-latency operation, the `drop=true max-buffers=1` policy discards frames that arrive while inference is running, capping latency at one frame period.

### 11.3 ROS 2 GPU Tensor Topics

In a ROS 2 robotics pipeline, GPU tensor zero-copy avoids expensive serialisation. The `rmw_cyclonedds` middleware with CUDA-aware MPI supports GPU-resident topic payloads where the data pointer addresses GPU memory. A semantic segmentation node receives a `sensor_msgs/Image` from an upstream camera node, processes it on GPU, and publishes a custom `SemanticImage` message whose data buffer is a CUDA/HIP device pointer. Downstream consumers (a planner, an occupancy grid updater) read from GPU memory directly.

[Note: Zero-copy GPU tensor topics in ROS 2 depend on the DDS middleware and are under active development as of mid-2026; verify RMW implementation before deploying. See Ch211 for the ROS 2 GPU compute integration picture.]

---

## 12. Integration with the Graphics Stack

### 12.1 Semantic Mask Driving GPU-Driven Indirect Draw

When scene understanding is combined with rendering, semantic labels can filter draw calls on GPU without CPU involvement. In a **GPU-driven indirect draw** pipeline (Ch208), each mesh draw call's `VkDrawIndirectCommand` is pre-populated by a compute shader. A semantic predicate — "draw object only if its semantic class is 'foreground'" — can be evaluated on GPU:

```glsl
// compute shader: populate indirect draw buffer based on semantic class
layout(local_size_x = 64) in;
layout(set=0, binding=0) buffer DrawCmds { VkDrawIndirectCommand cmds[]; };
layout(set=0, binding=1) buffer SemanticLabels { uint labels[]; };

void main() {
    uint id = gl_GlobalInvocationID.x;
    // Suppress draw call if semantic class is background (0)
    if (labels[id] == 0u)
        cmds[id].instanceCount = 0u;
}
```

This enables semantic-aware occlusion culling, semantic-specific level-of-detail selection, and material override from semantic class — all without a GPU–CPU round-trip.

### 12.2 Semantic Segmentation Output to Compositor Overlay

The semantic label image produced by the inference pipeline is a `VkImage` in GPU memory. To overlay it on a Wayland compositor:

1. Bind the `VkImage` as a texture in a GLSL fragment shader that maps class IDs to colours and alpha values.
2. For AR overlays, use the Monado OpenXR compositor (Ch27): import the `VkImage` as a `XrSwapchainImage` via `XR_KHR_vulkan_enable2`, blending the semantic overlay into the layer stack during projection layer submission.

Semantic colours can be rendered with per-class transparency — for instance, rendering "road" class at 30% alpha for an AR navigation overlay while rendering "obstacle" at 80% to signal hazards.

### 12.3 Semantic LOD and Material Assignment

**Semantic LOD** selects mesh detail level based on semantic class importance rather than solely on screen-space area. A semantic importance table maps class IDs to LOD bias values: background classes (sky, vegetation) receive a positive LOD bias (lower resolution), while safety-critical foreground classes (pedestrians, vehicles) receive a negative bias (higher resolution). The LOD bias is written by the same compute shader that evaluates the semantic predicate, feeding directly into the draw command's `firstIndex` offset selecting the LOD mesh.

**Per-object material assignment** from semantic class: the semantic label is sampled at the object's surface UV in the material shader, looked up in a small `ubo` table of per-class material parameters, and blended with the base material. This enables real-time stylisation driven by scene understanding (e.g., X-ray style, infrared palette) without CPU involvement.

---

## 13. Integrations

**Ch27 (VR & AR).** The Monado OpenXR runtime consumes the semantic `VkImage` as a compositor layer; semantic understanding feeds surface tracking, occlusion geometry estimation, and anchor placement. Semantic LOD (§12.3) directly reduces GPU load in Monado's render thread.

**Ch224 (3D Shape Analysis Algorithms).** PointNet++ set abstraction (§5.1) and the FPS/ball-query operators are shared between shape segmentation (Ch224) and scene-level point cloud segmentation. Object-level OBB fitting via PCA (§8.1) uses the GPU covariance and eigen-decomposition covered in Ch224 §2.

**Ch228 (GPU Graph Algorithms).** Scene graph GNN inference (§8.2) uses sparse graph GEMM and message-passing primitives. The SpMV and graph BFS implementations from Ch228 apply directly to scene graph propagation and transitive relation closure.

**Ch229 (GPU ML Inference Algorithms).** The transformer attention GEMMs underlying ViT image encoders (SAM, SegFormer), the quantisation and kernel-fusion strategies, and ONNX Runtime execution provider dispatch are covered algorithmically in Ch229. This chapter applies them to the scene-understanding domain.

**Ch232 (GPU Generative AI and LLM Inference on Linux).** Open-vocabulary scene understanding via LangSplat, LERF, and Gaussian Grouping relies on CLIP embeddings (a vision transformer) and natural-language query encoding. VLM-based scene description (GPT-4V, LLaVA-style models) is discussed in Ch232's multimodal section. The VLM produces scene-level semantic annotations that seed the per-object features in the scene graph.

**Ch238 (GPU Object Detection and 6DoF Pose Estimation).** Object detection provides the instance bounding boxes that seed Mask R-CNN's RoIAlign and RT-DETR's panoptic head (§3). Detected 6DoF object poses feed the OBB fitting and relation computation in scene graph construction (§8). 3D segmentation and detection pipelines share backbone FPN features in many production architectures.
