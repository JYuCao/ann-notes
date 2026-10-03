# 禾赛 XT 系列低线束 3D LiDAR 重定位算法调研

> 调研日期：2026-09-17
> 目标平台：Unitree Go2、Jetson Orin NX、ROS 2 Humble、FAST-LIO、DDDMR
> 目标任务：已知目标 `waypoint_id`，机器人移动结束后，离线比较当前三维点云与该 waypoint 的参考场景，判断是否准确到达，并输出相对目标 waypoint 的位姿残差和置信度。不要求持续在线跟踪或发布 TF。

## 1. 结论先行

需求最终澄清后，本项目本质上是**已知目标 waypoint 的离线到达验证**，而不是完整 AMCL，也不是从全部 waypoint 中检索当前位置。推荐主线为：

```text
目标 waypoint_id + 离线 query.pcd / rosbag
        │
        ▼
加载该 waypoint 的一个或多个参考点云
        │
        ▼
去畸变、过滤、短时 query 点云累积
        │
        ▼
里程计初值 + GICP/NDT 配准
        │
        ▼
检查位姿残差、重叠率、inlier 和退化程度
        │
        ▼
ARRIVED / NOT_ARRIVED / UNCERTAIN
```

推荐优先级如下：

1. **ICP 类算法适合这个任务，但首选 GICP，而不是基础 point-to-point ICP。** GICP 同时利用局部表面协方差，对稀疏、密度不均的 3D LiDAR 点云通常更稳。
2. **有可靠里程计初值时：GICP 或 NDT 即可。** FAST-LIO 的移动估计可把 query 预变换到目标 waypoint 邻域，再做精配准。
3. **没有可靠初值或移动误差可能很大时：不能只用 ICP。** 应先用 KISS-Matcher、FPFH+TEASER++ 等全局配准得到粗初值，再用 GICP 精化。
4. **“ICP converged”不等于“已到达”。** 必须同时检查配准后的 `x/y/yaw` 残差、重叠率、inlier ratio、RMSE、退化程度和多个模板的一致性。
5. **每个 waypoint 应允许多个参考模板。** 可覆盖不同朝向、季节、人员/家具状态和局部可见范围；选择几何质量最好的模板结果用于到达判定。
6. **输出应为三态。** `ARRIVED`、`NOT_ARRIVED`、`UNCERTAIN`，点云不足或场景退化时不能强行给二值结论。
7. **MCL、Range-MCL、Scan Context 全库检索不是首版必需项。** Scan Context 只可作为“当前点云确实像目标 waypoint”的辅助检查。

现有 FAST-LIO 和 DDDMR 的主要价值是提供去畸变点云、移动前后的里程计初值和场景切片；当前 `num_particles: 60` 的 MCL 不参与离线到达判定。

## 2. 传感器型号说明

用户描述中的“禾赛 XT60”在禾赛公开产品目录和官方 ROS 驱动支持列表中没有对应型号。官方公开的 XT 系列为 `XT32M/XT32/XT16`；本仓库现有系统也明确使用：

- 点云话题：`/xt16/points_hesai`
- FAST-LIO 工作区：`~/fast_lio2_xt16_ws`
- 线数：16
- 垂直视场配置：约 `-15° ~ +15°`
- 水平视场：360°

因此本文按 **PandarXT-16** 做工程判断。“XT60”很可能是“XT16”的笔误；如果实际设备确为未公开、定制或其他型号，需要重新确认线数、垂直角表、转速、单点时间戳和量程，但算法结论基本不变。

禾赛官方给出的 XT16 典型指标为 320,000 点/秒、360° × 30° 视场、0.18° × 2° 角分辨率、最远 120 m。官方 ROS 2 驱动可输出包含 `x/y/z/intensity/ring/timestamp` 的 `sensor_msgs/PointCloud2`。参考：[XT32M/32/16 产品页](https://www.hesaitech.com/product/xt16-32-32m/)、[HesaiLidar_ROS_2.0](https://github.com/HesaiTechnology/HesaiLidar_ROS_2.0)。

### XT16 对重定位的直接影响

- 16 线单帧垂直结构稀疏，单帧局部特征的可重复性弱于 32/64/128 线雷达。
- 360° 水平视场有利于 Scan Context 一类环形描述子，也有利于走廊外的全局辨识。
- Go2 有俯仰和横滚运动，不能把未校平点云直接当作严格平面车载数据；应使用 IMU/FAST-LIO 做去畸变和姿态补偿。
- 单帧过稀时，应累积短时局部子图，但必须按每帧位姿去畸变后再拼接，不能直接堆叠原始点云。
- 室内重复走廊、相似房间和大平面会造成错误收敛；应使用多模板、几何质量门限和相邻 waypoint hard negative 抑制假阳性。

## 3. 先区分三个容易混淆的问题

| 问题 | 输入先验 | 典型方法 | 能否解决“机器人现在完全不知道在哪” |
|---|---|---|---|
| 位姿跟踪 | 已知上一帧位姿 | ICP、GICP、NDT、FAST-LIO localization | 不能；初值错时易落入局部极值 |
| 地点识别 | 无精确位姿 | Scan Context、STD、BoW3D、MinkLoc3D | 只能给地点/关键帧，通常不够精确 |
| 全局重定位 | 无或很弱的先验 | MCL，或地点检索 + 全局配准 + 精配准 | 能，但不是本文最终目标 |

本文最终需求属于比全局重定位更受约束的问题：**目标 waypoint 已知，只验证是否到达该目标**。因此不需要维护全地图多峰位姿后验，也不需要发布或重置 `map -> odom`；全局重定位方法保留为技术背景和“大初值误差时的粗配准”参考。

## 4. 算法路线对比

### 4.1 三维 Monte Carlo Localization

三维 MCL 与 AMCL 的基本递推相同：

$$
p(x_t\mid z_{1:t},u_{1:t},M)
\propto
p(z_t\mid x_t,M)
\int p(x_t\mid u_t,x_{t-1})
p(x_{t-1}\mid z_{1:t-1},u_{1:t-1},M)\,dx_{t-1}.
$$

区别在于状态可能从平面 `x, y, yaw` 扩为 `x, y, z, roll, pitch, yaw`，测量模型要计算一帧三维点云与三维地图的匹配程度。状态空间和观测计算同时增大，导致全局初始化时粒子数迅速膨胀。

常见观测模型包括：

- 点到最近地图点的 likelihood field；
- 体素距离场；
- 少量射线的 beam model；
- 点云投影后的深度图/直方图相似度；
- 神经网络预测的扫描重叠率；
- 对每个粒子渲染虚拟深度图，再与当前扫描比较。

优点：

- 可表示多峰后验，天然适合“绑架机器人”问题；
- 能连续融合多帧和里程计，而不是单帧一次性猜测；
- 收敛后能自然转入持续跟踪。

缺点：

- 全局 6-DoF 搜索代价很高；
- 大范围地图需要高效距离场、分块地图或渲染加速；
- 对称场景可能长期保持多个错误峰；
- Go2 的运动模型与传统差速车不同，直接套用轮式模型会造成预测不一致。

#### `mcl_3dl`

[`at-wat/mcl_3dl`](https://github.com/at-wat/mcl_3dl) 是最典型的开源 3D AMCL：读取参考点云地图，估计点云的 6-DoF 位姿，使用里程计做运动预测，并提供 `/global_localization` 服务。它混合 likelihood-field 和少量 beam model，以兼顾速度与错误匹配抑制。

局限很明确：

- ROS 1/catkin；ROS 1 EOL 后项目转向 Alpine ROS；
- 只提供经典 MCL，没有 AMCL 常用的 KLD 自适应粒子数；
- README 明确写的是 differential-wheeled robot 运动模型；
- 移植到 ROS 2 后仍要重写 Go2/FAST-LIO 运动模型和 TF 接口。

适合用途：算法基线、理解观测模型、离线验证；不建议直接作为当前主线。

#### MOLA / MRPT 粒子滤波

[MOLA](https://docs.mola-slam.org/latest/localization.html) 提供 ROS 2 下的粒子滤波定位：`mrpt_map_server` 发布 `.mm` 地图，`mrpt_pf_localization` 融合降采样后的 3D LiDAR 和里程计。官方特别提醒不能把原始大点云直接送入 PF，应先用 `mrpt_pointcloud_pipeline` 降采样；并提供了完整的 [warehouse PF tutorial](https://github.com/MOLAorg/mola_warehouse_pf_tutorial)。

优点：

- ROS 2 Humble/Jazzy、amd64/arm64 路径清晰；
- 既有 PF，也有 GICP/NDT 定位、地图服务和标准 TF 接口；
- 比较适合在 Jetson 上快速建立“真正 3D PF”基线。

局限：

- 需要把现有 PCD/关键帧地图转换为 MOLA/MRPT `.mm`；
- 原始点云必须强降采样，参数对速度影响大；
- MOLA 当前采用 open-core，核心模块为 GPLv3，闭源商业部署需单独核对许可；
- 官方 LO 重定位教程主要是“给定近似位姿/协方差后重定位”，不能把它误写成无先验全地图搜索；无先验能力应单独用 PF 或外部地点检索器实现。

#### Range-MCL

Chen 等人的 [Range Image-based LiDAR Localization for Autonomous Vehicles](https://www.ipb.uni-bonn.de/wp-content/papercite-data/pdf/chen2021icra.pdf)（ICRA 2021）把当前点云投影为 range image，并从三角网格地图为每个粒子渲染合成 range image，再计算粒子权重。代码为 [`PRBonn/range-mcl`](https://github.com/PRBonn/range-mcl)，MIT 许可。

它对本项目有两个很有价值的结论：

- 论文在仿真中测试了 8/16/32/64/128 线雷达；16 线定位 RMSE 报告为 0.43 m、yaw RMSE 为 3.87°，说明低线束并非原则上不可用。
- 方法能跨不同 LiDAR 和环境，不需要为每种传感器训练网络。

但工程代价较大：需要由点云重建三角网格，并依赖 OpenGL/GPU。论文使用 10,000 粒子初始化、收敛后降到 100 粒子；在 i7-8700 + GTX 1080 Ti 上，冷启动阶段单帧最慢 56.7 s，收敛后可到 21.8 Hz。因此它更适合作为研究对照，不适合直接承担 Jetson 上的即时恢复。

#### Overlap-based MCL 与优化辅助 MCL

- [Learning an Overlap-based Observation Model for 3D LiDAR Localization](https://arxiv.org/abs/2105.11717)（Chen et al., IROS 2020）用 OverlapNet 预测扫描重叠率和 yaw 差，再作为 MCL 观测模型。它能减少粒子数量，但域外场景和不同线数雷达需要重新验证或训练。
- [Efficient Solution to 3D-LiDAR-based Monte Carlo Localization with Fusion of Measurement Model Optimization via Importance Sampling](https://arxiv.org/abs/2303.00216)（Akai, 2023 预印本）用 scan matching 优化结果构造 proposal distribution，再通过 importance sampling 与 PF 融合。论文显示约 1,000 粒子也能工作，并可在单 CPU 线程运行。思路很适合本项目，但没有像 MOLA 那样成熟的 ROS 2 集成。
- [3D Monte Carlo Localization with Efficient Distance Field Representation](https://doi.org/10.1109/IV47402.2020.9304679)（Akai et al., IV 2020）用稀疏三维距离场加速 likelihood-field，并显式区分地图内/地图外观测以抵抗动态物体。它适合指导自研观测模型。

### 4.2 地点检索 + 局部配准

这是当前全局 LiDAR 定位文献最主流、也最适合现有系统的 coarse-to-fine 结构。综述 [A Survey on Global LiDAR Localization](https://arxiv.org/abs/2302.07433) 将完整流程概括为：先在关键帧地图中做 place retrieval，再在候选局部地图中做 metric pose estimation。

#### Scan Context / Scan Context++

[Scan Context](https://github.com/gisbi-kim/scancontext) 把 3D 点云投影为极坐标高度矩阵，通过 ring key 快速检索候选，并通过循环平移估计 yaw。它专门面向户外稀疏、含噪 LiDAR，计算量小，已经被大量 LIO/SLAM 项目采用。Scan Context++（Kim et al., IEEE T-RO 2021）进一步改善旋转和横向位移鲁棒性。

对 XT16 的适配性：

- 360° 旋转雷达非常匹配其表示方式；
- 不需要训练，对 Jetson 友好；
- 单帧 16 线过稀时，使用已去畸变的短时局部子图生成描述子；
- 输出主要是候选地点与 yaw，仍需 KISS-Matcher/GICP/NDT 求精确平移和 6-DoF 位姿。

它是推荐的第一版检索器。

#### STD：Stable Triangle Descriptor

[`hku-mars/STD`](https://github.com/hku-mars/STD)（Yuan et al., ICRA 2023）从点云提取稳定关键点并组成刚体变换不变的三角形描述子，通过匹配和几何验证完成地点识别。官方示例覆盖 KITTI、Livox 小视场雷达，以及与 FAST-LIO2 的在线集成。

优点：

- 无需训练；
- 几何验证比纯全局描述子更能压制感知混淆；
- 能直接产生点对应和相对位姿线索；
- 已有 FAST-LIO2 集成示例，和现有栈的结构接近。

局限：

- 官方代码是 ROS 1，需要移植或只抽取 C++ 核心；
- GPLv2，README 说明商业使用需联系作者；
- 平面过多、稳定角点太少时会退化；单帧 XT16 可能需要局部子图。

它是推荐的第二检索器/几何验证器，可与 Scan Context 取交集或并集。

#### BoW3D

[`YungeCui/BoW3D`](https://github.com/YungeCui/BoW3D)（Cui et al., IEEE RA-L 2023）基于 LinK3D 局部特征构建词袋，能实时检测回环并恢复完整 6-DoF 相对位姿，也明确支持 LiDAR 重定位。优点是不只给地点；缺点是官方示例较老（Ubuntu 16.04、ROS Kinetic），需要较多工程移植。

#### RING++

[RING++](https://arxiv.org/abs/2210.05984)（Xu et al., IEEE T-RO）把 BEV、Radon 变换和傅里叶表示结合起来，对旋转和平移都构造不变表示，并用相关性求解 3-DoF 相对位姿。其特点是无学习、支持稀疏关键帧地图、对大视角差异更强。

它适合作为第二阶段研究候选，但主要解决平面 `x/y/yaw`，在 Go2 明显俯仰/横滚、楼梯或多层环境中仍需 IMU 校平与 6-DoF 精配准。

#### 学习式方法

MinkLoc3D、OverlapTransformer、EgoNN、RING# 等方法在公开数据集上具有很强的检索能力。其中 [`EgoNN`](https://github.com/jac99/Egonn) 能先用全局描述子检索，再用局部描述子 + RANSAC 求 6-DoF 位姿。

当前不建议把学习式方法作为第一版主线：

- 预训练模型主要基于 KITTI、MulRan、Apollo 等车载 64 线数据；
- XT16 + Go2 的安装高度、运动模式、室内外分布差异明显；
- 需要构造本地正负样本并训练/微调；
- Jetson 上还要处理 CUDA、MinkowskiEngine/TensorRT 版本和显存。

可在经典方案建立基线数据集后，再评估学习式检索是否真正降低感知混淆。

### 4.3 全局点云配准

全局配准不需要精确初值，但它要求两片点云有足够重叠。将单帧 XT16 直接与整个大地图配准，往往同时遭遇低重叠、密度差异和计算量问题；正确用法是先由地点检索缩小到若干局部子图。

#### KISS-Matcher

[`MIT-SPARK/KISS-Matcher`](https://github.com/MIT-SPARK/KISS-Matcher)（Lim et al., ICRA 2025）是当前很有吸引力的学习无关全局配准器，强调快速、鲁棒和较少调参，并提供 ROS 2 SLAM 示例。它继承 Quatro++、ROBIN、TEASER 一系列鲁棒对应剔除思想。

推荐用法：对检索得到的 Top-5 或 Top-10 关键帧局部子图分别全局配准，产生候选变换；随后用 small_gicp/GICP/NDT 精配准和评分。不要让它直接面对未经裁剪的全地图。

#### FPFH + RANSAC / TEASER++

经典组合易理解、依赖成熟：先降采样、提取 FPFH、建立对应，再由 RANSAC 或 [TEASER++](https://github.com/MIT-SPARK/TEASER-plusplus) 求鲁棒刚体变换。`hdl_global_localization` 已实现 `FPFH_RANSAC` 和 `FPFH_TEASER` 两种引擎。

它适合作为可解释基线，但低线束、平面占比高和动态物体会降低 FPFH 区分度。应使用短时局部子图、地面/机器人本体过滤、合理法向半径，并用精配准验证。

#### `hdl_global_localization` + `hdl_localization`

[`koide3/hdl_global_localization`](https://github.com/koide3/hdl_global_localization) 提供：

- 2D 栅格 Branch-and-Bound Search；
- FPFH + RANSAC；
- FPFH + TEASER++。

它能通过 [`hdl_localization`](https://github.com/koide3/hdl_localization) 的 `/relocalize` 服务重置 NDT/UKF 跟踪器，是一个完整、清晰的参考实现。主要问题是仅支持 ROS Melodic/Noetic；可先离线跑 bag 验证，再决定是否移植其中的全局定位节点。

### 4.4 2.5D 投影搜索

如果机器人主要在单层近似平面场地移动，可把 3D 点云投影成 2D 占据/高度图，再使用 Branch-and-Bound、Correlative Scan Matching 或 Nav2 AMCL 做全局搜索，之后用 3D GICP/NDT 精配准。这条路线通常是最快建立的强基线。

优点：搜索空间从 6-DoF 降为 `x/y/yaw`，速度和全局覆盖都明显改善。缺点是会丢失高度信息，在多楼层、坡道、楼梯、相似平面布局中容易混淆。

它适合实验室单层导航，但不应被描述为完整的 6-DoF 3D 全局定位。

### 4.5 已知 waypoint 到达验证：配准器如何选

| 方法 | 初值要求 | XT16 上的特点 | 本项目定位 |
|---|---|---|---|
| point-to-point ICP | 较好 | 实现简单，但对稀疏、密度不均点云和局部极值较敏感 | 仅作基线，不建议单独作为主方案 |
| point-to-plane ICP | 较好 | 平面场景收敛快，但依赖稳定法向，走廊中可能退化 | 可作 GICP 的轻量对照 |
| **GICP** | 较好到中等 | 同时建模两侧局部协方差，通常比基础 ICP 更适合稀疏 3D LiDAR | **有 FAST-LIO 初值时的首选** |
| NDT | 中等 | 体素概率表示对采样差异较稳，捕获范围可较大，但依赖体素尺度 | 首选对照或备选精配准器 |
| KISS-Matcher / FPFH+TEASER++ + GICP | 可无精确初值 | 先建立鲁棒粗对应，再精化；计算量更高 | 初值可能超出 GICP 捕获范围时的回退路径 |

所以“ICP 是否最合适”的准确回答是：**ICP 这一类局部配准方法最贴合该任务，但工程首选应是 GICP，而非最基础的 ICP。** 如果移动过程的 FAST-LIO 累计误差始终小于局部配准捕获范围，直接“里程计初值 + GICP”最简单；如果机器人可能被搬动、里程计失效或误差达到数米/大角度，则在 GICP 前增加全局粗配准。

不论使用哪种优化器，都不能把算法返回的 `converged=true` 当作到达结论。走廊的一面墙、地面或重复结构也可能产生数值收敛，最终必须检查位姿残差、有效重叠、对应点分布和退化性。

## 5. 开源方案工程比较

| 方案 | 无先验全局定位 | 输出自由度 | 地图 | ROS/平台 | XT16 适配 | 结论 |
|---|---:|---:|---|---|---|---|
| `mcl_3dl` | 是 | 6-DoF | 点云 | ROS 1 | 中；需改运动模型 | 最像 3D AMCL，适合基线 |
| MOLA/MRPT PF | 是，取决于初始化粒子范围 | 3D/6-DoF 可配 | `.mm` metric map | ROS 2、arm64 | 高；需强降采样 | ROS 2 PF 首选基线 |
| Range-MCL | 是 | `x/y/yaw` | 三角网格 | Python + OpenGL/GPU | 论文验证低线束 | 学术对照，冷启动慢 |
| Scan Context + GICP | 是 | 检索 + 3/6-DoF 精配准 | 关键帧 + 局部点云 | C++，易集成 | 高 | 最快可落地的主线 |
| STD + GICP | 是 | 候选 + 6-DoF 线索 | 关键帧/局部子图 | ROS 1/C++ | 高；建议累积子图 | 强几何验证方案 |
| BoW3D | 是 | 6-DoF | 关键帧数据库 | ROS 1 老环境 | 中到高 | 有能力，移植成本较高 |
| RING++ | 是 | 3-DoF | 稀疏 scan map | 研究代码 | 高，需姿态校平 | 平面场景研究候选 |
| KISS-Matcher | 需候选局部图 | 6-DoF | 两片重叠点云 | C++/ROS 2 示例 | 高 | 推荐的候选配准器 |
| `hdl_global_localization` | 是 | BBS 为 3-DoF，FPFH 为 6-DoF | PCD | ROS 1 | 高 | 很好的离线/移植基线 |
| `hdl_localization` / FAST-LIO localization | 否，需初值 | 6-DoF 跟踪 | PCD | ROS 1/各类 fork | 高 | 只负责精配准与跟踪 |
| EgoNN/MinkLoc3D | 是 | 检索；EgoNN 可 6-DoF | 学习特征库 | Python/CUDA | 未知，需本地训练验证 | 第二阶段研究 |

## 6. 对当前 DDDMR 栈的判断

当前配置中的相关参数为：

```yaml
mcl_3dl:
  init_x: 0.0
  init_y: 0.0
  init_z: 0.0
  init_var_x: 0.5
  init_var_y: 0.5
  init_var_z: 0.2
  init_var_yaw: 0.2
  num_particles: 60
  map_frame: map
  robot_frame: body
  odom_frame: camera_init
```

这组设置表达的是：机器人初始位姿已经在约半米、约 0.2 rad 的邻域内，60 个粒子用于局部概率跟踪。它无法覆盖几十米地图和全 yaw，也不可能在多个相似地点之间保留足够的假设。

因此现有 `mcl_feature/mcl_3dl` 不应进入本项目的离线到达判定链路。FAST-LIO 只需提供：

- 原始扫描的去畸变结果；
- 构建 0.5~1.5 s 查询子图所需的帧间位姿；
- 从移动终点到目标 waypoint 附近的配准初值；
- 可选的里程计协方差，用于决定是否启用粗配准回退。

离线验证器读取保存的点云和位姿信息，输出到达判决及相对残差，不发布 TF，也不重置 MCL。若未来另行增加在线定位功能，再单独设计 TF 权威发布者和状态机。

## 7. 推荐实现方案

### 7.1 明确需求：已知 `waypoint_id` 的到达验证

数据库保存一组带标签的 waypoint 场景。调用时已经给定目标 `waypoint_id = w*`，系统只加载该 waypoint 的参考模板，不需要搜索整个地图：

- **起点 waypoint**：例如 `waypoint_id=start` 或 `waypoint_id=0`；
- **多个任务 waypoint**：沿任务区域预先定义的离散目标位置；
- **每个 waypoint 的一个或多个参考模板**：覆盖不同朝向、采集时刻或局部可见范围；
- **目标位姿容差**：每个 waypoint 可单独定义允许的平移和朝向误差。

给定查询点云 `Q` 和目标 waypoint `w*`，把 `Q` 与该 waypoint 下的模板 `D_{w*,j}` 配准。若模板保存在各自的采集传感器坐标系中，配准先得到：

\[
{}^{template_j}T_{query}
=
\operatorname{Register}(Q,D_{w^*,j}).
\]

再利用采集时保存的模板相对 waypoint 标定变换，统一换算为：

\[
{}^{waypoint}T_{query,j}
=
{}^{waypoint}T_{template_j}\;{}^{template_j}T_{query}.
\]

如果模板点已经预变换到统一的 waypoint 坐标系，则 `T_waypoint_template_j` 为单位变换。只有完成这一步后，机器人准确到达时的最佳有效变换才应接近单位变换。首版可按下式判定：

\[
e_{xy}=\sqrt{t_x^2+t_y^2},\qquad
e_{yaw}=|\operatorname{yaw}({}^{waypoint}T_{query})|.
\]

```text
ARRIVED = registration_valid
       && e_xy  < waypoint.xy_tolerance
       && e_yaw < waypoint.yaw_tolerance
       && overlap > overlap_min
       && inlier_ratio > inlier_min
       && rmse < rmse_max
```

如果 waypoint 只表示位置、不约束朝向，可以不使用 `e_yaw`；如果要求完整落脚姿态，则同时检查 `z/roll/pitch`。阈值必须结合实际导航精度、XT16 点云噪声和地图体素离线标定，不能使用一套未经验证的通用常数。

系统的主要输出是：

```text
target_waypoint_id
decision: ARRIVED | NOT_ARRIVED | UNCERTAIN | INVALID_QUERY
relative_pose_to_waypoint
confidence
matched_scene_id
registration_metrics
```

完整离线验证流程为：

```text
数据库构建阶段
  起点 + 各任务 waypoint
              │
              ▼
  为每个 waypoint 保存一个或多个参考点云模板和容差
              │
              ▼
       记录 waypoint/template 元数据

离线到达验证
  target_waypoint_id + query.pcd/rosbag
              │
              ▼
  加载目标 waypoint 的参考模板
              │
              ▼
  里程计初值；必要时做全局粗配准
              │
              ▼
  GICP/NDT 精配准，计算相对 waypoint 的残差
              │
              ▼
  ARRIVED / NOT_ARRIVED / UNCERTAIN
```

这是一个“一对多模板验证”问题：一个 query 只与已知 waypoint 下的多个模板比较，不需要输出 Top-K waypoint。为防止重复走廊产生假阳性，可以额外保存相邻 waypoint 作为 hard negatives；它们仅用于校准或歧义检查，不改变“目标 waypoint 已知”的任务定义。

### 7.2 场景锚点的采集与选择

局部场景不应仅按固定距离机械保存。推荐同时使用以下触发条件：

- 从上一锚点移动 1~2 m；
- yaw 改变 10°~20°；
- 经过路口、门口、转角、坡道起止点；
- 点云结构或 Scan Context 描述子相对上一锚点变化明显；
- 当前区域对邻近场景具有足够区分度。

以下位置不宜单独作为高置信度锚点：

- 长走廊中部等高度重复区域；
- 只有地面、单面墙或大面积玻璃的位置；
- 长期被车辆、人员、移动货物占据的位置；
- 雷达被遮挡或点数明显不足的位置。

对重复走廊仍可保存模板，但应把相邻多帧组成局部子图，或者增加转角、门洞、立柱等上下文。同一 waypoint 的模板数量也不是越多越好；应覆盖主要观察条件，同时删除几乎重复或结构退化的模板。

每个局部场景建议由已去畸变的 5~15 帧点云累积而成，覆盖约 0.5~1.5 s。累积点云应：

- 通过 FAST-LIO 相对位姿变换到锚点坐标系；
- 删除机器人本体和近距离反射噪声；
- 删除 NaN/Inf；
- 保留一份较密点云用于精配准；
- 另存一份降采样点云用于描述子和粗配准；
- 可选地保存静态点掩码或动态点比例。

### 7.3 建议的数据格式

推荐把起点和所有局部场景统一组织为 `waypoints`，每个 waypoint 下允许多个模板：

```text
relocalization_map/
├── map.yaml
├── global_map.pcd
├── scenes.csv
├── descriptor_index.bin
└── waypoints/
    ├── waypoint_000_start/
    │   ├── waypoint.yaml
    │   ├── scene_0001/
    │   │   ├── cloud_dense.pcd
    │   │   ├── cloud_coarse.pcd
    │   │   ├── descriptor.bin
    │   │   ├── pose.yaml
    │   │   └── quality.yaml
    │   └── scene_0002/
    │       └── ...
    ├── waypoint_001/
    │   ├── waypoint.yaml
    │   ├── scene_0001/
    │   └── scene_0002/
    └── waypoint_002/
        └── ...
```

`scenes.csv` 至少包含：

```text
waypoint_id,scene_id,type,timestamp,x,y,z,qx,qy,qz,qw,cloud_dense,cloud_coarse,descriptor
waypoint_000,scene_0001,start,...
waypoint_000,scene_0002,start,...
waypoint_001,scene_0001,local_scene,...
```

每个 `waypoint.yaml` 至少定义到达容差：

```yaml
waypoint_id: waypoint_001
position_only: false
arrival_tolerance:
  xy_m: 0.40
  z_m: 0.25
  yaw_deg: 8.0
  roll_deg: 8.0
  pitch_deg: 8.0
```

以上数字只是初始示例，不能直接作为最终验收阈值。

`pose.yaml` 建议明确记录：

```yaml
waypoint_id: waypoint_001
scene_id: scene_0001
cloud_frame: waypoint_001_scene_0001
waypoint_frame: waypoint_001
global_frame: map
T_waypoint_scene:
  translation: [dx, dy, dz]
  quaternion_xyzw: [dqx, dqy, dqz, dqw]
T_map_scene:
  translation: [x, y, z]
  quaternion_xyzw: [qx, qy, qz, qw]
lidar_extrinsic_version: xt16_go2_v1
map_version: 2026_09_xx
```

`T_waypoint_scene` 是多模板计算统一 waypoint 残差所必需的；`T_map_scene` 仅用于地图可视化，可以省略。`quality.yaml` 可记录点数、覆盖半径、有效环数、动态点比例、平面退化评分、描述子最近邻间隔等信息，便于剔除低质量场景。

### 7.4 匹配器分层

调用方已经给出目标 `waypoint_id`，因此只对该 waypoint 的模板做一对多验证。建议分三层：

1. **初值检查**：读取 FAST-LIO/导航保存的终点位姿及协方差，计算 query 到各模板的初始相对变换；
2. **条件式粗配准**：初值可靠时跳过；初值缺失、协方差过大或 GICP 首次失败时，用 KISS-Matcher 或 FPFH + TEASER++ 计算 `T_scene_query`；
3. **精配准与验收**：对目标 waypoint 的各模板用 GICP/NDT 精化，选择质量最好的有效结果，并综合位姿残差、overlap、inlier ratio、RMSE 和退化程度输出判决。

不能只用“最小 ICP fitness”决定位置。至少要求：

- 最佳目标模板通过绝对质量门限；
- 配准后的相对目标位姿在该 waypoint 的到达容差内；
- 如果配置了相邻 waypoint hard negatives，目标模板得分应明显优于负样本；
- 如果 query 来自一段 rosbag，多个时间窗口的残差与判决相互一致；
- 配准后的重叠分布覆盖多个方向，而不是只对齐一面墙或一块地面；
- 得到的位姿变化符合机器人可能的运动范围。

### 7.5 场景保存与离线查询接口

建议提供两个 ROS 2 服务或 action：

```text
/relocalization/save_anchor
  waypoint_id: waypoint_000 | waypoint_xxx
  scene_id: scene_xxxxxx
  type: start | local_scene
  accumulate_duration: 1.0

/relocalization/build_index
  rebuild_descriptors: true
```

保存流程应冻结一个锚点参考时刻，累积期间利用 FAST-LIO 相对位姿将点云变换到该时刻，然后一次性写入临时目录；所有 PCD、位姿和描述子写完并校验后，再原子改名为正式锚点目录，避免断电留下半个场景。

还应支持自动采集模式：建图时按照距离、转角和描述子变化触发候选锚点，结束后离线去重和质量筛选；起点则显式命名为 `start` 并强制保留。

离线验证建议提供独立 CLI，不依赖 ROS 图持续运行，并强制传入目标 waypoint：

```bash
waypoint_arrival_verifier \
  --database /path/to/relocalization_map \
  --target-waypoint waypoint_017 \
  --query /path/to/query.pcd \
  --initial-pose /path/to/terminal_odom.yaml \
  --output /path/to/result.json
```

建议结果格式：

```json
{
  "decision": "ARRIVED",
  "target_waypoint_id": "waypoint_017",
  "confidence": 0.91,
  "matched_scene_id": "scene_0003",
  "relative_pose_xyz_rpy": [0.10, -0.20, 0.02, 0.01, -0.02, 0.03],
  "metrics": {
    "overlap": 0.72,
    "inlier_ratio": 0.68,
    "rmse_m": 0.09,
    "degenerate": false
  },
  "threshold_profile": "waypoint_017_v1"
}
```

### 7.6 离线索引生成与版本管理

场景锚点建议从原始建图 bag 和优化后的轨迹生成。仅有合并 PCD 时无法可靠恢复“从哪个观测位姿看到什么”，会削弱 Scan Context 和其他传感器中心描述子。

地图处理完成后应执行一次离线构建：

1. 根据优化轨迹把各帧转换到候选锚点坐标系；
2. 生成 `cloud_dense.pcd` 和 `cloud_coarse.pcd`；
3. 可选计算 Scan Context/STD 描述子，用于目标相似度检查和 hard-negative 分析；
4. 删除同一 waypoint 内近似重复的模板；
5. 标记结构退化、动态点过多或易与相邻 waypoint 混淆的模板；
6. 计算文件校验和；
7. 将模板、全局地图、外参和阈值配置绑定到同一个 `map_version`。

地图或外参改变后必须重建索引，不能混用旧锚点。起点和局部场景点云也必须来自同一全局坐标系版本。

首轮参数起点：

- 关键帧距离：1.0~2.0 m；
- 关键帧角度：10°~20°；
- 单个局部子图半径：15~30 m，室外可更大；
- 描述子输入体素：0.2~0.4 m；
- 配准输入体素：粗配准 0.4~0.8 m，精配准 0.15~0.3 m；
- 去除机器人本体、近距离噪声、远距离稀疏点和明显动态点。

这些是启动值，不是通用最优值，应由本地 bag 消融确定。

### 7.7 离线查询与判决流程

```text
LOAD_QUERY
     │ 点数、字段和时间跨度合格
     ▼
PREPROCESS ──数据不足──> INVALID_QUERY
     │
     ▼
LOAD_TARGET_TEMPLATES(target_waypoint_id)
     │
     ▼
INITIALIZE_FROM_ODOMETRY
     │ 初值不可靠时启用粗配准
     │
     ▼
GICP/NDT_EACH_TARGET_TEMPLATE
     ▼
DECISION
  ├── ARRIVED       -> 残差在容差内且配准可信
  ├── NOT_ARRIVED   -> 配准可信，但残差超出容差
  └── UNCERTAIN     -> 无可信配准或目标/负样本难区分
```

每个阶段的建议：

1. **加载查询**：支持单个 PCD、PCD 序列或 rosbag 时间段。
2. **点云准备**：优先使用 `/cloud_registered_body` 或等价的去畸变 body-frame 点云，过滤 NaN/Inf 和机器人本体。
3. **短时累积**：累积 5~15 帧或 0.5~1.5 s；有运动时按 FAST-LIO 相对位姿对齐每帧。
4. **加载目标**：只加载命令行指定 waypoint 的模板及容差配置。
5. **配准**：初值可靠时直接 GICP/NDT；初值不可靠或失败时，对目标模板运行 KISS-Matcher/TEASER++ 后再精化。
6. **模板选择**：在通过几何有效性检查的结果中选择质量最好者；不要仅按原始 RMSE 排序。
7. **到达判决**：先判断配准是否可信，再判断相对位姿残差是否在 waypoint 容差内。
8. **保存结果**：输出 JSON/CSV，并可选保存对齐点云用于人工复核。

### 7.8 描述子的可选作用

已知目标 ID 后，Scan Context/STD 不再负责全库检索。它们可以作为低成本辅助信号：

- 检查 query 是否在外观上类似目标 waypoint，提前拒绝明显错误场景；
- 估计 yaw 初值或为粗配准提供对应；
- 离线挖掘与目标高度相似的相邻 waypoint，形成 hard-negative 集；
- 分析某个 waypoint 是否先天缺少可辨识结构。

首版可以完全不使用描述子，先建立“里程计初值 + GICP + 几何门限”基线，再根据误接受样本决定是否加入。

### 7.9 离线置信度与拒识判据

不能只看描述子距离或一个 ICP fitness。建议联合：

- 描述子最佳匹配的绝对相似度；
- 最佳目标模板的绝对配准质量；
- 可选的目标模板与 hard-negative 模板分数间隔；
- 有效对应点比例和重叠率；
- GICP/NDT 目标函数；
- 优化 Hessian 的最小特征值或 condition number；
- 对齐对应是否覆盖多个方位和高度层；
- 同一查询被切成多个时间窗口时，残差和判决是否一致；
- 查询点数、有效环数、空间覆盖范围和动态点比例。

判决逻辑应允许拒识：

- 配准可信且位姿残差在容差内：`ARRIVED`；
- 配准可信但位姿残差超出容差：`NOT_ARRIVED`；
- 配准不可信、场景退化或目标与 hard negative 无法区分：`UNCERTAIN`；
- 查询自身点数或覆盖不足：`INVALID_QUERY`。

## 8. 建议的分阶段验证计划

### 阶段 A：建立带真值的数据集

1. 每个 waypoint 保存多个参考模板，记录目标 body/sensor 坐标系和到达容差。
2. 在另一次独立巡检中采集 query，包含“准确到达”“接近但未到达”“相邻错误 waypoint”和“点云质量不足”四类。
3. 为 query 获取独立真值，例如全站仪、AprilTag、人工精确测量或高质量离线 SLAM 轨迹；不能用待评估的 ICP 输出自身作为真值。
4. 按巡检批次/日期划分训练与测试，不能把同一段轨迹的相邻帧随机拆到两边，否则会产生严重数据泄漏。
5. 单独标注长走廊、相似房间、开阔区、玻璃、坡道和人员密集区。

### 阶段 B：配准器与初值消融

固定相同预处理和体素尺度，对比：

- point-to-point ICP、point-to-plane ICP、GICP、NDT；
- 真值附近初值、FAST-LIO 实际初值、无初值三种条件；
- GICP 直接精配准与 KISS-Matcher/TEASER++ + GICP；
- 单帧与 5/10/15 帧去畸变局部子图；
- 体素 0.2/0.4/0.6 m、去地面与保留地面。

不要只报告平均 RMSE；应画出不同初始平移/偏航误差下的收敛成功率，从而明确何时可以直接 GICP、何时必须启用粗配准。

### 阶段 C：到达判决标定

| 指标 | 定义与用途 |
|---|---|
| False Accept Rate（FAR） | 实际未到达却输出 `ARRIVED` 的比例；安全上最重要 |
| False Reject Rate（FRR） | 实际到达却输出 `NOT_ARRIVED` 或 `UNCERTAIN` 的比例 |
| Precision / Recall | 以 `ARRIVED` 为正类评估二值判决 |
| 位姿残差误差 | 配准估计的 `x/y/z/roll/pitch/yaw` 与独立真值之差 |
| Uncertain rate | 系统主动拒绝给出结论的比例 |
| 延迟与资源 | 单次查询 p50/p95 耗时、峰值内存、CPU/GPU 占用 |

先在标定集上联合选择 `xy/yaw` 容差和 overlap/inlier/RMSE/退化门限，再冻结参数到独立测试集。总体 FAR 之外还要逐 waypoint 报告结果，因为某些重复场景会被总体平均值掩盖。实际系统应优先压低 FAR：宁可输出 `UNCERTAIN`，也不能把尚未到达误判为到达。

### 阶段 D：鲁棒性与 hard-negative 测试

至少覆盖：

- 在目标前后左右按 0.2~0.5 m 间隔采集的边界样本；
- 朝向满足/不满足条件的样本；
- 相邻 waypoint 和外观高度相似 waypoint；
- 白天/夜间、动态人员、家具变化和部分遮挡；
- 原始 XT16 点云与 FAST-LIO 去畸变点云；
- 里程计正常、明显漂移和完全缺失；
- 单层、坡道与楼梯区域。

## 9. 建议实施顺序

### P0：最小可用基线

- 先选 5~10 个 waypoint，每个保存 2~4 个参考模板；
- 用 FAST-LIO 去畸变并累积终点附近 0.5~1.5 s 点云；
- 实现 `--target-waypoint`、`--query`、`--initial-pose` 输入；
- 使用里程计初值 + GICP，输出相对位姿、overlap、inlier ratio、RMSE 和对齐点云；
- 人工检查正确到达、未到达和相邻 waypoint 样本，不发布 TF、不控制机器人。

### P1：形成可靠判决

- 加入每个 waypoint 独立的位姿容差；
- 加入退化检测、空间覆盖检查和 `INVALID_QUERY`；
- 用独立标定/测试集确定 `ARRIVED / NOT_ARRIVED / UNCERTAIN` 门限；
- 重点统计 FAR、FRR、逐 waypoint 指标和 p95 延迟；
- 保存算法版本、地图版本、阈值版本及完整判决日志。

### P2：处理大初值误差和场景混淆

- GICP 失败或里程计协方差过大时，启用 KISS-Matcher/TEASER++ 粗配准回退；
- 保存相邻或相似 waypoint 作为 hard negatives；
- 可选加入 Scan Context/STD 目标相似度检查；
- 使用多个时间窗口的一致性压制偶然误配。

### P3：按数据决定的增强项

- 若 NDT 在本地场景的捕获范围或稳定性明显优于 GICP，将其作为主备切换方案；
- 若环境长期变化明显，研究静态点筛选、模板更新和模板版本管理；
- 只有经典方法仍无法区分 hard negatives 时，再评估本地训练的学习式描述子。

## 10. 论文与项目索引

| 工作 | 年份/状态 | 核心方法 | 与本项目的关系 |
|---|---|---|---|
| [A Survey on Global LiDAR Localization](https://arxiv.org/abs/2302.07433) | 2023 预印本，后有期刊版本 | 全局检索、位姿估计、序列定位综述 | 方法分类依据 |
| [Range Image-based LiDAR Localization](https://www.ipb.uni-bonn.de/wp-content/papercite-data/pdf/chen2021icra.pdf) | ICRA 2021 | mesh 渲染 range image + MCL | 低线束、跨传感器 MCL 对照 |
| [Learning an Overlap-based Observation Model](https://arxiv.org/abs/2105.11717) | IROS 2020 | OverlapNet + MCL | 学习式观测模型 |
| [3D MCL with Efficient Distance Field](https://doi.org/10.1109/IV47402.2020.9304679) | IV 2020 | 稀疏 3D 距离场 + 动态观测分类 | 自研 PF 观测模型参考 |
| [Efficient 3D LiDAR MCL via Importance Sampling](https://arxiv.org/abs/2303.00216) | 2023 预印本 | scan matching proposal + PF | 降低粒子数的好方向 |
| [Scan Context](https://github.com/gisbi-kim/scancontext) | IROS 2018 | 极坐标高度描述子 | 首选轻量检索器 |
| [Scan Context++](https://arxiv.org/abs/2109.13494) | T-RO 2021 | 旋转/横移更鲁棒的 Scan Context | 检索增强 |
| [STD](https://github.com/hku-mars/STD) | ICRA 2023 | 稳定三角描述子 + 几何验证 | 首选第二检索/验证器 |
| [BoW3D](https://github.com/YungeCui/BoW3D) | RA-L 2023 | LinK3D 词袋 + 6-DoF | 可替代的全流程前端 |
| [RING++](https://arxiv.org/abs/2210.05984) | T-RO | 旋转平移不变 BEV 表示 | 平面稀疏地图候选 |
| [EgoNN](https://github.com/jac99/Egonn) | RA-L 2022 | 全局/局部学习描述子 + RANSAC | 后续学习式 6-DoF 方案 |
| [KISS-Matcher](https://github.com/MIT-SPARK/KISS-Matcher) | ICRA 2025 | 鲁棒全局点云配准 | 推荐粗配准器 |
| [TEASER++](https://github.com/MIT-SPARK/TEASER-plusplus) | T-RO 2020 | 可认证鲁棒配准 | FPFH 对应后的基线 |
| [`mcl_3dl`](https://github.com/at-wat/mcl_3dl) | 开源软件 | 6-DoF 点云 MCL | 最像 3D AMCL 的参考 |
| [MOLA/MRPT PF localization](https://docs.mola-slam.org/latest/localization.html) | 持续维护 | ROS 2 3D PF/LO/LIO | ROS 2 粒子滤波基线 |
| [`hdl_global_localization`](https://github.com/koide3/hdl_global_localization) | 开源软件 | BBS、FPFH+RANSAC/TEASER | 全局初始化参考实现 |
| [`hdl_localization`](https://github.com/koide3/hdl_localization) | 开源软件 | NDT + UKF 跟踪 | 说明全局初始化与跟踪应分离 |

## 11. 最终建议

建议把项目名称和边界定义为：

> **基于 XT16 局部点云配准的 waypoint 离线到达验证器**

首版技术栈建议固定为：

- FAST-LIO：点云去畸变、短时局部子图和终点里程计初值；
- **GICP：主精配准器**；NDT 作为对照和备选；
- KISS-Matcher 或 FPFH+TEASER++：仅在初值不可靠或 GICP 失败时启用；
- 每 waypoint 多模板及独立到达容差；
- 位姿残差 + overlap + inlier ratio + RMSE + 退化程度联合判决；
- 输出 `ARRIVED / NOT_ARRIVED / UNCERTAIN / INVALID_QUERY` 和相对目标 waypoint 的位姿残差。

因此，**不是“只跑一次 ICP，看 fitness 是否够小”**。最稳妥的首版是“已知目标模板 + FAST-LIO 初值 + GICP + 几何有效性门限”，并用相邻 waypoint、目标边界附近样本和跨日期数据严格标定误接受率。MCL、全库地点检索、在线 TF 重置都不属于当前任务的必要范围。
