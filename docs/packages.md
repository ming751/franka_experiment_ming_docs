# 模块说明

这一页按包说明“代码放在哪里、负责什么、遇到问题先看哪里”。

## `on_orbit_msgs`

- 路径：`src/on_orbit_msgs`
- 职责：定义系统统一消息与服务接口
- 关键文件：
  - `msg/PlannerCommand.msg`
  - `msg/ReferenceTrajectory.msg`
  - `msg/ControllerDebug.msg`
  - `srv/SetMissionMode.srv`
  - `srv/SelectPlanner.srv`
  - `srv/SelectController.srv`

适合在这些场景先看：

- 想知道 planner / controller 之间怎么通信
- 想知道 debug topic 的字段含义
- 想给 supervisor 增加服务接口

## `on_orbit_control`

- 路径：`src/on_orbit_control`
- 职责：Franka `ros_control` 插件与实时控制逻辑
- 关键文件：
  - `include/on_orbit_control/base_controller.h`
  - `src/base_controller.cpp`
  - `src/impedance_controller.cpp`
  - `src/hqp_controller.cpp`
  - `src/hqp_solver.cpp`
  - `src/lib_class.cpp`
  - `franka_control_plugins.xml`

模块划分：

- `BaseController`：统一 reference 采样、debug/state 发布
- `ImpedanceController`：笛卡尔阻抗控制
- `HqpController`：HQP 控制主逻辑
- `HQPSolver`：两层 QP 求解与状态枚举
- `lib_class`：共享数学工具

## `on_orbit_planners`

- 路径：`src/on_orbit_planners`
- 职责：统一 planner node 与多种规划后端
- 关键文件：
  - `src/on_orbit_planner_node.cpp`
  - `src/trajectory_planning_opti.cpp`
  - `src/trajectory_planning_se3.cpp`
  - `src/command_target_resolution.cpp`
  - `launch/includes/planners.launch`
  - `config/planner_decoupled.yaml`
  - `config/planner_se3.yaml`

模块划分：

- `decoupled`：五次多项式轨迹
- `se3`：SE(3) 规划与时间参数化
- 统一节点：负责解析 `PlannerCommand`、调后端、输出 `ReferenceTrajectory`

## `on_orbit_services`

- 路径：`src/on_orbit_services`
- 职责：supervisor 路由、模式切换、controller 选择
- 关键文件：
  - `scripts/on_orbit_supervisor.py`
  - `package.xml`

supervisor 负责：

- 维护 mission mode
- 决定当前 active planner
- 决定当前 active controller
- 把 active planner 的轨迹改写后转发到 `/on_orbit/reference`
- 在需要时调用 `controller_manager`

## `on_orbit_apps`

- 路径：`src/on_orbit_apps`
- 职责：用户脚本、实验编排、可视化、日志
- 关键文件：
  - `scripts/planner_demo.py`
  - `scripts/experiment_runner.py`
  - `scripts/experiment_data_logger.py`
  - `scripts/runtime_visualizer.py`
  - `rviz/`
  - `config/experiment_logger.yaml`

用户最常直接用的是：

- `planner_demo.py`

## `on_orbit_bringup`

- 路径：`src/on_orbit_bringup`
- 职责：真机 / 仿真 launch 入口与系统参数
- 关键文件：
  - `launch/manual_experiment.launch`
  - `launch/closed_loop_experiment.launch`
  - `launch/planning_stack.launch`
  - `launch/runtime_visualization.launch`
  - `launch/hardware_stack.launch`
  - `launch/simulation_gazebo.launch`
  - `config/controllers.yaml`
  - `config/supervisor.yaml`

适合在这些场景先看：

- 想知道整套系统怎么启动
- 想调 launch 参数
- 想改默认 `mode / planner / controller`

## `toppra`

- 路径：`src/toppra`
- 职责：规划侧依赖库与相关构建内容
- 说明：它不是 on-orbit 六大业务包的一部分，但和 `se3` 规划后端构建有关

## 推荐排查路径

如果问题出在：

- 命令发不出去：先看 `on_orbit_apps`
- planner 没出轨迹：先看 `on_orbit_planners`
- 轨迹没转发到控制器：先看 `on_orbit_services`
- 控制器不接或力矩异常：先看 `on_orbit_control`
- 启动流程或参数不对：先看 `on_orbit_bringup`
