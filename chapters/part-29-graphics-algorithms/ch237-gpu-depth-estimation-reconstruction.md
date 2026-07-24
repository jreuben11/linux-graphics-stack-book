# Chapter 237: GPU Depth Estimation and Dense Reconstruction (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers building AR/XR depth pipelines on Linux; systems developers integrating GPU-based depth estimation and 3D reconstruction into robotics or scanning workflows; engineers building 3D capture pipelines using Vulkan compute, CUDA, or HIP on Linux hardware. This chapter assumes familiarity with GPU image processing (Ch220), Vulkan compute basics (Ch25), TSDF volumetric fusion (Ch210), and point cloud processing (Ch211).

---

## Table of Contents

1. [Depth Estimation and Reconstruction as a GPU Domain](#1-depth-estimation-and-reconstruction-as-a-gpu-domain)
2. [Monocular Depth Estimation](#2-monocular-depth-estimation)
3. [Stereo Depth Beyond SGM](#3-stereo-depth-beyond-sgm)
4. [RGB-D Processing](#4-rgb-d-processing)
5. [Structure-from-Motion GPU Stages](#5-structure-from-motion-gpu-stages)
6. [Multi-View Stereo GPU Inference](#6-multi-view-stereo-gpu-inference)
7. [NeRF Reconstruction Pipelines](#7-nerf-reconstruction-pipelines)
8. [Mesh Extraction from Implicit Representations](#8-mesh-extraction-from-implicit-representations)
9. [Large-Scale and Online Reconstruction](#9-large-scale-and-online-reconstruction)
10. [GPU Depth and Reconstruction in Linux Pipelines](#10-gpu-depth-and-reconstruction-in-linux-pipelines)
11. [Evaluation and Benchmarking](#11-evaluation-and-benchmarking)
12. [Integration with the Graphics Stack](#12-integration-with-the-graphics-stack)
- [Integrations](#integrations)

---

## 1. Depth Estimation and Reconstruction as a GPU Domain

### 1.1 Sensor Modalities

3D scene understanding requires converting 2D image projections or raw sensor readings into metric depth maps and volumetric models. Five sensor modalities dominate Linux-based GPU pipelines:

- **RGB camera (monocular):** A single camera provides no direct geometric constraint. Depth must be inferred from learned priors or from temporal motion parallax. Monocular depth estimation is fully learned and runs as GPU inference.
- **Stereo camera pair:** Two horizontally displaced cameras produce disparity maps convertible to metric depth via the stereo baseline. Classical SGM and learning-based methods both run on GPU.
- **Structured light:** An IR projector emits a known coded pattern (Gray codes, phase-shifting sinusoids); an IR camera decodes the deformation. Decoding runs in GPU compute. Used in RealSense D4xx and Azure Kinect at close range.
- **LiDAR (SLAM, AV):** Rotating or solid-state LiDAR produces sparse 3D point clouds. GPU preprocessing includes voxel downsampling, normal estimation, and ICP registration (Ch211 §90).
- **RGB-D / Time-of-Flight (ToF):** Active depth from amplitude-modulated light (Microsoft Azure Kinect, Intel RealSense L515, OAK-D). Produces dense depth at 30–90 fps but with noise, multi-path interference, and flying pixels. Requires GPU post-processing.

### 1.2 Sparse vs Dense Reconstruction

Reconstruction pipelines divide into sparse and dense stages:

**Sparse SfM** (Structure-from-Motion): tracks 2D feature points across images, solves for 3D point positions and camera poses simultaneously. Scales to thousands of images but produces only a sparse point cloud and camera trajectory.

**Dense reconstruction** operates on the full image content:
- **Multi-View Stereo (MVS):** given calibrated cameras from SfM, computes dense per-view depth maps and fuses them.
- **TSDF fusion:** integrates depth maps into a truncated signed distance function volume (Ch210 §63). Enables real-time fusion from RGB-D sensors.
- **Neural Radiance Fields (NeRF):** learns a continuous volumetric scene representation from images; depth is a side product of volume rendering.
- **3D Gaussian Splatting (3DGS):** represents scenes as a collection of anisotropic 3D Gaussians optimized from multi-view images; depth can be extracted by alpha-compositing the Gaussian depths.

### 1.3 GPU Parallelism Opportunities

Each stage of a reconstruction pipeline exposes distinct parallelism:

| Stage | Primary parallelism | GPU primitive |
|---|---|---|
| Feature extraction (SIFT, deep) | Per-image, per-scale level | 2D convolutions, DoG compute |
| Cost volume construction (stereo) | Per-pixel × per-disparity | Grouped convolution, matrix multiply |
| Depth map estimation (CNN/transformer) | Per-pixel output | Dense inference, attention |
| TSDF integration | Per-voxel update | 3D dispatch, atomic float ops |
| NeRF ray marching | Per-ray sample | warp-level MLP evaluation |
| 3DGS rasterisation | Per-Gaussian tile | tile-based sorting + splat |
| Mesh extraction (MC) | Per-voxel cube | parallel prefix scan |

### 1.4 Linux GPU Deployment Landscape

As of mid-2026, three compute frameworks host these workloads on Linux:

- **CUDA (NVIDIA):** the dominant framework. COLMAP, cuSIFT, RAFT-Stereo reference implementations, Instant-NGP, and 3DGS all target CUDA natively.
- **ROCm / HIP (AMD):** growing coverage. PyTorch ROCm executes monocular depth models and NeRF training. ONNX Runtime ships a MIGraphX execution provider (EP) for AMD GPUs. COLMAP ROCm support is community-maintained.
- **Vulkan compute (AMD, Intel, NVIDIA):** portable GPU compute via SPIR-V. Used for post-processing (normal estimation, hole-filling), TSDF integration in robotics stacks (Voxblox), and display integration.
- **Intel Arc (oneAPI / Level Zero):** ONNX Runtime OpenVINO EP supports Depth Anything v2 and MiDaS on Intel discrete GPUs. OpenCV DNN module runs on OpenVINO backend.

Embedded Linux deployments (NVIDIA Jetson Orin, Qualcomm Snapdragon 8cx) use CUDA and Hexagon DSP respectively. ROS 2 provides the system integration layer for sensor pipelines (§10).

---

## 2. Monocular Depth Estimation

### 2.1 DPT — Dense Prediction Transformer

DPT (Ranftl et al., ICCV 2021) was the first architecture to apply a Vision Transformer (ViT) backbone to dense prediction tasks including depth estimation. [Source: Ranftl et al., "Vision Transformers for Dense Prediction", ICCV 2021, https://github.com/isl-org/DPT]

The backbone divides the image into 16×16 patches, projects them to token embeddings, and applies multi-head self-attention across all patch tokens at each of the transformer's encoder stages. Four stages of ViT encoder activations are tapped at stages 3, 6, 9, and 12 (for ViT-B/16) and fed into DPT's **Reassemble** blocks, which invert the patch-projection back to 2D feature maps at progressively higher spatial resolution (1/32, 1/16, 1/8, 1/4 of input). A **Fusion** decoder then progressively merges features from coarse to fine using residual convolution blocks, producing a full-resolution inverse-depth map.

On GPU, the bottleneck is the attention computation: O(N²) in sequence length N = H/16 × W/16. For a 384×384 input, N=576; for 640×480, N=1200. Flash Attention [Source: Dao et al., https://github.com/Dao-AILab/flash-attention] reduces peak memory from O(N²) to O(N) and achieves 2–4× speedup on NVIDIA Ampere and AMD RDNA3 hardware.

### 2.2 MiDaS Multi-Scale Fusion

MiDaS (Ranftl et al., 2022) trains a unified depth model on a mixture of diverse depth datasets by normalising them to a common scale-and-shift-invariant training objective. [Source: Ranftl et al., "Towards Robust Monocular Depth Estimation", TPAMI 2022, https://github.com/isl-org/MiDaS]

MiDaS v3.1 supports multiple backbone options — DPT-BEiT-L (highest quality), DPT-SwinV2-L (speed-quality tradeoff), and MiDaS-small (MobileNetV2, fast CPU inference). The decoder is the DPT decoder (§2.1). Switching backbones changes only the feature extraction stage; the decoder and head remain fixed, enabling the model to consume mixed training data from datasets with incompatible depth ground-truth modalities.

ONNX export (`torch.onnx.export`) allows deployment on ONNX Runtime with TensorRT EP (NVIDIA), MIGraphX EP (AMD ROCm), or OpenVINO EP (Intel):

```bash
# Export MiDaS DPT-Small to ONNX
python export_onnx.py --model_type dpt_swin2_tiny_256 --output midas_swin2_tiny.onnx

# Run on ONNX Runtime with TensorRT EP
python run_onnx.py --model midas_swin2_tiny.onnx --provider TensorrtExecutionProvider
```

### 2.3 Depth Anything v2

Depth Anything v2 (Yang et al., 2024) addresses the training data quality bottleneck by using a teacher–student distillation pipeline: a large ViT-L model trained on high-quality synthetic labeled data teaches a student model trained on unlabeled real-world images, producing models that generalise dramatically better across environments. [Source: Yang et al., "Depth Anything V2", NeurIPS 2024, https://github.com/DepthAnything/Depth-Anything-V2]

The architecture mirrors DPT: ViT-S/B/L backbone + DPT decoder. The **metric** variant adds an additional prediction head calibrated to produce absolute metric depth (in meters), trained with metric supervision from high-quality datasets (HyperSim, Virtual KITTI). The relative variant predicts scale-and-shift-invariant inverse depth, suitable as a prior when metric scale is unknown.

ONNX Runtime deployment on TensorRT EP enables real-time inference: ViT-S at 640×480 achieves approximately 25–35 fps on an RTX 3090 with TensorRT INT8 quantisation.

### 2.4 ZoeDepth

ZoeDepth (Shariq et al., 2023) builds metric depth estimation on top of MiDaS relative depth. [Source: Shariq et al., "ZoeDepth: Zero-Shot Transfer by Combining Relative and Metric Depth", arXiv 2302.12288, https://github.com/isl-org/ZoeDepth]

The MiDaS relative depth head is frozen after pre-training. A lightweight **metric binning module** (ZoeHead) is appended that learns to predict a log-linear depth histogram from the scene context, then combines the histogram with the relative depth to produce absolute metric depth. This two-stage approach isolates the scale ambiguity problem: the relative head captures geometry; the metric head learns the depth scale distribution from dataset-specific statistics.

### 2.5 Real-Time Monocular Depth — FastDepth and Embedded GPU

FastDepth (Wofk et al., ICRA 2019) targets embedded GPU deployment by using a MobileNet-v1 encoder and a highly simplified depthwise-separable convolution decoder. [Source: Wofk et al., "FastDepth: Fast Monocular Depth Estimation on Embedded Systems", ICRA 2019, https://github.com/dwofk/fast-depth]

On NVIDIA Jetson AGX Xavier (embedded Linux, Volta iGPU), FastDepth achieves approximately 178 fps at 224×224 with TensorRT INT8 quantisation, enabling real-time AR overlay at sensor framerate. The Snapdragon 8cx Hexagon DSP can run similar MobileNet-based depth models via Qualcomm's SNPE (Snapdragon Neural Processing Engine) runtime.

---

## 3. Stereo Depth Beyond SGM

### 3.1 RAFT-Stereo

RAFT-Stereo (Lipson et al., BMVC 2021) adapts the RAFT optical flow architecture to the 1D stereo matching problem. [Source: Lipson et al., "RAFT-Stereo: Multilevel Recurrent Field Transforms for Stereo Matching", BMVC 2021, https://github.com/princeton-vl/RAFT-Stereo]

**Feature extraction:** a shared encoder (6-layer ResNet with group norm) processes left and right images to produce feature maps at 1/4 resolution (H/4 × W/4 × 256 channels). A separate context encoder processes the left image only and initialises the GRU hidden state.

**Correlation volume construction:** the 4D correlation volume C(u, v, u', v') is computed as the dot product of feature vectors at every left-right position pair. For stereo the search is 1D (horizontal disparity), so the volume collapses to C(u, v, d) where d ranges over max disparity D. Construction cost: O(H/4 × W/4 × D × C_feat), with C_feat=256; this is the GPU memory bottleneck. RAFT-Stereo constructs a pyramid of correlation volumes (4 levels) by average-pooling the right feature map.

**GRU iterative refinement:** starting from a zero disparity initialisation, the ConvGRU hidden state is updated for N iterations (default N=32 for full quality, N=7 for real-time):

```
# Pseudo-code: RAFT-Stereo GRU update at iteration k
corr_features = correlate(left_feat, right_feat_pyramid, disparity_k)  # lookup around current estimate
inp = cat(context, corr_features)
h_k+1 = ConvGRU(h_k, inp)
delta_d = flow_head(h_k+1)                # predict disparity residual
disparity_k+1 = disparity_k + delta_d
```

The final output is upsampled from 1/4 to full resolution via a learned convex upsampling module. RAFT-Stereo achieves state-of-the-art EPE on ETH3D and KITTI 2015 benchmarks.

### 3.2 CREStereo

CREStereo (Li et al., CVPR 2022) extends cascaded recurrent matching to stereo with **adaptive group correlation**. [Source: Li et al., "Practical Stereo Matching via Cascaded Recurrent Network with Adaptive Correlation", CVPR 2022, https://github.com/megvii-research/CREStereo]

The key innovation is computing correlation in grouped feature channels (reducing memory) and performing hierarchical refinement from coarse to fine disparity resolution. At each level, the disparity estimate from the previous level is used to initialise a refined search range, reducing the effective correlation volume size and enabling deployment at higher image resolutions.

### 3.3 HITNet

HITNet (Tankovich et al., CVPR 2021) uses **slanted plane hypotheses** rather than per-pixel disparity. [Source: Tankovich et al., "HITNet: Hierarchical Iterative Tile Refinement Network for Real-time Stereo Matching", CVPR 2021, https://arxiv.org/abs/2007.12140]

Each tile (4×4 pixel block) maintains a plane hypothesis (disparity + two tilt parameters). The GPU update kernel iterates: for each tile, it evaluates matching cost for the current hypothesis, propagates competing hypotheses from neighbouring tiles, and selects the minimum-cost hypothesis. The propagation step runs as a sequence of parallel passes (left-to-right, right-to-left, top-to-bottom, bottom-to-top), enabling GPU execution without sequential dependencies across tiles. HITNet achieves real-time inference (>100 fps at 480×640) because there is no dense 4D correlation volume.

### 3.4 SELECTIVE-IGEV

SELECTIVE-IGEV (Wang et al., CVPR 2024) introduces iterative geometry encoding combined with selective disparity hypothesis propagation. [Source: Wang et al., "SELECTIVE-IGEV: Selective Iterative Geometry Encoding for Stereo Matching", CVPR 2024] It selectively updates only uncertain disparity regions in each GRU iteration, reducing average computation per frame while maintaining accuracy on structured surfaces.

### 3.5 GPU Correlation Layer

The 4D cost volume is the central data structure in learning-based stereo. For a batch of B images at resolution H×W with max disparity D and feature dimension C:

```
Volume shape: B × H × W × D × C  (before reduction)
After dot-product: B × H × W × D  (scalar correlation per displacement)
```

For B=1, H=540, W=960, D=256: the full volume requires 540×960×256×4 bytes ≈ 530 MB — impractical. Real implementations use one of:
- **All-pairs correlation with stride:** store only every 4th disparity, interpolating between levels
- **Grouped convolution:** correlate in groups of C/G channels to reduce peak memory by factor G
- **Sparse correlation:** evaluate only around the current GRU estimate (radius r), cost O(H×W×(2r+1))

CUDA grouped convolution via `torch.nn.functional.conv2d` with `groups=G` directly implements grouped feature correlation; HIP/ROCm exposes the same API through PyTorch.

---

## 4. RGB-D Processing

### 4.1 Normal Estimation from Depth

Surface normals can be estimated from a depth map by computing the cross-product of horizontal and vertical depth gradients. For a depth image D at pixel (u, v), the unproject function maps to 3D:

```
P(u,v) = [(u - cx)/fx * D(u,v), (v - cy)/fy * D(u,v), D(u,v)]
```

The normal at (u,v) is:

```
n = normalize( (P(u+1,v) - P(u-1,v)) × (P(u,v+1) - P(u,v-1)) )
```

This two-neighbour cross-product is computed per-pixel in a compute shader or CUDA kernel, with invalid pixels (where `D=0`) excluded from the cross-product. A more robust integral-image based approach computes gradients over a k×k window by summing depth values in the four quadrants around each pixel, reducing noise from ToF sensor jitter.

### 4.2 Hole-Filling via Bilateral Inpainting

ToF and structured-light sensors produce holes (invalid depth, `D=0`) at reflective surfaces, object boundaries, and out-of-range regions. GPU bilateral inpainting iterates: for each invalid pixel, blend valid neighbor pixels weighted by:

- Spatial Gaussian weight `exp(-|p-q|²/σ_s²)`
- Color/intensity similarity `exp(-|I(p)-I(q)|²/σ_r²)` from the co-registered RGB image

The kernel propagates valid depth outward from boundaries. Typically 4–16 iterations of this guided filter are sufficient for typical sensor hole patterns.

### 4.3 Temporal Depth Smoothing

ToF sensors exhibit temporal noise up to ±2–5 cm per-pixel. A GPU IIR (infinite impulse response) filter blends current depth with the history buffer:

```glsl
// temporal_smooth.comp
layout(binding=0) uniform sampler2D depth_current;
layout(binding=1) uniform sampler2D depth_history;
layout(binding=2, r32f) uniform writeonly image2D depth_out;

void main() {
    ivec2 coord = ivec2(gl_GlobalInvocationID.xy);
    float d_curr = texelFetch(depth_current, coord, 0).r;
    float d_hist = texelFetch(depth_history, coord, 0).r;
    float alpha  = 0.3;  // blend weight, higher = more responsive
    float d_out  = (d_curr > 0.0 && d_hist > 0.0)
                   ? mix(d_hist, d_curr, alpha)
                   : max(d_curr, d_hist);  // fallback to valid source
    imageStore(depth_out, coord, vec4(d_out));
}
```

The moving-objects case requires detecting depth discontinuities between frames before blending, using a depth-difference threshold to reset the history.

### 4.4 Depth-to-Point-Cloud GPU Kernel

Unprojecting a full depth image to a point cloud is embarrassingly parallel — one thread per pixel:

```cuda
// depth_to_pointcloud.cu (CUDA; HIP version identical with __device__ annotations)
__global__ void depth_to_pointcloud(const float* depth, float3* cloud,
                                    int W, int H,
                                    float fx, float fy, float cx, float cy) {
    int u = blockIdx.x * blockDim.x + threadIdx.x;
    int v = blockIdx.y * blockDim.y + threadIdx.y;
    if (u >= W || v >= H) return;
    float d = depth[v * W + u];
    float3 p = {0.f, 0.f, 0.f};
    if (d > 0.f) {
        p.x = (u - cx) / fx * d;
        p.y = (v - cy) / fy * d;
        p.z = d;
    }
    cloud[v * W + u] = p;
}
```

On an RTX 3080, a 1280×720 depth image unprojection completes in under 0.2 ms.

### 4.5 Structured Light Decoding

Gray code structured light projectors emit a temporal sequence of binary stripe patterns. Each pixel in the captured sequence encodes a binary index into the projector column space. GPU decoding proceeds as: for each pixel, threshold K captured frames to extract K bits, assemble into an integer index, then convert Gray code to binary. This is a K-wide bitwise operation per pixel, trivially parallel.

Phase-shifting structured light uses sinusoidal patterns (typically 3 or 4 phase steps at offset `φ = 2πk/N`). GPU phase unwrapping computes:

```glsl
float phase = atan(
    (I1 - I3),   // numerator: I_1 - I_3
    (I0 - I2)    // denominator: I_0 - I_2
);  // three-step: atan2(sqrt(3)*(I1-I3), 2*I0-I1-I3)
```

The `atan` (or `atan2`) is computed per-pixel in a fragment or compute shader, yielding the wrapped phase in [-π, π]. Phase unwrapping (removing 2π discontinuities) requires spatial continuity propagation — typically solved with a GPU flood-fill or GPU-assisted quality-guided unwrapping.

---

## 5. Structure-from-Motion GPU Stages

### 5.1 GPU SIFT

SIFT (Scale-Invariant Feature Transform) involves a Gaussian pyramid construction, difference-of-Gaussians (DoG) computation, extrema detection, orientation assignment, and 128-dimensional descriptor computation. All stages are amenable to GPU parallelisation.

**cuSIFT** (Heymann, open-source) implements SIFT entirely in CUDA: Gaussian pyramid as separable 2D convolutions, DoG as image subtraction, keypoint localisation per scale level, orientation histogram via atomic accumulation, and descriptor via oriented gradient histograms in a 4×4 grid of 8-bin histograms. [Source: cuSIFT, https://github.com/Celebrandil/CudaSift]

**PopSIFT** (AliceVision team) is a CUDA implementation of SIFT designed for photogrammetry workloads. [Source: AliceVision/popsift, https://github.com/alicevision/popsift] Note: PopSIFT is CUDA-based, not OpenCL. It achieves high throughput by using CUDA cooperative groups and shared memory for the descriptor computation kernel, and is integrated into the AliceVision/Meshroom photogrammetry pipeline. Typical throughput: 40–80 ms per megapixel image on an RTX 3070.

COLMAP's GPU feature extraction uses its own CUDA SIFT pipeline internally. [Source: COLMAP, https://github.com/colmap/colmap]

### 5.2 GPU Feature Matching

**FAISS GPU brute-force matching:** Facebook's FAISS library provides `GpuIndexFlatL2` for exact nearest-neighbour search on GPU. [Source: Johnson et al., FAISS, https://github.com/facebookresearch/faiss] For SIFT descriptor matching (128-dim float32), FAISS builds no index — `GpuIndexFlatL2` stores all database descriptors in GPU memory and performs batch matrix-multiply for all-pairs distance computation (`D = ||a - b||² = a·a + b·b - 2a·b^T`). NVIDIA V100 achieves ~100 million 128-dim comparisons/second. Lowe's ratio test filters putative matches in a post-processing kernel.

**FLANN GPU approximate nearest-neighbor:** FLANN's GPU backend builds a randomised kd-tree forest on GPU for ANN queries. For descriptor matching, exact FAISS brute-force at descriptor dimensionality 128 is typically faster than ANN approximation at the scale of thousands to tens-of-thousands of keypoints per image.

### 5.3 GPU RANSAC — USAC Framework

Geometric verification (checking whether two sets of matched keypoints are related by a fundamental matrix or homography consistent with a single rigid geometry) uses RANSAC. The USAC (Universal RANSAC) framework [Source: Barath et al., "MAGSAC++: A Fast, Reliable and Accurate Robust Estimator", CVPR 2020, https://github.com/danini/graph-cut-ransac] implements GPU-accelerated hypothesis sampling and parallel evaluation:

- **Hypothesis generation:** sample 5 (for essential matrix), 7 (fundamental), or 4 (homography) point correspondences, compute minimal solver. GPU parallelises multiple hypothesis evaluations per warp.
- **Inlier counting:** for each hypothesis, count pixels within Sampson epipolar distance threshold. This inner loop is the GPU-parallelised bottleneck: O(N_correspondences) per hypothesis, run simultaneously across hundreds of hypotheses.
- **MAGSAC++** replaces the hard inlier threshold with a soft quality weight, reducing sensitivity to threshold selection.

### 5.4 COLMAP GPU Pipeline

COLMAP (Schonberger & Frahm, CVPR 2016) [Source: https://github.com/colmap/colmap] is the standard SfM+MVS pipeline for Linux. Its GPU stages:

1. **Feature extraction:** CUDA SIFT, configurable GPU batch size (`--SiftExtraction.use_gpu 1 --SiftExtraction.gpu_index 0`).
2. **Sequential / vocabulary-tree matching:** optional GPU ANN using custom FLANN GPU integration. Vocabulary tree matching scales to large collections by querying a pre-trained visual word tree.
3. **Incremental SfM:** CPU-bound (sparse bundle adjustment via Ceres Solver). GPU is used only in the feature/matching stages.

```bash
# COLMAP GPU pipeline
colmap feature_extractor \
  --database_path db.db \
  --image_path images/ \
  --SiftExtraction.use_gpu 1 \
  --SiftExtraction.gpu_index 0

colmap exhaustive_matcher \
  --database_path db.db \
  --SiftMatching.use_gpu 1

colmap mapper \
  --database_path db.db \
  --image_path images/ \
  --output_path sparse/
```

---

## 6. Multi-View Stereo GPU Inference

### 6.1 Plane Sweep Stereo

Classical plane sweep stereo warps source images onto a set of depth hypothesis planes and evaluates per-pixel matching cost at each depth. The warp from source camera to reference camera at depth d is a homography:

```
H(d) = K_ref * R_src_ref * (I - (1/d) * T_src_ref * n^T) * K_src^{-1}
```

The GPU kernel warps each source image into the reference frame using bilinear sampling, then evaluates NCC or feature-matching cost. The result is a cost volume of shape H×W×D (D = number of depth hypotheses). Warping all N source images and all D depth planes is GPU-parallelised: each dispatch thread processes one (pixel, depth) pair.

### 6.2 MVSNet

MVSNet (Yao et al., ECCV 2018) [Source: Yao et al., "MVSNet: Depth Inference for Unstructured Multi-view Stereo", ECCV 2018, https://github.com/YoYo000/MVSNet] replaces hand-crafted cost metrics with learned deep features:

1. **2D Feature Pyramid Network:** shared-weight encoder extracts per-image feature maps at 1/4 resolution.
2. **Differentiable homography warping:** warps all source feature maps to reference camera at each depth hypothesis, producing a cost volume.
3. **Variance-based cost metric:** per-depth-hypothesis cost = variance across N warped source feature maps at each pixel.
4. **3D CNN regularisation:** a 3D encoder-decoder CNN (3D U-Net architecture) regularises the noisy per-pixel cost volume, producing a probability volume over depths.
5. **Soft argmin:** the expected depth is the weighted sum of depth hypotheses, differentiable for training.

GPU memory is the key constraint: for H=512, W=640, D=192, C_feat=32, the cost volume is 512×640×192×32×4 bytes ≈ 8 GB. MVSNet uses a cascade approach (coarse-to-fine depth hypotheses) to reduce this.

### 6.3 UniMVSNet and IterMVS

**UniMVSNet** (Peng et al., CVPR 2022) extends MVSNet with a unified cost metric that combines variance-based and concatenation-based cost volumes via a learned fusion weight. [Source: Peng et al., "Rethinking Depth Estimation for Multi-View Stereo", CVPR 2022, https://github.com/prstrive/UniMVSNet]

**IterMVS** (Wang et al., CVPR 2022) replaces the 3D CNN regulariser with a GRU-based recurrent refinement module operating on 2D feature maps — eliminating the 3D memory bottleneck and enabling higher-resolution inference. [Source: Wang et al., "IterMVS: Iterative Probability Estimation for Efficient Multi-View Stereo", CVPR 2022, https://github.com/FangjinhuaWang/IterMVS]

### 6.4 Depth Fusion from MVS

Per-view depth maps produced by MVS are noisy and may disagree at overlapping regions. Fusion proceeds by:

1. **Photo-consistency check:** for each pixel in view i, reproject to view j using the estimated depth. Accept the pixel if reprojected depth matches view j's depth within threshold T_d, and the reprojected pixel location matches within T_p pixels.
2. **Normal consistency check:** the normal at the pixel in view i must be within angular threshold T_n of the normal from view j.
3. **Point cloud merging:** accepted pixels are unprojected to 3D points. Overlapping points from multiple views are averaged or fused by confidence weighting.

This consistency check is GPU-parallelised — one thread per pixel, querying all N source views simultaneously. [Source: COLMAP mvs/fusion.cc, https://github.com/colmap/colmap]

---

## 7. NeRF Reconstruction Pipelines

### 7.1 Instant-NGP

Instant-NGP (Müller et al., SIGGRAPH 2022) [Source: Müller et al., "Instant Neural Graphics Primitives with a Multiresolution Hash Encoding", SIGGRAPH 2022, https://github.com/NVlabs/instant-ngp] enables real-time NeRF training and rendering through two innovations: multi-resolution hash encoding and tiny-cuda-nn (tcnn).

**Multi-resolution hash encoding:** the 3D position `x` is encoded at L=16 resolution levels. At each level ℓ with grid resolution N_ℓ = N_min × b^ℓ (b ≈ 1.38), the 8 corners of the enclosing voxel are hashed to T=2^19 entries in a fixed-size GPU hash table (T×F feature vectors, F=2). Trilinear interpolation combines corner features. The concatenated L×F=32-dimensional encoding is the input to the density MLP.

**Network architecture in tcnn:** the framework `tiny-cuda-nn` [Source: Müller, tiny-cuda-nn, https://github.com/NVlabs/tiny-cuda-nn] provides `tcnn::NetworkWithInputEncoding`, which fuses the hash encoding lookup and MLP forward pass into a single CUDA kernel using half-precision matrix multiply (HMMA). The density network is a 1-hidden-layer MLP (64 neurons, ReLU) producing density σ and a 16-dim bottleneck feature. The color network is a 2-hidden-layer MLP (64 neurons) receiving the bottleneck feature and view-direction spherical harmonics encoding, producing RGB.

```python
import tinycudann as tcnn

model = tcnn.NetworkWithInputEncoding(
    n_input_dims=3,
    n_output_dims=4,  # sigma + RGB
    encoding_config={
        "otype": "HashGrid",
        "n_levels": 16,
        "n_features_per_level": 2,
        "log2_hashmap_size": 19,
        "base_resolution": 16,
        "per_level_scale": 1.3819,
    },
    network_config={
        "otype": "FullyFusedMLP",
        "activation": "ReLU",
        "output_activation": "None",
        "n_neurons": 64,
        "n_hidden_layers": 1,
    },
)
```

[Source: NVlabs/instant-ngp configs/, https://github.com/NVlabs/instant-ngp/tree/master/configs]

Training converges in 5–10 seconds on an RTX 3090 (30k training steps). The training loop: generate rays from training images → stratified sample along each ray → hash encoding lookup → MLP → volume render (sum over samples weighted by transmittance and density) → MSE loss → Adam gradient → hash table update.

### 7.2 Zip-NeRF

Zip-NeRF (Barron et al., ICCV 2023) combines the anti-aliased conical frustum sampling of mip-NeRF 360 with the hash encoding speed of Instant-NGP. [Source: Barron et al., "Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields", ICCV 2023, https://github.com/jonbarron/zipnerf] Each conical frustum is approximated by a small set of multi-scale queries to the hash grid, with weights chosen to match the frustum's Gaussian approximation.

### 7.3 3D Gaussian Splatting Training

3DGS (Kerbl et al., SIGGRAPH 2023) [Source: Kerbl et al., "3D Gaussian Splatting for Real-Time Novel View Synthesis", SIGGRAPH 2023, https://github.com/graphdeco-inria/gaussian-splatting] represents scenes as a set of anisotropic 3D Gaussians optimised via differentiable rasterisation.

**Adaptive densification and pruning** is the GPU-intensive training control loop. After every K iterations (default K=100):

- **Split:** Gaussians with high positional gradient magnitude (indicating under-reconstruction of fine detail) are split into two smaller Gaussians, initialised near the parent with smaller scale.
- **Clone:** Gaussians that are too small and have high gradient are cloned and offset.
- **Prune:** Gaussians with opacity below threshold α_min are removed.

GPU gradient accumulation tracks per-Gaussian 2D position gradient magnitude using atomic adds across the training batch. The densification kernel then evaluates each Gaussian against the split/clone/prune criteria in parallel.

### 7.4 Nerfstudio on Linux

Nerfstudio [Source: Tancik et al., "Nerfstudio: A Modular Framework for Neural Radiance Field Development", SIGGRAPH 2023, https://github.com/nerfstudio-project/nerfstudio] provides a modular NeRF training framework on Linux with support for Instant-NGP (via tcnn), Gaussian Splatting, Zip-NeRF, and custom pipelines.

ROCm support for Nerfstudio is community-maintained and experimental as of mid-2026: the PyTorch ROCm backend executes the Python training loop, but `tiny-cuda-nn` requires CUDA and has no official ROCm/HIP port. Users on AMD GPUs must substitute tcnn with alternative hash encoding implementations (e.g., `msplatting` or pure-PyTorch hash grids) with 3–5× training time penalty.

---

## 8. Mesh Extraction from Implicit Representations

### 8.1 Marching Cubes from NeRF and SDF Fields

As covered in Ch210 §65, GPU Marching Cubes evaluates the scalar field at voxel corners and extracts an isosurface. Applied to NeRF/SDF:

For **NeRF → SDF**: the density field σ can be converted to an approximate SDF using the relation `s(x) = -log(σ(x))` (for sigmoid density activations). Alternatively, NeuS [Source: Wang et al., "NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction", NeurIPS 2021, https://github.com/Totoro97/NeuS] directly learns an SDF representation optimised with volume rendering. Marching Cubes is applied to the NeuS SDF field sampled on a regular grid.

For **Instant-NGP → mesh**: the NGP density field is evaluated on a 512³ grid, and MC is applied at density threshold σ_t. The GPU implementation in instant-ngp uses a compute dispatch, one thread per 2×2×2 voxel cube.

### 8.2 Mesh Extraction from 3DGS

**SuGaR** (Guédon & Lepetit, CVPR 2024) [Source: Guédon & Lepetit, "SuGaR: Surface-Aligned Gaussian Splatting for Efficient 3D Mesh Reconstruction and Real-Time Rendering", CVPR 2024, https://github.com/Anttwo/SuGaR] adds a regularisation term encouraging Gaussians to align with the underlying surface (flat pancake shapes). A level-set mesh is then extracted from the density field at the aligned surface.

**Gaussian Opacity Fields (GOF)** (Yu et al., SIGGRAPH Asia 2024) define the geometry as opacity fields computed by ray-casting through the Gaussian mixture, enabling Marching Cubes extraction without requiring Gaussians to flatten. [Source: Yu et al., "Gaussian Opacity Fields: Efficient Adaptive Surface Reconstruction in Unbounded Scenes", SIGGRAPH Asia 2024, https://github.com/autonomousvision/gaussian-opacity-fields]

**2DGS** (Huang et al., SIGGRAPH 2024) represents scenes with 2D flat Gaussian discs (surfels) oriented to lie on the scene surface. [Source: Huang et al., "2D Gaussian Splatting for Geometrically Accurate Radiance Fields", SIGGRAPH 2024, https://github.com/hbb1/2d-gaussian-splatting] Mesh extraction via alpha-compositing depth along each ray produces a dense depth map, which is then fused via TSDF or Poisson reconstruction (Ch210 §63).

### 8.3 Poisson Surface Reconstruction and Ball-Pivoting

As covered in Ch208, screened Poisson reconstruction takes an oriented point cloud and solves a global Poisson equation to produce a watertight mesh. Applied to NeRF/MVS output, the input point cloud is generated by sampling the density field and estimating normals from the field gradient.

Ball-Pivoting Algorithm (BPA) [Source: Bernardini et al., "The Ball-Pivoting Algorithm for Surface Reconstruction", IEEE TVCG 1999] reconstructs mesh directly from point clouds: a ball of radius r rolls over the point cloud surface, creating triangle faces. GPU parallelisation is challenging due to data dependencies, but GPU-accelerated BPA implementations exist in Open3D. [Source: Open3D, https://github.com/isl-org/Open3D]

**Alpha-wrapping** (CGAL 5.5+) [Source: CGAL, https://www.cgal.org/2022/10/20/cgal55/] produces watertight and self-intersection-free meshes from noisy MVS or NeRF point clouds by shrink-wrapping a sphere around the point cloud. It runs on CPU but serves as a postprocessing step producing reconstruction-pipeline-quality meshes.

---

## 9. Large-Scale and Online Reconstruction

### 9.1 Voxblox — Lightweight TSDF for Robotics

Voxblox (Oleynikova et al., IROS 2017) [Source: Oleynikova et al., "Voxblox: Incremental 3D Euclidean Signed Distance Fields for On-Board MAV Planning", IROS 2017, https://github.com/ethz-asl/voxblox] provides a CPU-optimized TSDF implementation designed for robotic embedded Linux (Jetson, Intel NUC). The sparse block structure (8³ voxel blocks) avoids allocating empty space and enables memory-mapped storage. GPU-accelerated Voxblox variants exist for NUC/Jetson platforms via CUDA, integrating depth maps at up to 30 Hz for voxels of 5–10 cm resolution.

### 9.2 NanoVDB — Sparse Voxels on GPU

OpenVDB's NanoVDB [Source: NanoVDB, https://github.com/AcademySoftwareFoundation/openvdb/tree/master/nanovdb] is a compact, GPU-native representation of the OpenVDB sparse tree. NanoVDB linearises the VDB tree into a flat buffer suitable for GPU access: leaf nodes are 8×8×8 voxel blocks; internal nodes are 16×16×16 and 32×32×32 node tables. GPU access is via read-only `nanovdb::ReadAccessor<float>`, which traverses the tree using child masks stored in GPU-accessible memory.

TSDF update on NanoVDB: a CUDA kernel iterates over all valid depth pixels, unprojections to 3D, and performs random-access voxel updates using atomics. The sparse structure avoids wasted computation in empty regions of large scenes.

### 9.3 KinectFusion-Style GPU Pipeline

KinectFusion (Newcombe et al., ISMAR 2011) established the canonical GPU RGB-D fusion pipeline:

1. **Depth preprocessing:** bilateral filtering on GPU to remove ToF noise (Ch220).
2. **TSDF integration:** project each valid depth pixel into the voxel volume and update TSDF values (Ch210 §63).
3. **Raycasting for pose tracking:** raycast the current TSDF volume to generate a synthetic depth map. This is compared to the measured depth map via point-to-plane ICP to track camera pose.
4. **ICP on GPU:** Ch211 §90 covers ICP. KinectFusion uses a GPU reduction to accumulate the 6×6 normal-equation system from per-pixel correspondences.

The original KinectFusion used a fixed 256³ voxel volume at 3.2 m range. Open-source GPU implementations include KinFu in PCL [Source: Point Cloud Library, https://github.com/PointCloudLibrary/pcl] and InfiniTAM [Source: Prisacariu et al., InfiniTAM, https://github.com/victorprad/InfiniTAM].

### 9.4 ElasticFusion

ElasticFusion (Whelan et al., RSS 2015) [Source: Whelan et al., "ElasticFusion: Dense SLAM Without A Pose Graph", RSS 2015, https://github.com/mp3guy/ElasticFusion] replaces the TSDF voxel grid with a GPU surfel-based map: each surfel is a disk (position, normal, radius, colour, confidence). Surfels are rendered via OpenGL splatting for tracking and global model maintenance. Deformation is handled by a GPU deformation graph that warps the surfel map when loop closures are detected.

### 9.5 NeuralRecon — Online Neural Reconstruction

NeuralRecon (Sun et al., CVPR 2021) [Source: Sun et al., "NeuralRecon: Real-Time Coherent 3D Reconstruction from Monocular Video", CVPR 2021, https://github.com/zju3dv/NeuralRecon] runs a transformer-based architecture per keyframe to reconstruct a local TSDF volume. Unlike offline NeRF, it processes frames sequentially with a GRU that maintains a global feature state. GPU inference (PyTorch, CUDA) runs at ~8 fps at 160×120 local TSDF resolution on an RTX 2080.

### 9.6 Large-Scale COLMAP

COLMAP GPU feature extraction and matching scales to city-level datasets by processing images in batches on GPU. For a 100k-image dataset, GPU sequential matching (overlap = 10) processes ~100k × 20 image pairs; GPU exhaustive matching is impractical beyond 10k images. The vocabulary-tree matcher (pre-trained on ImageNet/Places365) scales to millions of images by first retrieving the top-K database images for each query.

---

## 10. GPU Depth and Reconstruction in Linux Pipelines

### 10.1 libcamera → GPU Depth via DMA-BUF

libcamera [Source: https://libcamera.org/] is the Linux camera stack abstraction layer covering ISP, sensor, and pipeline configuration. Its output can be zero-copy imported to GPU memory via DMA-BUF.

The Vulkan extension `VK_EXT_external_memory_dma_buf` [Source: Khronos, https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_external_memory_dma_buf.html] enables importing a DMA-BUF file descriptor as a `VkDeviceMemory` object backing a `VkImage`. The pipeline:

```
libcamera output (DMA-BUF fd)
  → vkImportMemoryFdInfoKHR (VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT)
  → VkImage (format: VK_FORMAT_R8G8B8A8_UNORM or YCbCr)
  → Vulkan compute shader (depth inference preprocessing)
  → depth VkImage → [optional: export back via DMA-BUF to display or ROS]
```

This zero-copy path avoids CPU readback and avoids one copy per frame at 4K resolution (~33 MB for RGBA8).

### 10.2 GStreamer GPU Depth Pipeline

GStreamer's `appsink` element can deliver camera frames as CPU memory or as DMABUF handles. A GPU depth inference pipeline:

```bash
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! \
  video/x-raw,width=640,height=480,framerate=30/1 ! \
  videoconvert ! \
  appsink name=sink caps="video/x-raw,format=RGB"
```

In the application, the `appsink` callback receives buffers and passes them to ONNX Runtime (TensorRT EP or MIGraphX EP) for depth inference. The result is written to a shared CUDA/HIP buffer and posted to the next pipeline stage.

### 10.3 RealSense and Azure Kinect GPU Post-Processing

Intel RealSense SDK2 (`librealsense`) [Source: https://github.com/IntelRealSense/librealsense] provides `rs2::spatial_filter` and `rs2::temporal_filter` as CPU post-processing filters. For GPU execution, the SDK exposes raw depth frames via `rs2::frame.get_data()` for transfer to CUDA/Vulkan. The CUDA sample `pointcloud` in the SDK demonstrates GPU depth post-processing via CUDA.

Microsoft Azure Kinect SDK [Source: https://github.com/microsoft/Azure-Kinect-Sensor-SDK] (k4a) delivers both depth and IR images. The SDK runs hardware depth processing on the Kinect internal processor; GPU GPU-side hole-filling and bilateral smoothing must be implemented externally by the application (e.g., using OpenCV CUDA modules or custom Vulkan compute).

### 10.4 ROS 2 GPU Image Transport

NVIDIA Isaac ROS provides NITROS (NVIDIA Isaac ROS Type Adaptation and Negotiation for Optimized Sensors) [Source: NVIDIA Isaac ROS, https://github.com/NVIDIA-ISAAC-ROS], which implements the ROS 2 type adaptation interface (REP 2007) to pass GPU tensors between nodes without serialisation through the ROS 2 middleware. Depth images from a camera node can be shared as CUDA device pointers to a depth inference node, with zero CPU copies, over the intra-process transport.

For non-NVIDIA hardware, the `image_transport` plugin ecosystem [Source: https://github.com/ros-perception/image_transport_plugins] supports compression transports; GPU zero-copy for non-NVIDIA platforms requires custom type adaptation wrapping OpenCL or HIP buffers, which is not yet standardised in ROS 2 as of mid-2026.

### 10.5 Depth Map as VkImage for AR Overlay

A depth map produced by GPU inference can be used directly as a `VkImage` input to a Vulkan render pass for AR compositing:

1. Depth inference output: `VkImage` with format `VK_FORMAT_R32_SFLOAT`, layout `SHADER_READ_ONLY_OPTIMAL`.
2. In the AR renderer's depth pre-pass: bind the depth image as a sampler, evaluate at each virtual object fragment's screen-space coordinate to determine occlusion.
3. Alternatively: initialize the depth attachment from the estimated depth map to simulate depth test against the real scene before rendering virtual objects.

This allows virtual objects to be correctly occluded by real objects (e.g., hands, furniture) using the GPU-estimated depth — a core requirement for AR headset passthrough rendering.

---

## 11. Evaluation and Benchmarking

### 11.1 Depth Estimation Metrics

Standard metrics for monocular depth evaluation on dataset with ground-truth depth `d*` and prediction `d`:

| Metric | Formula | Notes |
|---|---|---|
| AbsRel | `mean(|d - d*| / d*)` | Scale-invariant relative error |
| SqRel | `mean((d - d*)² / d*)` | Penalises large errors |
| RMSE | `sqrt(mean((d - d*)²))` | Absolute metric error (metres) |
| RMSElog | `sqrt(mean((log d - log d*)²))` | Log-scale RMSE |
| δ<1.25 | `% pixels: max(d/d*, d*/d) < 1.25` | Threshold accuracy |

For metric depth models, AbsRel and RMSE are primary; for relative depth models, scale-aligned SqRel is used.

### 11.2 Stereo Metrics

- **EPE (End-Point Error):** `mean(|d - d*|)` in pixels of disparity.
- **D1 outlier percentage:** fraction of pixels where `|d - d*| > 3 px` and `|d - d*|/d* > 0.05`.

### 11.3 Reconstruction Metrics

- **Chamfer distance:** average of (distance from predicted to GT point cloud) and (distance from GT to predicted), measuring completeness and accuracy.
- **F-score at threshold τ:** harmonic mean of precision (fraction of predicted within τ of any GT point) and recall (fraction of GT within τ of any predicted point). τ=5 mm and τ=10 mm are common for indoor scenes.

### 11.4 GPU Benchmark Datasets

| Dataset | Task | Scale | URL |
|---|---|---|---|
| KITTI | Stereo depth, monocular depth | Outdoor driving | https://www.cvlibs.net/datasets/kitti/ |
| ETH3D | Multi-view stereo, SLAM | Indoor/outdoor | https://www.eth3d.net/ |
| ScanNet | RGB-D reconstruction | Indoor | https://github.com/ScanNet/ScanNet |
| Tanks and Temples | Large-scale reconstruction | Landmark | https://www.tanksandtemples.org/ |
| Middlebury Stereo | Stereo depth | Indoor | https://vision.middlebury.edu/stereo/ |

### 11.5 Inference Throughput

Representative inference throughput (FPS) for key methods at two resolutions, measured approximately on RTX 3090 (CUDA, TensorRT INT8 or PyTorch FP16 as noted):

| Method | 640×480 | 1280×720 | Backend |
|---|---|---|---|
| Depth Anything v2 ViT-S | ~30 fps | ~12 fps | TensorRT FP16 |
| MiDaS DPT-SwinV2-Tiny | ~35 fps | ~15 fps | TensorRT FP16 |
| RAFT-Stereo (N=7 iters) | ~25 fps | ~10 fps | PyTorch CUDA |
| HITNet | >100 fps | ~45 fps | PyTorch CUDA |
| FastDepth | >200 fps | ~80 fps | TensorRT INT8 |

*Note: FPS figures are representative; exact performance varies by GPU model, batch size, and implementation. Verify against published benchmarks.*

---

## 12. Integration with the Graphics Stack

### 12.1 Depth Map to AR Rendering

A depth map from GPU inference initialises the depth buffer for an AR render pass, enabling virtual objects to be correctly occluded by real-world geometry. The workflow:

1. Capture RGB frame → GPU depth inference → `VkImage` (D32_SFLOAT).
2. Reproject depth to NDC (multiply by projection matrix).
3. Blit reprojected depth into the depth attachment of the main framebuffer.
4. Render virtual scene geometry with standard depth test (`VK_COMPARE_OP_LESS_OR_EQUAL`).

The reprojection step must account for the offset between the depth camera and display viewpoint (eye tracking or fixed offset for passthrough cameras).

### 12.2 NeRF / 3DGS → Vulkan Rasterisation

A reconstructed 3DGS scene can be rendered in a Vulkan pipeline as described in Ch211 (§87, point cloud / Gaussian splatting). The reconstruction pipeline produces the Gaussian parameter tensor (mean, covariance, opacity, colour SH coefficients), which is uploaded to GPU memory as a `VkBuffer` for the tile-based 3DGS rasteriser. Depth is available per Gaussian as a side product of alpha-compositing: the depth of the frontmost contributing Gaussian is stored per pixel for use in depth compositing.

### 12.3 Depth for DRM/KMS Stereoscopic Output

Depth maps can drive autostereoscopic or glasses-based stereoscopic display via DRM/KMS plane configuration. The left-eye and right-eye synthetic views are generated from the depth map and a pair of virtual camera offsets (interpupillary distance), then output as two `DRM_FORMAT_*` planes to a stereo display. For 3D TVs with frame-packing format, a KMS atomic commit configures the display pipeline with `VK_FORMAT_B8G8R8A8_UNORM` side-by-side or top-and-bottom arrangement.

### 12.4 Monado OpenXR Depth Integration

Monado [Source: https://gitlab.freedesktop.org/monado/monado] is the open-source OpenXR runtime on Linux. For AR depth passthrough, the relevant OpenXR extensions include `XR_FB_passthrough` (Meta passthrough API, partially stubbed in Monado for passthrough headsets) and `XR_VARJO_environment_depth_estimation` (for Varjo hardware). A standardised cross-vendor environment depth extension (`XR_META_environment_depth`) is under development; Monado support for this extension is experimental as of mid-2026. *Note: the `XR_EXT_hand_tracking` extension covers hand joint position tracking only and is unrelated to depth map passthrough.*

For applications using RGB-D sensors as XR depth sources (e.g., robot AR overlay), depth maps from GPU inference are injected via a custom Monado driver that presents the depth buffer as an `XrSwapchainImage` in the `XR_SWAPCHAIN_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT` layout.

---

## Integrations

- **Ch210 — GPU Physics and Volumetric Methods:** TSDF fusion (§63), Marching Cubes (§65), and level-set methods are the volumetric backbone for KinectFusion-style pipelines (§9.3) and NeRF mesh extraction (§8.1). This chapter applies those algorithms to reconstruction-specific input sources (MVS depth maps, NeRF/SDF fields).

- **Ch211 — GPU Terrain, Ray Tracing, and Point Cloud:** Point cloud processing (§87), Structure-from-Motion and MVS GPU stages (§88), feature descriptors (§89), and ICP registration (§90) are the geometric underpinning of COLMAP and multi-view stereo pipelines (§5, §6). 3DGS rendering (§87) is the output representation for NeRF-style reconstruction (§7).

- **Ch220 — GPU Image Processing Algorithms:** Bilateral filtering, Gaussian pyramid construction, and image warping are the low-level GPU primitives consumed by depth post-processing (§4), feature extraction (§5.1), and cost volume warping (§6.1). Normal estimation (§4.1) is an application of gradient computation from Ch220.

- **Ch224 — 3D Shape Analysis Algorithms:** Reconstructed meshes from MVS or NeRF pipelines (§8) become the input to shape analysis: feature descriptors, symmetry detection, and segmentation for object recognition or CAD alignment.

- **Ch236 — Semantic Labels Fused into Reconstruction:** Semantic segmentation from GPU inference can be fused into TSDF or 3DGS reconstructions alongside depth, producing labelled 3D maps. This is an extension of the depth fusion pipeline (§6.4, §9) with per-point semantic labels.
