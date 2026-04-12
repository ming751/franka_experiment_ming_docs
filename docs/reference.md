# 状态码与消息字典

系统内部所有自定义消息类型、服务接口、Mode 枚举、Debug 字段的统一速查手册。排查问题时，在本页 `Ctrl+F` 检索即可。

---

## Mode 组合映射

当前 mode 采用"控制器 + 规划器"组合命名：

| Mode | Active Planner | Active Controller | 说明 |
|---|---|---|---|
| `IMP_DECOUPLED` | `decoupled` | `imp` | 阻抗控制 + decoupled planner |
| `HQP_DECOUPLED` | `decoupled` | `hqp` | HQP 控制 + decoupled planner |
| `HQP_SE3` | `se3` | `hqp` | HQP 控制 + SE3 planner |

默认初始值定义在 `on_orbit_bringup/config/supervisor.yaml`：`initial_mode: IMP_DECOUPLED`。

---

## Planner 映射

| Planner ID | Reference Topic | Status Topic | 适配 Controller |
|---|---|---|---|
| `decoupled` | `/on_orbit/planners/decoupled/reference` | `/on_orbit/planners/decoupled/status` | `imp` |
| `se3` | `/on_orbit/planners/se3/reference` | `/on_orbit/planners/se3/status` | `hqp` |

所有 planner 共享命令入口 `/on_orbit/planner/command`，通过 `PlannerCommand.planner_id` 字段路由。

---

## Controller 映射

| Logical ID | ros_control 实例名 | `ReferenceTrajectory.control_mode` |
|---|---|---|
| `imp` | `imp_controller` | `imp` |
| `hqp` | `hqp_controller` | `hqp` |

历史兼容别名：`safe_impedance -> imp`, `impedance -> imp`。

---

## `PlannerCommand.target_type` 枚举

| 值 | 常量名 | 语义 |
|---|---|---|
| `0` | `TARGET_TYPE_ABSOLUTE` | 绝对目标位姿 |
| `1` | `TARGET_TYPE_RELATIVE_TO_CURRENT_EE` | 相对当前末端位姿 |
| `2` | `TARGET_TYPE_HOLD_CURRENT_EE` | 保持当前末端位姿 |

---

## Debug Topics 字段速查

### `/on_orbit/mode_state` (`ModeState`)

| 字段 | 含义 |
|---|---|
| `mode` | 当前 supervisor 选中的组合模式 |
| `active_planner` | 当前转发轨迹的 planner |
| `active_controller` | 当前 active controller 的逻辑名 |
| `planner_ready` | 当前 active planner 是否正常 |
| `controller_ready` | 当前 active controller 是否 ready |
| `detail` | 最近一次状态变化的简短说明 |

### `/imp_controller/debug/state` 与 `/hqp_controller/debug/state` (`ControllerDebug`)

#### 轨迹接收与模式匹配

| 字段 | 含义 |
|---|---|
| `controller_name` | 控制器名（`ImpedanceController` / `HqpController`） |
| `expected_mode` | 该控制器接受的 `control_mode` |
| `received_mode` | 最近一次收到的 reference 的 `control_mode` |
| `has_trajectory` | 是否已收到有效轨迹 |
| `mode_matches` | `received_mode` 与 `expected_mode` 是否一致 |
| `has_active_point` | 当前时刻是否成功采样到 active point |
| `reference_age` | 当前轨迹距 `header.stamp` 的经过时间 |
| `sample_time_from_start` | active point 在轨迹内的 `time_from_start` |

#### 命令与测量

| 字段 | 含义 |
|---|---|
| `command_torque_norm` | 当前下发关节力矩范数 |
| `command_velocity_norm` | 当前输出关节速度命令范数 |
| `commanded_joint_torque` | 实际写入关节句柄的力矩 |
| `measured_joint_velocity` | 当前关节速度测量值 |
| `measured_joint_torque_external` | 外部关节力矩估计 `tau_ext` |

#### 位姿与误差

| 字段 | 含义 |
|---|---|
| `reference_pose` | 当前追踪的参考位姿 |
| `measured_pose` | 当前末端测量位姿 |
| `position_error_norm` | 位置误差范数 |
| `orientation_error_norm` | 姿态误差范数 |

#### 接触与外力

| 字段 | 含义 |
|---|---|
| `contact_wrench` | 末端接触 wrench 估计 |
| `contact_alpha` | 接触自适应权重（越大表示外力越明显） |

#### HQP 求解状态

| 字段 | 含义 |
|---|---|
| `hqp_solve_status` | 总体状态码 |
| `hqp_solve_status_text` | 总体状态文本 |
| `hqp_degraded` | 是否处于退化路径 |
| `hqp_level0_qp_status` | L0 原始 proxqp 状态 |
| `hqp_level1_qp_status` | L1 原始 proxqp 状态 |

#### `detail` 常见值

| `detail` | 含义 |
|---|---|
| `waiting_reference` | 未收到有效轨迹 |
| `control_mode_mismatch` | 轨迹 `control_mode` 不属于当前控制器 |
| `active_point_unavailable` | 轨迹存在但当前时刻无有效 active point |
| `tracking_cartesian` | 正在跟踪笛卡尔参考 |

---

## HQP 总体状态码

| 代码 | 文本 | 含义 |
|---|---|---|
| `0` | `OK` | L0 正常，L1 正常 |
| `1` | `L0_DEGRADED` | L0 退化，L1 不再求解 |
| `2` | `L1_TIMEOUT_FALLBACK_L0` | L1 超时，退回 L0 |
| `4` | `L1_NONFINITE_FALLBACK_L0` | L1 非有限，退回 L0 |
| `6` | `L1_STATUS_FALLBACK_L0` | L1 状态异常，退回 L0 |
| `8` | `L1_DISABLED` | L1 关闭，仅执行 L0 |

`-1` 对应 `SOLVER_UNAVAILABLE` 或 `SOLVER_HANDLE_MISSING`，通过 `hqp_solve_status_text` 描述。

---

## `PlannerStatus` 字段

话题：`/on_orbit/planners/<id>/status`（`on_orbit_msgs/PlannerStatus`）

| 字段 | 含义 |
|---|---|
| `planner_id` | planner 名 |
| `state` | 当前状态 |
| `progress` | 任务进度（`0.0 ~ 1.0`） |
| `detail` | 详细状态说明 |

---

## Supervisor 事件流

话题：`/on_orbit_supervisor/debug/events`（`std_msgs/String`）

格式：`event=<name>; mode=<mode>; planner=<planner>; controller=<controller>; detail=<detail>`

适合快速判断 mode/planner/controller 的切换是否成功以及被拒绝的原因。

---

## 包文件索引

各包内的关键源码与配置文件路径汇总：

??? note "on_orbit_msgs — 消息与服务定义"
    - `msg/PlannerCommand.msg`
    - `msg/ReferenceTrajectory.msg`
    - `msg/ControllerDebug.msg`
    - `srv/SetMissionMode.srv`
    - `srv/SelectPlanner.srv`
    - `srv/SelectController.srv`

??? note "on_orbit_control — 控制器实现"
    - `include/on_orbit_control/base_controller.h`
    - `src/impedance_controller.cpp`
    - `src/hqp_controller.cpp`
    - `src/hqp_solver.cpp`
    - `src/lib_class.cpp`

??? note "on_orbit_planners — 规划器实现"
    - `src/on_orbit_planner_node.cpp`
    - `src/trajectory_planning_opti.cpp`（decoupled）
    - `src/trajectory_planning_se3.cpp`（se3）
    - `config/planner_decoupled.yaml`
    - `config/planner_se3.yaml`

??? note "on_orbit_services — Supervisor"
    - `scripts/on_orbit_supervisor.py`

??? note "on_orbit_apps — 应用脚本"
    - `scripts/planner_demo.py`
    - `scripts/joint_posture_demo.py`
    - `scripts/experiment_runner.py`
    - `scripts/experiment_data_logger.py`
    - `scripts/runtime_visualizer.py`

??? note "on_orbit_bringup — Launch 入口"
    - `launch/manual_experiment.launch`
    - `launch/closed_loop_experiment.launch`
    - `launch/hardware_stack.launch`
    - `launch/simulation_gazebo.launch`
    - `config/controllers.yaml`
    - `config/supervisor.yaml`

---

## 推荐排查顺序

1. 查看 `/on_orbit/mode_state` 确认 mode / planner / controller 是否符合预期。
2. 查看 active controller 的 `/debug/state` 中 `mode_matches` 和 `has_active_point`。
3. HQP 场景下重点关注 `hqp_solve_status_text` 和 `hqp_degraded`。
4. 轨迹缺失时查看 `PlannerStatus` 中 `state` 和 `detail`。
5. 切换异常时查看 `/on_orbit_supervisor/debug/events`。
