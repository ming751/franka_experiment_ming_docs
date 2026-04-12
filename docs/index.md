# Franka Experiment Documentation

Franka ROS1 / On-Orbit 2.0 实验系统。
本文档提供从容器环境启动、ROS1 算法栈编译、自动闭环仿真到一键生成科研论文图表的完整端到端工作流。

<figure markdown="span" style="margin-top: 2rem; margin-bottom: 3rem; display: block; text-align: center;">
  <img src="assets/terminal-demo.svg" alt="终端启动流程演示" style="width: 100%; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.1);">
</figure>

---

## 快速开始

### 1. 准备环境变量

复制环境变量模板，并按实际网络环境修改 `.env` 中的关键变量（例如 `FRANKA_ROBOT_IP`、`ROS_MASTER_URI` 等）。通常第一次配置后无需再变动。

```bash
cp .env.example .env
```

### 2. 启动并进入实验容器

所有依赖与编译工具链均已内置于配置好的 Docker 镜像中。执行以下命令拉起并进入容器：

```bash
docker compose up --build -d
docker exec -it ros1-franka bash
```

### 3. 编译工作区

在容器终端中依次执行，完成 ROS1 工作区的编译与刷新：

```bash
source /opt/ros/noetic/setup.bash
cd /workspace
catkin_make
source /workspace/devel/setup.bash
```

<br>

### 4. 启动闭环仿真实验

下发指令将自动启动 ROS Master、底层控制器以及规划器（SE3, Decoupled 等）。
运行期间会进行多套架构的自动序列切换，并使用 `rosbag` 进行全景数据抓取，保存至 `/workspace/closed/` 目录下。

```bash
roslaunch on_orbit_bringup closed_loop_experiment.launch
```

### 5. 科研结果绘图与指标计算

闭环实验结束后，执行数据处理与分析脚本。算法将自动对齐各策略维度数据，最终输出高清晰度对比图（PNG 与纯矢量 PDF 格式）至 `/workspace/closed/publication/`。

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py
```

执行产出包含跟踪误差与广义力响应等重要指标比对图幅：

<figure markdown="span" style="margin-top: 1rem; margin-bottom: 2rem;">
  ![跟踪误差对比图](assets/publication/fig_error_comparison_motion.png){ width="48%" }
  ![力响应对比图](assets/publication/fig_force_comparison_motion.png){ width="48%" }
</figure>

<br>

### 6. 进阶使用：Foxglove 调试面板

启动常规实验时，可通过附加 `start_foxglove_bridge:=true` 标志开启系统透传端口：

```bash
roslaunch on_orbit_bringup manual_experiment.launch \
  launch_visualization:=true \
  start_foxglove_bridge:=true
```

启动完成并在桌面端运行 Foxglove，连接至 `ws://localhost:8765` 即可获得涵盖 ModeState 与 3D 轨迹预演在内的全套运行时面板：

<figure markdown="span" style="margin-top: 1rem; margin-bottom: 1rem; display: block; text-align: center;">
  <img src="assets/foxglove-demo.svg" alt="Foxglove 面板展示" style="width: 100%; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.1);">
</figure>

---

## 导航说明

更专门、更深入的组件参数以及底层原理，已拆解在如下页面中：

* **[运行与命令](commands.md)**：包含 `planner_demo.py` 所有发送动作指令的指南，以及 Supervisor 的模式切换。
* **[观测与日志](observability.md)**：详细记录 Debug Topic 的字段构成与实验录包的物理路径。
* **[绘图与后处理](plotting.md)**：说明如何更改论文绘图阶段切割线、修改阈值对齐参数。
* **[架构总览](architecture.md)**：深度剖析系统控制链路、层级划分与通讯拓扑。
