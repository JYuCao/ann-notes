# cuvslam-autonomy dev 问题日志

## 文件调用链

```
system_cuvslam_cruise_route_planner_headless.sh
└── ros2 launch vehicle_simulator system_cuvslam_cruise_with_route_planner.launch
    ├── mapping_cuvslam.launch
    │   ├── transform_everything (L1 点云预处理)
    │   └── cuvslam_ros_bridge (cuVSLAM UDP → ROS 桥接)
    ├── local_planner_cruise.launch
    │   ├── localPlanner （订阅 /terrain_map 做局部路径规划）
    │   └── pathFollower (路径跟踪)
    ├── terrain_analysis.launch
    │   └── terrainAnalysis
    ├── terrain_analysis_ext.launch
    │   └── terrainAnalysisExt (扩展地形，供 far_planner 使用)
    ├── far_planner.launch
    │   └── farMaster (全局路径规划，config=cuvslam_go2)
    ├── joy_node
    └── static_transform_publisher x3 (map, aft_mapped→sensor, aft_mapped→camera_link)
```

## 开发日志

### filter box

filter box 是 src/utilities/transform_sensors/transform_sensors/transform_everything.py 中的一个功能模块，用于屏蔽掉机器狗自身的点云数据，以避免在避障和路径规划过程中受到干扰。

原始 LiDAR 点云经过变换得到 Body 坐标系的点。**filter box 来源于 Body 坐标系**，并进行了 yaw 45 度旋转与 xoy 平面平移，形成一个长方体区域。filter box 的坐标系几乎与所有坐标系解耦，只和 body tf 相关。

**原问题**：filter box 的位置错误出现在机器狗的右前方不远处，导致无法正确屏蔽掉机器狗自身的点云数据。

**诊断**：经检查发现，filter box 缺少 yaw 角度旋转，且由于与其他 tf 耦合性强导致难以调整。

**解决方法**：重写 transform_everything.py 并对 cuvslam_ros_bridge 的坐标变换部分进行轻量修改，重新设计 filter box 的坐标系，使其仅与 body tf 相关，并加入 yaw 角度旋转，使其正确对齐机器狗的坐标系，最后进行平移，从而正确屏蔽掉机器狗自身的点云数据。

**修复结果**：filter box 已经正确对齐机器狗的坐标系，能够有效屏蔽掉机器狗自身的点云数据，避免了在避障和路径规划过程中受到干扰。

### TF 链

原始 TF 链：

```
map
└── camera_init
    └── aft_mapped
        ├── sensor
        └── camera_link
```

* map: 全局固定轴、世界原点。由 static_transform_publisher 发布
* camera_init: cuVSLAM 的世界坐标系，里程计/位姿所在的坐标系。由 cuvslam_ros_bridge 动态发布
* aft_mapped: 机身坐标系，由 cuvslam_ros_bridge 动态发布
* sensor: 传感器坐标系，由 static_transform_publisher 发布
* camera_link: 相机坐标系，由 static_transform_publisher 发布

**现存问题**：cuVSLAM 的 camera_init（即map，两者目前完全重合）的原点和朝向是由 SLAM 启动时第一帧相机位姿决定的，是任意的

**修复方案**：增添写死的 world tf 固定 map 的原点和朝向，之后将 camera_init 和 aft_mapped 的 tf 发布改为相对于 world 发布，这样就能保证每次启动 SLAM 时 map 的原点和朝向都是固定的了。

**修复结果**：SLAM 启动后，只要重定位成功，map 的朝向就会固定在 world 坐标系下，保证了重定位后地图的正确对齐。

修复后的 TF 链：

```
world
└── map
    └── aft_mapped
        ├── sensor
        └── camera_link
```

### 局部导航

已修复的问题：

1. 启动巡航后，设置waypoint机器狗不动，最终发现是 local_planner 在 launch 文件中的 node 被错误注释，已修改。

目前存在的问题：

1. 机器狗到达 waypoint 位置时在原地抽搐。
2. 机器狗会把狭窄通道识别成障碍物，在通道入口摇头，且会撞上障碍物。
   - 现在屏蔽掉了机器狗行动时重定位，减少干扰
   - 开启 /free_paths 的可视化发布，观察 local_planner 的路径规划结果，分析问题原因。
     - 开启 /free_paths 发布后可视化卡顿严重，采用已有下采样工具链 _downsample_cloud + downsample_xyzi，走 fast TCP 发布；同时对 cuVLSAM 的位姿发布进行偶数过滤，不影响原始跟踪质量的情况下减少发布频率，降低系统负载。
     - 下采样后卡顿明显减少，可以进行诊断。初步诊断发现 free paths 静止时因无规划会变为扇形，属于正常现象；问题出在机器狗进入狭窄通道前的摇晃导致terrain map更新异常，free paths 规划混乱，导致机器狗无法正确识别狭窄通道并通过。可以尝试先降低前进速度和加速度，优化yaw阻尼减少摇晃。
     - 本质是调参问题，在参数调整后现在已经可以平稳通过门宽的狭窄通道了，但是更狭窄的通道仍然无法通过。

### cuVSLAM 相关

已修复的问题：

1. cuVSLAM 的 imu 无法启用，经检查是外围脚本把 host 的 librealsense 覆盖进 docker 容器，host的 librealsense 版本无法识别 D435i 的 imu。
   - 解决方法：在外围脚本中设置优先使用容器内的 librealsense 库，或者更新 host 的 librealsense 版本以支持 D435i 的 imu。
   - 修复结果：cuVSLAM 的 imu 已经成功启用，能够提供更稳定的位姿估计。

### 位姿与地图跳变问题分析

**此完整章节已迁移至 [`pose_jump_chronicle.md`](./pose_jump_chronicle.md)。** 

### 静态建图

> cuVSLAM 几乎不产生稠密点云（稀疏地标，无法用于地形重建），因此基于 L1 雷达点云经 terrainAnalysis 处理后的 `/terrain_map` 进行全局累积建图。

#### 数据流

```
terrain_map_cb (parse xyz once)
├── _global_points (np.array)       # 全局累积
│   ├── GLOBAL_MAP_LEAF = 0.20m    # 体素分辨率
│   ├── GLOBAL_MAP_MAX_POINTS = 500K
│   ├── 每帧: merge → global downsample_xyzi（体素去重 + 上限截断）
│   └── /global_map_points (TRANSIENT_LOCAL, 启动时发布一次)
└── map_points (list)              # 滑动窗口局部地图（避障用）
    ├── MAP_RADIUS = 20m
    ├── MAX_MAP_POINTS = 30K
    ├── leaf = REGISTERED_MAP_LEAF (0.10)
    └── /registered_map (1 Hz)
```

关键实现：
- `_global_points` 是 `np.ndarray`，每帧 `np.vstack` 合入新点后立即执行 `downsample_xyzi` 做全局体素去重（`GLOBAL_MAP_LEAF=0.20`），保证空间密度均匀。无需 `list buffer` / 独立锁。
- `/global_map_points` 使用 `TRANSIENT_LOCAL` QoS（延迟 3s 发布一次，等待 DDS 发现完成），非 1Hz 流式发布，避免序列化开销。
- 全局累积不依赖 `PUBLISH_REGISTERED_MAP`，只要 `_was_localized` 为 True 即运行。

#### 保存/载入（PLY 文件）

- **保存路径**: `$CUVSLAM_MAP_DIR/global_map.ply`（建图模式）或 `last_output_dir/global_map.ply`（回退）
- **自动保存**: `main()` 的 `finally` 块在 Ctrl+C / 异常退出时调用 `_save_global_ply()`，保存前做一次 `downsample_xyzi` 保证文件密度一致
- **导航模式下不保存**: `CUVSLAM_MAP_DIR` 已设置时（导航/定位模式）跳过保存，防止覆盖原始完整地图
- **启动载入**: `__init__` → `_load_global_ply()` 读取 PLY，降采样到 `GLOBAL_MAP_LEAF` 后赋值给 `_ply_pts` 和 `_global_points`
- **无 `/save_map` service**: 仅依靠 shutdown 自动保存（简化设计）
- **`static_map_publisher` 已废弃**: bridge 自身完成加载+累积+发布

#### 当前状态

- **密度**: 全局均匀，≤1 点/0.20m³。50m×50m 场景约 6 万点，PLY 文件约 3MB
- **覆盖保护**: 导航模式不写回原地图，避免地图退化
- **性能**: 纯 numpy 操作（无 Python 级循环），单次 `downsample_xyzi` 约 10-50ms

#### 已知限制

1. **位姿跳变导致重影**: cuVSLAM 位姿在重定位/大转角时仍有微跳变（1-3cm / 0.5-1°），累积到全局点云表现为重影/拖尾。详见 [`pose_jump_chronicle.md`](./pose_jump_chronicle.md)。
2. **导航模式不更新地图**: 出于保护，导航模式不做累积保存。如需增量更新需手动切建图模式或添加增量保存策略。

### Far Planner 全局路径规划

feat: 将 far_planner 集成到 cuVSLAM + autonomy_stack_go2 导航栈中，实现 "RViz 设定目标点 → far_planner 生成路径 → local_planner 跟踪" 的完整闭环。

#### 架构

```
RViz GoalpointTool (key g)
  └── /goal_point
      └── cruise_viz_bridge (control_source → TCP → control_sink)  # 跨网络转发
          └── far_planner (/goal_point, frame=map)
              ├── /way_point           → localPlanner → pathFollower → robot
              ├── /viz_path_topic      (规划路径可视化)
              ├── /viz_node_topic      (图节点可视化)
              ├── /viz_graph_topic     (图拓扑可视化)
              ├── /overall_map         (全局占据地图)
              ├── /free_paths          (搜索空间可视化)
              └── /far_reach_goal_status (是否可达)
```

两种操作模式：
- **Direct waypoint** → `/way_point` → localPlanner（绕过 far_planner）
- **Goal** → `/goal_point` → far_planner → `/way_point` → localPlanner

#### 文件变更

| 文件 | 变更 |
|------|------|
| `far_planner/launch/far_planner.launch` | 新增 `rviz`(IfCondition)、`rviz_config` arg |
| `far_planner/config/cuvslam_go2.yaml` | GO2 专用配置（`voxel_dim=0.15`, `sensor_range=8.0`, `robot_dim=0.5`） |
| `far_planner/src/far_planner.cpp` | 添加 1Hz 日志速率限制 (`should_log` 计数器) |
| `vehicle_simulator/launch/system_cuvslam_cruise_with_route_planner.launch` | 集成 far_planner + terrain_analysis_ext |
| `transform_sensors/cruise_viz_bridge.py` | 添加 `/goal_point` 控制话题、far_planner 可视化话题(Marker/MarkerArray/Bool/PointCloud2)、cancel 话题 |
| `goalpoint_rviz_plugin/src/goalpoint_tool.cpp` | 快捷鍵 `w`→`g`（避免与 WaypointTool 冲突） |
| `vehicle_simulator.rviz` | 新增 GoalpointTool 插件、GoalPoint/GlobalPath 显示 |
| `system_cuvslam_cruise_route_planner_headless.sh` | headless 启动脚本，含 `sync_runtime_overlay` |

#### 边界可视化问题

`/navigation_boundary` 在单机器人模式下始终为空，因为 `is_boundary` 仅在多机器人模式下被设为 true。尝试过两种修复（全局 is_boundary=true、基于多边形的 boundary 构建）均导致路径规划退化（原本可达区域变成不可达）。结论：单人机模式下 live without boundary。

#### key 配置参数

```yaml
far_planner:
  ros__parameters:
    main_run_freq: 5.0
    voxel_dim: 0.15          # 图搜索体素分辨率
    robot_dim: 0.5           # 机器人尺寸（路径规划用）
    sensor_range: 8.0        # 传感器范围
    terrain_range: 6.0       # 地形图范围
    local_planner_range: 3.0 # local planner 覆盖半径
    world_frame: map
```

### PLY 地图保护与降采样优化

#### 地图覆盖问题

**问题**：导航模式下（`CUVSLAM_MAP_DIR` 已设置），`_save_global_ply()` 在 shutdown 时将 `_global_points`（仅包含本次运行观测到的区域）保存到 `CUVSLAM_MAP_DIR/global_map.ply`，**覆盖**了原始完整的地图文件。下次启动加载的就是不完整的地图。

**修复**：`CUVSLAM_MAP_DIR` 已设置时跳过保存，仅记录 log：

```python
if CUVSLAM_MAP_DIR:
    self.get_logger().info("skipping save to preserve loaded map")
    return
```

#### 密度优化

**问题**：全局点云 `_global_points` 以 `leaf=0.05` 累积，`downsample_xyzi` 的早期返回(`if shape[0] <= max_points: return`)导致体素滤波仅在超上限时执行，跨回调重复点不断堆积，空间密度不均匀（反映机器人经过次数而非真实几何）。

**管道分辨率链**：

| 阶段 | 分辨率 | 消费者 |
|------|--------|--------|
| Raw L1 lidar | ~0.02m | — |
| terrain_analysis planarVoxelSize | **0.2m** | 高程网格 |
| terrain_analysis terrainVoxelSize | **1.0m** | 地形网格 |
| far_planner voxel_dim | **0.15m** | 图搜索 |
| bridge _global_points (旧) | **0.05m** ← 16倍冗余 | PLY 保存 |
| bridge _global_points (新) | **0.20m** | PLY 保存 |

**修复**：
1. `GLOBAL_MAP_LEAF = 0.20`（环境变量可覆写），匹配 terrain_analysis 最细网格，16× 点量减少
2. `GLOBAL_MAP_MAX_POINTS = 500000`（之前 2M → 随机 1.5M）
3. 移除 `downsample_xyzi` 的早期返回，每次调用都先做体素滤波再截断上限
4. `terrain_map_cb` 合入新点后调用 `downsample_xyzi(self._global_points, 0.20, 500K)` 保证全局均匀
5. `_load_global_ply` / `_save_global_ply` 统一使用相同 leaf

**效果**：PLY 文件从 ~50MB 降至 ~3MB（50m×50m 场景），空间密度完全均匀（≤1点/0.20m³），不损失下游消费者所需信息。

### Local Planner 配置

**问题**：`local_planner.launch` 默认参数激进（`maxYawRate=80°s`、`yawRateGain=1.5`），90°/180° 转弯时离墙太近。

**修复**：切换到 `local_planner_cruise.launch`（已有更柔和的默认值），保持 `local_planner.launch` git restore 恢复原样。

| 参数 | `local_planner.launch` | `local_planner_cruise.launch` |
|------|----------------------|-----------------------------|
| `maxYawRate` | 80.0°/s | 20.0°/s |
| `yawRateGain` | 1.5 | 0.3 |
| `maxAccel` | 2.0 m/s² | 0.8 m/s² |
| `dirDiffThre` | 0.4 rad | 0.6 rad |
| `dirWeight` | 0.02 | 0.15 |
| `lookAheadDis` | 0.5 m | 0.5 m |
| `pointPerPathThre` | 2 | 10（过松，待调） |
| `autonomySpeed` | 1.0 m/s | 0.22 m/s（系统覆写 0.25） |

**仍存在的问题**：`pointPerPathThre=10` 意味着一个路径需要 10 个障碍点才被阻塞，转弯时容易贴墙。`local_planner_cruise.launch` 中为硬编码值，未暴露为可覆写 arg。
