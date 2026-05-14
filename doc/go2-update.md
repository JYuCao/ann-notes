# 宇树 GO2 EDU Jetson 板卡升级流程

实验室的宇树 GO2 EDU 机器人使用 Jetson Orin NX 作为核心计算平台，官方出厂预装 Ubuntu 20.04，Jetpack 5.1.1，ROS2 版本为 Foxy，且没有官方的升级渠道。某些情况下，用户可能需要升级 Jetson 板卡的系统和驱动，以获得更好的兼容性。

本文档将介绍如何升级 Jetson 板卡的系统和驱动到 Ubuntu 22.04，Jetpack 6.2.1，Cuda 12.2，以确保板卡可以使用较新的软件栈。

> 文档写于2026.5.7，请注意时效性

## 数据备份

在升级之前，建议先备份 Jetson 板卡上的重要数据和配置文件。可以参考官方文档中的备份步骤：

1. 把拓展坞的 NVME 设备取出，通过读卡器连接到 Ubuntu PC，`lsblk -f` 查看该存储设备节点（如 `/dev/sdc/`，格式为 `ext4`）

2. 通过以下命令，将 NVME 设备上的数据备份到 PC 上：
```bash
sudo dd if=/dev/sdc status=progress | bzip > bzip2.img.bz2
```
备份完成后，将 NVME 存储设备重新安装回拓展坞。

但实际上，升级 Jetson 板卡后，原有的数据和配置文件可能无法直接使用，因此只需要备份重要文件即可，*上述官方提供的备份方法是针对整个系统的备份，仅作参考*。

## 刷机升级
首先，需要将 Jetson 板卡设置为 recovery mode，具体步骤如下：
1. 关闭 GO2 EDU 机器人电源，或断开 XT30 电源接口，将板卡置于断电状态。
2. 将一根 USB-C 数据线连接到板卡上的 USB-C 接口和 PC（Ubuntu） 上的 USB 端口。
3. 将一根针（回形针、订书针或取卡器）插入板卡上的 recovery 按钮孔中（靠近 USB 接口）并按下按钮。
4. 在按住按钮的同时，按下 GO2 EDU 机器人的电源按钮，或接上 XT30 电源接口，保持按住 recovery 按钮约 2 秒钟左右后松开。
> 注意：XT30 的电源接口形状一端为方形，另一端为圆形，连接时请确保形状对应，防止接反导致设备损坏。
5. 使用 `lsusb` 命令[检查](https://docs.nvidia.com/jetson/archives/r38.2/DeveloperGuide/IN/QuickStart.html?ref=theroboverse.com#to-determine-whether-the-developer-kit-is-in-force-recovery-mode) PC 上是否识别到 Jetson 板卡，若有显示如下字样，则说明板卡已成功进入 recovery mode：
```
Bus <bbb> Device <ddd>: ID 0955: <nnnn> Nvidia Corp.
```
6. 下载 Jetson Orin NX 的刷机工具到 PC，链接如下：
https://disk.yandex.com/d/90OADxPeLx_ztg?ref=theroboverse.com
7. 使用 `sudo tar -xjf [filename].tar.bz2` 解压，注意解压需要预留大概 80GB 的磁盘空间。
8. 运行刷写脚本：
```bash
# For Orin Nano
cd JetPack_6.2.1_Linux_JETSON_ORIN_NANO_TARGETS/Linux_for_Tegra/
sudo ./tools/kernel_flash/l4t_initrd_flash.sh --external-device nvme0n1p1 \
-p "-c ./bootloader/generic/cfg/flash_t234_qspi.xml" -c ./tools/kernel_flash/ \
flash_l4t_t234_nvme.xml --showlogs --network usb0  jetson-orin-nano-devkit-super external

# For Orin NX
cd JetPack_6.2.1_Linux_JETSON_ORIN_NX_TARGETS/Linux_for_Tegra/
sudo ./tools/kernel_flash/l4t_initrd_flash.sh --external-device nvme0n1p1 \
-p "-c ./bootloader/generic/cfg/flash_t234_qspi.xml" -c ./tools/kernel_flash/ \
flash_l4t_t234_nvme.xml --showlogs --network usb0  jetson-orin-nano-devkit external
```

## 升级结束后
### 有线连接板卡
升级完成后，使用网线链接 PC 和 板卡，通过 `ssh unitree@192.168.123.18` 连接到板卡，默认密码为 `123`。这一过程要确保 PC 和板卡在`192.168.123.0:24`的同一网段内。

如果需要图形界面，也可以连接一个 USB-C 到 HDMI 的转接器，直接访问桌面环境。

根文件系统不包含完整的 Unitree 软件栈，但所有组件都可以通过[官方 Unitree 仓库](https://github.com/unitreerobotics?ref=theroboverse.com)进行安装。

运行以下命令查看升级后的相关版本：
```bash
cat /etc/os-release # R36 对应 Jetpack 6.2.1
nvidia-smi
```

### 网卡驱动安装

Unitree GO2 EDU 的 Jetson 板卡不带无线网卡，因此需要接入 USB 无线网卡来实现无线连接，且需要安装相应的驱动。

实验室网卡型号为 CM762-35264_USB，安装相关的驱动 CM762-35264_USB无线网卡驱动_V1.3（跟随驱动资料安装）。

安装完成后，连接 wifi，即可以通过 ssh 进行远程连接。

### 相关基础环境安装
> 注意板卡架构为 aarch64，安装软件时需要注意选择对应架构的版本。
1. 安装 ROS2 Humble：https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html
2. 安装 unitree_ros2：https://github.com/unitreerobotics/unitree_ros2
3. 安装 unitree_sdk2：https://github.com/unitreerobotics/unitree_sdk2
4. 部署 VPN，建议使用 clash for linux：https://github.com/nelvko/clash-for-linux-install

## 参考
- [Unitree Go2 EDU: Jetpack 6.2.1 Update](https://theroboverse.com/unitree-go2-edu-jetpack-6-2-1-update/)
https://theroboverse.com/unitree-go2-edu-jetpack-6-2-1-update/

- [Jetson Orin NX刷机指南](https://blog.csdn.net/qq_39599112/article/details/151679837)
https://blog.csdn.net/qq_39599112/article/details/151679837

- [GO2 SDK 开发指南](https://support.unitree.com/home/zh/developer)
https://support.unitree.com/home/zh/developer
