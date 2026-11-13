# Rysen Retargeting

[English](README.md) | [中文](README_zh.md)

该仓库将 MANUS 手套动作重定向为 ApexHand 的 ROS 2 关节命令。目前支持 MANUS 手套和
ApexHand 灵巧手。

仓库只提供部署文件。Docker 镜像内已经包含 MANUS SDK、新版重定向算法、标定工具和 ROS 2
接口；镜像**不**包含 ApexHand SDK 的 ROS 后端。

## 前置条件

| 项目 | 说明 |
| --- | --- |
| CPU | x86_64 / amd64。 |
| Docker | Docker Engine 和 Docker Compose v2。 |
| MANUS | 将 USB 接收器连接到运行 Docker 的主机。 |
| ApexHand 后端 | 实机或仿真都需要独立的 ROS 2 后端订阅命令话题。 |
| ROS 2 配置 | manager、灵巧手后端和 ROS 调用端必须使用相同的 `ROS_DOMAIN_ID`。通常 `ROS_LOCALHOST_ONLY=0`，`RMW_IMPLEMENTATION=rmw_fastrtps_cpp`。 |

该容器只发布命令，不会连接灵巧手、使能手指，也不会调用 ApexHand SDK。如果没有外部 ROS 2
节点订阅命令话题，遥操作服务仍可能返回成功，但不会有任何设备动作；这是正常的。

如果需要在电脑上部署 ApexHand SDK ROS 后端，推荐使用
[Rysen Explorer](https://github.com/RysenRobotics/Rysen_Explorer)。请先在其中连接并使能灵巧手，
再回到本仓库启动遥操作。

## 快速开始

```bash
git clone https://github.com/RysenRobotics/rysen-retargeting.git
cd rysen-retargeting
cp .env.example .env
mkdir -p calibration manus_config logs
```

编辑 `.env`。最重要的是 `ROS_DOMAIN_ID`，它必须与独立部署的 ApexHand 后端一致。例如：

```dotenv
RYSEN_RETARGETING_IMAGE=ghcr.io/rysenrobotics/rysen-retargeting:0.2.0-amd64
ROS_DOMAIN_ID=111
ROS_LOCALHOST_ONLY=0
RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

启动 manager：

```bash
docker compose up -d
docker compose logs -f rysen_retargeting
```

`docker-compose.yml` 使用 `pull_policy: missing`：本地没有该标签时会自动拉取；本地已有同一
标签时不会重复拉取。如发布者更新了同名标签，请先执行 `docker compose pull`，再执行
`docker compose up -d`。

日志出现 `新版 MANUS manager 就绪` 时，服务已经可用。此时它不会读取手套、运行 IK、打开
Viewer，也不会发布灵巧手命令；这些进程会在调用服务后按需启动。

## 配置项

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| `RYSEN_RETARGETING_IMAGE` | `ghcr.io/rysenrobotics/rysen-retargeting:0.2.0-amd64` | Docker 镜像和版本。 |
| `RYSEN_UID` / `RYSEN_GID` | `1000` / `1000` | 一般无需设置；若 `id -u` / `id -g` 不是 `1000`，再将这两个可选字段设为对应数值。 |
| `ROS_DOMAIN_ID` | `111` | 必须与 ApexHand ROS 后端及调用端一致。 |
| `ROS_LOCALHOST_ONLY` | `0` | ROS 节点需要跨容器/主机通信时保持 `0`。 |
| `RMW_IMPLEMENTATION` | `rmw_fastrtps_cpp` | ROS 通信中间件，通常与灵巧手后端一致。 |
| `DISPLAY` | 空 | 只在图形标定时需要。 |
| `MUJOCO_GL` | `glfw` | 标定时 MuJoCo 使用的图形后端。 |

compose 会在本地生成并忽略下面的目录：

- `calibration/`：`*.json` 标定文件与 `active_profiles.json`。
- `manus_config/`：MANUS Core 运行时生成的本地配置。
- `logs/`：manager 和子进程日志。

容器使用 `privileged: true`、`/dev:/dev` 和只读 `/run/udev`。这是经真实 MANUS 接收器验证的
USB 配置；不要只挂载 `/dev/bus/usb`。

图形标定时，在桌面终端执行 `echo "$DISPLAY"`，将输出写入 `.env` 的 `DISPLAY`（例如
`DISPLAY=:1`）。manager 以同一桌面用户运行，因此正常重启/重新登录后不需要执行 `xhost`。修改
`.env` 后，只需执行一次 `docker compose up -d --force-recreate` 重建容器。

服务调用、标定、话题检查和常见问题见
[中文使用说明](docs/USAGE_zh.md) | [Usage guide](docs/USAGE.md)。

## 命令控制对象

当灵巧手 IP 为 `192.168.0.103` 时，该容器发布：

```text
/rysen/apexhand/ip_192_168_0_103/move_j_control_follow_command
```

外部 ApexHand SDK ROS 后端必须订阅该话题，并且已连接、使能这只手。新版不使用旧的
`move_j_position_follow_command` 话题。

## 许可证

见 [LICENSE](LICENSE)。MANUS 和其他第三方软件仍受各自许可证约束。
