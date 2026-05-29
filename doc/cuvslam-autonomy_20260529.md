# CUVSLAM-Autonomy 文档

本文档旨在梳理近期 CUVSLAM-Autonomy 项目的结构和数据流细节，帮助开发者更好地理解和使用该系统。

> 文档写于2026.5.29，请注意时效

## 结构简述

CUVSLAM-Autonomy 主要可以看作两个部分。上游是 SLAM 系统，负责提供位姿和地图信息；下游是 base autonomy 自主导航系统，负责根据 SLAM 提供的信息进行路径规划和控制。

该方案主要是把原本的 Point-LIO SLAM 替换为 CuVSLAM 视觉 SLAM，并在此基础上扩展功能。由于是分层结构，理论上 CuVSLAM 和 base autonomy 可以独立开发和优化，但在实际应用中需要保证两者之间的数据接口和通信效率。

该项目的开发流程是先实现 CuVSLAM 单链路，确保其稳定运行和高性能，然后再逐步集成到 base autonomy 中，最终实现完整的自主导航系统。下文将按照此顺序进行详细说明。

## cuVSLAM-Autonomy 集成

cuVSLAM-Autonomy 集成方案将 cuVSLAM（视觉 SLAM）作为上游定位模块，接入 `autonomy_stack_go2` 自主导航系统，替代原有的 Point-LIO 激光 SLAM。

### 1. 架构概览

cuVSLAM 运行在 Jetson Docker 容器内，只提供**位姿**（通过 UDP）；L1 雷达提供**当前帧点云**（通过 ROS2 topic）；bridge 节点用 cuVSLAM 位姿将 L1 点云变换到世界坐标系，向下游 autonomy 模块输出标准接口。

```
┌─ Docker Container (--network=host) ─────────────────────┐
│  cuVSLAM (RealSense D435i, preset=robot, use_imu)       │
│    UDP pose :12345 ↓   UDP landmarks :↓ 12346           │
└────────────────────┬──────────────────┬─────────────────┘
                     │                  │
┌─ Host (Jetson Orin NX, ROS2) ───────────────────────────┐
│  L1 LiDAR → transform_everything (sensor_transformer)   │
│    ↓ /utlidar/transformed_cloud                         │
│                                                         │
│  cuvslam_ros_bridge (Python bridge node)                │
│    ├─ UDP pose 12345 → /state_estimation (Odometry)     │
│    ├─ UDP landmarks 12346 → /cuvslam/landmarks          │
│    ├─ body点云+pose变换 → /registered_scan (PointCloud2) │
│    └─ TF: camera_init → aft_mapped                      │
│         │                                               │
│  terrainAnalysis → localPlanner → pathFollower → Go2    │
└─────────────────────────────────────────────────────────┘
```

### 2. cuvslam_ros_bridge（核心桥接节点）

| 属性 | 值 |
|------|-----|
| **位置** | `autonomy_stack_go2/src/utilities/transform_sensors/transform_sensors/cuvslam_ros_bridge.py` |
| **Node 名** | `cuvslam_ros_bridge` |
| **版本** | v9（顶层独立版：`autonomy_stack_go2/cuvslam_ros_bridge.py`） |

**发布 Topic：**

| Topic | 类型 | 频率 | 说明 |
|-------|------|------|------|
| `/state_estimation` | `nav_msgs/Odometry` | ~60Hz | **主定位输出**，frame: `camera_init→aft_mapped` |
| `/registered_scan` | `sensor_msgs/PointCloud2` | ~15Hz | L1 当前帧点云经 cuVSLAM 位姿变换到世界系 |
| `/registered_map` | `sensor_msgs/PointCloud2` | ~1Hz | 低频累计地图（可选，默认关闭） |
| `/cuvslam/landmarks` | `sensor_msgs/PointCloud2` | ~10Hz | cuVSLAM 每帧地标 |
| `/cuvslam/final_landmarks` | `sensor_msgs/PointCloud2` | ~10Hz | cuVSLAM 累积地图地标 |

**订阅 Topic：**

| Topic | 类型 | 来源 | 说明 |
|-------|------|------|------|
| `/utlidar/transformed_cloud` | `sensor_msgs/PointCloud2` | `transform_everything` | L1 点云经 IMU 校准+机体变换后的 body 帧点云 |

**内部通信：**

| 方式 | 地址 | 方向 | 格式 | 说明 |
|------|------|------|------|------|
| UDP | `127.0.0.1:12345` | Docker→Host | JSON pose | `{tx,ty,tz,qx,qy,qz,qw,timestamp_ns}` |
| UDP | `127.0.0.1:12346` | Docker→Host | JSON landmarks | `{type:"landmarks", points:[[x,y,z],...]}` |

### 3. 坐标转换

cuVSLAM（OpenCV 坐标系）转换成 ROS 坐标系：

```
cuVSLAM:  X=右, Y=下, Z=前
ROS:      X=前, Y=左, Z=上

转换四元数: Q_CV2ROS = [0.5, -0.5, 0.5, -0.5]
   q_ros = Q_CV2ROS * q_cuvslam * Q_CV2ROS_INV
   pos_ros = (tz_cv, -tx_cv, -ty_cv)
```

bridge 还叠加了：
- **135° 地图旋转**（R_MAP 矩阵）：对齐地图朝向
- **Z 轴平滑滤波**：阈值 `Z_DB=0.05`，平滑因子 `Z_SMOOTH=0.3`
- **异常值剔除**：相邻帧位移 >0.5m 或旋转 >30° 时丢弃，连续 5 帧稳定后恢复
- **混合姿态策略**：平移补偿使用完整姿态，对 autonomy 发布的姿态只保留 yaw（去除 roll/pitch，避免四足狗步态抖动污染下游）

### 4. 坐标系约定

| 坐标系 | 说明 |
|--------|------|
| `camera_init` | cuVSLAM 世界系原点 |
| `aft_mapped` | cuVSLAM 机体坐标系 |
| TF `map→camera_init` | 静态单位变换 |
| TF `camera_init→aft_mapped` | 动态，cuVSLAM 实时位姿 |

关键语义：`/state_estimation` 发布的是 **sensor pose**（相机位姿），非 vehicle center pose。下游 `localPlanner/pathFollower` 使用 `sensorOffsetX/Y` 自行换算到 vehicle 位姿。

### 5. DDS 通信

| 属性 | 值 |
|------|-----|
| **中间件** | Cyclone DDS（`rmw_cyclonedds_cpp`） |
| **机器人网卡** | `enP8p1s0`（IP `192.168.123.18`，直连外部电脑） |
| **外部 viewer 网卡** | `wlp1s0`（无线） |
| **Domain ID** | 默认 0；可视化桥接用 Domain 77 隔离 |
| **关键环境变量** | `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp` |

**DDS URI 模板：**
```xml
<CycloneDDS>
  <Domain>
    <General>
      <Interfaces>
        <NetworkInterface name="enP8p1s0" priority="default" multicast="default" />
      </Interfaces>
    </General>
  </Domain>
</CycloneDDS>
```

**Navigation-first 策略**：默认只绑定机器人网卡 `enP8p1s0`，不自动扩到 viewer 网卡。因为实测 DDS 跨网卡扩展会导致 L1 点云从 ~19Hz 掉到 0.1-0.5Hz。这部分使用转发桥接（`cruise_viz_bridge`）通过 TCP 传输数据到 viewer 网卡，避免 DDS 跨网卡问题，解决了L1点云掉速问题。

### 6. 远程可视化 TCP Bridge

巡航模式（`system_cuvslam_cruise_headless.sh`）下使用 `cruise_viz_bridge` 进行远程数据传输：

| 通道 | 端口 | 方向 | 内容 |
|------|------|------|------|
| fast | 37665 | Robot→Viewer | 低流量 topic（path, odom, TF, boundary） |
| cloud | 37666 | Robot→Viewer | 高流量点云 topic |
| control | 37667 | Viewer→Robot | 下发 `/way_point` 和 `/joy` |

格式：pickle 序列化，4 字节长度头 + payload。

### 7. 关键设计决策

**7.1 位姿来自 cuVSLAM，点云来自 L1**

不是直接让 autonomy 使用 cuVSLAM 的稀疏 landmarks 作为规划点云，而是：
- cuVSLAM 提供位姿
- L1 提供当前帧点云
- bridge 用位姿变换点云后发布
- 最大限度复用 autonomy 原有模块（terrainAnalysis、localPlanner 等无需修改）

**7.2 60 FPS 优于 30 FPS**

实际测试确认 cuVSLAM 在 60 FPS 下明显比 30 FPS 更稳定，`robot` preset 已改为 60 FPS。

**7.3 Navigation-first 模式**

默认不以本机可视化影响导航链：
- `ENABLE_VIEWER_DDS=0`：不自动扩 DDS 到 viewer 网卡
- `ENABLE_RVIZ=0`：不启动 RViz
- 遥测数据显示此模式下 `pose_hz=59.9`, `L1 cloud ~= 15.4Hz`, `RegScan ~= 15.4Hz`

**7.4 三颗 IMU 的分工**

| IMU | 用途 |
|-----|------|
| RealSense 相机 IMU | cuVSLAM 主定位来源 |
| L1 LiDAR 自带 IMU | 原始 Point-LIO / transform_sensors 链 |
| 机体 IMU | 当前未直接进入 integrated 主链 |

当前优先保证 `cuVSLAM + 相机 IMU` 自身稳定，不将三颗 IMU 混入同一定位链。

### 8. 相关新增文件

| 文件 | 位置 | 作用 |
|------|------|------|
| `cuvslam_ros_bridge.py` | `autonomy_stack_go2/cuvslam_ros_bridge.py` | v9 bridge（顶层独立版） |
| `cuvslam_ros_bridge.py` | `src/utilities/transform_sensors/` | v8 bridge（包内版本） |
| `transform_everything.py` | 同上 | L1 点云预处理（优化了点云读取路径） |
| `cruise_viz_bridge.py` | 同上 | 远程可视化 TCP bridge |
| `cuvslam_integration.md` | `autonomy_stack_go2/` | 集成方案文档 |
| `cuvslam_autonomy_integration_summary.md` | `autonomy_stack_go2/` | 调试总结文档 |
| `system_cuvslam_landmarks_headless.sh` | `autonomy_stack_go2/` | 启动脚本（headless） |
| `system_cuvslam_cruise_headless.sh` | `autonomy_stack_go2/` | 启动脚本（巡航模式） |
| `system_cuvslam_landmarks_local_viewer.sh` | `/home/jycao/` | 启动脚本（本机 viewer） |

### 9. 启动方式

**远端主链（Jetson）：**
```bash
cd ~/autonomy_stack_go2
ENABLE_RVIZ=0 ./system_cuvslam_landmarks_headless.sh
```

**巡航模式（主要）：**
```bash
cd ~/autonomy_stack_go2
ENABLE_VIEWER_DDS=0 ENABLE_RVIZ=0 ./system_cuvslam_cruise_headless.sh
```

**本机 viewer：**
```bash
DDS_INTERFACE=wlp1s0 /home/jycao/autonomy_stack_go2/system_cuvslam_landmarks_local_viewer.sh
```

**本机 viewer（巡航模式）（主要）：**
```bash
cd ~/autonomy_stack_go2
ENABLE_VIEWER_DDS=1 ENABLE_RVIZ=1 ./system_cuvslam_cruise_headless.sh
```

### 10. 已实现功能

1. 稳定利用 cuVSLAM 位姿，并对齐到 autonomy 坐标系
2. 初步实现简单短距离导航功能

### 11. 已知问题

- **RealSense D435i 在 Jetson Orin L4T 上不稳定** — 有时 `wait_for_frames()` 超时，需 USB 重置
- **DDS 跨网卡导致 L1 掉速** — 开启 viewer DDS 后 L1 点云从 ~19Hz 降到 0.1-0.5Hz，已通过转发解决
- **本地 RViz 卡顿** — 初始 `/registered_scan` 过度发布累计点云导致，已修复为只发当前帧
- **旋转时位置漂移** — 大幅减弱但仍有少量跳变，需进一步调试 cuVSLAM 旋转稳定性
- **本地 viewer TCP 发现问题** — 不同机器狗需要设置不同的 DDS Domain ID 才能正确接收数据，否则会接收到其他机器狗的数据，导致显示异常
- **本地 viewer 显示异常** — 偶发启动时无法显示 /terrain_map_viz 和 /way_point，需重启巡航主栈

---

## CuVSLAM 单链路

### 1. 核心功能

#### 1.1 视觉跟踪模式
- **Stereo（双目）**：基于特征匹配的立体视觉里程计与SLAM
  - 轨迹通常更稳定
  - 适合纹理丰富场景
  - 可视化数据量相对较小
  
- **RGBD（RGB-D）**：基于深度图的视觉SLAM
  - 近距离轮廓更完整
  - 更容易看出场景结构
  - 深度噪声和反光表面容易污染地图

#### 1.2 传感器融合
- **IMU融合**：与机器狗IMU传感器融合（Stereo模式下）
- **运动模型**：基于刚体运动的估计和优化
- **深度立体跟踪**：RGBD模式下增加2D立体跟踪约束

#### 1.3 约束优化
- **平面约束**（--planar）：主要用于四足机器狗水平面运动
- **地图单元网格化**：支持可调的地图细粒度控制

#### 1.4 地图重定位（新增）

支持载入已有的 `map/data.mdb` 进行重定位，将当前相机帧与数据库中的地标进行匹配。核心算法为**异步探针搜索 + 两级闭环检测**：

1. **探针生成**：以初始提示位姿为中心，在指定半径内生成螺旋排序的搜索网格（平移步长 × 角度步长）
2. **快速闭环**（`FastLoopClosure`）：使用 `SimplePoint` 求解器（无 RANSAC），快速筛选候选位姿
3. **精确闭环**（`AccurateLoopClosure`）：在快速筛选通过后，使用 `TwoStepsEasy` + PnP RANSAC 进行精化
4. **加权评估**：权重函数 `w = (good/tracked) × 2·atan(good/500)/π`，自动选择最优结果

定位成功后的 `slam_pose` 会进入已加载地图的坐标系。

---



### 2. 系统架构

#### 2.1 部署层级
```
Host (Jetson Orin/Thor)
  └─ Docker Container (Ubuntu 22.04/24.04)
      ├─ cuVSLAM 建图/定位进程（Stereo/RGBD）
      │   ├─ [新增] LocalizationManager（Python 协调层）
      │   ├─ [新增] PoseHintTracker（位姿提示跟踪）
      │   └─ [新增] 启动位姿采样引擎（从轨迹文件抽帧）
      ├─ Python Binding 层（Tracker API, localize_in_map）
      ├─ C++ 核心库（tracking & mapping & async_localizer）
      │   ├─ [新增] AsyncLocalizer（异步探针搜索定位器）
      │   ├─ [新增] FastLoopClosure（SimplePoint 快速筛选）
      │   └─ [新增] AccurateLoopClosure（TwoStepsEasy + PnP RANSAC）
      ├─ 可视化模块（Rerun SDK）
      └─ 地图工具（map_extractor）
```

#### 2.2 关键组件
| 组件 | 位置 | 功能 |
|------|------|------|
| **启动脚本** | `docker/start_realsense_slam_map.sh` | Docker编排、参数管理、生命周期 |
| **建图Worker** | `examples/realsense/run_stereo_slam_map.py` | 双目建图/定位 |
| **建图Worker** | `examples/realsense/run_rgbd_slam_map.py` | RGBD建图/定位 |
| **定位协调层** | `examples/realsense/localization_helper.py` | [新增] LocalizationManager、PoseHintTracker、启动位姿采样 |
| **离线回放** | `examples/realsense/view_saved_map.py` | 离线查看轨迹和地图 |
| **可视化模块** | `examples/realsense/visualizer.py` | Rerun数据发送 |
| **核心库** | `libs/` | C++/CUDA实现的Tracker |
| **异步定位器** | `libs/slam/async_localizer/` | [新增] 探针搜索 + 两级闭环定位核心 |
| **地图提取** | `tools/map_extractor/` | 从data.mdb提取map.json |

---

### 3. 建图输出产物

#### 3.1 轨迹文件（TUM格式）
```
trajectory_odom_tum.txt              # 原始里程计轨迹
trajectory_slam_live_tum.txt         # 实时SLAM轨迹（未优化）
trajectory_slam_optimized_tum.txt    # 最终优化轨迹（首选）
trajectory_slam_map_tum.txt          # 地图坐标系下的轨迹
```

#### 3.2 地图数据
```
map/data.mdb                         # SLAM后端可复用的地图数据库
map.json                             # 稀疏地标点（OpenCV坐标系）
final_landmarks.csv                  # 稀疏地标点导出（已废弃/兼容保留）
```

#### 3.3 其他产物
```
session.log                          # 建图过程日志
```

#### 3.4 产物特性
- **稀疏地图**，不是稠密点云
- 不是彩色点云
- 不是占据栅格地图
- 可作为后续定位、重定位的地图源
- **仅重定位模式**（`--localize-only`）不产生 `map/data.mdb`，只输出轨迹文本

---

### 4. 运行模式与预设

#### 4.1 三种性能预设

##### FAST（快速）
- 用于快速验证链路可用性
- 地标日志间隔：15帧
- 可视化频率：4-6帧一更新
- 更低的可视化带宽

##### LOWLOAD（低负载）
- 嵌入式设备优化
- 分辨率：424×240（vs 640×360）
- 帧率：15fps（vs 60fps）
- 更大的网格单元（0.4-0.5m）
- 适合防止Jetson掉帧

##### ROBOT（平衡，默认）
- 针对四足机器狗优化
- 分辨率：640×360
- 帧率：60fps
- 支持IMU融合（Stereo）
- 支持深度立体跟踪（RGBD）
- 支持平面约束（RGBD）

#### 4.2 可视化模式
- **实时流式**：通过Rerun实时发送轨迹、地标、位姿
- **仅保存**（默认）：后台建图，不向本机发送数据
- **离线回放**：建图完成后加载已保存结果进行查看
- **Headless**：无可视化，纯后端建图

#### 4.3 重定位模式（新增）

##### 4.3.1 载入已有地图并重定位
使用 `--input-map-dir` 指定历史会话目录：
```bash
./start_realsense_slam_map.sh --mode stereo --input-map-dir /home/unitree/datasets/cuvslam_maps/<旧会话目录>
```
- 正常实时 `track()`，启动后调用 `localize_in_map()` 在旧图中定位
- 未显式提供 `--initial-guess-*` 时，自动从旧会话轨迹文件（`trajectory_slam_optimized_tum.txt` > `trajectory_slam_map_tum.txt` > `trajectory_slam_live_tum.txt` > `trajectory_odom_tum.txt`）抽样最多 48 个候选位姿逐次尝试
- 定位成功后 `slam_pose` 进入该地图坐标系
- `--input-map-dir` 可传整次输出目录或直接传 `.../map`

##### 4.3.2 只重定位/定位校准
```bash
./start_realsense_slam_map.sh --mode stereo --input-map-dir <旧会话目录> --localize-only
```
- 仍输出本次轨迹文本到新 `output-dir`
- **不调用 `save_map()`**，不生成新的 `map/data.mdb`
- 适合"开机定位""局部校准""验证是否仍在旧图中"

##### 4.3.3 跟踪丢失后自动重定位
```bash
./start_realsense_slam_map.sh --mode stereo --input-map-dir <旧会话目录> -- --relocalize-on-tracking-loss
```
- 只在**成功定位过一次后**才启用
- 默认冷却时间 5 秒（`--relocalize-cooldown-sec`）
- 使用最近一次成功 `slam_pose` 作为 hint，非全局暴力搜索
- 快速移动/遮挡严重/视角突变时仍可能无法找回

#### 4.4 传输优化
- **--no-send-images**：不发送RGB/D图像，仅发送轨迹和地标
- **--send-images**：发送完整图像帧（高带宽）
- **--viz-every-n**：降低可视化更新频率

---

### 5. 工作流

#### 5.1 在线建图工作流
```
1. 初始化环境
   ├─ 启动USB电源服务
   ├─ 检测主机RealSense设备
   └─ 启动/复用Docker容器

2. 准备建图会话
   ├─ 停止旧进程（优雅关闭+强制杀死）
   ├─ 检查容器内RealSense可用性
   ├─ 应用性能预设参数
   ├─ 自动检测可视化地址（若通过SSH）
   └─ 生成输出目录

3. 启动建图进程
   └─ 后台运行Stereo/RGBD建图

4. 可视化（可选）
   ├─ 实时发送轨迹、地标到本机Rerun
   ├─ 支持降频更新（--viz-every-n）
   └─ 支持无图像模式（--no-send-images）
```

#### 5.2 离线查看工作流
```
1. 检查是否存在map.json
2. 若缺失，调用map_extractor从data.mdb提取
3. 选择轨迹文件（优先optimized > live，自动回退）
4. 加载到Rerun进行离线回放
   ├─ 完整显示最终轨迹和稀疏地图点
   ├─ 时间轴回放位姿和渐进轨迹
   └─ 支持交互查看
```

#### 5.3 地图重定位工作流（新增）
```
1. 准备阶段
   ├─ 传入 --input-map-dir（旧会话目录或 map/）
   ├─ 创建 LocalizationManager + PoseHintTracker
   └─ 采样候选位姿：
       ├─ 有 --initial-guess-* 时仅用该位姿
       └─ 无提示时从旧轨迹文件均匀抽最多48个候选

2. 建图循环（每帧执行）
   ├─ tracker.track(timestamp, images) ← 持续运行
   ├─ 若尚未定位成功且还有候选位姿：
   │   └─ localization_manager.start(候选位姿, images)
   │       └─ 异步调用 tracker.localize_in_map(...)
   │           ├─ AsyncLocalizer: 以提示位姿为中心生成螺旋探针
   │           ├─ 对每个探针: FastLC(SimplePoint) → 快速筛选
   │           ├─ 若通过: AccurateLC(TwoStepsEasy+PnP) → 精化
   │           └─ 选择最优结果触发回调
   └─ 若定位成功:
       ├─ has_localized_pose = True
       ├─ 后续 slam_pose 进入地图坐标系
       └─ pose_hint_tracker 开始记录最新位姿

3. 跟踪丢失恢复（--relocalize-on-tracking-loss）
   ├─ 仅在 has_localized_pose = True 后生效
   └─ 当 pose_estimate.world_from_rig is None：
       ├─ 检查 cooldown（默认5秒）
       └─ 用最近 slam_pose 作为 hint 重新发起定位

4. 结束阶段
   ├─ 写轨迹文件（odom + slam_live + slam_optimized）
   ├─ 写 final_landmarks.csv
   ├─ localize-only 模式: 跳过 save_map()
   └─ 正常模式: 保存新的 map/data.mdb
```

---

### 6. 关键参数

#### 6.1 通用参数
| 参数 | 说明 |
|------|------|
| `--width / --height` | 输入图像分辨率 |
| `--fps` | 处理帧率 |
| `--max-landmarks-distance` | 地标有效范围（米） |
| `--map-cell-size` | 地图网格大小（米） |
| `--landmark-log-interval` | 地标日志记录间隔（帧） |
| `--viz-every-n` | 可视化更新频率（帧） |

#### 6.2 重定位参数（新增）
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--input-map-dir PATH` | - | 载入已有地图目录进行重定位 |
| `--localize-only` | false | 只定位不保存新地图 |
| `--relocalize-on-tracking-loss` | false | 跟踪丢失后自动重定位 |
| `--relocalize-cooldown-sec` | 5.0 | 重试冷却时间（秒） |
| `--initial-guess-x/y/z` | 0.0 | 初始提示位姿平移（米） |
| `--initial-guess-qx/qy/qz/qw` | 0,0,0,1 | 初始提示位姿旋转（四元数） |
| `--localization-horizontal-radius` | 8.0 | 水平搜索半径（米） |
| `--localization-vertical-radius` | 2.0 | 垂直搜索半径（米） |
| `--localization-horizontal-step` | 0.5 | 水平搜索步长（米） |
| `--localization-vertical-step` | 0.2 | 垂直搜索步长（米） |
| `--localization-angular-step-rads` | 0.03 | 角度搜索步长（弧度） |

注：C++ 核心层（`AsyncLocalizerOptions`）默认搜索范围更保守（水平 1.5m，垂直 0.5m），Python 层默认值更宽松（水平 8m，垂直 2m），以适应更大范围的开机定位。

#### 6.3 特殊标志
| 标志 | 说明 |
|------|------|
| `--use-motion-model` | 启用运动模型估计 |
| `--use-imu` | 融合IMU数据（Stereo） |
| `--enable-depth-stereo-tracking` | 深度立体跟踪（RGBD） |
| `--planar` | 平面约束（RGBD） |
| `--no-send-images` | 禁用图像流传输 |

---

### 7. 应用场景

#### 7.1 推荐场景
- ✅ 四足机器狗SLAM建图与定位
- ✅ 视觉里程计+ IMU融合
- ✅ 实时轨迹追踪
- ✅ 稀疏地图构建（后续定位源）
- ✅ **载入已有地图重定位**（`--input-map-dir`）
- ✅ **定位校准/开机定位**（`--localize-only`）
- ✅ **跟踪丢失自动恢复**（`--relocalize-on-tracking-loss`）
- ✅ 远程可视化与监控

#### 7.2 不适用场景
- ❌ 需要稠密3D点云或网格重建
- ❌ 需要彩色点云
- ❌ 需要占据栅格地图
- ❌ 需要语义分割或目标检测

#### 7.3 质量要求
- 📷 场景需要丰富纹理（避免白墙、玻璃）
- 📏 VGA分辨率或更高
- ⏱️ 30fps以上帧率（理想60fps）
- 🔄 多次观察相同区域（闭环检测）

---

### 8. Docker部署

#### 8.1 镜像支持
| OS | CUDA | Python | Arch |
|----|------|--------|------|
| Ubuntu 22.04 | 12/13 | 3.10 | x86_64/aarch64 |
| Ubuntu 24.04+ | 12/13 | 3.12+ | x86_64/aarch64 |

初试部署时，使用 `docker/run_docker_fixed.sh` 构建镜像。

目前已知存在问题，镜像中 CUDA 架构与 go2 jetson 板卡不兼容，导致例程无法正常运行，需要进行修复。

#### 8.2 关键挂载
- 仓库：`/home/unitree/cuVSLAM` -> `/cuvslam`
- 数据：`/home/unitree/datasets` -> `/home/unitree/datasets`
- USB：`/dev/bus/usb` -> `/dev/bus/usb`

---

### 9. 典型用法

#### 9.1 最常用
```bash
# Stereo（推荐默认）
./start_realsense_slam_map.sh --mode stereo

# RGBD
./start_realsense_slam_map.sh --mode rgbd
```

#### 9.2 低功耗
```bash
./start_realsense_slam_map.sh --mode stereo --preset lowload
```

#### 9.3 快速验证
```bash
./start_realsense_slam_map.sh --mode stereo --preset fast --no-send-images
```

#### 9.4 地图重定位（新增）
```bash
# 载入已有地图重定位
./start_realsense_slam_map.sh --mode stereo --input-map-dir <旧会话目录>

# 只定位不保存新图
./start_realsense_slam_map.sh --mode stereo --input-map-dir <旧会话目录> --localize-only

# 带初始提示位姿
./start_realsense_slam_map.sh --mode stereo --input-map-dir <旧会话目录> \ 
--initial-guess-x 1.2 --initial-guess-y 0.0

# 跟踪丢失自动恢复
./start_realsense_slam_map.sh --mode stereo --input-map-dir <旧会话目录> \ 
--relocalize-on-tracking-loss
```

#### 9.5 停止与回放
```bash
# 停止
./start_realsense_slam_map.sh --stop

# 离线回放
./start_realsense_slam_map.sh --view-output-dir <输出目录>
```

---

### 10. 输出目录结构

```
/home/unitree/datasets/cuvslam_maps/
├── realsense_stereo_slam_20260529_114000/
│   ├── session.log
│   ├── trajectory_odom_tum.txt
│   ├── trajectory_slam_live_tum.txt
│   ├── trajectory_slam_optimized_tum.txt
│   ├── trajectory_slam_map_tum.txt
│   ├── final_landmarks.csv
│   ├── map/
│   │   └── data.mdb
│   └── map.json
└── realsense_rgbd_slam_20260529_115000/
    └── [同上结构]
```

---

## 下一步计划

1. 完善局部导航功能，实现简单的避障和路径跟随
2. 引入全局静态地图，实现基于地图的长距离路径规划和重定位
3. 实现简单的上位机可视化界面，可视化离线轨迹和地图，并支持远程指定导航目标
