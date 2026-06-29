# CuVSLAM-Autonomy 快速使用

本文档旨在说明 CuVSLAM-Autonomy 项目的使用方法、功能和架构设计。该项目仅保证在 **实验室 #3 机器狗** 上能稳定运行，其他平台请自行适配。

> 文档写于 2026.6.29，请注意时效性

## 简单说明

项目分为两个部分：
1. **cuVSLAM**：基于 ROS2 的视觉-imu SLAM 系统，仅提供稀疏点云地图和位姿，以及视觉重定位。传感器为 Realsense D435i，使用其 RGB 图像和 IMU 数据进行 SLAM。
2. **Autonomy**：基于 SLAM（这里是 cuVSLAM） 的自主导航系统，提供局部导航、全局规划、避障等功能。传感器为机器狗自带的 L1 雷达，使用其点云数据进行地形建图和避障。

整体上，简单来说是 Autonomy 使用 cuVSLAM 提供的稳定位姿信息进行自主导航，其建图使用的是 Autonomy 自带的地形建图模块，而不是 cuVSLAM 的稀疏点云地图。另外，Autonomy 还用到了 cuVSLAM 的视觉重定位功能，来辅助机器狗在必要的时候恢复正确位姿。

## 使用说明

### cuVSLAM

> 项目托管在 [cuVSLAM](https://git.modeloverfit.com/Embodied_AI/cuvslam_go2) 仓库 cuVSLAM 分支中. **详细文档可参考根目录下的 `cuvslam.md`**

#### 构建 Docker 镜像并运行例程

cuVSLAM 运行在官方提供的 Docker 容器中，构建方法具体可参考 `docker/README.md`。

主要需要注意的是实验室 #3 机器狗的系统环境为 Ubuntu 22.04 + Jetpacks 6，在 README 中需要运行对应的构建脚本。

构建命令如下：

```bash

# Ubuntu 22.04 + CUDA 12

docker build -f docker/Dockerfile.realsense-cu12 -t pycuvslam:realsense-cu12 .

```

构建完成后，不要运行官方的 `docker/run_docker.sh`，而是使用 `docker/run_docker_fixed.sh`。该脚本主要修复了官方容器中 cuda 架构与机器狗系统环境不兼容的报错问题，以及官方脚本中对 librealsense 库的覆盖问题，保证 cuVSLAM 能够正确使用 D435i 的 imu。

```bash

./docker/run_docker_fixed.sh

```

成功进入容器后，运行以下命令启动 cuVSLAM 的例程：

```bash

python3 examples/realsense/run_stereo.py

```

若这一步出问题，主要注意以下三点：

1. 确认容器中 Cuda 版本与机器狗系统环境的 Cuda 版本兼容，机器狗需要用 L4T 版本的 Cuda 12.2，官方 build 提供的是 `nvidia/cuda:12.2.0-devel-ubuntu22.04` 基础镜像。
2. 确认容器中 librealsense 库版本与 D435i 兼容，官方提供的 librealsense 版本会让 D435i 的 imu 无法启用，需要使用 librealsense 2.57.6。
3. 确保 Realsense D435i 成功连接到机器狗的 USB 端口，成功加装驱动，并且在容器中能够被识别。

#### 外围脚本使用

成功构建并运行 cuVSLAM 的 Docker 容器后，使用 `docker/start_realsense_slam_map.sh` 脚本启动 cuVSLAM。

**详细参数可以参考项目根目录的`cuvslam.md`**，通常情况下，使用以下命令进行 cuVSLAM 的建图功能：

```bash

./docker/start_realsense_slam_map.sh --preset robot

# 建图结束后运行以下命令

/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --stop

```

> 第一次使用外围脚本会报一个关于 deamon 的错误，忽略即可，从第二次使用开始就没有报错了。

该模式下，cuVSLAM 会默认启用 60 帧的双目图像和 IMU 数据进行 SLAM，生成稀疏点云地图和日志，保存在 `~/datasets/cuvslam_maps` 下.

建图结束后，要查看建图效果，需要在本地 PC（非机器狗）上安装 rerun-cli 0.22.1（版本不同通信协议不同，无法使用），然后运行以下命令启动 rerun：

```bash

rerun --bind 0.0.0.0 --port 9876

```

之后在机器狗上启动外围脚本（一般在 stop 后会有终端提示）：

```bash

/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --view-output-dir <本次输出目录>

```

> 这里需要注意的是，外围脚本中关于 viewer 的 ip 地址是写死的（192.168.31.223:9876），需要修改成指定的 ip

**重定位部分**，指定 `--input-map-dir` 后自动启用重定位功能，重定位会使用指定目录的视觉特征稀疏点云来进行视觉重定位。在刚启动时，cuVSLAM 会尝试进行一次初始重定位，将当前相机位姿与指定地图对齐，之后在 SLAM 过程中会在某些条件下，如位姿严重偏移时进行视觉重定位，保证位姿的稳定性。

重定位下，如果指定 `--localize-only`，则只进行重定位，不进行 SLAM 建图，新得到的文件夹不会保存 `map.mdb` 数据。

### Autonomy

> 项目托管在 [cuVSLAM-Autonomy](https://git.modeloverfit.com/Embodied_AI/autonomy_stack_go2/-/tree/cuvslam-autonomy) 仓库 cuvslam-autonomy 分支中.
> **详情可参考 cuvslam_cruise_stack_overview.md 和 cuvslam-autonomy_20260529.md**

Autonomy 部分主要基于 cuVSLAM 提供的位姿信息进行自主导航，使用机器狗自带的 L1 雷达进行地形建图和避障。

Autonomy 的使用主要有四个重要脚本：

1. `system_cuvslam_cruise_headless.sh`: 默认在无头模式（不在机器狗上运行可视化）运行局部地图可视化与局部规划。
2. `system_cuvslam_cruise_route_planner_headless.sh`: 默认在无头模式运行全局路径规划 + 局部规划，并查看全局地图。
3. `system_cuvslam_cruise_local_viewer.sh`: 用于在本地 PC 上实时查看可视化结果。
4. `manual_mapping_headless.sh`: 手动建图脚本，使用手柄控制机器狗进行建图，建图保存为 ply 文件，在 `~/datasets/cuvslam_maps` 下，和 cuVSLAM 的建图结果兼容。

#### 可视化配置

首先要在本地 PC 上安装 rviz，然后运行脚本 `system_cuvslam_cruise_local_viewer.sh`，在本地 PC 上查看可视化结果。

运行该脚本之前，需要启动机器狗上的无头脚本，以便在本地 PC 上接收相关的 ROS2 topic 数据。

> 注意事项：机器狗和本机需要在同一局域网下，指定正确的 dds 配置及网卡，设置 domain id 为 77

#### 局部导航栈

启用局部导航栈直接运行 `system_cuvslam_cruise_headless.sh`，该脚本会启动 cuVSLAM + autonomy_stack_go2 的局部导航栈，使用机器狗自带的 L1 雷达进行地形建图和避障。

如果需要开启重定位功能，需要按如下格式运行：

```bash

CUVSLAM_MAP_DIR=~/datasets/cuvslam_maps/<指定地图目录> ./docker/system_cuvslam_cruise_headless.sh

```

而后在本地 PC 上运行 `system_cuvslam_cruise_local_viewer.sh`，即可查看局部导航栈的可视化结果.

在 rviz 上，可以看到 TF 可视化组成的机器狗位姿、`\terrain_map` 可视化的局部地形地图. 可以通过快捷键 w 指定目标点 waypoint，机器狗会自动规划路径并跟踪，跟踪过程中，可以选择开启其他可视化，如 `\path`、`\free_paths`等，查看目前机器狗的规划方向和候选规划方向.

#### 全局规划栈

启用全局规划栈运行 `system_cuvslam_cruise_route_planner_headless.sh`，该脚本会启动 cuVSLAM + autonomy_stack_go2 的全局规划栈，在使用机器狗自带的 L1 雷达进行地形建图和避障的基础上，增加了 far_planner 的全局路径规划功能。far_planner 会在接收到目标点后，使用 ply 点云进行全局路径规划。

全局规划必须指定带有 ply 文件的 `CUVSLAM_MAP_DIR`，否则会出现错误。在这之前，可以使用 `manual_mapping_headless.sh` 脚本进行手动建图，生成带 ply 文件的 cuVSLAM 目录。

运行 `manual_mapping_headless.sh` 后，不需要开启可视化（该功能暂无），使用手柄控制机器狗进行建图，建图结束后 ply 文件会保存在 `~/datasets/cuvslam_maps/<指定目录>/global_map.ply`。

然后可以运行如下命令启动全局规划栈：

```bash

CUVSLAM_MAP_DIR=~/datasets/cuvslam_maps/<指定地图目录> ./docker/system_cuvslam_cruise_route_planner_headless.sh

```

在此之后，使用 `system_cuvslam_cruise_local_viewer.sh` 脚本在本地 PC 上查看可视化结果。

rviz 上可以看到比起局部导航栈多了白色点云构成的全局地图，g 键可以指定 far_planner 目标点 goal point，以及规划路径的可视化。

## 目前存在的问题和局限

1. 该项目目前仅在实验室 #3 机器狗上测试过，其他平台不保证适配；
2. 重定位功能不完全稳定，在某些情况下会出现重定位失败，导致机器狗的错误位姿被固定；
3. 局部规划和全局规划的参数尚未完全调优，目前会在狭窄通道中出现不停震荡无法前进的现象；
4. 在机器狗被碰撞以及频繁切换方向的情况下，位姿可能会出现较大偏差。
