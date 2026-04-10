# 真机模块化闭环实验实施计划（ROS1 + Docker）

## 1. 目标与约束

### 1.1 总目标
- 在 Ubuntu 24 宿主机上通过 Docker 运行 ROS1（Noetic），稳定控制真实 Franka 机械臂。
- 保持与 On-orbit 仿真一致的“规划方法 + 控制方法 + 消息接口”。
- 在真机场景下强化模块化：规划层、控制层、实验入口三者解耦。

### 1.2 架构约束
- 统一命令入口：`/on_orbit/planner/command`
- 统一控制参考出口：`/on_orbit/reference`
- 实验统一入口节点：`on_orbit_apps/scripts/experiment_runner.py`
- supervisor 仅负责路由与模式/控制器切换，不承载规划算法与控制算法。

## 2. 当前状态盘点（已完成基础）
- 已存在分层 launch：
  - `on_orbit_bringup/launch/upper_stack.launch`（规划 + supervisor）
  - `on_orbit_bringup/launch/hardware_stack.launch`（真机 bringup + upper stack）
  - `on_orbit_bringup/launch/simulation_gazebo.launch`（仿真 bringup + upper stack）
  - `on_orbit_bringup/launch/closed_loop_experiment.launch`（一键实验入口）
- 已存在实验入口节点：`on_orbit_apps/scripts/experiment_runner.py`
- 已存在 supervisor：`on_orbit_services/scripts/on_orbit_supervisor.py`
- 已存在统一 C++ 规划节点与统一 reference 消息流。

## 3. 当前主要问题（优先修复）

### P0-1 入口脚本重复，职责边界不清
- 历史上同时存在：
  - `on_orbit_apps/scripts/experiment_runner.py`（新架构入口）
- 风险：入口混用、维护分叉、行为不一致。
- 当前进展：旧入口已删除，正式入口已收敛到 on_orbit_apps。

### P0-2 controller 切换策略可能无法覆盖真机默认控制器
- `on_orbit_supervisor.py` 当前 stop 列表只覆盖 `controller_profiles` 内声明的控制器。
- 真机上如果官方默认控制器已运行（例如轨迹控制器），可能与自定义控制器资源冲突，导致 switch 失败。

### P0-3 构建脚本存在模板残留重复段
- 入口层历史上曾存在 `on_orbit_services` / `on_orbit_sim` / `on_orbit_bringup` 的职责重叠，现已收敛为 `on_orbit_apps + on_orbit_bringup`。
- 风险：可读性差、后续改动易引入构建歧义。

### P1-1 readiness 判据偏宽松
- supervisor 的 planner ready 具有“历史粘性”，可能掩盖 planner 已掉线场景。
- 风险：experiment_runner 通过 ready 校验但实际执行不可用。

## 4. 目标架构（真机与仿真一致，上下层分离）

### 4.1 分层职责
- 规划层（`on_orbit_planners`）：
  - 只做轨迹生成。
  - 订阅统一命令，输出 planner 专属 reference/status。
- 调度层（`on_orbit_services/on_orbit_supervisor.py`）：
  - 只做模式管理、planner/controller 选择、reference 路由、controller_manager 适配。
- 控制层（`on_orbit_control`）：
  - 只消费 `/on_orbit/reference` 并执行控制律。
- 实验入口（`on_orbit_apps/experiment_runner.py`）：
  - 只负责编排一次实验启动（ready 检查、模式对齐、命令下发、结果确认）。

### 4.2 统一数据流
- `experiment_runner` -> `/on_orbit/planner/command`
- planner -> `/on_orbit/planners/*/reference`
- supervisor -> `/on_orbit/reference`
- active ros_control controller -> real robot

## 5. 分阶段实施计划（按顺序执行）

### 阶段 A：基线清理与单一入口收敛（1 天）
- 明确 `on_orbit_apps/scripts/experiment_runner.py` 为唯一实验入口。
- 删除 `on_orbit_sim/scripts/experiment_runner.py`，并保持 `on_orbit_apps/scripts/experiment_runner.py` 为唯一正式入口。
- 清理 bringup 层与 apps/services 层之间的历史模板残留与职责交叉。
- README 增加“真机标准实验入口”与“禁止使用旧入口”说明。

验收标准：
- 全仓库只保留一个“标准实验入口”路径。
- `catkin_make` 通过。
- README 与 launch 实际行为一致。

### 阶段 B：控制器切换可靠性加固（1-2 天）
- 在 supervisor 增加“可配置 stop_controllers 列表”或“基于运行状态自动推导 stop 列表”。
- 切换后强校验目标控制器 running，失败时回报明确错误。
- 将 controller 切换策略参数化到 `supervisor.yaml`。

验收标准：
- 真机启动后能稳定从默认控制器切换到 `imp/hqp`。
- 切换失败可定位（日志中给出具体控制器名与失败阶段）。

### 阶段 C：实验入口参数规范化（1-2 天）
- 将实验参数组织为“场景配置”（例如 `config/experiments/*.yaml`）。
- `experiment_runner` 支持通过 `scenario_id` 加载配置，减少 launch 参数散落。
- 增加运行元数据（task_id、模式、控制器、约束参数）日志输出。

验收标准：
- 新增一个场景只改 YAML，不改 Python 逻辑。
- `run_once` 与 `auto_start` 两种模式都可用。

### 阶段 D：ready 判据与故障反馈收敛（1 天）
- supervisor 的 planner ready 改为“新鲜度优先、历史状态为辅”。
- experiment_runner 对超时和对齐失败给出更可操作的错误信息。
- 明确“可重试动作”与“必须人工介入动作”。

验收标准：
- planner 掉线时，实验入口不会误判 ready。
- 常见故障（服务未起、controller 未running、轨迹未转发）可直接从日志定位。

### 阶段 E：闭环验证与回归（1-2 天）
- 验证矩阵：
  - `planning_stack.launch`（纯规划）
  - `simulation_gazebo.launch + closed_loop_experiment.launch`（仿真闭环）
  - `hardware_stack.launch + closed_loop_experiment.launch`（真机闭环）
- 输出标准化检查记录（启动时序、切换耗时、首条 reference 延迟、实验成功率）。

验收标准：
- 三条链路均可复现实验启动流程。
- 真机与仿真在上层命令流一致，差异仅在底层 bringup。

## 6. 执行顺序建议
1. 阶段 A（先收敛入口与构建基线）
2. 阶段 B（优先解决真机最可能卡住的切换问题）
3. 阶段 C（入口场景化，方便批量实验）
4. 阶段 D（提高可诊断性）
5. 阶段 E（整体验收）

## 6.1 当前进度
- 阶段 A：已完成（2026-03-23）
- 阶段 B：已启动（控制器命名统一为 `imp/hqp`，并加入旧名兼容）
- 阶段 C-D-E：待执行

## 7. 已确认决策（2026-03-23）
1. 删除旧入口：`on_orbit_sim/scripts/experiment_runner.py`。
2. 真机控制器仅两种：`imp` 与 `hqp`，默认运行 `imp`。
3. 真机通过 `franka_hw` 直接下发关节速度；上层控制器不可用时速度命令保持为 0。
4. 当前先聚焦“实验编排入口”职责，数据落盘能力后续再并入。
