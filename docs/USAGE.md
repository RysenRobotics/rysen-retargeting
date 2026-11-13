# Rysen Retargeting Usage

[English](USAGE.md) | [中文](USAGE_zh.md)

This guide describes the new MANUS manager image. Run these commands inside
the container so the required ROS 2 service types are already available:

```bash
docker compose exec -it rysen_retargeting bash
source /opt/ros/humble/setup.bash
source /opt/rysen-retargeting/install/setup.bash
```

The manager must share `ROS_DOMAIN_ID` with the separately deployed ApexHand
SDK ROS backend. Before controlling a real hand, use that backend (for example
Rysen Explorer) to connect the hand and enable its fingers.

## Services

| Service | Type | Purpose |
| --- | --- | --- |
| `/rysen/apexhand/start_manus_teleop` | `rysen_apexhand_msgs/srv/StartTeleop` | Start, stop, or inspect teleoperation. |
| `/rysen/apexhand/start_manus_calibration_new` | `teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew` | Start calibration or confirm one step. |
| `/rysen/apexhand/manus_calibration_new/status` | `teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew` | Read calibration state. |
| `/rysen/apexhand/manus_calibration_new/cancel` | `teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew` | Cancel calibration and stop its child processes. |

## Teleoperation

`StartTeleop` uses `CODE:IP`.

| Code | Meaning | Example |
| --- | --- | --- |
| `1` | Start left hand | `1:192.168.0.102` |
| `2` | Start right hand | `2:192.168.0.103` |
| `5` | Read one hand state; `5` without IP shows all states | `5:192.168.0.103` |
| `0` | Stop one hand; `0` without IP stops all sessions | `0:192.168.0.103` |

The old `11` and `21` no-opposition commands are not available in the new
retargeter. Use `1` or `2`.

Start the right hand:

```bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '2:192.168.0.103'}"
```

Check and stop it:

```bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '5:192.168.0.103'}"

ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '0:192.168.0.103'}"
```

A successful start starts MANUS reading, fingertip conversion, IK, and the
command bridge. The manager needs a completed calibration profile for that
side. It publishes `sensor_msgs/msg/JointState` at about 100 Hz to:

```text
/rysen/apexhand/ip_192_168_0_103/move_j_control_follow_command
```

Only an external ApexHand ROS backend decides whether the command is acted on.
The manager does not check hand connection or finger enable state.

Check the command stream:

```bash
ros2 topic info -v \
  /rysen/apexhand/ip_192_168_0_103/move_j_control_follow_command

ros2 topic hz \
  /rysen/apexhand/ip_192_168_0_103/move_j_control_follow_command
```

## New calibration

Calibration needs a working X11 display. The manager opens a MuJoCo viewer and
later a Tk adjustment window. First start a right-hand `3xyz` calibration:

```bash
ros2 service call /rysen/apexhand/start_manus_calibration_new \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{side: 'right', mode: '3xyz', calibration_id: 'operator_a'}"
```

The response and `status` tell you the current `phase` and next action:

```bash
ros2 service call /rysen/apexhand/manus_calibration_new/status \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{side: 'right'}"
```

For `3xyz`, hold each requested pose still, then call `confirm` once:

1. Flat hand: `awaiting_flat_trigger`.
2. Length reference: `awaiting_length_trigger`.
3. Natural cylindrical grip: `awaiting_cylinder_reference_trigger`.

```bash
ros2 service call /rysen/apexhand/start_manus_calibration_new \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{action: 'confirm', side: 'right'}"
```

After the three samples, `awaiting_manual_xyz_trigger` opens the Tk window.
Adjust if needed and click **Confirm** in that window. This only accepts the
XYZ adjustment. The service then reports `awaiting_activation_trigger`; call
the same ROS `confirm` command **one final time**. Only when `phase=complete`
is returned has the JSON file been saved and selected as the active profile.

`9xyz` follows the same rule but contains six extra opposition/finger-root
contact poses. Always use the `status` response as the source of truth.

Cancel an unfinished calibration:

```bash
ros2 service call /rysen/apexhand/manus_calibration_new/cancel \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{side: 'right'}"
```

Completed profiles are stored under `calibration/` on the host:

```text
operator_a_right_3xyz.json
active_profiles.json
```

The manager restores `active_profiles.json` after restart. A failed or
cancelled calibration does not replace the last successful profile.

## Common checks

| Situation | Check |
| --- | --- |
| Service is missing | Run `docker compose logs rysen_retargeting`; ensure the caller uses the same `ROS_DOMAIN_ID`. |
| USB receiver is not found | Run `lsusb` on the host. Keep `privileged`, `/dev:/dev`, and `/run/udev` in compose. After a fresh start, glove discovery can take tens of seconds. |
| Start fails with a missing calibration file | Complete calibration for that side and profile first. |
| Start succeeds but the hand does not move | Confirm the external SDK ROS backend is running, the correct hand is connected and enabled, and it subscribes to the `move_j_control_follow_command` topic. |
| No calibration GUI; log says `Authorization required` or `X11: Failed to open display` | Set `DISPLAY` in `.env` to the value of `echo "$DISPLAY"` in the desktop terminal, then run `docker compose up -d --force-recreate`. |
| Motion pauses | Check `ros2 topic hz` on the control-follow topic and host CPU load. |
