# M1 FAST-LIO — Jetson Orin NX (Local Deployment)

GPU-accelerated FAST-LIO2 for the Genisom M1 quadruped robot with RoboSense Airy96 LiDAR.
Deployed directly on **Jetson Orin NX** (no cross-machine setup needed).

## Architecture

```
rslidar_sdk (M1 body)
  ├─ /front_lidar     (PointCloud2, raw, ~10 Hz)
  └─ /front_lidar/imu (Imu, Y-up gravity, ~100 Hz)
         │
         ▼  (Zenoh DDS, domain 66)
fastlio_mapping (NX local, GPU)
  │  IMU rotation R_x(90°) embedded in C++ (imu_cbk)
  │  Static TF embedded in C++
  ├─ /Odometry            (odom → imu_link)
  ├─ /cloud_registered    (world frame)
  └─ /cloud_registered_body (body frame)
         │
    ┌────┴────┐
    ▼         ▼
nvblox      Nav2
(TSDF map)  (robot_base_frame: imu_link)
```

## Key Differences from Original FAST-LIO

| Aspect | Original (xki2ng/m1-fastlio) | This Repo |
|--------|------------------------------|-----------|
| IMU rotation | External `imu_rot.py` | Embedded in `imu_cbk()` (C++) |
| Static TF | `static_transform_publisher` | Embedded in C++ |
| IMU topic | `/livox/imu` (rotated) | `/front_lidar/imu` (raw, rotated in C++) |
| Airy96 point struct | `PCL_ADD_POINT4D` (bug) | `float x,y,z` (fixed) |
| Ring buffer | 128 | 256 (Airy96 96 lines) |
| cube_side_length | 200 (bug) | 1000 (fixed) |

## Quick Start (On NX)

```bash
# Prerequisites: rslidar_sdk running, publishing /front_lidar and /front_lidar/imu

# Start full stack
bash /home/robot/fastlio/m1_full_stack.sh start

# Stop
bash /home/robot/fastlio/m1_full_stack.sh stop

# Status
bash /home/robot/fastlio/m1_full_stack.sh status
```

## Verified Configuration

See `scripts/m1_airy96_rot.yaml` — this is the **only** FAST-LIO config.

Critical verified parameters (2026-07-03):
- `lid_topic: "/front_lidar"`
- `imu_topic: "/front_lidar/imu"` (raw, rotated internally)
- `body_frame: "imu_link"`
- `extrinsic_R: R_y(90°)`
- `cube_side_length: 1000.0`
- `time_sync_en: false`
- Gravity verified: Z ≈ -9.81 ✅

## Files

```
├── fastlio_src/          # FAST-LIO source (ROS2 Humble, GPU)
│   ├── src/              # laserMapping.cpp (w/ embedded IMU rot), preprocess, etc.
│   ├── include/          # IKFoM, ikd-Tree, GPU utils
│   ├── config/           # LiDAR type reference configs
│   ├── msg/              # Pose6D custom message
│   ├── CMakeLists.txt    # CUDA optional build
│   └── package.xml
├── scripts/              # Configs and scripts
│   ├── m1_airy96_rot.yaml       # ⭐ Working FAST-LIO config
│   ├── m1_full_stack.sh          # One-click launcher
│   ├── m1_nav2.launch.py         # Nav2 launch
│   ├── m1_nav2_fastlio.yaml      # Nav2 params
│   ├── m1-fastlio.service        # systemd unit
│   ├── zenoh_nx_client.json5     # Zenoh client config
│   ├── nav2_params.yaml          # Nav2 detailed params
│   ├── costmap_zenoh_relay.py    # Costmap QOS relay
│   └── m1_vel_sign.py            # Velocity sign correction
└── README.md
```

## Build from Source

```bash
cd ~/m1_ws
source /opt/ros/humble/setup.bash

# GPU build (recommended)
colcon build --packages-select fast_lio \
  --cmake-args -DCMAKE_BUILD_TYPE=Release \
  -DFASTLIO_USE_CUDA=ON \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.6/bin/nvcc \
  -DCUDAToolkit_ROOT=/usr/local/cuda-12.6 \
  -DCMAKE_CUDA_ARCHITECTURES=87
```

## Credits

- FAST-LIO2: Xu W, Zhang F. TPAMI 2022.
- GPU port: Omer Mersin
- M1 integration & Airy96 fixes: This repo
