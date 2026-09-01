# Rysen Retargeting

[English](README.md) | [中文](README_zh.md)

该仓库提供 MANUS 手套到 ApexHand 灵巧手的重定向功能。目前支持：

- MANUS 手套作为输入设备
- 左右 ApexHand 灵巧手
- 通过 ROS 2 服务启动、停止和查看遥操作状态
- MANUS 手套的运行时姿态标定

该仓库只提供 Docker 镜像和部署配置。Docker 镜像已经包含 MANUS SDK、ROS 2、IK 求解和所需的 ROS 2 消息接口，使用者不需要额外挂载或编译 MANUS 动态库。

## 仓库结构

~~~text
.
├── calibration/                        # 运行时标定文件目录，挂载进 Docker
│   └── manus_runtime_calibration.yaml  # 保存的 MANUS 姿态标定（自动生成）
├── docs/
│   ├── USAGE.md                        # 使用说明（English）
│   └── USAGE_zh.md                     # 使用说明（中文）
├── docker-compose.yml                  # Docker Compose 部署配置
├── .env.example                        # 环境变量配置模板
├── LICENSE
├── README.md                           # 英文 README
└── README_zh.md                        # 本文件（中文）
~~~

## 前置需求

| 项目 | 说明 |
| --- | --- |
| 主机架构 | 仅支持 x86_64 / amd64。当前镜像不支持 arm64/aarch64。 |
| 系统 | 已在 Ubuntu 22.04 上验证。 |
| Docker | 需要 Docker Engine 和 Docker Compose v2。 |
| MANUS | 使用 MANUS 手套时，需要将 MANUS USB 接收器连接到运行 Docker 的主机。 |
| 控制对象 | 需要接入 ApexHand 实机驱动，或接入订阅同一 ROS 话题的仿真。 |
| ROS 2 网络 | Docker、ApexHand 驱动和 ROS 2 调用端需要使用相同的 ROS_DOMAIN_ID。通常 ROS_LOCALHOST_ONLY 应为 0，RMW_IMPLEMENTATION 应为 rmw_fastrtps_cpp。 |

> 没有接入实机灵巧手或仿真时，Docker 仍可启动，MANUS 数据和服务也可能正常，但没有控制对象接收关节命令。因此遥操作不会产生实际动作，标定也无法验证最终的手部效果。

左手的默认 IP 是 192.168.0.102，右手的默认 IP 是 192.168.0.103。实际使用时，IP 以部署的灵巧手或仿真配置为准。

## 快速开始

### 1. 获取仓库并创建配置

~~~bash
git clone https://github.com/RysenRobotics/rysen-retargeting.git
cd rysen-retargeting
cp .env.example .env
~~~

编辑 .env，至少确认 ROS_DOMAIN_ID 与 ApexHand 驱动一致：

~~~dotenv
RYSEN_RETARGETING_IMAGE=ghcr.io/rysenrobotics/rysen-retargeting:0.1.0-amd64
ROS_DOMAIN_ID=111
ROS_LOCALHOST_ONLY=0
RMW_IMPLEMENTATION=rmw_fastrtps_cpp
~~~

下面是 .env 中日常部署会用到的字段说明。

| 字段名 | 默认值 | 说明 |
| --- | --- | --- |
| RYSEN_RETARGETING_IMAGE | ghcr.io/rysenrobotics/rysen-retargeting:0.1.0-amd64 | 使用的 Docker 镜像和版本。 |
| ROS_DOMAIN_ID | 111 | 需要与 ApexHand 驱动和 ROS 2 调用端保持一致。 |
| ROS_LOCALHOST_ONLY | 0 | Docker 与其他主机上的 ROS 2 节点需要通信时，应保持为 0。 |
| RMW_IMPLEMENTATION | rmw_fastrtps_cpp | ROS 2 通信库，通常应与灵巧手驱动一致。 |
| MANUS_RUNTIME_CALIBRATION_FILE | /data/calibration/manus_runtime_calibration.yaml | 保存 MANUS 姿态标定的文件，实际保存在本地 calibration 目录中。 |
| ENABLE_VIEWER | false | 只有需要在本地打开 MuJoCo 窗口时才设为 true。 |
| DISPLAY | :0 | ENABLE_VIEWER 为 true 时使用的 X11 显示器。 |
| MANUS_UDEV | 1 | 让 Docker 尝试加载镜像内置的 MANUS USB 访问规则。 |

### 2. 拉取并启动 Docker

~~~bash
docker compose pull
docker compose up -d
docker compose logs -f rysen_retargeting
~~~

日志出现下面的内容，表示管理服务已经准备好：

~~~text
MANUS teleop manager ready at /rysen/apexhand/start_manus_teleop
~~~

此时 Docker 只启动了管理服务。手套数据发布和左右手的重定向进程会在调用启动服务后按需启动。

## 使用服务

完整的服务调用、姿态标定、话题检查和常见问题见：

[使用说明](docs/USAGE_zh.md) | [Usage guide (English)](docs/USAGE.md)

最简启动示例：进入 Docker 后调用左手服务。

~~~bash
docker compose exec -it rysen_retargeting bash

source /opt/ros/humble/setup.bash
source /opt/rysen-retargeting/install/setup.bash

ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '1:192.168.0.102'}"
~~~

## 常用 Docker 命令

~~~bash
# 查看日志
docker compose logs -f rysen_retargeting

# 停止 Docker。calibration/ 下的文件会保留。
docker compose down

# 再次启动
docker compose up -d

# 手动拉取当前标签的最新镜像，并重建 Docker
docker compose pull
docker compose up -d --force-recreate
~~~

docker-compose.yml 中的 pull_policy: missing 表示 Docker 本地已有同一镜像标签时，docker compose up -d 不会再次拉取。镜像发布者更新了同名标签后，可手动执行 docker compose pull。

## 许可证

见 [LICENSE](LICENSE)。MANUS 及其他第三方组件仍受其各自许可证约束。
