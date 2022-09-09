---
layout:     post
title:      "PX4容器化远程连接Airsim"
subtitle:   "px4的sitl容器化整理"
date:       2022-09-09
author:     "QuJun"
URL: "/2022/09/09/px4-docker-with-airsim/"
image:      "https://qiniu.bobozhu.cn/xihuLake.jpeg"
tags:
    - drones
    - px4
    - airsim

categories: [ UAV ]
---

## 前言

远程连接指Airsim作为服务器运行于云渲染环境，PX4作为客户端通过TCP协议进行远程连接。将PX4进程封装为容器运行，可以避免重复构建环境，保证运行环境的一致性，加快开发速度，是弹性大规模仿真训练的基础。

**本说明包含两部分内容：**

1. PX4容器化运行环境构建
2. Airsim的远程连接配置

**运行环境说明：**

| 组件 | 主机 | 版本 |
| --- | --- | --- |
| PX4-Autopilot | 192.168.6.221(ubuntu 18.04)       X86架构 | v1.13.0 |
| Airsim | 192.168.6.64(windows 10) | Master分支 |
| UE4 | 192.168.6.64(windows 10) | 4.26.2 |

## 1. **PX4容器化**

容器化分为两个步骤。以下步骤运行在Linux服务器上，并在ubuntu 18.04上通过了测试。

- 选择基础镜像
- 基于基础镜像构建软件在环应用镜像

### 1.1 PX4基础镜像

px4官方提供多种类型的基础镜像，根据不同类型预装了astyle、Gradle、Fast-DDS、ros、colcon、nuttx、JSBSim等开发人员所需的常见软件。

- [px4io/px4-dev-base-archlinux](https://hub.docker.com/r/px4io/px4-dev-base-archlinux)
- [px4io/px4-dev-base-bionic](https://hub.docker.com/r/px4io/px4-dev-base-bionic)
    - [px4io/px4-dev-clang](https://hub.docker.com/r/px4io/px4-dev-clang)
    - [px4io/px4-dev-nuttx-bionic](https://hub.docker.com/r/px4io/px4-dev-nuttx-bionic)
    - [px4io/px4-dev-nuttx-clang](https://hub.docker.com/r/px4io/px4-dev-nuttx-clang)
    - [px4io/px4-dev-raspi](https://hub.docker.com/r/px4io/px4-dev-raspi)
    - [px4io/px4-dev-simulation-bionic](https://hub.docker.com/r/px4io/px4-dev-simulation-bionic)
        - [px4io/px4-dev-ros-melodic](https://hub.docker.com/r/px4io/px4-dev-ros-melodic)
            - [px4io/px4-dev-ros2-dashing](https://hub.docker.com/r/px4io/px4-dev-ros2-dashing)
            - [px4io/px4-dev-ros2-eloquent](https://hub.docker.com/r/px4io/px4-dev-ros2-eloquent)
- [px4io/px4-dev-base-focal](https://hub.docker.com/r/px4io/px4-dev-base-focal)
    - [px4io/px4-dev-nuttx-focal](https://hub.docker.com/r/px4io/px4-dev-nuttx-focal)
    - [px4io/px4-dev-simulation-focal](https://hub.docker.com/r/px4io/px4-dev-simulation-focal)
        - [px4io/px4-dev-ros-noetic](https://hub.docker.com/r/px4io/px4-dev-ros-noetic)
            - [px4io/px4-dev-ros2-foxy](https://hub.docker.com/r/px4io/px4-dev-ros2-foxy)
    - [px4io/px4-dev-ros2-rolling](https://hub.docker.com/r/px4io/px4-dev-ros2-rolling)
    - [px4io/px4-dev-ros2-galactic](https://hub.docker.com/r/px4io/px4-dev-ros2-galactic)
- [px4io/px4-dev-armhf](https://hub.docker.com/r/px4io/px4-dev-armhf)
- [px4io/px4-dev-aarch64](https://hub.docker.com/r/px4io/px4-dev-aarch64)
- [px4io/px4-docs](https://hub.docker.com/r/px4io/px4-docs)

此次我们选择[px4io/px4-dev-ros2-foxy](https://hub.docker.com/r/px4io/px4-dev-ros2-foxy)作为基础镜像，该镜像使用ubuntu 20.04 系统制作，预装了astyle、Gradle、Fast-DDS、ros noetic以及ros foxy环境。

### 1.2 构建软件在环应用运行镜像

接下来我们编写运行容器的Dockerfile

> Dockerfile是可用于构建容器镜像的配置文件
> 
1. 创建/root/px4_docker目录；
2. 进入/root/px4_docker目录；
3. 执行以下命令，拉取PX4项目代码：

```bash
# 安装git命令
sudo apt-get install git

# 从github克隆PX4代码,通过--recursive参数，将依赖的submodule也一起拉取下来
git clone [https://github.com/PX4/PX4-Autopilot.git](https://github.com/PX4/PX4-Autopilot.git) --recursive

# 切换到v1.13.0分支
cd PX4-Autopilot
git checkout -b v1.13.0 v1.13.0
```

1. 新建一个Dockerfile文件，写入内容如下：

```docker
FROM [px4io/px4-dev-ros2-foxy](https://hub.docker.com/r/px4io/px4-dev-ros2-foxy):latest

# 将PX4-Autopilot/Tools/setup目录里ubuntu.sh和requirements.txt，拷贝至容器的/tmp目录下
ADD PX4-Autopilot/Tools/setup/ubuntu.sh  PX4-Autopilot/Tools/setup/requirements.txt /tmp/

# 执行ubuntu.sh脚本，安装px4编译相关的依赖包, 不安装nuttx和模拟器的包
RUN /tmp/ubuntu.sh --no-nuttx --no-sim-tools
```

1. 构建运行镜像：`docker build -t mypx4_image:v1 .`
2. 上条命令执行完成后，使用docker images命令将会看到mypx4_image镜像已经生成

### 1.3 使用容器镜像编译px4

1. 执行以下命令启动容器在后台运行，此容器将/root/px4_docker映射至其/src目录。

```docker
docker run -itd --privileged --env=LOCAL_USER_ID="$(id -u)" \
-v /root/px4_docker:/src:rw \
--network=host --name=mypx4-dev   mypx4_image:v1 /bin/bash
```

1. 登录容器

```docker
docker exec -it --user $(id -u) mypx4-dev bash
```

1. 在容器终端中执行编译命令

```bash
cd /src/PX4-Autopilot/
make px4_sitl_default none_iris
```

1. 输出类似以下内容则表示编译成功

```bash
SITL COMMAND: "/src/PX4-Autopilot/build/px4_sitl_default/bin/px4" "/src/PX4-Autopilot/build/px4_sitl_default"/etc -s etc/init.d-posix/rcS -t "/src/PX4-Autopilot"/test_data
Creating symlink /src/PX4-Autopilot/build/px4_sitl_default/etc -> /src/PX4-Autopilot/build/px4_sitl_default/tmp/rootfs/

_____  __   __    ___ 
| ___ \ \ \ / /   /   |
| |_/ /  \ V /   / /| |
|  __/   /   \  / /_| |
| |     / /^\ \ \___  |
\_|     \/   \/     |_/

px4 starting.

INFO  [px4] Calling startup script: /bin/sh etc/init.d-posix/rcS 0
INFO  [init] found model autostart file as SYS_AUTOSTART=10016
INFO  [param] selected parameter default file eeprom/parameters_10016
INFO  [parameters] BSON document size 401 bytes, decoded 401 bytes (INT32:16, FLOAT:5)
[param] Loaded: eeprom/parameters_10016
INFO  [dataman] data manager file './dataman' size is 7866640 bytes
PX4 SIM HOST: localhost
INFO  [simulator] Waiting for simulator to accept connection on TCP port 4560
```

1. 此时可以执行`exit`退出此容器，待下次有代码更新，再登录容器，重新编译

### 1.4 运行PX4

在上一步完成了PX4的编译后，接下来我们使用容器来启动PX4容器

1. 在/root/px4_docker创建Scripts目录
2. 进入Scripts目录，添加run_airsim_sitl.sh脚本，内容如下：

```bash
#!/bin/bash
# 判断当前环境是否有设置instance_num，如果没有则默认为0
[ ! -z "$instance_num" ] && instance_num="0"
export PX4_SIM_MODEL=iris
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
PARENT_DIR="$(dirname "$SCRIPT_DIR")"
BUILD_DIR=$PARENT_DIR/PX4-Autopilot/ROMFS/px4fmu_common
instance_path=$PARENT_DIR/PX4-Autopilot/build/px4_sitl_default
BIN_DIR=$PARENT_DIR/PX4-Autopilot/build/px4_sitl_default/bin/px4
TEST_DATA=$PARENT_DIR/PX4-Autopilot/test_data
working_dir="$instance_path/instance_$instance_num"
[ ! -d "$working_dir" ] && mkdir -p "$working_dir"
pushd "$working_dir" &>/dev/null
$BIN_DIR -i $instance_num $BUILD_DIR -s "etc/init.d-posix/rcS" -t $TEST_DATA
```

1. 启动容器

```bash
docker run -itd --privileged --env=LOCAL_USER_ID=1000 \
--env=instance_num=0 \
--env=PX4_SIM_HOST_ADDR=192.168.6.64 \
-v /home/danielqu/桌面/TDWorks/机载云/无人机:/src:rw \
 -v /tmp/.X11-unix:/tmp/.X11-unix:ro -e DISPLAY=:0 \
--network=host \
--name=mypx4-0  \
harbor.tiduyun.com/qujun/px4-dev-ros2-foxy:latest  \
/src/Scripts/run_airsim_sitl.sh
```

> --env=instance_num=0 容器添加instance_num环境变量，0表示当前为第1个px4实例
> 

> --env=PX4_SIM_HOST_ADDR=192.168.6.64 容器添加PX4_SIM_HOST_ADDR环境变量，指定远端airsim主机地址
> 

> -- name 后面指定此容器名称
> 

> 最后的/src/Scripts/run_airsim_sitl.sh 表示容器执行此命令启动
> 
1. 容器启动后，使用docker logs -f  mypx4-0将出现以下日志，当前px4处于等待连接服务端的状态

```bash
______  __   __    ___ 
| ___ \ \ \ / /   /   |
| |_/ /  \ V /   / /| |
|  __/   /   \  / /_| |
| |     / /^\ \ \___  |
\_|     \/   \/     |_/

px4 starting.

INFO  [px4] Calling startup script: /bin/sh etc/init.d-posix/rcS 0
INFO  [init] found model autostart file as SYS_AUTOSTART=10016
INFO  [param] selected parameter default file eeprom/parameters_10016
INFO  [parameters] BSON document size 359 bytes, decoded 359 bytes (INT32:14, FLOAT:5)
[param] Loaded: eeprom/parameters_10016
INFO  [dataman] data manager file './dataman' size is 7866640 bytes
PX4 SIM HOST: 192.168.6.64
INFO  [simulator] Simulator using TCP on remote host 192.168.6.64 port 4560
WARN  [simulator] Please ensure port 4560 is not blocked by a firewall.
INFO  [simulator] Waiting for simulator to accept connection on TCP port 4560
```

1. 如果需要启动多个px4运行实例，只需将上述的启动命令里的--env=instance_num改为其他数字，以及修改容器名称。px4将默认从4560端口开始连接airsim，并根据instance_num增加相应的值，如instance_num为2，则此实例将会连接4562端口。

## Airsim的PX4远程连接配置

在Airsim安装完成后，默认读取C:\Users{USER}\Documents\AirSim\settings.json文件，为Airsim提供默认的配置，如是否开启激光雷达，深度相机等，详细见[https://microsoft.github.io/AirSim/settings/](https://links.jianshu.com/go?to=https%3A%2F%2Fmicrosoft.github.io%2FAirSim%2Fsettings%2F)；

**针对于PX4场景的配置示例如下：**

```json
{
  "SeeDocsAt": "https://github.com/Microsoft/AirSim/blob/master/docs/settings.md",
  "SettingsVersion": 1.2,
  "SimMode": "Multirotor",
  "Vehicles": {
	"PX4-0": {
            "VehicleType": "PX4Multirotor",
            "UseSerial": false,
            "LockStep": true,
            "UseTcp": true,
			"ControlIp": "192.168.6.221",
			"LocalHostIp": "192.168.6.64",
            "TcpPort": 4560,
            "ControlPortLocal": 14540,
            "ControlPortRemote": 14580,
			"X": 0, "Y": 1, "Z": 0,
            "Sensors":{
                "Barometer":{
                    "SensorType": 1,
                    "Enabled": true,
                    "PressureFactorSigma": 0.0001825
                }
            },
            "Parameters": {
                "NAV_RCL_ACT": 0,
                "NAV_DLL_ACT": 0,
                "COM_OBL_ACT": 1,
                "LPE_LAT": 47.641468,
                "LPE_LON": -122.140165
            }
        },
			"PX4-1": {
            "VehicleType": "PX4Multirotor",
            "UseSerial": false,
            "LockStep": true,
            "UseTcp": true,
			"ControlIp": "192.168.6.221",
			"LocalHostIp": "192.168.6.64",
            "TcpPort": 4561,
            "ControlPortLocal": 14541,
            "ControlPortRemote": 14580,
			"X": 0, "Y": -1, "Z": 0,
            "Sensors":{
                "Barometer":{
                    "SensorType": 1,
                    "Enabled": true,
                    "PressureFactorSigma": 0.0001825
                }
            },
            "Parameters": {
                "NAV_RCL_ACT": 0,
                "NAV_DLL_ACT": 0,
                "COM_OBL_ACT": 1,
                "LPE_LAT": 47.641468,
                "LPE_LON": -122.140165
            }
        }
  }
}
```

其中重要字段说明如下：

1. Vehicles下的PX4-0、PX4-1为Airsim中启动载具名，在此字段下将会配置此载具的类型VehicleType类型，通信方式、出生点位、传感器等参数；
2. UseSerial设置为false,UseTcp设置为true，表示此载具使用TCP通信。
3. LocalHostIp、TcpPort设置当前载具的在airsim中监听的端口，用于给PX4连接；多个PX4连接时需要设置成对应的TcpPort端口。
4. ControlIp、ControlPortLocal、ControlPortRemote用于对接第三方的控制端口，如QGC等，此示例中可忽略；
5. LockStep是用于远程连接后时间同步，设为true。

将settings.json配置完成后，即可启动airsim。

此时前阶段的px4容器日志中将会出现：

```json
INFO  [simulator] Waiting for simulator to accept connection on TCP port 4560
INFO  [simulator] Simulator connected on TCP port 4560.
INFO  [commander] LED: open /dev/led0 failed (22)
INFO  [init] Mixer: etc/mixers/quad_w.main.mix on /dev/pwm_output0
INFO  [init] setting PWM_AUX_OUT none
INFO  [mavlink] mode: Normal, data rate: 4000000 B/s on udp port 18570 remote port 14550
INFO  [mavlink] mode: Onboard, data rate: 4000000 B/s on udp port 14580 remote port 14540
INFO  [mavlink] mode: Onboard, data rate: 4000 B/s on udp port 14280 remote port 14030
INFO  [mavlink] mode: Gimbal, data rate: 400000 B/s on udp port 13030 remote port 13280
INFO  [logger] logger started (mode=all)
INFO  [logger] Start file log (type: full)
INFO  [logger] [logger] ./log/2022-09-09/06_21_29.ulg	
INFO  [logger] Opened full log file: ./log/2022-09-09/06_21_29.ulg
INFO  [mavlink] MAVLink only on localhost (set param MAV_{i}_BROADCAST = 1 to enable network)
INFO  [mavlink] MAVLink only on localhost (set param MAV_{i}_BROADCAST = 1 to enable network)
INFO  [px4] Startup script returned successfully
pxh> INFO  [mavlink] partner IP: 192.168.6.64
INFO  [tone_alarm] home set
INFO  [tone_alarm] notify negative
```

在UE4的窗口中将会打印以下日志：

![UE4-connect-px4](https://qiniu.bobozhu.cn/blog%2FUE4-connect-px4.png)