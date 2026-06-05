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

**已确认的事实（由 Round 0–6 数据支撑）：**

1. **桥接过滤器不是根因。** 从 Round 0 分析 → Round 2 bypass 验证，过滤器本身不引发跳变。
2. **D435i IMU / Camera 在 Jetson 上持续丢帧。** 所有 90+ 个 cuVSLAM session 都有 `IMU stream message drop` 和 `Camera stream message drop`，丢帧计数随运行时间增长（2 → 400+）。
3. **IMU 丢帧导致 VIO 退化为纯视觉。** cuVSLAM 日志明确显示 `No IMU measurements between camera frames`，融合降级。
4. **--relocalize-on-tracking-loss 全程禁用。** 排除重定位参与。

**待确认的推测（缺直接证据）：**
- IMU 丢帧是 USB 带宽不足还是控制器/驱动层面的问题——通过降低帧率（30fps）未改善，倾向后者但**没有 USB 抓包确认**。
- 丢帧的根本原因是 Jetson 硬件限制还是驱动/内核配置问题——未排查。

**已排除的干预方案（Round 7–9）：**

| 尝试 | 结果 | 结论 |
|------|------|------|
| Round 7: 降低帧率 60→30fps | ❌ 丢帧更严重（751 vs 403） | 非带宽问题，恢复 60fps |
| Round 8: L1 IMU 陀螺积分桥接 | ❌ 效果差于 D435i IMU | L1 IMU 品质或标定可能不足 |
| Round 9: Go2 机体 IMU 绝对偏航替换 | ❌ 跳变更严重 | 世界系无法与 cuVSLAM 地图系简单对齐 |

> 干预方案全都实施在分支 cuvslam-autonomy-imu 上，对主线 cuvslam-autonomy 无影响。

### Round 7: 降低 D435i 帧率（试探性）

尝试降低 D435i 相机帧率以减轻 USB 带宽压力，验证丢帧是否由带宽引起。

**方法：** `/home/unitree/cuVSLAM/docker/start_realsense_slam_map.sh` 的 stereo robot preset `--fps 60` → `--fps 45` → `--fps 30`

**发现：**
- 45fps → D435i 不支持，`RuntimeError: Couldn't resolve requests`，cuVSLAM 进程崩溃
- 30fps → 启动成功，但 session `071306` IMU drops 达 **751**（60fps 最差 session 为 403）

**分析：** 如果丢帧是 USB 带宽饱和导致，减半帧率应大幅减少丢帧。实际反而增加，说明**带宽假设不成立**。但 USB 抓包未做，无法判断是控制器硬件、驱动还是其他原因。

**状态：** 已恢复 60fps，此方向暂停。

### Round 8: L1 IMU Bridge 侧陀螺积分（试探性）

由于 cuVSLAM Docker 不支持外部 IMU 订阅（只有 `--use-imu` 启用 D435i 内置 IMU），在桥接侧实现 IMU 旋转备份。

**方案：** 订阅 L1 LiDAR IMU（`/utlidar/transformed_imu`，以太网链路），陀螺积分跟踪相对旋转，在 cuVSLAM 偏航偏离超过 15° 时替换。

**结果：** ❌ 效果甚至不如 D435i IMU。推测原因：
- L1 本体 IMU 品质较低（主要用于 LiDAR 去畸变，非导航级）
- `transform_everything` 的标定参数可能有误差

**状态：** 此方向暂停。

### Round 9: Go2 机体 IMU 绝对偏航替换（试探性）

改用 Go2 机身内置 AHRS 的绝对四元数（`/lowstate.imu_state.quaternion`），消除积分漂移。

**方案：**
- 首帧计算 offset = cuVSLAM_yaw − Go2_yaw
- 每帧偏差 > 15° 时用 Go2_yaw + offset 替换
- offset 学习率 0.05 缓慢跟踪

**结果：** ❌ 跳变更严重。可能原因：
- Go2 IMU 的世界系（上电初始方向）与 cuVSLAM 地图系不一致
- AHRS 偏航在机器人运动时受加速度/振动干扰
- 15° 阈值过大，替换瞬间已造成视觉不连续

**状态：** 此方向暂停。桥接侧单 IMU yaw 替换不可行。

## 待确认 / Next Steps

1. ~~cuVSLAM Docker 内部 IMU 状态~~ ✅ 已确认正常
2. ~~RealSense D435i 相机标定~~ ✅ FW 5.17.0.10，IMU profiles 正常
3. ~~Jetson 系统资源~~ ✅ 正常，排除此因素
4. ~~cuVSLAM 配置参数~~ ✅ robot preset 正常
5. ~~降低 D435i 流参数（30fps）~~ ❌ 无效
6. ~~L1 IMU 桥接旋转校正~~ ❌ 失败
7. ~~Go2 机体 IMU 绝对偏航~~ ❌ 跳变更严重
8. [ ] **待探索：** USB 抓包分析 jetson xusb 控制器行为——确认是硬件限�还是驱动问题
9. [ ] **待探索：** powered USB hub 或外接 USB 控制器绕过 on-chip xusb
10. [ ] **待探索：** 放弃 cuVSLAM VIO，改用 Point-LIO（LiDAR-IMU 紧耦合）+ L1 IMU
