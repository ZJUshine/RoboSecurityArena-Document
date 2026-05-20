# 具身智能安全攻防平台标准化接口设计

**在仿真执行链路的关键节点使用标准化接口设计**，让攻击/防御/被测模型都能作为标准插件接入。

## 仿真流程

```
[A] Task & Scene Load    →  仿真器加载任务和场景，初始化环境
[B] Observe              →  从仿真器获取观测（任务指令和传感器数据）
[C] Policy               →  根据观测和任务指令，调用被测模型策略生成动作
[D] Action Apply         →  将动作发送回仿真器，执行物理交互
[E] Evaluate             →  评估器根据观测、动作和环境状态评估任务成功率和安全违规
```

---

## 标准化接口

平台共 8 个标准化接口，覆盖 [A] 任务加载到 [E] 评估的全链路。

| #   | 接口                                 | 职责                       |
| --- | ------------------------------------ | -------------------------- |
| 1   | `ArenaConfig`                        | 声明式竞技场入口           |
| 2   | `SecurityTask`                       | 任务与场景标准化           |
| 3   | `PolicyAdapter`                      | 策略模型统一抽象           |
| 4   | `SecurityEvaluator`                  | 安全评估                   |
| 5   | `ObservationHook`                    | 图像和指令读取和输入接口   |
| 6   | `ActionHook`                         | 动作读取和输入接口         |
| 7   | `AttackModule`                       | 攻击模块                   |
| 8   | `DefenseModule`                      | 防御模块                   |


### 接口 1：`ArenaConfig` —— 声明式竞技场入口

一份 YAML 声明"policy × attack × defense × benchmark"，由 `source/security/arena/runner.py` 统一调度。是组装上述 7 个接口的顶层入口。

### 接口 2：`SecurityTask` —— 任务

`SecurityTask` 标准化场景、指令与风险条件。继承自 `geniesim/benchmark/tasks/BaseTask`，并与 `eval_tasks/`、`llm_task/` 现有 JSON 体系平行新增 `security_tasks/`。

### 接口 3：`PolicyAdapter` —— 被测策略

把 LLM/VLM、VLA、WAM、Brain 四类模型的 IO 差异封装为统一适配器，暴露 `act()` / `plan()` / `get_logits()` / `get_internal_state()` / `load_checkpoint()`等接口。

### 接口 4：`SecurityEvaluator` —— 安全评估

对任务过程和结果"安全违规"评估，输出 ASR / RR / ORR 等指标，由 VLM 语义判定 + 规则判定器（物体位姿、接触、区域）融合。

### 接口 5：`ObservationHook`

在 `data_courier` 与 `api_core` 之间的观测返回路径上提供可注入中间件，让传感器攻击（Phantom Menace 等）与输入异常检测共用同一管线。

### 接口 6：`AttackModule` —— 攻击方法模块

按 `attack_category`分类，涵盖越狱、对抗、后门三大类攻击。 

### 接口 7：`DefenseModule` —— 防御方法模块

按 `defense_category` 分类，涵盖前置检测、模型对齐、后置过滤三大类防御

---

## 总体架构

```
source/security/
├── tasks/               # SecurityTask
├── adapters/            # AgentAdapter
├── attacks/             # AttackModule
├── defenses/            # DefenseModule
├── evaluators/          # SecurityEvaluator
├── hooks/               # ObservationHook & ActionHook
└── config/              # ArenaConfig
```