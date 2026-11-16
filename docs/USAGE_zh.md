# Rysen Retargeting 使用说明

[English](USAGE.md) | [中文](USAGE_zh.md)

本文档介绍新版 MANUS manager 镜像。建议在容器内执行命令，这样不需要在宿主机安装 ROS 2
服务类型。manager 日志显示就绪后，另开一个宿主机终端，或按 `Ctrl+C` 退出
`docker compose logs -f`（不会停止后台容器），然后执行：

```bash
docker compose exec -it rysen_retargeting bash
source /opt/ros/humble/setup.bash
source /opt/rysen-retargeting/install/setup.bash
```

下文的 `ros2 ...` 命令都在这个已加载环境的容器终端中执行。首次使用请先完成
[新版标定](#新版标定)，待 `phase=complete` 后再[启动遥操作](#遥操作)；已有该侧成功标定的
profile 时可直接启动遥操作。以下示例使用右手、目标灵巧手 IP `192.168.0.103`；请替换为
实际 IP。使用左手时，将标定命令中的 `right` 改为 `left`，遥操作启动码从 `2` 改为 `1`。
manager 就绪日志里的 IP 是示例，不表示设备已连接。

manager 必须与独立部署的 ApexHand SDK ROS 后端使用相同的 `ROS_DOMAIN_ID`。控制实机前，请先通过
该后端（例如 Rysen Explorer）连接灵巧手并使能手指。

## 服务

| 服务 | 类型 | 作用 |
| --- | --- | --- |
| `/rysen/apexhand/start_manus_teleop` | `rysen_apexhand_msgs/srv/StartTeleop` | 启动、停止或查询遥操作。 |
| `/rysen/apexhand/start_manus_calibration_new` | `teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew` | 启动标定或确认一个步骤。 |
| `/rysen/apexhand/manus_calibration_new/status` | `teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew` | 查看标定状态。 |
| `/rysen/apexhand/manus_calibration_new/cancel` | `teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew` | 取消标定并回收其子进程。 |

## 新版标定

标定需要可用的 X11 显示。manager 会打开 MuJoCo Viewer，随后打开 Tk 微调窗口。可选择启动
`3xyz` 或`9xyz` 标定，推荐使用`3xyz` 。以右手`3xyz` 为例：

```bash
ros2 service call /rysen/apexhand/start_manus_calibration_new \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{side: 'right', mode: '3xyz', calibration_id: 'operator_a'}"
```

接下来持续启动服务响应 `status`与确认`confirm`，	
服务响应和 `status` 会告诉你当前 `phase` 与下一步：

```bash
ros2 service call /rysen/apexhand/manus_calibration_new/status \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{side: 'right'}"
```

`3xyz` 需要依次保持每个姿态稳定，然后各调用一次 `confirm`：

1. 四指平放：四指并拢平放于桌面上，魔术贴置于桌沿外；`awaiting_four_finger_flat_trigger`。
2. 四指近指关节弯曲 90°：四指离开桌面，近指关节弯曲 90°，中指节、远指节伸直；`awaiting_four_finger_mcp_flexion_90_trigger`。
3. 握持圆柱体：要求拇指与食指接触（无需正对），食指、中指、无名指定位块连线与 Y 轴平行；`awaiting_cylinder_grasp_trigger`。

```bash
ros2 service call /rysen/apexhand/start_manus_calibration_new \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{action: 'confirm', side: 'right'}"
```

三次采样后，`awaiting_manual_xyz_trigger` 会打开 Tk 窗口。
尝试做出拇指与食指、中指、无名指的捏合动作，依据仿真模型，调整拇指相对于四指的位置，使捏合时模型微微穿模，调整完成后在窗口中点击 **Confirm**。
这一步只接受 XYZ 微调；服务接着会显示 `awaiting_activation_trigger`。
此时还必须再调用一次同样的ROS `confirm` 命令。
只有返回 `phase=complete`，JSON 文件才真正保存并成为当前 profile。


`9xyz` 遵循同一规则，但会多出六个捏合/指根接触姿态。始终以 `status` 服务的响应为准。

取消未完成的标定：

```bash
ros2 service call /rysen/apexhand/manus_calibration_new/cancel \
  teleoperation_hand_manus_manager_msgs/srv/ManusCalibrationNew \
  "{side: 'right'}"
```

完成的 profile 保存在宿主机 `calibration/`：

```text
operator_a_right_3xyz.json
active_profiles.json
```

manager 重启后会恢复 `active_profiles.json`。失败或取消的标定不会覆盖之前成功的 profile。

## 遥操作

`StartTeleop` 使用 `CODE:IP` 格式。

| Code | 功能 | 示例 |
| --- | --- | --- |
| `1` | 启动左手 | `1:192.168.0.102` |
| `2` | 启动右手 | `2:192.168.0.103` |
| `5` | 查询指定手状态；不带 IP 的 `5` 查询所有状态 | `5:192.168.0.103` |
| `0` | 停止指定手；不带 IP 的 `0` 停止全部会话 | `0:192.168.0.103` |

旧版的 `11`、`21` 无对指命令在新版中没有对应实现，请使用 `1` 或 `2`。

启动右手：

```bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '2:192.168.0.103'}"
```

查询和停止：

```bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '5:192.168.0.103'}"

ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '0:192.168.0.103'}"
```

启动成功后，manager 会启动 MANUS 读取、指尖转换、IK 和命令桥接。该侧必须已有成功标定的
profile。它会以约 100 Hz 向以下话题发布 `sensor_msgs/msg/JointState`：

```text
/rysen/apexhand/ip_192_168_0_103/move_j_control_follow_command
```

只有外部 ApexHand ROS 后端决定是否执行这些命令；manager 不检查手是否已连接或手指是否已使能。

检查命令流：

```bash
ros2 topic info -v \
  /rysen/apexhand/ip_192_168_0_103/move_j_control_follow_command

ros2 topic hz \
  /rysen/apexhand/ip_192_168_0_103/move_j_control_follow_command
```

## 常见检查

| 现象 | 检查方式 |
| --- | --- |
| 找不到服务 | 执行 `docker compose logs rysen_retargeting`；确认调用端的 `ROS_DOMAIN_ID` 一致。 |
| 看不到 USB 接收器 | 在宿主机执行 `lsusb`。不要移除 compose 中的 `privileged`、`/dev:/dev`、`/run/udev`。刚启动时手套发现可能需要数十秒。 |
| 启动提示缺少标定文件 | 先完成该侧、该 profile 的标定。 |
| 服务成功但手不动 | 确认外部 SDK ROS 后端正在运行，正确 IP 的手已连接和使能，并订阅 `move_j_control_follow_command`。 |
| 不显示标定界面；日志有 `Authorization required` 或 `X11: Failed to open display` | 在桌面终端执行 `echo "$DISPLAY"`，将输出写入 `.env` 的 `DISPLAY`，再执行 `docker compose up -d --force-recreate`。 |
| 四指X与Z方向经过XYZ微调仍无法同时满足 | 重新标定，注意`awaiting_cylinder_grasp_trigger`中 食指、中指、无名指定位块连线与 Y 轴平行。或进行如下调整：“若某一手指在标定后X/Z相较期望偏小，该步标定时向X/Z减小方向调整对应手指。若某一手指在标定后X/Z相较期望偏大，该步标定时向X/Z增大方向调整对应手指。”|
| 四指Y方向经过XYZ微调仍无法同时满足  | 尝试使用9xyz标定。|
| 捏合时存在间隙 |XYZ微调过程调节为微小穿模，以保持捏合力度。 XYZ微调为调整拇指与四指的相对位置，Y 直接改变拇指，X/Z 通过四指沿相反方向移动来形成等效的拇指相对位移。|
