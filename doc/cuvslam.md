# cuVSLAM 使用说明

## 概览

当前已经打通的能力：

- Jetson 上通过 Docker 运行 `cuVSLAM`
- 修复了特定 USB 口的 RealSense 供电/识别问题
- 支持将 `Rerun` 可视化发送到本机，而不是在 Jetson 本地出图
- 支持两种建图模式：
  - `stereo`
  - `rgbd`
- 支持两种预设：
  - `quality`
  - `fast`
  - `robot`

当前统一入口脚本：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh
```

## 相关文件

- 包装脚本：
  `/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh`
- 双目建图脚本：
  `/home/unitree/cuVSLAM/examples/realsense/run_stereo_slam_map.py`
- RGBD 建图脚本：
  `/home/unitree/cuVSLAM/examples/realsense/run_rgbd_slam_map.py`
- 可视化：
  `/home/unitree/cuVSLAM/examples/realsense/visualizer.py`

## 本机 Viewer

本机 `rerun` 当前应使用 `0.22.1`。

启动方式：

```bash
/home/jycao/.cargo/bin/rerun --bind 0.0.0.0 --port 9876
```

如果通过 SSH 从本机连 Jetson，包装脚本会自动把 viewer 地址识别为当前 SSH 客户端 IP 加 `9876`。

## 包装脚本行为

`start_realsense_slam_map.sh` 会自动做这些事：

- 检查 `realsense-usb-power.service`
- 等待宿主机 `lsusb` 识别 RealSense
- 启动或复用 `pycuvslam:realsense-cu12` 容器
- 清理容器里旧的 `run_stereo*` / `run_rgbd*` 进程
- 检查容器内 `pyrealsense2` 是否能访问相机
- 自动探测 viewer 地址
- 后台启动唯一一个建图进程

## 常用命令

查看状态：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --status
```

停止当前建图：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --stop
```

### 双目建图

质量优先：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --preset quality
```

速度优先：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --preset fast
```

机器狗运动优化：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --preset robot
```

### RGBD 建图

质量优先：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode rgbd --preset quality
```

速度优先：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode rgbd --preset fast
```

不建议默认直接使用 `robot` preset。`robot` 主要是针对高角速度下的双目视觉跟踪做的保守优化，RGBD 下不一定有同样收益，实际测试中可能效果更差。

### Headless 运行

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --no-viewer
```

### 无线链路下不发送图像

如果 Jetson 通过无线网络连接，本机 `Rerun` 延迟很高，建议关闭图像发送，只保留：

- 特征点
- 轨迹
- 稀疏地图点
- 位姿

双目模式：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --no-send-images
```

RGBD 模式：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode rgbd --preset quality --no-send-images
```

补充说明：

- `--no-send-images` 只是不发送图像帧，不会影响建图主流程
- 当前延迟优化不只依赖这个开关，还包括：
  - 丢弃积压旧帧，只处理最新帧
  - `Rerun` 降频更新 `--viz-every-n`

### 显式指定 viewer 地址

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh \
  --mode stereo \
  --viewer-addr 192.168.31.223:9876
```

### 传额外参数给底层脚本

示例：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh \
  --mode stereo \
  --preset quality \
  -- --duration-sec 60 --max-landmarks-distance 15 --map-cell-size 0.5
```

## 模式说明

### stereo

特点：

- 轨迹通常更稳
- 更适合作为主方案
- 地图是稀疏的，不会很“满”
- 可视化带宽压力相对更小

适用场景：

- 更看重位姿准确性
- 更看重单轮稳定性
- 场景纹理较丰富

### rgbd

特点：

- 近距离轮廓通常更完整
- 更容易看出场景大致结构
- 深度噪声、边缘错位、反光表面会更明显污染地图
- 发送到本机的可视化数据量更大，更容易出现延迟

适用场景：

- 更看重轮廓完整度
- 场景距离较近
- 光照和深度质量较稳定

## preset 说明

### quality

- 更偏向地图质量
- 会使用更保守的参数
- 默认更适合正式采图
- 在无线链路下，`rgbd` 更建议配合 `--no-send-images`

当前默认差异：

- `stereo`
  - `--max-landmarks-distance 15`
  - `--map-cell-size 0.5`
  - `--landmark-log-interval 5`
  - `--viz-every-n 2`
  - `--use-motion-model`
- `rgbd`
  - `--max-landmarks-distance 12`
  - `--map-cell-size 0.4`
  - `--landmark-log-interval 5`
  - `--viz-every-n 3`

理解方式：

- 地图点更新更频繁
- 地图范围控制更保守
- 更适合慢速、小范围、正式建图
- 更适合评估单轮建图质量

### fast

- 更偏向快速启动和轻量可视化
- 更适合快速试通链路
- 更适合带宽受限场景

理解方式：

- 地图点刷新频率更低
- 更适合先看链路是否跑通
- 更适合无线网络下减少 `Rerun` 延迟
- 更适合长时间运行时降低本机 viewer 压力

当前默认差异：

- `stereo`
  - `--landmark-log-interval 15`
  - `--viz-every-n 4`
- `rgbd`
  - `--landmark-log-interval 15`
  - `--viz-every-n 6`

### robot

- 主要针对四足机器狗这类急转、抖动、非平滑运动场景
- 核心思路是提高 `stereo` 模式下的时间分辨率，减小急转时帧间变化
- 目前更适合作为 `stereo` 的机器人专用 preset
- 不建议默认用于 `rgbd`

当前默认差异：

- `stereo`
  - `--width 640`
  - `--height 360`
  - `--fps 60`
  - `--max-landmarks-distance 12`
  - `--map-cell-size 0.5`
  - `--landmark-log-interval 5`
  - `--viz-every-n 3`
  - `--use-motion-model`
- `rgbd`
  - `--width 640`
  - `--height 360`
  - `--fps 60`
  - `--max-landmarks-distance 10`
  - `--map-cell-size 0.4`
  - `--landmark-log-interval 5`
  - `--viz-every-n 4`
  - `--use-motion-model`

建议：

- 四足实际建图优先先试 `stereo --preset robot`
- `rgbd` 更建议继续用 `quality`

## `fast` 与 `quality` 如何选

如果你的目标是：

- 先确认设备、容器、viewer、建图链路都正常：选 `fast`
- 正式采一轮地图并观察质量：选 `quality`
- 用无线网络连 Jetson，viewer 延迟明显：优先 `fast`，或在 `quality` 下加 `--no-send-images`
- 想看图像和特征点细节：优先 `quality`，必要时不要加 `--no-send-images`
- 机器狗急转、抖动明显：优先 `stereo --preset robot`

推荐组合：

快速检查链路：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --preset fast --no-send-images
```

正式双目建图：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --preset quality
```

正式 RGBD 建图：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode rgbd --preset quality --no-send-images
```

机器狗双目建图：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --preset robot --no-send-images
```

## 输出结果

默认输出目录：

```bash
/home/unitree/datasets/cuvslam_maps
```

每次运行会生成一个会话目录，例如：

```bash
/home/unitree/datasets/cuvslam_maps/realsense_stereo_slam_YYYYMMDD_HHMMSS
/home/unitree/datasets/cuvslam_maps/realsense_rgbd_slam_YYYYMMDD_HHMMSS
```

目录内通常包含：

- `session.log`
- `trajectory_odom_tum.txt`
- `trajectory_slam_live_tum.txt`
- `trajectory_slam_optimized_tum.txt`
- `final_landmarks.csv`
- `map/data.mdb`

其中：

- `map/data.mdb` 是 cuVSLAM 保存的可复用地图数据库
- `final_landmarks.csv` 是稀疏地图点，不是稠密点云

## 查看日志

启动后包装脚本会打印：

- `Container`
- `Output dir`
- `Log file`

跟踪实时日志：

```bash
docker exec <container_id> bash -lc 'tail -f <session.log路径>'
```

## 当前已知边界

- 这是 **稀疏 SLAM 地图**，不是稠密重建网格
- 不是彩色点云建图
- 不是占据栅格地图
- 如果目标是 RTAB-Map / TSDF / 彩色稠密建图，需要在 cuVSLAM 位姿之上再叠一层建图模块

## 建图质量建议

建议按下面方式采图：

- 场景要有纹理，避免白墙、玻璃、大面积纯地面
- 先小范围慢速移动
- 多平移，少快速原地转头
- 光照保持稳定
- 先做 20 到 40 秒的小回路

如果发现：

- 轨迹准，但地图粗糙：优先继续优化建图参数或切换 `rgbd`
- 轨迹不准：优先检查 USB、相机占用、快速运动、低纹理场景
- viewer 延迟很高：优先使用 `--no-send-images`
- 急转时角度偏移：优先尝试 `stereo --preset robot`

## 常见问题

### 1. 找不到设备 / `No device connected`

常见原因：

- 上一个 `run_stereo*` / `run_rgbd*` 进程还在占相机
- USB 供电服务没起来
- 相机刚重连，设备还没稳定枚举

优先用：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --status
```

### 2. viewer 没画面

先确认本机 viewer 真在监听：

```bash
/home/jycao/.cargo/bin/rerun --bind 0.0.0.0 --port 9876
```

再确认包装脚本启动时打印出的 `Viewer` 地址正确。

### 3. viewer 延迟很高

这通常不完全是 `cuVSLAM` 前端本身算不动，而是下面几项叠加：

- 无线链路带宽有限
- `rgbd` 下除了轨迹和地图点，还会多一张深度图
- `Rerun` 可视化更新过于频繁
- 旧帧积压没有及时丢弃

优先尝试：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --no-send-images
```

或者：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode rgbd --preset quality --no-send-images
```

当前已经做过的延迟优化：

- 可选 `--no-send-images`
- 只处理最新帧，主动丢弃积压旧帧
- 通过 `--viz-every-n` 降低 viewer 更新频率

### 4. 3D 地图看起来很乱

先区分两类情况：

- 真正的建图质量差
- 稀疏地图本来就不致密

当前脚本已经修正为使用 `SLAM Map` 层做 3D 地图显示，不再混用里程计坐标系。

## 推荐测试顺序

1. 先跑：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode stereo --preset quality --no-send-images
```

2. 在同一场景再跑：

```bash
/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh --mode rgbd --preset quality --no-send-images
```

3. 对比：

- 轨迹漂移
- 地图轮廓完整度
- 地图噪声

一般建议：

- 更看重位姿稳定：优先 `stereo`
- 更看重轮廓完整：尝试 `rgbd`
