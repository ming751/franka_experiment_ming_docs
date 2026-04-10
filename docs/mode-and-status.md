# 模式与状态映射

这一页集中整理运行时最容易混淆的几类映射：

- 组合模式名到 planner / controller 的映射
- planner id / backend / preferred controller 的映射
- `PlannerCommand` 的目标类型
- HQP 调试状态码

## Mode 映射

当前 mode 统一采用“控制器 + 规划器”的组合式命名：

| 推荐 Mode | Active Planner | Active Controller | 说明 |
| --- | --- | --- | --- |
| `IMP_DECOUPLED` | `decoupled` | `imp` | 默认阻抗控制 + decoupled planner |
| `HQP_DECOUPLED` | `decoupled` | `hqp` | HQP 控制 + decoupled planner |
| `HQP_SE3` | `se3` | `hqp` | HQP 控制 + SE3 planner |

默认初始值来自 `on_orbit_bringup/config/supervisor.yaml`：

- `initial_mode: IMP_DECOUPLED`
- `initial_planner: decoupled`
- `initial_controller: imp`

## Planner 映射

| Planner ID | Backend | Reference Topic | Status Topic | Preferred Controller |
| --- | --- | --- | --- | --- |
| `decoupled` | `decoupled` | `/on_orbit/planners/decoupled/reference` | `/on_orbit/planners/decoupled/status` | `imp` |
| `se3` | `se3` | `/on_orbit/planners/se3/reference` | `/on_orbit/planners/se3/status` | `hqp` |

说明：

- 所有 planner 共享命令入口 `/on_orbit/planner/command`
- planner 会根据 `PlannerCommand.planner_id` 决定是否处理该命令
- supervisor 只会转发当前 active planner 的结果

## Controller 映射

当前 `controller_profiles` 与 `controller_command_modes` 为：

| Logical Controller ID | ros_control 实例名 | 写入 `ReferenceTrajectory.control_mode` 的值 |
| --- | --- | --- |
| `imp` | `imp_controller` | `imp` |
| `hqp` | `hqp_controller` | `hqp` |

历史兼容别名：

- `safe_impedance -> imp`
- `impedance -> imp`

## `PlannerCommand.target_type` 映射

| 枚举值 | 常量名 | 语义 |
| --- | --- | --- |
| `0` | `TARGET_TYPE_ABSOLUTE` | 绝对目标位姿 |
| `1` | `TARGET_TYPE_RELATIVE_TO_CURRENT_EE` | 相对当前末端位姿 |
| `2` | `TARGET_TYPE_HOLD_CURRENT_EE` | 保持当前末端位姿 |

## HQP Debug 字段整理

当前 `ControllerDebug.msg` 对外保留的是一组更聚焦的 HQP 字段：

| 字段 | 语义 |
| --- | --- |
| `has_hqp_solver_state` | 当前 debug 消息是否包含 HQP 求解状态 |
| `hqp_solver_available` | 当前控制器是否具备可调用的 HQP solver |
| `hqp_solve_status` | 总体 HQP 求解状态码 |
| `hqp_solve_status_text` | 总体 HQP 求解状态文本 |
| `hqp_degraded` | 是否处于降级执行路径 |
| `hqp_level0_qp_status` | Level 0 原始 proxqp 状态 |
| `hqp_level1_qp_status` | Level 1 原始 proxqp 状态；若 `L0` 已退化或 `L1` 被关闭，则为 `PROXQP_NOT_RUN` |

整理思路：

- `hqp_solve_status*`：给上层可视化和日志用，关注“总体结果”
- `hqp_level*_qp_status`：给底层调试用，关注“QP 内核原始返回”
- `hqp_degraded`：明确是否进入 L0-only 或其它降级路径

## HQP 总体状态码

当前 `HQPSolveStatus` 定义如下：

| 代码 | 文本 | 含义 |
| --- | --- | --- |
| `0` | `OK` | L0 正常，L1 正常 |
| `1` | `L0_DEGRADED` | L0 已退化，HQP 直接停止在 L0，L1 不再求解 |
| `2` | `L1_TIMEOUT_FALLBACK_L0` | L1 超时，退回 L0 |
| `4` | `L1_NONFINITE_FALLBACK_L0` | L1 非有限，退回 L0 |
| `6` | `L1_STATUS_FALLBACK_L0` | L1 状态异常，退回 L0 |
| `8` | `L1_DISABLED` | L1 关闭，仅执行 L0 |

说明：

- 当 `hqp_solve_status = L0_DEGRADED` 时，`hqp_level1_qp_status` 应看到 `PROXQP_NOT_RUN`
- 旧的组合码 `3 / 5 / 7 / 9` 目前保留为历史编号，但按现逻辑不会再发出

补充：

- `SOLVER_UNAVAILABLE`
- `SOLVER_HANDLE_MISSING`

这两类情况目前通过 `hqp_solve_status_text` 直接表达，状态码用 `-1`。
