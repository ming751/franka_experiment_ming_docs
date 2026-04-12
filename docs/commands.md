# 手动实验与系统级调试

当自动闭环链路无法覆盖特定验证场景时，可通过本章所述的手动操作进行单步控制与状态诊断。内容涵盖基础启动参数、运动目标注入与运行时架构切换。

---

## A. 基础环境与接口加载

通过向 Launch 文件传递不同参数，可按需加载硬件栈、仿真环境与调试组件：

=== "🛠️ 默认手动实验"
    启动最小控制进程，用于发送 `absolute` / `relative` 指令并验证轨迹平滑度。
    ```bash
    roslaunch on_orbit_bringup manual_experiment.launch
    ```

=== "🌐 完整可视化 (RViz + Foxglove)"
    三维空间透视与实时数据端口。默认已关闭 Foxglove 的 service introspection。
    ```bash
    roslaunch on_orbit_bringup manual_experiment.launch \
      launch_visualization:=true \
      start_foxglove_bridge:=true
    ```

=== "🦾 仿真模式 (Gazebo)"
    挂载物理引擎，验证动力学响应与接触反馈。
    ```bash
    roslaunch on_orbit_bringup manual_experiment.launch use_sim:=true
    ```

=== "🎛️ 指定初始组合"
    以特定的 planner / controller 组合启动，跳过默认状态。
    ```bash
    roslaunch on_orbit_bringup manual_experiment.launch \
      initial_mode:=HQP_SE3 \
      initial_planner:=se3 \
      initial_controller:=hqp
    ```

---

## B. 运动目标注入指令集

`planner_demo.py` 是系统唯一的规划指令注入入口。通过 `_target_type` 参数选择不同的运动模式：

=== "绝对位姿目标 (`absolute`)"
    基于基坐标系的绝对到达点。
    > **示例**：同时指定位置与姿态（四元数）。
    ```bash
    rosrun on_orbit_apps planner_demo.py \
      _planner_id:=se3 \
      _target_type:=absolute \
      _target_position:=[0.50,0.20,0.30] \
      _target_orientation:=[0.0,0.0,0.0,1.0]
    ```

=== "相对偏移目标 (`relative`)"
    基于当前末端位姿（EE）的局部增量。
    > **示例**：沿当前方向前进 `5cm`，姿态使用 3 维 `so(3)` 向量。
    ```bash
    rosrun on_orbit_apps planner_demo.py \
      _planner_id:=se3 \
      _target_type:=relative_to_current_ee \
      _target_position:=[0.05,0.0,0.0]
    ```

=== "多航点序列 (`waypoints`)"
    连续追踪，支持 JSON 格式的航点列表。
    > **示例**：相对坐标模式下依次应用两次增量。
    ```bash
    rosrun on_orbit_apps planner_demo.py \
      _planner_id:=se3 \
      _target_type:=relative_to_current_ee \
      _use_waypoints:=true \
      '_waypoints:=[{"position": [0.05, 0.0, 0.0], "orientation": [0.0, 0.0, 0.0]}, {"position": [0.0, 0.05, 0.0], "orientation": [0.0, 0.0, 0.0]}]'
    ```

=== "原位驻留 (`hold`)"
    锁定当前末端物理位姿。
    ```bash
    rosrun on_orbit_apps planner_demo.py \
      _planner_id:=se3 \
      _target_type:=hold_current_ee
    ```

!!! info "前置条件"
    `planner_demo.py` 为前端脚本，执行前需确保步骤 A 的后端 Launch 已正常运行并载入对应规划器。

### 核心参数说明

| 参数 | 默认值 | 说明 |
|---|---|---|
| **`_planner_id`** | `se3` | 规划器 ID。可选：`se3`、`decoupled`。 |
| **`_target_type`** | `absolute` | 位姿类型：`absolute` / `relative_to_current_ee` / `hold_current_ee`。 |
| **`_target_position`** | — | 三维平移 `[x, y, z]`。省略时按 type 默认处理。 |
| **`_target_orientation`** | — | 绝对系：`[qx, qy, qz, qw]`；相对系：3 维 `so3` 增量。 |
| **`_use_waypoints`** | `false` | 是否开启航点序列模式（配合 `_waypoints` 使用）。 |
| **`_prepend_current_pose`** | `true` | 将当前位姿插入为起始航点，确保平滑追踪。 |

!!! tip "动力学约束覆写"
    支持通过 `_v_max`、`_a_max` 参数显式重写内部速度与加速度限制。

---

## C. 运行时架构切换

在 `manual_experiment.launch` 运行状态下，所有规划器与控制器独立注册，支持通过 `rosservice` 进行热切换。

切换前建议先确认当前状态：

```bash
rostopic echo /on_orbit/mode_state
```

=== "🧩 切换 Planner"
    ```bash
    rosservice call /on_orbit_supervisor/select_planner "planner_id: 'se3'"
    ```

=== "🥊 切换 Controller"
    ```bash
    rosservice call /on_orbit_supervisor/select_controller "controller_id: 'hqp'"
    ```

=== "🚀 整组切换 Mode"
    ```bash
    rosservice call /on_orbit_supervisor/set_mission_mode "mode: 'HQP_SE3'"
    ```

=== "🔁 重新发布当前轨迹"
    ```bash
    rosservice call /on_orbit_supervisor/republish_current_reference
    ```

---

## D. 诊断与排障

### 控制器 Debug 状态

```bash
rostopic echo /imp_controller/debug/state
rostopic echo /hqp_controller/debug/state
```

### 实验数据文件检查

```bash
find /workspace/closed/hqp_se3/latest -maxdepth 2 -type f | sort
```

!!! note "完整字段参考"
    Debug 话题的完整字段含义参见 [状态码与消息字典](reference.md)。
