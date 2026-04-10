# 快速开始

## 1. 准备环境变量

复制模板：

```bash
cp .env.example .env
```

按实际网络修改 `.env` 中的关键变量：

- `FRANKA_ROBOT_IP`
- `ROS_MASTER_URI`
- `ROS_IP` / `ROS_HOSTNAME`
- `ROS_LOG_DIR`
- `LOCAL_UID` / `LOCAL_GID`

## 2. 构建并进入容器

```bash
docker compose up --build -d
docker exec -it ros1-franka bash
```

说明：

- 所有 ROS1 依赖、编译工具链和 Franka 相关环境都在容器内
- 当前镜像已内置 `ros-noetic-foxglove-bridge`

## 3. 编译工作区

```bash
source /opt/ros/noetic/setup.bash
cd /workspace
catkin_make
source /workspace/devel/setup.bash
```

## 4. 启动最常用入口

手动实验主入口：

```bash
roslaunch on_orbit_bringup manual_experiment.launch
```

Gazebo 仿真：

```bash
roslaunch on_orbit_bringup manual_experiment.launch use_sim:=true
```

手动实验 + RViz：

```bash
roslaunch on_orbit_bringup manual_experiment.launch launch_visualization:=true
```

手动实验 + RViz + Foxglove bridge：

```bash
roslaunch on_orbit_bringup manual_experiment.launch \
  launch_visualization:=true \
  start_foxglove_bridge:=true
```

当前默认不会在 Foxglove 中暴露 ROS service 列表，这样可以避免 `foxglove_bridge` 因 `/franka_gripper/set_logger_level` 类型探测超时而反复报错。

自动闭环实验：

```bash
roslaunch on_orbit_bringup closed_loop_experiment.launch
```

## 5. 在另一个终端发送规划命令

先确保已经 `source /workspace/devel/setup.bash`，再发送：

如果你已经通过下面这种方式启动：

```bash
roslaunch on_orbit_bringup manual_experiment.launch \
  launch_visualization:=true \
  start_foxglove_bridge:=true
```

那么在发规划命令之前或运行中，也可以先切换 planner / controller：

```bash
rosservice call /on_orbit_supervisor/select_planner "planner_id: 'se3'"
rosservice call /on_orbit_supervisor/select_controller "controller_id: 'hqp'"
rostopic echo /on_orbit/mode_state
```

如果想一条命令直接切整套组合，可以改用：

```bash
rosservice call /on_orbit_supervisor/set_mission_mode "mode: 'HQP_SE3'"
```

绝对目标位姿：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=absolute \
  _use_waypoints:=false \
  _target_position:=[0.50,0.20,0.30] \
  _target_orientation:=[0.0,0.0,0.0,1.0]
```

相对位移：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=relative_to_current_ee \
  _use_waypoints:=false \
  _target_position:=[0.05,0.0,0.0]
```

保持当前位姿：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=hold_current_ee
```

## 6. 常用查看命令

ROS 日志：

```bash
ls -lah /workspace/log/ros
find /workspace/log/ros -maxdepth 2 -type f | sort
tail -f /workspace/log/ros/latest/*.log
```

实验数据日志：

```bash
ls -lah /workspace/closed
find /workspace/closed/imp_decoupled/latest -maxdepth 2 -type f | sort
find /workspace/closed/hqp_se3/latest -maxdepth 2 -type f | sort
```

控制器当前正在追踪的参考点：

```bash
rostopic echo /imp_controller/debug/active_reference_pose
rostopic echo /hqp_controller/debug/active_reference_pose
```

## 7. 论文图生成

闭环实验结束后，直接运行：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py
```

这条命令会自动：

- 清洗并对齐 `/workspace/closed/imp_decoupled/latest` 与 `/workspace/closed/hqp_se3/latest`
- 在 `/workspace/closed/publication/` 下生成论文风格 PNG/PDF 对比图

默认输出：

```bash
ls -lah /workspace/closed/publication
```

## 8. Foxglove 连接方式

在当前 `docker-compose` 采用 `network_mode: host` 的前提下，Foxglove Desktop 直接连接：

```text
ws://localhost:8765
```
