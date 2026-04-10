# 架构总览

## 六大核心包

| 包 | 语言 | 部署层 | 核心职责 | 关键交互 |
| --- | --- | --- | --- | --- |
| `on_orbit_msgs` | msg/srv | 接口层 | 定义统一消息与服务接口 | 所有包依赖 |
| `on_orbit_control` | C++ | 实时控制层 | `imp` / `hqp` 控制器与 Franka 插件 | 接收 `/on_orbit/reference` |
| `on_orbit_planners` | C++ | 规划层 | `decoupled` 与 `se3` 规划后端 | 消费 `PlannerCommand` |
| `on_orbit_services` | Python | 路由层 | supervisor 模式切换与转发 | 连接 planner 与 controller |
| `on_orbit_apps` | Python | 应用层 | demo、实验编排、可视化、日志 | 面向用户入口 |
| `on_orbit_bringup` | launch/yaml | 集成层 | launch 入口与参数拼装 | 组合整套系统 |

## 主消息流

```text
用户命令
  -> /on_orbit/planner/command
  -> on_orbit_planners
  -> /on_orbit/planners/<planner>/reference
  -> on_orbit_supervisor
  -> /on_orbit/reference
  -> on_orbit_control
  -> Franka / Gazebo
```

## 包 / 节点 / 话题简图

### 包级视图

```text
on_orbit_apps
  -> 发布 PlannerCommand / 调 supervisor service
  -> on_orbit_planners
  -> on_orbit_services
  -> on_orbit_control

on_orbit_planners
  -> 输出 planner-specific ReferenceTrajectory / PlannerStatus
  -> on_orbit_services

on_orbit_services
  -> 转发统一 /on_orbit/reference
  -> 选择 planner / controller
  -> on_orbit_control

on_orbit_control
  -> 输出 ControllerDebug / active_reference_pose
  -> Franka hardware 或 Gazebo
```

### 节点级视图

```text
[planner_demo.py / experiment_runner.py]
  -> /on_orbit/planner/command
  -> [decoupled_planner] -----------------> /on_orbit/planners/decoupled/reference
  -> [se3_planner] -----------------------> /on_orbit/planners/se3/reference
  -> [decoupled_planner] -----------------> /on_orbit/planners/decoupled/status
  -> [se3_planner] -----------------------> /on_orbit/planners/se3/status

[/on_orbit_supervisor]
  <- planner reference / status
  -> /on_orbit/reference
  -> /on_orbit/mode_state
  -> /on_orbit_supervisor/debug/events
  -> /controller_manager/*   (真机 / Gazebo 下切换控制器)

[/imp_controller]  <- /on_orbit/reference
  -> /imp_controller/debug/state
  -> /imp_controller/debug/active_reference_pose

[/hqp_controller]  <- /on_orbit/reference
  -> /hqp_controller/debug/state
  -> /hqp_controller/debug/active_reference_pose

[runtime_visualizer.py / RViz / Foxglove]
  <- /on_orbit/mode_state
  <- /on_orbit/reference
  <- /imp_controller/debug/state
  <- /hqp_controller/debug/state
  <- /franka_state_controller/franka_states
```

统一接口约束：

- 任务输入统一为 `on_orbit_msgs/PlannerCommand`
- 规划器输出统一为 `on_orbit_msgs/ReferenceTrajectory`
- 控制器只根据 `header.stamp + time_from_start` 做实时采样追踪

## launch 分层

### 公开入口

- `manual_experiment.launch`
- `closed_loop_experiment.launch`
- `planning_stack.launch`
- `runtime_visualization.launch`

### 底层 / 调试入口

- `hardware_stack.launch`
- `simulation_gazebo.launch`
- `hardware_controller_only.launch`

### 内部拼装件

- `launch/includes/upper_stack.launch`
- `on_orbit_planners/launch/includes/planners.launch`

## 谁负责什么

如果只想快速知道“问题大概率在哪一层”，可以按下面的顺序看：

- 命令没发出去：`on_orbit_apps`
- planner 没出 reference：`on_orbit_planners`
- reference 没被转发：`on_orbit_services`
- 控制器没接管 / HQP 状态异常：`on_orbit_control`
- 真机 / 仿真启动失败：`on_orbit_bringup`

## 真机 / 仿真的复用原则

真机与 Gazebo 的差异被限制在最底层硬件后端：

- 真机：`franka_control + libfranka + Panda`
- 仿真：`franka_gazebo + 模拟 Franka 硬件`

而以下层面尽量保持一致：

- `PlannerCommand` 统一命令入口
- planner 输出结构
- supervisor 路由逻辑
- `/on_orbit/reference`
- `imp` / `hqp` 控制器插件

## 控制层说明

`on_orbit_control` 当前由一个共享基类和两个控制器实现组成：

- `BaseController`：reference 接收、轨迹采样、debug 发布、力矩限幅与下发
- `ImpedanceController`：经典笛卡尔阻抗力矩控制
- `HqpController`：基于 body frame 误差与二层 HQP 的控制器

公共数学工具集中在 `lib_class` 中，包括：

- Franka 模型数据封装
- 操作空间惯量
- 广义逆与零空间投影
- 接触力估计与 `alpha` 平滑计算

## planner 分层

当前默认包含两个 planner：

- `decoupled`：五次多项式插值，默认期望控制器 `imp`
- `se3`：SE(3) / TOPPRA 风格时间参数化，默认期望控制器 `hqp`

它们共享统一命令入口 `/on_orbit/planner/command`，内部通过 `PlannerCommand.planner_id` 决定是否消费命令。
