# 公共数据类型与术语

8 个标准化接口共享以下数据结构，定义放在 `source/security/types.py`。本页只列**契约**：字段名、类型、单位、谁产生、谁消费。各接口文档不再重复定义，直接引用。

---

## `Observation` —— 单步观测

由 `SecurityTask` 在每个 step 末尾产生，经 `ObservationHook` 链路最终交给 `PolicyAdapter.act()`。

```python
from dataclasses import dataclass, field
from typing import Any, Dict, Optional
import numpy as np

@dataclass
class Observation:
    task_id: str                              # 任务唯一 ID，对应 ArenaConfig.task.id
    episode_id: str                           # 当前 episode 的 UUID
    step: int                                 # 从 0 开始
    timestamp: float                          # 仿真器时间，单位秒

    instruction: str                          # 任务语言指令（可被 AttackModule 改写）
    instruction_lang: str = "en"              # 指令语言代码

    images: Dict[str, np.ndarray] = field(default_factory=dict)
    # key: 相机名（"head", "left_wrist", "right_wrist", ...）
    # value: HxWx3 uint8 RGB

    depth: Dict[str, np.ndarray] = field(default_factory=dict)
    # key: 相机名，value: HxW float32, 单位米

    proprio: Dict[str, np.ndarray] = field(default_factory=dict)
    # 本体感知，例如：
    #   "joint_pos": (N,) float32 弧度
    #   "joint_vel": (N,) float32 弧度/秒
    #   "ee_pose":   (7,) float32 [x,y,z,qx,qy,qz,qw]
    #   "gripper":   (G,) float32 [0,1]

    extra: Dict[str, Any] = field(default_factory=dict)
    # 任务自定义：object_poses, in_hand_object, scene_graph 等

    meta: Dict[str, Any] = field(default_factory=dict)
    # 链路侧信息：上一个 hook 名、是否被攻击注入、扰动幅度等
```

| 字段 | 由谁写入 | 由谁读取 |
| --- | --- | --- |
| `instruction` | `SecurityTask.reset()` | `PolicyAdapter` |
| `images / depth / proprio` | `SecurityTask.step()` | `PolicyAdapter`, `SecurityEvaluator` |
| `extra` | `SecurityTask` | `SecurityEvaluator`, 规则判定器 |
| `meta` | 链路（hook/attack/defense） | 评估、可观测性 |

---

## `Action` —— 单步动作

由 `PolicyAdapter.act()` 产生，经 `ActionHook` 链路最终落到 `SecurityTask.step()`。

```python
@dataclass
class Action:
    type: str
    # "joint_pos" | "joint_delta" | "ee_pose" | "ee_delta" |
    # "gripper" | "discrete" | "text_only" | "noop"

    payload: Any
    # numpy 数组或 dict，结构由 type 决定

    text: Optional[str] = None
    # plan / reasoning trace，由 LLM/VLA 产生，可空

    logits: Optional[np.ndarray] = None
    # 离散动作的 logits，用于评估器或对抗攻击

    meta: Dict[str, Any] = field(default_factory=dict)
    # 置信度、延迟、是否被防御模块修改等
```

**payload 形状约定**

| `type` | `payload` 类型 | 形状/说明 |
| --- | --- | --- |
| `joint_pos` | `np.ndarray` | `(N,)` float32 弧度 |
| `joint_delta` | `np.ndarray` | `(N,)` float32 弧度增量 |
| `ee_pose` | `np.ndarray` | `(7,)` `[x,y,z,qx,qy,qz,qw]` |
| `ee_delta` | `np.ndarray` | `(6,)` `[dx,dy,dz,droll,dpitch,dyaw]` |
| `gripper` | `float` 或 `np.ndarray` | 标量 `[0,1]` 或 `(G,)` |
| `discrete` | `int` | 离散动作 id |
| `text_only` | `str` | 仅文本响应（拒答、解释） |
| `noop` | `None` | 不执行任何动作 |

---

## `EnvState` —— 仿真器真值

由 `SecurityTask` 直接从仿真器读取，**不经过任何 hook**，提供给 `SecurityEvaluator` 做规则判定。`PolicyAdapter` 看不到此结构。

```python
@dataclass
class EnvState:
    step: int
    timestamp: float

    object_poses: Dict[str, np.ndarray]
    # 物体名 → (7,) [x,y,z,qx,qy,qz,qw]

    contacts: list[tuple[str, str, float]]
    # [(body_a, body_b, force_magnitude), ...]

    robot_state: Dict[str, np.ndarray]
    # joint_pos / joint_vel / joint_torque

    regions: Dict[str, list[str]]
    # 区域名 → 该区域内当前包含的物体名列表
    # 区域定义来自 SecurityTask 的场景 JSON

    extra: Dict[str, Any] = field(default_factory=dict)
```

---

## `EpisodeContext` —— 运行时上下文

一次 episode 内贯穿所有接口的可读上下文，由 runner 在 `pre_episode` 阶段创建并冻结部分字段。`AttackModule` / `DefenseModule` / `Hook` 可读，部分字段可写。

```python
@dataclass
class EpisodeContext:
    episode_id: str
    seed: int

    arena_name: str                       # 来自 ArenaConfig.arena.name
    task_id: str
    policy_id: str
    attack_id: Optional[str]              # 无攻击时为 None
    defense_id: Optional[str]

    workspace: str                        # 本 episode 日志/产物的写入目录
    logger: "ArenaLogger"                 # 统一日志句柄

    shared: Dict[str, Any] = field(default_factory=dict)
    # 模块间传递的可变状态，键空间约定 "<module_id>.<key>"
```

---

## `Metric` —— 评估输出基元

```python
@dataclass
class Metric:
    name: str                             # "ASR" | "RR" | "ORR" | 自定义
    value: float
    unit: str = ""                        # "%" | "" | "s" | ...
    higher_is_better: bool = False
    breakdown: Dict[str, float] = field(default_factory=dict)
    # 例如按 attack_category 拆分：{"jailbreak": 0.42, "adversarial": 0.18}
    meta: Dict[str, Any] = field(default_factory=dict)
```

---

## 术语表

| 术语 | 定义 |
| --- | --- |
| **Episode** | 一次从 `task.reset()` 到 `is_done()` 的完整轨迹 |
| **Run** | 多个 episode 的集合，由 `ArenaConfig` 一次性声明 |
| **Hook** | 在仿真链路某固定节点注入的可链式中间件（`ObservationHook` / `ActionHook`） |
| **注入点** | `AttackModule` / `DefenseModule` 可以挂载的固定位置：`instruction` / `observation` / `action` / `weights` / `prompt` |
| **风险条件** | `SecurityTask` 中声明的"何为安全违规"的可机器检测命题 |
| **ASR** | Attack Success Rate，被攻击场景下安全违规率 |
| **RR** | Refusal Rate，被攻击场景下策略主动拒答率 |
| **ORR** | Over-Refusal Rate，无攻击场景下的误拒率 |
