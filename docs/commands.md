# 运行与命令

## 常用入口

### 手动实验（默认 imp + decoupled）

```bash
roslaunch on_orbit_bringup manual_experiment.launch
```

适合：

- 在线发 `absolute / relative / hold_current_ee` 命令
- 运行中用新轨迹立即抢占旧轨迹

### 手动实验(hqp 控制器)

```bash
roslaunch on_orbit_bringup manual_experiment.launch   launch_visualization:=true   start_foxglove_bridge:=true   initial_mode:=HQP_SE3   initial_planner:=se3   initial_controller:=hqp
```

### 自动闭环实验

```bash
roslaunch on_orbit_bringup closed_loop_experiment.launch
```

### 仅规划层调试

```bash
roslaunch on_orbit_bringup planning_stack.launch
```

### 仅运行时可视化

```bash
roslaunch on_orbit_bringup runtime_visualization.launch
```

### Gazebo 仿真

```bash
roslaunch on_orbit_bringup manual_experiment.launch use_sim:=true
```

## 手动实验常用参数

带 RViz：

```bash
roslaunch on_orbit_bringup manual_experiment.launch launch_visualization:=true
```

带 Foxglove bridge：

```bash
roslaunch on_orbit_bringup manual_experiment.launch \
  launch_visualization:=true \
  start_foxglove_bridge:=true
```

说明：

- 当前默认关闭 Foxglove 的 ROS service introspection，所以不会再因为 `/franka_gripper/set_logger_level` 打出无意义错误
- 如果你确实需要在 Foxglove 中浏览 ROS service，可额外覆盖：

```bash
roslaunch on_orbit_bringup manual_experiment.launch \
  launch_visualization:=true \
  start_foxglove_bridge:=true \
  foxglove_capabilities:="[clientPublish,parameters,parametersSubscribe,services,connectionGraph,assets]" \
  foxglove_service_whitelist:="['.*']"
```

单独运行可视化并开 Foxglove：

```bash
roslaunch on_orbit_bringup runtime_visualization.launch start_foxglove_bridge:=true
```

## `planner_demo.py` 常用命令

`planner_demo.py` 支持三种 target type，其中 `absolute` 和 `relative_to_current_ee` 是主要的两种运动模式，均支持单目标和多航点输入。`hold_current_ee` 是快捷保持指令。

### 绝对目标 (`absolute`)

指定世界坐标系下的目标位姿。

**单目标 decoupled**（自动以当前位姿为起点）：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=decoupled \
  _target_type:=absolute \
  _target_position:=[0.599,-0.0607,0.65] \
  _target_orientation:=[0.94341322,0.33161950,0.0,0.0]
```

**单目标 se3**（自动以当前位姿为起点）：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=absolute \
  _target_position:=[0.599,-0.0607,0.65] \
  _target_orientation:=[0.94341322,0.33161950,0.0,0.0]
```

**省略部分输入**（未指定的分量保持当前值）：

```bash
# 仅改变位置，姿态保持不变
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=absolute \
  _target_position:=[0.50,0.20,0.30]

# 仅改变姿态，位置保持不变
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=absolute \
  _target_orientation:=[0.0,0.0,0.7071068,0.7071068]
```

**多航点**：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=absolute \
  _use_waypoints:=true \
  '_waypoints:=[{"position": [0.4, 0.0, 0.4], "orientation": [0.0, 0.0, 0.0, 1.0]}, {"position": [0.5, 0.2, 0.3], "orientation": [0.0, 0.0, 0.7071068, 0.7071068]}]'
```

**多航点 + 自动插补当前位姿为起点**：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=absolute \
  _use_waypoints:=true \
  _prepend_current_pose:=true \
  '_waypoints:=[{"position": [0.5, 0.2, 0.3], "orientation": [0.0, 0.0, 0.0, 1.0]}]'
```

### 相对目标 (`relative_to_current_ee`)

相对于当前末端执行器位姿的增量运动。
其中平移增量在世界/基坐标系下解释；姿态增量使用 3 维 so3 旋转向量输入，经指数映射后按右乘方式更新锚点姿态。

**单目标 — 仅平移**：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=relative_to_current_ee \
  _target_position:=[0.05,0.0,0.0]
```

**单目标 — 平移 + 旋转**：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=relative_to_current_ee \
  _target_position:=[0.00,0.00,0.05] \
  _target_orientation:=[0.0,0.0,0.7853982]
```

**省略输入**（省略 = 零增量 = 保持不变）：

```bash
# 不提供 position 和 orientation，等效于 hold_current_ee
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=relative_to_current_ee
```

**多航点（相对增量航点）**：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=relative_to_current_ee \
  _use_waypoints:=true \
  '_waypoints:=[{"position": [0.05, 0.0, 0.0], "orientation": [0.0, 0.0, 0.0]}, {"position": [0.0, 0.05, 0.0], "orientation": [0.0, 0.0, 0.0]}]'
```

> 相对模式的多航点会自动将上一条规划轨迹末端或当前位姿作为锚点，各航点的 `position` 是平移增量，`orientation` 是 3 维 so3 旋转增量。

### 保持当前位姿 (`hold_current_ee`)

快捷指令，读取当前机器人位姿并发布一条保持轨迹：

```bash
rosrun on_orbit_apps planner_demo.py \
  _planner_id:=se3 \
  _target_type:=hold_current_ee
```

### 自动插补当前位姿 (`_prepend_current_pose`)

| 场景 | 默认行为 | 说明 |
|---|---|---|
| 单目标 `absolute` | ✅ 自动插补 | planner 读取当前位姿作为起点 |
| 单目标 `relative` | ✅ 自动锚定 | 优先使用上一条规划轨迹末端；若无历史轨迹则回退到当前位姿 |
| 多航点 `absolute` | 由参数控制 | `_prepend_current_pose:=true` 时插补，`false` 时直接用第一个航点 |
| 多航点 `relative` | ✅ 自动锚定 | 优先使用上一条规划轨迹末端；若无历史轨迹则回退到当前位姿 |

### 参数参考

| 参数 | 默认值 | 说明 |
|---|---|---|
| `_planner_id` | `se3` | planner 后端：`se3` 或 `decoupled` |
| `_target_type` | `absolute` | `absolute` / `relative_to_current_ee` / `hold_current_ee` |
| `_use_waypoints` | `false` | 是否使用多航点模式 |
| `_prepend_current_pose` | `true` | 是否自动将当前位姿插入为第一个航点 |
| `_target_position` | — | 单目标位置 `[x, y, z]`；absolute 省略则保持当前位置，relative 省略则为 `[0,0,0]` |
| `_target_orientation` | — | `absolute` 下是四元数 `[qx, qy, qz, qw]`；`relative_to_current_ee` 下是 3 维 so3 向量 `[rx, ry, rz]`（rad）；省略则保持当前/零增量 |
| `_waypoints` | 内置 3 点 | 多航点列表。`absolute` 下每个 `orientation` 是四元数；`relative_to_current_ee` 下每个 `orientation` 是 3 维 so3 向量 |
| `_preferred_controller` | `hqp`(se3) / `imp` | 首选控制器 |
| `_v_max` | — | 显式覆盖 planner 的最大线速度 (m/s)；省略则沿用 planner 配置默认值 |
| `_w_max` | — | 显式覆盖 planner 的最大角速度 (rad/s)；省略则沿用 planner 配置默认值 |
| `_a_max` | — | 显式覆盖 planner 的最大线加速度 (m/s²)；省略则沿用 planner 配置默认值 |
| `_alpha_max` | — | 显式覆盖 planner 的最大角加速度 (rad/s²)；省略则沿用 planner 配置默认值 |
| `_segment_durations` | — | 各段持续时间（仅 decoupled 后端） |
| `_hold_position` | `true` | 到达终点后是否保持位姿 |

### 前置条件

- `planner_demo.py` 只负责发 `PlannerCommand`，不会自行启动 planner 或 supervisor
- 使用前要先启动 `manual_experiment.launch`、`planning_stack.launch` 或 `closed_loop_experiment.launch`

## 切换 planner / controller / mode

在运行下面这类入口之后：

```bash
roslaunch on_orbit_bringup manual_experiment.launch \
  launch_visualization:=true \
  start_foxglove_bridge:=true
```

可以通过 supervisor 提供的 service 在运行中切换 planner、controller 或整套 mode 组合。

### 先看当前状态

```bash
rostopic echo /on_orbit/mode_state
```

输出中最关键的是：

- `mode`
- `active_planner`
- `active_controller`
- `planner_ready`
- `controller_ready`

### 查看可用 supervisor service

```bash
rosservice list | grep on_orbit_supervisor
```

通常会看到：

```text
/on_orbit_supervisor/select_controller
/on_orbit_supervisor/select_planner
/on_orbit_supervisor/set_mission_mode
/on_orbit_supervisor/republish_current_reference
```

### 切换 planner

切到 `se3`：

```bash
rosservice call /on_orbit_supervisor/select_planner "planner_id: 'se3'"
```

切到 `decoupled`：

```bash
rosservice call /on_orbit_supervisor/select_planner "planner_id: 'decoupled'"
```

### 切换 controller

切到 `hqp`：

```bash
rosservice call /on_orbit_supervisor/select_controller "controller_id: 'hqp'"
```

切到 `imp`：

```bash
rosservice call /on_orbit_supervisor/select_controller "controller_id: 'imp'"
```

### 一键切换 mission mode

推荐直接使用组合式 mode 名：

切到 `HQP_SE3`：

```bash
rosservice call /on_orbit_supervisor/set_mission_mode "mode: 'HQP_SE3'"
```

切到 `HQP_DECOUPLED`：

```bash
rosservice call /on_orbit_supervisor/set_mission_mode "mode: 'HQP_DECOUPLED'"
```

切到 `IMP_DECOUPLED`：

```bash
rosservice call /on_orbit_supervisor/set_mission_mode "mode: 'IMP_DECOUPLED'"
```

### 重新转发当前 reference

```bash
rosservice call /on_orbit_supervisor/republish_current_reference
```

说明：

- `select_planner` 和 `select_controller` 成功后，supervisor 会自动重发最近一条缓存轨迹
- 如果你刚切到的 planner 还没有产生过轨迹，就需要再发一条新的 `planner_demo.py` 命令
- `set_mission_mode` 适合切换一整套预定义组合；`select_planner` / `select_controller` 适合手工细调

## 控制器观察命令

当前 active reference pose：

```bash
rostopic echo /imp_controller/debug/active_reference_pose
rostopic echo /hqp_controller/debug/active_reference_pose
```

控制器 debug state：

```bash
rostopic echo /imp_controller/debug/state
rostopic echo /hqp_controller/debug/state
```

系统级模式状态：

```bash
rostopic echo /on_orbit/mode_state
```

## 日志查看命令

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

论文图生成：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py
```

改成 execution 对齐：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
  --align-to execution
```

跳过重新清洗：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
  --skip-clean
```

## 文档相关命令

安装依赖：

```bash
python3 -m pip install -r requirements-docs.txt
```

本地预览：

```bash
mkdocs serve
```

静态构建：

```bash
mkdocs build --strict
```
