# Rysen MANUS Retargeting Runtime

This repository is the deployment package for Rysen MANUS glove to ApexHand
retargeting. It contains Docker Compose runtime configuration, calibration
storage, and user documentation.

The published runtime image includes ROS 2 Humble, the MANUS Integrated SDK,
MANUS ROS interfaces, MuJoCo/Mink IK retargeting, and rysen_apexhand_msgs.

## What the runtime does

The diagram below is the complete command path. This runtime container owns the
middle three stages: it receives MANUS glove poses, solves glove-to-hand inverse
kinematics, and publishes ROS 2 joint-follow commands. It does not connect to
or drive motor hardware directly; the separately deployed ApexHand driver
consumes those commands and communicates with the physical hand.

~~~text
MANUS glove + USB receiver
  -> /manus_glove_0, /manus_glove_1
  -> MANUS / MuJoCo IK retargeting
  -> /rysen/apexhand/ip_<HAND_IP>/move_j_position_follow_command
  -> separately deployed ApexHand driver
  -> APEX Hand
~~~

In detail:

1. The MANUS USB receiver delivers glove tracking data to the MANUS SDK.
2. manus_ros2 publishes that data on the /manus_glove_* ROS 2 topics.
3. The retargeting process converts glove pose to ApexHand joint targets using
   the MuJoCo hand model and Mink IK solver.
4. It publishes targets on the IP-specific
   /rysen/apexhand/ip_<HAND_IP>/move_j_position_follow_command topic.
5. rysen_apexhand, or another compatible ApexHand driver, subscribes to this
   topic and sends the actual network/control commands to the hand.

The container starts the teleoperation manager first. The MANUS data publisher
and each left/right retarget process are started only after a teleoperation
service request. Glove handedness is determined by ManusGlove.side; topic index
0 or 1 is not a handedness guarantee.

## Scope and prerequisites

This package provides the MANUS-to-ApexHand retargeting pipeline. It does not
include the ApexHand device driver. To move a physical hand, deploy
rysen_apexhand, or an equivalent driver, separately. That driver must subscribe
to the follow-command topic and use compatible ROS 2 DDS settings.

| Item | Requirement |
| --- | --- |
| Host architecture | **x86_64 / amd64 only.** arm64/aarch64 is not supported by this MANUS Integrated SDK release. |
| Host system | Linux; Ubuntu 22.04 is the tested platform. |
| Container runtime | Docker Engine and Docker Compose v2. |
| MANUS hardware | MANUS glove(s) and their USB receiver connected to the deployment host. |
| Hand driver | A separately running ApexHand driver able to reach the target hand IP. |
| ROS network | Identical ROS_DOMAIN_ID; normally ROS_LOCALHOST_ONLY=0 and RMW_IMPLEMENTATION=rmw_fastrtps_cpp across retargeting, driver, and callers. |
| Network | The hand-driver host must reach the hand IP. This container uses host networking for ROS 2 DDS. |

For stable teleoperation, use a mains-powered x86 PC with sufficient CPU
headroom. On an underpowered laptop, a browser/WebGUI, a MuJoCo viewer, or other
CPU-heavy workloads can delay IK and command publication.

## Deploy a published release

### 1. Clone and configure

~~~bash
git clone https://github.com/RysenRobotics/rysen-retargeting.git
cd rysen-retargeting
cp .env.example .env
~~~

Edit .env. At minimum, set ROS_DOMAIN_ID to the value used by the ApexHand
driver and all ROS clients. Pin an explicit release tag in production.

~~~dotenv
RYSEN_RETARGETING_IMAGE=ghcr.io/rysenrobotics/rysen-retargeting:0.1.0-amd64
ROS_DOMAIN_ID=111
ROS_LOCALHOST_ONLY=0
RMW_IMPLEMENTATION=rmw_fastrtps_cpp
~~~

### 2. Optional: configure a MANUS .mcal file

A .mcal file is a **MANUS device calibration file**, normally created for a
specific glove in MANUS software. It is not the same as the runtime open/fist
pose calibration described below.

If your MANUS setup requires it, place the file in calibration/:

~~~bash
cp /path/to/your_glove.mcal calibration/
~~~

Then set its path as seen inside the container:

~~~dotenv
MANUS_CALIBRATION_FILE=/data/calibration/your_glove.mcal
MANUS_CALIBRATION_GLOVE_ID=0x12345678
~~~

Leave both values empty when the glove does not need an .mcal file or an
explicit glove ID. The local calibration/ directory is mounted at
/data/calibration, and device-specific files are ignored by Git.

### 3. Pull and start

~~~bash
docker compose pull
docker compose up -d
docker compose logs -f rysen_retargeting
~~~

A healthy startup includes a message like:

~~~text
MANUS teleop manager ready at /rysen/apexhand/start_manus_teleop
~~~

At this point the manager is ready, but teleoperation is stopped. No glove
publisher or retarget process is started until a service request is made.

## Configuration reference

All user configuration is in .env.

| Variable | Default | Meaning |
| --- | --- | --- |
| RYSEN_RETARGETING_IMAGE | Versioned GHCR image | Runtime image to deploy. |
| ROS_DOMAIN_ID | 111 | Must match the hand driver and ROS clients. |
| ROS_LOCALHOST_ONLY | 0 | Keep 0 for cross-host ROS 2 communication. |
| RMW_IMPLEMENTATION | rmw_fastrtps_cpp | Normally must match the rest of the deployment. |
| MANUS_CALIBRATION_FILE | empty | In-container .mcal path, for example /data/calibration/glove.mcal. |
| MANUS_CALIBRATION_GLOVE_ID | empty | Corresponding glove ID in decimal or 0x hexadecimal form. |
| MANUS_RUNTIME_CALIBRATION_FILE | /data/calibration/manus_runtime_calibration.yaml | Persistent file created by runtime pose calibration. |
| ENABLE_VIEWER | false | Set true only for a local X11 MuJoCo viewer. |
| DISPLAY | :0 | X11 display, used only when the viewer is enabled. |
| MANUS_UDEV | 1 | Installs bundled MANUS udev rules inside the privileged container. |

The Compose service uses host networking, host IPC, and mounts /dev plus
/run/udev. These settings are required for ROS 2 DDS discovery and USB receiver
access.

## Teleoperation service

The manager provides this ROS 2 service:

| Service | Type |
| --- | --- |
| /rysen/apexhand/start_manus_teleop | rysen_apexhand_msgs/srv/StartTeleop |

The request field is command. Its required format is CODE:IP.

| Code | Action | Example |
| --- | --- | --- |
| 1 | Start left-hand teleoperation | 1:192.168.0.102 |
| 2 | Start right-hand teleoperation | 2:192.168.0.103 |
| 5 | Query the binding/status for a hand IP | 5:192.168.0.102 |
| 0 | Stop teleoperation bound to a hand IP | 0:192.168.0.102 |

A bare 1 or 2 is invalid; a target IP is always required. Left and right can
run at the same time if each is bound to its own hand IP. Starting a side again
rebinds that side only.

### Call from inside the runtime container

This is the recommended bring-up method because it needs no ROS installation or
message package on the host.

Enter a shell:

~~~bash
docker compose exec -it rysen_retargeting bash
~~~

Inside the container, source ROS once:

~~~bash
source /opt/ros/humble/setup.bash
source /opt/rysen-retargeting/install/setup.bash
~~~

Start left-hand teleoperation for a hand at 192.168.0.102:

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '1:192.168.0.102'}"
~~~

Start a right hand:

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '2:192.168.0.104'}"
~~~

Query status or stop a session:

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '5:192.168.0.102'}"

ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '0:192.168.0.102'}"
~~~

A successful start response includes success: true and normally states that
forwarding was established. The manager starts/reuses the MANUS publisher,
waits for data from the requested glove side, launches that side's retarget
process, and waits until a follow-command publisher appears.

For code 5, success: true means that the query succeeded; read message to learn
whether teleoperation is actually active and forwarding.

A successful request does not override hand-driver enablement, e-stop, limits,
or any other physical safety mechanism.

### Call from another ROS 2 host or a WebGUI backend

An external caller can invoke the same service only when:

1. It can discover the runtime container through ROS 2 DDS.
2. It uses the same ROS_DOMAIN_ID, ROS_LOCALHOST_ONLY=0, and compatible RMW/DDS
   transport configuration.
3. ROS 2 Humble and the rysen_apexhand_msgs interface are sourced there.

Example:

~~~bash
source /opt/ros/humble/setup.bash
source /path/to/rysen_apexhand_msgs/install/setup.bash

ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '1:192.168.0.102'}"
~~~

In particular, a WebGUI/backend with no ROS_DOMAIN_ID environment variable
usually defaults to domain 0. It cannot discover a manager running in domain
111.

## Runtime MANUS pose calibration

Runtime pose calibration adjusts the glove-to-hand relationship. It is
independent for left and right and is saved in
MANUS_RUNTIME_CALIBRATION_FILE, which defaults to:

~~~text
calibration/manus_runtime_calibration.yaml
~~~

This runtime YAML is different from a MANUS .mcal device calibration file.

All calibration services use rysen_apexhand_msgs/srv/ManusCalibration and
accept one field:

~~~yaml
{side: 'left'}
~~~

Use right for the right glove.

| Service | Action |
| --- | --- |
| /rysen/apexhand/start_manus_calibration | Begin calibration and disarm follow-command publication for that side. |
| /rysen/apexhand/calibrate_manus_open | Record thumb naturally open with the other four fingers together. |
| /rysen/apexhand/calibrate_manus_fist | Record a stable fist. |
| /rysen/apexhand/clear_manus_calibration | Remove the runtime calibration for one side. |

Enter the runtime container and source ROS as described above, then calibrate
the left hand in order.

1. Start the calibration session. The MANUS publisher is started if needed and
   physical hand command publication is disabled for the selected side.

   ~~~bash
   ros2 service call /rysen/apexhand/start_manus_calibration \
     rysen_apexhand_msgs/srv/ManusCalibration \
     "{side: 'left'}"
   ~~~

2. Hold the thumb naturally open and the remaining four fingers together. Keep
   the pose still, then record it.

   ~~~bash
   ros2 service call /rysen/apexhand/calibrate_manus_open \
     rysen_apexhand_msgs/srv/ManusCalibration \
     "{side: 'left'}"
   ~~~

3. Make a stable fist, then record it.

   ~~~bash
   ros2 service call /rysen/apexhand/calibrate_manus_fist \
     rysen_apexhand_msgs/srv/ManusCalibration \
     "{side: 'left'}"
   ~~~

4. Start teleoperation again. Calibration intentionally leaves the selected
   side disarmed.

   ~~~bash
   ros2 service call /rysen/apexhand/start_manus_teleop \
     rysen_apexhand_msgs/srv/StartTeleop \
     "{command: '1:192.168.0.102'}"
   ~~~

To discard the saved runtime calibration for one side:

~~~bash
ros2 service call /rysen/apexhand/clear_manus_calibration \
  rysen_apexhand_msgs/srv/ManusCalibration \
  "{side: 'left'}"
~~~

The response contains success, correction angle values, sampled tip positions,
a sample count, and a human-readable message. Do not continue to the next stage
if the previous request failed.

## Topics and verification

After left teleoperation is started for 192.168.0.102, the command topic is:

~~~text
/rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

In general:

~~~text
/rysen/apexhand/ip_<IP-with-dots-replaced-by-underscores>/move_j_position_follow_command
~~~

Inside the sourced container shell, inspect the graph and command rate:

~~~bash
ros2 node list
ros2 service list -t | grep /rysen/apexhand
ros2 topic list | grep -E 'manus|move_j_position_follow_command'

ros2 topic hz \
  /rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

To view command contents:

~~~bash
ros2 topic echo \
  /rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

Useful MANUS topics are /manus_glove_0, /manus_glove_1, and the manager's
handedness-demultiplexed topics /manus_teleop/glove_left and
/manus_teleop/glove_right.

## Operations and troubleshooting

### Lifecycle

~~~bash
# Follow logs.
docker compose logs -f rysen_retargeting

# Stop and remove the container. Files in ./calibration remain.
docker compose down

# Start it again.
docker compose up -d

# Refresh an already-pulled image tag and recreate the container.
docker compose pull
docker compose up -d --force-recreate
~~~

pull_policy: missing means docker compose up -d uses a locally available image
with the requested tag and pulls only when it is absent. Run docker compose pull
explicitly when a publisher replaces an existing image tag.

### Common problems

| Symptom | Checks and likely remedy |
| --- | --- |
| Service is not visible immediately | Wait a few seconds, inspect docker compose logs rysen_retargeting, then list services inside the container. |
| waiting for service to become available | The manager is not running, DDS/domain settings differ, or the external caller did not source rysen_apexhand_msgs. Test first from inside the container. |
| Start reports no matching glove data | Verify the USB receiver, glove connection, MANUS setup/calibration, and side code: 1 is left and 2 is right. |
| USB receiver is absent | On the host run lsusb and ls -l /dev/hidraw*. Reconnect the receiver, then inspect container logs. /dev and udev metadata are already passed into the privileged container. |
| Service succeeds but the hand does not move | Check follow-topic messages, ApexHand driver subscription, shared ROS_DOMAIN_ID, IP, driver enable/e-stop state, and hand network reachability. |
| Motion pauses or feels delayed | Measure the follow topic with ros2 topic hz and inspect host CPU load. Reduce browser/WebGUI/viewer load, connect laptop power, or use a more capable host if IK misses timing deadlines. |
| MuJoCo viewer does not open | Keep ENABLE_VIEWER=false for headless/remote use. Local viewing requires a valid X11 DISPLAY and container X11 permission. |
| Container restarts continuously | Run docker compose logs --tail=200 rysen_retargeting; check the image tag, .env syntax, image version, and configured calibration-file path. |

## Safety

1. Validate the pipeline with simulation or a disabled hand before first
   physical motion.
2. Confirm hand IP, requested side, namespace, joint direction, limits, and
   driver enable/e-stop state before enabling the hand.
3. Keep clear space around the hand and stop teleoperation with 0:IP before
   reconnecting hardware or changing configuration.
4. Do not assume a successful StartTeleop response bypasses physical safety
   mechanisms in the hand driver.

## License

See [LICENSE](LICENSE). MANUS and other third-party components remain subject
to their respective licenses.
