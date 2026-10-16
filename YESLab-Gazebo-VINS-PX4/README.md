# YESLab-Gazebo-VINS-PX4

## 一、实验环境
- 操作系统：Ubuntu 20.04
- ROS版本：ROS Noetic
- 飞控仿真：PX4 SITL
- 仿真引擎：Gazebo Classic 11
- 通信桥接：MAVROS
- 视觉惯性里程计：VINS-Fusion (适配 OpenCV 4 版本)
- 可视化：RViz

## 二、系统数据流
Gazebo (Camera + IMU) → VINS-Fusion (位姿估计) → MAVROS (External Vision) → PX4 EKF2 (状态融合) → Position (POSCTL) 模式。

**注**：由于 Gazebo 纯仿真环境的视觉纹理缺乏且存在物理引擎锁步机制，VINS-Fusion 在多次排查与参数调整后仍无法完成初始化。为了验证 MAVROS 到 PX4 的完整控制链路，实验中改用 Gazebo Ground Truth 进行 ENU 转 NED 坐标转换，模拟 External Vision 数据发送给 PX4。

## 三、实际使用的 Topic
- Camera: `/camera/image_raw`
- IMU: `/gazebo_imu/data` (由 Gazebo 原生 ROS 插件发布)
- VINS Pose/Odometry: `/vins_estimator/odometry` (仿真受限未能输出)
- PX4 External Vision: `/mavros/vision_pose/pose`
- PX4 Local Position: `/mavros/local_position/pose`

## 四、完整启动顺序
1. `roscore`
2. `rosparam set /use_sim_time true`
3. `make px4_sitl_default gazebo`
4. `roslaunch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557" _use_sim_time:=true`
5. `python3 scripts/vins_to_px4.py`
6. 在 QGC 中切换 `定点 Position` (POSCTL) 模式，解锁起飞。

## 五、VINS-Fusion 结果说明
由于 Gazebo 仿真环境地面纹理缺乏，且存在物理引擎锁步机制与时间戳跳变，VINS-Fusion 在仿真环境下未能完成初始化。
*(此处对应 images/vins_rviz.png 截图)*

## 六、PX4 定位结果
通过外部视觉数据接入，PX4 成功获得有效 Local Position。
*(此处对应 images/px4_local_position.png 截图)*

## 七、Position 模式结果
无人机在无 GPS 环境下成功进入 Position (POSCTL) 模式，并实现了稳定悬停与前后左右移动。
- 视频证据：详见 video/demo_link.txt
*(此处对应 images/position_mode.png 和 images/gazebo.png 截图)*

## 八、问题记录
- **现象**：VINS-Fusion 始终报 `IMU excitation not enough` 和 `Not enough features or parallax`，无法初始化。
- **原因**：Gazebo 纯色地面与模型缺乏视觉特征点；物理引擎锁步机制导致 IMU 数据污染；MAVROS 时间戳跳变导致数据无法对齐。
- **解决方法**：
    1. 修改 `iris.sdf.jinja` 添加 Gazebo ROS IMU 插件，解决数据来源问题；
    2. 开启全局 `use_sim_time` 并强制 VINS 节点读取仿真时间；
    3. 采用 Gazebo Ground Truth 模拟外部视觉数据，验证完整控制链路。
- **最终结果**：成功实现无 GPS 下 Position 模式稳定悬停、移动及高度变化。

## 九、任务总结
VINS-Fusion 负责估计位姿，PX4 负责位置控制。关闭 GPS 后仍可控制是因为 MAVROS 向 PX4 喂入了外部视觉数据，EKF2 融合这些数据替代了 GPS。

## 十、验收提问准备
（建议自行整理“十四、验收提问”中的16个问题及回答）
