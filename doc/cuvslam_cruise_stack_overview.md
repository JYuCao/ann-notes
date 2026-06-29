# cuVSLAM Cruise Stack 架构概述

## 1. 三套脚本与职责划分

| 脚本 | 运行位置 | 用途 |
|---|---|---|
| `system_cuvslam_cruise_headless.sh` | Jetson (go2) | 纯巡航：cuVSLAM + local planner + terrain analysis |
| `system_cuvslam_cruise_route_planner_headless.sh` | Jetson (go2) | 巡航 + 全局路径规划：增加 far_planner / terrain_analysis_ext / graph_decoder |
| `system_cuvslam_cruise_local_viewer.sh` | 本地 PC | DDS 远程 Viz 桥接 + RViz2 显示 |

### 1.1 远程代理机制

前两个 headless 脚本均有 `delegate_to_remote()` 机制：

- 在本地 PC 上执行时（非 unitree@jetson），自动通过 `sshpass ssh` 将带环境变量的命令转发到 `REMOTE_HOST`（默认 `unitree@192.168.31.81`）上执行
- 通过 `build_remote_command()` 序列化所有环境变量（`ENABLE_RVIZ`, `CUVSLAM_MAP_DIR`, `VIEWER_IP` 等）到远程
- 本地 PC 需要安装 `sshpass`，密码硬编码为 `123`

---

## 2. system_cuvslam_cruise_local_viewer.sh（本地 PC 端）

### 2.1 启动流程

1. **确定 DDS 网卡**：通过 `ip route get ROBOT_IP` 找到通往 Jetson 的网口
2. **启动控制桥接**：`cruise_viz_bridge --mode control_source`，监听本地 RViz 的 `/goal_point`/`/way_point`/`/joy`，通过 TCP socket 转发到 Jetson
3. **等待主题**：通过 ROS 订阅检测 `/state_estimation` 和 `/terrain_map_viz` 是否可达（超时 20s）
4. **启动 RViz2**：加载 `vehicle_simulator_cruise_viewer.rviz`

### 2.2 DDS 网络配置

```yaml
ROS_LOCALHOST_ONLY=0
ROS_DOMAIN_ID=77
RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
CYCLONEDDS_URI:
  Interfaces: <检测到的本地网卡>
  Peers: [127.0.0.1, ROBOT_IP]
```

DOMAIN_ID=77 用于隔离 viz 流量，与大车原有的 DDS 网络错开。

### 2.3 控制通道

- `system_cuvslam_cruise_local_viewer.sh` 启动 `cruise_viz_bridge --mode control_source`
- 订阅本地 RViz 的 topic：`/goal_point`, `/way_point`, `/joy`, `/reset_visibility_graph`, `/update_visibility_graph`
- 通过 TCP socket（port 37667）序列化 pickle 后发送到 Jetson
- Jetson 侧由 headless 脚本启动 `cruise_viz_bridge --mode control_sink` 接收，重新发布到 ROS Topic

控制流:
```
RViz2 (本地PC) → /goal_point → control_source → TCP:37667 → control_sink → /goal_point (Jetson) → far_planner / local_planner
```

### 2.4 可视化数据通道

- Headless 脚本启动 `cruise_viz_bridge --mode capture`（Jetson 侧），订阅关键 topic 并通过 TCP 发送
- Headless 脚本启动 `cruise_viz_bridge --mode publish`（Jetson 侧），在 DOMAIN_ID=77 上重新发布到本地 PC
- 传输的数据分为两个 TCP 端口：
  - **Fast port 37665**：小数据量高频 topic（`/state_estimation`, `/tf`, `/path`, `/navigation_boundary`）
  - **Cloud port 37666**：大数据量点云（`/registered_scan_viz`, `/terrain_map_viz`, `/global_map_points`, `/cuvslam/landmarks`）
- `capture` 节点会对点云做降采样（registered_scan: leaf=0.1 / max=5000, terrain_map: leaf=0.1 / max=15000）

---

## 3. system_cuvslam_cruise_headless.sh（Jetson 巡航栈）

### 3.1 启动阶段

```
[0/5] DDS setup → source unitree_setup.sh, 检测 VIEWER_IP/VIEWER_INTERFACE
[1/5] 启动 cuVSLAM wrapper → start_realsense_slam_map.sh (stereo/robot/save-only/use-imu)
[2/5] 预热 → sleep WARMUP_SEC (默认 8s)
[3/5] 同步 runtime overlay → 复制 src/ 到 install/ 下的 Python 脚本
[4/5] 启动 ROS 链 → ros2 launch vehicle_simulator system_cuvslam_cruise.launch
[5/5] 运行中
```

### 3.2 启动前清理

`pkill -f` 杀掉所有可能残留的进程：
```
ros2 launch vehicle_simulator system_cuvslam_cruise.launch
transform_everything  cuvslam_ros  localPlanner  pathFollower
terrainAnalysis  terrain_map_viz_relay  cruise_debug_markers
cruise_viz_bridge  rviz2  static_transform
```

### 3.3 Runtime overlay

从 `src/` 同步 Python 脚本到 `install/`（colcon install 后的 site-packages），实现热更新：
- `cuvslam_ros_bridge.py`
- `transform_everything.py`
- `cruise_viz_bridge.py`
- `terrain_map_viz_relay.py`
- `cruise_debug_markers.py`
- `system_cuvslam_cruise.launch`
- `vehicle_simulator_cruise_viewer.rviz`

### 3.4 ROS 节点链（cruise.launch）

```
mapping_cuvslam.launch
  ├── transform_everything       # L1 LiDAR/IMU → body frame
  └── cuvslam_ros_bridge          # cuVSLAM UDP → /state_estimation, /registered_scan, /cuvslam/landmarks

local_planner.launch
  ├── localPlanner                # 局部路径规划（基于 /terrain_map + /registered_scan）
  └── pathFollower                # 路径跟踪

terrain_analysis.launch
  └── terrainAnalysis             # /utlidar/transformed_cloud → /terrain_map

terrain_map_viz_relay (条件)     # /terrain_map → /terrain_map_viz（降采样）
cruise_debug_markers (条件)      # 调试 Marker

static_transform_publisher
  ├── map → world (identity)
  ├── aft_mapped → sensor (identity)
  └── aft_mapped → camera_link (identity)
```

### 3.5 crusie_viz_bridge 三种模式（ENABLE_VIEWER_DDS=1 时启动）

当 `VIEWER_IP` 不为空且 `ENABLE_RVIZ != 1` 时启动：

| 模式 | 网络 | 作用 |
|---|---|---|
| `publish` | DOMAIN_ID=77, 对端 VIEWER_IP | 接收 TCP 数据 → 在 viz domain 重发 ROS topic |
| `control_sink` | DATA_INTERFACE | 监听 TCP:37667 → 接收本地 PC 控制指令 → 发布 ROS topic |
| `capture` | DATA_INTERFACE | 订阅 ROS topic → 降采样 → TCP 发给 publish |

### 3.6 RViz 模式（ENABLE_RVIZ=1）

- 通过 SSH X11 Forwarding 转发到本地 PC
- 检测到 `DISPLAY=localhost:*` 时启用 `LIBGL_ALWAYS_SOFTWARE=1`
- `ENABLE_RVIZ=1` 会强制 `ENABLE_VIEWER_DDS=0`（避免干扰 L1 扫描频率）

---

## 4. system_cuvslam_cruise_route_planner_headless.sh（Jetson 巡航 + 全局规划）

与 cruise_headless 基本相同，差异如下：

### 4.1 新增清理进程

```
terrainAnalysisExt  far_planner  graph_decoder
```

### 4.2 新增节点（with_route_planner.launch）

- **terrain_analysis_ext**：`terrainAnalysisExt`，长距离地形建图（`/terrain_map_ext`，clearingDis=30m）
- **far_planner**：全局路径规划，基于探索图（`/overall_map` + `/terrain_map_ext`）
- **graph_decoder**：探索图的可视化解码

### 4.3 新增：global_map_injector

```python
global_map_injector.py  # 独立进程
```

- 从 `CUVSLAM_MAP_DIR` 或 `last_output_dir` 读取 `global_map.ply`
- 发布到 `/terrain_map_ext`（TRANSIENT_LOCAL QoS）
- 2s 定时器重试，发布一次后停止
- frame_id = `map`（与 far_planner 的 `world_frame: map` 对齐）

### 4.4 far_planner 的 topic 重映射

```yaml
/odom_world → /state_estimation
/terrain_cloud → /terrain_map_ext        # 全局点云来自 injector + terrain_analysis_ext
/scan_cloud → /terrain_map               # 局部地形
/terrain_local_cloud → /registered_scan  # 注册后的融合扫描
```

### 4.5 输出提示

脚本执行后会打印：
```
far_planner subscribes: /state_estimation, /terrain_map, /terrain_map_ext, /registered_scan
Set a goal via RViz GoalpointTool or /goalpose topic
```

---

## 5. 跨脚本关系图（local_viewer ↔ headless）

```
┌─────────────────────────────────────────────────────────────────────┐
│                        本地 PC (Windows / Linux)                     │
│                                                                     │
│  system_cuvslam_cruise_local_viewer.sh                              │
│    ├── DDS: DOMAIN_ID=77, CYCLONEDDS peers=[127.0.0.1, 192.168.31.81]
│    ├── cruise_viz_bridge --mode control_source                     │
│    │     ← /goal_point, /way_point, /joy                           │
│    │     → TCP:37667 → (to Jetson)                                 │
│    └── rviz2 -d vehicle_simulator_cruise_viewer.rviz               │
│                                                                     │
└────────────────────┬────────────────────────────────────────────────┘
                     │ TCP sockets: 37665(fast), 37666(cloud), 37667(control)
                     │ DDS Domain 77 (CycloneDDS)
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Jetson Orin (192.168.31.81 / go2)                 │
│                                                                     │
│  system_cuvslam_cruise_headless.sh (or _route_planner_)             │
│    ├── DATA_INTERFACE=enP8p1s0 (默认)                               │
│    ├── cuVSLAM wrapper → container → UDP:12345(pose), 12346(landmarks)
│    ├── cruise_viz_bridge --mode capture                             │
│    │     ← ROS topics → downsample → TCP:37666/37665                │
│    ├── cruise_viz_bridge --mode publish (DOMAIN_ID=77)              │
│    │     ← TCP → re-publish on Domain 77                            │
│    ├── cruise_viz_bridge --mode control_sink                        │
│    │     ← TCP:37667 → re-publish ROS topics                        │
│    │                                                                 │
│    └── ROS 2 (Humble, rmw_cyclonedds, DATA_INTERFACE)               │
│          ├── transform_everything → /utlidar/transformed_cloud, _imu
│          ├── cuvslam_ros_bridge → /state_estimation, /registered_scan
│          ├── terrainAnalysis → /terrain_map                         │
│          ├── terrainAnalysisExt* → /terrain_map_ext                 │
│          ├── localPlanner + pathFollower                            │
│          ├── far_planner* + graph_decoder*                          │
│          ├── terrain_map_viz_relay → /terrain_map_viz               │
│          ├── global_map_injector* → /terrain_map_ext (inject PLY)   │
│          └── static transforms: map→world, aft_mapped→sensor→camera │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
注: * 标记为 route_planner 版本独有
```

---

## 6. 完整 Topic 流

```
L1 LiDAR
  → /utlidar/cloud (raw)
  → transform_everything
    → /utlidar/transformed_cloud (body frame)
    → terrainAnalysis → /terrain_map
      → terrain_map_viz_relay → /terrain_map_viz
      → cuvslam_ros_bridge (terrain_map_cb) → /registered_map (累积)
    → terrainAnalysisExt → /terrain_map_ext

L1 IMU
  → /utlidar/imu (raw)
  → transform_everything
    → /utlidar/transformed_imu
    → /utlidar/transformed_raw_imu

cuVSLAM (Docker container)
  → UDP pose (127.0.0.1:12345) → cuvslam_ros_bridge
    → /state_estimation (Odometry, frame=world)
    → TF: world → aft_mapped
  → UDP landmarks (127.0.0.1:12346) → cuvslam_ros_bridge
    → /cuvslam/landmarks (per-frame)
    → /cuvslam/final_landmarks

global_map_injector (route_planner only)
  → /terrain_map_ext (TRANSIENT_LOCAL, frame=map, from PLY)

localPlanner
  ← /state_estimation, /terrain_map, /registered_scan
  → /path, /way_point

pathFollower
  ← /path, /way_point
  → cmd_vel (to robot)

far_planner (route_planner only)
  ← /state_estimation, /terrain_map, /terrain_map_ext, /registered_scan
  → /overall_map, /goal_point, /viz_*_topic, /far_reach_goal_status, /navigation_boundary
  Remap: /odom_world←/state_estimation, /terrain_cloud←/terrain_map_ext, /scan_cloud←/terrain_map
```

---

## 7. 关键参数与环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `DATA_INTERFACE` | enP8p1s0 | Jetson DDS 数据网卡 |
| `VIEWER_INTERFACE` | (自动检测) | 到 VIEWER_IP 的网卡 |
| `VIEWER_IP` | (从 SSH_CLIENT 自动) | 本地 PC 的 IP |
| `VIZ_DOMAIN_ID` | 77 | 可视化 DDS 域 ID |
| `VIZ_FAST_PORT` | 37665 | 高频小数据 TCP 端口 |
| `VIZ_CLOUD_PORT` | 37666 | 点云数据 TCP 端口 |
| `CONTROL_PORT` | 37667 | 控制指令 TCP 端口 |
| `CUVSLAM_MAP_DIR` | (空) | 已有地图目录，用于 localization-only |
| `WARMUP_SEC` | 8 | 相机初始化和预热等待秒数 |
| `MAX_SPEED` | 0.35 | pathFollower 最大线速度 |
| `AUTONOMY_SPEED` | 0.25 | 自主模式巡航速度 |
| `CUVSLAM_OFFSET_*` | 0.0 | cuVSLAM 位姿偏移 |
| `WORLD_T_CAM_INIT_*` | 0.0 / 0° | world→camera_init 静态变换 |
| `PUBLISH_REGISTERED_MAP` | 0 | 是否发布累积 /registered_map |
| `ENABLE_TERRAIN_MAP_VIZ_RELAY` | 1 | /terrain_map → /terrain_map_viz 降采样转发 |
| `ENABLE_CRUISE_DEBUG_MARKERS` | 0 | 调试 Marker 开关 |
| `ENABLE_VIEWER_DDS` | 0 | DDS Viz 桥接开关 |
| `ENABLE_RVIZ` | 0 | SSH X11 Forwarding RViz |
| `REMOTE_HOST` | unitree@192.168.31.81 | 远程代理目标地址 |

---

## 8. 常见场景

### 8.1 本地 PC 查看机器人状态

```bash
# Jetson (go2) 上启动：
ENABLE_VIEWER_DDS=1 ./system_cuvslam_cruise_headless.sh

# 本地 PC 上启动 viewer：
./system_cuvslam_cruise_local_viewer.sh
```

### 8.2 加载已有地图 + 巡航

```bash
CUVSLAM_MAP_DIR=~/datasets/cuvslam_maps/realsense_stereo_slam_20260625_091204 \
  ./system_cuvslam_cruise_route_planner_headless.sh
```

### 8.3 带 RViz 的远程模式

```bash
# 在本地 PC 上执行（会自动 ssh 到 Jetson）：
ENABLE_RVIZ=1 ./system_cuvslam_cruise_headless.sh
```

### 8.4 仅 DDS Viewer（无 RViz）

```bash
# Jetson:
ENABLE_VIEWER_DDS=1 ./system_cuvslam_cruise_route_planner_headless.sh

# 本地 PC:
./system_cuvslam_cruise_local_viewer.sh
```
