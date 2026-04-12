# 架构总览

Franka On-Orbit 2.0 系统在设计上遵循“高内聚、低耦合”的模块化原则。各核心功能由独立包承载，并通过标准的 ROS1 消息流与服务调用贯穿始终。

## 系统链路数据流向

整个系统的运行可概括为自上而下的漏斗式计算：从上层用户意图出发，经由统一接口下放给特定的规划器，再将规划点阵通过路由中枢无缝派发至不同力矩控制器，最终下发至物理接口。

以下为全景信息流拓扑图：

```mermaid
graph TD
    %% Define Styles
    classDef app fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px;
    classDef planner fill:#e8f5e9,stroke:#43a047,stroke-width:2px;
    classDef supervisor fill:#fff3e0,stroke:#fb8c00,stroke-width:2px;
    classDef control fill:#fce4ec,stroke:#d81b60,stroke-width:2px;
    classDef visualizer fill:#f3e5f5,stroke:#8e24aa,stroke-width:2px;
    classDef hardware fill:#cfd8dc,stroke:#546e7a,stroke-width:2px;
    
    subgraph 应用层 [on_orbit_apps]
        Apps["planner_demo.py<br/>(发送绝对/相对轨迹靶点)"]:::app
    end

    subgraph 规划层 [on_orbit_planners]
        CmdTopic(["话题: /on_orbit/planner/command"])
        PlannerD["Decoupled Planner<br/>(五次多项式插值)"]:::planner
        PlannerS["SE3 Planner<br/>(TOPPRA 参数化)"]:::planner
    end

    subgraph 路由与服务层 [on_orbit_services]
        Supervisor{"on_orbit_supervisor<br/>(总线路由与状态仲裁)"}:::supervisor
        Services[["服务: select_planner / controller<br/>set_mission_mode"]]
    end

    subgraph 控制层 [on_orbit_control]
        RefTopic(["话题: /on_orbit/reference"])
        CtrlIMP["IMP Controller<br/>(笛卡尔阻抗控制)"]:::control
        CtrlHQP["HQP Controller<br/>(层级二次规划控制)"]:::control
    end
    
    subgraph 物理与仿真层 [集成 / 底层]
        Robot["Franka Hardware / Gazebo<br/>(hardware_interface)"]:::hardware
    end

    subgraph 观测与可视化 [Observability]
        Viz["Foxglove / RViz<br/>(runtime_visualizer)"]:::visualizer
    end

    %% Data Flow
    Apps --"1. 推送 PlannerCommand"--> CmdTopic
    Apps -.调用.-> Services
    
    CmdTopic --> PlannerD
    CmdTopic --> PlannerS
    
    PlannerD --"2. 抛出连续 ReferenceTrajectory"--> Supervisor
    PlannerS --"2. 抛出连续 ReferenceTrajectory"--> Supervisor
    
    Services -.异步切换控制策略.-> Supervisor
    Supervisor --"3. 透传所选轨迹"--> RefTopic
    Supervisor --"动态激活 controller_manager"--> Robot
    
    RefTopic --> CtrlIMP
    RefTopic --> CtrlHQP
    
    CtrlIMP --"4. 运算输出关节力矩"--> Robot
    CtrlHQP --"4. 运算输出关节力矩"--> Robot
    
    %% Observability flows
    Supervisor -.发布 ModeState.-> Viz
    CtrlIMP -.发布 DebugState.-> Viz
    CtrlHQP -.发布 DebugState.-> Viz
    Robot -.反馈 FrankaStates.-> Viz
```

---

## 模块分工解析

各分层中的关键包职能如下，可结合上图查看：

### 1. 应用入口层 (`on_orbit_apps`)
面向开发者的触手。
* **主要职责**：负责任务编排与参数转化。用户直接运行 `planner_demo.py` 或由 `experiment_runner.py` 发起自动化实验。这些入口将粗略的移动目标包装为标准的 `on_orbit_msgs/PlannerCommand` 进行广播。

### 2. 位姿规划层 (`on_orbit_planners`)
承接上层需求并转化为物理可实现的高密度时间序列点阵。
* **特征**：无论底下运行何种规划器，此层的统一输入必然是 `PlannerCommand`，统一输出必然是 `ReferenceTrajectory`。
* **现有后端**：
  * `decoupled`：基于五次多项式的高保真定长插值，适配 `imp` 控制器。
  * `se3`：基于李代数的时间最优参数化寻址，适配复杂的 `hqp` 逻辑。

### 3. 路由仲裁层 (`on_orbit_services`)
控制中枢大脑，消除系统状态发散的可能性。
* **Supervisor**：汇聚所有 Planner 产生的密集轨迹线，依靠用户或 `rosservice` 调用的状态切换逻辑，精简出**唯一一条**合法轨迹发布至公用话题 `/on_orbit/reference` 喂给目标控制器。同时也负责向硬件底层的 `controller_manager` 发起实际的 Plugin 加载和停用请求。

### 4. 实时控制层 (`on_orbit_control`)
高速运算力矩指令的插件层，通常在 1kHz 频率以上执行。
* **公共数学库**：依托 `lib_class` 处理操作空间惯量、接触力估计、零空间映射等复杂矩阵代数。
* **控制器集成**：通过从基类 `BaseController` 派生，各策略仅需实现专有的力矩计算。控制器不保存历史轨迹，严格依靠提取流经 `/on_orbit/reference` 对应时间截面的采样点进行闭环追捕。

### 5. 接口统一层 (`on_orbit_msgs`)
确保系统不会产生依赖地狱的骨架。
* **极简原则**：无论底层怎么更换，所有跨包通信均严格限制在消息包定义的 `PlannerCommand` 和 `ReferenceTrajectory` 中，各层完全解耦互不感知。

### 6. 配置集成层 (`on_orbit_bringup`)
拼图的核心组织者，聚合参数配置与多级 `launch`。
* **分级调阅**：从 `hardware_stack.launch` 一路套用到面向最终场景的 `closed_loop_experiment.launch`，确保真机与 Gazebo 仿真能够复用同一套核心逻辑而仅需切换硬件宿主。


## 故障归因速查

根据信息流单向传递的特性，当问题发生时，可逆推对应层级：
* **发布移动目标无效**：检查 `on_orbit_apps` 是否成功建联并推入指令。
* **无法生成轨迹警告**：检查 `on_orbit_planners` 输入目标的可达性与算法极点。
* **控制器不转动但日志有下发**：检查 `on_orbit_services` 汇集轨迹后有没有因为 Supervisor 阻塞掉抛出。
* **力矩抖动或关节限位**：检查 `on_orbit_control` 端矩阵解算边界或物理奇异性。
