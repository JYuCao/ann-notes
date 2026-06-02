
system_cuvslam_cruise_headless.sh
    ros2 launch vehicle_simulator system_cuvslam_cruise_headless.launch
        mapping_cuvslam.launch
            transfrom_everything (L1 点云预处理)
            cuvslam_ros_bridge (cuVSLAM UDP -> ROS 桥接)
        terrain_analysis.launch
            terrainAnalysis
        local_planner.launch
            localPlanner （订阅 /terrain_map 做路径规划）
            pathFollower (路径跟踪)
        joy_node
        static_transform_publisher x2 (map->camera_init, aft_mapped->sensor)

## 6.1

目前机器狗能做到一定程度的巡航，但在避障方面仍有待提升的空间。

存在的问题：
1. ✅用于屏蔽自身的点云数据的滤波器目前位置错误，在机器狗的右前方不远处。
    - 相关部分在 src/utilities/transform_sensors/transform_sensors/transform_everything.py 中有关 filter 的参数，需进行调整
    - 首先解决 filter box 角度没有和机体对齐的问题
    - 调整 filter box 的 xyz 参数，使其能够正确地屏蔽掉机器狗自身的点云数据
    - transform_everything.py需要进行重写，整个坐标系的对齐问题很大

2. （待解决）waypoint 判定过于严格，导致机器狗到达goal时在原地抽搐。
    - 相关参数：
        - src\base_autonomy\local_planner\launch\local_planner.launch:
            - goalCloseDis(launch arg & pathFollower)
            - goalCloseDis(localPlanner node)
            - stopDisThre
            - slowDwnDisThre
        - src/base_autonomy/vehicle_simulator/launch/system_cuvslam_cruise.launch
            - goalCloseDis
        
3. （待解决）避障部分存在的问题，我认为是避障的约束过于不敏感，导致机器狗在遇到障碍物时不会及时调整路径，甚至可能会撞上障碍物。
4. ✅如果发现启动巡航后，设置waypoint机器狗不动，很有可能是local_planner 或 path_follower 没有编译通过，或者没有正确启动；也有可能是进程池没有清理干净，导致之前的进程占用了资源。可以通过检查相关节点的日志来确定问题所在。
    - 最终发现是 local_planner 在 launch 文件中的 node 被错误注释，已修改
5. ✅cuVSLAM 的 imu 无法启用，经检查是外围脚本把 host 的 librealsense 覆盖进 docker 容器，host的 librealsense 版本无法识别 D435i 的 imu。
    - 解决方法：在外围脚本中设置优先使用容器内的 librealsense 库，或者更新 host 的 librealsense 版本以支持 D435i 的 imu。

## 6.2

昨天初步加入了重定位功能。在刚启动时会进行一次重定位，之后在巡航过程中如果轨迹突然偏移较大如碰撞后地图漂移，也会触发重定位。

目前来看重定位功能问题较多，主要表现如下：

1. （待解决）由于 cuVSLAM 的静态地图没有与 autonomy 部分的坐标系对齐，导致每次重定位虽然成功了，但方向没有统一。
    - 解决方法：目前我认为可以统一一个静态的全局坐标系，将所有 TF 都对齐到这个坐标系上，这样无论是重定位还是正常巡航，都会在同一个坐标系下进行。这样每次初始重定位都可以将地图方向统一。

2. （待解决）巡航过程中重定位的触发条件过于敏感，导致在正常巡航过程中偶尔会误触发重定位，导致地图到处乱飞。
