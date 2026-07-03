# M1 FAST-LIO + WebUI — Jetson Orin NX (Final)

GPU-accelerated FAST-LIO2 + WebUI control panel for the Genisom M1 quadruped robot.
Deployed on **Jetson Orin NX** with cloud access via SSH tunnel.

**Status**: ✅ Final verified 2026-07-03

## Architecture

```
┌─ M1 Body ──────────────────────────────────┐
│  rslidar_sdk: /front_lidar  /front_lidar/imu │
│  robot_forward, perception, GPS, etc.       │
└──────────────┬──────────────────────────────┘
               │ Zenoh DDS (domain 66)
┌─ Jetson Orin NX (192.168.168.100) ─────────┐
│  m1_voxel_filter   → /front_lidar/filtered  │
│  fastlio_mapping   → /Odometry, /Laser_map  │
│  m1_nvblox_br      → TSDF mapping           │
│  control_server.py → :8000 (API + static)   │
└──────────────┬──────────────────────────────┘
               │ SSH Tunnel (8000+9090)
┌─ Cloud Server (39.105.71.173) ─────────────┐
│  cloud_proxy.py   → :8443 (proxy + static)  │
│  WebUI access: http://39.105.71.173:8443    │
└─────────────────────────────────────────────┘
```

## WebUI Features

- **Dashboard**: Process status (rosbridge, controller, Zenoh, voxel, FAST-LIO, nvblox, nav2)
- **Controls**: Start/stop voxel, FAST-LIO, nvblox independently
- **Map Save**: Background ros2 bag record of /Laser_map, custom naming
- **Map Management**: List, delete saved maps (tab)
- **Trajectory Management**: GPS path record, save, load
- **Cloud Access**: SSH tunnel proxy, single port 8443

## Files

```
scripts/
├── index.html              # WebUI frontend
├── control_server.py       # NX API + static server (:8000)
├── cloud_proxy.py          # Cloud proxy (:8443, socket-based)
├── save_map.py             # Map save script (ros2 bag record)
├── tunnel_guard.sh         # SSH tunnel auto-reconnect
├── m1_airy96_rot.yaml      # FAST-LIO verified config
├── m1-webui.service        # systemd unit for auto-start
├── m1_full_stack.sh        # All-in-one launcher
├── m1_nav2.launch.py       # Nav2 launch
├── m1_nav2_fastlio.yaml    # Nav2 params
├── m1-fastlio.service      # FAST-LIO systemd unit
├── nav2_params.yaml        # Nav2 detailed params
├── zenoh_nx_client.json5   # Zenoh client config
├── costmap_zenoh_relay.py  # Costmap QoS relay
├── m1_vel_sign.py          # Velocity sign correction
├── activate.sh / start_nav2.sh
└── m1_bt_nav.xml
```

## FAST-LIO Verified Config

See `scripts/m1_airy96_rot.yaml`:

```yaml
lid_topic: /front_lidar/filtered    # voxel 0.05m pre-filter
imu_topic: /front_lidar/imu         # raw, rotated in C++ (R_x 90°)
body_frame: imu_link
extrinsic_R: R_y(90°)
cube_side_length: 1000.0
point_filter_num: 1
filter_size_surf: 0.3
filter_size_map: 0.3
acc_cov: 0.2 / gyr_cov: 0.5
pcd_save_en: true
```

## Quick Deploy

```bash
# On NX
cd ~/webui && python3 control_server.py &   # :8000

# Cloud proxy
scp cloud_proxy.py admin@39.105.71.173:~/webui/
ssh admin@39.105.71.173 "python3 ~/webui/cloud_proxy.py &"  # :8443

# SSH tunnel
bash tunnel_guard.sh &
```

## Build FAST-LIO

```bash
cd ~/m1_ws && source /opt/ros/humble/setup.bash
colcon build --packages-select fast_lio \
  --cmake-args -DCMAKE_BUILD_TYPE=Release \
  -DFASTLIO_USE_CUDA=ON \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.6/bin/nvcc \
  -DCMAKE_CUDA_ARCHITECTURES=87
```

## Credits

- FAST-LIO2: Xu W, Zhang F. TPAMI 2022
- GPU port: Omer Mersin
- M1 integration: xki2ng
