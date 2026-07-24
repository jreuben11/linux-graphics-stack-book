# Chapter 212: Python 3D ML Libraries — Open3D, PyTorch3D, and Kaolin

**Target audiences:** Graphics application developers building perception and reconstruction
systems on Linux; robot and autonomous-systems developers consuming point cloud and depth
camera output; researchers implementing inverse rendering, neural field training, and
differentiable 3D geometry pipelines.

This chapter covers three complementary Python libraries that sit above the CUDA/PyTorch
layer and below application-level 3D ML systems: **Open3D** (3D data processing, ICP
registration, reconstruction, and ML inference on point clouds); **PyTorch3D** (batched
3D data structures, differentiable rasterization, and the Implicitron NeRF framework);
and **Kaolin** (NVIDIA's 3D deep learning library — Sparse Point Cloud octrees, USD I/O,
PBR rendering, and Gaussian Splatting support). Chapter 115 covers the NeRFStudio and 3D
Gaussian Splatting training stack, including differentiable rasterization comparisons
(nvdiffrast, the Kaolin/DIBRenderer path, and PyTorch3D's soft renderer); this chapter
covers the foundational data structures and operations beneath those training loops.

---

## Table of Contents

1. [Open3D: 3D Data Processing and Reconstruction](#1-open3d-3d-data-processing-and-reconstruction)
   - [1.1 Architecture: Tensor API vs Legacy API](#11-architecture-tensor-api-vs-legacy-api)
   - [1.2 Point Cloud Processing](#12-point-cloud-processing)
   - [1.3 Global Registration: FPFH and RANSAC](#13-global-registration-fpfh-and-ransac)
   - [1.4 ICP Fine Registration](#14-icp-fine-registration)
   - [1.5 3D Reconstruction: TSDF Integration](#15-3d-reconstruction-tsdf-integration)
   - [1.6 Surface Reconstruction: Poisson](#16-surface-reconstruction-poisson)
   - [1.7 Open3D-ML: Point Cloud Segmentation and Detection](#17-open3d-ml-point-cloud-segmentation-and-detection)
   - [1.8 Visualization and Headless Rendering](#18-visualization-and-headless-rendering)
   - [1.9 What is Open3D?](#19-what-is-open3d)
   - [1.10 What is a Point Cloud?](#110-what-is-a-point-cloud)
   - [1.11 What is ICP Registration?](#111-what-is-icp-registration)
2. [PyTorch3D: Differentiable 3D Deep Learning](#2-pytorch3d-differentiable-3d-deep-learning)
   - [2.1 Data Structures: Meshes, Pointclouds, Volumes](#21-data-structures-meshes-pointclouds-volumes)
   - [2.2 Differentiable Mesh Rendering](#22-differentiable-mesh-rendering)
   - [2.3 Points Rendering](#23-points-rendering)
   - [2.4 Loss Functions](#24-loss-functions)
   - [2.5 IO Layer](#25-io-layer)
   - [2.6 Implicitron: Neural Radiance Fields](#26-implicitron-neural-radiance-fields)
3. [Kaolin: NVIDIA's 3D Deep Learning Library](#3-kaolin-nvidias-3d-deep-learning-library)
   - [3.1 Module Architecture and SurfaceMesh](#31-module-architecture-and-surfacemesh)
   - [3.2 Sparse Point Cloud (SPC) Octree Operations](#32-sparse-point-cloud-spc-octree-operations)
   - [3.3 Differentiable Rendering](#33-differentiable-rendering)
   - [3.4 USD I/O and Omniverse Integration](#34-usd-io-and-omniverse-integration)
   - [3.5 Metrics and Mesh Operations](#35-metrics-and-mesh-operations)
   - [3.6 Installation](#36-installation)
4. [trimesh: CPU Mesh Utility and NumPy Bridge](#4-trimesh-cpu-mesh-utility-and-numpy-bridge)
   - [4.1 Data Model and Format Support](#41-data-model-and-format-support)
   - [4.2 Boolean Operations](#42-boolean-operations)
   - [4.3 Ray Casting and Collision](#43-ray-casting-and-collision)
   - [4.4 NumPy Bridge to PyTorch3D and Open3D](#44-numpy-bridge-to-pytorch3d-and-open3d)
5. [Sparse 3D Convolution: MinkowskiEngine, torchsparse, and spconv](#5-sparse-3d-convolution-minkowskiengine-torchsparse-and-spconv)
   - [5.1 MinkowskiEngine](#51-minkowskiengine)
   - [5.2 torchsparse (TorchSparse++)](#52-torchsparse-torchsparse)
   - [5.3 spconv: SubMConv3d and indice_key Caching](#53-spconv-submconv3d-and-indice_key-caching)
   - [5.4 Library Comparison](#54-library-comparison)
6. [torch_geometric: Graph Neural Networks for Point Clouds](#6-torch_geometric-graph-neural-networks-for-point-clouds)
   - [6.1 Data Object and Transforms](#61-data-object-and-transforms)
   - [6.2 Conv Layers: PointNetConv, DynamicEdgeConv, PointTransformerConv](#62-conv-layers-pointnetconv-dynamicedgeconv-pointtransformerconv)
   - [6.3 PointNet++ Set Abstraction](#63-pointnet-set-abstraction)
   - [6.4 DGCNN](#64-dgcnn)
   - [6.5 Installation](#65-installation)
7. [Warp: Differentiable GPU Kernels and Geometry](#7-warp-differentiable-gpu-kernels-and-geometry)
   - [7.1 Kernel Model and Arrays](#71-kernel-model-and-arrays)
   - [7.2 Mesh Queries](#72-mesh-queries)
   - [7.3 Differentiability via wp.Tape](#73-differentiability-via-wptape)
8. [Library Comparison and Integration Patterns](#8-library-comparison-and-integration-patterns)
9. [Integrations](#9-integrations)

---

## 1. Open3D: 3D Data Processing and Reconstruction

Open3D ([open3d.org](http://www.open3d.org), Apache 2.0,
[github.com/isl-org/Open3D](https://github.com/isl-org/Open3D)) is the standard
open-source library for 3D data processing on Linux. Its C++ core implements all
algorithms; Python bindings expose a near-identical API. As of v0.19 the library ships
CUDA-accelerated paths via its Tensor API alongside the original CPU-only Legacy API.
[Source: open3d.org/docs/release/introduction.html](https://www.open3d.org/docs/release/introduction.html)

### 1.1 Architecture: Tensor API vs Legacy API

Open3D maintains two parallel geometry namespaces. The **Legacy API** (`open3d.geometry`,
`open3d.pipelines`) predates GPU support and remains CPU-only but stable. The **Tensor
API** (`open3d.t.geometry`, `open3d.t.pipelines`) is the newer GPU-capable layer:
algorithms automatically run on the device of their input tensors.

| | Legacy API | Tensor API |
|---|---|---|
| Namespace | `open3d.geometry` | `open3d.t.geometry` |
| GPU support | No | Yes — `.cuda(device_id)` |
| ICP | `pipelines.registration.registration_icp` | `t.pipelines.registration.icp` |
| TSDF | `pipelines.integration.ScalableTSDFVolume` | `t.geometry.VoxelBlockGrid` |
| Odometry | `pipelines.odometry.*` | `t.pipelines.odometry.*` |

```bash
# Standard wheel includes CUDA support when an NVIDIA GPU is present:
pip install open3d

# CPU-only (x86_64 Linux, smaller install):
pip install open3d-cpu

# Verify GPU availability:
python -c "import open3d as o3d; print(o3d.core.cuda.is_available())"
```

Python 3.8–3.12 are supported. NumPy ≥ 2.0 requires Open3D > 0.18.0.
[Source: open3d.org/docs/release/getting_started.html](https://www.open3d.org/docs/release/getting_started.html)

### 1.2 Point Cloud Processing

```python
import open3d as o3d

pcd = o3d.io.read_point_cloud("scan.ply")

# Voxel downsampling (averages points within each voxel cell)
pcd_down = pcd.voxel_down_sample(voxel_size=0.05)

# Normal estimation via PCA over k-nearest neighbours
pcd_down.estimate_normals(
    search_param=o3d.geometry.KDTreeSearchParamHybrid(radius=0.1, max_nn=30)
)
# Propagate consistent orientation (required for Poisson reconstruction and
# point-to-plane ICP)
pcd_down.orient_normals_consistent_tangent_plane(100)
```

[Source: open3d.org/docs/release/tutorial/geometry/pointcloud.html](https://www.open3d.org/docs/release/tutorial/geometry/pointcloud.html)

### 1.3 Global Registration: FPFH and RANSAC

Coarse alignment uses Fast Point Feature Histograms (FPFH) — 33-dimensional descriptors
per point — as the basis for RANSAC correspondence matching.

```python
voxel_size = 0.05
radius_feature = voxel_size * 5   # feature neighbourhood radius

# Compute 33-dim FPFH feature per point
source_fpfh = o3d.pipelines.registration.compute_fpfh_feature(
    pcd_down,
    o3d.geometry.KDTreeSearchParamHybrid(radius=radius_feature, max_nn=100)
)

distance_threshold = voxel_size * 1.5

result_ransac = o3d.pipelines.registration.registration_ransac_based_on_feature_matching(
    source_down, target_down,
    source_fpfh, target_fpfh,
    True,                          # mutual_filter
    distance_threshold,
    o3d.pipelines.registration.TransformationEstimationPointToPoint(False),
    3,                             # 3-point minimal hypothesis
    [
        o3d.pipelines.registration.CorrespondenceCheckerBasedOnEdgeLength(0.9),
        o3d.pipelines.registration.CorrespondenceCheckerBasedOnDistance(
            distance_threshold)
    ],
    o3d.pipelines.registration.RANSACConvergenceCriteria(100000, 0.999)
)
# result_ransac.transformation: (4, 4) numpy array — coarse T_source→target
# result_ransac.fitness: overlap ratio (0–1)
# result_ransac.inlier_rmse: inlier alignment error
```

For deterministic coarse alignment without RANSAC use Fast Global Registration:

```python
result_fgr = o3d.pipelines.registration.registration_fgr_based_on_feature_matching(
    source_down, target_down, source_fpfh, target_fpfh,
    o3d.pipelines.registration.FastGlobalRegistrationOption(
        maximum_correspondence_distance=distance_threshold
    )
)
```

[Source: open3d.org/docs/release/tutorial/pipelines/global_registration.html](https://www.open3d.org/docs/release/tutorial/pipelines/global_registration.html)

### 1.4 ICP Fine Registration

Point-to-plane ICP (requires normals on the target) refines the coarse RANSAC result:

```python
# Legacy API — CPU
reg = o3d.pipelines.registration.registration_icp(
    source, target,
    max_correspondence_distance=0.02,
    init=result_ransac.transformation,
    estimation_method=o3d.pipelines.registration.TransformationEstimationPointToPlane(),
    criteria=o3d.pipelines.registration.ICPConvergenceCriteria(max_iteration=2000)
)
print(reg.fitness, reg.inlier_rmse)
print(reg.transformation)  # refined 4x4 numpy array
```

The Tensor API runs the same algorithm on GPU with multi-scale scheduling:

```python
import open3d.t.pipelines.registration as treg

source_t = o3d.t.io.read_point_cloud("source.ply").cuda(0)
target_t = o3d.t.io.read_point_cloud("target.ply").cuda(0)

voxel_sizes = [0.1, 0.05, 0.025]
criteria_list = [
    treg.ICPConvergenceCriteria(1e-3, 1e-3, 50),
    treg.ICPConvergenceCriteria(1e-4, 1e-4, 30),
    treg.ICPConvergenceCriteria(1e-6, 1e-6, 14),
]
max_correspondence_distances = [0.3, 0.14, 0.07]

reg_ms = treg.multi_scale_icp(
    source_t, target_t,
    voxel_sizes, criteria_list, max_correspondence_distances,
    init_source_to_target=o3d.core.Tensor.eye(4, dtype=o3d.core.Dtype.Float64),
    estimation=treg.TransformationEstimationPointToPlane()
)
```

[Source: open3d.org/docs/release/tutorial/t_pipelines/t_icp_registration.html](https://www.open3d.org/docs/release/tutorial/t_pipelines/t_icp_registration.html)

### 1.5 3D Reconstruction: TSDF Integration

The `ScalableTSDFVolume` (Legacy API) integrates a sequence of RGB-D frames into a
TSDF map stored in GPU-friendly hash blocks:

```python
volume = o3d.pipelines.integration.ScalableTSDFVolume(
    voxel_length=4.0 / 512.0,   # ~7.8 mm per voxel
    sdf_trunc=0.04,              # truncation distance in metres
    color_type=o3d.pipelines.integration.TSDFVolumeColorType.RGB8
)

intrinsic = o3d.camera.PinholeCameraIntrinsic(
    o3d.camera.PinholeCameraIntrinsicParameters.PrimeSenseDefault
)

for (color_path, depth_path), extrinsic in zip(frames, poses):
    color = o3d.io.read_image(color_path)
    depth = o3d.io.read_image(depth_path)
    rgbd = o3d.geometry.RGBDImage.create_from_color_and_depth(
        color, depth,
        depth_trunc=4.0,
        convert_rgb_to_intensity=False
    )
    volume.integrate(rgbd, intrinsic, extrinsic)

mesh = volume.extract_triangle_mesh()
mesh.compute_vertex_normals()
```

[Source: open3d.org/docs/release/tutorial/pipelines/rgbd_integration.html](https://www.open3d.org/docs/release/tutorial/pipelines/rgbd_integration.html)

The Tensor API uses `VoxelBlockGrid`, a sparse hash-based structure that runs entirely on
CUDA when given a GPU device:

```python
vbg = o3d.t.geometry.VoxelBlockGrid(
    ('tsdf', 'weight', 'color'),
    (o3d.core.float32, o3d.core.float32, o3d.core.uint8),
    ((1,), (1,), (3,)),
    voxel_size=3.0 / 512,
    block_resolution=16,
    block_count=50000,
    device=o3d.core.Device('CUDA:0')
)

for depth_t, color_t, extrinsic_t in frames:
    frustum_block_coords = vbg.compute_unique_block_coordinates(
        depth_t, intrinsic_t, extrinsic_t,
        depth_scale=1000.0, depth_max=3.0
    )
    vbg.integrate(frustum_block_coords, depth_t, color_t,
                  intrinsic_t, extrinsic_t,
                  depth_scale=1000.0, depth_max=3.0)

mesh = vbg.extract_triangle_mesh(weight_threshold=3.0)
```

[Source: open3d.org/docs/release/python_api/open3d.t.geometry.VoxelBlockGrid.html](https://www.open3d.org/docs/release/python_api/open3d.t.geometry.VoxelBlockGrid.html)

### 1.6 Surface Reconstruction: Poisson

Poisson reconstruction fits a watertight mesh to an oriented point cloud. It requires
normals pre-computed and consistently oriented (see §1.2):

```python
import numpy as np

mesh, densities = o3d.geometry.TriangleMesh.create_from_point_cloud_poisson(
    pcd, depth=9
)
# Returns (TriangleMesh, numpy.ndarray) — the density array measures how many
# input points support each output vertex.

# Remove low-support vertices (artefacts at the reconstruction boundary):
vertices_to_remove = densities < np.quantile(densities, 0.01)
mesh.remove_vertices_by_mask(vertices_to_remove)
```

`depth` controls the octree resolution; depth=9 gives ~2 mm resolution for a 1 m scene.
[Source: open3d.org/docs/release/tutorial/geometry/surface_reconstruction.html](https://www.open3d.org/docs/release/tutorial/geometry/surface_reconstruction.html)

### 1.7 Open3D-ML: Point Cloud Segmentation and Detection

**Open3D-ML** ([github.com/isl-org/Open3D-ML](https://github.com/isl-org/Open3D-ML))
extends Open3D with a full 3D ML pipeline — dataset loaders, model zoo, and training/
inference pipelines — accessed through `open3d.ml.torch` (PyTorch backend, recommended)
or `open3d.ml.tf` (TensorFlow; requires Docker on Linux from v0.18).

| Task | Supported models |
|---|---|
| Semantic segmentation | RandLA-Net, KPConv (KPFCNN), SparseConvUnet, PointTransformer |
| Object detection | PointPillars, PointRCNN, PVCNN |

```bash
pip install open3d
pip install -r https://raw.githubusercontent.com/isl-org/Open3D-ML/main/requirements-torch-cuda.txt
```

**Inference on a standard dataset:**

```python
import open3d.ml.torch as ml3d

cfg_file = "ml3d/configs/randlanet_semantickitti.yml"
cfg = ml3d.utils.Config.load_from_file(cfg_file)

dataset  = ml3d.datasets.SemanticKITTI(dataset_path="/data/kitti", **cfg.dataset)
model    = ml3d.models.RandLANet(**cfg.model)
pipeline = ml3d.pipelines.SemanticSegmentation(
               model, dataset=dataset, device="cuda", **cfg.pipeline)

pipeline.load_ckpt(ckpt_path="randlanet_semantickitti_202010091306utc.pth")

test_split = dataset.get_split("test")
data   = test_split.get_data(0)    # {'point': ndarray, 'feat': ndarray, ...}
result = pipeline.run_inference(data)
# result['predict_labels']: (N,) int32
# result['predict_scores']: (N, num_classes) float32
```

**Inference on custom point clouds (no dataset class required):**

```python
import numpy as np

pts  = np.load("mycloud.npy")    # (N, 3) or (N, 6) with colour
data = {"point": pts[:, :3], "feat": pts[:, 3:]}

pipeline = ml3d.pipelines.SemanticSegmentation(model, device="cuda")
pipeline.load_ckpt("weights.pth")
result = pipeline.run_inference(data)
labels = result["predict_labels"]
```

[Source: github.com/isl-org/Open3D-ML](https://github.com/isl-org/Open3D-ML)

### 1.8 Visualization and Headless Rendering

The new high-level `draw()` API wraps Open3D's Filament-backed renderer (the same
renderer used by Filament, the Android/desktop PBR engine — see Chapter 83):

```python
o3d.visualization.draw(
    geometry=[{"name": "scan", "geometry": pcd},
              {"name": "mesh", "geometry": mesh}],
    title="Reconstruction",
    width=1280, height=720,
    bg_color=(0.1, 0.1, 0.1, 1.0),
    point_size=3.0,
    lookat=[0, 0, 0], eye=[0, -2, 2], up=[0, 1, 0],
    field_of_view=60.0,
    ibl="default",
)
```

**WebRTC remote visualization** streams the Filament-rendered scene to any browser —
useful for headless servers and containerised environments:

```python
import open3d.visualization.webrtc_server as webrtc_server

webrtc_server.enable_webrtc()          # sets up ICE/STUN; default port 8888
o3d.visualization.draw(geometries=[pcd])
# Connect from any browser: http://<server-ip>:8888
```

[Source: open3d.org/docs/release/tutorial/visualization/web_visualizer.html](https://www.open3d.org/docs/release/tutorial/visualization/web_visualizer.html)

**GPU-accelerated headless rendering (EGL + Filament):** the `OffscreenRenderer`
works with the standard pip wheel on Linux + NVIDIA without a display server:

```python
renderer = o3d.visualization.rendering.OffscreenRenderer(width=1280, height=720)
renderer.scene.set_background([0.1, 0.1, 0.1, 1.0])

mat = o3d.visualization.rendering.MaterialRecord()
mat.shader = "defaultUnlit"
renderer.scene.add_geometry("pcd", pcd, mat)

renderer.setup_camera(60.0, [0, 0, 0], [0, -2, 2], [0, 1, 0])
img = renderer.render_to_image()
o3d.io.write_image("output.png", img)
```

The alternative — OSMesa software rendering — requires building Open3D from source with
`-DENABLE_HEADLESS_RENDERING=ON` and supports only the legacy `Visualizer`, not the
Filament-based GUI.
[Source: open3d.org/docs/release/tutorial/visualization/headless_rendering.html](https://www.open3d.org/docs/release/tutorial/visualization/headless_rendering.html)

### 1.9 What is Open3D?

Open3D is an open-source library (Apache 2.0) for 3D data processing, reconstruction, and machine learning inference on point clouds and meshes. Its C++ core implements geometry algorithms, spatial data structures, and rendering pipelines; Python bindings expose a nearly identical API. The library occupies the layer between raw sensor input — depth cameras, LiDAR scanners, photogrammetry pipelines — and application-level perception and reconstruction systems. On the Linux graphics stack, Open3D targets CUDA-capable NVIDIA GPUs through its Tensor API, falling back to CPU paths when no GPU is available, and uses the Filament rendering engine for real-time visualization.

Open3D's principal abstractions are the `PointCloud` (an unordered set of 3D points with optional color, normal, and label attributes), the `TriangleMesh` (indexed triangle geometry), the `VoxelGrid`, and the `Image`/`RGBDImage` pair for depth-integrated sensor data. Its pipelines implement the full photogrammetric reconstruction chain: point cloud preprocessing and downsampling, feature-based global registration, ICP fine registration, TSDF volume integration, and surface reconstruction via Poisson solving. The library maintains two parallel namespaces — the original `open3d.geometry` Legacy API (CPU-only, stable) and the newer `open3d.t.geometry` Tensor API (GPU-capable via `.cuda(device_id)`) — so that algorithms run automatically on the device of their input tensors. Open3D-ML extends the library with PyTorch and TensorFlow backends for semantic segmentation and 3D object detection trained on point cloud data.
[Source: open3d.org/docs/release/introduction.html](https://www.open3d.org/docs/release/introduction.html)

### 1.10 What is a Point Cloud?

A point cloud is an unordered collection of 3D Cartesian coordinate samples, typically produced by a depth camera (structured-light or time-of-flight sensor) or a LiDAR scanner. Each point encodes at minimum an (x, y, z) position in a reference frame; practical point clouds also carry per-point color (r, g, b from an aligned RGB image), surface normals estimated from local neighborhood geometry, reflectance intensity from LiDAR return amplitude, and semantic labels assigned by a segmentation network.

Unlike a triangle mesh, a point cloud carries no explicit connectivity — there are no edges or faces, only spatial proximity relationships that algorithms compute on demand through k-nearest-neighbor or radius searches implemented in structures such as KD-trees or octrees. This makes point clouds the natural output format for range sensors: they require no assumption about the surface topology of the scene. The core operations on point clouds are downsampling (voxel averaging reduces density while preserving spatial coverage), normal estimation (PCA over local neighborhoods recovers surface orientation needed for point-to-plane ICP and Poisson reconstruction), registration (alignment of two partially overlapping clouds), and segmentation (labeling each point by semantic class or object instance). On the Linux stack, point clouds arrive from depth cameras through the librealsense or OpenNI2 SDK, are exchanged in ROS 2 as `sensor_msgs/PointCloud2`, stored as PLY or PCD files, and consumed by Open3D, PyTorch3D, or Kaolin for downstream ML inference.

### 1.11 What is ICP Registration?

Iterative Closest Point (ICP) is the standard algorithm for fine-aligning two partially overlapping 3D point clouds given an approximate initial transformation. ICP alternates two steps until convergence: for each point in the source cloud, find its nearest neighbor in the target cloud (the correspondence step); then solve for the rigid transformation — rotation R and translation t — that minimizes the sum of squared distances between corresponding pairs (the minimization step). The loop repeats until the change in transformation between iterations falls below a convergence threshold.

Two formulations dominate in practice. Point-to-point ICP minimizes the Euclidean distance between source points and their target correspondences, with a closed-form solution via singular value decomposition. Point-to-plane ICP minimizes the distance from each source point to the tangent plane at its target correspondence, requiring normals on the target; it converges in fewer iterations on smooth surfaces and is the preferred mode for depth-camera data. Robust variants weight correspondences by a Huber or Tukey function to suppress the influence of outliers from dynamic objects or sensor noise.

Because ICP is a local optimizer, it requires a sufficiently accurate initial alignment to converge to the correct solution. For large-scale misalignments, a global registration step — typically FPFH feature matching with RANSAC, or Fast Global Registration — provides the coarse transformation that ICP then refines. Open3D implements both point-to-point and point-to-plane ICP in its Legacy API (`registration_icp`) and in the GPU-capable Tensor API (`t.pipelines.registration.icp`), the latter supporting multi-scale voxel scheduling to escape shallow local minima.

---

## 2. PyTorch3D: Differentiable 3D Deep Learning

PyTorch3D (Meta, BSD 2-Clause, v0.7.9,
[github.com/facebookresearch/pytorch3d](https://github.com/facebookresearch/pytorch3d))
provides heterogeneous-batch 3D data structures, differentiable mesh and point cloud
renderers, and the Implicitron framework for neural radiance field training. It targets
inverse rendering — optimising 3D geometry or appearance by backpropagating through a
renderer — and 3D deep learning on non-uniform batches of meshes or point clouds.

**Installation.** PyTorch3D compiles C++/CUDA extensions; the cleanest path on Linux is:

```bash
# Conda (resolves PyTorch / CUDA dependency automatically):
conda install pytorch3d -c pytorch3d

# Or build from source (requires gcc/g++ ≥ 4.9 and a matching CUDA toolkit):
pip install "git+https://github.com/facebookresearch/pytorch3d.git@stable"
```

Supported PyTorch versions: 2.1.x through 2.4.x; CUDA ≥ 9.2.
[Source: github.com/facebookresearch/pytorch3d/blob/main/INSTALL.md](https://github.com/facebookresearch/pytorch3d/blob/main/INSTALL.md)

### 2.1 Data Structures: Meshes, Pointclouds, Volumes

All three structures support heterogeneous batches (elements with different vertex/point
counts) and expose three parallel tensor representations — list, padded, and packed —
between which the library converts without user involvement.

[Source: pytorch3d.readthedocs.io/en/latest/modules/structures.html](https://pytorch3d.readthedocs.io/en/latest/modules/structures.html)

**Meshes:**

```python
from pytorch3d.structures import Meshes, join_meshes_as_batch

# verts: List of (V_n, 3) float tensors; faces: List of (F_n, 3) long tensors
meshes = Meshes(verts=verts, faces=faces, textures=None)

# Padded (uniform tensor, padded with 0/-1):
meshes.verts_padded()   # (N, max_V, 3)
meshes.faces_padded()   # (N, max_F, 3)

# Packed (concatenated, no padding):
meshes.verts_packed()              # (sum(V_n), 3)
meshes.faces_packed()              # (sum(F_n), 3)
meshes.verts_packed_to_mesh_idx()  # (sum(V_n),) — mesh membership

# List form:
meshes.verts_list()    # List of (V_n, 3)
meshes.faces_list()    # List of (F_n, 3)

# Batching helpers:
batch   = join_meshes_as_batch(meshes_list)
sub     = meshes[0]                      # single-element Meshes
parts   = meshes.split([2, 3])           # List[Meshes]
```

**Pointclouds:**

```python
from pytorch3d.structures import Pointclouds

pcl = Pointclouds(
    points=points,    # List[Tensor (P_n, 3)] or (N, max_P, 3)
    normals=None,
    features=None,    # (P_n, C) — arbitrary per-point features
)

pcl.points_padded()    # (N, max_P, 3)
pcl.points_packed()    # (sum(P_n), 3)
pcl.subsample(max_points=1024)
```

**Volumes:**

```python
from pytorch3d.structures import Volumes

vol = Volumes(
    densities,                   # (N, D, sZ, sY, sX) float
    features=None,               # (N, F, sZ, sY, sX)
    voxel_size=1.0,
    volume_translation=(0., 0., 0.),
)

vol.world_to_local_coords(pts_world)   # normalize to [-1, 1]³ for grid_sample
```

### 2.2 Differentiable Mesh Rendering

The MeshRenderer pipeline is three stages: rasterize → shade → composite.

```python
from pytorch3d.renderer import (
    MeshRenderer, MeshRasterizer, RasterizationSettings,
    SoftPhongShader, FoVPerspectiveCameras, PointLights,
)

# Camera — FoV convention (game/synthetic data):
cameras = FoVPerspectiveCameras(fov=60.0, device=device, R=R, T=T)

raster_settings = RasterizationSettings(
    image_size=256,         # (H, W) or single int
    blur_radius=0.0,        # NDC distance to inflate faces; >0 enables soft gradients
    faces_per_pixel=1,      # K faces tracked per pixel (set >1 for soft rendering)
    cull_backfaces=False,
)

lights    = PointLights(device=device, location=[[0.0, 0.0, -3.0]])
renderer  = MeshRenderer(
    rasterizer=MeshRasterizer(cameras=cameras, raster_settings=raster_settings),
    shader=SoftPhongShader(device=device, cameras=cameras, lights=lights),
)

images = renderer(meshes)   # (N, H, W, 4) RGBA tensor; differentiable w.r.t. verts
```

`MeshRasterizer.forward()` returns a `Fragments` dataclass:

```
pix_to_face   (N, H, W, K) long   — face index per pixel per depth layer; -1=background
zbuf          (N, H, W, K) float  — NDC z depth
bary_coords   (N, H, W, K, 3) float — barycentric coordinates
dists         (N, H, W, K) float  — signed 2D distance to face edge
```

**Shader types:**

| Shader | Lighting | Gradient at silhouette | Use case |
|---|---|---|---|
| `SoftPhongShader` | Per-pixel Phong | Yes (soft blend) | Inverse rendering |
| `HardPhongShader` | Per-pixel Phong | No | Appearance rendering |
| `SoftGouraudShader` | Per-vertex → interpolated | Yes | Faster soft rendering |
| `HardGouraudShader` | Per-vertex → interpolated | No | Lightweight appearance |
| `SoftSilhouetteShader` | None (mask only) | Yes | Silhouette-based shape opt. |

Soft shaders blend the top-K faces per pixel weighted by distance; this requires
`blur_radius > 0` and `faces_per_pixel > 1` in `RasterizationSettings`.

**Camera models:**

```python
from pytorch3d.renderer.cameras import (
    FoVPerspectiveCameras,   # fov + near/far — synthetic/game data
    PerspectiveCameras,      # focal_length + principal_point — SfM/COLMAP data
    OrthographicCameras,
    FoVOrthographicCameras,
)

# SfM convention (in_ndc=False for screen-space focal length):
cameras = PerspectiveCameras(
    focal_length=((fx, fy),),
    principal_point=((cx, cy),),
    in_ndc=False,
    image_size=((H, W),),
    R=R, T=T, device=device
)
```

**Textures:**

```python
from pytorch3d.renderer import TexturesVertex, TexturesUV, TexturesAtlas

TexturesVertex(verts_features=verts_rgb)         # (N, V, C) per-vertex colours
TexturesUV(maps=tex_img, faces_uvs=f_uvs,
           verts_uvs=v_uvs)                      # UV-mapped textures
TexturesAtlas(atlas=atlas)                       # (N, F, R, R, C) per-face atlas
```

[Source: pytorch3d.readthedocs.io — renderer modules](https://pytorch3d.readthedocs.io/en/latest/modules/renderer/mesh/rasterizer.html)

### 2.3 Points Rendering

```python
from pytorch3d.renderer import (
    PointsRenderer, PointsRasterizer, PointsRasterizationSettings,
    AlphaCompositor,
)

raster_settings = PointsRasterizationSettings(
    image_size=256,
    radius=0.01,         # point disk radius in NDC
    points_per_pixel=8,  # compositing layers
)
renderer = PointsRenderer(
    rasterizer=PointsRasterizer(cameras=cameras,
                                raster_settings=raster_settings),
    compositor=AlphaCompositor(background_color=(1, 1, 1)),
)
images = renderer(point_clouds)    # (N, H, W, C)
```

### 2.4 Loss Functions

```python
from pytorch3d.loss import (
    chamfer_distance,
    mesh_edge_loss,
    mesh_laplacian_smoothing,
    mesh_normal_consistency,
)
```

**`chamfer_distance`** — symmetric nearest-neighbour distance between two point sets:

```python
loss, normal_loss = chamfer_distance(
    x,              # (N, P1, 3) or Pointclouds
    y,              # (N, P2, 3) or Pointclouds
    x_normals=None,
    y_normals=None,
    norm=2,                          # L1 or L2
    single_directional=False,        # x→y only if True
    batch_reduction='mean',
    point_reduction='mean',
)
```

**`mesh_edge_loss`** — penalises deviation of edge lengths from a target:

```python
loss = mesh_edge_loss(meshes, target_length=0.0)   # scalar
```

**`mesh_laplacian_smoothing`** — Laplacian regulariser with three weighting modes:

```python
# method: 'uniform' | 'cot' (cotangent) | 'cotcurv'
loss = mesh_laplacian_smoothing(meshes, method='uniform')   # scalar
```

**`mesh_normal_consistency`** — penalises dihedral angle between adjacent faces:

```python
loss = mesh_normal_consistency(meshes)   # scalar; mean(1 - cos(θ))
```

A typical deformable-mesh inverse rendering loop combines all four:

```python
optimizer = torch.optim.Adam([mesh.verts_list()[0]], lr=1e-3)
for step in range(500):
    optimizer.zero_grad()
    images     = renderer(meshes)
    loss_rgb   = ((images[..., :3] - target_rgb) ** 2).mean()
    loss_edge  = mesh_edge_loss(meshes)
    loss_lap   = mesh_laplacian_smoothing(meshes, method='uniform')
    loss_norm  = mesh_normal_consistency(meshes)
    loss = loss_rgb + 1.0 * loss_edge + 0.1 * loss_lap + 0.01 * loss_norm
    loss.backward()
    optimizer.step()
```

[Source: pytorch3d.readthedocs.io/en/latest/modules/loss.html](https://pytorch3d.readthedocs.io/en/latest/modules/loss.html)

### 2.5 IO Layer

```python
from pytorch3d.io import IO, load_objs_as_meshes

io = IO()

# Format inferred from extension; OBJ, PLY, OFF supported natively:
mesh = io.load_mesh("model.obj", device=device, include_textures=True)
io.save_mesh(mesh, "out.ply", binary=True)

pcl  = io.load_pointcloud("cloud.ply", device=device)
io.save_pointcloud(pcl, "cloud.ply")

# glTF (experimental — must register explicitly):
from pytorch3d.io.experimental_gltf_io import MeshGlbFormat
io.register_default_mesh_format(MeshGlbFormat())
mesh = io.load_mesh("scene.glb", device=device)

# Convenience function for OBJ-only batch loading:
meshes = load_objs_as_meshes(["a.obj", "b.obj"], device=device)
```

Supported formats: OBJ (with MTL textures), PLY (binary + ASCII), OFF (vertex/face
colours), glTF 2.0/GLB (experimental). STL is not supported.
[Source: pytorch3d.readthedocs.io/en/latest/modules/io.html](https://pytorch3d.readthedocs.io/en/latest/modules/io.html)

**Point cloud sampling and normal estimation:**

```python
from pytorch3d.ops import sample_points_from_meshes, estimate_pointcloud_normals

# Area-weighted Monte Carlo sampling; differentiable w.r.t. verts:
points, normals = sample_points_from_meshes(
    meshes, num_samples=10000,
    return_normals=True, return_textures=False
)   # (N, 10000, 3) each

# PCA-based normal estimation with MST orientation propagation:
normals = estimate_pointcloud_normals(
    pointclouds,
    neighborhood_size=50,
    disambiguate_directions=True,
)   # (N, P, 3)
```

### 2.6 Implicitron: Neural Radiance Fields

Implicitron is PyTorch3D's modular framework for novel-view synthesis. It is
config-driven (omegaconf/Hydra) and provides interchangeable implicit function
implementations atop a shared volumetric ray marching infrastructure.

```python
from pytorch3d.implicitron.models.implicit_function.neural_radiance_field import (
    NeuralRadianceFieldImplicitFunction,   # MLP NeRF with skip connections
    NeRFormerImplicitFunction,             # transformer encoder variant
)
from pytorch3d.implicitron.models.implicit_function.voxel_grid import (
    FullResolutionVoxelGrid,   # dense feature voxels
    VMFactorizedVoxelGrid,     # TensoRF-style vector-matrix factorization
    CPFactorizedVoxelGrid,     # CP tensor decomposition
)
```

**Ray bundle utilities:**

```python
from pytorch3d.renderer.implicit.utils import RayBundle, ray_bundle_to_ray_points
from pytorch3d.renderer.implicit.raymarching import (
    EmissionAbsorptionRaymarcher,  # standard NeRF volume rendering
    AbsorptionOnlyRaymarcher,
)

# EmissionAbsorptionRaymarcher integrates density+colour along sorted ray samples:
raymarcher = EmissionAbsorptionRaymarcher(surface_thickness=1)
# forward(rays_densities, rays_features, eps=1e-10) -> (..., feature_dim+1)
```

Implicitron's `GenericModel` wires an implicit function, a ray sampler, and a
raymarcher into a complete differentiable render-and-reconstruct pipeline. For the
full NeRF training stack on top of Implicitron see Chapter 115.
[Source: pytorch3d.readthedocs.io/en/latest/modules/implicitron/index.html](https://pytorch3d.readthedocs.io/en/latest/modules/implicitron/index.html)

---

## 3. Kaolin: NVIDIA's 3D Deep Learning Library

Kaolin (NVIDIA, Apache 2.0, v0.18.0,
[github.com/NVIDIAGameWorks/kaolin](https://github.com/NVIDIAGameWorks/kaolin)) is a
PyTorch extension library for 3D deep learning. Its most distinctive features are the
**Sparse Point Cloud (SPC)** octree data structure — enabling GPU-efficient sparse 3D
convolution and volumetric ray marching — and first-class USD I/O with Omniverse
interoperability.
[Source: kaolin.readthedocs.io/en/latest/](https://kaolin.readthedocs.io/en/latest/)

### 3.1 Module Architecture and SurfaceMesh

| Module | Role |
|---|---|
| `kaolin.ops` | GPU ops: SPC, mesh, conversions, rasterization |
| `kaolin.metrics` | Geometry similarity: Chamfer, IoU, F-score, point-to-mesh |
| `kaolin.render` | Differentiable rendering: mesh, SPC, camera, easy_render |
| `kaolin.io` | I/O: USD, OBJ, GLTF, PLY |
| `kaolin.rep` | High-level containers: `SurfaceMesh`, `Spc`, `GaussianSplatModel` |
| `kaolin.physics` | Simplicits deformable simulation (LBS + FEM, v0.16+) |

**`SurfaceMesh`** is the primary mesh container, returned by all I/O functions and
consumed by `easy_render`:

```python
from kaolin.rep import SurfaceMesh

mesh = SurfaceMesh(
    vertices,          # (V, 3) or (B, V, 3) FloatTensor
    faces,             # (F, 3) LongTensor
    uvs=None,
    face_uvs_idx=None,
    normals=None,
    vertex_colors=None,
    materials=None,
    allow_auto_compute=True  # lazy-compute normals, face areas, etc. on access
)
```

Accessing `mesh.face_normals` when only `vertices` and `faces` are provided triggers
automatic computation from the stored geometry — no explicit `compute_face_normals()`
call required.

### 3.2 Sparse Point Cloud (SPC) Octree Operations

The SPC is Kaolin's packed-octree data structure for GPU-resident sparse 3D data. It
stores occupied voxels as a `ByteTensor` in Morton (Z-curve) order, enabling O(log N)
occupancy queries and sparse 3D convolution directly on the octree without materialising
a dense grid.
[Source: kaolin.readthedocs.io/en/latest/modules/kaolin.ops.spc.html](https://kaolin.readthedocs.io/en/latest/modules/kaolin.ops.spc.html)

**Building an SPC from a point cloud:**

```python
import kaolin
import torch

pointcloud = torch.rand(1024, 3) * 2 - 1   # (N, 3) float in [-1, 1]
level = 7                                   # octree depth → 2^7 = 128 voxels/axis

# Quantize float coords to integer grid, then build octree bytes
octree = kaolin.ops.spc.unbatched_points_to_octree(
    kaolin.ops.spc.quantize_points(pointcloud, level),
    level, sorted=False
)
lengths = torch.tensor([len(octree)], dtype=torch.int32)

from kaolin.rep import Spc
spc = Spc(octrees=octree, lengths=lengths, max_level=level)
```

Or via the conversion helper:

```python
octree, lengths, features = kaolin.ops.conversions.unbatched_pointcloud_to_spc(
    pointcloud, level, features=None   # optional (N, C) per-point features
)
```

**Occupancy query:**

```python
# Auxiliary structures required by all ops:
max_level, pyramids, exsum = kaolin.ops.spc.scan_octrees(octree, lengths)
point_hierarchy = kaolin.ops.spc.generate_points(octree, pyramids, exsum)

# Query: returns octree cell index for each query point (-1 = unoccupied)
pidx = kaolin.ops.spc.unbatched_query(
    octree, exsum, query_coords, level, with_parents=False
)
```

**Sparse 3D convolution on the octree:**

```python
# SPC convolution — analogous to MinkowskiEngine but built into Kaolin
conv = kaolin.ops.spc.Conv3d(
    in_channels=32, out_channels=64,
    kernel_vectors=kv,    # (K, 3) ShortTensor of 3D offsets
    jump=0,               # 0 = same octree level
    bias=True
)
out_feats, new_level = conv(octrees, point_hierarchies, level, pyramids, exsum, feats)
```

**Volumetric ray marching through the SPC** lives in `kaolin.render.spc`:

```python
import kaolin.render.spc as spc_render

ray_index, point_index, depths = spc_render.unbatched_raytrace(
    octree, point_hierarchy, pyramid, exsum,
    origin,      # (N_rays, 3) FloatTensor
    direction,   # (N_rays, 3) unit vectors
    level,
    return_depth=True
)

# Alpha compositing over sorted ray hits:
integrated, transmittance = spc_render.exponential_integration(
    feats, tau, boundaries, exclusive=True
)
```

[Source: kaolin.readthedocs.io/en/latest/modules/kaolin.render.spc.html](https://kaolin.readthedocs.io/en/latest/modules/kaolin.render.spc.html)

### 3.3 Differentiable Rendering

Kaolin provides three rendering paths.

**`easy_render`** — one-liner PBR rendering (v0.16+, the recommended high-level API):

```python
import kaolin.render.easy_render as easy

camera  = easy.default_camera(resolution=512)
lighting = easy.default_lighting()    # SgLightingParameters (Spherical Gaussians)

render_dict = easy.render_mesh(
    camera, mesh,           # kaolin.rep.SurfaceMesh
    lighting=None,
    backend='nvdiffrast',   # or 'cuda' for the DIBRenderer path
)
image = render_dict['render']   # (H, W, 4) RGBA tensor
```

[Source: kaolin.readthedocs.io/en/latest/modules/kaolin.render.easy_render.html](https://kaolin.readthedocs.io/en/latest/modules/kaolin.render.easy_render.html)

**`kaolin.render.mesh`** — lower-level primitives with choice of rasterizer:

```python
import kaolin.render.mesh as mesh_render

# Project vertices and compute face normals
face_vertices_camera, face_vertices_image, face_normals_z = mesh_render.prepare_vertices(
    vertices, faces, camera_proj,
    camera_rot=None, camera_trans=None
)

# Hard rasterization (nvdiffrast backend — antialiased):
image_features, face_idx = mesh_render.rasterize(
    height, width,
    face_vertices_z, face_vertices_image, face_features,
    backend='nvdiffrast'
)

# Soft rasterization (DIBRenderer — differentiable through occlusion):
image_features, soft_mask, face_idx = mesh_render.dibr_rasterization(
    height, width,
    face_vertices_z, face_vertices_image, face_features, face_normals_z,
    sigmainv=7000,   # controls soft-edge sharpness
    knum=30,
    rast_backend='cuda'
)
```

**Camera model:**

```python
from kaolin.render.camera import Camera

cam = Camera.from_args(
    eye=torch.tensor([0., 0., 3.]),
    at=torch.tensor([0., 0., 0.]),
    up=torch.tensor([0., 1., 0.]),
    fov=45.0, width=512, height=512, device='cuda'
)
rays_origin, rays_dir = cam.generate_rays()
```

[Source: kaolin.readthedocs.io/en/latest/modules/kaolin.render.mesh.html](https://kaolin.readthedocs.io/en/latest/modules/kaolin.render.mesh.html)

### 3.4 USD I/O and Omniverse Integration

Kaolin's USD module supports five geometry types, animated time samples, and PBR
materials. Files written by `kaolin.io.usd` are directly readable in NVIDIA Omniverse
Composer and Pixar's USDView.

```python
import kaolin.io.usd as usd

# Mesh — returns SurfaceMesh with vertices, faces, UVs, normals, materials:
mesh = usd.import_mesh('scene.usd', scene_path='/World/Mesh')
usd.export_mesh('out.usd', scene_path='/World/Mesh',
                vertices, faces, uvs=None, normals=None, materials=None)

# Point cloud:
pc = usd.import_pointcloud('scene.usd', scene_path='/World/PC')
usd.export_pointcloud('out.usd', pointcloud, scene_path='/World/PC')

# Voxel grid:
vg = usd.import_voxelgrid('scene.usd', scene_path='/World/VG')

# 3D Gaussian Splats (v0.17+):
gs = usd.import_gaussiancloud('scene.usd', scene_path='/World/GS')
usd.export_gaussiancloud('out.usd', gaussiancloud, scene_path='/World/GS')

# Physics materials (for Simplicits deformable simulation):
usd.add_physics_material(stage, scene_path,
                         youngs_modulus=1e5, poissons_ratio=0.3, density=1000.0)
```

[Source: kaolin.readthedocs.io/en/latest/modules/kaolin.io.usd.html](https://kaolin.readthedocs.io/en/latest/modules/kaolin.io.usd.html)

### 3.5 Metrics and Mesh Operations

**`kaolin.metrics.pointcloud`:**

```python
import kaolin.metrics.pointcloud as pc_metrics

# Bidirectional Chamfer distance; returns (B,) tensor:
loss = pc_metrics.chamfer_distance(p1, p2,   # (B, N, 3), (B, M, 3)
                                   w1=1.0, w2=1.0, squared=True)

# F-score at radius r:
fscore = pc_metrics.f_score(gt_points, pred_points, radius=0.01)  # (B,)

# One-sided nearest-neighbour distances:
dists, idxs = pc_metrics.sided_distance(p1, p2)   # (B, N), (B, N)
```

**`kaolin.metrics.voxelgrid`:**

```python
import kaolin.metrics.voxelgrid as vg_metrics

iou = vg_metrics.iou(pred, gt)   # (B, X, Y, Z) binary → (B,) IoU
```

**`kaolin.metrics.trianglemesh`:**

```python
import kaolin.metrics.trianglemesh as tm_metrics

sq_dists, face_idxs, _ = tm_metrics.point_to_mesh_distance(
    pointclouds, face_vertices   # (B, N, 3), (B, F, 3, 3)
)
avg_len = tm_metrics.average_edge_length(vertices, faces)
```

**`kaolin.ops.mesh` — mesh operations:**

```python
import kaolin.ops.mesh as mesh_ops

areas   = mesh_ops.face_areas(vertices, faces)          # (B, F) or (F,)
fnorms  = mesh_ops.face_normals(face_vertices, unit=True)
vnorms  = mesh_ops.compute_vertex_normals(faces, fnorms, num_vertices)
fv_feat = mesh_ops.index_vertices_by_faces(verts_feat, faces)  # (B, F, 3, C)

points, face_idxs, feats = mesh_ops.sample_points(
    vertices, faces, num_samples=10000
)
verts_sub, faces_sub = mesh_ops.subdivide_trianglemesh(
    vertices, faces, iterations=1
)
L = mesh_ops.uniform_laplacian(num_vertices, faces)     # sparse (V, V)
```

[Source: kaolin.readthedocs.io/en/latest/modules/kaolin.ops.mesh.html](https://kaolin.readthedocs.io/en/latest/modules/kaolin.ops.mesh.html)

**Cross-representation conversions:**

```python
import kaolin.ops.conversions as conv

# Voxelgrid ↔ mesh (marching cubes):
verts, faces = conv.voxelgrids_to_trianglemeshes(voxelgrids, iso_value=0.5)

# FlexiCubes — differentiable marching cubes for shape optimisation:
fc = conv.FlexiCubes(device='cuda')
verts, faces, reg_loss = fc(
    voxelgrid_vertices, scalar_field, cube_idx, resolution,
    qef_reg_scale=0.001, weight_scale=0.99, training=True
)

# Marching Tetrahedra (DefTet):
verts, faces = conv.marching_tetrahedra(vertices, tets, sdf)
```

### 3.6 Installation

Kaolin ships CUDA-specific pre-built wheels via an S3 index — plain `pip install kaolin`
from PyPI is not supported:

```bash
# Replace TORCH_VER and CUDA_VER with your combination:
pip install kaolin==0.18.0 \
    -f https://nvidia-kaolin.s3.us-east-2.amazonaws.com/torch-2.7.0_cu126.html

# Browse the full index:
# https://nvidia-kaolin.s3.us-east-2.amazonaws.com/index.html

# Verify:
python -c "import kaolin; print(kaolin.__version__)"
```

CUDA 11.8 through 12.9 is supported with PyTorch 2.3 through 2.8.
[Source: kaolin.readthedocs.io/en/latest/notes/installation.html](https://kaolin.readthedocs.io/en/latest/notes/installation.html)

**Kaolin Wisp** ([github.com/NVIDIAGameWorks/kaolin-wisp](https://github.com/NVIDIAGameWorks/kaolin-wisp))
is a separate repository built on top of Kaolin Core that provides neural field
architectures (NGLOD, Instant-NGP hash grid NeRF), an interactive training loop, and
Kaolin SPC / ray-tracing wrappers. The repository has been in maintenance mode since
v1.0.3 (April 2023); active Instant-NGP-style work has migrated to NeRFStudio (ch115)
and `tiny-cuda-nn`.

---

## 4. trimesh: CPU Mesh Utility and NumPy Bridge

trimesh ([github.com/mikedh/trimesh](https://github.com/mikedh/trimesh), MIT, v4.x) is a
CPU-only Python library for loading, creating, and processing triangular meshes. It has no
CUDA dependency and no GPU execution path, but its clean NumPy-native data model makes it
the most convenient bridge between file formats and higher-level 3D ML libraries. Use
trimesh to load meshes, inspect mesh properties, run boolean operations, and cast rays,
then hand the raw NumPy arrays directly to PyTorch3D or Open3D without any conversion
overhead.
[Source: trimesh.org](https://trimesh.org)

### 4.1 Data Model and Format Support

```bash
# Minimal install
pip install trimesh

# With optional dependency bundle (pyembree, rtree, scipy, shapely, Pillow,
# networkx, mapbox_earcut, lxml, svg.path, cascaded_union):
pip install trimesh[easy]
```

A `trimesh.Trimesh` stores mesh data as plain NumPy arrays accessible as attributes:

```python
import trimesh
import numpy as np

mesh = trimesh.load("model.ply", force="mesh")

mesh.vertices   # (V, 3) float64 — vertex positions
mesh.faces      # (F, 3) int64   — triangle vertex indices
mesh.face_normals   # (F, 3) float64, auto-computed on first access
mesh.vertex_normals # (V, 3) float64, area-weighted average of adjacent faces
mesh.bounds     # (2, 3): [[min_x, min_y, min_z], [max_x, max_y, max_z]]
mesh.centroid   # (3,) float64 — centre of mass (uniform density)
mesh.volume     # float — signed volume (positive for watertight outward normals)
mesh.area       # float — total surface area
mesh.is_watertight  # bool — True if mesh is a closed manifold with no boundary edges
mesh.is_winding_consistent  # bool — all faces wound consistently
```

Supported file formats (no FBX or STEP):

| Category | Formats |
|---|---|
| Common 3D | STL, OBJ (+MTL), PLY, GLTF 2.0 / GLB, DAE (Collada), 3MF, OFF |
| Point cloud | XYZ, PTS |
| Robotics | URDF (loads linked meshes), SDF |
| Voxel | Binvox |
| Compressed | Draco (requires `pip install DracoPy`) |

Format is inferred from the file extension. The `force="mesh"` argument merges multi-body
scenes returned as `trimesh.Scene` into a single mesh.
[Source: github.com/mikedh/trimesh/tree/main/trimesh/exchange](https://github.com/mikedh/trimesh/tree/main/trimesh/exchange)

### 4.2 Boolean Operations

trimesh delegates boolean operations to a pluggable backend. The default and recommended
backend is **manifold3d** (a Python wrapper around the Manifold library, which implements
parallel boolean operations on GPU via thrust):

```bash
pip install manifold3d   # installs the Manifold bindings
```

```python
import trimesh

box   = trimesh.creation.box(extents=[1, 1, 1])
sphere = trimesh.creation.icosphere(radius=0.7)

# Boolean difference — subtract sphere from box
result = trimesh.boolean.difference([box, sphere],
                                    engine="manifold")   # default engine

# Boolean union and intersection
union        = trimesh.boolean.union([box, sphere], engine="manifold")
intersection = trimesh.boolean.intersection([box, sphere], engine="manifold")

print(result.is_watertight)  # True — manifold3d preserves watertightness
```

The `blender` engine is also available when Blender is on `PATH`; it supports a wider
class of non-manifold inputs but requires an external installation. The manifold3d engine
is preferred for reproducible headless environments.
[Source: github.com/mikedh/trimesh/blob/main/trimesh/boolean.py](https://github.com/mikedh/trimesh/blob/main/trimesh/boolean.py)

### 4.3 Ray Casting and Collision

trimesh provides two ray intersection backends: **pyembree** (LLVM-accelerated ray
tracing, fast) and a pure-Python fallback using scikit-spatial. The interface is
identical; install pyembree to activate it:

```bash
pip install pyembree
```

```python
import trimesh
import numpy as np

mesh = trimesh.load("scene.ply", force="mesh")
ray_caster = trimesh.ray.ray_pyembree.RayMeshIntersector(mesh)

# Cast a batch of rays
ray_origins    = np.array([[0, 0, 5], [0, 1, 5]], dtype=np.float64)
ray_directions = np.array([[0, 0, -1], [0, 0, -1]], dtype=np.float64)

locations, index_ray, index_tri = ray_caster.intersects_location(
    ray_origins, ray_directions,
    multiple_hits=True    # return all hits per ray, not just the first
)
# locations:  (N_hits, 3) float64 — world-space intersection points
# index_ray:  (N_hits,) int — which ray produced each hit
# index_tri:  (N_hits,) int — which face was hit

# First-hit only (faster):
hit_points, hit_ray_idx, hit_face_idx = ray_caster.intersects_first(
    ray_origins, ray_directions)
```

**Proximity and signed-distance queries:**

```python
# Nearest point on mesh surface to a set of query points
query_pts = np.random.rand(100, 3)
closest_points, distances, triangle_id = trimesh.proximity.closest_point(
    mesh, query_pts)
# distances: (100,) float64 — unsigned distance to nearest face

# Signed distance (negative inside a watertight mesh):
signed_dists = trimesh.proximity.signed_distance(mesh, query_pts)
```

[Source: trimesh.org/trimesh/ray](https://trimesh.org/trimesh/ray)

### 4.4 NumPy Bridge to PyTorch3D and Open3D

Because trimesh stores vertices and faces as NumPy arrays, conversion to PyTorch3D or
Open3D requires no deep copy — just a `torch.from_numpy` or `np.asarray` wrapper:

```python
import trimesh, torch, open3d as o3d
from pytorch3d.structures import Meshes
from pytorch3d.renderer import TexturesVertex

mesh = trimesh.load("model.obj", force="mesh")

# ── PyTorch3D ───────────────────────────────────────────────────────────
device = torch.device("cuda")

# vertices and faces are CPU numpy arrays; share memory with torch tensors:
verts_t = torch.from_numpy(mesh.vertices.astype("float32")).unsqueeze(0).to(device)
faces_t = torch.from_numpy(mesh.faces.astype("int64")).unsqueeze(0).to(device)

textures = TexturesVertex(
    verts_features=torch.ones_like(verts_t)  # white vertex colours
)
meshes = Meshes(verts=[verts_t[0]], faces=[faces_t[0]], textures=textures)

# ── Open3D (Legacy API) ─────────────────────────────────────────────────
o3d_mesh = o3d.geometry.TriangleMesh()
o3d_mesh.vertices = o3d.utility.Vector3dVector(mesh.vertices)
o3d_mesh.triangles = o3d.utility.Vector3iVector(mesh.faces)
o3d_mesh.compute_vertex_normals()

# ── Open3D (Tensor API — GPU) ───────────────────────────────────────────
o3d_mesh_t = o3d.t.geometry.TriangleMesh.from_legacy(o3d_mesh).cuda(0)
```

trimesh is CPU-only: any CUDA execution comes from the downstream library after the
hand-off. This makes it a lightweight preprocessing tool that imposes no GPU dependency
on systems that only need to load and inspect meshes.
[Source: github.com/mikedh/trimesh](https://github.com/mikedh/trimesh)

---

## 5. Sparse 3D Convolution: MinkowskiEngine, torchsparse, and spconv

Point clouds and voxelised 3D data are inherently sparse — even a LiDAR scan of a
cluttered scene typically occupies fewer than 5% of the cells in a bounding voxel grid.
Dense 3D convolutions waste memory and compute on empty voxels. Sparse 3D convolution
operates only on occupied voxels, representing them as a (coordinate, feature) pair list
without materialising the dense grid. Three libraries implement this on CUDA:
**MinkowskiEngine** (pioneered the research API), **torchsparse** (performance focus,
actively maintained), and **spconv** (autonomous-driving ecosystem, pip-installable).

### 5.1 MinkowskiEngine

MinkowskiEngine (NVIDIA, MIT,
[github.com/NVIDIA/MinkowskiEngine](https://github.com/NVIDIA/MinkowskiEngine)) introduced
the `SparseTensor` API adopted by subsequent libraries. Its `CoordinateManager` maintains
a per-process global sparse index cache, enabling multiple layers sharing the same
coordinate structure to reuse index computations.

**SparseTensor construction:**

```python
import MinkowskiEngine as ME
import numpy as np, torch

# Step 1: quantize continuous xyz to integer voxel indices
voxel_size = 0.05   # 5 cm
quantized_coords, feats = ME.utils.sparse_quantize(
    coordinates=raw_coords,    # (N, 3) float32 numpy array
    features=raw_feats,        # (N, C) float32; optional
    quantization_size=voxel_size,
    # quantization_mode: RANDOM_SUBSAMPLE (default), UNWEIGHTED_AVERAGE,
    #                    MAX_POOL, SPLAT_LINEAR_INTERPOLATION
)

# Step 2: add batch dimension (prepends batch index as column 0)
coords_batch = ME.utils.batched_coordinates([quantized_coords])  # single sample

# Step 3: construct SparseTensor
sinput = ME.SparseTensor(
    features=torch.from_numpy(feats).float(),
    coordinates=coords_batch,
    # minkowski_algorithm: MEMORY_EFFICIENT or SPEED_OPTIMIZED
)
```

**Key layer types** — all operate on `SparseTensor`, return `SparseTensor`:

```python
# 3D sparse convolution; stride > 1 spatially downsamples (new coordinates)
ME.MinkowskiConvolution(in_channels, out_channels,
    kernel_size=3, stride=1, dilation=1, bias=False, dimension=3)

# Transposed convolution for upsampling; stride=2 → 2× upsample
ME.MinkowskiConvolutionTranspose(in_channels, out_channels,
    kernel_size=3, stride=2, bias=False, dimension=3)

ME.MinkowskiBatchNorm(num_features)
ME.MinkowskiGlobalPooling(averaging=True)  # one vector per batch element
```

`MinkowskiFunctional` provides stateless ops: `MF.relu(x)`, `MF.sigmoid(x)`, etc.
`ME.cat()` concatenates sparse tensors sharing the same coordinate manager (skip
connections).

**Sparse UNet encoder-decoder** (adapted from
[examples/unet.py](https://github.com/NVIDIA/MinkowskiEngine/blob/master/examples/unet.py)):

```python
import MinkowskiEngine as ME
import MinkowskiEngine.MinkowskiFunctional as MF

class SparseUNet3D(ME.MinkowskiNetwork):
    def __init__(self, in_nchannel, out_nchannel, D=3):
        super().__init__(D)
        self.block1 = torch.nn.Sequential(
            ME.MinkowskiConvolution(in_nchannel, 32, kernel_size=3, dimension=D),
            ME.MinkowskiBatchNorm(32))
        self.block2 = torch.nn.Sequential(
            ME.MinkowskiConvolution(32, 64, kernel_size=3, stride=2, dimension=D),
            ME.MinkowskiBatchNorm(64))
        self.block3 = torch.nn.Sequential(
            ME.MinkowskiConvolution(64, 128, kernel_size=3, stride=2, dimension=D),
            ME.MinkowskiBatchNorm(128))
        self.block3_tr = torch.nn.Sequential(
            ME.MinkowskiConvolutionTranspose(128, 64, kernel_size=3, stride=2, dimension=D),
            ME.MinkowskiBatchNorm(64))
        self.block2_tr = torch.nn.Sequential(
            ME.MinkowskiConvolutionTranspose(128, 32, kernel_size=3, stride=2, dimension=D),
            ME.MinkowskiBatchNorm(32))
        self.final = ME.MinkowskiConvolution(64, out_nchannel, kernel_size=1, dimension=D)

    def forward(self, x):
        s1 = MF.relu(self.block1(x))
        s2 = MF.relu(self.block2(s1))
        s4 = MF.relu(self.block3(s2))
        out = MF.relu(self.block3_tr(s4))
        out = ME.cat(out, s2)           # skip connection
        out = MF.relu(self.block2_tr(out))
        out = ME.cat(out, s1)
        return self.final(out)
```

**Installation** requires OpenBLAS development headers; conda is the recommended path:

```bash
conda create -n mink python=3.8
conda activate mink
conda install openblas-devel -c anaconda
conda install pytorch=1.9.0 cudatoolkit=11.1 -c pytorch -c nvidia
pip install -U git+https://github.com/NVIDIA/MinkowskiEngine -v --no-deps \
    --install-option="--blas_include_dirs=${CONDA_PREFIX}/include" \
    --install-option="--blas=openblas"
```

CUDA ≥ 10.1 and GCC ≥ 7.4 are required. The latest PyPI release is **v0.5.4 (May
2021)** — the project is effectively unmaintained against CUDA 12.x and modern PyTorch;
new projects should prefer torchsparse or spconv.
[Source: nvidia.github.io/MinkowskiEngine](https://nvidia.github.io/MinkowskiEngine/)

### 5.2 torchsparse (TorchSparse++)

torchsparse (MIT Han Lab, MIT,
[github.com/mit-han-lab/torchsparse](https://github.com/mit-han-lab/torchsparse), v2.1.0
— branded **TorchSparse++**) achieves 4.6–4.8× training speedup over MinkowskiEngine on
A100 and RTX 2080 Ti across seven benchmarks via adaptive grouping of irregular memory
accesses. It integrates with MMDetection3D and OpenPCDet.

**SparseTensor construction:**

```python
import torchsparse
import torchsparse.nn as spnn
from torchsparse import SparseTensor
from torchsparse.utils.quantize import sparse_quantize
from torchsparse.utils.collate import sparse_collate

# Per-sample voxelization
voxel_size = 0.05
quantized_coords = sparse_quantize(raw_coords, voxel_size=voxel_size,
                                   return_index=True)   # deduplicates by floor
# Construct per-sample tensor (no batch index yet)
pc = SparseTensor(feats=feats, coords=quantized_coords)

# Batch collation: prepends per-sample batch index to coords column 0
batch = sparse_collate([pc_0, pc_1, pc_2])
```

`SparseTensor(feats, coords, stride=1, spatial_range=None)` — coordinates are `(N, 4)`
int32 with batch index in column 0 after collation.

**Key layers** (`torchsparse.nn` aliased as `spnn`):

```python
# Transposed convolution uses the same class via transposed=True
spnn.Conv3d(in_channels, out_channels, kernel_size=3, stride=1,
            padding=0, dilation=1, bias=False,
            transposed=False,    # set True for upsampling
            generative=False)    # generative transposed conv

spnn.BatchNorm(num_features)
spnn.ReLU(inplace=True)
```

Encoder-decoder example:

```python
import torch.nn as nn
import torchsparse.nn as spnn
from torchsparse import SparseTensor

class TorchsparseUNet(nn.Module):
    def __init__(self, in_ch=4, out_ch=20):
        super().__init__()
        self.encoder = nn.Sequential(
            spnn.Conv3d(in_ch, 32, kernel_size=3, stride=1),
            spnn.BatchNorm(32), spnn.ReLU(True),
            spnn.Conv3d(32, 64, kernel_size=3, stride=2),
            spnn.BatchNorm(64), spnn.ReLU(True),
        )
        self.decoder = nn.Sequential(
            spnn.Conv3d(64, 32, kernel_size=3, stride=2, transposed=True),
            spnn.BatchNorm(32), spnn.ReLU(True),
            spnn.Conv3d(32, out_ch, kernel_size=1),
        )

    def forward(self, x: SparseTensor) -> SparseTensor:
        x = self.encoder(x)
        return self.decoder(x)
```

**Installation:**

```bash
python -c "$(curl -fsSL \
  https://raw.githubusercontent.com/mit-han-lab/torchsparse/master/install.py)"

# Or from source:
pip install git+https://github.com/mit-han-lab/torchsparse.git

# Enable inference memory optimisation:
import torchsparse
torchsparse.backends.benchmark = True
```

Use `sparse_collate` as the DataLoader `collate_fn`. Mixed-precision training uses
PyTorch `amp.GradScaler` unchanged.
[Source: github.com/mit-han-lab/torchsparse](https://github.com/mit-han-lab/torchsparse)

### 5.3 spconv: SubMConv3d and indice_key Caching

spconv ([github.com/traveller59/spconv](https://github.com/traveller59/spconv), Apache
2.0, v2.3.8, December 2024) is the de facto standard backbone for LiDAR-based 3D
object detection. It distinguishes **submanifold convolution** (`SubMConv3d` — output
locations fixed to input locations) from **strided sparse convolution** (`SparseConv3d`
— may activate new voxels, used for downsampling). An `indice_key` string caches the
sparse index computation so multiple layers at the same scale share one lookup table.

**`SparseConvTensor` construction:**

```python
import spconv.pytorch as spconv   # v2.x import path

# voxel_features: (N, C) float  — non-zero voxel feature vectors
# voxel_coords:   (N, 4) int32  — [batch, z, y, x] (ZYX convention)
# grid_shape:     [Z, Y, X]     — spatial extent of the voxel grid
x = spconv.SparseConvTensor(
    features=voxel_features,
    indices=voxel_coords,
    spatial_shape=grid_shape,    # e.g. [41, 1600, 1408] for nuScenes LiDAR
    batch_size=batch_size,
)
```

Note the **ZYX coordinate convention** — index columns are `[batch, z, y, x]`, matching
the `spatial_shape=[Z, Y, X]` argument. Most LiDAR-based networks (CenterPoint,
PointPillars, SECOND) use this convention via voxelizers that output ZYX order.

**SubMConv3d vs SparseConv3d:**

| | `SubMConv3d` | `SparseConv3d` |
|---|---|---|
| Output voxel set | Same as input | May expand (activates new voxels) |
| Downsampling | No | Yes — `stride=2` halves each dim |
| `indice_key` sharing | Multiple layers at same scale can share one key | Unique key per stride level |
| `SparseInverseConv3d` | N/A | Requires matching `indice_key` for upsampling |

```python
import spconv.pytorch as spconv
import torch.nn as nn

class SpconvEncoder(nn.Module):
    def __init__(self, spatial_shape, batch_size):
        super().__init__()
        self.spatial_shape = spatial_shape
        self.batch_size = batch_size
        self.net = spconv.SparseSequential(
            # Multiple SubMConv3d layers share indice_key → one index table:
            spconv.SubMConv3d(32, 64, 3, padding=1, indice_key="subm1"),
            nn.BatchNorm1d(64), nn.ReLU(),
            spconv.SubMConv3d(64, 64, 3, padding=1, indice_key="subm1"),
            nn.BatchNorm1d(64), nn.ReLU(),
            # SparseConv3d: stride=2 downsamples, separate key:
            spconv.SparseConv3d(64, 128, 3, stride=2, padding=1, indice_key="cp1"),
            nn.BatchNorm1d(128), nn.ReLU(),
            spconv.SubMConv3d(128, 128, 3, padding=1, indice_key="subm2"),
            nn.BatchNorm1d(128), nn.ReLU(),
            spconv.ToDense(),   # materialise dense (B, C, Z, Y, X) for detection head
        )

    def forward(self, voxel_features, voxel_coords):
        x = spconv.SparseConvTensor(
            voxel_features, voxel_coords,
            self.spatial_shape, self.batch_size)
        return self.net(x)
```

`spconv.SparseSequential` is a drop-in for `nn.Sequential` that passes `SparseConvTensor`
between layers. Standard `nn.BatchNorm1d` and `nn.ReLU` operate on the `.features`
tensor inside the sparse container transparently.

Networks using spconv: **CenterPoint** (Waymo/nuScenes 3D detection), **SECOND**, **PointPillars**
backbone, **VoxelNet**, **BEVFusion**, **PartA²** — all via OpenPCDet
([github.com/open-mmlab/OpenPCDet](https://github.com/open-mmlab/OpenPCDet)).

**Installation** — CUDA-version-specific wheels, no source build required:

```bash
pip install spconv-cu118   # CUDA 11.8
pip install spconv-cu121   # CUDA 12.1
pip install spconv-cu124   # CUDA 12.4
pip install spconv-cu126   # CUDA 12.6 (latest, Dec 2024)
pip install spconv         # CPU-only

# Uninstall previous versions before upgrading:
pip uninstall spconv cumm -y
```

Python 3.7–3.13 on Linux; CUDA driver ≥ 450.82 (CUDA 11) or ≥ 520 (CUDA 11.8+).
[Source: github.com/traveller59/spconv/blob/master/docs/USAGE.md](https://github.com/traveller59/spconv/blob/master/docs/USAGE.md)

### 5.4 Library Comparison

| | MinkowskiEngine | torchsparse | spconv |
|---|---|---|---|
| SparseTensor args | `features=`, `coordinates=` | `feats=`, `coords=` | `features, indices, spatial_shape, batch_size` |
| Coordinate format | `(N, D+1)` int, batch in col 0 | `(N, 4)` int32 | `(N, D+1)` int32, **ZYX** |
| Transposed conv | Separate `MinkowskiConvolutionTranspose` | Same `Conv3d(transposed=True)` | `SparseInverseConv3d` + `indice_key` |
| SubMConv concept | Implicit via kernel design | Implicit | Explicit `SubMConv3d` vs `SparseConv3d` |
| Index caching | Auto via `CoordinateManager` | Internal | Explicit `indice_key` string |
| Batch norm | `MinkowskiBatchNorm` | `spnn.BatchNorm` | Standard `nn.BatchNorm1d` on `.features` |
| Latest release | 0.5.4 (May 2021) — **inactive** | 2.1.0 (TorchSparse++) — active | 2.3.8 (Dec 2024) — active |
| Install | Needs OpenBLAS headers, conda | Install script or pip | pip `spconv-cuXXX` wheel |
| Ecosystem | Research, semantic seg | MMDet3D, OpenPCDet | Autonomous driving, 3D detection |

---

## 6. torch_geometric: Graph Neural Networks for Point Clouds

**torch_geometric (PyG)** ([github.com/pyg-team/pytorch_geometric](https://github.com/pyg-team/pytorch_geometric),
MIT, v2.8.0, June 2026) is the standard library for graph neural networks in PyTorch,
and the primary vehicle for graph-based 3D point cloud deep learning. Where sparse conv
(§5) treats voxelised point clouds as a regular sparse 3D grid, PyG treats each point
cloud as a graph whose nodes are points and whose edges are k-NN or radius-ball
connections. This enables PointNet++, DGCNN, and Point Transformer — the architectures
that dominate non-voxelised point cloud classification and segmentation.
[Source: pytorch-geometric.readthedocs.io](https://pytorch-geometric.readthedocs.io)

### 6.1 Data Object and Transforms

The `torch_geometric.data.Data` object is a COO-format graph container:

```python
from torch_geometric.data import Data

data = Data(
    x=node_features,     # (N, F) float — per-node feature matrix; None for xyz-only
    pos=positions,       # (N, 3) float — point XYZ coordinates
    edge_index=edges,    # (2, E) LongTensor — COO adjacency (row 0=src, row 1=dst)
    edge_attr=None,      # (E, D) — per-edge features
    y=labels,            # (N,) or (1,) — node or graph labels
)
# batch: (N,) LongTensor added by DataLoader; maps each node to its graph in a minibatch
```

You rarely build `edge_index` by hand — apply transforms to `data.pos` instead:

```python
import torch_geometric.transforms as T

pre_transform = T.Compose([
    T.SamplePoints(num=1024),   # sample 1024 points uniformly from mesh faces
    T.NormalizeScale(),         # centre + normalise pos to (-1, 1)
])
transform = T.KNNGraph(k=6)     # build k-NN graph from data.pos at runtime

# Dataset usage:
dataset = ShapeNet(root="/data/shapenet",
                   pre_transform=pre_transform,
                   transform=transform)
data = dataset[0]   # edge_index is (2, 6*N) after KNNGraph
```

`T.RadiusGraph(r)` builds edges to all neighbours within radius r.
[Source: pytorch-geometric.readthedocs.io/modules/transforms.html](https://pytorch-geometric.readthedocs.io/en/latest/modules/transforms.html)

### 6.2 Conv Layers: PointNetConv, DynamicEdgeConv, PointTransformerConv

All conv layers live in `torch_geometric.nn.conv` and operate on `(x, pos, edge_index)`.

**`PointNetConv`** (the message-passing layer behind PointNet and PointNet++):

```python
from torch_geometric.nn import PointNetConv, MLP

conv = PointNetConv(
    local_nn=MLP([3 + in_ch, 64, 64]),   # maps cat(x_j, pos_j − pos_i) → features
    global_nn=MLP([64, out_ch]),          # maps aggregated features → output
    add_self_loops=False,                 # False for Set Abstraction (downsampled centroids)
)
# forward:
x_out = conv(x=(x, x_dst),           # bipartite (all, centroid subset) for SA modules
              pos=(pos, pos_dst),
              edge_index=edge_index)  # → (N_dst, out_ch)
```

**`DynamicEdgeConv`** (DGCNN — k-NN graph recomputed in feature space each forward pass):

```python
from torch_geometric.nn import DynamicEdgeConv

conv = DynamicEdgeConv(
    nn=MLP([2 * in_ch, 64, 64]),  # maps cat(x_i, x_j) for each edge → out features
    k=20,
    aggr='max',                    # 'add' | 'mean' | 'max'
)
x_out = conv(x, batch)   # edge_index recomputed internally; → (N, 64)
```

**`PointTransformerConv`** (Point Transformer — vector self-attention):

```python
from torch_geometric.nn import PointTransformerConv

conv = PointTransformerConv(
    in_channels=64,
    out_channels=64,
    pos_nn=MLP([3, 64]),        # maps pos_j − pos_i → positional encoding δ
    attn_nn=MLP([64, 64]),      # maps transformed features → attention weights α
    add_self_loops=True,
)
x_out = conv(x, pos, edge_index)   # → (N, 64)
```

**Global pooling:**

```python
from torch_geometric.nn import global_max_pool, global_mean_pool, global_add_pool

# x: (N, F), batch: (N,) — returns (num_graphs, F)
graph_emb = global_max_pool(x, batch)
```

[Source: pytorch-geometric.readthedocs.io — conv modules](https://pytorch-geometric.readthedocs.io/en/latest/modules/nn.html)

### 6.3 PointNet++ Set Abstraction

Set Abstraction (SA) = Farthest Point Sampling → radius-ball grouping → PointNetConv.
The following is adapted directly from
[examples/pointnet2_classification.py](https://github.com/pyg-team/pytorch_geometric/blob/master/examples/pointnet2_classification.py):

```python
import torch
import torch.nn.functional as F
from torch_geometric.nn import MLP, PointNetConv, fps, global_max_pool, radius

class SAModule(torch.nn.Module):
    """Set Abstraction: FPS centroid selection + radius grouping + PointNetConv."""
    def __init__(self, ratio, r, nn):
        super().__init__()
        self.ratio = ratio   # fraction of points to keep as centroids
        self.r = r           # ball-query radius
        self.conv = PointNetConv(nn, add_self_loops=False)

    def forward(self, x, pos, batch):
        # FPS: select centroids
        idx = fps(pos, batch, ratio=self.ratio)
        # radius ball query: edges from all points within r of each centroid
        row, col = radius(pos, pos[idx], self.r, batch, batch[idx],
                          max_num_neighbors=64)
        edge_index = torch.stack([col, row], dim=0)
        x_dst = None if x is None else x[idx]
        x = self.conv((x, x_dst), (pos, pos[idx]), edge_index)
        pos, batch = pos[idx], batch[idx]
        return x, pos, batch

class GlobalSAModule(torch.nn.Module):
    """Global SA: MLP on cat(features, pos) + global max pool."""
    def __init__(self, nn):
        super().__init__()
        self.nn = nn

    def forward(self, x, pos, batch):
        x = self.nn(torch.cat([x, pos], dim=1))
        x = global_max_pool(x, batch)          # → (num_graphs, F)
        pos = pos.new_zeros((x.size(0), 3))
        batch = torch.arange(x.size(0), device=batch.device)
        return x, pos, batch

class PointNet2(torch.nn.Module):
    def __init__(self, num_classes=40):
        super().__init__()
        self.sa1 = SAModule(0.5, 0.2, MLP([3, 64, 64, 128]))
        self.sa2 = SAModule(0.25, 0.4, MLP([128 + 3, 128, 128, 256]))
        self.sa3 = GlobalSAModule(MLP([256 + 3, 256, 512, 1024]))
        self.mlp = MLP([1024, 512, 256, num_classes], dropout=0.5, norm=None)

    def forward(self, data):
        sa0 = (data.x, data.pos, data.batch)
        sa1 = self.sa1(*sa0)
        sa2 = self.sa2(*sa1)
        sa3 = self.sa3(*sa2)
        return self.mlp(sa3[0]).log_softmax(dim=-1)
```

The `MLP([in, h1, h2, out])` helper is `torch_geometric.nn.MLP` — it wires BatchNorm and
ReLU between linear layers automatically. The `3` prefix in `MLP([3, ...])` is the raw
xyz dimension at the first SA level (before any features).

### 6.4 DGCNN

DGCNN (Dynamic Graph CNN) recomputes the k-NN graph in feature space at each layer,
enabling long-range information aggregation without a fixed neighbourhood radius. Adapted
from [examples/dgcnn_classification.py](https://github.com/pyg-team/pytorch_geometric/blob/master/examples/dgcnn_classification.py):

```python
from torch_geometric.nn import DynamicEdgeConv, MLP, global_max_pool
from torch.nn import Linear
import torch.nn.functional as F

class DGCNN(torch.nn.Module):
    def __init__(self, num_classes, k=20, aggr='max'):
        super().__init__()
        self.conv1 = DynamicEdgeConv(MLP([2 * 3, 64, 64, 64]), k, aggr)
        self.conv2 = DynamicEdgeConv(MLP([2 * 64, 128]), k, aggr)
        self.lin1  = Linear(128 + 64, 1024)
        self.mlp   = MLP([1024, 512, 256, num_classes], dropout=0.5, norm=None)

    def forward(self, data):
        pos, batch = data.pos, data.batch
        x1 = self.conv1(pos, batch)           # input is raw xyz; MLP takes cat(x_i, x_j)
        x2 = self.conv2(x1, batch)
        out = self.lin1(torch.cat([x1, x2], dim=1))
        out = global_max_pool(out, batch)
        return F.log_softmax(self.mlp(out), dim=1)
```

The first `DynamicEdgeConv` takes `pos` directly (not a pre-built `x`), so the MLP input
dimension is `2 * 3` — the concatenation of source and destination raw xyz. The `k`-NN
graph is built in the current feature space at each forward call.

### 6.5 Installation

```bash
# Core (pure Python, no compiled extensions):
pip install torch_geometric   # works standalone since PyG 2.3+

# Optional compiled accelerators (torch_scatter, torch_sparse, pyg_lib):
# Must match your exact PyTorch version + CUDA version.
python -c "import torch; print(torch.__version__, torch.version.cuda)"
# Example: torch 2.10 + CUDA 12.8:
pip install pyg_lib torch_scatter torch_sparse \
    -f https://data.pyg.org/whl/torch-2.10.0+cu128.html
```

The most common install failure is a version mismatch in the wheel URL — use
`--force-reinstall --no-cache-dir` when switching PyTorch versions. Prebuilt wheels
cover CUDA 12.6, 12.8, 13.0, 13.2.
Python 3.10–3.14, PyTorch 2.9+ for v2.8.0.
[Source: pytorch-geometric.readthedocs.io/install](https://pytorch-geometric.readthedocs.io/en/latest/install/installation.html)

---

## 7. Warp: Differentiable GPU Kernels and Geometry

**NVIDIA Warp** ([github.com/NVIDIA/warp](https://github.com/NVIDIA/warp),
Apache 2.0, v1.15.0, July 2026, `pip install warp-lang`) is a Python framework for
writing GPU-accelerated simulation, geometry, and ML kernels. Ordinary Python functions
annotated with `@wp.kernel` are JIT-compiled to native C++/CUDA. Warp builds in
first-class differentiability via adjoint autodiff, making it suitable for
physics-in-the-loop training and differentiable geometry pipelines.

Warp complements the higher-level libraries in this chapter: it sits at the raw-kernel
level, providing differentiable mesh SDF queries and ray casting that Open3D, PyTorch3D,
and Kaolin do not expose at this granularity.

**Requirements:** Python ≥ 3.10, CUDA-capable NVIDIA GPU + driver, CUDA 12.x or 13.
[Source: github.com/NVIDIA/warp](https://github.com/NVIDIA/warp)

### 7.1 Kernel Model and Arrays

Every `@wp.kernel` function is statically typed and JIT-compiled. Arguments must be type-
annotated with Warp types (`wp.array`, `wp.vec3`, `wp.uint64`, etc.); the kernel returns
nothing and may only write through its array arguments. `wp.tid()` returns the current
thread's linear index into the launch grid:

```python
import warp as wp
import numpy as np

@wp.kernel
def gravity_step(pos: wp.array(dtype=wp.vec3),
                 vel: wp.array(dtype=wp.vec3),
                 dt:  float):
    i = wp.tid()
    dist_sq = wp.length_sq(pos[i]) + 0.01
    acc = -1000.0 / dist_sq * wp.normalize(pos[i])
    vel[i] = vel[i] + acc * dt
    pos[i] = pos[i] + vel[i] * dt

rng = np.random.default_rng(0)
positions  = wp.array(rng.normal(size=(1024, 3)), dtype=wp.vec3, device="cuda")
velocities = wp.array(rng.normal(size=(1024, 3)), dtype=wp.vec3, device="cuda")

for _ in range(100):
    wp.launch(gravity_step, dim=1024,
              inputs=[positions, velocities, 0.01])

print(positions.numpy())   # copy to host
```

`wp.array(data, dtype, device, requires_grad)` — `.numpy()` synchronises and returns a
NumPy view; `.grad` holds the gradient when `requires_grad=True`.
[Source: github.com/NVIDIA/warp — README quickstart](https://github.com/NVIDIA/warp)

### 7.2 Mesh Queries

`wp.Mesh` wraps a triangle mesh for BVH-accelerated queries inside kernels. Pass
`mesh.id` (a `wp.uint64`) into kernel arguments — not the Python object itself.

**Closest-point and signed-distance query** (from
[examples/core/example_mesh.py](https://github.com/NVIDIA/warp/blob/main/warp/examples/core/example_mesh.py)):

```python
mesh = wp.Mesh(
    points=wp.array(vertices, dtype=wp.vec3),     # (V, 3) float32
    indices=wp.array(face_indices, dtype=int),    # (F*3,) int32 — flat triangle list
)

@wp.kernel
def sdf_query(positions: wp.array(dtype=wp.vec3),
              mesh:      wp.uint64,
              sdf_out:   wp.array(dtype=float)):
    tid = wp.tid()
    x = positions[tid]
    max_dist = 1.0e6
    query = wp.mesh_query_point_sign_normal(mesh, x, max_dist)
    if query.result:
        # Reconstruct the closest surface point
        p = wp.mesh_eval_position(mesh, query.face, query.u, query.v)
        dist = wp.length(x - p) * query.sign   # signed: negative inside
        sdf_out[tid] = dist

sdf = wp.zeros(len(query_pts), dtype=float, device="cuda")
wp.launch(sdf_query, dim=len(query_pts),
          inputs=[query_positions, mesh.id, sdf])
```

`mesh_query_point_sign_normal` returns a struct with fields:
- `query.result` — bool, True if a closest point was found within `max_dist`
- `query.face` — triangle index of the closest face
- `query.u`, `query.v` — barycentric coordinates on that face
- `query.sign` — ±1: +1 outside the mesh, −1 inside (requires a watertight mesh)

**Ray–mesh intersection** (from
[examples/core/example_raycast.py](https://github.com/NVIDIA/warp/blob/main/warp/examples/core/example_raycast.py)):

```python
@wp.kernel
def raytrace(mesh:    wp.uint64,
             origins: wp.array(dtype=wp.vec3),
             dirs:    wp.array(dtype=wp.vec3),
             hits:    wp.array(dtype=wp.vec3)):
    tid = wp.tid()
    query = wp.mesh_query_ray(mesh, origins[tid], dirs[tid], 1.0e6)
    if query.result:
        hits[tid] = query.normal * 0.5 + wp.vec3(0.5, 0.5, 0.5)
```

### 7.3 Differentiability via wp.Tape

Warp uses adjoint-mode autodiff. Mark arrays `requires_grad=True`, run launches inside a
`wp.Tape()` context, then call `tape.backward(loss)`. Gradients are read from
`tape.gradients[array]`.

The following pattern (from
[examples/optim/example_diffray.py](https://github.com/NVIDIA/warp/blob/main/warp/examples/optim/example_diffray.py))
optimises mesh vertex positions via differentiable ray tracing:

```python
# Mark differentiable:
mesh = wp.Mesh(
    points=wp.array(vertices, dtype=wp.vec3, requires_grad=True),
    indices=wp.array(face_indices, dtype=int),
)
rot = wp.array(rot_array, dtype=wp.quat, requires_grad=True)

# Forward under tape:
tape = wp.Tape()
with tape:
    wp.launch(render_kernel, dim=num_pixels,
              inputs=[mesh.id, rot, pixels, loss])

# Backward:
tape.backward(loss)

# Read gradients:
vert_grad = tape.gradients[mesh.points]   # (V, 3) gradient w.r.t. vertex positions
rot_grad  = tape.gradients[rot]
```

This pattern enables geometry-level inverse problems — surface fitting to depth maps,
physics parameter identification — that require differentiating through BVH queries, which
PyTorch autograd cannot do natively.
[Source: github.com/NVIDIA/warp/blob/main/warp/examples/optim/example_diffray.py](https://github.com/NVIDIA/warp/blob/main/warp/examples/optim/example_diffray.py)

---

## 8. Library Comparison and Integration Patterns

| | Open3D | PyTorch3D | Kaolin |
|---|---|---|---|
| Primary strength | Classical 3D processing + reconstruction + ML inference | Differentiable rendering + heterogeneous mesh/PC batches | Sparse octree ops + USD I/O + NVIDIA stack integration |
| GPU support | Tensor API (CUDA via `.cuda()`) | CUDA kernels for all ops | All ops CUDA-native |
| Differentiable renderer | None | Soft/Hard rasterizer + Implicitron | DIBRenderer, nvdiffrast, easy_render |
| Point cloud ML | Open3D-ML (RandLA-Net, KPConv…) | `estimate_pointcloud_normals`, knn_points | SPC + sparse Conv3d |
| Format I/O | PLY, PCD, OBJ, RGBD, depth | OBJ, PLY, OFF, glTF (exp.) | USD, OBJ, GLTF, PLY |
| Omniverse / USD | No | No | Yes (`kaolin.io.usd`) |
| Install | `pip install open3d` | conda or build-from-source | CUDA-specific S3 wheel |
| License | Apache 2.0 | BSD 2-Clause | Apache 2.0 |

**Common integration pattern — registration pipeline feeding a reconstruction:**

```python
# 1. Open3D: coarse RANSAC + ICP alignment
result_icp = o3d.pipelines.registration.registration_icp(
    source, target, 0.02, result_ransac.transformation,
    o3d.pipelines.registration.TransformationEstimationPointToPlane()
)

# 2. Open3D: TSDF fusion with the registered poses
volume = o3d.pipelines.integration.ScalableTSDFVolume(
    voxel_length=4.0 / 512.0, sdf_trunc=0.04,
    color_type=o3d.pipelines.integration.TSDFVolumeColorType.RGB8
)
for rgbd, extrinsic in zip(rgbd_frames, poses):
    volume.integrate(rgbd, intrinsic, extrinsic)
reconstructed_mesh = volume.extract_triangle_mesh()

# 3. PyTorch3D: load mesh, optimise texture via inverse rendering
from pytorch3d.io import IO
mesh_pt3d = IO().load_mesh("reconstructed.obj", device="cuda")
# ... differentiable render loop ...

# 4. Kaolin: export to USD for Omniverse review
import kaolin.io.usd as usd
usd.export_mesh("scene.usd", scene_path="/World/Reconstruction",
                vertices, faces)
```

**ROCm/AMD note.** Open3D's Tensor API CUDA paths and Kaolin's CUDA kernels require
NVIDIA hardware; they do not support ROCm. PyTorch3D's C++ extensions rely on
PyTorch's CUDA dispatch and likewise do not run on ROCm. For AMD GPU deployment of
these pipelines, the CPU-only Open3D path (Legacy API or `open3d-cpu` wheel) is the
only officially supported option.

---

## 9. Integrations

- **Chapter 115 (NeRFStudio, Neural Radiance Fields, and 3D Gaussian Splatting)**: §14
  of ch115 covers differentiable rendering comparisons — nvdiffrast, PyTorch3D's soft
  renderer, and Kaolin's DIBRenderer. §16 (threestudio) uses the DMTet geometry backend
  whose FlexiCubes conversion is described in §3.5 here. §17–18 (DUSt3R/MASt3R) produce
  dense `(H, W, 3)` pointmaps that feed directly into Open3D Poisson reconstruction
  (§1.6) and PyTorch3D `Pointclouds` (§2.1). The spconv/torchsparse sparse conv
  backbones in §5 are the GPU architecture underpinning LiDAR 3D detection models in
  ch115 §5 and in Open3D-ML (§1.7 here). The `torch_geometric` `Data` object (§6) and
  Warp mesh queries (§7) apply directly to the point cloud outputs of DUSt3R/MASt3R
  global alignment and the surfel models in 2DGS (ch115 §20).
- **Chapter 114 (OpenCV and GPU-Accelerated Computer Vision)**: OpenCV's CUDA point
  cloud ops, `cv::viz` for 3D visualisation, and the RGBD depth processing pipeline
  that produces the frames consumed by Open3D's TSDF integrator.
- **Chapter 211 (ROS 2 Multimodal Sensor and Perception Pipeline)**: The
  `sensor_msgs/PointCloud2` streams from LiDAR and RGBD cameras in ch211 are the raw
  input to Open3D-ML's inference pipelines and to the RANSAC+ICP registration described
  here. `pcl::fromROSMsg` / `pcl::toROSMsg` or Open3D's `o3d.t.io.read_point_cloud`
  bridge the ROS and Open3D worlds.
- **Chapter 210 (SLAM Theory and State of the Art)**: FAST-LIO2, LIO-SAM, and
  Cartographer produce registered point cloud sequences — exactly the input required by
  Open3D's `ScalableTSDFVolume` for dense reconstruction. Kaolin's SPC octree is the
  GPU-side data structure for the kind of sparse voxel maps those systems maintain.
- **Chapter 209 (OpenSLAM)**: G2O graph optimisation produces the pose graph whose
  vertex poses feed directly into the TSDF integration loop in §1.5.
- **Chapter 48 (ROCm and Machine Learning on Linux GPUs)**: ROCm/HIP as the AMD
  alternative for the PyTorch training stack underlying Open3D-ML and PyTorch3D; the
  limitations noted in §4 apply when porting these libraries to AMD hardware.
- **Chapter 83 (Filament)**: Open3D's GUI renderer — `draw()`, `OffscreenRenderer`,
  and WebRTC visualisation — is built on the Filament physically-based renderer. The
  material system and IBL lighting in Open3D's visualiser are Filament's PBR pipeline
  accessed through the Open3D C++ bindings.
