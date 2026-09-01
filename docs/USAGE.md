# Rysen Retargeting Usage Guide

[English](USAGE.md) | [中文](USAGE_zh.md)

This guide explains the ROS 2 services, MANUS pose calibration, and basic topic
checks after Docker has started.

> This repository only publishes joint commands. It does not connect or enable
> an ApexHand through the SDK. StartTeleop can return success: true even when no
> hand moves, because it does not check whether another ROS 2 node subscribes
> to the command topic. A hand ROS backend or simulator must subscribe to the
> topic and handle the command.

For a PC setup with the ApexHand SDK and ROS backend, we recommend
[Rysen Explorer](https://github.com/RysenRobotics/Rysen_Explorer). Connect the
hand and power it on/enable it there before starting teleoperation here. Its
ROS_DOMAIN_ID needs to match this repository.

## Enter Docker

All commands in this guide can run inside Docker. The host does not need ROS 2
message packages for this method.

~~~bash
cd rysen-retargeting
docker compose exec -it rysen_retargeting bash

source /opt/ros/humble/setup.bash
source /opt/rysen-retargeting/install/setup.bash
~~~

Another ROS 2 host can also call these services when it uses the same
ROS_DOMAIN_ID, ROS_LOCALHOST_ONLY=0, and compatible RMW settings. It also needs
the rysen_apexhand_msgs package.

## ROS Services

The manager service is:

| Service name | Service type |
| --- | --- |
| /rysen/apexhand/start_manus_teleop | rysen_apexhand_msgs/srv/StartTeleop |

The request format is CODE:IP. The IP address is required.

| Code | Action | Example |
| --- | --- | --- |
| 1 | Start left-hand teleoperation | 1:192.168.0.102 |
| 2 | Start right-hand teleoperation | 2:192.168.0.103 |
| 5 | Check the state for a hand IP | 5:192.168.0.102 |
| 0 | Stop teleoperation for a hand IP | 0:192.168.0.102 |

### Start the left hand

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '1:192.168.0.102'}"
~~~

### Start the right hand

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '2:192.168.0.103'}"
~~~

### Check state

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '5:192.168.0.102'}"
~~~

For state checking service, success: true only means that the service handled the request.
Check active and forwarding in message to see whether the hand is really
running.

### Stop teleoperation

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '0:192.168.0.102'}"
~~~

The left and right hands can run at the same time. Starting one side again only
changes the binding for that side.

After a successful start request, Docker starts or reuses the MANUS publisher,
waits for data from the selected glove side, then starts the retargeting process
for that side. It starts publishing joint commands only after these steps are
ready.

## MANUS pose calibration

Pose calibration adjusts the link between the MANUS glove pose and the hand
pose. Left and right are saved separately.

The pose calibration file is saved on the host by default:

~~~text
calibration/manus_runtime_calibration.yaml
~~~

During calibration, joint commands for the selected side are turned off. After
calibration is complete or cleared, **start teleoperation again** before the hand
can receive commands.

All calibration services use rysen_apexhand_msgs/srv/ManusCalibration. The
request has one field:

~~~yaml
{side: 'left'}
~~~

Use {side: 'right'} for the right glove.

| Service name | Action |
| --- | --- |
| /rysen/apexhand/start_manus_calibration | Start calibration and turn off commands. |
| /rysen/apexhand/calibrate_manus_open | Save the open pose. |
| /rysen/apexhand/calibrate_manus_fist | Save the fist pose. |
| /rysen/apexhand/clear_manus_calibration | Clear saved pose calibration. |

The following example is for the left hand.

### 1. Start calibration

~~~bash
ros2 service call /rysen/apexhand/start_manus_calibration \
  rysen_apexhand_msgs/srv/ManusCalibration \
  "{side: 'left'}"
~~~

### 2. Save the open pose

Keep the thumb naturally open and the other four fingers together. Hold still,
then call:

~~~bash
ros2 service call /rysen/apexhand/calibrate_manus_open \
  rysen_apexhand_msgs/srv/ManusCalibration \
  "{side: 'left'}"
~~~

### 3. Save the fist pose

Make a fist and hold still, then call:

~~~bash
ros2 service call /rysen/apexhand/calibrate_manus_fist \
  rysen_apexhand_msgs/srv/ManusCalibration \
  "{side: 'left'}"
~~~

### 4. Start teleoperation again

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '1:192.168.0.102'}"
~~~

To clear saved calibration for the left hand:

~~~bash
ros2 service call /rysen/apexhand/clear_manus_calibration \
  rysen_apexhand_msgs/srv/ManusCalibration \
  "{side: 'left'}"
~~~

After clearing the calibration, call the start_manus_teleop service again
before the hand resumes receiving commands.

Each calibration response includes success, sample count, and message. If one
step fails, read message and fix the problem before the next step.

## ROS Topics

For the left hand at 192.168.0.102, the command topic is:

~~~text
/rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

For the right hand at 192.168.0.103, the command topic is:

~~~text
/rysen/apexhand/ip_192_168_0_103/move_j_position_follow_command
~~~

The general form is:

~~~text
/rysen/apexhand/ip_<IP with dots changed to underscores>/move_j_position_follow_command
~~~

Check command frequency:

~~~bash
ros2 topic hz \
  /rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

Show command messages:

~~~bash
ros2 topic echo \
  /rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

Useful MANUS topics are:

~~~text
/manus_glove_0
/manus_glove_1
/manus_teleop/glove_left
/manus_teleop/glove_right
~~~

## Common problems

| Problem | Things to check |
| --- | --- |
| The service is not visible yet | Wait a few seconds, check docker compose logs rysen_retargeting, then run ros2 service list -t inside Docker. |
| The call waits for the service | The manager may not be running, or the caller and Docker use different ROS_DOMAIN_ID values. Test the call inside Docker first. |
| Start reports no matching glove data | Check the USB receiver, glove connection, and selected side: 1 is left and 2 is right. |
| The USB receiver is not found | On the host, run lsusb and ls -l /dev/hidraw*. Reconnect the receiver and check Docker logs. |
| The service succeeds but the hand does not move | This can be normal when no hand ROS backend or simulator subscribes to the joint command topic. Connect and enable the hand through its SDK backend first; Rysen Explorer is the recommended PC deployment. Then check the topic, driver subscription, ROS_DOMAIN_ID, hand IP, e-stop, and enable state. |
| Motion pauses or feels slow | Check the command rate with ros2 topic hz and check host CPU use. Closing heavy programs and connecting laptop power may help. |
| Docker restarts | Run docker compose logs --tail=200 rysen_retargeting and check the image tag, .env file, and calibration directory mount. |
