# Rysen Retargeting

[English](README.md) | [中文](README_zh.md)

This repository supports retargeting from MANUS gloves to ApexHand hands. It
currently supports:

- MANUS gloves as the input device
- Left and right ApexHand hands as the output device
- Start, stop, and check teleoperation through ROS 2 services
- MANUS pose calibration

This repository provides a Docker image and Docker Compose files. It does not
include the retargeting source code. The Docker image already contains the
MANUS SDK, ROS 2, IK tools, and the ROS 2 message packages. No extra MANUS
library needs to be mounted or built on the host.

## Repository Structure

~~~text
.
├── calibration/                        # Runtime calibration files, mounted into Docker
│   └── manus_runtime_calibration.yaml  # Saved MANUS pose calibration (generated)
├── docs/
│   ├── USAGE.md                        # Usage guide (English)
│   └── USAGE_zh.md                     # Usage guide (中文)
├── docker-compose.yml                  # Docker Compose deployment file
├── .env.example                        # Environment config template
├── LICENSE
├── README.md                           # This file (English)
└── README_zh.md                        # README (中文)
~~~

## Prerequisites

| Item | Notes |
| --- | --- |
| Host CPU | x86_64/amd64 only. arm64/aarch64 is not supported by this image. |
| System | Tested on Ubuntu 22.04. |
| Docker | Docker Engine and Docker Compose v2 are needed. |
| MANUS | Connect the MANUS USB receiver to the host that runs Docker. |
| Control target | Connect an ApexHand ROS backend for a real hand, or a simulator that subscribes to the same ROS topic. |
| ROS 2 network | Docker, the ApexHand driver, and ROS 2 callers need the same ROS_DOMAIN_ID. Usually ROS_LOCALHOST_ONLY is 0 and RMW_IMPLEMENTATION is rmw_fastrtps_cpp. |

> Without a real hand driver or a simulator, Docker can still start and MANUS
> data and services can still work. However, no control target receives the
> joint commands. Teleoperation will not move anything, and calibration cannot
> be checked on a hand.

> **Important:** This repository only publishes joint commands. It does not use
> the ApexHand SDK to connect a hand, enable it, or send motor commands. A
> successful StartTeleop call means that this repository can publish its ROS 2
> command topic. It does not mean that another ROS 2 node is receiving or
> processing that topic.

The default left-hand IP is 192.168.0.102. The default right-hand IP is
192.168.0.103. Use the IP address from your hand or simulator setup if it is
different.

## Quick Start

### 1. Get the repository and create a config file

~~~bash
git clone https://github.com/RysenRobotics/rysen-retargeting.git
cd rysen-retargeting
cp .env.example .env
~~~

Edit .env and make sure ROS_DOMAIN_ID is the same as the ApexHand driver:

~~~dotenv
RYSEN_RETARGETING_IMAGE=ghcr.io/rysenrobotics/rysen-retargeting:0.1.0-amd64
ROS_DOMAIN_ID=111
ROS_LOCALHOST_ONLY=0
RMW_IMPLEMENTATION=rmw_fastrtps_cpp
~~~

The table below explains the normal deployment fields in .env.

| Field | Default value | Meaning |
| --- | --- | --- |
| RYSEN_RETARGETING_IMAGE | ghcr.io/rysenrobotics/rysen-retargeting:0.1.0-amd64 | Docker image and version to use. |
| ROS_DOMAIN_ID | 111 | Must match the ApexHand driver and ROS 2 callers. |
| ROS_LOCALHOST_ONLY | 0 | Keep 0 when Docker and other ROS 2 nodes need to talk across hosts. |
| RMW_IMPLEMENTATION | rmw_fastrtps_cpp | ROS 2 communication library. It should normally match the hand driver. |
| MANUS_RUNTIME_CALIBRATION_FILE | /data/calibration/manus_runtime_calibration.yaml | File used to save MANUS pose calibration. It is stored in the local calibration directory. |
| MANUS_UDEV | 1 | Lets Docker try to load the bundled MANUS USB access rules. |

### 2. Pull and start Docker

~~~bash
docker compose pull
docker compose up -d
docker compose logs -f rysen_retargeting
~~~

The manager is ready when the log contains:

~~~text
MANUS teleop manager ready at /rysen/apexhand/start_manus_teleop
~~~

At this time Docker starts only the manager. The glove publisher and left/right
retargeting processes start after a start service call.

## Connect and enable the hand

Before teleoperation can move a real hand, an ApexHand ROS backend needs to
connect to that hand and enable it. That backend must subscribe to:

~~~text
/rysen/apexhand/ip_<IP with dots changed to underscores>/move_j_position_follow_command
~~~

For example, the left hand at 192.168.0.102 uses:

~~~text
/rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

If no ROS 2 backend or simulator subscribes to this topic, a StartTeleop
request can still return success: true, but the hand will not move. This is
normal: there is no node that receives and handles the command.

For a PC deployment of the ApexHand SDK and ROS backend, we recommend
[Rysen Explorer](https://github.com/RysenRobotics/Rysen_Explorer). After it is
deployed, use its front end to connect the target hand IP and power on/enable
the hand. Make sure its ROS_DOMAIN_ID matches this repository. After that, the
topic published by this repository has a node that can process it, and
teleoperation can be started from the command below.

## Use the service

The full guide for service calls, pose calibration, topics, and common problems
is here:

[Usage guide](docs/USAGE.md) | [中文使用说明](docs/USAGE_zh.md)

The shortest left-hand example is:

~~~bash
docker compose exec -it rysen_retargeting bash

source /opt/ros/humble/setup.bash
source /opt/rysen-retargeting/install/setup.bash

ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '1:192.168.0.102'}"
~~~

## Common Docker commands

~~~bash
# Show logs
docker compose logs -f rysen_retargeting

# Stop Docker. Files under calibration/ stay on the host.
docker compose down

# Start Docker again
docker compose up -d

# Pull the image again and recreate Docker
docker compose pull
docker compose up -d --force-recreate
~~~

pull_policy: missing in docker-compose.yml means docker compose up -d uses a
local image when the requested tag already exists. If a publisher updates the
same image tag, docker compose pull can fetch the new image.

## License

See [LICENSE](LICENSE). MANUS and other third-party components remain subject
to their respective licenses.
