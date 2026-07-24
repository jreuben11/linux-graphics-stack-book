# Chapter 209: OpenSLAM — Classical and Graph-Based SLAM on the Linux Stack

> **Part**: Part VII-B — Multimedia Frameworks and Desktop Integration
> **Audience**: Graphics application developers (GPU compute, sensor fusion), embedded Linux engineers, XR/robotics developers
> **Status**: First draft — 2026-07-20

---

## Table of Contents

1. [Overview](#1-overview)
2. [SLAM Fundamentals: The Front-End/Back-End Split](#2-slam-fundamentals-the-front-endback-end-split)
3. [The OpenSLAM Repository and Its Scope](#3-the-openslam-repository-and-its-scope)
4. [Particle Filter SLAM: GMapping](#4-particle-filter-slam-gmapping)
5. [Pose Graph Optimization: G2O](#5-pose-graph-optimization-g2o)
6. [TORO: Tree-Based Network Optimizer](#6-toro-tree-based-network-optimizer)
7. [Laser Feature Detection: FLIRTLib](#7-laser-feature-detection-flirtlib)
8. [RGB-D SLAM and the HOG-Man Optimizer](#8-rgb-d-slam-and-the-hog-man-optimizer)
9. [Visual SLAM on OpenSLAM: ORB-SLAM and Successors](#9-visual-slam-on-openslam-orb-slam-and-successors)
10. [Visual-Inertial Odometry: Basalt and Monado's XR Tracking Pipeline](#10-visual-inertial-odometry-basalt-and-monadoxr-tracking-pipeline)
11. [The Linux Camera Pipeline as SLAM Input](#11-the-linux-camera-pipeline-as-slam-input)
12. [GPU Acceleration Strategies for SLAM Workloads](#12-gpu-acceleration-strategies-for-slam-workloads)
13. [ROS 2 and the Linux Robotics Integration Layer](#13-ros-2-and-the-linux-robotics-integration-layer)
14. [Integrations](#14-integrations)

---

## 1. Overview

Simultaneous Localisation and Mapping (SLAM) is the computational problem of building a map of an unknown environment while concurrently tracking a sensor platform's pose within that map. It is a foundational problem in robotics and, increasingly, in extended reality: every modern inside-out XR headset running on Linux uses a SLAM-derived algorithm to track the wearer's position without external infrastructure.

[OpenSLAM.org](https://openslam-org.github.io/) was founded in 2006 as a publication platform for SLAM researchers, providing open-source implementations that would otherwise remain buried in academic repositories. Migrated to GitHub in 2018, it hosts around thirty distinct algorithm implementations spanning laser range, RGB-D, and camera-based mapping, together with optimisation back-ends that are now embedded in widely-used open-source robotics stacks.

### Audiences

This chapter serves three overlapping groups:

- **Graphics application developers** who are integrating SLAM into XR pipelines or GPU-compute workflows and need to understand how the classical algorithm building blocks — particle filters, factor graphs, pose-graph optimisation — connect to Linux GPU APIs.
- **Embedded Linux engineers** building robot mapping systems or autonomous vehicle stacks on top of V4L2, libcamera, and GPU compute runtimes.
- **XR and OpenXR developers** building on Monado who need to understand how inside-out tracking is implemented via visual-inertial odometry and which Linux-stack components supply sensor data.

GPU-accelerated successors to the original OpenSLAM algorithms — ORB-SLAM3 with CUDA front-end, DROID-SLAM, and MonoGS — are covered in depth in Chapter 114 §14. This chapter focuses on the algorithmic foundations and their integration with the Linux sensor and compute stack.

### OpenSLAM's Position in the Stack

```
Camera / LiDAR Sensor
        │
  V4L2 / libcamera / ROS 2 sensor drivers
        │
  ┌─────▼────────────────────────────────────┐
  │  SLAM Front-End                          │
  │  Feature extraction · Scan matching ·   │
  │  IMU preintegration · Data association  │
  └─────┬────────────────────────────────────┘
        │  Pose constraints / measurements
  ┌─────▼────────────────────────────────────┐
  │  SLAM Back-End                           │
  │  GMapping · G2O · TORO · HOG-Man         │
  │  Pose graph optimisation · Bundle adj.   │
  └─────┬────────────────────────────────────┘
        │  Trajectory + Map
  ┌─────▼────────────────────────────────────┐
  │  Consumers                               │
  │  Monado / OpenXR · ROS 2 nav2 · Viz      │
  └──────────────────────────────────────────┘
```

### 1.1 What is SLAM?

SLAM (Simultaneous Localisation and Mapping) is the problem of building a consistent map of an unknown environment while simultaneously estimating the sensor platform's pose within that map. The challenge is circular: accurate mapping requires knowing where the sensor is, and accurate localisation requires knowing the map. Classical solutions break this circularity probabilistically by maintaining a joint distribution over robot pose and map, updated incrementally as new sensor measurements arrive.

In the Linux graphics and XR context, SLAM is the enabling technology for inside-out tracking on XR headsets: a headset running Monado uses camera feeds from the V4L2 or libcamera stack to estimate its own position in a room without external trackers. In robotics, SLAM provides the map and trajectory that autonomous navigation (ROS 2 nav2) needs to plan collision-free paths. The two principal algorithmic families — recursive Bayesian filtering and pose-graph optimisation — trade completeness of the posterior for scalability, and each has different GPU-acceleration characteristics that connect to the Vulkan compute and OpenCL runtimes available on the Linux stack.

### 1.2 What is OpenSLAM?

OpenSLAM (openslam-org.github.io) is an open-source repository and publication platform for SLAM algorithm implementations. Founded in 2006 and migrated to GitHub in 2018, it provides reference C++ implementations of approximately thirty distinct SLAM algorithms, including particle-filter approaches (GMapping, FastSLAM variants), pose-graph optimisers (G2O, TORO, HOG-Man), feature extractors for laser range data (FLIRTLib), and early RGB-D and visual SLAM systems.

OpenSLAM occupies a specific niche in the Linux robotics stack: it sits between the sensor drivers (V4L2, libcamera, urg_node for LiDAR) and the navigation consumers (ROS 2 nav2, Monado's SLAM tracking module). Its implementations are research-grade code that became the algorithmic cores of production-grade packages — G2O is embedded in ORB-SLAM3 and Google Cartographer. Most OpenSLAM packages carry BSD or LGPL licenses compatible with embedding in larger open-source stacks, though some carry GPLv3 restrictions that affect how they can be combined with proprietary middleware. On Linux, the build toolchain for OpenSLAM components is CMake-based, with external dependencies on Eigen, SuiteSparse, and optionally Qt for visualisation.

### 1.3 What is Pose Graph Optimization?

A pose graph is a directed graph in which each vertex represents a sensor platform pose (position and orientation) at a specific point in time, and each edge encodes a noisy relative-pose measurement between two poses — either an odometry measurement linking consecutive poses, or a loop-closure constraint linking two non-adjacent poses identified as returning to the same physical location. The SLAM back-end solves a nonlinear least-squares problem over the graph to find the pose assignment that best satisfies all constraints simultaneously, in a maximum-likelihood sense under Gaussian noise.

Pose graph optimisation superseded Kalman-filter SLAM for large-scale environments because the sparse factor graph structure enables efficient Cholesky factorisation via SuiteSparse, scaling to hundreds of thousands of poses where EKF methods would require O(n²) storage. The optimisation iterates Gauss-Newton or Levenberg-Marquardt steps on the graph Jacobian, converging in a small number of iterations when initialised from odometry estimates. On the Linux stack, pose graph optimisers (G2O, TORO, HOG-Man) link against SuiteSparse or Eigen for the linear algebra back-end, and their integration with GPU compute is indirect: the front-end producing the constraints may use Vulkan compute or CUDA for feature extraction, while the optimiser itself runs on CPU sparse linear algebra.

---

## 2. SLAM Fundamentals: The Front-End/Back-End Split

Every SLAM system separates two concerns that run at very different rates and with very different computational profiles.

**The front-end** processes raw sensor data at sensor rate (typically 10–60 Hz for cameras, 10–40 Hz for LiDAR). Its job is to answer the question: *given this new observation, what is the most likely relative motion from the previous pose?* Front-end tasks include:

- Keypoint detection and descriptor extraction (ORB, SURF, SIFT for cameras; FLIRT interest points for laser scans)
- Feature matching and outlier rejection (RANSAC, nearest-neighbour search)
- IMU preintegration (integrating gyroscope and accelerometer measurements between camera frames)
- Depth association for RGB-D sensors (aligning colour pixels to depth measurements)

**The back-end** receives the sequence of noisy relative-pose constraints produced by the front-end and seeks the globally consistent trajectory and map. It runs asynchronously, often at 1–5 Hz for real-time SLAM. Two major paradigms dominate:

- **Recursive Bayes filtering** — the robot pose is represented as a probability distribution and updated incrementally using a Bayesian filter. Particle filters (GMapping, FastSLAM) and Kalman-family filters (EKF-SLAM, CEKF-SLAM from OpenSLAM) belong here. Particle filters are particularly effective for 2D laser SLAM.
- **Pose graph optimisation** — all estimated poses and the constraints between them are encoded as a factor graph; a nonlinear least-squares solver minimises the total error. G2O, TORO, and HOG-Man are all graph optimisers. This paradigm scales better than filtering to large environments and is the dominant approach in modern visual SLAM.

The separation matters for GPU integration: the front-end is embarrassingly parallel (process many features per frame) and benefits directly from GPU compute; the back-end involves sparse linear algebra (Cholesky, Schur complement) that is often more efficiently handled by optimised CPU libraries such as SuiteSparse, with GPU acceleration adding value mainly at very large scale.

---

## 3. The OpenSLAM Repository and Its Scope

OpenSLAM hosts implementations spanning the full spectrum of SLAM approaches. The table below lists the algorithms most relevant to the Linux graphics and compute stack, grouped by modality.

| Algorithm | Modality | Back-End | License | Status |
|-----------|----------|----------|---------|--------|
| GMapping | 2D Laser | Particle filter | LGPL v2 | Widely deployed (ROS `slam_gmapping`) |
| G2O | Any (library) | Pose graph | BSD/LGPL v3 | Active; embedded in ORB-SLAM3, Cartographer |
| TORO | 2D/3D Laser | Pose graph (tree) | LGPL | Superseded by G2O |
| FLIRTLib | 2D Laser front-end | Library | LGPL | Stable |
| HOG-Man | 3D | Pose graph | LGPL | Superseded by G2O |
| RGBDSlam | RGB-D | HOG-Man | BSD | Historically significant |
| ORB-SLAM | Monocular | G2O | GPLv3 | Superseded by ORB-SLAM3 (Ch114 §14) |
| EKFMonoSLAM | Monocular | EKF | GPLv3 | Research baseline |
| COP-SLAM | Laser | Constraint-based | LGPL | Niche |
| tinySLAM | 2D Laser | Particle filter | BSD | Embedded / microcontroller use |

[Source: openslam-org.github.io, github.com/OpenSLAM-org](https://openslam-org.github.io/)

---

## 4. Particle Filter SLAM: GMapping

GMapping implements a **Rao-Blackwellized particle filter** (RBPF) for 2D laser SLAM. The Rao-Blackwellization insight is that, conditioned on the robot trajectory, the grid map becomes a collection of independent binary variables (each cell is either occupied or free). This factorisation allows each particle to carry its own private map, making the filter exact rather than approximate.

### Algorithm Sketch

Each of the *N* particles represents one hypothesis for the robot trajectory. At each laser scan *t*:

1. **Sampling**: propagate each particle's pose using the odometry motion model, injecting Gaussian noise proportional to the control uncertainty.
2. **Importance weighting**: score each particle by how well the laser scan matches the particle's private map. The improved proposal distribution in GMapping conditions on the laser observation *before* sampling, reducing the variance of the particle weights and allowing fewer particles for the same map quality.
3. **Resampling**: when the effective number of particles *N_eff* falls below a threshold, resample using low-variance resampling to avoid weight collapse.
4. **Map update**: update the winning particle's occupancy grid using a log-odds model.

```cpp
/* GMapping Particle — gridfastslam/gridslamprocessor.h (simplified; actual struct
 * also includes previousPose, weightSum, gweight, previousIndex, TNode*) */
struct Particle {
    ScanMatcherMap map;       /* private occupancy grid, one per particle */
    OrientedPoint  pose;      /* current pose estimate (x, y, theta) */
    double         weight;    /* importance weight for resampling */
};

/* Actual ScanMatcher::likelihood signature — scanmatcher/scanmatcher.h:
 *   double likelihood(double& lmax, OrientedPoint& mean, CovarianceMatrix& cov,
 *                     const ScanMatcherMap& map, const OrientedPoint& p,
 *                     const double* readings);
 *
 * The function evaluates how well the laser readings from pose p agree with
 * the particle's private map. Internally it integrates log-odds cell scores
 * over the beam endpoints, accumulates the maximum and covariance of the
 * likelihood surface, and returns the total log-likelihood. */
```

[Source: github.com/OpenSLAM-org/openslam_gmapping/blob/master/gridfastslam/gridslamprocessor.h; github.com/ros-perception/openslam_gmapping/blob/master/include/gmapping/scanmatcher/scanmatcher.h](https://github.com/OpenSLAM-org/openslam_gmapping)

### Strengths and Practical Limits

GMapping excels in indoor 2D environments with good odometry: it produces clean occupancy grids at low computational cost (N = 30–100 particles typically suffices). Its weakness is the particle weight collapse in long corridors with few distinctive landmarks, and poor scalability to very large environments where each particle must maintain a gigabyte-scale map.

In ROS 2, `slam_toolbox` has largely superseded `slam_gmapping` for new deployments because it supports lifelong and multi-session mapping via a pose-graph back-end. However, GMapping remains the clearest pedagogical illustration of the RBPF approach and is widely referenced in SLAM literature.

---

## 5. Pose Graph Optimization: G2O

G2O (*General (Hyper) Graph Optimisation*) is a C++ framework for solving nonlinear least-squares problems expressed as factor graphs. It is the de-facto optimisation back-end for a wide range of SLAM systems — ORB-SLAM3, Cartographer, and many others — and is one of the most consequential algorithms in the OpenSLAM corpus.

### Factor Graph Formulation

A SLAM factor graph encodes:

- **Vertices** (nodes): unknown quantities — robot poses *x_i*, or landmark positions *l_j*.
- **Edges** (factors): noisy measurements constraining vertices — an odometry edge constraining consecutive poses *x_i*, *x_{i+1}*, or a loop-closure edge linking two non-adjacent poses identified as the same place.

The optimisation minimises:

```
F(x) = Σ_ij  e_ij(x_i, x_j)^T Ω_ij e_ij(x_i, x_j)
```

where *e_ij* is the residual (difference between the predicted and measured constraint) and *Ω_ij* is the information matrix (inverse covariance of the measurement).

### G2O Architecture

G2O abstracts the solver from the problem definition through a plugin system:

```
┌──────────────────────────────────────────────────────────┐
│  Problem definition (user code)                          │
│  g2o::SparseOptimizer optimizer;                         │
│  optimizer.addVertex(pose_vertex);                       │
│  optimizer.addEdge(odometry_edge);                       │
└──────────────────────┬───────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────┐
│  LinearSolver selection (compile/runtime)                │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │ SuiteSparse │  │  Eigen dense │  │ CHOLMOD (block) │ │
│  │ (CSparse)   │  │  (small prob)│  │                 │ │
│  └─────────────┘  └──────────────┘  └─────────────────┘ │
└──────────────────────┬───────────────────────────────────┘
                       │
              Gauss-Newton / LM iterations
```

G2O provides SE(2) and SE(3) vertex types for 2D and 3D SLAM respectively, and SO(3) parametrised using quaternions to avoid gimbal lock. The Jacobians for standard pose–pose and pose–landmark edges are computed analytically.

```cpp
/* Minimal G2O pose graph: two poses connected by an odometry edge */
#include <g2o/core/sparse_optimizer.h>
#include <g2o/core/block_solver.h>
#include <g2o/core/optimization_algorithm_gauss_newton.h>
#include <g2o/solvers/csparse/linear_solver_csparse.h>
#include <g2o/types/slam3d/vertex_se3.h>
#include <g2o/types/slam3d/edge_se3.h>

using BlockSolverType = g2o::BlockSolver<g2o::BlockSolverTraits<6, 3>>;
using LinearSolverType = g2o::LinearSolverCSparse<BlockSolverType::PoseMatrixType>;

g2o::SparseOptimizer optimizer;
auto linearSolver = std::make_unique<LinearSolverType>();
auto blockSolver  = std::make_unique<BlockSolverType>(std::move(linearSolver));
optimizer.setAlgorithm(
    new g2o::OptimizationAlgorithmGaussNewton(std::move(blockSolver)));

/* Add first pose (fixed — no degrees of freedom to optimise) */
auto* v0 = new g2o::VertexSE3;
v0->setId(0);
v0->setEstimate(Eigen::Isometry3d::Identity());
v0->setFixed(true);
optimizer.addVertex(v0);

/* Add second pose */
auto* v1 = new g2o::VertexSE3;
v1->setId(1);
v1->setEstimate(Eigen::Isometry3d::Identity());  /* initial guess */
optimizer.addVertex(v1);

/* Odometry edge: measured transform from v0 to v1 */
auto* e = new g2o::EdgeSE3;
e->setVertex(0, v0);
e->setVertex(1, v1);
e->setMeasurement(measured_transform);  /* Eigen::Isometry3d */
e->setInformation(information_matrix);  /* 6×6 Eigen::Matrix */
optimizer.addEdge(e);

optimizer.initializeOptimization();
optimizer.optimize(10);  /* 10 Gauss-Newton iterations */
```

[Source: github.com/RainerKuemmerle/g2o](https://github.com/RainerKuemmerle/g2o)

### GPU Acceleration of G2O

G2O's default solvers (SuiteSparse/CHOLMOD, CSparse) are CPU-only and exploit the sparsity of the SLAM Hessian. GPU acceleration of sparse Cholesky factorisation exists — NVIDIA's cuSolver library (`cusolverSpDcsrlsvchol`) and MAGMA provide GPU sparse Cholesky — but the overhead of transferring the sparse system to GPU memory only becomes worthwhile for graphs with tens of thousands of nodes. For typical robot mapping sessions (a few thousand poses), the CPU path is faster.

Research systems like iSAM2 (a factor graph implementation with incremental Bayes tree updates) explore GPU parallelism more aggressively by decomposing the graph into independent sub-problems. For readers implementing SLAM on NVIDIA hardware, CUDA-based block-sparse Cholesky factorisation is the primary acceleration target, wrapping the `cusparse` API (Ch25 §5).

---

## 6. TORO: Tree-Based Network Optimizer

TORO (*Tree-based netwORk Optimiser*) solves the same pose graph problem as G2O but uses an older gradient-descent approach with a tree parameterisation of the graph. It was influential in the 2007–2012 period and is cited extensively in the SLAM literature; G2O supersedes it in practice.

### Tree Parameterisation

TORO represents the graph as a spanning tree rooted at the initial pose. Non-tree edges (loop closures) introduce constraints that the optimiser satisfies by distributing corrections through the tree. The gradient of the total error with respect to each tree-edge's relative pose is computed analytically; gradient steps are applied iteratively until convergence.

The key scaling property: the optimisation complexity is bounded by the *size of the mapped area* (number of distinct places) rather than by the *length of the trajectory*, because revisited places share nodes in the tree. This contrasts with EKF-SLAM, whose complexity grows quadratically with landmark count.

```
/* TORO input format: 2D pose graph */
VERTEX2 0   0.000  0.000  0.000        /* id  x  y  theta */
VERTEX2 1   1.020  0.012 -0.003
VERTEX2 2   2.031  0.019  0.001
EDGE2   0 1  1.020  0.012 -0.003  500 0 0 500 0 500  /* i j dx dy dth  info(6) */
EDGE2   1 2  1.011  0.007  0.004  500 0 0 500 0 500
EDGE2   2 0  -2.031 -0.019 -0.001 100 0 0 100 0 100   /* loop closure */
```

[Source: openslam-org.github.io/toro.html; Grisetti et al., ICRA 2007](https://openslam-org.github.io/toro.html)

Running TORO:
```bash
./toro3d graph.g2o  # g2o format also accepted; outputs optimised poses
```

The `.g2o` file format originated in TORO and was inherited by G2O, making the two tools largely interoperable at the data level. For new projects, G2O is the recommended choice due to its active maintenance, plugin-based solver selection, and native SE(3) support.

---

## 7. Laser Feature Detection: FLIRTLib

Laser range finders — SICK LMS, Hokuyo, Velodyne — produce 2D or 3D point clouds that are the primary sensor input for many mobile robot SLAM systems. Camera-based feature detectors (ORB, SIFT) are inapplicable to range data; FLIRTLib (*Fast Laser Interest Region Transform*) fills this gap.

### Multi-Scale Interest Point Detection for Laser Data

FLIRTLib defines four interest point detectors applied to 1D range profiles:

| Detector | Responds to |
|----------|-------------|
| Range detector | Local minima/maxima of the range profile |
| Normal edge detector | Discontinuities in surface normals |
| Normal blob detector | Regions with consistent surface orientation |
| Curvature detector | High curvature points (corners, edges) |

Each detector operates at multiple scales via a Gaussian-weighted smoothing of the range profile. Scale-space extrema are detected across levels, making the descriptors invariant to sensor range.

Two descriptors encode the local neighbourhood for matching:

- **Shape Context** (adapted from computer vision) — a log-polar histogram of point density in a polar region around the interest point.
- **Beta-Grid** — a coarser occupancy grid descriptor optimised for speed.

Matching uses nearest-neighbour search in descriptor space followed by RANSAC outlier rejection — the same pipeline as camera-based feature SLAM, instantiated for laser geometry.

```cpp
/* FLIRTLib usage: detect interest points in a laser scan */
#include <feature/InterestPoint.h>
#include <feature/RangeDetector.h>
#include <feature/ShapeContext.h>

RangeDetector      detector;
ShapeContextMaker  descriptorMaker;

/* laser_scan: vector<double> of range measurements at uniform angular spacing */
std::vector<InterestPoint*> points = detector.detect(laser_scan);
descriptorMaker.describe(points, laser_scan);  /* compute descriptors in-place */

/* points[i]->getDescriptor() returns a ShapeContext for matching */
```

[Source: openslam-org.github.io/flirtlib.html; Tipaldi & Arras, ICRA 2010](https://openslam-org.github.io/flirtlib.html)

FLIRTLib is typically used as a front-end feeding constraints into G2O or TORO: matched interest point pairs across scans define relative-pose edges, which the back-end optimiser distributes globally. The CPU-only implementation is fast enough for real-time 2D SLAM at 10 Hz laser rates.

---

## 8. RGB-D SLAM and the HOG-Man Optimizer

Before ORB-SLAM and its GPU-accelerated successors, RGBDSlam (F. Endres et al., 2011) was the reference RGB-D SLAM implementation. It pairs a visual front-end with the HOG-Man (*Hierarchical Optimisation on Manifolds for online and time-adaptive robot navigation*) pose graph optimiser, which predates G2O and takes the same factor-graph approach but with a hierarchical clustering strategy for large-scale problems.

### RGBDSlam Architecture

```
Kinect / RealSense / ROS /camera topic
        │
  cv::FeatureDetector (SURF or SIFT)
        │  keypoints + depth-lifted 3D points
  RANSAC 3D transformation estimation
        │  relative pose T_{i,i-1}
  HOG-Man pose graph back-end
        │  global trajectory
  OpenGL point cloud visualisation (OGRE)
```

The 3D transformation is estimated using the RANSAC-based variant of the Perspective-n-Point algorithm (PnP): correspondences between 3D points (lifted from depth) in frame *i* and 2D projections in frame *i+1* constrain a 6DoF pose, and RANSAC rejects outliers from incorrect feature matches.

HOG-Man represents the graph hierarchically: local clusters of pose nodes are optimised internally, and inter-cluster edges represent the aggregate constraint. This reduces the cost of repeated re-optimisation as the graph grows — a property similar to iSAM's Bayes tree. In practice, G2O's incremental extension (g2o with marginalisation) achieves comparable performance with a simpler implementation.

[Source: openslam-org.github.io/rgbdslam.html; Endres et al., T-RO 2014](https://openslam-org.github.io/rgbdslam.html)

---

## 9. Visual SLAM on OpenSLAM: ORB-SLAM and Successors

The original ORB-SLAM (Mur-Artal et al., IROS 2015, GPLv3) is hosted on OpenSLAM and introduced the design that ORB-SLAM3 inherits: three concurrent threads (tracking, local mapping, loop closing) operating on an ORB keypoint vocabulary built offline using DBoW2.

### ORB-SLAM Threading Model

```
Camera frame (30 Hz)
     │
┌────▼──────────────────────────────────────────────────────┐
│ Tracking thread (real-time)                               │
│ ORB extraction → feature matching against local map →     │
│ PnP pose estimation → keyframe decision                   │
└────┬──────────────────────────────────────────────────────┘
     │ new keyframe
┌────▼──────────────────────────────────────────────────────┐
│ Local Mapping thread                                      │
│ MapPoint triangulation → local bundle adjustment (G2O) →  │
│ map point culling                                         │
└────┬──────────────────────────────────────────────────────┘
     │ covisibility graph update
┌────▼──────────────────────────────────────────────────────┐
│ Loop Closing thread                                       │
│ DBoW2 bag-of-words place recognition → geometric check →  │
│ Essential graph optimisation (G2O) → full BA              │
└───────────────────────────────────────────────────────────┘
```

The local bundle adjustment in the Local Mapping thread uses G2O with an SE(3) graph encoding keyframe poses and map-point positions as vertices, and reprojection error edges. This is where the OpenSLAM back-end ecosystem becomes a direct dependency of a mainstream visual SLAM system.

ORB-SLAM3 (Ch114 §14.1) extends the architecture to monocular, stereo, and RGB-D sensors, adds IMU preintegration for visual-inertial odometry, and introduces multi-map support. For GPU acceleration of the ORB extraction front-end, see Ch114 §14.1 (CUDA `cv::cuda::ORB`).

[Source: openslam-org.github.io/orbslam.html; ORB-SLAM3 github.com/UZ-SLAMLab/ORB_SLAM3](https://github.com/UZ-SLAMLab/ORB_SLAM3)

---

## 10. Visual-Inertial Odometry: Basalt and Monado's XR Tracking Pipeline

Inside-out 6DoF tracking — the ability of an XR headset to determine its position without external base-stations — is implemented on Linux via Monado, the open-source OpenXR runtime (Ch27). Monado integrates visual-inertial odometry through a pluggable SLAM tracker interface.

### Basalt VIO

Basalt ([gitlab.com/VladyslavUsenko/basalt](https://gitlab.com/VladyslavUsenko/basalt)) is a visual-inertial mapping system developed at the Technical University of Munich. Its key algorithmic contribution is **non-linear factor recovery with square-root marginalisation**: instead of discarding old states when they leave a sliding window (which degrades map quality), Basalt computes a compact nonlinear factor that summarises their constraints on remaining states. This gives it accuracy competitive with full-batch optimisation at bounded computational cost.

Monado ships a patched Basalt fork called **Monado-Basalt** that interfaces with Monado's device driver system:

```
┌────────────────────────────────────────────────────────────┐
│  Monado XR runtime                                         │
│                                                            │
│  Device driver (e.g. WMR, Rift S, Index)                  │
│      │ raw stereo frames + IMU stream                      │
│  xrt_slam_sinks / xrt_fs::slam_stream_start interface      │
│      │                                                     │
│  Basalt VIO pipeline (stereo camera + IMU)                 │
│      │ SE(3) pose at camera timestamp                      │
│  pose prediction + reprojection → headset display          │
└────────────────────────────────────────────────────────────┘
```

The `xrt_slam_sinks` struct (defined in Monado's SLAM tracker subsystem) aggregates the data sinks that a device driver must populate:

```c
/* Monado SLAM sinks — container of input endpoints for the tracker.
 * Sinks are considered disabled if null.
 * Source: monado.pages.freedesktop.org/monado/structxrt__slam__sinks.html */
struct xrt_slam_sinks {
    int                      cam_count;    /* number of active cameras (≤5) */
    struct xrt_frame_sink   *cams[5];      /* per-camera frame sinks */
    struct xrt_imu_sink     *imu;          /* IMU sample stream */
    struct xrt_pose_sink    *gt;           /* ground truth poses (optional) */
    struct xrt_hand_masks_sink *hand_masks; /* hand occlusion masks (optional) */
};
```

A device driver starts the tracker via `xrt_fs::slam_stream_start(sinks)`, then pushes frames and IMU samples into `sinks->cams[i]` and `sinks->imu` respectively. The tracker (Basalt) runs asynchronously and exposes the latest SE(3) pose through a separate query interface.

[Source: monado.pages.freedesktop.org/monado/structxrt__slam__sinks.html](https://monado.pages.freedesktop.org/monado/structxrt__slam__sinks.html)

### Supported Devices

Monado's experimental inside-out tracking stack supports a range of headsets including the Valve Index, Oculus Rift S, Windows Mixed Reality (WMR) headsets, and the open-hardware North Star. Each device exposes its stereo camera stream and IMU via a Monado device driver; the driver populates `xrt_slam_sinks` with the camera and IMU endpoints, and the tracker runs independently of the specific device.

[Source: monado.pages.freedesktop.org — Tracking group documentation](https://monado.pages.freedesktop.org/monado/group__aux__tracking.html)

The stereo camera frames are acquired from the headset's USB camera stream via the Linux USB Video Class (UVC) driver and delivered to Basalt through the `xrt_frame` pipeline. IMU measurements arrive via HID reports parsed by the device driver.

### Basalt's Algorithmic Pipeline

```
Stereo frames (T)
        │
  FAST corner detection (per camera)
        │  raw corners
  Optical flow tracking (Lucas-Kanade, multi-scale)
        │  tracked corner positions across T, T-1, T-2
  IMU preintegration (Forster et al., 2017)
        │  pre-integrated measurement between frames
  Sliding-window nonlinear optimisation
    │  window of K keyframes
    │  reprojection error + IMU residuals
    │  square-root marginalisation at window boundary
  Recovered pose SE(3) at frame T
```

Optical flow in Basalt is implemented in C++ using SSE/AVX SIMD intrinsics rather than GPU compute — the frame sizes from headset cameras are modest (640×480 per eye), and SIMD is sufficient for real-time operation. This contrasts with DROID-SLAM (Ch114 §14.2), which replaces the optical-flow step with a CUDA-resident dense update network.

[Source: gitlab.com/VladyslavUsenko/basalt; Usenko et al., RA-L 2020 — doi:10.1109/LRA.2020.3004028](https://arxiv.org/abs/1911.01500)

---

## 11. The Linux Camera Pipeline as SLAM Input

SLAM algorithms consume raw camera frames. On Linux, three distinct camera access paths deliver these frames, each with different latency and metadata characteristics.

### V4L2 Direct Capture

The Video for Linux 2 (V4L2) subsystem exposes camera devices as character nodes (`/dev/video0`). Direct V4L2 capture gives the lowest latency and is used by robotics stacks where a dedicated camera (UVC webcam, CSI sensor) is reserved for SLAM:

```c
/* V4L2 frame capture for SLAM — mmap streaming */
int fd = open("/dev/video0", O_RDWR);

struct v4l2_requestbuffers req = {
    .count  = 4,
    .type   = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .memory = V4L2_MEMORY_MMAP,
};
ioctl(fd, VIDIOC_REQBUFS, &req);

/* mmap each buffer, enqueue, start streaming */
/* ...setup omitted for brevity... */

struct v4l2_buffer buf = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
                            .memory = V4L2_MEMORY_MMAP };
ioctl(fd, VIDIOC_DQBUF, &buf);  /* dequeue filled buffer */

/* buf.m.offset is the mmap offset; buf.timestamp is the kernel timestamp */
/* CRITICAL for SLAM: use buf.timestamp for IMU synchronisation */
struct timespec ts = { .tv_sec  = buf.timestamp.tv_sec,
                       .tv_nsec = buf.timestamp.tv_usec * 1000 };
```

The `v4l2_buffer.timestamp` is a kernel-side capture timestamp — essential for IMU-camera synchronisation in visual-inertial SLAM. Hardware trigger synchronisation (via GPIO) can reduce this timestamp uncertainty to sub-millisecond, which matters for aggressive VIO performance.

[Source: kernel.org/doc/html/latest/userspace-api/media/v4l/v4l2.html; Ch96 §3](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/v4l2.html)

### libcamera for Complex Sensor Pipelines

libcamera (Ch96) abstracts the ISP (Image Signal Processor) pipeline, enabling SLAM access to cameras that require ISP tuning — Raspberry Pi cameras, Qualcomm CSI cameras, and similar embedded sensors. SLAM typically requires:

- **Raw Bayer frames** (disable auto-exposure and auto-white-balance, which change pixel values non-deterministically between frames)
- **Fixed exposure / gain** (set via `libcamera::controls::ExposureTime` and `AnalogueGain`)
- **High frame-rate capture** (60 Hz or faster for aggressive motion)

```cpp
/* libcamera: fixed-exposure capture for SLAM */
#include <libcamera/libcamera.h>

auto camera = cm->get(cm->cameras()[0]->id());
camera->acquire();

std::unique_ptr<libcamera::CameraConfiguration> config =
    camera->generateConfiguration({ libcamera::StreamRole::Raw });
config->at(0).pixelFormat = libcamera::formats::SBGGR10;  /* raw Bayer 10-bit */
config->at(0).size        = { 1280, 720 };
camera->configure(config.get());

libcamera::ControlList controls;
controls.set(libcamera::controls::AeEnable,      false);  /* manual exposure */
controls.set(libcamera::controls::ExposureTime,  8000);   /* 8 ms */
controls.set(libcamera::controls::AnalogueGain,  4.0f);
```

[Source: libcamera.org/api-html; Ch96 §4](https://libcamera.org/)

### DMA-BUF Zero-Copy to GPU

When SLAM front-end computation (feature extraction, optical flow) runs on GPU, copying camera frames from CPU memory is a bottleneck. The Linux DMA-BUF framework enables zero-copy sharing: the camera driver allocates buffers in device-accessible memory and exports them as file descriptors that can be imported directly into CUDA, OpenCL, or Vulkan.

```c
/* DMA-BUF export from V4L2 → import into CUDA */

/* Export V4L2 buffer as DMA-BUF fd */
struct v4l2_exportbuffer expbuf = {
    .type  = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .index = buf_index,
};
ioctl(fd, VIDIOC_EXPBUF, &expbuf);
int dmabuf_fd = expbuf.fd;

/* Import into CUDA via EGL image */
EGLImageKHR egl_image = eglCreateImageKHR(
    egl_display, EGL_NO_CONTEXT,
    EGL_LINUX_DMA_BUF_EXT, NULL,
    (EGLint[]){
        EGL_WIDTH,  1280, EGL_HEIGHT, 720,
        EGL_LINUX_DRM_FOURCC_EXT,    DRM_FORMAT_NV12,
        EGL_DMA_BUF_PLANE0_FD_EXT,   dmabuf_fd,
        EGL_DMA_BUF_PLANE0_OFFSET_EXT, 0,
        EGL_DMA_BUF_PLANE0_PITCH_EXT,  1280,
        EGL_NONE
    });

/* Bind to CUDA external memory */
cudaExternalMemoryHandleDesc extDesc = {};
extDesc.type               = cudaExternalMemoryHandleTypeOpaqueFd;
extDesc.handle.fd          = dup(dmabuf_fd);
extDesc.size               = frame_size_bytes;
cudaExternalMemory_t extMem;
cudaImportExternalMemory(&extMem, &extDesc);
```

This pipeline is relevant to GPU-accelerated SLAM front-ends (Ch114 §14, Ch25 §6) and eliminates a PCIe copy that would otherwise constrain throughput at high frame rates.

---

## 12. GPU Acceleration Strategies for SLAM Workloads

SLAM workloads have a bimodal GPU fit: the front-end is highly parallel and benefits from GPU; the back-end is dominated by sparse linear algebra that is better on CPU for typical map sizes.

### Front-End: Feature Extraction and Optical Flow

| Operation | GPU API | Notes |
|-----------|---------|-------|
| ORB extraction | CUDA `cv::cuda::ORB` | Parallel per-octave FAST + BRIEF |
| SURF/SIFT | CUDA `cv::cuda::SURF_CUDA` | Hessian detector, descriptor orientation |
| Optical flow (LK) | CUDA `cv::cuda::SparsePyrLKOpticalFlow` | Per-pyramid-level parallelism |
| Stereo matching | CUDA `cv::cuda::StereoBM` / `StereoSGM` | Row-parallel DP for disparity |
| ORB vocabulary query | FAISS GPU (nearest-neighbour in descriptor space) | DBoW2 replacement for place recognition |

For Vulkan-based pipelines, ORB feature extraction can be implemented as a compute shader dispatched via `vkCmdDispatch` (Ch25 §3), enabling vendor-agnostic GPU acceleration without a CUDA dependency:

```glsl
/* GLSL compute shader: FAST-9 corner score at each pixel */
#version 450
layout(local_size_x = 16, local_size_y = 16) in;
layout(binding = 0, r8ui)  uniform readonly  uimage2D inputImage;
layout(binding = 1, r32f)  uniform writeonly image2D  scoreImage;

void main() {
    ivec2 pos = ivec2(gl_GlobalInvocationID.xy);
    uint  center = imageLoad(inputImage, pos).r;

    /* FAST-9 Bresenham circle test (16 pixels on radius-3 circle) */
    const ivec2 offsets[16] = ivec2[16](
        ivec2( 0, 3), ivec2( 1, 3), ivec2( 2, 2), ivec2( 3, 1),
        ivec2( 3, 0), ivec2( 3,-1), ivec2( 2,-2), ivec2( 1,-3),
        ivec2( 0,-3), ivec2(-1,-3), ivec2(-2,-2), ivec2(-3,-1),
        ivec2(-3, 0), ivec2(-3, 1), ivec2(-2, 2), ivec2(-1, 3)
    );

    int bright = 0, dark = 0;
    uint threshold = 20u;
    for (int i = 0; i < 16; i++) {
        uint p = imageLoad(inputImage, pos + offsets[i]).r;
        if (p > center + threshold) bright++;
        else if (p < center - threshold) dark++;
    }

    /* FAST corner: 9 consecutive bright or dark pixels */
    float score = (bright >= 9 || dark >= 9) ? float(center) : 0.0;
    imageStore(scoreImage, pos, vec4(score));
}
```

### Back-End: Sparse Linear Algebra at Scale

For large-scale SLAM (tens of thousands of poses, such as autonomous vehicle city-scale mapping), GPU-accelerated sparse Cholesky solvers become competitive:

```cpp
/* cuSolver sparse Cholesky for SLAM Hessian */
#include <cusolverSp.h>
#include <cusparse.h>

cusolverSpHandle_t solverHandle;
cusolverSpCreate(&solverHandle);

/* H: sparse 6n×6n SLAM Hessian (CSR format) */
/* b: 6n RHS vector */
int singularity;
cusolverSpDcsrlsvchol(
    solverHandle,
    n_rows, nnz,
    descr,
    d_H_values, d_H_rowPtr, d_H_colInd,
    d_b,
    1e-12,   /* tolerance */
    0,       /* reorder = no fill-reducing permutation */
    d_x,     /* solution */
    &singularity);
```

[Source: docs.nvidia.com/cuda/cusolver/index.html#cusolverspcsrlsvchol](https://docs.nvidia.com/cuda/cusolver/index.html)

For AMD GPUs, the equivalent is `rocsolver_dcsrsm` via the ROCm sparse library (Ch108).

### Dense Reconstruction on GPU

Beyond trajectory estimation, dense surface reconstruction from RGB-D streams runs effectively on GPU. Approaches include:

- **KinectFusion / VoxelHashing** — volumetric TSDF (Truncated Signed Distance Function) fusion, where each voxel's occupancy is updated by parallel GPU threads processing each depth pixel.
- **3D Gaussian Splatting SLAM** (MonoGS, Ch114 §14.3) — a differentiable renderer replaces the TSDF with a set of 3D Gaussians optimised end-to-end.
- **ElasticFusion** — deformable dense SLAM using a GPU-resident surfel map; requires CUDA for the surface normal estimation and ICP alignment.

---

## 13. ROS 2 and the Linux Robotics Integration Layer

Robot Operating System 2 (ROS 2) is the de-facto middleware for Linux robotics. SLAM algorithms from OpenSLAM are typically consumed through ROS 2 packages that wrap the C++ libraries and expose them as composable nodes.

### Key ROS 2 SLAM Packages

| Package | Wraps | Back-End |
|---------|-------|----------|
| `slam_toolbox` | Custom (Karto SLAM) | G2O pose graph |
| `slam_gmapping` | GMapping | Particle filter |
| `rtabmap_ros` | RTAB-Map | G2O + GTSAM |
| `cartographer_ros` | Google Cartographer | Ceres sparse optimiser |
| `ORB_SLAM3_ros2` | ORB-SLAM3 | G2O |

### ROS 2 SLAM Node Architecture

```
/camera/image_raw  (sensor_msgs/Image, 30 Hz)
/camera/camera_info (sensor_msgs/CameraInfo)
/imu/data          (sensor_msgs/Imu, 200 Hz)
         │
    slam_toolbox node
         │ TF2 transform tree
    /map         (nav_msgs/OccupancyGrid, 1 Hz)
    /odom        (nav_msgs/Odometry, 30 Hz)
    /tf          (geometry_msgs/TransformStamped)
         │
    nav2 (Navigation2) — path planning
```

The ROS 2 Time abstraction and `rclcpp` executor model allow SLAM nodes to be composed into a single process for zero-copy inter-node communication via `intra_process_comms`, eliminating the serialise/deserialise overhead on camera frame data.

### DMA-BUF and Zero-Copy in ROS 2

The `ros2_v4l2_camera` driver supports DMA-BUF buffer export, allowing zero-copy delivery of camera frames from V4L2 into a downstream SLAM node's CUDA pipeline via the `image_transport` plugin infrastructure. This pattern is enabled by [REP-2007](https://ros.org/reps/rep-2007.html) (Type Adaptation, ROS 2 Galactic+), which allows a node to publish a DMA-BUF-backed type that is transmitted without serialisation to a subscribed SLAM node in the same process. It is increasingly used in GPU-accelerated ROS 2 robotics stacks on NVIDIA Jetson.

```python
# ROS 2 Python: publishing SLAM pose as a TF2 transform
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import TransformStamped
import tf2_ros

class SLAMPublisher(Node):
    def __init__(self):
        super().__init__('slam_publisher')
        self.tf_broadcaster = tf2_ros.TransformBroadcaster(self)

    def publish_pose(self, translation, rotation, stamp):
        t = TransformStamped()
        t.header.stamp    = stamp
        t.header.frame_id = 'map'
        t.child_frame_id  = 'base_link'
        t.transform.translation.x = translation[0]
        t.transform.translation.y = translation[1]
        t.transform.translation.z = translation[2]
        t.transform.rotation.x = rotation[0]
        t.transform.rotation.y = rotation[1]
        t.transform.rotation.z = rotation[2]
        t.transform.rotation.w = rotation[3]
        self.tf_broadcaster.sendTransform(t)
```

### GPU Memory Considerations on Embedded Platforms

On NVIDIA Jetson (Orin, Xavier), the CPU and GPU share physical memory via unified memory architecture. V4L2 allocations on Jetson can be mapped directly into CUDA with zero copy via `cudaHostRegister` on the mmap'd V4L2 buffer. AMD ROCm on embedded platforms (Radeon RX embedded SoCs) similarly supports `hipHostRegister`. This eliminates the PCIe bottleneck present on discrete GPU systems.

---

## 14. Integrations

- **Chapter 25 — GPU Compute**: Vulkan compute shaders for FAST corner detection and ORB descriptor computation; CUDA sparse linear algebra for large-scale pose graph optimisation (`cuSolver`, `cuSPARSE`).
- **Chapter 27 — VR & AR**: Monado's `xrt_slam_sinks` / `xrt_fs::slam_stream_start` interface that feeds camera and IMU streams into Basalt VIO to drive 6DoF headset pose for OpenXR applications; DRM Lease for direct display (Ch121).
- **Chapter 38 — PipeWire**: camera frame delivery through the PipeWire video session layer to SLAM consumers running as privileged apps or in sandboxed contexts.
- **Chapter 96 — libcamera**: ISP-tuned raw Bayer capture for camera-based SLAM; fixed-exposure control essential for photometric consistency across frames.
- **Chapter 114 — OpenCV and GPU-Accelerated Computer Vision**: the GPU-accelerated successors to classical OpenSLAM algorithms — ORB-SLAM3 CUDA front-end (§14.1), DROID-SLAM dense BA (§14.2), MonoGS Gaussian splatting SLAM (§14.3).
- **Chapter 115 — NeRFStudio and 3DGS**: Neural radiance field and Gaussian splatting scene representations that complement SLAM-estimated trajectories for photorealistic dense reconstruction.
- **Chapter 142 — V4L2 and the Linux Media Subsystem**: V4L2 streaming API, buffer types, and DMA-BUF export used to feed camera frames into SLAM pipelines with kernel-accurate timestamps.
- **Chapter 150 — EGL and DMA-BUF Integration**: EGL image import from DMA-BUF file descriptors enabling zero-copy camera frame hand-off to OpenGL or CUDA SLAM front-ends.
- **Chapter 203 — WebXR**: browser-side 6DoF tracking that builds on the same algorithmic lineage in browser-embedded SLAM implementations.

---

*OpenSLAM.org, founded 2006, migrated to GitHub 2018: [github.com/OpenSLAM-org](https://github.com/OpenSLAM-org)*
