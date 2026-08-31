# Rysen MANUS Retargeting Runtime

This repository is the deployment package for MANUS-to-ApexHand retargeting.
It deliberately contains no retargeting source tree.  The executable runtime
is distributed as a versioned Docker image built by the private build pipeline.

## What runs

```text
MANUS glove receiver
  -> /manus_glove_*
  -> MANUS/MuJoCo IK retargeting
  -> /rysen/apexhand/ip_<HAND_IP>/move_j_position_follow_command
  -> external rysen_apexhand driver
```

The container launches the MANUS teleop manager only.  A `StartTeleop` request
then starts the glove publisher and the retarget process for the requested hand.

## Requirements

- Linux x86_64 host, Docker Engine and Docker Compose v2.
- A separately deployed ApexHand driver on the same DDS network.
- Matching `ROS_DOMAIN_ID`, `ROS_LOCALHOST_ONLY=0`, and
  `RMW_IMPLEMENTATION=rmw_fastrtps_cpp`.
- MANUS USB receiver connected to the host.

`calibration/*.mcal` is optional and ignored by Git.  Put a glove-specific file
there only when it is required by your MANUS setup.

## Deploy a published release

```bash
git clone https://github.com/RysenRobotics/rysen-retargeting.git
cd rysen-retargeting
cp .env.example .env
# Edit RYSEN_RETARGETING_IMAGE and ROS_DOMAIN_ID.
docker compose pull
docker compose up -d
```

Check the manager:

```bash
docker logs -f rysen_retargeting
```

Start left-hand teleoperation for an ApexHand at `192.168.0.102` from a ROS 2
environment that has `rysen_apexhand_msgs` installed and shares the same domain:

```bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '1:192.168.0.102'}"
```

Use `2:<IP>` for the right hand and `0:<IP>` to stop.
