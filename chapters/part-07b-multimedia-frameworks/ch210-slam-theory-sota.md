# Chapter 210: SLAM Theory and State of the Art

**Target audiences:** Systems developers building sensor-fusion pipelines on Linux; robotics
engineers integrating LiDAR and IMU data with ROS 2; application developers choosing a SLAM
back-end for autonomous navigation or AR; researchers bridging classical probabilistic SLAM
with modern learning-based approaches.

This chapter provides the probabilistic and algorithmic foundations underlying every SLAM
system in this book — from the GMapping and G2O pipelines of Chapter 209 to the GPU-accelerated
visual odometry of Chapter 114. It then surveys the major open-source Linux SLAM systems not
covered elsewhere: **LiDAR-inertial odometry** (FAST-LIO2, LIO-SAM), **submap-based LiDAR
SLAM** (Google Cartographer), and the **factor-graph toolkit** (GTSAM/iSAM2) that powers
their optimisation back-ends. Learning-based and neural SLAM systems (DROID-SLAM, SplaTAM,
MonoGS) are covered in full in Chapter 114 and cross-referenced here.

---

## Table of Contents

1. [The SLAM Problem: Formal Statement](#1-the-slam-problem-formal-statement)
   - [1.1 What is SLAM?](#11-what-is-slam)
   - [1.2 What is a Factor Graph?](#12-what-is-a-factor-graph)
   - [1.3 What is LiDAR?](#13-what-is-lidar)
   - [1.4 What is an IMU?](#14-what-is-an-imu)
2. [Bayesian Filtering: The Online Backbone](#2-bayesian-filtering-the-online-backbone)
3. [Factor Graphs and MAP Estimation](#3-factor-graphs-and-map-estimation)
4. [Linearisation, Normal Equations, and Schur Complement](#4-linearisation-normal-equations-and-schur-complement)
5. [Incremental Smoothing: iSAM2 and the Bayes Tree](#5-incremental-smoothing-isam2-and-the-bayes-tree)
6. [IMU Pre-Integration in Factor Graphs](#6-imu-pre-integration-in-factor-graphs)
7. [LiDAR-Inertial Odometry: FAST-LIO2](#7-lidar-inertial-odometry-fast-lio2)
8. [LiDAR-Inertial SLAM with Loop Closure: LIO-SAM](#8-lidar-inertial-slam-with-loop-closure-lio-sam)
9. [Submap-Based LiDAR SLAM: Google Cartographer](#9-submap-based-lidar-slam-google-cartographer)
10. [Visual-Inertial SLAM on Linux](#10-visual-inertial-slam-on-linux)
11. [Learning-Based and Neural SLAM (Pointers to Chapter 114)](#11-learning-based-and-neural-slam-pointers-to-chapter-114)
12. [Benchmarks and Honest Evaluation](#12-benchmarks-and-honest-evaluation)
13. [System Integration: End-to-End LiDAR SLAM on Linux](#13-system-integration-end-to-end-lidar-slam-on-linux)
14. [GPU and CPU Parallelism on Linux](#14-gpu-and-cpu-parallelism-on-linux)
15. [Integrations](#15-integrations)

---

## 1. The SLAM Problem: Formal Statement

SLAM requires a robot to simultaneously estimate its own trajectory and construct a map of an
unknown environment using only on-board sensors. The joint posterior over robot poses
`x_{1:t}` and map `m` given observations `z_{1:t}` and control inputs `u_{1:t}` is:

```
p(x_{1:t}, m | z_{1:t}, u_{1:t})
```

**Full SLAM** (offline batch) estimates the entire trajectory at once; **online SLAM**
marginalises out all but the current pose, tracking only `p(x_t, m | z_{1:t}, u_{1:t})`.
Particle filters (FastSLAM, Chapter 209) approximate the online posterior; smoothers such as
iSAM2 (§5) handle the full trajectory but do so incrementally.

The two canonical decompositions:

| Variant | What is maintained | Representative systems |
|---|---|---|
| Online filtering | Current pose + map | EKF-SLAM, FastSLAM, FAST-LIO2 |
| Batch smoothing | Full trajectory | GTSAM, g2o (Chapter 209), Ceres |
| Incremental smoothing | Full trajectory, on-the-fly update | iSAM2, LIO-SAM |

The choice has direct hardware consequences on Linux: filtering solutions run on CPU-only
embedded boards such as Jetson Nano or Raspberry Pi 4; incremental smoothers typically need
≥512 MB RAM and benefit significantly from NEON/AVX SIMD in Eigen or from GPU-resident
sparse solvers (cuSolver, ROCm hipSolver).

### 1.1 What is SLAM?

SLAM (Simultaneous Localization and Mapping) is the problem of building a consistent map of an unknown environment while simultaneously tracking position within that map — a circular dependency that makes it one of the central challenges in mobile robotics. Without a map, accurate localization is impossible; without accurate localization, building a consistent map is impossible. Classical solutions break this circularity through probabilistic state estimation: the robot maintains a probability distribution over all possible poses and map configurations, then refines it incrementally as new sensor data arrives.

In Linux robotics pipelines, SLAM typically consumes data from one or more range sensors (LiDAR, stereo cameras, RGB-D) combined with an inertial measurement unit. The output is a pose estimate in a global or local reference frame, plus a map representation — occupancy grid, point cloud, signed-distance field, or factor graph — that downstream planners and task executors consume via ROS 2 topics or shared memory.

This chapter treats SLAM at the algorithmic level: the probabilistic formulations, optimisation structures, and open-source implementations available on Linux. Hardware-level sensor drivers (LiDAR kernel drivers, V4L2, the IIO IMU subsystem) are covered in earlier parts; GPU acceleration of SLAM back-ends appears in Chapter 114.

### 1.2 What is a Factor Graph?

A factor graph is a bipartite graphical model used to represent the joint probability distribution over all unknowns in the SLAM problem. One class of nodes represents variables — robot poses at each timestep, landmark positions in the map, and IMU bias parameters. The other class represents factors — probabilistic constraints derived from sensor measurements, each linking a small subset of variable nodes. The joint distribution factorises as a product of these local factors, which is why the structure is called a factor graph.

The key advantage over full-covariance representations such as EKF-SLAM is sparsity: most factors connect only two or three variable nodes, so the corresponding information matrix has very few non-zero entries. Sparse direct solvers (Cholesky, QR) can therefore solve systems with thousands of poses efficiently. Factor graphs also support incremental updates — adding a new factor and solving only the affected portion of the graph — which is the foundation of iSAM2 (§5).

On Linux, the GTSAM library (available as `libgtsam-dev` on Ubuntu 22.04 and 24.04) provides the canonical C++ factor-graph implementation. GTSAM is used internally by LIO-SAM and is compatible with ROS 2 through standard CMake integration. [Source: github.com/borglab/gtsam](https://github.com/borglab/gtsam)

### 1.3 What is LiDAR?

LiDAR (Light Detection and Ranging) is an active range sensor that measures distances by timing the round-trip of emitted laser pulses. Modern spinning LiDAR units (Velodyne VLP-16, Ouster OS1, Livox Avia) emit hundreds of thousands to millions of points per second, each tagged with range, azimuth, elevation, and return intensity, forming a structured 3D point cloud. Solid-state LiDAR variants use MEMS mirrors or optical phased arrays to eliminate moving parts, reducing cost and size at the expense of a narrower field of view.

On Linux, LiDAR devices connect primarily over Ethernet (UDP broadcast) or USB, exposing raw packet streams. ROS 2 driver packages such as `velodyne_driver` and `ouster-ros` parse these packets into `sensor_msgs/PointCloud2` messages on the `/points` topic. LiDAR data suits SLAM particularly well because range measurements are metric and largely illumination-independent, unlike camera images. The point cloud density and the structure of LiDAR returns (single vs. multiple returns per pulse) affect which scan-matching algorithm — NDT, ICP, or LOAM-style feature extraction — is most appropriate, and those choices propagate directly into the factor-graph formulation used in §§7–9.

### 1.4 What is an IMU?

An IMU (Inertial Measurement Unit) is a sensor package that measures linear acceleration and angular velocity at high rates — typically 100 Hz to 1 kHz — using MEMS accelerometers and gyroscopes. Some units also integrate a magnetometer for heading reference. In SLAM systems, the IMU provides motion predictions between slower sensor updates (LiDAR at 10–20 Hz, cameras at 30–60 Hz), dramatically improving pose estimation during fast motion, aggressive rotations, or temporary sensor occlusion.

On Linux, IMU devices appear in the IIO (Industrial I/O) subsystem when connected via I2C or SPI, or as USB HID devices. Common units used in robotics — VectorNav VN-100, Xsens MTi, Phidgets Spatial — provide calibrated data over USB serial or Ethernet. The ROS 2 package `imu_tools` provides complementary filters and calibration utilities operating on `sensor_msgs/Imu` messages. A critical preprocessing step is IMU pre-integration (§6): instead of adding a variable node to the factor graph for every IMU sample, measurements between two keyframe timestamps are combined into a single relative-motion constraint. This keeps the graph size proportional to the number of keyframes rather than the number of IMU samples, which is essential for real-time operation on resource-constrained Linux hardware.

---

## 2. Bayesian Filtering: The Online Backbone

Every online SLAM system implements a **predict–update** cycle driven by Bayes' rule.

**Predict** (motion model):

```
p(x_t | z_{1:t-1}, u_{1:t}) = ∫ p(x_t | x_{t-1}, u_t) p(x_{t-1} | z_{1:t-1}) dx_{t-1}
```

**Update** (measurement model):

```
p(x_t | z_{1:t}, u_{1:t}) ∝ p(z_t | x_t, m) · p(x_t | z_{1:t-1}, u_{1:t})
```

### 2.1 Extended Kalman Filter SLAM

EKF-SLAM maintains a joint Gaussian over the robot pose and all landmark positions. The
state vector has dimension `3 + 2N` (2D) or `6 + 3N` (3D) for N landmarks. Jacobians
linearise the non-linear motion and observation models at the current estimate.

Computational complexity is O(N²) per update — the landmark covariance block grows
quadratically — making EKF-SLAM impractical beyond ~500 landmarks. This drove the
development of sparse factor-graph methods (§3) and sub-map approaches (§9).

### 2.2 Unscented Kalman Filter

The UKF propagates a set of **sigma points** through the non-linear model, recovering mean
and covariance without computing Jacobians. This improves accuracy on highly non-linear
systems (e.g. aggressive rotations) at a constant-factor overhead vs. EKF. The ROS 2
`robot_localization` package ships a production-quality UKF used with VectorNav, Xsens, and
Phidgets IMUs.

### 2.3 Particle Filter (FastSLAM — cross-reference Chapter 209)

FastSLAM and GMapping represent the pose distribution as a weighted particle cloud, each
particle maintaining an independent occupancy-grid map estimate. Chapter 209 §3–4 covers
the Linux implementations (`openslam_gmapping`, `slam_toolbox`) in detail. The key
limitation is particle degeneracy at high dimensionality; the Rao-Blackwellisation trick
(particles over poses only, EKF over landmarks per particle) mitigates but does not
eliminate it.

---

## 3. Factor Graphs and MAP Estimation

A **factor graph** is a bipartite graph with variable nodes (robot poses, landmark positions,
IMU biases) and factor nodes (measurement constraints). Each factor `f_i` represents a
probabilistic constraint over a subset of variables:

```
p(x) ∝ ∏_i f_i(x_i)
```

MAP (maximum a posteriori) inference finds:

```
x* = argmax_x ∏_i f_i(x_i)
```

Under the Gaussian assumption, each factor is `f_i(x_i) = exp(-½ ||e_i(x_i)||²_{Ω_i})`,
where `e_i` is the error function and `Ω_i = Σ_i⁻¹` is the information matrix. Maximising
the product is equivalent to minimising the sum of squared Mahalanobis distances:

```
x* = argmin_x Σ_i ||e_i(x_i)||²_{Ω_i}
```

Factor graphs expose the **sparsity** of the problem: each factor only connects a few
variable nodes, so the information matrix is sparse, and sparse linear algebra makes
large-scale SLAM tractable.

### 3.1 GTSAM: Factor Graphs on Linux

GTSAM (Georgia Tech Smoothing and Mapping) is the principal open-source C++ factor-graph
library in robotics. Available as `apt install libgtsam-dev` on Ubuntu 22.04/24.04 and
as a CMake dependency in ROS 2. [Source: github.com/borglab/gtsam](https://github.com/borglab/gtsam)

```cpp
// gtsam/slam/BetweenFactor.h, gtsam/geometry/Pose3.h
#include <gtsam/nonlinear/NonlinearFactorGraph.h>
#include <gtsam/nonlinear/Values.h>
#include <gtsam/slam/BetweenFactor.h>
#include <gtsam/geometry/Pose3.h>
#include <gtsam/inference/Symbol.h>

using namespace gtsam;
using symbol_shorthand::X;   // Pose3 keys: X(0), X(1), ...

NonlinearFactorGraph graph;
Values initial;

// Prior on the first pose (anchor the gauge freedom)
auto priorModel = noiseModel::Diagonal::Sigmas(
    (Vector6() << 1e-6, 1e-6, 1e-6, 1e-4, 1e-4, 1e-4).finished());
graph.addPrior(X(0), Pose3::Identity(), priorModel);
initial.insert(X(0), Pose3::Identity());

// Relative-pose factor from odometry or LiDAR scan matching
auto odomModel = noiseModel::Diagonal::Sigmas(
    (Vector6() << 0.1, 0.1, 0.1, 0.01, 0.01, 0.01).finished());
Pose3 relativePose(Rot3::RzRyRx(0, 0, 0.1), Point3(0.5, 0, 0));
graph.emplace_shared<BetweenFactor<Pose3>>(X(0), X(1), relativePose, odomModel);
initial.insert(X(1), relativePose);

// Batch optimisation
LevenbergMarquardtOptimizer optimizer(graph, initial);
Values result = optimizer.optimize();
```

`BetweenFactor<Pose3>` encodes a relative-pose measurement on SE(3). `noiseModel::Diagonal`
assumes uncorrelated noise; `noiseModel::Gaussian` accepts a full covariance for correlated
errors such as those returned by ICP scan matching.

### 3.2 Robust Noise Models for Loop Closure

Gaussian noise models are appropriate when measurements are reliable (IMU, short-range
odometry). Loop-closure factors introduced by potentially incorrect place recognition must
be treated with **robust (M-estimator) noise models** that down-weight large residuals
rather than penalising them quadratically:

```cpp
// Huber loss: quadratic below delta, linear above
auto robustModel = noiseModel::Robust::Create(
    noiseModel::mEstimator::Huber::Create(1.345),  // delta in stddev units
    noiseModel::Diagonal::Sigmas(
        (Vector6() << 0.3, 0.3, 0.3, 0.05, 0.05, 0.05).finished()));

// Cauchy loss: heavier down-weighting for potential outliers
auto cauchyModel = noiseModel::Robust::Create(
    noiseModel::mEstimator::Cauchy::Create(0.1),
    baseNoise);

// Use the robust model for loop closure constraints
graph.emplace_shared<BetweenFactor<Pose3>>(
    X(loopFrom), X(i), loopRelPose, robustModel);
```

The Huber estimator behaves as standard Gauss-Newton below the threshold and as IRLS
(Iteratively Reweighted Least Squares) above it. In practice, Huber is recommended over
Cauchy for LiDAR loop closure: Cauchy's very heavy tails can cause the optimiser to ignore
a genuinely large loop closure if the initial pose graph is badly initialised.
[Source: gtsam.org/tutorials/intro.html](https://gtsam.org/tutorials/intro.html)

---

## 4. Linearisation, Normal Equations, and Schur Complement

At each Gauss-Newton iteration the non-linear cost is linearised at the current estimate
`x̄`:

```
F(x̄ + δx) ≈ Σ_i ||J_i δx - b_i||²_{Ω_i}
```

where `J_i = ∂e_i/∂x |_{x̄}` is the Jacobian and `b_i = -e_i(x̄)`. Summing gives the
**normal equations**:

```
H δx = g
H = Σ_i J_i^T Ω_i J_i       (sparse information matrix)
g = Σ_i J_i^T Ω_i b_i
```

H is positive semi-definite and sparse because each factor touches only a few variables.
Sparse Cholesky factorisation (via CHOLMOD, SuiteSparse) runs in O(N^{1.5}) for planar
environments, far better than the dense O(N³).

### 4.1 Schur Complement for Landmark Marginalisation

In visual SLAM, poses (p) are few and landmarks (l) are numerous. Partitioning H:

```
[H_pp  H_pl] [δx_p]   [g_p]
[H_lp  H_ll] [δx_l] = [g_l]
```

Because each landmark is seen from only a local set of poses, `H_ll` is block-diagonal and
cheaply invertible. The **Schur complement** eliminates landmarks:

```
S δx_p = g_p - H_pl H_ll⁻¹ g_l
S = H_pp - H_pl H_ll⁻¹ H_lp
```

The reduced system involves only pose variables. GTSAM exposes this through
`EliminateableFactorGraph::eliminateMultifrontal()`, with the COLAMD variable-ordering
heuristic minimising Cholesky fill-in.

---

## 5. Incremental Smoothing: iSAM2 and the Bayes Tree

Batch re-optimisation after every new factor is wasteful when only a local region of the
graph is affected. **iSAM2** (Incremental Smoothing and Mapping v2) maintains a **Bayes
tree** — a junction-tree representation of the joint Gaussian — and re-eliminates only the
cliques touched by new factors.
[Paper: Kaess et al. 2012, doi:10.1177/0278364911430419](https://doi.org/10.1177/0278364911430419)

**Bayes tree construction:** Eliminating the factor graph in variable order produces a Bayes
net (upper-triangular factor form). Chordal completion groups variables into cliques that
form a tree. Each clique stores a conditional density `p(x_C | x_{sep(C)})` where `sep(C)`
is the separator (shared variables between parent and child).

**Incremental update:** When a new factor connecting variables `{v₁, v₂}` arrives:
1. Find the cliques containing `v₁`, `v₂`; the top affected clique is their lowest common
   ancestor in the tree.
2. Remove the sub-tree rooted at that ancestor, regenerate it by re-elimination.
3. All other cliques remain cached — amortised O(1) update for typical robot trajectories.

### 5.1 ISAM2 API

```cpp
// gtsam/nonlinear/ISAM2.h
#include <gtsam/nonlinear/ISAM2.h>

ISAM2Params params;
params.relinearizeThreshold = 0.01;   // fraction of variable range
params.relinearizeSkip      = 1;      // check relinearization every update
ISAM2 isam(params);

// Per sensor-frame incremental update
NonlinearFactorGraph newFactors;
Values newValues;
// ... populate newFactors and newValues ...

isam.update(newFactors, newValues);
Values result = isam.calculateEstimate();   // full trajectory + map

// Clear the incremental accumulators
newFactors.resize(0);
newValues.clear();
```

`ISAM2::update()` differs fundamentally from the older `NonlinearISAM::update()`: it accepts
an *increment* (new factors and variable initialisations) and manages the Bayes tree
internally. `calculateEstimate()` back-substitutes from root to leaves in the cached tree —
no global solve needed.
[Source: github.com/borglab/gtsam/blob/develop/examples/ImuFactorsExample.cpp](https://github.com/borglab/gtsam/blob/develop/examples/ImuFactorsExample.cpp)

---

## 6. IMU Pre-Integration in Factor Graphs

An IMU delivers angular velocity `ω` and specific force `a` at 100–1000 Hz — far faster
than camera or LiDAR frames. **Pre-integrating** the IMU between keyframes produces a compact
relative-pose+velocity constraint (the pre-integrated measurement, PIM) that survives
re-linearisation without reprocessing raw data.
[Paper: Forster et al. 2017, doi:10.1109/TRO.2016.2597321](https://doi.org/10.1109/TRO.2016.2597321)

GTSAM's `CombinedImuFactor` jointly estimates pose, velocity, and IMU bias over a single
factor, which is more numerically stable than separate IMU + bias-random-walk factors:

```cpp
// gtsam/navigation/CombinedImuFactor.h
#include <gtsam/navigation/CombinedImuFactor.h>
using symbol_shorthand::V;   // Velocity3
using symbol_shorthand::B;   // imuBias::ConstantBias

// Parameters from IMU datasheet (e.g. Xsens MTi-30)
auto p = PreintegrationCombinedParams::MakeSharedU(9.81);
p->accelerometerCovariance = I_3x3 * 1e-3;    // m²/s⁴/Hz
p->gyroscopeCovariance     = I_3x3 * 1e-5;    // rad²/s²/Hz
p->integrationCovariance   = I_3x3 * 1e-8;    // numerical integration error
imuBias::ConstantBias zeroBias;
PreintegratedCombinedMeasurements pim(p, zeroBias);

// Integrate each IMU sample (dt in seconds, acc/gyro in sensor frame)
for (auto& [dt, acc, gyro] : imuBuffer)
    pim.integrateMeasurement(acc, gyro, dt);

// Single combined factor spanning keyframe i-1 → i
graph.emplace_shared<CombinedImuFactor>(
    X(i-1), V(i-1), X(i), V(i), B(i-1), B(i), pim);
```

Bias random-walk is modelled by a `BetweenFactor<imuBias::ConstantBias>` linking `B(i-1)`
and `B(i)`, with covariance set to the IMU's bias instability specification (typically
`1e-6 rad/s/√s` for MEMS gyroscopes, `1e-5 m/s²/√s` for accelerometers).

---

## 7. LiDAR-Inertial Odometry: FAST-LIO2

FAST-LIO2 is a tightly-coupled LiDAR-inertial odometry system that achieves real-time state
estimation (>100 Hz) without feature extraction, registering raw point clouds directly
against a continuously updated map via an iterated Kalman filter.
[Source: github.com/hku-mars/FAST_LIO](https://github.com/hku-mars/FAST_LIO);
[Paper: Xu et al. 2022, doi:10.1109/TRO.2021.3099048](https://doi.org/10.1109/TRO.2021.3099048)

### 7.1 LiDAR Hardware on Linux

Linux LiDAR drivers ship as ROS 2 packages or user-space SDKs:

| Sensor family | Linux driver | ROS 2 package |
|---|---|---|
| Ouster OS0/OS1/OS2 | `ouster-ros` | `ros-humble-ouster-ros` |
| Velodyne VLP-16/HDL-64 | `velodyne_driver` (UDP) | `ros-humble-velodyne` |
| Livox Mid-360 / Avia | `livox_ros_driver2` | vendor-provided |
| HESAI PandarXT-32 | `hesai_ros_driver` | vendor-provided |
| Intel L515 (solid-state ToF) | `librealsense2` | `ros-humble-realsense2` |

Points arrive as `sensor_msgs/PointCloud2` messages on a ROS 2 `/points` topic. FAST-LIO2
subscribes to `/points` and `/imu/data`, publishing `/Odometry` and `/cloud_registered`.

### 7.2 Point Cloud Preprocessing: Deskewing and Voxelisation

Spinning LiDARs (Velodyne, Ouster) rotate continuously; each laser beam is fired at a
different azimuth angle and therefore at a different timestamp within a single 100 ms scan.
If the sensor is moving, this produces **motion distortion**: the point cloud is smeared
relative to the true scene geometry. Solid-state LiDARs (Livox, Intel L515) have different
distortion patterns depending on their scan pattern (e.g. Livox's Lissajous or rosette).

**IMU-based deskewing** corrects each point `p_i` captured at relative time `τ_i` within
the scan by transforming it to the frame at the scan end-time `T`:

```
p_i^{deskewed} = R_{τ_i → T} p_i + t_{τ_i → T}
```

where `R_{τ_i → T}`, `t_{τ_i → T}` are interpolated from IMU-integrated poses between
`τ_i` and `T`. FAST-LIO2 performs this deskewing at the start of each scan before any
feature extraction or matching. Without deskewing, ATE typically increases substantially on sequences
with aggressive rotation, with the magnitude depending on scan rate and platform dynamics.

After deskewing, a **voxel downsampler** (voxel size typically 0.2–0.5 m) reduces the
point count from 65,536 (Ouster OS1-64) to ~2,000–5,000 representative points, bounding
IEKF computation time independent of sensor density. FAST-LIO2 uses Eigen's `VoxelGrid`
equivalent, implemented as a hash map over voxel indices:

```cpp
// Conceptual voxel downsampling (FAST-LIO2 / ikd-Tree insertion)
std::unordered_map<VoxelKey, PointXYZ> voxelMap;
float voxelSize = 0.3f;  // metres
for (auto& p : deskewedScan) {
    VoxelKey key = { (int)(p.x / voxelSize),
                     (int)(p.y / voxelSize),
                     (int)(p.z / voxelSize) };
    voxelMap[key] = p;   // last point per voxel survives
}
```

### 7.3 Iterated Extended Kalman Filter (IEKF)

FAST-LIO2 uses an **Iterated EKF** rather than a factor graph. The state vector lives on a
Lie manifold:

```
x = [R ∈ SO(3),  p ∈ ℝ³,  v ∈ ℝ³,  b_g ∈ ℝ³,  b_a ∈ ℝ³]
```

**Predict step** (IMU integration on SO(3) using the exponential map):

```
R̂_i = R_{i-1} Exp((ω - b_g) Δt)
p̂_i = p_{i-1} + v_{i-1} Δt + ½ (R_{i-1}(a - b_a) - g) Δt²
v̂_i = v_{i-1} + (R_{i-1}(a - b_a) - g) Δt
```

Covariance is propagated via the linearised noise Jacobians.

**Iterative update** (pseudocode, per LiDAR scan):

```
κ ← 0;  x^(0) ← x̂  (IMU prediction)
while not converged:
    for each point p_i in scan:
        q_i ← ikd-Tree.nearest(R^(κ) p_i + t^(κ))
        n_i ← local normal at q_i in map
        r_i ← n_i^T (R^(κ) p_i + t^(κ) - q_i)   // point-to-plane residual
    H ← Jacobian of {r_i} w.r.t. δx on the Lie algebra
    K ← P^(κ) H^T (H P^(κ) H^T + R_lidar)^{-1}  // Kalman gain
    δx ← K (−r) + K H (x̂ ⊖ x^(κ))              // correction on manifold
    x^(κ+1) ← x^(κ) ⊞ δx                         // retract to manifold
    κ ← κ + 1
P_i ← (I − K H) P^(κ)
```

Convergence is typically 3–5 iterations per scan. The Lie-group retraction (`⊞`) keeps the
rotation in SO(3) without normalisation overhead.

### 7.4 ikd-Tree: Incremental k-d Tree

The map is stored in an **ikd-Tree**, a dynamically balanced binary search tree supporting:

- **Incremental insertion** of new scan points without global rebuild.
- **Lazy deletion** (flagging removed points; actual removal during rebalance).
- **Bounded map size** via downsampling on insertion.
- **Approximate nearest-neighbour search** in O(log N) expected time.

The ikd-Tree enables FAST-LIO2's map to grow continuously without the per-scan rebuild cost
of a static k-d tree.
[Source: github.com/hku-mars/ikd-Tree](https://github.com/hku-mars/ikd-Tree)

### 7.5 Performance on Linux Hardware

On a laptop with an Intel i7-10750H (no GPU), FAST-LIO2 processes Ouster OS1-64 scans
(65,536 points, 10 Hz) in ~3 ms per scan (IEKF + ikd-Tree update). On Jetson Orin NX
(ARM Cortex-A78AE), the Eigen IEKF uses NEON SIMD automatically when built with
`-DENABLE_NEON=ON`. Build with `-DUSE_OPENMP=ON` to parallelise the nearest-neighbour
queries across ikd-Tree leaves.

FAST-LIO2 has no loop closure: positional drift accumulates over long traversals. For
closed-loop mapping use LIO-SAM (§8) or Cartographer (§9).

### 7.6 IMU-Camera-LiDAR Sensor Calibration

Accurate extrinsic calibration between IMU, camera, and LiDAR is prerequisite to tight
coupling. A miscalibrated extrinsic transform typically contributes more systematic error
than algorithmic differences between odometry methods.

**Kalibr** performs joint camera-IMU calibration via batch spline optimisation over an
AprilGrid target sequence:
[Source: github.com/ethz-asl/kalibr](https://github.com/ethz-asl/kalibr)

```bash
# Record a calibration bag: move sensor suite in front of static AprilGrid
#   camera at 20 Hz, IMU at 200 Hz, duration ~120 s, excite all axes
ros2 bag record -o calib_bag /camera/image_raw /imu/data

# Run camera-IMU calibration
kalibr_calibrate_imu_camera \
    --bag calib_bag.bag \
    --cam camera.yaml \              # intrinsics from kalibr_calibrate_cameras
    --imu imu.yaml \                 # noise-density and bias-random-walk from datasheet
    --target aprilgrid.yaml          # 6x6 grid, tag size 0.088 m
```

Outputs: `T_cam_imu` (4×4 SE(3) extrinsic), `time_offset` (camera-to-IMU timestamp shift,
typically ±5 ms for USB cameras). For LiDAR-IMU calibration, `lidar_align`
([github.com/ethz-asl/lidar_align](https://github.com/ethz-asl/lidar_align)) minimises
the scan-consistency error across the IMU-integrated trajectory to estimate `T_lidar_imu`.

---

## 8. LiDAR-Inertial SLAM with Loop Closure: LIO-SAM

FAST-LIO2 accumulates drift over long distances. **LIO-SAM** (LiDAR Inertial Odometry via
Smoothing and Mapping) adds a GTSAM factor-graph back-end that fuses IMU pre-integration,
feature-based LiDAR odometry, GPS priors, and loop-closure constraints via iSAM2.
[Source: github.com/TixiaoShan/LIO-SAM](https://github.com/TixiaoShan/LIO-SAM);
[Paper: Shan et al. 2020, doi:10.1109/IROS45743.2020.9341176](https://doi.org/10.1109/IROS45743.2020.9341176)

### 8.1 LOAM Feature Types (Background)

LIO-SAM inherits its front-end feature extraction from **LOAM** (LiDAR Odometry And Mapping),
the 2014 algorithm that established the edge/planar classification widely used in LiDAR SLAM.
[Paper: Zhang & Singh, RSS 2014](https://www.ri.cmu.edu/pub_files/2014/7/Ji_LidarMapping_RSS2014_v8.pdf)

LOAM projects the LiDAR ring scan into a range image and computes a smoothness value for
each point:

```
c(i) = (1 / (|S| × ||p_i||)) × ||Σ_{j ∈ S, j≠i} (p_i − p_j)||
```

where `S` is the neighbouring-points set in the same ring, and `||...||` is the Euclidean
norm of the summed displacement vector. Points with high `c` become **edge points** (sharp
corners, building edges); points with low `c` become **planar points** (flat ground, walls).
LIO-SAM uses these two classes as scan-matching features rather than raw points, reducing
the matching set to a small fraction of the raw scan — two orders of magnitude fewer than
FAST-LIO2's direct approach — at the cost of accuracy on feature-poor environments (open
fields, long corridors).

### 8.2 Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                           LIO-SAM                              │
│                                                                │
│  /imu/data ──► IMU Pre-integrator ─────────────────────────┐  │
│                       (PIM per keyframe interval)           │  │
│  /points ──► Feature Extraction ──► Scan Matching           │  │
│              (edge + planar pts)   (point-to-edge/plane)    │  │
│                                          │ relative pose     │  │
│  /gps/fix ───────────────────────────────┼──────────────────┤  │
│                                          │                  │  │
│                                ┌─────────▼──────────────────▼─┐│
│                                │    GTSAM iSAM2 Back-End      ││
│                                │  CombinedImuFactor            ││
│                                │  BetweenFactor<Pose3> (LiDAR) ││
│                                │  GPSFactor                    ││
│                                │  BetweenFactor<Pose3> (loop)  ││
│                                └──────────────────────────────┘│
│                                          │ optimised trajectory │
│                                          ▼                      │
│                               Global keyframe map              │
└────────────────────────────────────────────────────────────────┘
```

### 8.3 GTSAM Factor Graph Back-End

```cpp
// LIO-SAM-style back-end (simplified)
// Ref: github.com/TixiaoShan/LIO-SAM/blob/master/src/mapOptmization.cpp
#include <gtsam/nonlinear/ISAM2.h>
#include <gtsam/nonlinear/NonlinearFactorGraph.h>
#include <gtsam/nonlinear/Values.h>
#include <gtsam/slam/BetweenFactor.h>
#include <gtsam/navigation/CombinedImuFactor.h>
#include <gtsam/navigation/GPSFactor.h>
#include <gtsam/inference/Symbol.h>

using namespace gtsam;
using symbol_shorthand::X;  // Pose3
using symbol_shorthand::V;  // Velocity3
using symbol_shorthand::B;  // imuBias::ConstantBias

ISAM2Params isam2Params;
isam2Params.relinearizeThreshold = 0.1;
isam2Params.relinearizeSkip = 1;
ISAM2 isam(isam2Params);

NonlinearFactorGraph factors;
Values values;

// IMU factor (keyframe i-1 → i, from pre-integrated measurements)
factors.emplace_shared<CombinedImuFactor>(
    X(i-1), V(i-1), X(i), V(i), B(i-1), B(i), pim);

// LiDAR odometry factor (from feature-based scan matching)
auto lidarNoise = noiseModel::Diagonal::Sigmas(
    (Vector6() << 0.1, 0.1, 0.1, 0.01, 0.01, 0.01).finished());
factors.emplace_shared<BetweenFactor<Pose3>>(
    X(i-1), X(i), lidarRelativePose, lidarNoise);

// GPS factor (ENU coordinates, noise from horizontal dilution of precision)
auto gpsNoise = noiseModel::Diagonal::Sigmas(
    (Vector3() << gps.hdop, gps.hdop, gps.vdop).finished());
factors.emplace_shared<GPSFactor>(X(i), gps.enu, gpsNoise);

// Loop closure factor (only when Scan Context match found)
if (loopDetected) {
    auto loopNoise = noiseModel::Diagonal::Sigmas(
        (Vector6() << 0.3, 0.3, 0.3, 0.05, 0.05, 0.05).finished());
    factors.emplace_shared<BetweenFactor<Pose3>>(
        X(loopFrom), X(i), loopRelPose, loopNoise);
}

// Incremental optimisation
isam.update(factors, values);
isam.update();                              // second pass for re-linearisation
Values optimised = isam.calculateEstimate();
factors.resize(0);
values.clear();
```

### 8.4 Scan Context Loop Detection

LIO-SAM uses **Scan Context** to detect revisited locations without exhaustive ICP.
[Source: github.com/irapkaist/scancontext](https://github.com/irapkaist/scancontext);
[Paper: Kim & Kim 2018, doi:10.1109/IROS.2018.8593953](https://doi.org/10.1109/IROS.2018.8593953)

The Scan Context descriptor is a 2D histogram of max-height values from the LiDAR scan,
binned into `N_r` range rings × `N_s` sector columns (default 20×60). Each column is
independently rotation-invariant via a column-shift trick at query time. Candidate loop
pairs are found by:

1. Hashing each context descriptor into a **ring key** (column-wise max) and building a
   k-d tree over ring keys.
2. At query time: nearest-neighbour search in ring-key space → shortlist of candidates.
3. For each candidate: try all column-shift offsets; score by element-wise cosine similarity.
4. Accept if similarity > threshold (typically 0.35); reject otherwise.

Accepted candidates are then refined by ICP to produce the relative-pose constraint inserted
into the GTSAM factor graph.

### 8.5 LIO-SAM on ROS 2

The ROS 2 port runs as composable nodes (`imuPreintegration`, `featureExtraction`,
`mapOptimization`, `imageProjection`), enabling intra-process zero-copy for
`sensor_msgs/PointCloud2` via the DMA-BUF transport introduced in ROS 2 Galactic
(REP-2007 Type Adaptation Feature). On a mid-range x86 laptop, LIO-SAM sustains 10 Hz
keyframe insertion on campus-scale maps of ~500,000 points with sub-centimetre loop-closure
correction after revisit.

---

## 9. Submap-Based LiDAR SLAM: Google Cartographer

Google Cartographer is a real-time SLAM library supporting both 2D and 3D LiDAR operation,
with a **submap-based** architecture that decouples local trajectory building from global
pose-graph optimisation. It uses the Ceres Solver back-end rather than GTSAM.
[Source: github.com/cartographer-project/cartographer](https://github.com/cartographer-project/cartographer);
[Paper: Hess et al. ICRA 2016, doi:10.1109/ICRA.2016.7487258](https://doi.org/10.1109/ICRA.2016.7487258)

### 9.1 Submap Architecture

A **submap** is a local occupancy grid (2D) or truncated signed distance field (3D) built
from a fixed number of consecutive LiDAR scans (~100 scans, configurable). Once closed, a
submap is frozen and treated as a fixed map chunk for global alignment.

Submaps grow until they accumulate enough scans, then a new submap begins. At any moment
two submaps are active: the older one partially complete (to which loop closure can match),
and the newer one under active construction. This overlap prevents gaps at submap boundaries.

### 9.2 Local SLAM (Ceres-Based Scan Matching)

The local trajectory builder maintains a running pose estimate and inserts each scan into
the current active submap via two-stage matching:

1. **Correlative scan matcher**: exhaustive grid search over a discretised translational and
   rotational window (typical: ±0.5 m, ±30°). Fast because it operates on a pre-computed
   look-up table of the submap. Returns a coarse initial pose for the Ceres step.

2. **Ceres scan matcher**: fine non-linear optimisation minimising the sum of bilinear-
   interpolated occupancy values at each scan hit point, given the submap as a smooth field.
   The cost function is:

   ```
   E(ξ) = Σ_i (1 - M_smooth(T_ξ h_i))²
   ```

   where `M_smooth` is the bicubically interpolated submap, `T_ξ` the 2D rigid transform
   parametrised by `ξ = (x, y, θ)`, and `h_i` the hit points. The Jacobian is analytic via
   the map gradient.

IMU integration (gravity orientation only in 2D mode; full pre-integration in 3D mode) is
used to initialise the search window and to constrain roll/pitch in 3D operation.

### 9.3 Global SLAM (Loop Closure via Branch-and-Bound)

When a new submap is closed, the **constraint builder** attempts to match it against all
existing submaps within a search radius. Unlike LIO-SAM's descriptor-based loop detection,
Cartographer uses **branch-and-bound scan matching** — a hierarchical coarse-to-fine search
that provably finds the global optimum within the search window:

1. Build a multi-resolution occupancy grid pyramid from the candidate submap.
2. At the coarsest level, enumerate all translation/rotation candidates; prune branches
   whose upper-bound score is below the current best.
3. Recurse on surviving branches at progressively finer resolutions.
4. Accept if final match score exceeds a configured threshold.

Accepted constraints (which may be from distant parts of the trajectory) are added to a
**pose graph** solved by Ceres:

```cpp
// Conceptual Ceres residual for inter-submap constraint
// (Cartographer uses ceres::AutoDiffCostFunction internally)
struct SubmapConstraintCost {
    bool operator()(const double* pose_a, const double* pose_b,
                    double* residual) const {
        // relative pose error between the two submap origins
        // weighted by the match score covariance
        ...
        return true;
    }
    Rigid2d measured_relative_pose;
    ceres::Matrix2d sqrt_information;
};
```

The Ceres pose-graph problem is solved periodically (not per-scan), decoupling the real-time
local SLAM from the globally consistent optimisation.

### 9.4 ROS 2 Integration

```bash
# Install Cartographer and its ROS 2 wrapper
sudo apt install ros-humble-cartographer ros-humble-cartographer-ros

# Launch 2D LiDAR SLAM on a robot
ros2 launch cartographer_ros demo_revo_lds.launch.py \
    use_sim_time:=false

# Save the finished map
ros2 run cartographer_ros cartographer_pbstream_to_ros_map \
    --pbstream_file=/path/to/map.pbstream \
    --map_filestem=/path/to/output_map
```

Cartographer is widely deployed in production industrial robots (including Amazon Robotics
warehouse AMRs) because its global consistency guarantees are bounded: the branch-and-bound
loop detector never accepts a false positive that scores below the threshold, unlike
nearest-neighbour descriptors that can match similar-looking environments.

---

## 10. Visual-Inertial SLAM on Linux

LiDAR systems dominate outdoor robotics; inside XR headsets and in indoor AR, **visual-
inertial SLAM** (VI-SLAM) is the dominant approach — cameras and IMUs are cheaper, lighter,
and have lower power consumption than spinning LiDARs.

### 10.1 Basalt VIO (cross-reference Chapter 209)

Basalt is covered in Chapter 209 §10 in the context of Monado's `xrt_slam_sinks` OpenXR
tracking interface. Its key architectural differentiator is **square-root marginalisation**:
instead of marginalising out old states via the dense Schur complement (§4.1), Basalt
maintains a square-root factor (Cholesky factor) of the information matrix, achieving better
numerical stability at the cost of more complex bookkeeping.
[Source: gitlab.com/VladyslavUsenko/basalt](https://gitlab.com/VladyslavUsenko/basalt)

### 10.2 OpenVINS and Kimera-VIO

| System | Back-end | Map | Linux package | Headset support |
|---|---|---|---|---|
| OpenVINS | MSCKF (filter) | Point features | `ros-humble-openvins` | No |
| Kimera-VIO | GTSAM iSAM2 | Point features + 3D mesh | Source build | No |
| Basalt VIO | Square-root factor graph | Point + marginalization prior | Source (Monado) | Yes (via Monado) |
| ORB-SLAM3 | G2O local BA + DBoW2 LC | ORB map | Source build | Partial |

**OpenVINS** uses a Multi-State Constraint Kalman Filter (MSCKF), which avoids explicit
feature-state augmentation by projecting feature observations into the null space of their
position — constant-time updates regardless of map size, at the cost of weaker consistency.
[Source: github.com/rpng/open_vins](https://github.com/rpng/open_vins)

**Kimera-VIO** builds a metric-semantic 3D mesh map in real-time using a GTSAM iSAM2
back-end, making it the natural choice when a watertight 3D reconstruction is needed
alongside trajectory estimation.
[Source: github.com/MIT-SPARK/Kimera-VIO](https://github.com/MIT-SPARK/Kimera-VIO)

---

## 11. Learning-Based and Neural SLAM (Pointers to Chapter 114)

Chapter 114 covers GPU-accelerated computer vision in depth, including:

- **DROID-SLAM** (§14.2): deep visual SLAM using recurrent GRU updates and a Dense Bundle
  Adjustment (DBA) layer that solves a linearised bundle-adjustment system at video rate
  entirely on GPU. Trained end-to-end; 82% ATE reduction vs. classical methods on EuRoC.
  [Source: github.com/princeton-vl/DROID-SLAM](https://github.com/princeton-vl/DROID-SLAM)

- **SplaTAM** (§14.3): Gaussian-splatting SLAM that jointly tracks camera pose against an
  anisotropic 3DGS map and densifies the map with incoming RGB-D frames.
  [Source: github.com/spla-tam/SplaTAM](https://github.com/spla-tam/SplaTAM);
  [Paper: Keetha et al. CVPR 2024](https://arxiv.org/abs/2312.02126)

- **MonoGS** (§14.4): monocular Gaussian-splatting SLAM with geometric verification and
  shape regularisation; faster convergence than SplaTAM on texture-rich indoor scenes.
  [Source: github.com/muskie82/MonoGS](https://github.com/muskie82/MonoGS);
  [Paper: Matsuki et al. CVPR 2024](https://arxiv.org/abs/2312.06294)

**Neural implicit SLAM taxonomy:**

| Family | Scene representation | Tracking loss | Min. GPU VRAM |
|---|---|---|---|
| NeRF-based (iMAP 2021, NICE-SLAM 2022, Co-SLAM 2023) | MLP implicit | Rendering photometric | 8 GB |
| 3DGS-based (SplaTAM 2024, MonoGS 2024) | Anisotropic Gaussians | Rasteriser photometric | 8 GB |
| Hybrid TSDF + features (Vox-Fusion 2022) | TSDF voxels | SDF + photometric | 6 GB |

All systems require CUDA 11.8+ on Linux. Chapter 114 §14 covers the CUDA integration paths.

---

## 12. Benchmarks and Honest Evaluation

### 12.1 Standard Datasets

| Dataset | Sensors | Environments | Sequences | Primary metric |
|---|---|---|---|---|
| TUM RGB-D | Monocular + Xtion depth | Indoor | 39 | ATE [m] |
| EuRoC MAV | Stereo + VI-Sensor IMU | Indoor industrial | 11 | ATE [m] |
| KITTI Odometry | Stereo + Velodyne HDL-64 | Urban driving | 22 | Trans. error [%] |
| Hilti SLAM | Livox MID-70 + IMU + cam | Construction sites | 21 | Position RMSE [m] |
| MulRan | Ouster OS1 + Navtech radar | Korean cities | 4 env. | Relative pose error |

### 12.2 Evaluation Metrics

**ATE** (Absolute Trajectory Error): RMSE of per-pose Euclidean distance after rigid
alignment of estimated trajectory to ground truth:

```
ATE = √( (1/N) Σ_i ||p̂_i − S · p_i^gt||² )
```

where S is the least-squares SE(3) alignment transform.

**RPE** (Relative Pose Error): average drift over a fixed time window Δ, measuring
short-horizon accuracy rather than global consistency:

```
RPE_Δ = mean({ ||trans(Q_i⁻¹ Q_{i+Δ})||,  ||log(rot(Q_i⁻¹ Q_{i+Δ}))|| })
```

where `Q_i = (T̂_i)⁻¹ T_i^gt`.

### 12.3 Evaluation Tools

**evo** is the standard Linux command-line tool for trajectory evaluation. It is
format-agnostic, actively maintained, and used in nearly all recent SLAM papers to report
ATE and RPE.
[Source: github.com/MichaelGrupp/evo](https://github.com/MichaelGrupp/evo)

```bash
pip install evo --upgrade

# Absolute trajectory error (TUM format)
evo_ape tum ground_truth.txt estimated.txt -a --plot

# Relative pose error over 1-second windows (EuRoC CSV format)
evo_rpe euroc groundtruth.csv estimated.csv \
    --delta 1 --delta_unit s --plot

# Compare multiple methods on the same ground truth
evo_ape tum gt.txt fast-lio2.txt liosam.txt cartographer.txt \
    -a --plot --save_results results/
```

evo supports TUM (`x y z qx qy qz qw` per line), KITTI (3×4 matrix per line), EuRoC CSV,
and ROS 2 bag formats. The `--align` flag (`-a`) computes the Umeyama alignment before
computing ATE; omit it for raw odometry-drift evaluation where accumulated drift is the
quantity of interest. Use `evo_res` to aggregate multiple runs and compute statistics across
datasets for a publication-ready comparison table.

### 12.4 SLAM System Selection Guide for Linux

The choice between SLAM systems on Linux is frequently driven by three factors: sensor type,
loop-closure requirements, and GPU availability. The following table summarises:

| System | Sensor | Loop closure | Back-end | GPU needed | Best for |
|---|---|---|---|---|---|
| slam_toolbox | 2D LiDAR | Scan matching (Ceres) | Ceres | No | Indoor 2D Nav2 robots |
| Cartographer 2D | 2D LiDAR | Branch-and-bound | Ceres | No | Industrial, guaranteed no false positive |
| Cartographer 3D | 3D LiDAR | Branch-and-bound | Ceres | No | Outdoor 3D, structured envs |
| FAST-LIO2 | 3D LiDAR + IMU | None (odometry only) | IEKF | No | Fast, feature-sparse envs |
| LIO-SAM | 3D LiDAR + IMU | Scan Context | GTSAM iSAM2 | No | Outdoor, GPS optional |
| DROID-SLAM | Monocular/stereo | Learned retrieval | CUDA DBA | Yes (CUDA) | Texture-rich, camera-only |
| MonoGS / SplaTAM | RGB-D / monocular | Visual place recognition | 3DGS | Yes (CUDA) | Dense photorealistic mapping |

**slam_toolbox** is the de-facto standard for 2D navigation in ROS 2:
`apt install ros-humble-slam-toolbox`. It uses a continuous scanning update (not submap-
based) with Ceres optimisation and supports map serialisation to `.posegraph` files for
lifelong mapping across robot restarts. Unlike Cartographer, slam_toolbox supports
**localisation-only mode** natively via `ros2 launch slam_toolbox localization.launch.py`.
[Source: github.com/SteveMacenski/slam_toolbox](https://github.com/SteveMacenski/slam_toolbox)

### 12.5 Common Evaluation Pitfalls

- **Training-set leakage**: learning-based systems (DROID-SLAM, MonoGS) may train on EuRoC
  or TUM; confirm that reported test sequences are held out from training.
- **Selective sequence reporting**: some papers omit the hardest sequences (EuRoC `V2_03
  difficult`); compare against the full benchmark table when evaluating for production use.
- **Loop closure on/off**: ATE for LIO-SAM drops dramatically with loop closure enabled;
  always record whether results are odometry-only or full SLAM.
- **Sensor calibration quality**: miscalibrated IMU-LiDAR extrinsics contribute more
  systematic error than algorithmic differences; report calibration method alongside ATE.
- **Hardware dependency**: DROID-SLAM and 3DGS-SLAM results are meaningless without
  reporting GPU model and VRAM; classical methods should report CPU model and core count.

---

## 13. System Integration: End-to-End LiDAR SLAM on Linux

The following walks through a complete FAST-LIO2 deployment on Ubuntu 22.04 with a real
Ouster OS1-64 sensor, illustrating the full ROS 2 data path from raw UDP packets to a
globally consistent map in RViz2.

### 13.1 Installation

```bash
# ROS 2 Humble (Ubuntu 22.04)
sudo apt install ros-humble-desktop ros-humble-pcl-ros

# Ouster ROS 2 driver (produces /points PointCloud2 + /imu sensor_msgs/Imu)
sudo apt install ros-humble-ouster-ros

# FAST-LIO2 (source build — requires Eigen 3.3+, PCL 1.10+)
cd ~/ros2_ws/src
git clone https://github.com/hku-mars/FAST_LIO --recursive
cd ~/ros2_ws && colcon build --packages-select fast_lio --cmake-args -DUSE_OPENMP=ON
```

### 13.2 Sensor Configuration and Launch

```bash
# Start the Ouster driver with hardware timestamps enabled
ros2 launch ouster_ros sensor.launch.xml sensor_hostname:=192.168.1.100 \
    timestamp_mode:=TIME_FROM_PTP_1588

# Launch FAST-LIO2 (Ouster OS1-64 config ships in the package)
ros2 launch fast_lio mapping.launch.xml config_file:=ouster64.yaml

# Visualise in RViz2
ros2 run rviz2 rviz2 -d ~/ros2_ws/src/FAST_LIO/config/rviz2.rviz
```

The `ouster64.yaml` specifies the IMU topic, scan topic, LiDAR scan rate (10 Hz), voxel
size (0.5 m), and IEKF convergence threshold. The driver publishes IMU at 100 Hz and the
LiDAR at 10 Hz; FAST-LIO2 internally synchronises these by IMU timestamp interpolation.

### 13.3 Map Saving and Reuse

FAST-LIO2 publishes the live point-cloud map on `/cloud_registered`. For persistent storage,
pipe it through `pcl_ros` to a PCD file, then reload with `map_server` or serve it as a
static `octomap`:

```bash
# Save map to PCD while running
ros2 run pcl_ros pointcloud_to_pcd \
    --ros-args -r input:=/cloud_registered

# Convert PCD to OctoMap binary format (standalone tool from octomap-tools)
# sudo apt install octomap-tools
pcd2bt cloud.pcd cloud.bt
```

For LIO-SAM, save and reload the pose graph via the `save_map` ROS 2 service:
`ros2 service call /save_map std_srvs/srv/Empty`. This serialises the GTSAM `Values`
(poses) and the keyframe point clouds to disk, enabling localisation-only mode on restart.

## 14. GPU and CPU Parallelism on Linux

### 14.1 CUDA Requirements per System

| System | CUDA usage | Minimum NVIDIA GPU |
|---|---|---|
| DROID-SLAM | DBA linear solver, correlation volumes | RTX 3060 / A2000 |
| SplaTAM / MonoGS | 3DGS differentiable rasteriser | RTX 3080 (8 GB VRAM) |
| Cartographer (3D) | None (CPU Ceres) | — |
| FAST-LIO2 | None (CPU Eigen + SIMD) | — |
| LIO-SAM | None (GTSAM Cholesky on CPU) | — |

CUDA is exposed on Linux via the `nvidia-open` kernel module (Turing+, since driver 515) or
the proprietary `nvidia` module. Verify with `nvidia-smi` that `CUDA Version: ≥ 11.8` is
reported.

### 14.2 CPU Threading for SLAM Pipelines

FAST-LIO2 uses three threads:
1. **IMU thread**: integrates IMU samples at full rate, propagates covariance on SO(3).
2. **LiDAR odometry thread**: IEKF predict + iterative point-to-plane update + ikd-Tree.
3. **Visualisation thread**: publishes `sensor_msgs/PointCloud2` to RViz2 at 10 Hz.

Pin threads to dedicated CPU cores via `pthread_setaffinity_np()` to minimise cache
contention. On NUMA systems (dual-socket x86), bind the IMU and LiDAR threads to the
same NUMA node as the PCIe LiDAR bus to avoid cross-socket cache misses.

Cartographer uses a separate thread pool for the constraint builder (global SLAM) and a
dedicated thread for each trajectory's local SLAM. The `num_background_threads` Lua
parameter controls the constraint-builder pool size; set it to `nproc - 2` on dedicated
mapping hardware.

### 14.3 Vulkan Compute as a CUDA Alternative

For production systems that cannot depend on NVIDIA hardware (e.g. embedded ARM SoCs with
Mali or PowerVR GPUs), Vulkan compute provides a vendor-neutral GPU path. The `kompute`
library ([github.com/KomputeProject/kompute](https://github.com/KomputeProject/kompute))
provides a Vulkan-compute tensor abstraction that can replace simple CUDA kernels (e.g.
point-cloud voxelisation, nearest-neighbour table construction). Full deep-SLAM migration to
Vulkan remains a research-level effort; the DBA layer in DROID-SLAM and the 3DGS rasteriser
in SplaTAM/MonoGS depend heavily on CUDA-specific primitives (warp shuffle, cooperative
groups) with no direct Vulkan equivalents as of mid-2025.

---

## 15. Integrations

- **Chapter 114 (OpenCV + GPU Vision)** — §14 covers DROID-SLAM, SplaTAM, and MonoGS
  with full CUDA implementation detail; §11 here is a taxonomy cross-reference.

- **Chapter 209 (OpenSLAM: GMapping, g2o, TORO, Basalt)** — §3–4 of Chapter 209 cover
  GMapping particle-filter SLAM (§2.3 here) and g2o factor-graph optimisation (an
  alternative to GTSAM's §3–5 here). The `xrt_slam_sinks` Monado interface (ch209 §10)
  accepts Basalt and OpenVINS back-ends described in §10 here.

- **Chapter 208 (Monado / OpenXR)** — Monado's `xrt_slam_sinks` struct connects VI-SLAM
  back-ends (Basalt, Kimera-VIO, OpenVINS) to headset driver pose output; the iSAM2 window
  (§5) is conceptually equivalent to Basalt's square-root marginalisation window.

- **Chapter 206 (V4L2 and libcamera)** — Camera capture pipelines feeding visual SLAM
  front-ends; hardware timestamping for sub-millisecond IMU-camera synchronisation;
  DMA-BUF zero-copy from kernel capture buffers to SLAM feature extractors.

- **Chapter 205 (ROS 2 middleware)** — `sensor_msgs/PointCloud2`, `sensor_msgs/Imu`, and
  `nav_msgs/Odometry` message types; intra-process zero-copy via REP-2007 type adaptation
  (relevant to LIO-SAM's composable-node point-cloud pipeline); TF2 transform tree
  publication by all SLAM systems.

- **Chapter 203 (Vulkan compute)** — Vendor-neutral GPU compute path for point-cloud
  preprocessing; `VK_KHR_acceleration_structure` as a potential hardware-accelerated
  nearest-neighbour search to replace the ikd-Tree for GPU-resident maps.
