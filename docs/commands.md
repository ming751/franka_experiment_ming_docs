# 手动实验与系统级调试

当自动闭环链路无法满足你的精细化验证需求时，本章提供了一组用于微操和系统单步控制的交互手段。涵盖基础启动参数、运动目标注入与运行状态的热重切。

## A. 基础环境与接口加载

通过向 Launch 文件传递特征参数，系统能够选择性地加载硬件、仿真栈与调试组件：

=== "🛠️ 默认手动实验"
    最轻量的控制进程。适合在线发送各类 `absolute / relative` 指令，并验证轨迹抢占与插值平滑度。
    ```bash
    roslaunch on_orbit_bringup manual_experiment.launch
    ```

=== "🌐 完整可视化 (RViz + Foxglove)"
    启动三维空间透视与实时数据抓表端口。
    *注：默认已关闭 Foxglove 内省服务避免探测污染。*
    ```bash
    roslaunch on_orbit_bringup manual_experiment.launch \
      launch_visualization:=true \
      start_foxglove_bridge:=true
    ```

=== "🦾 仿真模式 (Gazebo)"
    挂载物理引擎，验证动力学响应与接触反馈情况。
    ```bash
    roslaunch on_orbit_bringup manual_experiment.launch use_sim:=true
    ```

=== "🎛️ 特殊初态指定"
    直接以某种预设组合开机，跳过默认状态。
    ```bash
    roslaunch on_orbit_bringup manual_experiment.launch \
      initial_mode:=HQP_SE3 \
      initial_planner:=se3 \
      initial_controller:=hqp
    ```

## B. 运动目标注入指令集

`planner_demo.py` 是唯一允许向整个系统注入规划指令的开发应用。通过传递不同的 `_target_type`，即可激发出差异化极大的运动表现：

=== "绝对位姿目标 (`absolute`)"
    基于世界/基坐标系的绝对到达点。
    > **示例**：同时修改位置与姿态（四元数）。
    ```bash
    rosrun on_orbit_apps planner_demo.py \
      _planner_id:=se3 \
      _target_type:=absolute \
      _target_position:=[0.50,0.20,0.30] \
      _target_orientation:=[0.0,0.0,0.0,1.0]
    ```

=== "相对偏移目标 (`relative`)"
    针对当前末端位姿（End-Effector）的局部增量控制。
    > **示例**：沿当前方位前进 `5cm`。其中姿态使用 3维 `so(3)` 向量输入。
    ```bash
    rosrun on_orbit_apps planner_demo.py \
      _planner_id:=se3 \
      _target_type:=relative_to_current_ee \
      _target_position:=[0.05,0.0,0.0]
    ```

=== "多航点序列 (`waypoints`)"
    连续追踪动作下发，支持使用 JSON 形式配置列表。
    > **示例**：相对目标下的双连击。系统会自动以外推锚点依次应用增量。
    ```bash
    rosrun on_orbit_apps planner_demo.py \
      _planner_id:=se3 \
      _target_type:=relative_to_current_ee \
      _use_waypoints:=true \
      '_waypoints:=[{"position": [0.05, 0.0, 0.0], "orientation": [0.0, 0.0, 0.0]}, {"position": [0.0, 0.05, 0.0], "orientation": [0.0, 0.0, 0.0]}]'
    ```

=== "原位驻留 (`hold`)"
    直接锁定最后获取的物理位姿。
    ```bash
    rosrun on_orbit_apps planner_demo.py \
      _planner_id:=se3 \
      _target_type:=hold_current_ee
    ```

!!! info "执行提示"
    `planner_demo.py` 为前端交互脚本，不负责调配算力。执行前必须确保步骤 A 的基础后端 Launch 已经稳步运行并载入对应的规划器。

### 核心控制字典

为了精细化指令，提供以下主调参数说明：

| 核心调用参数 | 默认值 | 作用域与说明 |
|---|---|---|
| **`_planner_id`** | `se3` | 期望后端的 ID 标识。可选值：`se3`, `decoupled`。 |
| **`_target_type`** | `absolute` | 位姿类型：`absolute` / `relative_to_current_ee` / `hold_current_ee`。 |
| **`_target_position`** | `—` | 三维平移目标 `[x, y, z]`，省略时视 type 为不变或全零增量。 |
| **`_target_orientation`**| `—` | 绝对系下要求 `[qx, qy, qz, qw]`，相对系下要求 3 维 `so3` 旋转增量。 |
| **`_use_waypoints`** | `false` | 是否开启序列解析模式（配合 `_waypoints` 列表使用）。 |
| **`_prepend_current_pose`**| `true`| 强制推入当前物理状态为真实轨迹航点，可确保系统平滑追踪防越变。 |

*注：亦支持使用 `_v_max`, `_a_max` 显式重写内部动力学约束设定。*

---

## C. 运行时动态重切架构

在 `manual_experiment.launch` 运行状态下，由于所有规划器与控制器独立注册，开发者可通过 `rosservice` （或 Supervisor 层）让物理实体或仿真“热置换”。

无论进行何种切换，首要操作是先审查当前的拓扑信息：
```bash
rostopic echo /on_orbit/mode_state
```

=== "🧩 切换 Planner"
    重新绑定规划输出口：
    ```bash
    rosservice call /on_orbit_supervisor/select_planner "planner_id: 'se3'"
    ```

=== "🥊 切换 Controller"
    强行下线并替换底层力矩追踪器：
    ```bash
    rosservice call /on_orbit_supervisor/select_controller "controller_id: 'hqp'"
    ```

=== "🚀 一键任务组合切换"
    将规划器与控制器成套捆绑置换：
    ```bash
    rosservice call /on_orbit_supervisor/set_mission_mode "mode: 'HQP_SE3'"
    ```

=== "🔁 重新激发现有路线"
    触发系统内部状态机，重新应用已接收命令：
    ```bash
    rosservice call /on_orbit_supervisor/republish_current_reference
    ```

## 诊断排障探测

在执行各类非标准的命令调度时，通过抓取特定话题可以轻易洞穿当前报错。

**开启观测面谱**：
直接拦截控制器的 Debug 结构数据：
```bash
rostopic echo /imp_controller/debug/state
rostopic echo /hqp_controller/debug/state
```

**复查数据入库落点**：
排查 `rosbag` 日志保存分流：
```bash
find /workspace/closed/hqp_se3/latest -maxdepth 2 -type f | sort
```
