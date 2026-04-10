# 绘图与后处理

## 闭环实验后的默认流程

当前推荐流程已经收敛为一条命令：

1. 运行闭环实验，生成或更新 `/workspace/closed/imp_decoupled/latest` 与 `/workspace/closed/hqp_se3/latest`
2. 直接执行：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py
```

这条命令会自动完成两步：

- 先调用 `clean_experiment_series.py` 对两组最新实验记录做时间对齐与清洗
- 再生成论文风格的对比图到 `/workspace/closed/publication/`

默认参数：

- 对齐方式：`motion`
- 阶段分界线：`20 s`
- 接触时刻线：`27 s`
- 绘图时间范围：`0 s` 到 `40 s`

## 输出内容

默认会生成四个文件：

- `/workspace/closed/publication/fig_force_comparison_motion.png`
- `/workspace/closed/publication/fig_force_comparison_motion.pdf`
- `/workspace/closed/publication/fig_error_comparison_motion.png`
- `/workspace/closed/publication/fig_error_comparison_motion.pdf`

其中：

- `fig_force_comparison_*`：左图为 `Z` 轴接触力绝对值，右图为六维广义力范数
- `fig_error_comparison_*`：左图为 `XY` 跟踪误差，右图为姿态轴角误差

## 常用命令

默认使用 `motion` 对齐：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py
```

改成 `execution` 对齐：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
  --align-to execution
```

自定义阶段线、接触线和截止时间：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
  --phase-boundary-s 20 \
  --contact-time-s 27 \
  --time-end-s 40
```

如果已经存在清洗后的 CSV，跳过重新清洗：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
  --skip-clean
```

显式指定要对比的两条会话目录：

```bash
python3 /workspace/src/on_orbit_apps/scripts/plot_experiment_publication.py \
  /workspace/closed/imp_decoupled/latest \
  /workspace/closed/hqp_se3/latest
```

## 清洗脚本的职责

`clean_experiment_series.py` 仍然保留，主要用于单独调试后处理逻辑。

它负责：

- 检测 `command`、`planner_ready`、`execution`、`reference`、`motion` 等起点
- 导出 `processed/cleaned_series_*.csv`
- 生成 `alignment_summary_*.json`

但在日常使用中，不需要手动先跑它；`plot_experiment_publication.py` 会自动调用。

## 实验日志目录

闭环实验日志统一写到 `/workspace/closed/`，典型结构如下：

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

单次会话目录通常包含：

- `metadata.json`
- `events.jsonl`
- `samples.jsonl`
- `controller_debug.jsonl`
- `active_reference_pose.jsonl`
- `commands/`
- `references/`
- `processed/`

其中 `processed/` 会在绘图时自动生成。
