# cuvslam-autonomy dev 问题日志

## 文件调用链

```
system_cuvslam_cruise_headless.sh
└── ros2 launch vehicle_simulator system_cuvslam_cruise_headless.launch
    ├── mapping_cuvslam.launch
    │   ├── transfrom_everything (L1 点云预处理)
    │   └── cuvslam_ros_bridge (cuVSLAM UDP -> ROS 桥接)
    ├── terrain_analysis.launch
    │   └── terrainAnalysis
    ├── local_planner_cruise.launch
    │   ├── localPlanner （订阅 /terrain_map 做路径规划）
    │   └── pathFollower (路径跟踪)
    ├── joy_node
    └── static_transform_publisher x2 (map->camera_init, aft_mapped->sensor)
```

## 6.1 - 6.7

### 下一步计划

1. feat：尝试使用 terrain_map/cuVLSAM 点云进行静态建图与地图更新。
2. feat：基于静态地图进行重定位校准。

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

1. 将 /landmarks 坐标与现有坐标系对齐，查看点云效果。
    * 最新发现 /landmarks 无法用于稠密地图重建: rviz 中 /landmarks 的点云十分稀疏（单帧数十个点），累计点云 /final_landmarks 几乎看不出地形；官方 [github discussion](https://github.com/nvidia-isaac/cuVSLAM/discussions/28) 中 maintainer 回复 cuVSLAM 几乎不产生稠密点云，建议尝试 nvblox。

2. 尝试使用 L1 雷达点云进行建图

```
terrain_map_cb
├── map_points          # 已有滑动窗口（20m radius，30k max），继续用于 /registered_map
└── global_map_points   # 新增的全局累积，只采样不清旧点，供保存用
```
