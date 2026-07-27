---
date: 2026-07-21
updated: 2026-07-21
name: 曹家宇
email: jycao233@163.com
---

# DDDMR Navigation：架构、机制与使用说明

> 整理时间：2026-07-21
> 上游仓库：[dfl-rlab/dddmr_navigation](https://github.com/dfl-rlab/dddmr_navigation)
> 阅读基准：`main` 分支，commit `1d03673`（2026-07-18）
> 实验室部署参考：Go2 + Pandar XT16 + `~/fast_lio2_xt16_ws`，远端部署快照曾为 `fd61ad5`

## 1. 概览

`dddmr_navigation` 是一套面向移动机器人的 ROS 2 Humble 三维导航工作区。它把传统二维导航中的“建图—定位—感知—全局规划—局部控制—恢复”流程搬到三维点云和地面图上，主要解决坡道、多层地面、立体通道以及三维障碍物标记/清除等二维栅格地图不容易表达的问题。

它不是 Nav2 的配置集合，也不是单独的 SLAM 算法，而是包含约 25 个 ROS 包的一体化导航栈。核心组件包括：

- 基于 LeGO-LOAM-BOR 深度修改的三维建图和特征提取；
- 读取 Pose Graph、按局部子地图工作的 MCL 3D 定位；
- 插件化三维感知和动态代价图 `dGraph`；
- 在地面点云图上运行的 A* 全局规划；
- 类似 Nav2 DWB、但使用三维包围盒碰撞检查的局部轨迹规划器；
- 负责规划、控制、等待和恢复切换的 Point-to-Point 状态机；
- 地图编辑、三维初始位姿、三维目标点等 RViz 工具。

## 2. 适用场景与边界

适合：

- 坡道、起伏地面、无障碍通道和多层结构；
- 轮式、履带、四足或人形等能够接收速度指令的移动平台；
- 具有三维 LiDAR、基本可靠的里程计和完整 TF 的机器人；
- 希望以点云地面图而非二维 OccupancyGrid 做规划的系统。

不应误解为：

- 它不会自动提供底盘驱动。栈输出 `/cmd_vel`，机器人侧仍需实现对应控制接口；
- 它不能替代可靠的传感器标定、时间同步和里程计；
- `dddmr_odom_3d` 只是一个 2D 里程计加 IMU 姿态的演示，不是生产级状态估计器；
- 当前地图、定位和特征管线耦合较深，不能把任意 `.pcd` 文件直接当成完整定位地图；
- 深度语义分割和 TensorRT 是可选扩展，不是基础导航的必需条件。

## 3. 系统总架构

```text
公共前端
========

 [3D LiDAR] ---> [ImageProjection] ---> [FeatureAssociation] <--- [Odometry + TF]
                   投影 / 地面分割          特征提取 / 帧间运动

建图阶段
========

 [FeatureAssociation] ---> [MapOptimization] ---> [Pose Graph]
                            Scan-to-Map + GTSAM       |
                                                      +---> [Pose Graph Editor]

定位阶段
========

 [Pose Graph] ---> [PG Map Server] ---> [SubMaps] --------+
                                                           |
 [ImageProjection] ----------------------------------------+--> [MCL 3D]
                                                           |        |
 [FeatureAssociation] -------------------------------------+        v
                                                        [map -> odom TF]

感知与规划阶段
============

 [PG Map Server] -----+
                      |
 [ImageProjection] ---+---> [Perception 3D] ---> [Static / Dynamic dGraph]
                      |                                      |          |
 [map -> odom TF] ----+                                      |          |
                                                             v          v
                                                    [Global Planner] [Local Planner]
                                                      A* / DWA-aware       ^
                                                             |             |
 [3D Goal] ---> [P2P Move Base FSM] <------------------------+             |
                      |                                                    |
                      +----------------------------------------------------+
                                                                           |
                                                                           v
                                                                      [/cmd_vel]
```

整个工程有两个明显阶段：

1. 建图阶段运行 `lego_loam`，将每个关键帧的位姿、边缘/平面/地面特征和图边保存成 Pose Graph。
2. 导航阶段关闭建图，地图服务器读取 Pose Graph，MCL 在局部子地图内定位，感知层在地面图上叠加动态障碍代价，全局和局部规划器共同产生速度指令。

项目也支持“边建图边导航”的工作流，但首次适配硬件时，先分开验证建图和定位更容易排错。

## 4. 软件包分层

### 4.1 主链包

| 包 | 角色 | 关键输出/能力 |
|---|---|---|
| `lego_loam_bor` | 点云投影、地面分割、特征关联、建图优化 | 特征点云、地面、里程计、关键帧、Pose Graph |
| `cloud_msgs` | LeGO-LOAM 内部消息 | `CloudInfo` |
| `dddmr_pg_map_server` | 读取和发布一个或多个 Pose Graph 地图 | `key_poses`、`mapcloud/mapsurface/mapground`、关键帧服务 |
| `mcl_3dl` | 三维粒子滤波定位与子地图管理 | `map -> odom`、粒子和定位结果 |
| `perception_3d` | 插件化静态/动态三维感知 | 静态图、动态障碍、限速/禁行层、`dGraph` |
| `global_planner` | 地面点云图上的 A* 和动态窗口感知规划 | 全局路径、规划 Action/Service |
| `local_planner` | 路径裁剪、局部轨迹生成与选优 | 最优局部轨迹、控制状态 |
| `trajectory_generators` | 可插拔运动学轨迹采样 | 差速直行/转向、原地旋转等候选轨迹 |
| `mpc_critics` | 可插拔轨迹评分器 | 碰撞、贴合路径、Pure Pursuit、朝向等代价 |
| `recovery_behaviors` | 恢复行为 | 原地旋转、位置控制等 Action |
| `p2p_move_base` | 点到点导航状态机 | `/p2p_move_base` Action、`/cmd_vel` |
| `dddmr_sys_core` | 公共接口和状态枚举 | Action、Service、规划状态定义 |

### 4.2 工具和扩展包

| 包 | 用途 |
|---|---|
| `dddmr_beginner_guide` | Gazebo 与实机的示例 launch、YAML 和 RViz 配置 |
| `dddmr_rviz_default_plugins` | 3D Pose Estimate、3D Goal、点云选择/删除等 RViz 工具 |
| `mapping_panel`、`map_editor_panel` 等 | 交互建图和 Pose Graph 编辑面板 |
| `dddmr_odom_3d` | 2D 里程计结合 IMU 姿态的三维里程计示例 |
| `dddmr_pcl` | 项目自带的 PCL 扩展，例如 `VoxelGridOMP` |
| `dddmr_semantic_segmentation` | 图像语义分割投影到点云 |
| `dddmr_trt` | YOLO/TensorRT 推理支持 |
| `dddmr_explore_and_search` | 探索空间和目标搜索扩展 |

## 5. 坐标系、输入和输出

### 5.1 最小硬件接口

上游实机指南要求：

| 方向 | 默认接口 | 类型 | 含义 |
|---|---|---|---|
| 机器人 → DDDMR | `/lidar_point_cloud` 或 launch 中重映射后的点云 | `sensor_msgs/msg/PointCloud2` | 三维 LiDAR 点云 |
| 机器人 → DDDMR | `/odom` | `nav_msgs/msg/Odometry` | 连续局部里程计 |
| 机器人 → DDDMR | `/tf`、`/tf_static` | `tf2_msgs/msg/TFMessage` | 传感器和机器人坐标关系 |
| DDDMR → 机器人 | `/cmd_vel` | `geometry_msgs/msg/Twist` | 速度控制命令 |

典型 TF 树为：

```text
map
└── odom
    └── base_link
        ├── base_footprint
        └── lidar_link
```

- `odom -> base_link` 一般由里程计或 SLAM 发布，要求连续但允许慢漂移；
- `map -> odom` 由 MCL 发布，用于消除长期漂移并对齐保存地图；
- `base_link -> lidar_link` 必须来自正确外参；
- 如果机体中心不在地面，可增加 `base_footprint`；
- 点云 `header.frame_id` 必须能通过 TF 连接到 `base_link` 和 `map`。

### 5.2 导航数据流

```text
LiDAR PointCloud2
  -> mcl_ip / ImageProjection
  -> segmented_cloud_pure + ground/features
  -> mcl_fa / FeatureAssociation
  -> mcl_3dl + perception_3d

Pose Graph
  -> dddmr_pg_map_server
  -> map feature/surface/ground + key poses
  -> SubMaps / MCL + StaticLayer

MCL map->odom + dynamic perception dGraph
  -> global_planner
  -> global plan
  -> P2P FSM + local_planner
  -> /cmd_vel
```

## 6. 核心机制

### 6.1 建图：修改版 LeGO-LOAM-BOR

`lego_loam` 可执行文件在同一进程中创建多个 ROS 节点，并通过内部 `Channel` 传输结构化结果：

1. `ImageProjection`：把点云投影成深度/距离图，按环和邻接关系分割；根据安装姿态、Ground FOV、坡度和高度阈值提取地面；输出 `segmented_cloud_pure` 等点云。
2. `FeatureAssociation`：提取边缘和平面特征，结合外部轮速里程计或 LiDAR 里程计计算帧间运动。
3. `MapOptimization`：将当前特征与周围关键帧做 Scan-to-Map 优化，按距离/角度阈值保存关键帧，并用 GTSAM 因子图管理位姿约束。
4. Loop Closure：搜索历史关键帧，使用优化 ICP 做闭环匹配，满足适应度阈值后向因子图加入闭环边。
5. Interactive Editor：允许暂停/恢复建图、手工加闭环、编辑或合并 Pose Graph。

`ImageProjection` 的关键参数不是通用默认值，必须按雷达修改：

- `laser.num_vertical_scans`、`num_horizontal_scans`；
- `vertical_angle_bottom/top`、`scan_period`；
- `minimum/maximum_detection_range`；
- `ground_fov_*`、`ground_slope_tolerance`、`ground_dz_tolerance`；
- 非重复扫描雷达使用的 `stitcher_num`；
- `laser.odom_type` 和 `base_ground_frame`。

### 6.2 地图不是单个 PCD，而是 Pose Graph 目录

调用 `/save_mapped_point_cloud` 后，地图通常保存到 `/tmp/<时间戳>`，核心结构为：

```text
<pose_graph_dir>/
├── poses.pcd                 # 每个关键帧的 6DoF 位姿
├── edges.pcd                 # 位姿图边
└── pcd/
    ├── 0_feature.pcd         # 第 0 帧边缘/特征点，base_link 局部坐标
    ├── 0_surface.pcd         # 第 0 帧平面点
    ├── 0_ground.pcd          # 第 0 帧地面点
    └── ...
```

这种格式保留了“位姿—关键帧点云”的对应关系。定位时可只取当前位置附近的关键帧，而不必把超大完整点云全部用于匹配；代价是地图格式与 DDDMR 建图器紧密耦合。

### 6.3 定位：子地图 MCL 3D

DDDMR 的 `mcl_3dl` 基于开源 `mcl_3dl` 深度修改，主要机制为：

- 粒子状态是六自由度位姿，但针对地面移动机器人加入地面约束；
- 只有位移或旋转超过阈值时才更新，机器人静止时减少无效计算；
- 地图服务器发布关键帧位姿，`SubMaps` 用 KD-tree 按 `sub_map_search_radius` 选择邻近关键帧；
- 当前子地图由 Feature/Surface/Ground 关键帧拼接得到；
- 在接近子地图边界前准备 warm-up 子地图，再切换，降低大地图定位成本；
- 粒子评分不是只逐点计数，还使用欧氏聚类和地面法向，改善远处稀疏物体以及长走廊“虚拟打滑”问题；
- 粒子权重归一化、重采样并估计最终位姿，随后发布 `map -> odom`。

上游声称在 Jetson Orin Nano 上可对约 500 m × 500 m 地图使用子地图定位。实际规模仍取决于关键帧密度、搜索半径、粒子数和点云降采样。

### 6.4 感知：Stacked Perception 与 dGraph

`perception_3d` 不是传统二维 costmap，而是围绕地面点云图工作的插件框架。

- `StaticLayer`：读取地图和地面点云，为地面节点建立连接；根据地图边缘、静态障碍和孤立程度施加代价。
- LiDAR/Depth 插件：把当前观测变换到全局坐标，聚类障碍物，执行 Marking、Tracking 和 Clearing。
- `NoEntryLayer`、`SpeedLimitLayer`：把编辑好的点云区域变为禁行或限速约束。
- `PathBlockedStrategy`：为局部规划判断当前路径是否被阻塞。

地面点是图节点，节点间的可达关系和权重构成 `dGraph`。静态层提供基础图；传感器层把动态障碍附近节点加权或置为 lethal，并在障碍消失后清除。全局感知维护更大范围的图，局部感知使用较小窗口和更高频率服务控制器。

### 6.5 全局规划：三维地面图上的 A*

`global_planner` 在地面点云的连接图上运行 A*：

- 起点和终点先投影/搜索到附近可行地面节点；
- 邻接边来自 `StaticLayer` 建立的地面连接；
- `dGraph` 提供静态边界、障碍膨胀和动态障碍代价；
- `turning_weight` 惩罚频繁转向，减少锯齿路径；
- `a_star_expanding_radius` 控制图搜索扩展范围；
- 可选择普通全局规划或 Dynamic-Window-Aware 的短前视重规划接口。

最终路径仍包含三维位置，因而可以沿坡面变化，而不是把不同高度压到同一个二维格子中。

### 6.6 局部规划：候选轨迹 + 插件评分

局部规划器的组织方式类似 DWB：

1. 从全局路径截取机器人前后一定距离的 `prune plan`；
2. 每个 `trajectory_generator` 根据运动学、速度/加速度、仿真时长和采样数生成候选轨迹；
3. 将机器人八个顶点定义的三维 `cuboid` 沿每条候选轨迹变换；
4. 不同 `mpc_critics` 对轨迹评分；
5. 排除碰撞或非法轨迹，选择总代价最低的一条；
6. 输出轨迹首段对应的线速度和角速度。

常用评分器包括：

- `CollisionModel`：三维包围盒与感知点云碰撞；
- `StickPathModel`：贴近裁剪后的全局路径；
- `PurePursuitModel`：位置和朝向跟踪；
- `TowardGlobalPlanModel`：鼓励朝全局路径前进；
- `ShortestAngleModel`：原地旋转时选择较短方向。

差速轨迹生成器还显式考虑轮径、减速比和电机最大转速，避免在线速度已经到电机极限时仍生成需要更高单轮转速的角速度组合。

### 6.7 P2P Move Base：导航状态机

`p2p_move_base` 对外提供 `/p2p_move_base` Action，内部状态大致为：

```text
主路径：

 [开始] ---> [planning] ---> [planning_waitdone]
                                      |
                                      | 获得全局路径
                                      v
                              [align_heading]
                                      |
                                      | 朝向对齐
                                      v
                               [controlling]
                                      |
                                      | 到达位置
                                      v
                           [align_goal_heading]
                                      |
                                      | 位姿容差满足
                                      v
                                    [完成]

异常与重规划分支：

 [planning_waitdone] --规划超时------------> [recovery_waitdone]
 [align_heading]      --振荡/无可行轨迹----> [recovery_waitdone]
 [recovery_waitdone]  --恢复成功-----------> [planning]
 [controlling]        --路径需更新---------> [planning]
 [controlling]        --路径暂时阻塞-------> [waiting] --等待超时--> [planning]
```

状态机同时处理：

- 新目标、取消和抢占；
- 全局规划超时；
- 传感器过期、TF 失败和配置错误时输出零速度；
- 控制失败、振荡检测和恢复 Action；
- 位置到达后的最终朝向对齐；
- 成功、失败和反馈状态返回。

## 7. 官方 Docker 与构建方式

### 7.1 支持的镜像

当前 `dddmr_docker` 提供：

- `dddmr:x64`：Ubuntu 22.04，无 GPU；
- `dddmr:cuda`：x64 + CUDA/TensorRT；
- `dddmr:l4t_r36`：JetPack 6 / L4T r36；
- `dddmr_gz:x64`：Gazebo 示例。

上游 Docker README 标注基础环境为 ROS 2 Humble、PCL 1.15 和 GTSAM 4.2a9。实机已有镜像可能使用不同 PCL 版本，不能只看镜像标签推断 ABI。

### 7.2 获取源码和构建镜像

```bash
cd ~
git clone https://github.com/dfl-rlab/dddmr_navigation.git
cd ~/dddmr_navigation/dddmr_docker/docker_file
./build.bash
```

交互选择：

- 普通 PC：`x64`；
- Jetson JetPack 6：`l4t`；
- Gazebo：`x64_gz`。

### 7.3 创建演示容器

```bash
cd ~/dddmr_navigation/dddmr_docker
./run_demo.bash
```

脚本默认使用 host network、挂载 `/dev` 和 `/tmp`，并将：

```text
~/dddmr_navigation -> /root/dddmr_navigation
~/dddmr_bags       -> /root/dddmr_bags
```

映射进容器。默认容器名是 `dddmr_humble_dev`。

### 7.4 容器内编译

```bash
cd /root/dddmr_navigation
source /opt/ros/humble/setup.bash
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
```

如果机器不使用 TensorRT/语义分割，可按实际依赖跳过可选包。例如：

```bash
colcon build --symlink-install \
  --packages-skip dddmr_explore_and_search dddmr_semantic_segmentation \
  --cmake-args -DCMAKE_BUILD_TYPE=Release
```

## 8. 实机使用流程

### 8.1 启动前检查

```bash
ros2 topic echo /odom --once
ros2 topic echo /lidar_point_cloud --once
ros2 run tf2_ros tf2_echo odom base_link
ros2 run tf2_ros tf2_echo base_link lidar_link
```

重点确认：

- 点云持续更新，时间戳与系统时间一致；
- 点云 frame 能连接到 `base_link`；
- 机器人静止时里程计不会高速漂移；
- 上下坡时姿态和 Z 方向合理；
- `/cmd_vel` 的约定与底盘一致，但首次测试先不要接通执行桥。

### 8.2 建图

按雷达和机器人修改：

- `src/dddmr_beginner_guide/config/airy_tilt45_mapping.yaml`；
- `src/dddmr_beginner_guide/launch/airy_tilt45_mapping.launch` 中的话题重映射和静态 TF。

启动：

```bash
source /opt/ros/humble/setup.bash
source /root/dddmr_navigation/install/setup.bash
ros2 launch dddmr_beginner_guide airy_tilt45_mapping.launch
```

人工驾驶机器人覆盖环境。RViz 中应看到关键帧持续增加、地面和特征点合理分离。完成后保存：

```bash
ros2 service call /save_mapped_point_cloud std_srvs/srv/Empty
```

将生成目录移动到持久挂载：

```bash
mv /tmp/<时间戳> /root/dddmr_bags/
```

建议保存后检查 `poses.pcd`、`edges.pcd` 和 `pcd/*_{feature,surface,ground}.pcd` 的数量是否与关键帧一致。

### 8.3 导航

修改 `airy_tilt45_navigation.yaml`：

- `map1.pose_graph_dir` 指向保存地图；
- MCL 的 `map_frame/odom_frame/robot_frame` 与 TF 对齐；
- 雷达扫描模型和 Ground FOV 与建图时一致；
- 机器人的 `cuboid`、速度、加速度、电机参数正确；
- local/global perception 的点云 topic、窗口和高度阈值正确。

启动：

```bash
source /opt/ros/humble/setup.bash
source /root/dddmr_navigation/install/setup.bash
ros2 launch dddmr_beginner_guide airy_tilt45_navigation.launch
```

操作顺序：

1. 在 RViz 用 `3D Pose Estimate` 给初始位姿；
2. 检查实时特征与地图重合、`map -> base_link` 稳定；
3. 用 `3D Goal Pose` 下发目标；
4. 先观察全局路径和局部候选轨迹，再接通底盘 `/cmd_vel`；
5. 低速验证停止、取消、障碍等待和恢复行为。

## 9. 参数适配优先级

按以下顺序调整，避免多个误差互相掩盖：

1. **TF 和时间**：frame 名、外参、时间戳、DDS 通信；
2. **雷达模型**：扫描线数、水平分辨率、垂直角、周期；
3. **地面提取**：Ground FOV、坡度、安装高度和检测范围；
4. **里程计**：`odom_type`、噪声、MCL 更新阈值；
5. **地图和定位**：地图路径、子地图半径、粒子数、匹配距离；
6. **感知**：高度范围、局部窗口、聚类阈值、障碍膨胀；
7. **几何与运动学**：cuboid、轮径、最大速度、加速度、采样密度；
8. **行为层**：规划/控制耐心时间、等待、恢复次数和目标容差。

不要一开始就调 A* 或 critic 权重。如果 TF、地面分割或机器人包围盒错误，规划权重再精细也不会得到可靠结果。

## 10. Go2 + XT16 + FAST-LIO 实验室接入

这一节是实验室实测适配，不是上游默认配置。

### 10.1 实际拓扑

```text
宿主机 ROS 2 Foxy / CycloneDDS
  ~/fast_lio2_xt16_ws
  ├── hesai_ros_driver       -> /xt16/points_hesai
  ├── xt16_fastlio_adapter   -> FAST-LIO 输入
  └── fastlio_mapping        -> /cloud_registered_body, /Odometry

Docker ROS 2 Humble / CycloneDDS / host network
  ~/dddmr_navigation
  ├── DDDMR 点云输入          <- /cloud_registered_body
  ├── DDDMR 里程计输入        <- /Odometry
  └── DDDMR 导航输出          -> /cmd_vel（尚未桥接 Go2 Sport API）
```

已验证的启动方式：

```bash
cd ~/dddmr_navigation
./start_fastlio_dddmr.sh --mode navigation

# 仅建图栈
./start_fastlio_dddmr.sh --mode mapping
```

脚本先启动 `~/fast_lio2_xt16_ws` 中的 FAST-LIO，等待 `/cloud_registered_body` 和 `/Odometry` 出现，再进入 `dddmr_humble_l4t_dev` 容器启动 DDDMR。使用 host network 和 CycloneDDS，使 Foxy 与 Humble 通过标准 ROS 2 接口互通。

这套接入中的 frame 对应关系为：

```text
FAST-LIO global/odom frame: camera_init
FAST-LIO robot frame:       body
DDDMR map frame:            map
兼容 TF:                    body -> base_link
```

`mcl_feature` 内部仍有固定使用 `base_link` 的代码，所以即使 DDDMR 参数将机器人 frame 设为 `body`，仍需要提供 `body -> base_link` 的静态兼容 TF。

### 10.2 为什么使用 `/cloud_registered_body`

XT16 原始点云带有厂商字段和自定义布局。DDDMR 的 LeGO-LOAM 管线更适合标准 PCL XYZ/I 布局。FAST-LIO 输出的 `/cloud_registered_body` 已经是 body frame 下的标准点云，因此比直接消费 `/xt16/points_hesai` 更稳定，也减少一次格式适配。

### 10.3 Jetson/aarch64 已遇到的问题

实测发现以下上游代码在 Jetson 上更容易暴露未定义行为：

- `dddmr_pcl::VoxelGridOMP::AssignTask` 在采样数少于线程数时可能越界；
- 自定义并行 voxel 合并路径对空结果处理不稳；
- `MultiLayerSpinningLidar` 的聚类中心曾未初始化；
- MCL 收敛期间全局变换可能短暂产生非有限点，进入 PCL 聚类/KD-tree 会触发断言。

当前实验室适配采用了更保守的处理：地面降采样使用 PCL 官方 `VoxelGrid`，聚类累加器显式置零，并在全局变换后移除 NaN/Inf。若以后更新上游，需重新检查这些本地补丁是否已经合入或发生冲突。

### 10.4 当前安全边界

联合栈已验证 FAST-LIO、MCL、全局规划器和局部规划器可同时运行并形成有效的 `map -> body` TF；但当前没有 `/cmd_vel -> Go2 Sport API` 桥，所以只会规划和发布速度，不会直接驱动机器狗。这是实机调参阶段应保留的安全隔离。

## 11. 常见故障

| 现象 | 常见原因 | 检查方式 |
|---|---|---|
| 容器看不到宿主机 topic | RMW 不同、DDS 网卡选错、Domain ID 不同、非 host network | 比较 `RMW_IMPLEMENTATION`、`ROS_DOMAIN_ID`、CycloneDDS 网卡配置 |
| 一直等待 TF | frame 名不一致、缺静态外参、点云 frame 不在 TF 树中 | `tf2_echo map base_link`、检查点云 header |
| `mcl_feature` 无输出 | 雷达扫描参数不匹配、点云字段不兼容、Ground FOV 错 | 查看 `segmented_cloud_pure`、改用标准 PCL 点云 |
| MCL 地图为空或 KD-tree 报空 | `pose_graph_dir` 错、关键帧文件缺失、子地图半径过小 | 检查 Pose Graph 目录和 map server 日志 |
| `dGraph not initialized` | 地面图没加载、感知点云没到、TF 尚未建立 | 检查 `mapground`、`segmented_cloud_pure` 和 `map -> robot` |
| PCL `radiusSearch` NaN 断言 | 点云或定位变换含 NaN/Inf | 在变换后过滤非有限点，检查 MCL 是否发散 |
| 有路径但无速度 | FSM 未进入 controlling、无合法局部轨迹、传感器过期 | 查看 P2P feedback、局部 planner 状态和 obstacle cloud |
| `/cmd_vel` 有值但机器人不动 | 缺底盘桥或 topic/type 不一致 | 单独验证底盘速度接口，勿直接高速度联调 |
| 狭窄处振荡 | cuboid 偏大、膨胀半径大、采样不足、恢复参数激进 | 先校准几何，再调 critic 和恢复参数 |
| 关闭 launch 后残留进程 | 多进程 launch 未按进程组退出 | 外围脚本按 INT→TERM→KILL 分级回收并检查 PID |

## 12. 调试命令速查

```bash
# 节点与话题
ros2 node list
ros2 topic list
ros2 topic info /segmented_cloud_pure
ros2 topic info /Odometry

# TF
ros2 run tf2_ros tf2_echo map base_link
ros2 run tf2_ros tf2_echo odom base_link
ros2 run tf2_ros tf2_echo base_link lidar_link

# 频率
ros2 topic hz /Odometry
ros2 topic hz /segmented_cloud_pure

# Action
ros2 action list
ros2 action info /p2p_move_base

# 参数
ros2 param list /mcl_3dl
ros2 param dump /perception_3d_global

# 保存地图
ros2 service call /save_mapped_point_cloud std_srvs/srv/Empty
```

## 13. 推荐的验收顺序

每一级通过后再进入下一级：

1. 点云、里程计、TF 在同一 ROS Domain 内稳定可见；
2. 单独运行点云投影，地面和非地面分割正确；
3. 建图关键帧增长、闭环合理、Pose Graph 文件完整；
4. 导航模式下地图服务器正确发布地面和特征地图；
5. 初始位姿后 MCL 收敛，`map -> base_link/body` 连续且有限；
6. 静态层和动态 LiDAR 层均完成初始化；
7. 全局规划能返回路径；
8. 局部规划有合法候选轨迹，但先不接底盘；
9. 接通底盘后用低速度、空旷场地验证停止和取消；
10. 最后验证动态障碍、狭窄通道、坡道和恢复行为。

## 14. 结论

DDDMR 的核心价值不是“把 2D costmap 增加一个 Z”，而是用 Pose Graph 保存三维关键帧地图，用地面点云构建可行驶图，再把静态结构、动态障碍和区域规则叠加为 `dGraph`，最终由 A* 和三维局部轨迹控制共同完成导航。

它的优势是三维链路完整、模块可替换、适合坡道和多层结构；主要代价是配置面很大、地图格式耦合、对 TF/点云质量敏感，而且部分自定义 PCL 代码在 ARM 上需要额外稳健性处理。工程接入时，最重要的是严格分层验收，而不是一次性把所有节点拉起后再凭日志猜问题。

## 参考资料

- [DDDMR Navigation 主仓库](https://github.com/dfl-rlab/dddmr_navigation)
- [Beginner Guide](https://github.com/dfl-rlab/dddmr_navigation/tree/main/src/dddmr_beginner_guide)
- [Mapping / LeGO-LOAM-BOR](https://github.com/dfl-rlab/dddmr_navigation/tree/main/src/dddmr_lego_loam)
- [MCL 3D Localization](https://github.com/dfl-rlab/dddmr_navigation/tree/main/src/dddmr_mcl_3dl)
- [Perception 3D](https://github.com/dfl-rlab/dddmr_navigation/tree/main/src/dddmr_perception_3d)
- [Global Planner](https://github.com/dfl-rlab/dddmr_navigation/tree/main/src/dddmr_global_planner)
- [Local Planner](https://github.com/dfl-rlab/dddmr_navigation/tree/main/src/dddmr_local_planner)
- [P2P Move Base](https://github.com/dfl-rlab/dddmr_navigation/tree/main/src/dddmr_p2p_move_base)
- [DDDMR Docker](https://github.com/dfl-rlab/dddmr_navigation/tree/main/dddmr_docker)
