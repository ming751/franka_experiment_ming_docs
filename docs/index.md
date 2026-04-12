# Franka Experiment Documentation

Franka ROS1 / On-Orbit 2.0 文档。
本文档提供从容器启动、编译、闭环仿真实验，到一键生成科研绘图的完整端到端的工作流说明。

## 快速开始

### 1. 准备环境变量

复制环境变量模板，按实际网络修改 `.env` 中的关键变量（例如 `FRANKA_ROBOT_IP`、`ROS_MASTER_URI` 等）：

```bash
cp .env.example .env
```

### 2. 启动并进入实验容器

所有 ROS1 及项目依赖均已内置于 Docker 镜像中。

```bash
docker compose up --build -d
docker exec -it ros1-franka bash
```

### 3. 编译工作区

在容器终端中执行：

```bash
source /opt/ros/noetic/setup.bash
cd /workspace
catkin_make
source /workspace/devel/setup.bash
```

### 4. 启动闭环仿真实验

自动启动 ROS Master、控制器与规划器（SE3, Decoupled 等），进行多套架构自动序列切换，并使用 `rosbag` 自动记录全部实验数据至 `/workspace/closed/`。

```bash
roslaunch on_orbit_bringup closed_loop_experiment.launch
```

### 5. 生成结果图

闭环实验结束后，执行以下命令自动对齐数据，并在 `/workspace/closed/publication/` 下生成对比图（PNG 与 PDF 格式）：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py
```

<figure markdown="span">
  ![跟踪误差对比图](assets/publication/fig_error_comparison_motion.png){ width="48%" }
  ![力响应对比图](assets/publication/fig_force_comparison_motion.png){ width="48%" }
</figure>

### 6. Foxglove 可视化

在启动实验时附加 `start_foxglove_bridge:=true` 参数：

```bash
roslaunch on_orbit_bringup manual_experiment.launch \
  launch_visualization:=true \
  start_foxglove_bridge:=true
```

服务就绪后，在桌面端 Foxglove 连接至 `ws://localhost:8765`，即可监视模式状态与三维轨迹：

![Foxglove 面板展示](assets/foxglove-demo.svg)

## 后续导航

* **[运行与命令](commands.md)**：`planner_demo.py` 的用法与 supervisor 服务切换等说明。
* **[观测与日志](observability.md)**：包含更多调试字段配置与日志结构介绍。
* **[绘图与后处理](plotting.md)**：修改绘图阶段、对齐方式的详细指南。
* **[架构总览](architecture.md)**：系统控制模块、消息路由的深度说明。
