# Franka Experiment Documentation

这个仓库用于公开维护 Franka ROS1 / On-Orbit 架构 2.0 项目的文档部分，便于在毕业论文、项目说明和评审材料中直接引用。

当前文档聚焦于：

- 顶层命令输入
- 统一规划输出
- supervisor 路由与模式切换
- 1 kHz `ros_control` 力矩控制
- 真机 / Gazebo 复用同一上层架构

说明：

- 本仓库仅公开文档内容
- 主源码仓库可继续保持私有，后续按需公开
- 推荐引用入口：`https://github.com/ming751/franka_experiment_ming_docs`

## 文档导航

- [快速开始](quickstart.md)：Docker、编译、启动整套系统的最短路径
- [架构总览](architecture.md)：六大核心包、消息流与 launch 分层
- [模块说明](packages.md)：各包职责、关键文件、常见入口
- [运行与命令](commands.md)：常用 `roslaunch`、`rosrun`、日志查看命令
- [模式与状态映射](mode-and-status.md)：mission mode、planner/controller 映射、HQP 状态码
- [观测与日志](observability.md)：RViz、Foxglove、`ControllerDebug`、实验日志
- [Debug Topics](debug-topics.md)：`ModeState`、`ControllerDebug`、`PlannerStatus` 字段说明
- [文档维护](docs-workflow.md)：如何本地启动与维护 MkDocs 站点

## 系统主链路

```text
PlannerCommand
  -> on_orbit_planners
  -> ReferenceTrajectory
  -> on_orbit_supervisor
  -> /on_orbit/reference
  -> on_orbit_control (imp / hqp)
  -> Franka Hardware / Gazebo
```

## 核心包

| 包 | 层级 | 核心职责 |
| --- | --- | --- |
| `on_orbit_msgs` | 接口层 | 统一定义 `PlannerCommand`、`ReferenceTrajectory`、`ControllerDebug` 等消息 |
| `on_orbit_control` | 实时控制层 | Franka `ros_control` 插件、阻抗控制、HQP 控制、公共数学工具 |
| `on_orbit_planners` | 规划层 | `decoupled` 与 `se3` 两类轨迹规划后端 |
| `on_orbit_services` | 路由层 | supervisor 模式切换、planner/controller 选择、reference 转发 |
| `on_orbit_apps` | 应用层 | `planner_demo`、实验运行脚本、运行时可视化与数据记录 |
| `on_orbit_bringup` | 集成层 | launch 入口、参数 YAML、真机 / 仿真拼装 |

## 仓库结构

```text
src/
├── on_orbit_msgs/
├── on_orbit_control/
├── on_orbit_planners/
├── on_orbit_services/
├── on_orbit_apps/
└── on_orbit_bringup/
```

## 推荐阅读顺序

1. 先看 [快速开始](quickstart.md)
2. 再看 [架构总览](architecture.md)
3. 需要定位代码时看 [模块说明](packages.md)
4. 做实验或调试时看 [运行与命令](commands.md) 与 [模式与状态映射](mode-and-status.md)
