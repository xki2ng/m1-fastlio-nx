# M1 FAST-LIO — Jetson Orin NX (Final Verified)

GPU-accelerated FAST-LIO2 for the Genisom M1 quadruped robot with RoboSense Airy96 LiDAR.
Deployed directly on **Jetson Orin NX** (no cross-machine setup needed).

**Status**: ✅ Verified 2026-07-03 — Gravity Z=-9.81, Odometry 10Hz, no drift.

## Architecture

```
rslidar_sdk (M1 body)
  ├─ /front_lidar       (PointCloud2, raw 86K pts, ~10 Hz)
  └─ /front_lidar/imu   (Imu, Y-up gravity, ~100 Hz)
         │
         ▼  (Zenoh DDS, domain 66)
m1_voxel_filter (NX, CUDA, leaf=0.05m)
  └─ /front_lidar/filtered  (16K pts, ~10 Hz)
         │
         ▼
fastlio_mapping (NX, GPU)
  │  IMU rotation R_x(90°) embedded in C++ (imu_cbk)
  │  Static TF embedded in C++
  ├─ /Odometry               (odom → imu_link, ~10 Hz)
  ├─ /cloud_registered       (world frame, 0.8 MB/s)
  └─ /cloud_registered_body  (body frame, 0.8 MB/s)
```

## Key Differences from Original FAST-LIO

| Aspect | Original | This Repo |
|--------|----------|-----------|
| IMU rotation | External `imu_rot.py` | Embedded in `imu_cbk()` (C++) |
| Static TF | `static_transform_publisher` | Embedded in C++ |
| IMU topic | `/livox/imu` (rotated) | `/front_lidar/imu` (raw, rotated in C++) |
| LiDAR topic | `/front_lidar` (raw 86K) | `/front_lidar/filtered` (voxel 0.05m → 16K) |
| Airy96 point struct | `PCL_ADD_POINT4D` (bug) | `float x,y,z` (fixed) |
| Ring buffer | 128 | 256 (Airy96 96 lines) |
| cube_side_length | 200 (bug → FOV deletion) | 1000 (fixed) |
| Output bandwidth | 44 MB/s | 0.8 MB/s (50x reduction) |

## Quick Start

```bash
# 1. Start voxel pre-filter (required!)
source /opt/ros/humble/setup.bash
/home/robot/m1_ws/build/m1_voxel_filter/m1_voxel_flt \
  --ros-args -p input_topic:=/front_lidar \
  -p output_topic:=/front_lidar/filtered \
  -p voxel_leaf_size:=0.05 \
  -r __name:=voxel_front &

# 2. Start FAST-LIO
/home/robot/m1_ws/install/fast_lio/lib/fast_lio/fastlio_mapping \
  --ros-args --params-file /home/robot/fastlio/m1_airy96_rot.yaml &

# Or use the all-in-one launcher:
bash /home/robot/fastlio/m1_full_stack.sh start
```

## Verified Configuration

See `scripts/m1_airy96_rot.yaml` — **this is the only config, do not modify.**

```yaml
lid_topic:               "/front_lidar/filtered"   # voxel pre-filtered
imu_topic:               "/front_lidar/imu"         # raw, rotated in C++
body_frame:              "imu_link"
extrinsic_R:             R_y(90°)                   # [0,0,1, 0,1,0, -1,0,0]
extrinsic_est_en:        false
cube_side_length:        1000.0
point_filter_num:        1
filter_size_surf:        0.3
filter_size_map:         0.3
acc_cov:                 0.2
gyr_cov:                 0.5
b_acc_cov:               0.0001
b_gyr_cov:               0.01
time_sync_en:            false
lidar_type:              5                          # RoboSense Airy96
scan_line:               96
```

**Verified**: Gravity Z ≈ -9.81, Odometry ~10 Hz, no drift.

## Build from Source

```bash
cd ~/m1_ws
source /opt/ros/humble/setup.bash

# GPU build
colcon build --packages-select fast_lio \
  --cmake-args -DCMAKE_BUILD_TYPE=Release \
  -DFASTLIO_USE_CUDA=ON \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.6/bin/nvcc \
  -DCUDAToolkit_ROOT=/usr/local/cuda-12.6 \
  -DCMAKE_CUDA_ARCHITECTURES=87
```

## Important Notes

- **Do not run colcon build while FAST-LIO is running** — compiler CPU contention causes
  LiDAR-IMU timestamp desync → registration failure → drift → crash.
- Voxel pre-filter (`m1_voxel_filter`) must be started before FAST-LIO.
- This repo supersedes `xki2ng/m1-fastlio` (archived).

## Credits

- FAST-LIO2: Xu W, Zhang F. TPAMI 2022.
- GPU port: Omer Mersin
- M1 integration & Airy96 fixes: xki2ng
