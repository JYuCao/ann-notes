# 位姿/地图跳变排查记录

## 背景

Go2 机器狗在 cuVSLAM + autonomy_stack_go2 巡航模式下，旋转/晃动时出现位姿跳变、局部地图旋转，导致失控闭环。

## 排查记录

### Round 0: 初始分析（2026-06-05）

**现象**：旋转时位姿跳变，日志中 `Recovered after 5 rejects` 频繁触发（30+ 次/200s），`pose_hz` 从 29 降至 5 Hz。

**第一步：排除重定位因素**
通过参数传递链全线确认 `--relocalize-on-tracking-loss` 为 False，relocalization 不参与跳变。

**第二步：确认 `_pose_ok` recovery 为直接诱因**

```python
# cuvslam_ros_bridge.py _pose_ok
if dist > 0.5m or rot > 30°:
    self._reject_buf.append(pose)
    if len(self._reject_buf) >= 5:
        recent = self._reject_buf[-5:]
        std = np.std(...)
        if std < 0.1:
            self.last_pose_raw = pose
            ok = True
```

跳变链：旋转 → track() 瞬间不稳 → 带噪声位姿突破阈值（0.5m/30°）→ 连续拒绝5帧，这5帧恰好相互接近（std<0.1）→ recovery 将其作为"新稳定位姿"接受 → 永久跳变。

**日志证据：**
```
[WARN] Recovered after 5 rejects (std=0.000)   ← +4.7s
[WARN] Recovered after 5 rejects (std=0.000)   ← +9.1s
[WARN] Recovered after 5 rejects (std=0.008)   ← +23.1s
[WARN] Recovered after 5 rejects (std=0.007)   ← +59.8s
... (持续 200s 内 30+ 次)
pose_hz: 29Hz → 16Hz → 11Hz → 5Hz
```

**初步修复方案**（已实施后逐步回退）：
1. `_pose_ok` 阈值收紧：`dist<0.5m→0.10m`, `rot<30°→5.0°`
2. Recovery 帧数：5→20，std<0.1→0.01，新增四元数 std 检查
3. 输入位姿 EMA 平滑（α=0.5）

### Round 1: 桥接收敛优化迭代（6.5）

**修改内容**（多轮迭代）：

| 调整 | 初始值 | R1a | R1b | R1c | R1d |
|------|--------|-----|-----|-----|-----|
| 帧间位置阈值 | 0.5m | 0.10m | 0.5m（恢复） | 0.5m | 0.5m |
| 帧间角度阈值 | 30° | 5° | 30°（恢复） | 30° | 30° |
| Recovery 帧数 | 5 | 20 | 20 | 10 | bypass |
| Recovery 条件 | `std<0.1` | `t_std<0.01 && q_std<0.01` | 同上 | 同上 | — |
| 新增 | — | 添加 q_std 检查 | — | **移除 q_std** | — |
| 输出平滑 | — | EMA α=0.5 | 同上 | 同上 | — |

**测试反馈**：
- 收紧阈值后：旋转问题更严重（过于频繁拒绝，桥接卡死）
- 恢复阈值 + 保留 20 帧 recovery：仍然卡死（q_std 阻止 recovery）
- 移除 q_std + 10 帧 recovery：好一些，但仍跳变
- 完全 bypass `_pose_ok`：仍跳变

**结论：问题不在桥接过滤器。**

### Round 2: 无过滤验证（6.5）

将 `_pose_ok(pose)` 改为直通（`else:`），所有 cuVSLAM 原始 UDP 位姿直接发布为 `/state_estimation`。

**日志关键发现**（bypass 测试，06:37 UTC）：

```
时间轴    | Odom 位置          | pose_hz | RegScan
----------|--------------------|---------|--------
0:00      | [0.35, 0.13]      | 20.2    | 15.4
0:50      | [-1.04, -2.94]    | 13.6    | 15.4
0:55      | [-1.15, -2.03]    | 13.1    | 15.4   ← y 跳 +0.91m
1:05      | [-0.38, -0.50]    | 12.8    | 15.4   ← 回跳 ~1.5m
1:10      | [0.20, 0.19]      | 12.1    | 15.4
1:25      | [-0.06, 1.24]     | 10.3    | 15.4
1:35      | [-1.19, 1.25]     | 10.3    | 15.4   ← x 跳 -1.13m
1:40      | [-2.60, 0.63]     | 10.1    | 15.4
1:50      | [-4.14, -0.45]    | 9.4     | 15.4
2:00      | [-4.03, -0.77]    | 9.9     | 15.4
2:05      | [-5.33, -0.37]    | 8.2     | 15.4
```

- **位置跳变明显**：0.9m~1.5m 的阶跃（每 5s 采样），无 `Recovered` 日志
- **pose_hz 持续下降**：20.2 → 8.2 Hz（2 分钟内掉了一半），说明 cuVSLAM 发送 UDP 位姿的速率在降低
- **RegScan 稳定**：15.4 Hz，传感器数据正常
- 逐帧漂移量小（帧间 delta < 0.5m/30°），所以过滤器全部直通无拦截

**根因确认：cuVSLAM 自身输出不稳定。** 机器狗晃动/旋转时 cuVSLAM 视觉跟踪漂移（帧间小步变化），偶尔重新定位回正确位置形成跳变，且输出频率随时间下降。

### Round 3: IMU 架构分析（6.5）

系统存在三个 IMU：

| IMU | 来源 | 通信方式 | cuVSLAM 模式 | Autonomy 模式 |
|-----|------|---------|-------------|---------------|
| L1 LiDAR 内置 IMU | L1 本体 | ROS 话题 `/utlidar/imu` | ❌ `imu_en: false`，不参与 | ✅ transform_everything→Point-LIO（仅 `system_real_robot.launch`） |
| RealSense D435i IMU | D435i，USB 连接 Jetson | Docker 内 librealsense 直读 | ✅ 启动参数 `--use-imu`，在容器内部使用 | ❌ 不参与 |
| Go2 机身 IMU | 主板 IMU | Unitree SDK（`IMUState.msg`） | ❌ | ❌ 仅在 LowState/SportModeState 消息中 |

**关键发现**：
- cuVSLAM 模式下，L1 IMU 已被禁用（`mapping_cuvslam.launch` 不传 IMU 配置，`utlidar.yaml` 中 `imu_en: false`）
- RealSense IMU 是 cuVSLAM 的唯一惯性信息来源
- RealSense IMU 是否真正生效存疑——历史问题"cuVSLAM 的 imu 无法启用"曾因 host/container librealsense 冲突导致 IMU 不可用，当时的修复是修改外围脚本优先使用容器内库

**LiDAR → cuVSLAM 数据流（cuvslam 模式）：**
```
L1 LiDAR
  ├── /utlidar/imu ──→ transform_everything ──→ /utlidar/transformed_imu （无人订阅，imu_en=false）
  └── /utlidar/cloud ──→ transform_everything ──→ /utlidar/transformed_cloud ──→ cuvslam_ros_bridge（注册点云）

RealSense D435i
  └── librealsense SDK ──→ cuVSLAM Docker 容器（--use-imu，仅 SLAM 内部使用，不发布 ROS 话题）
```

**TF 树：**
```
world ──→ map ──→ aft_mapped ──→ sensor
                              └── camera_link
```
- world：全局固定轴，cuvslam_ros_bridge 通过环境变量配置平移和偏航
- map：cuVSLAM camera_init 的别名，重定位后与 world 对齐
- aft_mapped：机身坐标系，cuVSLAM 位姿输出
- 注：旧版有 camera_init 帧，已合并为 map

### Round 4: cuVSLAM 内部跟踪丢失机制

cuVSLAM（`run_stereo_slam_map.py`）的 pose 输出逻辑：

```python
odom_pose = pose_estimate.world_from_rig.pose      # 纯视觉里程计位姿
pose_for_output = slam_pose if has_localized_pose
                            and slam_pose is not None
                            else odom_pose           # ← 切换点！
```

- 定位成功前：发送 odom_pose（里程计，漂移但连续）
- 定位成功后：发送 slam_pose（地图坐标系下的全局位姿）
- 两个位姿之间几乎必然存在跳跃

bridge 端强制接受这个跳跃：
```python
if is_localized and not self._was_localized:
    self._was_localized = True
    self.last_pose_raw = pose      # ← 直接覆盖，跳过 _pose_ok 滤波
    self._reject_buf = []
    self.publish_odom(pose)        # ← 立即发布，造成跳变
```

旋转 → 特征丢失 → track() 返回 None → 不发送 UDP → bridge latest_pose 冻结 → 重新跟踪成功（或 reloc success）→ odom→slam 切换 → 突然跳变

### Round 5: IMU 影响与电机振动

| 因素 | 影响 |
|------|------|
| 电机振动 → IMU 噪声 | 降低 track() 稳定性，更容易丢跟踪 |
| 纯旋转 → 视场快速变化 | 特征点丢失，world_from_rig = None |
| odom→slam pose 切换 | 直接导致跳变 |
| bridge 强制接受 | 绕过 _pose_ok 滤波，跳变直接输出 |

### 其他线索

- **坐标系对齐**：cuVSLAM 静态地图与 autonomy 坐标系已通过 world→map TF 统一，但重定位后的方向偏差仍有待验证
- **重定位触发敏感**：巡航中误触发重定位，地图乱飞，与跳变互为因果（已通过参数链确认当前全线禁用）

---

## Round 6: 全面排查结果汇总（2026-06-05）

### 各项排查结果

| # | 排查项 | 状态 | 结论 |
|---|--------|------|------|
| 1 | cuVSLAM Docker IMU 状态 | **确认正常** | 容器内 librealsense 2.57.6，Accel 400/200/100Hz + Gyro 400/200Hz profiles 可用，`--use-imu` 参数有效传递 |
| 2 | cuVSLAM 日志 IMU 运行状态 | **发现问题** | `IMU fusion enabled with 200 Hz` 确认启动；**但全程存在 IMU stream drops**（详见下方分析） |
| 3 | Jetson 系统资源 | **正常** | CPU 2% idle, RAM 2/15GB 已用, temp 55-57°C, CPU freq 2.0GHz, 无降频 |
| 4 | cuVSLAM 配置参数 | **正常** | robot preset: `640x360@60fps, --use-imu, --use-motion-model, max-landmarks-distance=15m, map-cell-size=0.4` |
| 5 | 桥接过滤器 | **已 bypass** | git checkout 恢复原始代码 + bypass, 当前所有位姿直通 |

### 关键发现：IMU / Camera 流持续丢帧

从 cuVSLAM session log (`realsense_stereo_slam_20260605_063445/session.log`) 中发现：

```
IMU fusion enabled with 200 Hz motion streams
Warning: IMU stream message drop: timestamp gap (29.47 ms) exceeds threshold 25.00 ms; count=51
Warning: Camera stream message drop: timestamp gap (66.79 ms) exceeds threshold 35.00 ms; count=21
Warning: No IMU measurements between camera frames X and Y; count=21
```

**IMU stream drops 在所有 session 中持续出现**（统计 90+ 个 session）：

- 早期 session（6月1日首次调试）：2-5 次丢帧 / session
- 后期 session（6月3-5日）：10-400+ 次丢帧 / session（随运行时间增加）
- 最严重：403 次 IMU drops + 87 次 camera drops（session `025412`）

**丢帧模式**：
- IMU 期望 200Hz（5ms 间隔），阈值 25ms = 连续丢 5 帧算一次"drop"
- Camera 期望 60fps（16.7ms 间隔），阈值 35ms = 连续丢 2 帧算一次"drop"
- 丢帧计数持续增长说明这是**系统性问题**，而非偶发

### 根因判断

```
RealSense D435i USB 3.0 → Jetson tegra-xusb 控制器
  ├── 4x UVC (camera) streams: 640x360@60fps stereo+depth+color
  └── 1x HID (IMU) stream: 200Hz Accel+Gyro
       ↓
  Jetson USB 控制器无法维持全部 isochronous 流
       ↓
  周期性 IMU/Camera 帧丢失
       ↓
  cuVSLAM visual-inertial 融合退化为纯视觉里程计
       ↓
  旋转时姿态漂移 → 偶发重定位 → 位姿跳变
```

**根本原因**：Jetson Orin NX 的 tegra-xusb 控制器无法稳定维持 D435i 全部流的实时性，导致 IMU 数据频繁丢失，cuVSLAM 的 VIO 融合降级。

### 可能的缓解方案

1. **降低 D435i 帧率/分辨率**：尝试 30fps 或 480p，减少 USB 带宽压力
2. **禁用颜色流**（如果当前已启用）：减少一个 UVC stream
3. **启用 L1 IMU**（当前被禁用 `imu_en: false`）：作为 D435i IMU 的冗余/备份
4. **调整 cuVSLAM IMU 丢帧阈值**：放宽 `25ms` 阈值，减少警告但不解决实际问题
5. **自研 IMU+LiDAR 姿态融合**：用 L1 IMU（稳定性更好）做旋转时的姿态预测，cuVSLAM 做慢漂校正

### 结论

**已确认的事实（由 Round 0–10 数据支撑）：**

1. **桥接过滤器不是根因。** 从 Round 0 分析 → Round 2 bypass 验证，过滤器本身不引发跳变。
2. **D435i IMU / Camera 在 Jetson 上持续丢帧。** 所有 90+ 个 cuVSLAM session 都有 `IMU stream message drop` 和 `Camera stream message drop`，丢帧计数随运行时间增长（2 → 400+）。
3. **IMU 丢帧不是跳变的直接原因。** Round 10 对比实验：无重定位模式下 IMU 丢帧（21-151 次/session）但位姿几乎无跳变；有重定位模式下丢帧严重（1641 次）且跳变明显。说明 IMU 丢帧单独存在不会引起跳变，需结合其他因素。
4. **--relocalize-on-tracking-loss 全程禁用。** 排除运行中重定位参与。

**根因修正（Round 10）：**

跳变核心原因是桥接 `udp_pose_listener` 接收了 cuVSLAM 非定位态（`localized: false`）的 UDP 位姿并发布了错误的 `world→aft_mapped`。当 cuVSLAM 定位成功（`localized: true`）时，桥接强制发布正确位姿，形成跳变。修复：在 `_pose_ok` 分之前加 `is_localized and` 守卫，禁止非定位态位姿流入。

**已排除的干预方案（Round 7–9，已精简 → 详见 `cuvslam-autonomy-imu` 分支）：**

| 尝试 | 结果 | 结论 |
|------|------|------|
| Round 7: 降低帧率 60→30fps | ❌ 丢帧更严重（751 vs 403） | 非带宽问题，恢复 60fps |
| Round 8: L1 IMU 陀螺积分桥接 | ❌ 效果差于 D435i IMU | L1 IMU 品质或标定可能不足 |
| Round 9: Go2 机体 IMU 绝对偏航替换 | ❌ 跳变更严重 | 世界系无法与 cuVSLAM 地图系简单对齐 |

> R7: 45fps 不支持崩溃，30fps 反而丢帧增加 → 带宽假设不成立。R8: L1 IMU 以太网链路，陀螺积分在偏航偏差 > 15° 时替换 cuVSLAM yaw。R9: Go2 AHRS 绝对四元数 + offset 替换。三者均在 `cuvslam-autonomy-imu` 分支，主线无影响。

### Round 10（2026-06-09）：有/无重定位对比 + 根因定论

**实验设计：** 七轮对比实验，交替运行无重定位（fresh SLAM）和有重定位（localization，map `realsense_stereo_slam_20260601_151246`），观测位姿跳变和 IMU 丢帧。

**结果：**

| # | 时间 | 类型 | Poses | IMU 最终 count | 位姿跳变 |
|---|------|------|------|---------------|---------|
| 1 | 06:57 | Mapping（无重定位） | 8164 | 21 | ✅ 几乎无跳变 |
| 2 | 07:03 | Mapping（无重定位） | 1895 | 1 | ✅ 几乎无跳变 |
| 3 | 07:04 | Mapping（无重定位） | 5791 | 101 | ✅ 几乎无跳变 |
| 4 | 07:07 | **Localization**（有重定位） | 3385 | 101 | ❌ 跳变明显 |
| 5 | 07:10 | Mapping（无重定位） | 7385 | 151 | ✅ 几乎无跳变 |
| 6 | **07:22** | **Localization**（有重定位） | CRASHED | **1641** | ❌ CUDA 崩溃 |
| 7 | 07:24 | Localization（有重定位） | CRASHED | 1 | ❌ CUDA 崩溃 |

**关键发现：**

1. **无重定位（Mapping）位姿几乎无跳变**，尽管 IMU 丢帧与之前（21-151 次）类似 → **IMU 丢帧单独不是跳变根因。**
2. **有重定位（Localization）跳变明显**，且后两次重定位崩溃（`[CUDA] error driver shutting down(4)`）
3. 重定位会话 072215 包含 **1641 次 IMU 丢帧**（之前最严重为 403），加上 CUDA 崩溃，表明重定位过程本身显著增大了系统压力

**跳变根因分析：**

桥接 `udp_pose_listener` 在重定位启动阶段收到 cuVSLAM 的 `localized: false` 位姿。原代码 `elif self._pose_ok(pose):` 没有 `is_localized` 守卫，导致非定位态位姿被接受并发布。当 cuVSLAM 最终定位成功（`localized: true`）时，桥接通过 `if is_localized and not self._was_localized:` 强制发布正确位姿，形成跳变。

```
cuVSLAM 启动（加载地图）
  ├── localized: false → 桥接 else 分支 → 发布错误位姿 → latest_pose 污染
  └── localized: true  → 桥接 force-publish → 正确位姿 → JUMP！
```

无重定位（Mapping）模式下 cuVSLAM 一启动就是 `localized: true`（构建新地图无需对齐），不存在上述切换，所以平滑。

**修复（已部署）：**

在桥接 `elif self._pose_ok(pose):` 前加 `is_localized and` 守卫，确保非定位态位姿永远不会通过 `_pose_ok` 发布。另修复了 `_R_total_lm` 竞态条件和新增 TF 心跳机制。

**遗留问题：**
- 重定位 session 6/7 的 CUDA 崩溃可能与内存泄漏或长时间 relocalization 尝试有关（日志显示 `nanobind` 引用泄漏）
- 地图 `realsense_stereo_slam_20260601_151246` 建图时已有 IMU 丢帧，地图质量存疑

### Round 11（2026-06-09）：IMU 丢帧系统分析 + 缓解措施

**问题：** 有重定位会话 IMU 丢帧（951 次 / 81s ≈ 11.7次/秒）远高于无重定位（151 次 / 123s ≈ 1.2次/秒），即使定位成功后仍持续丢帧（~8.9次/秒）。

**排查过程：**

**设备层：** D435i（FW 5.17.0.10）通过 USB 3.2 连接 Jetson tegra-xusb 控制器。`lsusb -t` 显示 5 路 UVC + 1 路 HID 活跃：
```
IR Left, IR Right, Depth, RGB, Metadata, Motion(IMU)
```
即使 cuVSLAM 只订阅 IR1+IR2，深度和 RGB 流仍通过 USB 传输（D435i 固件默认行为）。

**系统资源：** MAXN 功率模式，CPU 8核 @ 1.98GHz，内存 2.1/15GB 已用，温度 58-62°C，Docker 无资源限制。

**代码层面：** Mapping 和 Localization 使用完全相同的 camera pipeline（`config.enable_stream(IR1, IR2)`）+ motion pipeline（`rs.stream.accel + rs.stream.gyro`）。唯一差异是 Localization 额外运行 `LocalizationManager`（对每帧做 40000+ 地图陆标的特征匹配），增加 CPU/GPU 开销。

**丢帧时间分布分析：**

| 阶段 | 时长 | IMU stream drops | Camera drops | No IMU meas |
|------|------|-----------------|-------------|-------------|
| 定位中（前 35s） | 35s | 1 | 2 | **27** |
| 跟踪中（后 46s） | 46s | 19 | 31 | 1 |

- **定位阶段** 27 次 "No IMU measurements" → IMU worker 线程偶尔未及时送数到 buffer，仅 ~1.3% camera frames 受影响
- **跟踪阶段** IMU/Camera stream drops 持续发生 → USB 等时传输实际丢帧

**关键发现（时序分析法）：**
```
06:57 Mapping:   21  IMU drops
07:03 Mapping:    1
07:04 Mapping:  101
07:07 Localization: 101
07:10 Mapping:   151    ← Mapping 自身也在递增！
07:22 Localization: 1641 (崩溃)
07:49 Localization: 951
```
Mapping 会话的丢帧量也在随总运行时间递增（21→101→151），说明 tegra-xusb 控制器的等时传输质量随持续运行时间**单调下降**，与模式无关。

**结论：**

丢帧差异的主要原因不是"重定位模式本身"，而是两个因素的叠加：

1. **时序因素**：D435i + tegra-xusb 随持续运行 USB 等时传输质量逐步下降（所有会话的丢帧量按时间递增）
2. **负载因素**：重定位的额外 CPU/GPU 开销（特征匹配 + 位姿图优化）间接干扰 USB 中断调度，放大丢帧

**缓解措施：**

1. **禁用未使用的 D435i 流**：通过 librealsense 配置限制 depth/color 输出，减少 USB 带宽占用
2. **D435i USB 硬重置**：每次 cuVSLAM 启动前重置 USB 设备，清空控制器累积状态

## 待确认 / Next Steps

1. ~~cuVSLAM Docker 内部 IMU 状态~~ ✅ 已确认正常
2. ~~RealSense D435i 相机标定~~ ✅ FW 5.17.0.10，IMU profiles 正常
3. ~~Jetson 系统资源~~ ✅ 正常，排除此因素
4. ~~cuVSLAM 配置参数~~ ✅ robot preset 正常
5. ~~降低 D435i 流参数（30fps）~~ ❌ 无效
6. ~~L1 IMU 桥接旋转校正~~ ❌ 失败
7. ~~Go2 机体 IMU 绝对偏航~~ ❌ 跳变更严重
8. [x] ~~用 `is_localized` 守卫跑一轮重定位巡航~~ ✅ 位姿跳变已基本消除（接近无重定位效果）
9. [ ] **待验证：** 禁用 depth/color 流 + USB 重置是否能降低 IMU 丢帧
10. [ ] **待测试：** 如果丢帧改善，评估是否需要重建重定位地图（现有地图建图时有 IMU 丢帧）
11. [ ] **待排查：** 重定位时的 CUDA 崩溃（`nanobind` 泄漏 + `error driver shutting down`）
12. [ ] **待探索：** USB 抓包分析 jetson xusb 控制器行为
13. [ ] **待探索：** powered USB hub 或外接 USB 控制器绕过 on-chip xusb
