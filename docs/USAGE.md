# Rysen Retargeting Usage

[English](USAGE.md) | [中文](USAGE_zh.md)

This guide describes the new MANUS manager image. Run these commands inside
the container so the required ROS 2 service types are already available.
After the manager reports it is ready, open another host terminal, or press
`Ctrl+C` to leave `docker compose logs -f` (the background container keeps
running), then run:

```bash
docker compose exec -it rysen_retargeting bash
source /opt/ros/humble/setup.bash
source /opt/rysen-retargeting/install/setup.bash
```

Run all subsequent `ros2 ...` commands in this container shell with the
environment loaded. On a first run, complete [New calibration](#new-calibration)
until `phase=complete` before starting [Teleoperation](#teleoperation).
If a completed profile already exists for that side, go straight to teleoperation.
The examples use the right hand and target ApexHand IP `192.168.0.103`; replace
this with the actual hand IP. For the left hand, change `right` to `left` in
calibration requests and use start code `1` instead of `2`. The IP in the
manager's ready log is an example, not a connected device.

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

## New calibration

Calibration needs a working X11 display. The manager opens a MuJoCo viewer and
later a Tk adjustment window. You can use `3xyz` or `9xyz`; `3xyz` is
recommended. This example starts a right-hand `3xyz` calibration:

```bash
ros2 service call /rysen/apexhand/start_manus_calibration_new \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{side: 'right', mode: '3xyz', calibration_id: 'operator_a'}"
```

Continue by checking the service response or `status` and calling `confirm`
when instructed. The response and `status` tell you the current `phase` and
next action:

```bash
ros2 service call /rysen/apexhand/manus_calibration_new/status \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{side: 'right'}"
```

For `3xyz`, hold each requested pose still, then call `confirm` once:

1. Four fingers flat: place the four fingers together and flat on the table, with the hook-and-loop strap beyond the table edge; `awaiting_four_finger_flat_trigger`.
2. Four-finger MCP flexion at 90°: lift the four fingers from the table, flex the MCP joints to 90°, and keep the middle and distal phalanges straight; `awaiting_four_finger_mcp_flexion_90_trigger`.
3. Cylinder grasp: the thumb must contact the index finger (exact opposition is unnecessary), and the index, middle, and ring finger marker blocks must form a line parallel to the Y axis; `awaiting_cylinder_grasp_trigger`.

```bash
ros2 service call /rysen/apexhand/start_manus_calibration_new \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{action: 'confirm', side: 'right'}"
```

After the three samples, `awaiting_manual_xyz_trigger` opens the Tk window.
Try pinching the thumb with the index, middle, and ring fingers. Using the
simulated model as a guide, adjust the thumb position relative to the four
fingers so that the model slightly interpenetrates during each pinch, then
click **Confirm** in the window. This step only accepts the XYZ adjustment;
the service then reports `awaiting_activation_trigger`. At that point, call
the same ROS `confirm` command one more time. Only when `phase=complete` is
returned has the JSON file been saved and selected as the active profile.

`9xyz` follows the same rule but contains six extra pinch/finger-root
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

## Common checks

| Situation | Check |
| --- | --- |
| Service is missing | Run `docker compose logs rysen_retargeting`; ensure the caller uses the same `ROS_DOMAIN_ID`. |
| USB receiver is not found | Run `lsusb` on the host. Keep `privileged`, `/dev:/dev`, and `/run/udev` in compose. After a fresh start, glove discovery can take tens of seconds. |
| Start fails with a missing calibration file | Complete calibration for that side and profile first. |
| Start succeeds but the hand does not move | Confirm the external SDK ROS backend is running, the correct hand is connected and enabled, and it subscribes to the `move_j_control_follow_command` topic. |
| No calibration GUI; log says `Authorization required` or `X11: Failed to open display` | Set `DISPLAY` in `.env` to the value of `echo "$DISPLAY"` in the desktop terminal, then run `docker compose up -d --force-recreate`. |
| XYZ adjustment cannot align all four fingers in X/Z simultaneously | Recalibrate. During `awaiting_cylinder_grasp_trigger`, keep the index, middle, and ring finger marker blocks aligned with the Y axis. Alternatively, compensate the affected finger during this pose: if its calibrated X or Z coordinate value is lower than desired, move that finger during capture in the direction that decreases the corresponding coordinate; if the value is higher than desired, move it in the direction that increases the corresponding coordinate. |
| XYZ adjustment cannot align all four fingers in Y simultaneously | Try `9xyz` calibration. |
| A gap remains during pinching | During XYZ adjustment, introduce slight interpenetration to maintain pinching force. XYZ adjustment changes the thumb position relative to the four fingers: Y moves the thumb directly, while X/Z moves the four fingers in the opposite direction to create the equivalent relative thumb displacement. |
