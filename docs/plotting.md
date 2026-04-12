# 数据与绘图

---

## Part A：数据记录

### 实验日志器 (`experiment_data_logger`)

闭环与手动实验在 Launch 时默认启动结构化日志器 `experiment_data_logger.py`。该节点持续订阅系统关键话题，并以 JSONL 格式分流写入磁盘。

核心配置定义在 `src/on_orbit_apps/config/experiment_logger.yaml`：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| **`log_root`** | `/workspace/closed` | 日志存储根目录 |
| **`sample_rate`** | `10.0` Hz | 定频采样写入频率 |
| **`planner_command_topic`** | `/on_orbit/planner/command` | 订阅规划指令 |
| **`mode_state_topic`** | `/on_orbit/mode_state` | 订阅模式状态 |
| **`reference_topic`** | `/on_orbit/reference` | 订阅参考轨迹 |
| **`franka_state_topic`** | `/franka_state_controller/franka_states` | 订阅机器人物理状态 |
| **`supervisor_events_topic`** | `/on_orbit_supervisor/debug/events` | 订阅 Supervisor 事件流 |

如不需要数据记录，在 Launch 时显式关闭：

```bash
roslaunch on_orbit_bringup manual_experiment.launch enable_experiment_logger:=false
```

### 日志目录结构

日志按控制策略自动分组存放，每次会话独立建目录并维护 `latest` 软链接：

```text
/workspace/closed/
├── hqp_se3/
│   ├── latest -> closed_loop_...
│   └── closed_loop_.../
├── imp_decoupled/
│   ├── latest -> closed_loop_...
│   └── closed_loop_.../
└── publication/
```

### 单次会话文件说明

| 文件 | 内容 |
|---|---|
| `metadata.json` | 会话元信息（启动时间、主机名、模式快照） |
| `events.jsonl` | 模式切换、planner command、reference 到达等离散事件流 |
| `samples.jsonl` | `ModeState`、`FrankaState`、`PlannerStatus`、`ControllerDebug` 的定频采样记录 |
| `controller_debug.jsonl` | 控制器 debug 状态序列 |
| `active_reference_pose.jsonl` | 控制器实际追踪的参考位姿序列 |
| `commands/` | 原始 `PlannerCommand` 消息快照存档 |
| `references/` | 原始 `ReferenceTrajectory` 消息快照存档 |
| `processed/` | 后处理产出（对齐 CSV、清洗摘要），由绘图脚本自动生成 |

ROS 系统日志位于 `/workspace/log/ros/`：

```bash
tail -f /workspace/log/ros/latest/*.log
```

---

## Part B：论文绘图

### 一键生成

闭环实验结束后执行：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py
```

脚本自动完成两步操作：

1. 调用 `clean_experiment_series.py` 对最新两组实验记录做时间对齐与清洗，产出 `processed/cleaned_series_*.csv`。
2. 在 `/workspace/closed/publication/` 下输出论文风格对比图（PNG + PDF）。

### 默认输出

| 文件 | 内容 |
|---|---|
| `fig_error_comparison_motion.png/.pdf` | 位置跟踪误差 + 姿态轴角误差 |
| `fig_force_comparison_motion.png/.pdf` | Z 轴接触力 + 六维广义力范数 |

### 参数自定义

=== "切换对齐方式"
    ```bash
    python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
      --align-to execution
    ```

=== "修改阶段线与截止时间"
    ```bash
    python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
      --phase-boundary-s 20 \
      --contact-time-s 27 \
      --time-end-s 40
    ```

=== "跳过重新清洗"
    ```bash
    python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
      --skip-clean
    ```

=== "显式指定对比目录"
    ```bash
    python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
      /workspace/closed/imp_decoupled/latest \
      /workspace/closed/hqp_se3/latest
    ```

### 默认绘图参数

| 参数 | 默认值 |
|---|---|
| 对齐方式 | `motion` |
| 阶段分界线 | `20 s` |
| 接触时刻线 | `27 s` |
| 绘图时间范围 | `0 – 40 s` |
