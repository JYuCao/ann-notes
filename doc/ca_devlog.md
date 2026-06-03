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

1. fix：修复局部导航问题，确保机器狗能够正确识别狭窄通道并通过。
2. fix：优化重定位触发条件，减少误触发的情况。
3. feature：尝试使用 terrain_map 进行静态建图与地图更新。

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

### 重定位

已修复的问题：

1. 由于 cuVSLAM 的静态地图没有与 autonomy 部分的坐标系对齐，导致每次重定位虽然成功了，但方向没有统一。解决方法与结果详见 “TF 链” 部分

目前存在的问题：

1. 巡航过程中重定位的触发条件过于敏感，导致在正常巡航过程中偶尔会误触发重定位，导致地图到处乱飞。


### cuVSLAM 相关

已修复的问题：
1. cuVSLAM 的 imu 无法启用，经检查是外围脚本把 host 的 librealsense 覆盖进 docker 容器，host的 librealsense 版本无法识别 D435i 的 imu。
    - 解决方法：在外围脚本中设置优先使用容器内的 librealsense 库，或者更新 host 的 librealsense 版本以支持 D435i 的 imu。
    - 修复结果：cuVSLAM 的 imu 已经成功启用，能够提供更稳定的位姿估计。
