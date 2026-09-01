# Rysen Retargeting 使用说明

[English](USAGE.md) | [中文](USAGE_zh.md)

本文档介绍该仓库启动后的 ROS 2 服务、MANUS 姿态标定和常用检查方法。

## 进入 Docker

下面的命令都可以直接在 Docker 中执行，不需要在宿主机安装 ROS 2 消息包。

~~~bash
cd rysen-retargeting
docker compose exec -it rysen_retargeting bash

source /opt/ros/humble/setup.bash
source /opt/rysen-retargeting/install/setup.bash
~~~

也可以从另一台 ROS 2 主机调用这些服务。调用端需要：

- 与 Docker 使用相同的 ROS_DOMAIN_ID
- 设置 ROS_LOCALHOST_ONLY=0
- 使用兼容的 RMW_IMPLEMENTATION
- 已安装并 source rysen_apexhand_msgs

## ROS 服务

管理服务如下：

| 服务名 | 服务类型 |
| --- | --- |
| /rysen/apexhand/start_manus_teleop | rysen_apexhand_msgs/srv/StartTeleop |

请求格式为 CODE:IP。IP 不能省略。

| Code | 功能 | 示例 |
| --- | --- | --- |
| 1 | 启动左手遥操作 | 1:192.168.0.102 |
| 2 | 启动右手遥操作 | 2:192.168.0.103 |
| 5 | 查看指定 IP 的状态 | 5:192.168.0.102 |
| 0 | 停止指定 IP 的遥操作 | 0:192.168.0.102 |

### 左手

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '1:192.168.0.102'}"
~~~

### 右手

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '2:192.168.0.103'}"
~~~

### 查看状态

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '5:192.168.0.102'}"
~~~

对状态请求来说，success: true 只表示服务正确处理了请求。请查看 message 中的 active 和 forwarding，确认该手是否真正处于遥操作状态。

### 停止

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '0:192.168.0.102'}"
~~~

左手和右手可以同时运行。再次启动同一侧时，只会更新这一侧的绑定关系。

启动成功后，Docker 会启动或复用 MANUS 数据发布进程，等待正确侧的手套数据，再启动该侧的重定向进程。服务返回 success: true 后，才会开始发布该 IP 对应的关节命令。

## MANUS 姿态标定

姿态标定用于调整 MANUS 手套姿态到灵巧手姿态之间的关系。左手和右手分别保存。

标定结果默认保存在宿主机的下面位置：

~~~text
calibration/manus_runtime_calibration.yaml
~~~

标定时，选中侧的关节命令会被关闭。完成标定或清除标定后，需要**再次调用启动遥操作服务**，灵巧手才会继续接收命令。

标定服务都使用 rysen_apexhand_msgs/srv/ManusCalibration，参数为：

~~~yaml
{side: 'left'}
~~~

右手时，使用 {side: 'right'}。

| 服务名 | 功能 |
| --- | --- |
| /rysen/apexhand/start_manus_calibration | 开始标定，并关闭关节命令。 |
| /rysen/apexhand/calibrate_manus_open | 记录张开姿态。 |
| /rysen/apexhand/calibrate_manus_fist | 记录握拳姿态。 |
| /rysen/apexhand/clear_manus_calibration | 清除已保存的姿态标定。 |

以左手为例：

### 1. 开始标定

~~~bash
ros2 service call /rysen/apexhand/start_manus_calibration \
  rysen_apexhand_msgs/srv/ManusCalibration \
  "{side: 'left'}"
~~~

### 2. 记录张开姿态

拇指自然张开，其他四指并拢，保持稳定后调用：

~~~bash
ros2 service call /rysen/apexhand/calibrate_manus_open \
  rysen_apexhand_msgs/srv/ManusCalibration \
  "{side: 'left'}"
~~~

### 3. 记录握拳姿态

五指握拳并保持稳定后调用：

~~~bash
ros2 service call /rysen/apexhand/calibrate_manus_fist \
  rysen_apexhand_msgs/srv/ManusCalibration \
  "{side: 'left'}"
~~~

### 4. 重新启动遥操作

~~~bash
ros2 service call /rysen/apexhand/start_manus_teleop \
  rysen_apexhand_msgs/srv/StartTeleop \
  "{command: '1:192.168.0.102'}"
~~~

如果想清除左手标定：

~~~bash
ros2 service call /rysen/apexhand/clear_manus_calibration \
  rysen_apexhand_msgs/srv/ManusCalibration \
  "{side: 'left'}"
~~~

清除标定后，需要再次调用启动遥操作服务，灵巧手才会继续接收命令。

每个标定服务都会返回 success、采样数量和 message。上一步失败时，建议先处理 message 中的原因，再继续下一步。

## ROS 话题

左手使用 192.168.0.102 时，关节命令话题为：

~~~text
/rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

右手使用 192.168.0.103 时，关节命令话题为：

~~~text
/rysen/apexhand/ip_192_168_0_103/move_j_position_follow_command
~~~

话题格式如下：

~~~text
/rysen/apexhand/ip_<IP 中的点替换为下划线>/move_j_position_follow_command
~~~

可以查看命令频率：

~~~bash
ros2 topic hz \
  /rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

也可以直接查看命令内容：

~~~bash
ros2 topic echo \
  /rysen/apexhand/ip_192_168_0_102/move_j_position_follow_command
~~~

常用的 MANUS 话题包括：

~~~text
/manus_glove_0
/manus_glove_1
/manus_teleop/glove_left
/manus_teleop/glove_right
~~~

## 常见问题

| 现象 | 可以检查的内容 |
| --- | --- |
| 服务暂时找不到 | 先等待几秒，再用 docker compose logs rysen_retargeting 查看启动日志。也可以在 Docker 中执行 ros2 service list -t。 |
| 调用一直停在 waiting for service to become available | 管理服务可能没有启动，或调用端与 Docker 的 ROS_DOMAIN_ID 不同。可以先在 Docker 内调用一次，区分服务问题和跨主机发现问题。 |
| 返回 no matching glove data | 检查 USB 接收器、手套连接状态，并确认服务代码与手套侧别一致：1 为左，2 为右。 |
| 看不到 USB 接收器 | 在宿主机执行 lsusb 和 ls -l /dev/hidraw*，重新插拔接收器后再看 Docker 日志。 |
| 服务成功但手不动 | 检查关节命令话题是否有数据，ApexHand 驱动是否订阅该话题，ROS_DOMAIN_ID 是否一致，手的 IP 是否可达，以及急停和使能状态。 |
| 手的动作有停顿或延迟 | 用 ros2 topic hz 查看关节命令频率，并检查主机 CPU。关闭不必要的图形界面或高负载程序；笔记本建议接电源。 |
| Docker 一直重启 | 执行 docker compose logs --tail=200 rysen_retargeting，检查镜像标签、.env 格式和 calibration 目录挂载。 |
