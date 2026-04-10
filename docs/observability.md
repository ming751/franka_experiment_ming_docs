# 观测与日志

## 运行时可视化

运行时可视化入口：

```bash
roslaunch on_orbit_bringup runtime_visualization.launch
```

`runtime_visualizer.py` 主要订阅：

- `ModeState`
- `PlannerStatus`
- `ControllerDebug`
- `ReferenceTrajectory`
- `FrankaState`

更详细的 topic 字段说明见：

- [Debug Topics](debug-topics.md)

可视化中通常会看到：

- 当前末端位姿
- `O_T_EE` 坐标系三轴
- 控制器当前 active reference pose
- 参考路径
- 当前末端历史路径
- 模式 / planner / controller 文本状态

## Foxglove

带 bridge 启动：

```bash
roslaunch on_orbit_bringup runtime_visualization.launch start_foxglove_bridge:=true
```

宿主机连接地址：

```text
ws://localhost:8765
```

默认配置说明：

- 当前 launch 默认关闭 Foxglove 的 ROS service introspection
- 这样做是为了避免 `foxglove_bridge` 对 `/franka_gripper/set_logger_level` 之类的 service 做类型探测时反复打印无意义错误
- 如果确实需要在 Foxglove 中浏览 service，可显式覆盖 `foxglove_capabilities` 和 `foxglove_service_whitelist`

## 控制器 debug topic

两个主要 topic：

- `/imp_controller/debug/state`
- `/hqp_controller/debug/state`

两个 active reference pose topic：

- `/imp_controller/debug/active_reference_pose`
- `/hqp_controller/debug/active_reference_pose`

完整字段解释见：

- [Debug Topics](debug-topics.md)

## `ControllerDebug` 的阅读方式

一般排查顺序：

1. 先看 `mode_matches`
2. 再看 `has_active_point`
3. 再看 `reference_age`
4. 看 `position_error_norm / orientation_error_norm`
5. 如果是 HQP，再看 `hqp_solve_status_text`

对 HQP 控制器，建议优先关注：

- `hqp_solve_status_text`
- `hqp_degraded`
- `hqp_level0_qp_status`
- `hqp_level1_qp_status`

## ROS 日志目录

ROS 日志默认写到：

```text
/workspace/log/ros/
```

常用命令：

```bash
ls -lah /workspace/log/ros
find /workspace/log/ros -maxdepth 2 -type f | sort
tail -f /workspace/log/ros/latest/*.log
```

## 实验数据日志目录

结构化实验日志默认写到：

```text
/workspace/closed/
```

闭环实验默认会维护两组最新记录：

- `/workspace/closed/imp_decoupled/latest`
- `/workspace/closed/hqp_se3/latest`

单次会话通常包含：

- `metadata.json`
- `events.jsonl`
- `samples.jsonl`
- `controller_debug.jsonl`
- `active_reference_pose.jsonl`
- `commands/`
- `references/`
- `processed/`

说明：

- `events.jsonl`：模式切换、planner command、reference、supervisor / runner 事件
- `samples.jsonl`：`ModeState`、`FrankaState`、`PlannerStatus`、`ControllerDebug` 的定频采样
- `processed/`：后处理生成的对齐 CSV 与清洗摘要

## 关闭实验数据日志

```bash
roslaunch on_orbit_bringup manual_experiment.launch enable_experiment_logger:=false
roslaunch on_orbit_bringup closed_loop_experiment.launch enable_experiment_logger:=false
```

## 论文绘图支持

闭环实验结束后，直接执行：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py
```

这条命令会自动：

- 调用 `clean_experiment_series.py` 生成对齐后的 `processed/cleaned_series_*.csv`
- 在 `/workspace/closed/publication/` 下导出论文风格对比图

默认生成：

- `fig_force_comparison_motion.png`
- `fig_force_comparison_motion.pdf`
- `fig_error_comparison_motion.png`
- `fig_error_comparison_motion.pdf`

如果需要改成 `execution` 对齐：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
  --align-to execution
```
