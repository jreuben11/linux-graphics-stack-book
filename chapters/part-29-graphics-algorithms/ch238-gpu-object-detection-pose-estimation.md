# Chapter 238: GPU Object Detection and 6DoF Pose Estimation (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers building AR object anchoring or robotic manipulation pipelines; systems developers deploying detection and pose inference on Linux with ROCm/Vulkan/ONNX Runtime; engineers integrating detection into real-time GPU rendering pipelines.

This chapter covers the complete detection-to-pose pipeline as a set of GPU algorithms: from 2D bounding-box detection, through 3D lifting and RGB-D fusion, to 6DoF pose estimation, feature matching, RANSAC-based robust estimation, and pose tracking. The production-deployment section covers ONNX Runtime ROCm EP, TensorRT, OpenVINO, and Triton Inference Server on Linux. The final section shows how to route detection and pose output back into a Vulkan rendering pipeline and Wayland compositor for AR overlays.

---

## Table of Contents

1. [Detection and Pose as a GPU Algorithm Domain](#1-detection-and-pose-as-a-gpu-algorithm-domain)
2. [2D Object Detection GPU Inference](#2-2d-object-detection-gpu-inference)
3. [RT-DETR and Transformer-Based Detection](#3-rt-detr-and-transformer-based-detection)
4. [3D Detection from LiDAR Point Clouds](#4-3d-detection-from-lidar-point-clouds)
5. [RGB-D and Multimodal 3D Detection](#5-rgb-d-and-multimodal-3d-detection)
6. [6DoF Pose Estimation — Keypoint-Based](#6-6dof-pose-estimation--keypoint-based)
7. [Direct and Dense 6DoF Pose Estimation](#7-direct-and-dense-6dof-pose-estimation)
8. [GPU Feature Matching for Pose](#8-gpu-feature-matching-for-pose)
9. [GPU RANSAC and Robust Estimation](#9-gpu-ransac-and-robust-estimation)
10. [Pose Tracking and Temporal Refinement](#10-pose-tracking-and-temporal-refinement)
11. [Production Deployment on Linux](#11-production-deployment-on-linux)
12. [Integration with the Graphics Pipeline](#12-integration-with-the-graphics-pipeline)
- [Integrations](#integrations)

---

## 1. Detection and Pose as a GPU Algorithm Domain

### 1.1 The Pipeline

Object detection and 6DoF pose estimation are typically composed into a three-stage pipeline:

```
RGB / RGB-D / LiDAR frame
          │
          ▼
Stage 1 — 2D or 3D Detection
  Locates objects; produces bounding boxes or 3D proposals
          │
          ▼
Stage 2 — 3D Lifting / Multimodal Fusion
  From 2D boxes + depth, or LiDAR, produces oriented 3D boxes
          │
          ▼
Stage 3 — 6DoF Pose Estimation + Refinement
  Produces full rotation R ∈ SO(3) and translation t ∈ ℝ³
          │
          ▼
  Pose matrix → AR rendering / robot actuator
```

Each stage is GPU-accelerated. Stage 1 is dominated by convolutional or transformer inference. Stage 2 requires specialised GPU kernels for voxelisation or depth unprojection. Stage 3 combines neural inference with classical solvers (PnP, RANSAC) implemented on the GPU.

### 1.2 Sensor Modalities

Three sensor families are used in AR and robotics:

- **RGB cameras.** Cheapest, highest resolution. Depth must be inferred. Pose estimation requires known or estimated intrinsics.
- **RGB-D cameras** (Intel RealSense, Microsoft Azure Kinect, structured-light). Per-pixel depth at VGA–1080p resolution. Depth quality degrades outdoors and on glossy surfaces.
- **LiDAR.** Sparse 3D point cloud (32–128 beam rotating or solid-state arrays). Accurate metric depth, no colour. Dominant in autonomous driving; increasingly common in mobile robotics via lower-cost solid-state variants.

Multimodal pipelines fuse two or more of these, requiring calibrated extrinsics between sensors.

### 1.3 Inference Latency Budget for Real-Time AR

A 60 Hz AR frame gives 16.7 ms per frame end-to-end. A practical budget for detection+pose in a mobile AR headset running at 60 Hz:

| Stage | Budget |
|---|---|
| Detection (GPU inference) | 4–8 ms |
| Depth preprocessing / voxelisation | 1–2 ms |
| Pose estimation (keypoints + PnP or dense) | 3–6 ms |
| Pose refinement (ICP or render-compare) | 2–4 ms |
| **Total** | **10–20 ms** (must run concurrently with render) |

Parallel execution across GPU queues (async compute) is essential: detection inference runs on the compute queue while the render queue draws the previous frame.

### 1.4 Linux GPU Deployment Stack

On Linux, inference deployments for this pipeline are built on:

- **ROCm HIP** (AMD) — direct GPU compute with HIP kernels for custom operators.
- **CUDA** (NVIDIA) — cuDNN for convolutions, cuBLAS for GEMM, NCCL for multi-GPU.
- **ONNX Runtime** — cross-GPU inference via ROCm EP (MIGraphX) or CUDA EP.
- **TensorRT** — NVIDIA-only, highest-throughput quantised inference.
- **OpenVINO** — Intel GPU (Arc/Xe) via Level Zero or OpenCL backend.
- **Vulkan compute** — vendor-neutral via GLSL compute shaders or SPIR-V; used by llama.cpp and emerging inference stacks.

---

## 2. 2D Object Detection GPU Inference

### 2.1 The YOLO Family Architecture

The YOLO (You Only Look Once) family dominates real-time 2D detection. All versions since YOLOv8 (2023) are **anchor-free** and use a **decoupled detection head** that separates classification and regression branches.

**YOLOv8** (Ultralytics, 2023) [Source: Ultralytics YOLOv8, https://github.com/ultralytics/ultralytics] introduced the C2f (Cross-Stage Partial Bottleneck with two convolutions and feature fusion) backbone block. The neck is a PAN (Path Aggregation Network) FPN. The decoupled head uses separate branches: `cls_head` (classification) and `reg_head` (box regression via distribution focal loss over a discrete distance grid).

**YOLOv9** (Wang et al., 2024) [Source: arXiv:2402.13616] introduced **Programmable Gradient Information (PGI)** — an auxiliary reversible branch that ensures the gradient signal reaching early backbone layers is not diluted by the forward-pass depth — and **GELAN** (Generalised Efficient Layer Aggregation Network) backbone blocks that extend C2f with multi-path aggregation.

**YOLOv10** (Wang et al., 2024) [Source: arXiv:2405.14458] eliminates NMS at inference by using a **dual-assignment** strategy during training: one-to-many matching builds rich gradient signal; one-to-one matching is used to select the single final prediction per object. At inference only the one-to-one head is active, removing NMS entirely.

**YOLO11** (Ultralytics, 2024) [Source: https://docs.ultralytics.com/models/yolo11/] refines the backbone with **C3k2** blocks (a two-kernel variant of C3 with deeper feature reuse) and **C2PSA** (Cross-Stage Partial with Positional Spatial Attention), further improving small-object recall.

### 2.2 GPU Inference: Backbone, FPN, and Head

On the GPU, YOLO inference decomposes into three stages:

**Backbone feature extraction.** C2f blocks are composed of depthwise-separable Conv2D, BatchNorm, SiLU activation, and skip connections. On a modern AMD RDNA3 GPU, a YOLOv8-m backbone processing a 640×640 image runs in ~2 ms using MIGraphX-compiled operators via the ONNX Runtime ROCm EP.

**FPN neck.** The feature pyramid upsample-and-concatenate pattern involves bilinear upsampling followed by channel-wise concatenation and 1×1 convolution. The GPU executes these as fused kernels; ONNX Runtime's graph optimizer fuses the `Resize + Concat + Conv` pattern into a single kernel invocation where the operator fuser recognises the adjacent nodes.

**Detection head.** The decoupled head runs two branches in parallel (achievable with `vkCmdDispatch` into independent workgroups, or with two concurrent CUDA streams). The classification branch produces a `[N, num_classes, H, W]` logit tensor; the regression branch produces a `[N, 4*reg_bins, H, W]` distribution. DFL (Distribution Focal Loss) decoding collapses the distribution to 4 box coordinates via `softmax + convolution with position weights`.

### 2.3 GPU NMS Algorithms

Non-Maximum Suppression filters overlapping detections. Three GPU strategies are used:

**Class-wise parallel NMS via reduction.** Each class is processed independently on separate workgroups. Within a class, detections are sorted by score (GPU radix sort). Each surviving detection is checked against all lower-scoring boxes; if IoU exceeds a threshold, the lower box is suppressed. This reduces to a parallel upper-triangular evaluation:

```glsl
// GLSL compute shader sketch — parallel IoU suppression
layout(local_size_x = 256) in;
layout(binding = 0) readonly buffer Boxes { vec4 boxes[]; };      // [x1,y1,x2,y2]
layout(binding = 1) readonly buffer Scores { float scores[]; };
layout(binding = 2) writeonly buffer Keep { int keep[]; };

shared int suppress[256];

void main() {
    uint i = gl_GlobalInvocationID.x;
    suppress[gl_LocalInvocationID.x] = 0;
    barrier();

    for (uint j = 0; j < i; ++j) {
        if (keep[j] == 0) continue;
        float iou = compute_iou(boxes[j], boxes[i]);
        if (iou > NMS_THRESHOLD) {
            suppress[gl_LocalInvocationID.x] = 1;
            break;
        }
    }
    barrier();
    keep[i] = suppress[gl_LocalInvocationID.x] == 0 ? 1 : 0;
}
```

**Batched NMS (torchvision GPU NMS).** TorchVision implements `torchvision.ops.nms` as a CUDA kernel that operates on the full detection set. Internally it uses a bitmask approach: an `N×N` IoU matrix is materialised in shared memory for small N (<1024 detections), and a merge-scan loop eliminates suppressed boxes. [Source: TorchVision source, https://github.com/pytorch/vision/blob/main/torchvision/csrc/ops/cuda/nms_kernel.cu]

**Soft-NMS.** Rather than hard suppression, soft-NMS decays the score of overlapping boxes by a Gaussian: `s_i ← s_i · exp(-IoU(m, b_i)² / σ)`. This is an elementwise operation per surviving box — fully parallelisable per detection.

### 2.4 Deployment

To deploy YOLOv8 via ONNX Runtime with the ROCm execution provider:

```python
import onnxruntime as ort
import numpy as np

# Export from PyTorch
from ultralytics import YOLO
model = YOLO("yolov8m.pt")
model.export(format="onnx", dynamic=True, opset=17)

# Inference via ROCm EP
providers = [("ROCMExecutionProvider", {"device_id": 0})]
sess = ort.InferenceSession("yolov8m.onnx", providers=providers)

img = np.random.rand(1, 3, 640, 640).astype(np.float32)  # NCHW
outputs = sess.run(None, {"images": img})
# outputs[0]: shape [1, 84, 8400] — 84 = 4 box coords + 80 classes
```

[Source: ONNX Runtime ROCm EP, https://onnxruntime.ai/docs/execution-providers/ROCm-ExecutionProvider.html]

For TensorRT INT8 calibration on NVIDIA, a calibration dataset of ~500 representative images is used with the `IInt8EntropyCalibrator2` interface. The calibrated engine reduces YOLOv8-m inference from ~3.5 ms (FP16) to ~2.1 ms (INT8) on an RTX 4090.

---

## 3. RT-DETR and Transformer-Based Detection

### 3.1 RT-DETR Architecture

RT-DETR (Real-Time Detection Transformer, Zhao et al., 2023) [Source: arXiv:2304.08069] is the first real-time end-to-end transformer detector competitive with YOLO on speed while surpassing it on accuracy. The three-stage architecture:

1. **HGNetv2 backbone** (High-Performance GPU Network v2, developed by Baidu's PaddleDetection team). A ConvNext-inspired CNN backbone with large-kernel depthwise convolutions and LayerScale initialisation. Outputs multi-scale feature maps at `{C3, C4, C5}` strides.

2. **Hybrid encoder.** Two phases applied sequentially:
   - *Intra-scale feature interaction (AIFI)*: a single-scale transformer applied at `C5` only (the highest-level, lowest-resolution features), capturing global context via multi-head self-attention.
   - *Cross-scale feature fusion (CCFF)*: a CNN-based module that fuses C3/C4/C5 features via repeated 1×1 convolutions and element-wise add, avoiding the computational cost of cross-scale transformer attention.

3. **RT-DETR decoder.** A stack of 6 transformer decoder layers with **deformable cross-attention**: each query attends to a small number of sampling points on the encoder feature map rather than to all spatial positions. See §3.2.

### 3.2 GPU Deformable Attention

Deformable attention [Source: Deformable DETR, arXiv:2010.04159] computes attention over K sampling points per query head rather than over all spatial positions. For query feature `q`, reference point `p`, the output is:

```
DeformAttn(q, p, x) = Σ_{m=1}^{M} W_m · Σ_{k=1}^{K} A_{mk} · x(p + Δp_{mk})
```

where `Δp_{mk}` are learned spatial offsets for head `m`, point `k`, and `A_{mk}` are attention weights (sum to 1 across k).

**GPU implementation.** Each query thread computes its K offsets, performs bilinear interpolation at each sampled location in the feature map, and accumulates the weighted sum. The bilinear interpolation at fractional pixel coordinates is a 4-tap gather:

```cpp
// Bilinear interpolation at fractional feature map coordinate (fx, fy)
// Feature map shape: [H, W, C], C channels per spatial location
__device__ float bilinear_sample(
    const float* feat, int H, int W, int C, int c,
    float fx, float fy)
{
    int x0 = (int)fx, y0 = (int)fy;
    int x1 = min(x0 + 1, W - 1), y1 = min(y0 + 1, H - 1);
    float wx = fx - x0, wy = fy - y0;
    return (1-wy)*((1-wx)*feat[(y0*W+x0)*C+c] + wx*feat[(y0*W+x1)*C+c])
         +    wy *((1-wx)*feat[(y1*W+x0)*C+c] + wx*feat[(y1*W+x1)*C+c]);
}
```

With K=4 sampling points and M=8 heads, each query performs 32 bilinear interpolations — a compute-to-memory ratio that maps well onto GPU shared memory preloading.

### 3.3 End-to-End Detection Without NMS

RT-DETR uses set prediction with Hungarian matching during training (assigning each ground-truth box to one decoder query) and outputs one prediction per query at inference. Since each object is assigned to exactly one query, NMS is unnecessary. This reduces inference latency by ~1 ms compared to an equivalent YOLO requiring NMS, and eliminates the NMS IoU threshold hyperparameter.

**Hungarian matching approximation for training on GPU.** The `scipy.optimize.linear_sum_assignment` solver is CPU-bound. GPU approximations using auction algorithms or successive shortest-path approaches exist [Source: "Auction algorithms for network flow problems", Bertsekas, 1992], but in practice the PyTorch implementation batches the cost matrix computation on the GPU (pairwise classification cost + box L1 + GIoU cost) and passes the small cost matrix (~300×gt_count) to the CPU solver, which is fast enough for training batch sizes of 2–8 images.

### 3.4 DINO-DETR for High-Accuracy Offline Detection

DINO-DETR [Source: arXiv:2203.03605] extends DETR with improved anchor box initialisation (mixed query selection), contrastive denoising training, and look-forward-twice refinement. It achieves state-of-the-art accuracy on COCO (>63 AP with a Swin-L backbone) at the cost of 3–5× longer inference than RT-DETR, making it suitable for offline processing (video archives, batch annotation pipelines) rather than real-time AR.

---

## 4. 3D Detection from LiDAR Point Clouds

### 4.1 PointPillars: Pillar Feature Encoding

PointPillars [Source: "PointPillars: Fast Encoders for Object Detection from Point Clouds", Lang et al., arXiv:1812.05784] represents the point cloud as a 2D pseudo-image by grouping points into vertical columns (pillars) over a BEV grid.

**GPU pillar scatter kernel.** Each point is mapped to its pillar's `(r, c)` cell in the BEV grid. Points within a pillar are encoded by a PointNet-style MLP. The scatter step writes the per-pillar feature vector into a dense 2D tensor at position `(r, c)`:

```cpp
// HIP/CUDA pillar scatter — one thread per pillar
__global__ void scatter_pillars(
    const float* pillar_features,  // [num_pillars, C]
    const int*   pillar_coords,    // [num_pillars, 2] — (row, col)
    float*       pseudo_image,     // [C, H, W] — BEV feature map
    int num_pillars, int C, int H, int W)
{
    int p = blockIdx.x * blockDim.x + threadIdx.x;
    if (p >= num_pillars) return;
    int row = pillar_coords[p * 2 + 0];
    int col = pillar_coords[p * 2 + 1];
    for (int c = 0; c < C; ++c)
        pseudo_image[c * H * W + row * W + col] =
            pillar_features[p * C + c];
}
```

Once the pseudo-image is formed, a standard 2D ResNet backbone + SSD detection head runs in the BEV space, treating the 3D detection problem as 2D object detection on the flattened ground plane. PointPillars achieves ~16 ms end-to-end on a single Titan XP GPU at the time of publication; modern hardware runs it in under 5 ms.

### 4.2 CenterPoint: Heatmap-Based 3D Detection

CenterPoint [Source: "Center-based 3D Object Detection and Tracking", Yin et al., arXiv:2006.11205] replaces the anchor-based SSD head with a **Gaussian heatmap head** over the BEV feature map. The voxel backbone (VoxelNet or PointPillars variant) produces a BEV feature tensor. The heatmap head predicts:

- **Class heatmap** `ĤK ∈ ℝ^{H×W×K}`: a Gaussian blob at each object center for K classes.
- **Sub-voxel offset** (corrects quantisation): 2D offset from voxel center to true center.
- **Height regression**: absolute z coordinate of object center.
- **Size regression**: `(l, w, h)` of the 3D box.
- **Rotation**: sin/cos of yaw angle.

At inference, the heatmap local maxima are extracted with a GPU 3×3 max-pool peak finding operation (locations where the heatmap value equals the max of the 3×3 neighbourhood). No NMS is needed for the heatmap peak extraction; a final cross-class score threshold removes low-confidence detections.

### 4.3 VoxelNet: Voxel Feature Encoding

VoxelNet [Source: "VoxelNet: End-to-End Learning for Point Cloud Based 3D Object Detection", Zhou & Tuzel, arXiv:1711.06396] pioneered learnable voxel feature encoding (VFE). For each non-empty voxel, a mini-PointNet processes all contained points: augment each point with the mean offset to the voxel centroid, apply a shared MLP, take the element-wise max across points. The result is a C-dimensional voxel feature.

**GPU voxelisation scatter with atomics.** Assigning points to voxels races across threads that might write to the same voxel. The canonical GPU implementation uses `atomicAdd` on a point-count buffer per voxel and `atomicCAS`-based spin locks to serialize the per-voxel point list construction, or pre-sorts points by voxel ID (radix sort) and then does a non-atomic sequential fill per voxel in a second kernel pass.

### 4.4 Range Image Detection

**RangeNet++** [Source: "RangeNet++: Fast and Accurate LiDAR Semantic Segmentation", Milioto et al., 2019, https://www.ipb.uni-bonn.de/wp-content/papercite-data/pdf/milioto2019iros.pdf] and **Cylinder3D** [Source: "Cylindrical and Asymmetrical 3D Convolution Networks for LiDAR Segmentation", Zhu et al., arXiv:2011.10033] project the LiDAR sphere onto a 2D range image (azimuth × elevation), enabling standard 2D CNN processing. Cylinder3D uses a cylindrical voxel grid (radius × azimuth × height) instead of Cartesian, better matching the angular distribution of LiDAR beams. Both approaches reduce the voxelisation cost at the expense of projection distortions near the sensor and occlusion artefacts.

---

## 5. RGB-D and Multimodal 3D Detection

### 5.1 Frustum PointNets

Frustum PointNets [Source: "Frustum PointNets for 3D Object Detection from RGB-D Data", Qi et al., arXiv:1711.08488] is a two-stage pipeline: a 2D detector finds objects in the RGB image, then a frustum defined by the 2D bounding box and camera intrinsics selects a subset of the depth point cloud. A PointNet then estimates the 3D bounding box within this frustum. The key GPU operation is the frustum point extraction: for each detected 2D box, project all depth points through the known camera intrinsics and test whether the projected pixel falls inside the box — a massively parallel per-point test.

### 5.2 ImVoxelNet: Image-to-Voxel Projection

ImVoxelNet [Source: "ImVoxelNet: Image to Voxels Projection for Monocular and Multi-View 3D Object Detection", Rukhovich et al., arXiv:2106.01178] lifts 2D image features into a 3D voxel volume by casting rays from the camera through each voxel center, sampling bilinear features from the 2D feature map at the projected pixel coordinate. For a voxel grid of `D×H×W` voxels:

```python
# PyTorch pseudocode — ray casting GPU
voxel_centers = create_voxel_grid(D, H, W)       # [D*H*W, 3]
projected = camera_intrinsics @ voxel_centers.T   # [3, N]
pixel_xy = projected[:2] / projected[2]           # [2, N] homogeneous divide
features_3d = grid_sample(img_features, pixel_xy) # bilinear from 2D feat
```

This is equivalent to a trilinear volume sampling, implemented efficiently via `torch.nn.functional.grid_sample` on GPU.

### 5.3 BEVFusion: Camera + LiDAR BEV Fusion

BEVFusion [Source: "BEVFusion: Multi-Task Multi-Sensor Fusion with Unified Bird's-Eye View Representation", Liu et al., MIT CSAIL, arXiv:2205.13542] fuses camera and LiDAR by projecting both into a shared BEV feature space. Camera features are lifted using **Lift-Splat-Shoot (LSS)** [Source: "Lift, Splat, Shoot", Philion & Fidler, arXiv:2008.05711]; LiDAR features are produced by the pillar/voxel encoder. The fused BEV feature map is passed to a detection head.

### 5.4 Lift-Splat-Shoot GPU Kernel

LSS predicts, for each image pixel, a **depth distribution** over discrete depth bins, then scatters pixel features into 3D space by outer product of the feature vector and depth probabilities:

```python
# LSS outer product + BEV scatter (PyTorch, runs on GPU)
# depth_probs: [N, D, H, W] — depth distribution per pixel
# img_feats:   [N, C, H, W] — image feature per pixel
# For each camera pixel (h,w), compute 3D feature at each depth bin:
#   feat3d[n, d, h, w, :] = depth_probs[n, d, h, w] * img_feats[n, :, h, w]
feat3d = depth_probs.unsqueeze(2) * img_feats.unsqueeze(1)
# Shape: [N, D, C, H, W]

# Unproject each (d, h, w) voxel to (X, Y, Z) via camera intrinsics
# Scatter-add into BEV grid using cumsum-based voxelisation:
bev = voxel_pooling(feat3d, frustum_coords, bev_grid_shape)
```

The `voxel_pooling` step is the GPU bottleneck: for each of the ~50k 3D voxels touched per camera, the GPU performs a scatter-add of a C-dimensional feature vector into the BEV grid cell. Efficient implementations use `torch_scatter.scatter_add` or a custom HIP kernel with `atomicAdd` on the BEV buffer.

---

## 6. 6DoF Pose Estimation — Keypoint-Based

### 6.1 PVNet: Pixel-Wise Voting

PVNet [Source: "PVNet: Pixel-wise Voting Network for 6DoF Pose Estimation", Peng et al., arXiv:1812.11788] predicts, for each visible object pixel, a unit 2D vector pointing toward each of K predefined 3D keypoints. Lines from each pixel in its predicted direction are aggregated by RANSAC voting to estimate the 2D projection of each keypoint. The voted 2D keypoint positions are then passed to a PnP solver to recover the 6DoF pose.

**GPU voting.** Each of the N×K pixel-direction pairs defines a 2D line. Finding the best-fit intersection of N lines for each keypoint is a robust mean shift or RANSAC problem. GPU voting iterates: sample a hypothesis (two lines → one intersection point), count inliers (all N threads evaluate the hypothesis in parallel), keep the best.

### 6.2 FFB6D: Full Flow Bidirectional Fusion

FFB6D [Source: "FFB6D: A Full Flow Bidirectional Fusion Network for 6D Pose Estimation", He et al., arXiv:2103.02242] processes RGB-D input by fusing appearance (ResNet+FPN on RGB) and geometry (PointNet++ on the depth point cloud) at every encoder and decoder level — "full flow bidirectional" refers to this cross-modal fusion at each scale. The network outputs per-point keypoint offset predictions for the visible 3D keypoints. Visible keypoints are selected by a learned visibility mask, reducing the PnP problem to well-conditioned visible points only.

### 6.3 GPU PnP Solvers

Given 2D–3D correspondences (n image keypoints with known 3D positions), PnP (Perspective-n-Point) recovers the camera rotation R and translation t. Three GPU-relevant approaches:

**EPnP** [Source: "EPnP: An Accurate O(n) Solution to the PnP Problem", Lepetit, Moreno-Noguer, Fua, IJCV 2009]: expresses the 3D point positions as a weighted sum of four virtual control points, then solves for the weights via a linear system (DLT), recovering the pose algebraically in O(n). For n ≥ 6, EPnP is non-iterative and GPU-parallel in the GEMV step that assembles the 12×12 matrix. OpenCV's GPU implementation (via CUDA) exposes EPnP as `cv::solvePnP` with `SOLVEPNP_EPNP`.

**P3P (Kneip)** [Source: "A Novel Parametrization of the P3P Problem", Kneip et al., CVPR 2011]: minimal solver using exactly 3 correspondences. Returns up to 4 solution candidates. Each candidate is evaluated by re-projection error on remaining points. In GPU RANSAC (§9.2), each warp evaluates one P3P hypothesis: 4 threads compute the 4 candidates; all threads in the warp vote on inliers.

**Levenberg-Marquardt iterative refinement.** After a closed-form solution, LM refines by minimising the sum of squared re-projection errors: `Σ ||π(R·X_i + t) − x_i||²`. On the GPU, the Jacobian `J ∈ ℝ^{2n×6}` is computed in parallel (2 rows per correspondence, 1 thread per correspondence). The normal equations `J^T J δ = −J^T r` are solved via Cholesky of the 6×6 system (always small — CPU-side or GPU with cuSolver for batched multi-object refinement).

---

## 7. Direct and Dense 6DoF Pose Estimation

### 7.1 FoundPose: Template Retrieval with Foundation Features

FoundPose [Source: "FoundPose: Unseen Object Pose Estimation with Foundation Features", Ornek et al., ECCV 2024; https://github.com/foundpose/foundpose — needs verification] uses DINOv2 ViT features as a foundation for template matching. At query time, the network extracts DINOv2 patch features from the observed image and retrieves the closest rendered template from a pre-rendered gallery (coarse pose). A refinement stage aligns features between the retrieved template and observation to correct the coarse pose. The pipeline requires no per-object training — only a 3D model is needed to render the gallery.

**GPU inference.** DINOv2 ViT-L inference processes a 518×518 crop in ~8 ms on an A100 (FP16). Template retrieval is a GPU k-NN search over the gallery embedding database using faiss [Source: https://github.com/facebookresearch/faiss] `IndexFlatL2` on GPU.

### 7.2 DUSt3R: Dense Uncalibrated Stereo Reconstruction

DUSt3R [Source: "DUSt3R: Geometric 3D Vision Made Easy", Wang et al., arXiv:2312.14132] takes a pair of images (or more, via global alignment) and predicts per-pixel 3D pointmaps `X ∈ ℝ^{H×W×3}` in a shared reference frame. The core model is a ViT-L encoder (shared weights) applied to each image independently, followed by a transformer decoder that cross-attends between the two image tokens to predict pointmaps and confidence maps.

**Pose recovery from pointmaps.** Given the predicted pointmaps `X1, X2` for a stereo pair, the relative pose is recovered by aligning `X1` to `X2` via procrustes (or robust weighted procrustes): a GPU SVD of the 3×3 cross-covariance matrix gives `R`, then `t = mean(X2) − R · mean(X1)`. For absolute pose with known 3D model, ICP (Ch224) refines the alignment.

**GPU inference.** DUSt3R ViT-L processes a 512×384 image pair in ~90 ms on an RTX 3090 at FP32, or ~40 ms at FP16. MASt3R (below) builds on DUSt3R at comparable speed.

### 7.3 MASt3R: 3D Matching and Structure

MASt3R [Source: "MASt3R: Grounding Image Matching in 3D with MASt3R", Leroy et al., arXiv:2406.09756, https://github.com/naver/mast3r] extends DUSt3R with a **local descriptor head**: in addition to the pointmap, the decoder predicts a per-pixel 24-dim descriptor optimised for geometric matching. This allows MASt3R to serve simultaneously as a 3D reconstruction backbone and a feature matcher for pose estimation. The descriptor matching on GPU uses a fast approximate nearest-neighbour search over the dense descriptor grid, followed by cycle-consistency filtering.

### 7.4 DeepIM: Render-and-Compare Refinement

DeepIM [Source: "DeepIM: Deep Iterative Matching for 6D Pose Estimation", Li et al., arXiv:1804.00175] takes an initial pose hypothesis, renders the object using the pose (via OpenGL or Vulkan rasterisation), and feeds the concatenated `[rendered | observed]` image pair through a FlowNet-style CNN that predicts a pose update `(Δt, Δq)`. This is applied to the pose and the process repeats (typically 3–5 iterations).

**GPU render for hypothesis evaluation.** Vulkan rasterisation for a single object at 640×480 in ~0.3 ms is achievable with `vkCmdDrawIndexed` dispatched from an offscreen render pass. For batch hypothesis evaluation (e.g., 100 pose candidates), rendering is parallelised across Vulkan instances using layered rendering (`VK_IMAGE_VIEW_TYPE_2D_ARRAY` + geometry shader layer selection or `VK_KHR_multiview`).

---

## 8. GPU Feature Matching for Pose

### 8.1 SuperPoint: Self-Supervised Detector and Descriptor

SuperPoint [Source: "SuperPoint: Self-Supervised Interest Point Detection and Description", DeTone, Malisiewicz, Rabinovich, arXiv:1712.07629, https://github.com/rpautrat/SuperPoint] is a shared-encoder CNN with two heads:

- **Detector head** (Heatmap softmax): produces a `[H/8, W/8, 65]` tensor of cell probabilities (64 pixel positions + 1 "no keypoint" dustbin). Keypoint positions are decoded by `argmax` within each cell.
- **Descriptor head**: produces a `[H/8, W/8, 256]` dense descriptor tensor. Descriptors at keypoint positions are extracted by bilinear interpolation and L2-normalised.

GPU inference for a 480×640 image runs in ~3 ms on an RTX 3090.

### 8.2 SuperGlue: Graph Neural Network Matching

SuperGlue [Source: "SuperGlue: Learning Feature Matching with Graph Neural Networks", Sarlin et al., arXiv:1911.11763, https://github.com/magicleap/SuperGluePretrainedNetwork] formulates feature matching as an optimal transport problem solved by a **Sinkhorn iteration** on a score matrix `S ∈ ℝ^{(m+1)×(n+1)}` (including a dustbin row/column for unmatched keypoints).

**Sinkhorn GPU iteration.** The Sinkhorn algorithm alternates row and column normalisation of the score matrix:

```python
# GPU Sinkhorn — batched log-domain for numerical stability
def log_sinkhorn(log_S, num_iters=100):
    # log_S: [B, m+1, n+1]
    log_mu = torch.zeros(B, m+1)   # row marginals (uniform)
    log_nu = torch.zeros(B, n+1)   # col marginals (uniform)
    for _ in range(num_iters):
        log_S = log_S - torch.logsumexp(log_S, dim=2, keepdim=True) + log_nu.unsqueeze(1)
        log_S = log_S - torch.logsumexp(log_S, dim=1, keepdim=True) + log_mu.unsqueeze(2)
    return log_S
```

Each Sinkhorn iteration is a logsumexp + add operation over `(m+1)×(n+1)` — embarrassingly parallel across the batch and matrix dimensions. With m=n=512 keypoints, each iteration is a ~500k-element reduction on the GPU.

### 8.3 LightGlue: Adaptive-Depth Fast Matching

LightGlue [Source: "LightGlue: Local Feature Matching at Light Speed", Lindenberger et al., arXiv:2306.13643, https://github.com/cvg/LightGlue] replaces Sinkhorn with a transformer-based classifier per keypoint pair, and introduces two GPU-efficiency innovations:

- **Early exit (adaptive depth)**: after each attention layer, LightGlue predicts a confidence score for each keypoint. Keypoints above a confidence threshold are "pruned" from further computation. This reduces the effective sequence length across layers — the first layer processes all N keypoints, later layers may process only 30–50%, enabling a 2–3× speedup on easy image pairs.
- **Flash-Attention integration**: self-attention over the keypoint graph uses Flash-Attention [Source: https://github.com/Dao-AILab/flash-attention] for O(N) memory footprint, allowing longer keypoint sequences without GPU OOM.

LightGlue is 2–5× faster than SuperGlue at matching quality parity on standard benchmarks.

### 8.4 LoFTR: Dense Coarse-to-Fine Matching

LoFTR [Source: "LoFTR: Detector-Free Local Feature Matching with Transformers", Sun et al., arXiv:2104.00680, https://github.com/zju3dv/LoFTR] uses a coarse-to-fine approach without keypoint detection:

1. **Coarse matching** (at 1/8 resolution): both images pass through a CNN+transformer encoder producing `[H/8 × W/8, C]` feature maps. A linear attention layer (O(N) via kernel trick) performs self- and cross-attention. Mutual nearest-neighbour matching on the coarse features produces coarse correspondences.
2. **Fine matching** (at 1/2 resolution): around each coarse match, a local window (5×5 at 1/2 res) is extracted. A fine-level transformer refines the match to sub-pixel accuracy.

On GPU, LoFTR achieves ~70 ms per pair on a V100 for 640×480 images; Efficient LoFTR (2024) reduces this to ~30 ms with optimised linear attention kernels.

### 8.5 GPU Efficiency Comparison

| Method | Keypoints | GPU time (RTX 3090) | Matching quality |
|---|---|---|---|
| SuperPoint + SuperGlue | Sparse (~512) | ~30 ms | High |
| SuperPoint + LightGlue | Sparse (~512) | ~8 ms | High |
| LoFTR | Dense (thousands) | ~30 ms | Very high |
| Efficient LoFTR | Dense | ~12 ms | High |
| DUSt3R (pair) | Dense | ~40 ms (FP16) | Very high + 3D |

---

## 9. GPU RANSAC and Robust Estimation

### 9.1 The USAC Framework

USAC (Universal Sample Consensus) [Source: "USAC: A Universal Framework for Random Sample Consensus", Raguram et al., IEEE TPAMI 2013] unifies RANSAC variants under a common interface with pluggable components: hypothesis sampler, model solver, inlier counter, and local optimiser. Key variants:

- **Graph-Cut RANSAC** [Source: "Graph-Cut RANSAC", Barath & Matas, arXiv:1706.00984]: applies graph-cut based spatial coherence to group nearby inliers, then refines within the coherent inlier set. Reduces the number of RANSAC iterations needed.
- **MAGSAC++** [Source: "MAGSAC++", Barath et al., arXiv:1912.05909]: instead of a hard inlier threshold, weights each correspondence by its probability of being an inlier given a marginalised noise model (σ-consensus). Produces more accurate models on noisy data.
- **DEGENSAC** [Source: "Two-View Geometry Estimation Unaffected by a Dominant Plane", Chum et al., CVPR 2005]: detects degenerate configurations (all points on a plane for the essential matrix solver) and switches to a specialised degenerate solver. Prevents locking onto a degenerate model.

OpenCV integrates these via `cv::findEssentialMat`, `cv::findHomography`, `cv::solvePnPRansac` with `cv::UsacParams`. [Source: OpenCV USAC documentation, https://docs.opencv.org/4.x/d9/d0c/group__calib3d.html]

### 9.2 GPU RANSAC Parallelism

GPU RANSAC parallelises hypothesis evaluation rather than the sequential draw-and-test loop:

**Warp-per-hypothesis model.** Each warp (32 threads on NVIDIA, 64 on AMD) processes one RANSAC hypothesis. The minimal solver (P3P, 5-point, homography DLT) runs in the first few threads of the warp; all threads evaluate the resulting model against their assigned correspondences in parallel; a warp-level reduction (`__reduce_add_sync`, GLSL `subgroupAdd`) counts inliers.

```glsl
// GLSL GPU RANSAC — warp-per-hypothesis inlier counting
layout(local_size_x = 32) in;  // one warp per hypothesis
// ... hypothesis drawn from shared memory, model solved ...
float err = reprojection_error(model, correspondence[gl_GlobalInvocationID.x]);
uint inlier = uint(err < THRESHOLD);
uint total_inliers = subgroupAdd(inlier);
if (subgroupElect())
    atomicMax(best_inlier_count, total_inliers);
```

**Independent hypothesis threads.** A 3D dispatch with `(num_hypotheses / 32, 1, 1)` workgroups runs all hypotheses simultaneously. For 10,000 hypotheses with 512 correspondences, this launches ~313 workgroups, fully saturating a GPU with 256 compute units.

### 9.3 LO-RANSAC: Local Optimisation

After each hypothesis that achieves a new best inlier count, LO-RANSAC refines by fitting a new model using all current inliers (rather than just the minimal sample). The inlier set re-fitting is a GPU GEMV (assembling the least-squares system) followed by a CPU-side linear solve on the small (6×6 or 9×9) system.

### 9.4 GPU Minimal Solvers

**5-point essential matrix (Nister)** [Source: "An Efficient Solution to the Five-Point Relative Pose Problem", Nister, IEEE TPAMI 2004]: Given 5 point correspondences, solves a degree-10 polynomial system yielding up to 10 solutions. GPU batched evaluation generates 10 candidate essential matrices per hypothesis; inlier counting selects the best. In a GPU RANSAC loop at 10,000 iterations, this requires 100,000 essential matrix candidate evaluations — each a simple `[3×3]` matrix multiplication per correspondence.

**P3P (Kneip)** [Source: "A Novel Parametrization of the P3P Problem", Kneip et al., CVPR 2011]: 4 solution candidates per 3-point sample. GPU implementation in OpenCV's USAC backend.

**Homography DLT (4-point)**: Given 4 point correspondences, assembles an `8×9` matrix and extracts the null space via SVD. For GPU batching, the SVD of many 8×9 matrices is solved via the cuSOLVER or rocSOLVER batched SVD API.

---

## 10. Pose Tracking and Temporal Refinement

### 10.1 FoundationPose Tracking Mode

FoundationPose [Source: "FoundationPose: Unified 6D Pose Estimation and Tracking of Novel Objects", Wen et al., arXiv:2312.08344, https://github.com/NVlabs/FoundationPose] operates in two modes: initialisation (estimates pose from scratch given an observation) and tracking (propagates pose between frames). In tracking mode, the previous pose is used to render a set of perturbed hypothesis poses; a scoring network ranks them by comparing rendered vs. observed features; the best-scoring pose is refined with ICP. The multi-hypothesis scoring runs as a batched GPU inference of a lightweight 3D feature extractor.

### 10.2 GPU Kalman Filter for 6DoF Pose

A Kalman filter propagates pose between detection frames. The state vector for 6DoF tracking is `x = [t_x, t_y, t_z, q_x, q_y, q_z, q_w, v_x, v_y, v_z]` (position + quaternion + velocity). On the GPU, all matrix operations (predict: `x̂ = F·x`, `P = F·P·F^T + Q`; update: `K = P·H^T·(H·P·H^T + R)^{-1}`, `x = x̂ + K·(z − H·x̂)`) are standard dense GEMM. For tracking M objects simultaneously, the matrices are batched: `P ∈ ℝ^{M×10×10}`, dispatched via cuBLAS or rocBLAS batched GEMM.

### 10.3 Pose Graph Optimisation

For multi-object or multi-frame consistency (SLAM-style AR anchoring), a pose graph records pairwise pose constraints between keyframes. The graph is optimised by minimising the sum of squared constraint errors, equivalent to a sparse nonlinear least-squares problem.

**g2o** [Source: https://github.com/RainerKuemmerle/g2o] and **GTSAM** [Source: https://github.com/borglab/gtsam] implement pose graph optimisation with sparse Cholesky factorisation of the information matrix (Hessian approximation). For GPU acceleration, the Jacobian computation (embarrassingly parallel across factors) runs on GPU via custom CUDA/HIP kernels; the sparse linear solve runs on CPU using SuiteSparse CHOLMOD or on GPU via cuSPARSE `cusparseSpSolve` for the solve step.

### 10.4 Photometric Pose Refinement

Given a pose estimate, render the object, compare per-pixel intensities with the observed image, and compute a photometric loss:

```
L_photo = Σ_{p ∈ visible} ||I_rendered(π(R·X_p + t)) − I_obs(x_p)||²
```

Differentiable rendering (Ch229) provides `∂L/∂(R, t)` via automatic differentiation through rasterisation. The GPU pipeline: Vulkan/CUDA rasterise object → compute photometric loss → backpropagate through screen-space pixels to pose Jacobian → pose update step. This gradient-based refinement converges in 5–10 GPU iterations and achieves sub-millimetre accuracy on smooth surfaces.

---

## 11. Production Deployment on Linux

### 11.1 Model Export: PyTorch → ONNX

```python
import torch
import torch.onnx

model = load_pretrained_pose_model()
model.eval()

dummy_rgb   = torch.randn(1, 3, 480, 640, device="cuda")
dummy_depth = torch.randn(1, 1, 480, 640, device="cuda")

torch.onnx.export(
    model,
    (dummy_rgb, dummy_depth),
    "pose_model.onnx",
    opset_version=17,
    dynamic_axes={
        "rgb":   {0: "batch"},
        "depth": {0: "batch"},
    },
    input_names=["rgb", "depth"],
    output_names=["keypoint_offsets", "visibility_mask"],
)
```

[Source: PyTorch ONNX export guide, https://pytorch.org/docs/stable/onnx.html]

### 11.2 ONNX Runtime ROCm Execution Provider

```cpp
// ONNX Runtime C API — ROCm EP
#include <onnxruntime_c_api.h>

const OrtApi* g_ort = OrtGetApiBase()->GetApi(ORT_API_VERSION);

OrtEnv* env;
g_ort->CreateEnv(ORT_LOGGING_LEVEL_WARNING, "pose", &env);

OrtSessionOptions* opts;
g_ort->CreateSessionOptions(&opts);

OrtROCMProviderOptions rocm_opts{};
rocm_opts.device_id = 0;
rocm_opts.miopen_conv_exhaustive_search = 1;
g_ort->SessionOptionsAppendExecutionProvider_ROCM(opts, &rocm_opts);

OrtSession* session;
g_ort->CreateSession(env, L"pose_model.onnx", opts, &session);

// Input binding
OrtMemoryInfo* mem_info;
g_ort->CreateMemoryInfo("Hip", OrtDeviceAllocator, 0,
                        OrtMemTypeDefault, &mem_info);
// ... create input tensors from GPU buffers, Run() ...
```

[Source: ONNX Runtime ROCm EP documentation, https://onnxruntime.ai/docs/execution-providers/ROCm-ExecutionProvider.html]

### 11.3 TensorRT INT8 Calibration

```python
import tensorrt as trt

logger = trt.Logger(trt.Logger.INFO)
builder = trt.Builder(logger)
config = builder.create_builder_config()
config.set_flag(trt.BuilderFlag.INT8)

# INT8 entropy calibrator
calibrator = trt.IInt8EntropyCalibrator2()
config.int8_calibrator = calibrator

# Build engine from ONNX
network = builder.create_network(
    1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH))
parser = trt.OnnxParser(network, logger)
parser.parse_from_file("yolov8m.onnx")

engine = builder.build_serialized_network(network, config)
with open("yolov8m_int8.engine", "wb") as f:
    f.write(engine)
```

[Source: TensorRT Python API reference, https://docs.nvidia.com/deeplearning/tensorrt/api/python_api/]

### 11.4 OpenVINO for Intel GPU

```bash
# Convert ONNX to OpenVINO IR using the Model Conversion tool (OVC)
ovc pose_model.onnx \
    --input_shape "[1,3,480,640],[1,1,480,640]" \
    --output_model pose_model.xml

# Run inference on Intel GPU (Arc/Xe via Level Zero)
python3 -c "
import openvino as ov
core = ov.Core()
model = core.read_model('pose_model.xml')
compiled = core.compile_model(model, 'GPU')
infer_req = compiled.create_infer_request()
# ... set input tensors, infer() ...
"
```

[Source: OpenVINO Model Conversion documentation, https://docs.openvino.ai/2024/openvino-workflow/model-preparation.html]

### 11.5 Triton Inference Server on Linux

Triton Inference Server [Source: https://github.com/triton-inference-server/server] hosts detection and pose models with dynamic batching and concurrent model execution. A minimal model repository for YOLOv8 + pose estimation:

```
model_repository/
├── yolov8_detection/
│   ├── config.pbtxt
│   └── 1/model.onnx
└── pose_estimator/
    ├── config.pbtxt
    └── 1/model.plan   # TensorRT engine
```

```protobuf
# config.pbtxt for YOLOv8 ONNX on ROCm
name: "yolov8_detection"
backend: "onnxruntime"
default_model_filename: "model.onnx"
max_batch_size: 8
dynamic_batching { preferred_batch_size: [1, 4, 8] }
input  [{ name: "images" data_type: TYPE_FP32 dims: [3, 640, 640] }]
output [{ name: "output0" data_type: TYPE_FP32 dims: [84, 8400] }]
instance_group [{ kind: KIND_GPU gpus: [0] }]
```

Client-side request via HTTP:

```python
import tritonclient.http as httpclient
import numpy as np

client = httpclient.InferenceServerClient("localhost:8000")
inp = httpclient.InferInput("images", [1, 3, 640, 640], "FP32")
inp.set_data_from_numpy(image_array)
result = client.infer("yolov8_detection", [inp])
detections = result.as_numpy("output0")
```

### 11.6 Latency Profiling

```bash
# ONNX Runtime profiling — produces JSON timeline
sess_opts.enable_profiling = True
# After inference, collect profile file and view in chrome://tracing

# TensorRT profiling — per-layer timing
trtexec --onnx=yolov8m.onnx --int8 --calib=calib.cache \
        --profilingVerbosity=detailed --exportProfile=trt_profile.json

# AMD GPU profiling with rocprof
rocprof --hip-trace --hsa-trace -o pose_trace.csv \
        python3 run_pose_inference.py
```

---

## 12. Integration with the Graphics Pipeline

### 12.1 Detection Output → GPU-Driven Draw Call Filter

Detected object bounding boxes can drive GPU indirect rendering to skip draw calls for non-visible or irrelevant objects. For each object in the scene, a compute shader tests whether its 3D AABB overlaps any detected bounding box's frustum, writing a 0 or 1 into a `VkDrawIndexedIndirectCommand.indexCount` multiplier:

```glsl
// Compute shader — per-object visibility from detection results
layout(binding = 0) buffer DetectionResults {
    vec4 detected_boxes_3d[MAX_DETECTIONS]; // [cx, cy, cz, label]
    int  detection_count;
};
layout(binding = 1) buffer IndirectDraws {
    VkDrawIndexedIndirectCommand commands[MAX_OBJECTS];
};

void main() {
    uint obj = gl_GlobalInvocationID.x;
    bool visible = false;
    for (int d = 0; d < detection_count; ++d)
        if (aabb_intersects(object_aabbs[obj], detected_boxes_3d[d]))
            visible = true;
    commands[obj].instanceCount = visible ? 1 : 0;
}
```

`vkCmdDrawIndexedIndirect` then dispatches only visible objects without CPU readback.

### 12.2 Pose Estimate → Vulkan Push Constant → AR Rendering

A 6DoF pose `(R, t)` is converted to a 4×4 model matrix `M = [R | t; 0 0 0 1]` and uploaded to a Vulkan render pass via push constant:

```cpp
struct PosePushConstant {
    float model_matrix[16];   // column-major 4x4
    float object_colour[4];   // RGBA for AR overlay tint
};

PosePushConstant pc;
mat4_from_Rt(pc.model_matrix, pose.R, pose.t);

vkCmdPushConstants(
    cmd,
    pipeline_layout,
    VK_SHADER_STAGE_VERTEX_BIT,
    0,
    sizeof(PosePushConstant),
    &pc);

vkCmdDrawIndexed(cmd, index_count, 1, 0, 0, 0);
```

The vertex shader reads `push_constant.model_matrix` and transforms object vertices into camera space, compositing the virtual object over the live camera feed.

### 12.3 GPU Detection Output → Wayland Overlay Surface

For an AR HUD built on Wayland, a detection overlay is rendered into a separate Wayland surface using the `wl_subcompositor` protocol. The compositor blends the overlay above the camera preview surface. Detection bounding boxes and labels are drawn via GPU (a lightweight Vulkan-based 2D renderer or Cairo+EGL) into the overlay buffer, then `wl_surface.commit` presents the frame. The compositor handles the blend without involving the CPU for each frame after the initial protocol negotiation.

### 12.4 Depth from RGB-D → Pose Refinement → AR Anchor Stability

Depth from an RGB-D sensor feeds directly into ICP-based pose refinement (Ch224) as a final step in the pose pipeline: the estimated pose aligns the object's 3D model with the observed depth point cloud, reducing drift from detection-only pipelines. The AR anchor is then expressed in world coordinates (accumulated pose × initial detection pose), providing stable virtual object placement through head motion.

### 12.5 Monado OpenXR Scene Understanding

Monado [Source: https://monado.freedesktop.org/] implements OpenXR runtime on Linux and provides extension support for scene understanding extensions. The extensions relevant to object detection integration:

- `XR_MSFT_scene_understanding`: detects planes, meshes, and semantic objects (walls, floors, tables) from depth sensors. The extension's `xrComputeNewSceneMSFT` call triggers device-side inference and is intended to compose with application-level object detection.
- `XR_EXT_plane_detection`: cross-vendor plane detection (horizontal/vertical planes) directly usable for AR surface anchoring without running a full object detection pipeline. [Source: Khronos OpenXR spec, https://www.khronos.org/registry/OpenXR/specs/1.0/html/xrspec.html]

These extensions are implemented in Monado's scene understanding module (`src/xrt/drivers/wmr/` for Windows Mixed Reality hardware and WMR passthrough; generic plane detection for other depth-capable devices). As of mid-2026, full `XR_MSFT_scene_understanding` support in Monado remains hardware-specific (needs verification against current Monado master).

---

## Integrations

This chapter connects to the following chapters in this book:

**Ch27 (VR/AR Applications).** The AR rendering pipeline described in §12 builds on the OpenXR application layer covered in Ch27: `XrSession`, `XrSwapchain`, `xrEndFrame` composition layers. Detected object poses provide the `XrPosef` for spatial anchors and overlay layer placement.

**Ch224 (3D Shape Analysis — ICP and Shape Descriptors).** ICP (Iterative Closest Point) is used in §7.2 to align DUSt3R pointmaps with known 3D models and in §10.1 as the final refinement step in FoundationPose tracking. The GPU ICP kernel (parallel closest-point query via BVH, batched SVD for the rotation update) is described in Ch224 §5.

**Ch228 (GPU Graph Algorithms).** SuperGlue's Sinkhorn matching (§8.2) iterates over a bipartite assignment graph; the Hungarian matching approximation for RT-DETR training (§3.3) is a minimum-weight matching problem on a bipartite graph. The graph neural network layer in SuperGlue's keypoint attention is an instance of the GNN inference pattern covered in Ch228 §10.

**Ch229 (GPU ML Inference Algorithms).** The backbone CNN inference for YOLOv8/RT-DETR/SuperPoint/LoFTR uses the Conv2D-to-GEMM lowering, transformer attention, and quantisation algorithms described in Ch229. Production deployment via ONNX Runtime EP and TensorRT covered in Ch229 §9 applies equally to detection and pose models.

**Ch236 (Semantic Segmentation on GPU).** Instance segmentation (Mask R-CNN, YOLOv8-seg) produces per-pixel class masks that feed object detection by providing tight instance masks for the frustum extraction step (§5.1) and visible-keypoint selection in FFB6D (§6.2). The segmentation mask defines the visible surface for photometric pose refinement (§10.4).

**Ch237 (Depth Estimation on GPU).** Monocular depth estimation provides per-pixel depth when an RGB-D sensor is unavailable, enabling the frustum PointNet (§5.1) and lift-splat-shoot (§5.4) pipelines on RGB-only cameras. Metric depth from networks such as Depth Anything V2 or UniDepth feeds directly into the 3D lifting stage and the ICP refinement step.
