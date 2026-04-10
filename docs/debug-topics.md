# Debug Topics

这一页专门解释运行时最常看的几个 debug / status topic，重点是：

- `/on_orbit/mode_state`
- `/imp_controller/debug/state`
- `/hqp_controller/debug/state`
- `/on_orbit_supervisor/debug/events`

## `/on_orbit/mode_state`

消息类型：`on_orbit_msgs/ModeState`

| 字段 | 含义 | 排查时怎么用 |
| --- | --- | --- |
| `mode` | 当前 supervisor 选中的组合模式 | 先确认系统是否切到了你预期的 `IMP_DECOUPLED / HQP_DECOUPLED / HQP_SE3` |
| `active_planner` | 当前正在向控制层转发哪个 planner 的输出 | 如果 planner command 发了但没有轨迹，先核对这里 |
| `active_controller` | 当前 active controller 的逻辑名 | 常见值是 `imp` 或 `hqp` |
| `planner_ready` | 当前 active planner 是否仍有新鲜状态且未报错 | `false` 时优先查 planner 节点或 status topic |
| `controller_ready` | 当前 active controller 是否 ready | 真机/仿真下通常依赖 `controller_manager` 状态 |
| `detail` | 最近一次状态变化的简短说明 | 能快速看出是 `mode set`、`planner selected` 还是 `controller selected` |

## `/imp_controller/debug/state` 与 `/hqp_controller/debug/state`

消息类型：`on_orbit_msgs/ControllerDebug`

### 一. 轨迹接收与模式匹配

| 字段 | 含义 |
| --- | --- |
| `controller_name` | 控制器名字，如 `ImpedanceController` / `HqpController` |
| `expected_mode` | 该控制器愿意接收的 `ReferenceTrajectory.control_mode` |
| `received_mode` | 最近一次收到的 reference 的 `control_mode` |
| `active_trajectory_id` | 当前 reference 的轨迹 ID |
| `has_trajectory` | 是否已经收到有效轨迹 |
| `mode_matches` | `received_mode` 是否和 `expected_mode` 一致 |
| `has_active_point` | 当前时刻是否成功从轨迹中采样到 active point |
| `hold_last_point` | 轨迹结束后是否保持最后一个点 |
| `reference_age` | 当前轨迹距其 `header.stamp` 过去了多久 |
| `sample_time_from_start` | 当前 active point 在轨迹内部对应的 `time_from_start` |
| `last_point_time` | 轨迹最后一个点的 `time_from_start` |
| `command_timeout` | 控制器判定 reference 过期的超时阈值 |

优先看这组字段的场景：

- 控制器完全不动
- 切换 planner / controller 后没有接管
- reference 明明发了，但看起来没被消费

### 二. 命令与测量

| 字段 | 含义 |
| --- | --- |
| `max_joint_velocity` | 当前控制器使用的关节速度限制 |
| `max_delta_torque` | 力矩变化率限幅 |
| `command_velocity_norm` | 当前输出关节速度命令范数 |
| `command_torque_norm` | 当前下发关节力矩范数 |
| `desired_joint_velocity` | 控制器内部生成的关节速度命令 |
| `commanded_joint_torque` | 实际写入关节句柄的力矩命令 |
| `measured_joint_velocity` | 当前关节速度测量值 |
| `measured_joint_torque_external` | 当前外部关节力矩估计 `tau_ext` |

### 三. 位姿与误差

| 字段 | 含义 |
| --- | --- |
| `has_reference_pose` | 当前 active point 是否包含位姿参考 |
| `reference_pose` | 当前正在追踪的参考位姿 |
| `has_measured_pose` | 是否成功填充当前机器人测量位姿 |
| `measured_pose` | 当前末端测量位姿 |
| `has_pose_error` | 是否成功计算位姿误差 |
| `position_error` | 三维位置误差 |
| `orientation_error_axis_angle` | 轴角形式的姿态误差 |
| `position_error_norm` | 位置误差范数 |
| `orientation_error_norm` | 姿态误差范数 |

### 四. 接触与外力

| 字段 | 含义 |
| --- | --- |
| `has_contact_wrench` | 是否发布接触 wrench 估计 |
| `contact_wrench` | 控制器使用的末端接触 wrench |
| `has_contact_alpha` | 是否发布接触自适应系数 |
| `contact_alpha` | 接触自适应权重，越大通常表示外力越明显 |

### 五. HQP 求解状态

这一组字段只在 `HqpController` 走过 HQP 路径时有意义。

| 字段 | 含义 |
| --- | --- |
| `has_hqp_solver_state` | 当前消息是否包含 HQP 求解状态 |
| `hqp_solver_available` | 当前控制器是否具备 HQP solver |
| `hqp_solve_status` | HQP 总体状态码 |
| `hqp_solve_status_text` | HQP 总体状态文本 |
| `hqp_degraded` | 是否处于退化路径 |
| `hqp_level0_qp_status` | L0 原始 proxqp 状态 |
| `hqp_level1_qp_status` | L1 原始 proxqp 状态；若未运行则通常是 `PROXQP_NOT_RUN` |

其中 `hqp_solve_status*` 的具体映射见：

- [模式与状态映射](mode-and-status.md)

### 六. `detail` 常见值

`detail` 是控制器对自己当前执行状态的简短总结。当前常见值有：

| `detail` | 含义 |
| --- | --- |
| `waiting_reference` | 还没收到有效轨迹 |
| `control_mode_mismatch` | 收到轨迹了，但 `control_mode` 不属于当前控制器 |
| `active_point_unavailable` | 轨迹存在，但当前时刻没采到有效 active point |
| `tracking_joint_velocity` | 当前主要在跟踪关节速度命令 |
| `tracking_cartesian` | 当前主要在跟踪笛卡尔参考 |
| `tracking` | 正在跟踪，但不属于上面几个更具体的分类 |

## `/on_orbit_supervisor/debug/events`

消息类型：`std_msgs/String`

这个 topic 是 supervisor 的事件流，内容通常是：

```text
event=<event_name>; mode=<mode>; planner=<planner>; controller=<controller>; detail=<detail>
```

适合用来快速观察：

- mode / planner / controller 是否切换成功
- 当前 reference 是否被 republish
- 切换被拒绝时的原因

## `PlannerStatus`

planner status topic 通常为：

- `/on_orbit/planners/decoupled/status`
- `/on_orbit/planners/se3/status`

消息类型：`on_orbit_msgs/PlannerStatus`

| 字段 | 含义 |
| --- | --- |
| `planner_id` | planner 名字 |
| `state` | planner 当前状态 |
| `progress` | 当前任务进度，范围通常在 `0.0 ~ 1.0` |
| `detail` | 更详细的状态说明 |

## 推荐排查顺序

1. 先看 `/on_orbit/mode_state`
2. 再看 active controller 对应的 `/debug/state`
3. 如果是 HQP，再看 `hqp_solve_status_text`
4. 如果 planner 没出轨迹，再看对应的 `PlannerStatus`
5. 如果切换有问题，再看 `/on_orbit_supervisor/debug/events`
