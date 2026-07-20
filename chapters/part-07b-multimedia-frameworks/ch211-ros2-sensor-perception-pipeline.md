# Chapter 211: ROS 2 Multimodal Sensor and Perception Pipeline

**Target audiences:** Systems and driver developers integrating sensor hardware into Linux robots;
graphics application developers building perception systems on Vulkan/CUDA; robot software
engineers who need to understand the plumbing between sensors, message buses, and inference
engines on Ubuntu-class Linux.

This chapter covers the ROS 2 (Humble/Jazzy) sensor middleware stack: the DDS transport and
QoS layer, the canonical `sensor_msgs` and `vision_msgs` message taxonomy, the TF2 transform
tree, `image_transport` plugin architecture, multi-sensor EKF/GPS fusion via
`robot_localization`, composable-node zero-copy, and NVIDIA Isaac ROS/NITROS for GPU-resident
perception pipelines. Model training, CUDA kernels, and neural SLAM internals are in
Chapter 114 (OpenCV/GPU vision) and Chapter 210 (SLAM theory); camera hardware and V4L2
pipeline internals are in Chapter 96 (libcamera). The three chapters together form the
complete Linux robot perception stack.

---

## Table of Contents

1. [ROS 2 Middleware: DDS, rmw, and QoS](#1-ros-2-middleware-dds-rmw-and-qos)
   - [1.1 rmw Abstraction and DDS Implementations](#11-rmw-abstraction-and-dds-implementations)
   - [1.2 QoS Profiles for Sensor Data](#12-qos-profiles-for-sensor-data)
   - [1.3 Composable Node Containers and Intra-Process Communication](#13-composable-node-containers-and-intra-process-communication)
2. [Sensor Message Taxonomy: sensor_msgs](#2-sensor-message-taxonomy-sensor_msgs)
   - [2.1 The sensor_msgs Roster](#21-the-sensor_msgs-roster)
   - [2.2 PointCloud2 Internals](#22-pointcloud2-internals)
   - [2.3 IMU, NavSatFix, LaserScan, and Range](#23-imu-navsatfix-laserscan-and-range)
   - [2.4 point_cloud_transport: Compressed LiDAR Streams](#24-point_cloud_transport-compressed-lidar-streams)
3. [The TF2 Transform Tree and URDF](#3-the-tf2-transform-tree-and-urdf)
   - [3.1 Static and Dynamic Transforms](#31-static-and-dynamic-transforms)
   - [3.2 URDF and robot_state_publisher](#32-urdf-and-robot_state_publisher)
   - [3.3 tf2_ros C++ Lookup API](#33-tf2_ros-c-lookup-api)
4. [Camera Pipeline: image_transport and CameraInfo](#4-camera-pipeline-image_transport-and-camerainfo)
   - [4.1 image_transport Plugin Architecture](#41-image_transport-plugin-architecture)
   - [4.2 Camera Calibration and CameraInfo](#42-camera-calibration-and-camerainfo)
   - [4.3 Depth Camera Integration](#43-depth-camera-integration)
   - [4.4 depth_image_proc: Depth to Point Cloud](#44-depth_image_proc-depth-to-point-cloud)
   - [4.5 Message Synchronisation with message_filters](#45-message-synchronisation-with-message_filters)
5. [vision_msgs: Unified Perception Interface](#5-vision_msgs-unified-perception-interface)
   - [5.1 Message Type Taxonomy](#51-message-type-taxonomy)
   - [5.2 Detection2DArray and Detection3DArray](#52-detection2darray-and-detection3darray)
   - [5.3 nav_msgs: Perception Pipeline Outputs](#53-nav_msgs-perception-pipeline-outputs)
6. [Object Detection on ROS 2](#6-object-detection-on-ros-2)
   - [6.1 ultralytics_ros: YOLO on ROS 2 Humble](#61-ultralytics_ros-yolo-on-ros-2-humble)
   - [6.2 Wiring Detection Output to Downstream Nodes](#62-wiring-detection-output-to-downstream-nodes)
7. [Multi-Sensor Fusion: robot_localization](#7-multi-sensor-fusion-robot_localization)
   - [7.1 ekf_filter_node: EKF and UKF Modes](#71-ekf_filter_node-ekf-and-ukf-modes)
   - [7.2 GPS/GNSS Integration: navsat_transform_node](#72-gpsgnss-integration-navsat_transform_node)
   - [7.3 Full Fusion Configuration Example](#73-full-fusion-configuration-example)
8. [GPU Acceleration: Isaac ROS and NITROS](#8-gpu-acceleration-isaac-ros-and-nitros)
   - [8.1 Type Adaptation (REP-2007) and Type Negotiation (REP-2009)](#81-type-adaptation-rep-2007-and-type-negotiation-rep-2009)
   - [8.2 Isaac ROS Package Landscape](#82-isaac-ros-package-landscape)
   - [8.3 DNN Encoder → Triton → DetectNet Pipeline](#83-dnn-encoder--triton--detectnet-pipeline)
   - [8.4 Nvblox: GPU-Accelerated 3D Reconstruction](#84-nvblox-gpu-accelerated-3d-reconstruction)
9. [Diagnostics and Performance Profiling](#9-diagnostics-and-performance-profiling)
10. [System Integration: End-to-End Perception Stack](#10-system-integration-end-to-end-perception-stack)
    - [10.4 Data Recording with rosbag2 and MCAP](#104-data-recording-with-rosbag2-and-mcap)
11. [Integrations](#11-integrations)

---

## 1. ROS 2 Middleware: DDS, rmw, and QoS

### 1.1 rmw Abstraction and DDS Implementations

ROS 2 decouples application code from the underlying data transport through the **ROS Middleware
interface (rmw)**. The rmw layer is a thin C API (`rmw/rmw.h`) that every DDS vendor or
transport implementation must satisfy; application code calls `rclcpp`/`rclpy` which calls rmw
which calls the vendor library at runtime.
[Source: github.com/ros2/rmw](https://github.com/ros2/rmw)

Two DDS implementations dominate on Linux:

| Implementation | apt package | `RMW_IMPLEMENTATION` env var | Use case |
|---|---|---|---|
| eProsima FastDDS | `ros-humble-rmw-fastrtps-cpp` | `rmw_fastrtps_cpp` | Default for Humble and Jazzy; configurable QoS |
| Eclipse CycloneDDS | `ros-humble-rmw-cyclonedds-cpp` | `rmw_cyclonedds_cpp` | Low-latency alternative; was default in Galactic |
| Zenoh (via rmw_zenoh) | `ros-humble-rmw-zenoh-cpp` | `rmw_zenoh_cpp` | Cloud-edge bridging; Jazzy+ |

Switch DDS at runtime without recompilation:

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
ros2 run my_pkg my_node
```

For multi-machine setups, DDS discovery is UDP multicast by default. Set a shared domain ID to
scope discovery to a subnet:

```bash
export ROS_DOMAIN_ID=42   # integer 0–101; must match across all hosts
```

On networks without multicast, configure `FASTRTPS_DEFAULT_PROFILES_FILE` or a CycloneDDS XML
peer list for unicast peer-to-peer discovery.

### 1.2 QoS Profiles for Sensor Data

ROS 2 inherits DDS Quality of Service (QoS) policies. The combination of **Reliability**,
**Durability**, **History**, and **Depth** must match between publisher and subscriber; a
mismatch causes silent incompatibility — no message is delivered and no error is raised.

```cpp
#include <rclcpp/rclcpp.hpp>
#include <sensor_msgs/msg/point_cloud2.hpp>

// Sensor-data profile: best-effort, keep-last-1, volatile
// Use for high-rate data (LiDAR, camera) where losing a frame is acceptable
auto qos_sensor = rclcpp::SensorDataQoS();  // preset: best-effort + depth-5

// Services / odometry: reliable, volatile
auto qos_reliable = rclcpp::QoS(10)
    .reliability(rclcpp::ReliabilityPolicy::Reliable)
    .durability(rclcpp::DurabilityPolicy::Volatile);

// Lifecycle / map topics: transient-local (late-joiners get last message)
auto qos_map = rclcpp::QoS(1)
    .reliability(rclcpp::ReliabilityPolicy::Reliable)
    .durability(rclcpp::DurabilityPolicy::TransientLocal);

auto sub = node->create_subscription<sensor_msgs::msg::PointCloud2>(
    "/points", qos_sensor,
    [](sensor_msgs::msg::PointCloud2::SharedPtr msg) { /* ... */ });
```

`rclcpp::SensorDataQoS()` is the standard preset for camera and LiDAR topics — use it on both
publisher and subscriber to guarantee compatibility.
[Source: docs.ros.org/en/humble/Concepts/About-Quality-of-Service-Settings.html](https://docs.ros.org/en/humble/Concepts/About-Quality-of-Service-Settings.html)

### 1.3 Composable Node Containers and Intra-Process Communication

A **composable node** is a `rclcpp::Node` loaded as a shared library into a common process via
`rclcpp_components::NodeFactory`. When two composable nodes subscribe and publish the same
topic within the same container process, `rclcpp` can skip serialisation entirely — the
`shared_ptr<Msg>` is passed directly across the subscription callback boundary, achieving
true zero-copy for messages that fit within a single allocator block.

```bash
# Launch a composable container and load nodes into it
ros2 launch my_robot perception_composable.launch.py
```

```python
# perception_composable.launch.py
from launch_ros.actions import ComposableNodeContainer
from launch_ros.descriptions import ComposableNode

container = ComposableNodeContainer(
    name='perception_container',
    namespace='',
    package='rclcpp_components',
    executable='component_container',
    composable_node_descriptions=[
        ComposableNode(
            package='image_proc',
            plugin='image_proc::RectifyNode',
            name='rectify'),
        ComposableNode(
            package='depth_image_proc',
            plugin='depth_image_proc::PointCloudXyzrgbNode',
            name='depth_to_cloud'),
    ],
    output='screen',
)
```

For larger messages (point clouds, full-resolution images), intra-process zero-copy still
requires the publisher to use `rclcpp::PublisherOptionsWithAllocator` with a loaned-message
allocator, or to rely on the **type adaptation** mechanism described in §8.1.
[Source: docs.ros.org/en/humble/How-To-Guides/Using-ros2-launch-for-large-projects.html](https://docs.ros.org/en/humble/How-To-Guides/Using-ros2-launch-for-large-projects.html)

---

## 2. Sensor Message Taxonomy: sensor_msgs

### 2.1 The sensor_msgs Roster

`sensor_msgs` is the lingua franca of ROS 2 hardware. Every sensor driver publishes one or
more of these standard types; every perception algorithm subscribes to them. Using standard
message types is what allows drivers (from different vendors) and algorithms (from different
research groups) to compose without custom interface code.
[Source: github.com/ros2/common_interfaces/tree/humble/sensor_msgs](https://github.com/ros2/common_interfaces/tree/humble/sensor_msgs)

| Message | Typical topic | Description |
|---|---|---|
| `Image` | `/camera/image_raw` | Raw image frame (width, height, encoding, step, data[]) |
| `CompressedImage` | `/camera/image_raw/compressed` | JPEG/PNG-encoded image |
| `CameraInfo` | `/camera/camera_info` | Intrinsics, distortion, projection matrix |
| `PointCloud2` | `/points` | Binary-encoded N-dimensional point array |
| `Imu` | `/imu/data` | Linear acceleration, angular velocity, orientation |
| `NavSatFix` | `/gps/fix` | GNSS position (lat/lon/alt + covariance) |
| `LaserScan` | `/scan` | 2D LiDAR range array (start/end angle, ranges[], intensities[]) |
| `Range` | `/sonar` | Single-beam ultrasonic or IR range measurement |
| `MagneticField` | `/imu/mag` | 3-axis magnetometer |
| `FluidPressure` | `/baro` | Barometric pressure (altitude estimation) |
| `JointState` | `/joint_states` | Position/velocity/effort for each joint |
| `BatteryState` | `/battery_state` | Voltage, current, percentage, cell voltages |

### 2.2 PointCloud2 Internals

`sensor_msgs/msg/PointCloud2` stores point data as a flat binary blob with a schema
description — it is not a list of structs but a single `uint8[]` buffer with per-point fields
described by `PointField` entries:

```
Header header          # frame_id ties the cloud to the TF tree
uint32 height          # 1 for unorganised; N_rows for organised (spinning LiDAR)
uint32 width           # points per row (= total points if height==1)
sensor_msgs/PointField[] fields  # schema: name, offset, datatype, count per field
bool   is_bigendian
uint32 point_step      # bytes per point (e.g. 16 for XYZI float32×4)
uint32 row_step        # bytes per row = point_step × width
uint8[] data           # the raw payload
bool   is_dense        # false if NaN points are present
```

The `PointField` datatype codes are: `INT8=1`, `UINT8=2`, `INT32=5`, `FLOAT32=7`, `FLOAT64=8`, etc.
PCL's `sensor_msgs::msg::PointCloud2` helpers (`pcl::fromROSMsg`, `pcl::toROSMsg`) automate
field packing and unpacking.

An Ouster OS1-64 at 10 Hz produces organised clouds of 64 × 1024 = 65,536 points. Each
point is 16 bytes (x, y, z as float32 + intensity as float32), giving a 1 MB message at
10 Hz — 10 MB/s before any compression. This is why composable-node zero-copy and QoS
best-effort are used together for LiDAR pipelines (§1.3, §1.2).

### 2.3 IMU, NavSatFix, LaserScan, and Range

**IMU** (`sensor_msgs/msg/Imu`): carries `angular_velocity` (3D, rad/s), `linear_acceleration`
(3D, m/s²), `orientation` (quaternion), and 3×3 covariance matrices for each. Drivers that
do not provide orientation fill `orientation_covariance[0] = -1` as a sentinel — the EKF in
`robot_localization` checks this flag before fusing.

**NavSatFix** (`sensor_msgs/msg/NavSatFix`): latitude (degrees), longitude (degrees), altitude
(m, WGS-84 ellipsoid), `status.status` (`STATUS_FIX=0`, `STATUS_SBAS_FIX=1`,
`STATUS_GBAS_FIX=2`), and a 3×3 ENU position covariance in m².

**LaserScan** (`sensor_msgs/msg/LaserScan`): `angle_min`, `angle_max`, `angle_increment` (rad),
`time_increment` (s per sample), `range_min`/`range_max` (m), `ranges[]` (m, NaN for no
return), `intensities[]`. Used by 2D LiDARs (Hokuyo, RPLIDAR) and by the `pointcloud_to_laserscan`
node which projects PointCloud2 slices into 2D for Navigation2.

### 2.4 point_cloud_transport: Compressed LiDAR Streams

Point clouds are large. At 10 Hz an Ouster OS1-64 generates ~10 MB/s of raw
`PointCloud2` data. `point_cloud_transport` is the LiDAR analogue of `image_transport`:
it transparently multiplexes a `/points` topic through compression plugins, letting
bandwidth-constrained subscribers (remote monitoring, WiFi-connected operators) choose a
compressed transport without the publisher knowing.
[Source: github.com/ros-perception/point_cloud_transport](https://github.com/ros-perception/point_cloud_transport)

Available plugins:

| Plugin | Sub-topic suffix | Algorithm | Notes |
|---|---|---|---|
| `raw` (built-in) | (base topic) | None | Intra-process, LAN |
| `draco` | `/draco` | Google Draco (lossy) | ~5–15× compression; geometry only |
| `zlib` | `/zlib` | Zlib (lossless) | Moderate ratio, fast decode |
| `zstd` | `/zstd` | Zstandard (lossless) | Best ratio/speed trade-off |

C++ publisher/subscriber API mirrors `image_transport`:

```cpp
#include <point_cloud_transport/point_cloud_transport.hpp>

point_cloud_transport::PointCloudTransport pct(node);

// Publisher: advertises both raw and all installed compressed sub-topics
point_cloud_transport::Publisher pub = pct.advertise("points", 1);

// Subscriber: transport selected at runtime via ~point_cloud_transport parameter
point_cloud_transport::Subscriber sub = pct.subscribe(
    "points", 1,
    [](const sensor_msgs::msg::PointCloud2::ConstSharedPtr& cloud) {
        // cloud is always PointCloud2 regardless of wire transport
    });
```

Select the transport at runtime:

```bash
ros2 run my_viz cloud_viewer --ros-args \
    -r points:=/velodyne_points \
    -p point_cloud_transport:=zstd
```

---

## 3. The TF2 Transform Tree and URDF

### 3.1 Static and Dynamic Transforms

Every sensor reading in ROS 2 carries a `frame_id` string in its `Header`. The TF2 system
maintains a tree of coordinate frames and time-stamped transforms between them. Nodes can
query "where was sensor A at time T relative to frame B?" across arbitrary frame chains.

The canonical robot coordinate frames follow **REP-105**:
- `map` — global fixed frame (origin = SLAM map origin)
- `odom` — drifting odometry frame (origin = robot start pose)
- `base_link` — robot body centre (defined by URDF)
- `base_footprint` — projection of `base_link` onto the ground plane
- `camera_link`, `lidar_link`, `imu_link` — sensor mounts

The transform chain is: `map → odom → base_link → sensor_link`. Each `→` is a
`geometry_msgs/TransformStamped` broadcast by a node:

```cpp
#include <tf2_ros/transform_broadcaster.h>
#include <geometry_msgs/msg/transform_stamped.hpp>

tf2_ros::TransformBroadcaster tf_broadcaster(node);

geometry_msgs::msg::TransformStamped ts;
ts.header.stamp    = node->now();
ts.header.frame_id = "odom";
ts.child_frame_id  = "base_link";
ts.transform.translation.x = pose.x;
ts.transform.translation.y = pose.y;
ts.transform.translation.z = 0.0;
ts.transform.rotation = tf2::toMsg(q);  // tf2::Quaternion q from heading

tf_broadcaster.sendTransform(ts);
```

Static transforms (sensor extrinsics, fixed mounts) are broadcast once via
`tf2_ros::StaticTransformBroadcaster` or the `static_transform_publisher` node:

```bash
ros2 run tf2_ros static_transform_publisher \
    --x 0.1 --y 0.0 --z 0.3 \
    --roll 0 --pitch 0 --yaw 0 \
    --frame-id base_link --child-frame-id lidar_link
```

### 3.2 URDF and robot_state_publisher

The **Unified Robot Description Format (URDF)** is an XML file describing the robot's links
(rigid bodies), joints (kinematic connections), and sensor frame positions. The
`robot_state_publisher` node parses the URDF, listens to `/joint_states`, and broadcasts
the resulting transforms:

```bash
ros2 run robot_state_publisher robot_state_publisher \
    --ros-args -p robot_description:="$(xacro my_robot.urdf.xacro)"
```

For mobile robots without articulated joints (ground vehicles, drones), the URDF defines
only the static extrinsics between sensor frames and `base_link`; the dynamic
`odom → base_link` transform is published by the odometry or SLAM node.

### 3.3 tf2_ros C++ Lookup API

```cpp
#include <tf2_ros/buffer.h>
#include <tf2_ros/transform_listener.h>

auto tf_buffer = std::make_shared<tf2_ros::Buffer>(node->get_clock());
tf2_ros::TransformListener tf_listener(*tf_buffer);

// Look up the transform from lidar_link to base_link at the time of a scan
geometry_msgs::msg::TransformStamped t;
try {
    t = tf_buffer->lookupTransform(
        "base_link",        // target frame
        "lidar_link",       // source frame
        scan_msg->header.stamp,
        rclcpp::Duration::from_seconds(0.1));  // timeout
} catch (const tf2::LookupException& ex) {
    RCLCPP_WARN(node->get_logger(), "TF lookup failed: %s", ex.what());
}
```

`lookupTransform` blocks until the transform is available or the timeout expires. Use
`canTransform()` for non-blocking checks. The TF2 buffer keeps a configurable history window
(default 10 s) to handle out-of-order sensor messages.

---

## 4. Camera Pipeline: image_transport and CameraInfo

### 4.1 image_transport Plugin Architecture

Raw camera frames are large: a 1080p RGB8 image is 6.2 MB. Transmitting raw images at 30 Hz
is 186 MB/s — acceptable on gigabit LAN but prohibitive over WiFi or across ROS 2 domain
boundaries. `image_transport` is a ROS 2 library that transparently multiplexes an image
topic across multiple **transport plugins**, letting subscribers select their preferred
encoding without requiring the publisher to know in advance.
[Source: github.com/ros-perception/image_transport](https://github.com/ros-perception/image_transport)

Available plugins (`apt install ros-humble-image-transport-plugins`):

| Plugin name | Sub-topic suffix | Encoding | Typical use |
|---|---|---|---|
| `raw` (built-in) | (base topic) | `sensor_msgs/Image` | Intra-process, LAN |
| `compressed` | `/compressed` | JPEG or PNG | WiFi, bandwidth-limited links |
| `compressed_depth` | `/compressedDepth` | PNG16 + RVL | Depth over network |
| `theora` | `/theora` | Ogg Theora video | Video recording |
| `zstd` | `/zstd` | Zstandard frame | CPU-efficient compression |

Publisher side:

```cpp
#include <image_transport/image_transport.hpp>
#include <sensor_msgs/msg/image.hpp>

image_transport::ImageTransport it(node);
image_transport::Publisher pub = it.advertise("camera/image", 1);

sensor_msgs::msg::Image img_msg;
// ... fill img_msg.header, encoding, width, height, step, data ...
pub.publish(img_msg);
```

Subscriber side — transport selected at runtime via `~image_transport` parameter:

```bash
ros2 run my_viewer view_camera --ros-args \
    -r image:=/camera/image \
    -p image_transport:=compressed
```

The `image_proc` package provides standard processing nodes as composable components —
`RectifyNode` (undistortion), `CropDecimateNode`, `DebayerNode`, `ResizeNode` — all
consuming `sensor_msgs/Image` and `sensor_msgs/CameraInfo`.

### 4.2 Camera Calibration and CameraInfo

`sensor_msgs/msg/CameraInfo` carries the calibration data needed to project 3D points to
pixels:

```
Header header
uint32 height, width
string distortion_model   # "plumb_bob" or "equidistant"
float64[] D               # distortion coefficients (5 for plumb_bob)
float64[9]  K             # 3x3 camera matrix (intrinsics)
float64[9]  R             # 3x3 rectification matrix (stereo only)
float64[12] P             # 3x4 projection matrix
```

The `camera_calibration` package provides an interactive calibrator using a chessboard or
ChArUco target. The calibrated YAML is loaded by `camera_info_manager` and published
synchronised with image frames:

```bash
ros2 run camera_calibration cameracalibrator \
    --size 8x6 --square 0.025 \
    --ros-args -r image:=/camera/image_raw
```

A correctly synchronised `(Image, CameraInfo)` pair on matching timestamps is the contract
that `image_proc::RectifyNode`, OpenCV's `cv::undistortPoints`, and every depth-projection
algorithm requires.

### 4.3 Depth Camera Integration

Linux supports three mainstream depth camera families via dedicated ROS 2 wrappers:

**Intel RealSense D400/L500** — `librealsense2` SDK + `realsense2_camera` ROS 2 node:

```bash
sudo apt install ros-humble-realsense2-camera
ros2 launch realsense2_camera rs_launch.py \
    enable_depth:=true enable_color:=true \
    align_depth.enable:=true
```

Publishes: `/camera/color/image_raw`, `/camera/depth/image_rect_raw`,
`/camera/depth/color/points` (registered colour + depth PointCloud2).

**Stereolabs ZED** — `zed-ros2-wrapper` (proprietary SDK required):
[Source: github.com/stereolabs/zed-ros2-wrapper](https://github.com/stereolabs/zed-ros2-wrapper)

**Luxonis OAK-D** — `depthai_ros` (open-source):
[Source: github.com/luxonis/depthai-ros](https://github.com/luxonis/depthai-ros)

For all three, align the depth frame to the colour frame so that each pixel in the colour
image has a corresponding depth value at the same spatial location. This aligned pointcloud
is the input to the registration and SLAM pipelines covered in Chapter 210.

### 4.4 depth_image_proc: Depth to Point Cloud

The `depth_image_proc` composable nodes convert registered depth images to point clouds using
the camera projection model in the paired `CameraInfo`:
[Source: github.com/ros-perception/image_pipeline/tree/humble/depth_image_proc](https://github.com/ros-perception/image_pipeline/tree/humble/depth_image_proc)

```bash
sudo apt install ros-humble-depth-image-proc
```

Key composable nodes:

| Node plugin | Inputs | Output | Function |
|---|---|---|---|
| `depth_image_proc::PointCloudXyzNode` | `depth/image_rect` + `depth/camera_info` | `depth/points` (XYZ) | Monochrome depth cloud |
| `depth_image_proc::PointCloudXyzrgbNode` | `depth/image_rect` + `rgb/image_rect_color` + `depth/camera_info` | `depth/color/points` (XYZRGB) | Colour-registered cloud |
| `depth_image_proc::RegisterDepthNode` | `depth/image` + both `CameraInfo` | `depth/image_rect` | Align depth to colour frame |

Inside `PointCloudXyzrgbNode`, each pixel `(u, v)` with depth `d` is back-projected:

```
x = (u - cx) * d / fx
y = (v - cy) * d / fy
z = d
```

where `(cx, cy, fx, fy)` come from the `CameraInfo.K` intrinsic matrix. The RGB value is
sampled from the registered colour image at the same pixel. Points with `d == 0` or
`d > range_max` are emitted as `NaN` (the `is_dense: false` flag in PointCloud2).

These nodes run as composable components and benefit from intra-process zero-copy (§1.3)
when loaded in the same container as the downstream SLAM or detection nodes.

### 4.5 Message Synchronisation with message_filters

Perception algorithms that fuse multiple sensor streams — depth-coloured point clouds,
stereo rectification, image-plus-CameraInfo pipelines — require messages from two or more
topics to be temporally aligned before processing. `message_filters` provides the
synchronisation primitives.
[Source: github.com/ros2/message_filters](https://github.com/ros2/message_filters);
[Docs: docs.ros.org/en/ros2_packages/humble/api/message_filters](https://docs.ros.org/en/ros2_packages/humble/api/message_filters/doc/index.html)

**ExactTime** — delivers a callback only when messages with exactly matching `header.stamp`
arrive on all topics (appropriate for drivers that publish Image and CameraInfo with the same
timestamp):

```cpp
#include <message_filters/subscriber.h>
#include <message_filters/time_synchronizer.h>
#include <sensor_msgs/msg/image.hpp>
#include <sensor_msgs/msg/camera_info.hpp>

message_filters::Subscriber<sensor_msgs::msg::Image>      img_sub(node, "/camera/image_raw");
message_filters::Subscriber<sensor_msgs::msg::CameraInfo> info_sub(node, "/camera/camera_info");

message_filters::TimeSynchronizer<
    sensor_msgs::msg::Image,
    sensor_msgs::msg::CameraInfo> sync(img_sub, info_sub, 10);

sync.registerCallback(
    [](const sensor_msgs::msg::Image::ConstSharedPtr& img,
       const sensor_msgs::msg::CameraInfo::ConstSharedPtr& info) {
        // img and info have exactly matching stamps
    });
```

**ApproximateTime** — matches messages whose timestamps are within an adaptive window
(appropriate for sensors with independent clocks or different publish rates):

```cpp
#include <message_filters/sync_policies/approximate_time.h>

using SyncPolicy = message_filters::sync_policies::ApproximateTime<
    sensor_msgs::msg::Image,
    sensor_msgs::msg::PointCloud2>;

message_filters::Subscriber<sensor_msgs::msg::Image>      rgb_sub(node, "/camera/color/image_raw");
message_filters::Subscriber<sensor_msgs::msg::PointCloud2> cloud_sub(node, "/points");

auto sync = std::make_shared<message_filters::Synchronizer<SyncPolicy>>(
    SyncPolicy(10), rgb_sub, cloud_sub);
sync->registerCallback(
    [](const sensor_msgs::msg::Image::ConstSharedPtr& rgb,
       const sensor_msgs::msg::PointCloud2::ConstSharedPtr& cloud) {
        // fuse RGB image with nearest-in-time point cloud
    });
```

The `ApproximateTime` policy supports up to 9 simultaneous topics; set
`setMaxIntervalDuration(rclcpp::Duration::from_seconds(0.05))` to reject pairs that differ
by more than 50 ms. For camera-LiDAR fusion, synchronised pairs are required before
projecting LiDAR points into the image plane (for depth colouring or 3D detection).

---

## 5. vision_msgs: Unified Perception Interface

### 5.1 Message Type Taxonomy

`vision_msgs` provides algorithm-agnostic computer vision message types that form the
contract between perception modules and their consumers (planners, UI, logging).
[Source: github.com/ros-perception/vision_msgs](https://github.com/ros-perception/vision_msgs)

| Message | Description |
|---|---|
| `Classification` | Single classification result (no pose) |
| `ClassificationArray` | Batch classification results |
| `ObjectHypothesis` | `class_id` (string) + `score` (float64) |
| `ObjectHypothesisWithPose` | `ObjectHypothesis` + `PoseWithCovariance` |
| `BoundingBox2D` | 2D bounding box: `center` (Pose2D) + `size_x`, `size_y` |
| `BoundingBox3D` | 3D oriented box: `center` (Pose) + `size` (Vector3) |
| `Detection2D` | `header` + `results` (ObjectHypothesisWithPose[]) + `bbox` (BoundingBox2D) + `id` (string) |
| `Detection2DArray` | `header` + `detections` (Detection2D[]) |
| `Detection3D` | 3D detection with oriented bounding box |
| `Detection3DArray` | Batch 3D detections |
| `LabelInfo` | Semantic segmentation label metadata (maps class_id → name) |
| `VisionInfo` | Classifier metadata: method, database location |

### 5.2 Detection2DArray and Detection3DArray

A minimal producer for 2D detections:

```cpp
#include <vision_msgs/msg/detection2_d_array.hpp>
#include <vision_msgs/msg/detection2_d.hpp>
#include <vision_msgs/msg/object_hypothesis_with_pose.hpp>

auto array_pub = node->create_publisher<vision_msgs::msg::Detection2DArray>(
    "/detections", 10);

auto make_detection(std::string class_id, float score,
                    float cx, float cy, float sx, float sy)
{
    vision_msgs::msg::Detection2D det;
    vision_msgs::msg::ObjectHypothesisWithPose hyp;
    hyp.hypothesis.class_id = class_id;
    hyp.hypothesis.score    = score;
    det.results.push_back(hyp);

    det.bbox.center.position.x = cx;
    det.bbox.center.position.y = cy;
    det.bbox.size_x = sx;
    det.bbox.size_y = sy;
    return det;
}

vision_msgs::msg::Detection2DArray msg;
msg.header.stamp    = node->now();
msg.header.frame_id = "camera_link";
msg.detections.push_back(make_detection("person", 0.92f, 320, 240, 80, 160));
array_pub->publish(msg);
```

3D detections follow the same structure but use `geometry_msgs/Pose` for the box centre and
`geometry_msgs/Vector3` for the half-extents. Downstream consumers (Navigation2 costmap
layers, RViz2 marker plugins, data-recording pipelines) subscribe to `Detection2DArray` or
`Detection3DArray` and need no knowledge of which detector produced them.

### 5.3 nav_msgs: Perception Pipeline Outputs

The downstream consumers of the perception pipeline — Navigation2 (Nav2), RViz2, map
servers, and mission planners — communicate via `nav_msgs`:

| Message | Topic convention | Description |
|---|---|---|
| `nav_msgs/Odometry` | `/odometry/filtered` | Full 6-DOF pose + twist with covariance |
| `nav_msgs/OccupancyGrid` | `/map` | 2D probability grid (−1=unknown, 0=free, 100=occupied) |
| `nav_msgs/Path` | `/plan` | Sequence of `PoseStamped` waypoints |
| `nav_msgs/MapMetaData` | (embedded in OccupancyGrid) | Resolution (m/cell), origin, width, height |

`nav_msgs/Odometry` is the primary output of `ekf_filter_node` and the primary input of
Nav2's controller server. Its key fields:

```
Header header
string child_frame_id          # "base_link"
geometry_msgs/PoseWithCovariance pose    # 6-DOF pose + 6x6 covariance
geometry_msgs/TwistWithCovariance twist  # linear + angular velocity + covariance
```

The `OccupancyGrid` is the output of 2D SLAM systems (`slam_toolbox`, Cartographer) and is
loaded by the Nav2 map server for localisation and global planning. It is also consumed by
`nvblox_ros` which projects its 3D TSDF slice to a 2D obstacle-distance map compatible with
Nav2's costmap layers.

`map_saver_cli` persists the live OccupancyGrid to a PNG+YAML pair for deployment:

```bash
ros2 run nav2_map_server map_saver_cli -f ~/maps/warehouse_map
# Writes: warehouse_map.pgm (pixel = occupancy) + warehouse_map.yaml (metadata)
```

---

## 6. Object Detection on ROS 2

The model training, CUDA kernel implementation, and inference engine internals for
YOLO/RT-DETR/CenterPoint are covered in Chapter 114. This section covers the ROS 2 wrapper
layer that connects those models to the sensor pipeline.

### 6.1 ultralytics_ros: YOLO on ROS 2 Humble

**ultralytics_ros** is the de-facto ROS 2 wrapper for Ultralytics YOLO models (YOLOv8,
YOLOv10, YOLO11) on Humble and Jazzy.
[Source: github.com/Alpaca-zip/ultralytics_ros](https://github.com/Alpaca-zip/ultralytics_ros)

```bash
sudo apt install ros-humble-vision-msgs
git clone https://github.com/Alpaca-zip/ultralytics_ros ~/ros2_ws/src/ultralytics_ros
cd ~/ros2_ws && colcon build --packages-select ultralytics_ros
```

The central node is `tracker_node`:

```bash
ros2 launch ultralytics_ros tracker.launch.xml \
    yolo_model:=yolov8n.pt \
    input_topic:=/camera/image_raw \
    device:=cuda:0
```

Parameters:

| Parameter | Type | Description |
|---|---|---|
| `yolo_model` | string | Model file or Ultralytics model name |
| `input_topic` | string | `sensor_msgs/Image` subscription |
| `device` | string | `cpu`, `cuda:0`, `mps` |
| `conf_thres` | float | Detection confidence threshold (default 0.25) |
| `iou_thres` | float | NMS IoU threshold (default 0.45) |
| `result_topic` | string | Output `Detection2DArray` topic |

Published topics:
- `result_topic` — `vision_msgs/msg/Detection2DArray` (one entry per tracked object)
- `debug_image` — `sensor_msgs/msg/Image` with bounding-box overlays (optional)

For 3D detections from LiDAR, see **OpenPCDet** and **CenterPoint** wrappers described in
Chapter 114 §13; they emit `Detection3DArray` on compatible topics.

### 6.2 Wiring Detection Output to Downstream Nodes

A common architecture is:

```
/camera/image_raw ──► tracker_node ──► /detections (Detection2DArray)
                                            │
                                   ┌────────┴──────────┐
                                   ▼                   ▼
                            Nav2 costmap         rviz2 marker
                           (obstacle layer)       visualiser
```

Topic remapping in a launch file connects generic topic names to robot-specific namespaces
without changing node source:

```python
ComposableNode(
    package='ultralytics_ros',
    plugin='ultralytics_ros::TrackerNode',
    name='yolo_detector',
    remappings=[
        ('input_topic', '/front_camera/image_raw'),
        ('result_topic', '/perception/detections'),
    ],
),
```

---

## 7. Multi-Sensor Fusion: robot_localization

`robot_localization` is the standard ROS 2 package for fusing odometry, IMU, and GPS into a
globally consistent pose estimate via an Extended Kalman Filter (EKF) or Unscented Kalman
Filter (UKF). It publishes the fused pose as `nav_msgs/Odometry` on `odometry/filtered` and
broadcasts the `odom → base_link` TF.
[Source: github.com/cra-ros-pkg/robot_localization](https://github.com/cra-ros-pkg/robot_localization)

### 7.1 ekf_filter_node: EKF and UKF Modes

The filter state vector is 15-dimensional:

```
[x, y, z,  roll, pitch, yaw,  ẋ, ẏ, ż,  ṙoll, ṗitch, ẏaw,  ẍ, ÿ, z̈]
```

Each sensor input specifies which state variables it constrains via a 15-element boolean
array (`[sensor]_config`). This allows partial observability — an IMU typically contributes
angular velocity and linear acceleration; wheel odometry contributes x-velocity and yaw-rate;
GPS contributes absolute x, y, z position.

Key `ekf_filter_node` parameters:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `frequency` | float | 30.0 | Filter update rate (Hz) |
| `sensor_timeout` | float | 0.1 | Seconds before a sensor is considered stale |
| `two_d_mode` | bool | false | Constrain to XY plane (ground vehicles) |
| `publish_tf` | bool | true | Broadcast `odom → base_link` |
| `map_frame` | string | `map` | Global reference frame |
| `odom_frame` | string | `odom` | Odometry frame |
| `base_link_frame` | string | `base_link` | Robot body frame |
| `world_frame` | string | `odom` | Frame for fused output |
| `odom0` | string | — | Topic of first odometry input |
| `imu0` | string | — | Topic of first IMU input |
| `pose0` | string | — | Topic of first pose input |

The filter also accepts:
- `odom0_config`, `imu0_config`, … — 15-element boolean arrays selecting fused variables
- `process_noise_covariance` — 15×15 Q matrix (diagonal is usually sufficient)
- `initial_estimate_covariance` — 15×15 P₀ matrix

For UKF mode use `ukf_filter_node` (identical parameters); UKF is more accurate for
highly non-linear dynamics (fast ground vehicles, aerial platforms) at the cost of ~3× CPU.

### 7.2 GPS/GNSS Integration: navsat_transform_node

Raw GPS delivers latitude/longitude/altitude in WGS-84; the EKF works in a local Cartesian
frame. `navsat_transform_node` converts between the two:

**Subscribed topics:**
- `odometry/filtered` — `nav_msgs/Odometry` (from `ekf_filter_node`)
- `gps/fix` — `sensor_msgs/NavSatFix`
- `imu` — `sensor_msgs/Imu` (for yaw when `use_odometry_yaw` is false)

**Published topics:**
- `odometry/gps` — `nav_msgs/Odometry` in the odometry frame (ENU)
- `gps/filtered` — `sensor_msgs/NavSatFix` (filtered GPS, if `publish_filtered_gps: true`)

Key parameters:

| Parameter | Description |
|---|---|
| `magnetic_declination_radians` | True north correction for local area |
| `yaw_offset` | Additional heading correction (e.g. IMU mounting offset) |
| `use_local_cartesian` | `true` = ENU/local Cartesian; `false` = UTM |
| `wait_for_datum` | Wait for manual datum before accepting GPS |
| `publish_filtered_gps` | Publish the back-converted filtered GPS fix |
| `use_odometry_yaw` | Use heading from odometry rather than IMU |

The output `odometry/gps` is fed back into `ekf_filter_node` as `odom1`, closing the loop
so that GPS positions anchor the growing odometry drift.

### 7.3 Full Fusion Configuration Example

```yaml
# ekf.yaml — fuses wheel odometry + IMU + GPS-derived odometry
ekf_filter_node:
  ros__parameters:
    frequency: 30.0
    sensor_timeout: 0.1
    two_d_mode: true        # planar ground vehicle
    publish_tf: true
    map_frame: map
    odom_frame: odom
    base_link_frame: base_link
    world_frame: odom

    odom0: /wheel/odometry
    odom0_config: [false, false, false,   # x y z
                   false, false, false,   # roll pitch yaw
                   true,  false, false,   # ẋ ẏ ż
                   false, false, true,    # roll_rate pitch_rate yaw_rate
                   false, false, false]   # ẍ ÿ z̈

    imu0: /imu/data
    imu0_config: [false, false, false,
                  true,  true,  true,    # roll pitch yaw from orientation
                  false, false, false,
                  true,  true,  true,    # angular velocities
                  true,  true,  false]   # x and y acceleration (not z — gravity)
    imu0_remove_gravitational_acceleration: true

    odom1: /odometry/gps          # navsat_transform_node output
    odom1_config: [true, true, false,    # fuse GPS x and y, not z (barometer)
                   false, false, false,
                   false, false, false,
                   false, false, false,
                   false, false, false]
```

```bash
ros2 run robot_localization ekf_filter_node \
    --ros-args --params-file ekf.yaml

ros2 run robot_localization navsat_transform_node \
    --ros-args \
    -p magnetic_declination_radians:=0.0349 \
    -p yaw_offset:=1.5708 \
    -p use_local_cartesian:=true \
    -p publish_filtered_gps:=true
```

---

## 8. GPU Acceleration: Isaac ROS and NITROS

### 8.1 Type Adaptation (REP-2007) and Type Negotiation (REP-2009)

ROS 2 Humble introduced two complementary hardware-acceleration features:

**REP-2007 — Type Adaptation**: allows a node to declare that it can accept a topic's data in
a non-serialised, hardware-native format. The `rclcpp` infrastructure calls a user-provided
conversion function to materialise the ROS message from the adapted type on demand, but if
both producer and consumer in the same process declare the same adapted type, zero
serialisation ever occurs and the data pointer is passed directly.

**REP-2009 — Type Negotiation**: extends type adaptation to multi-node pipelines. Each node
advertises the set of adapted types it can produce or consume; the framework negotiates the
best common format across all nodes on a topic at startup, maximising the share of the
pipeline that runs without copy.
[Source: ros.org/reps/rep-2007.html](https://www.ros.org/reps/rep-2007.html);
[Source: ros.org/reps/rep-2009.html](https://www.ros.org/reps/rep-2009.html)

**NITROS (NVIDIA Isaac Transport for ROS)** is NVIDIA's implementation of both REPs. In a
NITROS-accelerated pipeline:

1. The camera driver (or DNN encoder) outputs a `NitrosImage` — a GPU-side tensor backed by
   CUDA managed memory or a GXF memory buffer.
2. Consecutive NITROS-accelerated nodes pass the `NitrosImage` pointer directly — no
   CPU-GPU copy occurs across node boundaries within the container process.
3. The terminal consumer (or a non-NITROS node) triggers the REP-2007 conversion back to a
   standard `sensor_msgs/Image` for interoperability.

The underlying acceleration runtime is NVIDIA's **GXF (Graph Execution Framework)**: a
graph of typed entities connected by message-passing edges, compiled ahead of time and
executed by a scheduler on CPU threads and CUDA streams.
[Source: nvidia-isaac-ros.github.io/concepts/nitros/index.html](https://nvidia-isaac-ros.github.io/concepts/nitros/index.html)

### 8.2 Isaac ROS Package Landscape

| Package | Function |
|---|---|
| `isaac_ros_nitros` | Base `NitrosNode` class; REP-2007/2009 negotiation core |
| `isaac_ros_gxf` | Precompiled GXF extensions used by all Isaac ROS nodes |
| `isaac_ros_managed_nitros` | Simplified NITROS node authoring API |
| `isaac_ros_nitros_type` | Per-type adaptation: `NitrosImage`, `NitrosTensorList`, `NitrosCompressedImage`, etc. |
| `isaac_ros_tensor_rt` | TensorRT inference node (CUDA, Jetson-optimised) |
| `isaac_ros_triton` | Triton Inference Server client node |
| `isaac_ros_dnn_image_encoder` | Preprocessing: resize + normalise image → `NitrosTensorList` |
| `isaac_ros_detectnet` | DetectNet_v2 decoder: `NitrosTensorList` → `Detection2DArray` |
| `isaac_ros_segformer` | Segmentation decoder: tensor → pixel-class mask |
| `isaac_ros_ess` | Stereo depth estimation (ELAS-based neural stereo) |

All packages target Jetson platforms (Orin, AGX Xavier) and x86_64 + NVIDIA GPU. The minimum
requirement is CUDA 11.8, JetPack 5.1+ on Jetson, or Ubuntu 22.04 + ROS 2 Humble on x86_64.

**Jetson unified memory and NITROS:** On Jetson SoCs, the CPU and GPU share physical DRAM
(unified memory architecture). CUDA's `cudaMallocManaged` allocates memory accessible by
both processor types without explicit `cudaMemcpy` transfers. NITROS exploits this: a
`NitrosImage` allocated with `cudaMallocManaged` is read by the CPU camera driver and
processed by CUDA inference kernels with no copy at any pipeline stage. On discrete GPU
systems (PCIe GPUs, x86_64 desktops), the CPU-to-GPU DMA transfer is unavoidable for the
first camera frame ingestion, but subsequent NITROS node-to-node transfers within the GPU
remain copy-free. The GXF scheduler controls CUDA stream synchronisation between nodes to
avoid read-after-write hazards across the pipeline graph.

### 8.3 DNN Encoder → Triton → DetectNet Pipeline

The three-stage Isaac ROS object-detection pipeline is:

```
sensor_msgs/Image
    │
    ▼
isaac_ros_dnn_image_encoder::DnnImageEncoderNode
    (resize to 960×544, normalise, pack → NitrosTensorList)
    │ NitrosTensorList (GPU tensor, no copy)
    ▼
isaac_ros_triton::TritonNode
    (Triton Inference Server: model=peoplenet, backend=tensorrt)
    │ NitrosTensorList (inference output)
    ▼
isaac_ros_detectnet::DetectNetDecoderNode
    (decode bounding-box tensors → Detection2DArray)
    │
    ▼
vision_msgs/Detection2DArray   ← standard ROS 2 message
```

Launch sequence:

```bash
# Start Triton Inference Server (Docker, NVIDIA GPU required)
docker run --rm --gpus all -p 8000:8000 -p 8001:8001 -p 8002:8002 \
    -v /path/to/model_repository:/models \
    nvcr.io/nvidia/tritonserver:23.12-py3 \
    tritonserver --model-repository=/models

# Launch the ROS 2 perception pipeline
ros2 launch isaac_ros_detectnet isaac_ros_detectnet.launch.py \
    model_name:=peoplenet \
    model_repository_paths:=[/path/to/model_repository] \
    input_binding_names:=[input_1] \
    output_binding_names:=[output_cov/Sigmoid,output_bbox/BiasAdd] \
    network_image_width:=960 network_image_height:=544 \
    image_mean:=[0.5,0.5,0.5] image_stddev:=[0.5,0.5,0.5]
```

The PeopleNet model is available pre-trained from the
[NVIDIA NGC model catalogue](https://catalog.ngc.nvidia.com/models).

### 8.4 Nvblox: GPU-Accelerated 3D Reconstruction

**Nvblox** is an Isaac ROS package for real-time GPU-resident voxel reconstruction. It
consumes depth images (from a RealSense, ZED, or Isaac ROS stereo depth estimator) and builds
a Truncated Signed Distance Function (TSDF) voxel map on the GPU, outputting:
- `nvblox_msgs/Mesh` — triangulated mesh of the reconstructed surface
- `nvblox_msgs/DistanceMapSlice` — 2D obstacle distance map for Navigation2

```bash
sudo apt install ros-humble-nvblox
ros2 launch nvblox_ros nvblox_realsense.launch.py
```

Unlike octomap (CPU octree), Nvblox keeps the entire map resident in CUDA device memory and
performs ray-casting and TSDF integration as CUDA kernels, achieving reconstruction at 30 Hz
on an NVIDIA GPU.
[Source: github.com/NVIDIA-ISAAC-ROS/isaac_ros_nvblox](https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_nvblox)

---

## 9. Diagnostics and Performance Profiling

**`diagnostic_msgs`**: Every sensor driver should publish a `diagnostic_msgs/DiagnosticArray`
on `/diagnostics` reporting health (OK / WARN / ERROR) and key metrics (frame rate, drop
count, temperature). The `diagnostic_aggregator` node collects per-device diagnostics and
exposes them in RViz2's diagnostics panel.

**`ros2 doctor`**: The built-in `ros2 doctor` command checks DDS configuration, QoS
mismatches, and package version consistency:

```bash
ros2 doctor --report
```

**`tracetools`** and **`ros2_tracing`**: LTTng-based tracing framework for ROS 2. Provides
per-callback latency, executor scheduling, and message-passing traces with microsecond
resolution:

```bash
sudo apt install ros-humble-tracetools-launch lttng-tools
ros2 launch tracetools_launch trace.launch.py
```

Captured traces are analysed with `tracetools_analysis` Python library or with the
`ros2_tracing` Jupyter notebooks.

**`diagnostic_updater`**: The C++ API for sensor drivers to publish structured health
diagnostics:

```cpp
#include <diagnostic_updater/diagnostic_updater.hpp>

diagnostic_updater::Updater updater(node);
updater.setHardwareID("velodyne_vlp16_0");
updater.add("LiDAR frame rate", [this](diagnostic_updater::DiagnosticStatusWrapper& stat) {
    if (current_hz_ < 8.0)
        stat.summary(diagnostic_msgs::msg::DiagnosticStatus::WARN, "Low frame rate");
    else
        stat.summary(diagnostic_msgs::msg::DiagnosticStatus::OK, "Nominal");
    stat.add("Hz", current_hz_);
    stat.add("Dropped frames", drop_count_);
});
// call updater.force_update() in a timer callback at ~1 Hz
```

The `diagnostic_aggregator` collects these per-device entries and publishes a structured
tree, displayed in the RViz2 Robot Monitor plugin. Production deployments typically configure
a `diagnostic_updater` for every sensor driver so that operator dashboards surface hardware
faults before they corrupt SLAM map quality.

**`ros2 topic hz` and `ros2 topic bw`**: Quick field diagnostics for rate and bandwidth:

```bash
ros2 topic hz /points          # prints mean/stddev/min/max of inter-message intervals
ros2 topic bw /camera/image_raw   # prints throughput in MB/s
```

Thread affinity for real-time callbacks: use `pthread_setaffinity_np` to pin the
`rclcpp::executors::MultiThreadedExecutor` worker threads to isolated CPU cores, paired with
`SCHED_FIFO` priority (requires `CAP_SYS_NICE`):

```cpp
// Pin current thread to CPU 3
cpu_set_t cpuset;
CPU_ZERO(&cpuset); CPU_SET(3, &cpuset);
pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
```

---

## 10. System Integration: End-to-End Perception Stack

The following assembles a complete outdoor robot perception stack on Ubuntu 22.04 / ROS 2
Humble using a RealSense D435i (RGB-D + IMU), a Velodyne VLP-16 LiDAR, and a u-blox F9P
GPS receiver.

### 10.1 Package Installation

```bash
sudo apt install ros-humble-desktop \
    ros-humble-realsense2-camera \
    ros-humble-velodyne \
    ros-humble-robot-localization \
    ros-humble-image-transport-plugins \
    ros-humble-vision-msgs \
    ros-humble-slam-toolbox \
    ros-humble-nav2-bringup
```

### 10.2 Topic Graph

```
/camera/color/image_raw   ──► tracker_node ──► /perception/detections
/camera/depth/image_rect_raw ─┐
                               ├──► depth_image_proc::PointCloudXyzrgbNode
/camera/color/camera_info ────┘         │
                                         ▼ /camera/depth/color/points
/velodyne_points ──────────────────────► fast_lio ──► /Odometry, /cloud_registered

/camera/imu ──► ekf_filter_node ──► /odometry/filtered ──► odom→base_link TF
/gps/fix ──► navsat_transform_node ──► /odometry/gps ──► ekf_filter_node (odom1)
```

### 10.3 Launch File

```python
# outdoor_perception.launch.py
from launch import LaunchDescription
from launch_ros.actions import Node, ComposableNodeContainer
from launch_ros.descriptions import ComposableNode

def generate_launch_description():
    return LaunchDescription([
        # Intel RealSense D435i
        Node(package='realsense2_camera', executable='realsense2_camera_node',
             parameters=[{'enable_depth': True, 'enable_color': True,
                          'align_depth.enable': True,
                          'enable_gyro': True, 'enable_accel': True,
                          'unite_imu_method': 2}]),

        # Velodyne VLP-16
        Node(package='velodyne_driver', executable='velodyne_driver_node',
             parameters=[{'model': 'VLP16', 'rpm': 600.0, 'port': 2368}]),

        # YOLO detection (composable for zero-copy with image_proc)
        ComposableNodeContainer(
            name='perception_container',
            namespace='',
            package='rclcpp_components',
            executable='component_container',
            composable_node_descriptions=[
                ComposableNode(
                    package='image_proc', plugin='image_proc::RectifyNode',
                    name='rectify_color',
                    remappings=[('image', '/camera/color/image_raw')]),
            ]),

        # Multi-sensor EKF fusion
        Node(package='robot_localization', executable='ekf_filter_node',
             parameters=['ekf.yaml']),
        Node(package='robot_localization', executable='navsat_transform_node',
             parameters=[{'magnetic_declination_radians': 0.0,
                          'use_local_cartesian': True,
                          'publish_filtered_gps': True}]),
    ])
```

### 10.4 Data Recording with rosbag2 and MCAP

`rosbag2` is the ROS 2 data-recording system. It records live topic streams to disk for
offline replay, dataset creation, and CI regression testing. The default storage backend
on Humble is SQLite3; the **MCAP** format is preferred for production and large datasets.

```bash
sudo apt install ros-humble-rosbag2-storage-mcap

# Record all sensor topics to an MCAP bag (highest write throughput)
ros2 bag record -s mcap \
    --storage-preset-profile fastwrite \
    -o outdoor_run_001 \
    /points /camera/color/image_raw /camera/depth/image_rect_raw \
    /imu/data /gps/fix /odometry/filtered /tf /tf_static
```

MCAP advantages over SQLite3:
- Sequential write design: no transaction overhead during recording; better for high-rate
  LiDAR + camera combinations that exceed 100 MB/s.
- Built-in message index in the file footer: random access by topic and timestamp without
  a separate index database.
- Language-agnostic readers (Python, C++, Go, Rust) via the `mcap` library; used by
  Foxglove Studio for visualisation.
  [Source: mcap.dev](https://mcap.dev)

Playback at recorded rate or accelerated:

```bash
ros2 bag play outdoor_run_001 --rate 2.0   # 2× speed
ros2 bag play outdoor_run_001 \
    --topics /points /imu/data             # subset of topics
ros2 bag info outdoor_run_001              # print topic list, duration, message counts
```

Bags recorded with MCAP are the primary input to offline perception development workflows:
replay the bag at reduced rate, run experimental detection nodes against the recorded sensor
data, and compare output against ground-truth annotations without needing physical hardware.
The `--topics` filter on playback lets a development node receive only the sensors it cares
about, keeping CPU/GPU utilisation bounded during algorithm iteration.

For **Foxglove Studio** integration (web or desktop), install the bridge and connect while
the bag plays back:

```bash
sudo apt install ros-humble-foxglove-bridge
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
# Open app.foxglove.dev and connect to ws://localhost:8765
```

---

## 11. Integrations

- **Chapter 96 (libcamera and the Linux Camera Stack)**: Camera hardware pipeline, V4L2
  Media Controller, ISP pipeline, RAW capture, and `ros2_v4l2_camera`. The
  `image_transport` pipeline in §4 begins where ch96's libcamera stack ends.
- **Chapter 114 (OpenCV and GPU-Accelerated Computer Vision)**: YOLO model training,
  CUDA-accelerated inference, monocular depth, point-cloud segmentation, and the DROID-SLAM/
  SplaTAM/MonoGS neural SLAM implementations. The `ultralytics_ros` wrapper in §6.1 connects
  ch114 inference engines to the `vision_msgs` bus described here.
- **Chapter 209 (OpenSLAM)**: G2O, GMapping, RTAB-Map, and Basalt VIO. The TF2 transform
  tree in §3 and the EKF sensor fusion in §7 are prerequisites to running any of those SLAM
  systems; ch209 consumes the `/odometry/filtered` and TF broadcasts described here.
- **Chapter 210 (SLAM Theory and State of the Art)**: FAST-LIO2, LIO-SAM, Cartographer,
  and LiDAR hardware drivers. The `sensor_msgs/PointCloud2` internals in §2.2 and the QoS
  profiles in §1.2 apply directly to those LiDAR pipelines.
- **Chapter 133 (Vulkan Compute Queues and Task Graphs)**: Vulkan compute as an alternative
  to CUDA for GPU-resident perception; `kompute` library for cross-vendor GPU ML inference
  on non-NVIDIA hardware.
- **Chapter 141 (Vulkan Cooperative Matrices and GPU ML Acceleration)**: GPU matrix
  operations underpinning TensorRT and Triton inference (§8.3); relevant when targeting
  AMD/Intel GPUs where NITROS's CUDA dependency is not available.
- **Chapter 48 (ROCm and Machine Learning on Linux GPUs)**: ROCm/HIP as the AMD alternative
  to the CUDA stack required by Isaac ROS/NITROS; PyTorch + ROCm for training YOLO models
  deployed via the `ultralytics_ros` wrapper.
