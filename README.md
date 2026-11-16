# Rysen Retargeting

[English](README.md) | [中文](README_zh.md)

Rysen Retargeting turns MANUS glove motion into ApexHand ROS 2 joint commands.
This release currently supports MANUS gloves and ApexHand hands.

The repository contains only deployment files. The Docker image contains the
MANUS SDK, the new retargeting algorithm, calibration tools, and ROS 2
packages. It does **not** contain the ApexHand SDK ROS backend.

## What is needed

| Item | Notes |
| --- | --- |
| CPU | x86_64 / amd64. |
| Docker | Docker Engine and Docker Compose v2. |
| MANUS | Connect the USB receiver to the Docker host. |
| ApexHand backend | A real hand or simulator needs a separate ROS 2 backend that subscribes to the command topic. |
| ROS 2 settings | The manager, hand backend, and ROS callers must use the same `ROS_DOMAIN_ID`. Usually `ROS_LOCALHOST_ONLY=0` and `RMW_IMPLEMENTATION=rmw_fastrtps_cpp`. |

This container only publishes commands. It does not connect a hand, enable its
fingers, or call the ApexHand SDK. If no external ROS 2 node subscribes to its
command topic, a teleoperation request can succeed but nothing will move. This
is expected.

For a PC deployment of the ApexHand SDK ROS backend, we recommend
[Rysen Explorer](https://github.com/RysenRobotics/Rysen_Explorer). Use it to
connect the hand and enable it before starting teleoperation here.

## Quick start

```bash
git clone https://github.com/RysenRobotics/rysen-retargeting.git
cd rysen-retargeting
cp .env.example .env
mkdir -p calibration manus_config logs
```

Edit `.env`. The important field is `ROS_DOMAIN_ID`: it must match the
separately deployed ApexHand backend. For example:

```dotenv
RYSEN_RETARGETING_IMAGE=ghcr.io/rysenrobotics/rysen-retargeting:0.2.1-amd64
ROS_DOMAIN_ID=111
ROS_LOCALHOST_ONLY=0
RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

Start the manager:

```bash
docker compose up -d
docker compose logs -f rysen_retargeting
```

The image is pulled automatically only when this tag is absent locally because
the compose file uses `pull_policy: missing`. To update an existing tag, run
`docker compose pull` before `docker compose up -d`.

The manager is ready when its log says `新版 MANUS manager 就绪`. At this point
it exposes ROS services but it does not yet read the glove, run IK, open a
viewer, or publish hand commands.

## Configuration

| Field | Default | Meaning |
| --- | --- | --- |
| `RYSEN_RETARGETING_IMAGE` | `ghcr.io/rysenrobotics/rysen-retargeting:0.2.1-amd64` | Image and release version. |
| `RYSEN_UID` / `RYSEN_GID` | `1000` / `1000` | Usually no change is needed. On a host where `id -u` / `id -g` are not `1000`, set these two optional fields to those values. |
| `ROS_DOMAIN_ID` | `111` | Must match the ApexHand ROS backend and callers. |
| `ROS_LOCALHOST_ONLY` | `0` | Keep `0` when ROS nodes need to communicate outside one container. |
| `RMW_IMPLEMENTATION` | `rmw_fastrtps_cpp` | ROS middleware; normally match the hand backend. |
| `DISPLAY` | empty | Needed only for graphical calibration. |
| `MUJOCO_GL` | `glfw` | MuJoCo graphics backend used by calibration. |

The compose file stores generated data in local ignored directories:

- `calibration/`: `*.json` calibration files and `active_profiles.json`.
- `manus_config/`: MANUS Core local settings created at runtime.
- `logs/`: manager and child-process logs.

The container runs with `privileged: true`, `/dev:/dev`, and a read-only
`/run/udev` mount. This is intentional: it is the USB setup verified with a
real MANUS receiver. Do not replace it with only `/dev/bus/usb`.

For graphical calibration, set `DISPLAY` in `.env` to the value printed by
`echo "$DISPLAY"` in the desktop terminal (for example `DISPLAY=:1`). The
manager runs as the same desktop user, so normal reboot/login does not require
an `xhost` command. After changing `.env`, recreate the manager once with
`docker compose up -d --force-recreate`.

The full service, calibration, topic, and troubleshooting guide is in
[docs/USAGE.md](docs/USAGE.md) | [中文说明](docs/USAGE_zh.md).

## Command target

For an ApexHand at `192.168.0.103`, this container publishes:

```text
/rysen/apexhand/ip_192_168_0_103/move_j_control_follow_command
```

The external ApexHand SDK ROS backend must subscribe to this topic and must
already have connected and enabled that hand. The old
`move_j_position_follow_command` topic is not used by this release.

## License

See [LICENSE](LICENSE). MANUS and other third-party software remain subject to
their own licenses.
