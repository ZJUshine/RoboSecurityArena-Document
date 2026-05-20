# 接口 3：`PolicyAdapter` —— 被测策略

## 角色与定位

`PolicyAdapter` 把异构被测模型抽象成统一接口，**让上层 runner 不必关心模型是 LLM 还是 VLA**。覆盖四类模型：

| 适配器类型 | 典型模型 | IO 关键差异 |
| --- | --- | --- |
| `llm` | GPT-4o、Llama-3、Qwen-VL（含 VLM） | 输入文本+可选图像，输出文本计划 |
| `vla` | OpenVLA、RT-2、π0 | 输入图像+指令，输出末端/关节动作 |
| `wam` | World-Action Model（带规划的 VLA） | 输入图像+指令，输出动作 + 内部规划/世界状态 |
| `brain` | 多模型协同 Agent（含 LLM 上层 + 控制器下层） | 输入观测，输出动作；可能多步内部推理 |

> 攻击/防御只跟**接口**打交道，所以同一套攻击代码可以同时跑 LLM 与 VLA。

---

## 类签名

```python
# source/security/adapters/base.py
from abc import ABC, abstractmethod
from typing import Any, Optional
import numpy as np

from source.security.types import Observation, Action

class PolicyAdapter(ABC):
    adapter_type: str            # "llm" | "vla" | "wam" | "brain"
    policy_id: str

    # ---- 生命周期 ----
    def setup(self, ctx: "EpisodeContext") -> None: ...
    def reset(self) -> None: ...
    def teardown(self) -> None: ...

    # ---- 核心 ----
    @abstractmethod
    def act(self, obs: Observation) -> Action: ...

    # ---- 可选能力 ----
    def plan(self, obs: Observation) -> Optional[str]:
        """返回自然语言计划/推理。LLM/WAM 必实现，VLA 可返回 None。"""
        return None

    def get_logits(self, obs: Observation) -> Optional[np.ndarray]:
        """离散动作或词表 logits。白盒攻击需要时调用。"""
        return None

    def get_internal_state(self) -> dict[str, Any]:
        """暴露白盒探针。键空间约定见下表。"""
        return {}

    def load_checkpoint(self, path: str, *, strict: bool = True) -> None: ...

    # ---- 元信息 ----
    def capabilities(self) -> set[str]:
        """声明实现了哪些可选方法，避免 runner 反射判断。"""
        return {"act"}
```

### 必须 / 可选实现矩阵

| 方法 | `llm` | `vla` | `wam` | `brain` |
| --- | --- | --- | --- | --- |
| `act` | ✅ | ✅ | ✅ | ✅ |
| `plan` | ✅ | 可选 | ✅ | ✅ |
| `get_logits` | 黑盒可省 | 白盒攻击需 | 白盒攻击需 | 通常省 |
| `get_internal_state` | 可选 | 可选 | ✅ 推荐 | 可选 |
| `load_checkpoint` | ✅ | ✅ | ✅ | ✅ |

`capabilities()` 用于 runner 在装配阶段就拒绝"白盒攻击 + 黑盒策略"这种不兼容组合。

### `get_internal_state` 键空间约定

| 键 | 类型 | 说明 |
| --- | --- | --- |
| `hidden_states.<layer>` | `np.ndarray` | 隐藏层激活，layer 名由模型自决 |
| `attention.<layer>` | `np.ndarray` | 注意力权重 |
| `world_model.predicted_frames` | `np.ndarray` | WAM 预测的未来帧 |
| `world_model.value` | `float` | WAM 评估的当前价值 |
| `brain.subgoals` | `list[str]` | Brain 当前子目标栈 |

未实现的键直接不返回；不要返回 `None`。

---

## 输入与输出

### 输入

| 方法 | 输入 |
| --- | --- |
| `setup(ctx)` | `EpisodeContext`，含 `policy_id`、日志句柄、workspace |
| `act(obs)` | `Observation` |
| `plan(obs)` | `Observation` |
| `get_logits(obs)` | `Observation` |
| `load_checkpoint(path)` | 文件路径或 HF 仓库 id |

`Observation` 字段约束（按适配器类型）：

| 适配器 | 必读字段 | 推荐字段 |
| --- | --- | --- |
| `llm` | `instruction` | `images["head"]`（多模态时） |
| `vla` | `instruction`, `images`, `proprio` | `depth` |
| `wam` | `instruction`, `images`, `proprio` | `depth`, `extra.object_poses` |
| `brain` | `instruction`, `images`, `proprio`, `extra` | — |

适配器**不允许直接读 `EnvState`**——那是评估器特权。

### 输出

`act()` 必须返回 `Action`。各适配器的常见 `Action.type` 取值：

| 适配器 | 常见 `type` | 备注 |
| --- | --- | --- |
| `llm` | `text_only` / `discrete` | 无连续控制能力时只能产文本 |
| `vla` | `ee_delta` / `joint_pos` | 由模型 head 决定 |
| `wam` | `ee_delta` + `text`（plan 字段） | `text` 字段附带规划 |
| `brain` | `ee_delta` / `joint_pos` + 长 `text` | 内部多步推理被压缩成单步动作 |

`Action.meta` 推荐字段：`latency_ms` (float)、`confidence` (float in [0,1])、`adapter_id` (str)。

---

## 配置（在 ArenaConfig 中的对应块）

```yaml
policy:
  id: <str>                    # 必填，本次评测的策略 ID
  adapter: <"llm"|"vla"|"wam"|"brain">
  module: <python.path>        # PolicyAdapter 子类
  config:                      # 透传给子类构造函数
    checkpoint: <str>
    device: <str>              # 默认 "cuda:0"
    dtype: <"fp32"|"fp16"|"bf16">
    max_new_tokens: <int>      # llm/wam/brain
    temperature: <float>
    action_chunk_size: <int>   # vla/wam
    action_space:              # 可选，显式声明输出
      type: <"ee_delta"|"joint_pos"|...>
      dim: <int>
      bounds: [[<float>, <float>], ...]
    api:                       # 远端模型才填
      base_url: <str>
      api_key_env: <str>
      timeout: <int>
```

字段说明：

| 字段 | 必填 | 备注 |
| --- | --- | --- |
| `id` | ✅ | 日志与指标 breakdown 用 |
| `adapter` | ✅ | 必须与 `module` 中实现的 `adapter_type` 一致，否则 runner 报错 |
| `module` | ✅ | 形如 `source.security.adapters.vla.OpenVLAAdapter` |
| `config.checkpoint` | ✅ | 本地路径或 HF 仓库 id |
| `config.action_space` | ❌ | 不填时 runner 在第一次 `act()` 后从返回值推断；填了可校验 |
| `config.api.*` | 视情况 | 远端 API 必填；本地模型留空 |

---

## 最小可用示例

### LLM/VLM 适配器

```python
# source/security/adapters/llm.py
from source.security.adapters.base import PolicyAdapter
from source.security.types import Action

class GPTAdapter(PolicyAdapter):
    adapter_type = "llm"

    def __init__(self, config: dict):
        self.client = build_openai_client(config["api"])
        self.model = config["checkpoint"]      # e.g. "gpt-4o-mini"
        self.system = config.get("system_prompt",
            "You are a household robot. Output JSON {action, plan}.")

    def act(self, obs):
        messages = self._build_messages(obs)
        resp = self.client.chat.completions.create(
            model=self.model, messages=messages, max_tokens=512)
        parsed = parse_json(resp.choices[0].message.content)
        return Action(type="text_only",
                      payload=parsed.get("action", ""),
                      text=parsed.get("plan"),
                      meta={"adapter_id": "gpt", "tokens": resp.usage.total_tokens})

    def plan(self, obs):
        return self.act(obs).text

    def capabilities(self):
        return {"act", "plan"}
```

### VLA 适配器

```python
# source/security/adapters/vla.py
import torch

class OpenVLAAdapter(PolicyAdapter):
    adapter_type = "vla"

    def __init__(self, config):
        self.model = load_openvla(config["checkpoint"], device=config["device"])
        self.device = config["device"]
        self.action_chunk = config.get("action_chunk_size", 1)

    def act(self, obs):
        img = obs.images["head"]
        instr = obs.instruction
        with torch.no_grad():
            delta = self.model.predict_action(image=img, instruction=instr,
                                              proprio=obs.proprio["joint_pos"])
        return Action(type="ee_delta",
                      payload=delta.cpu().numpy(),
                      meta={"adapter_id": "openvla"})

    def get_logits(self, obs):
        img = obs.images["head"]
        return self.model.action_logits(img, obs.instruction).cpu().numpy()

    def capabilities(self):
        return {"act", "get_logits", "load_checkpoint"}
```

---

## 自定义扩展

1. 继承 `PolicyAdapter`，设置 `adapter_type`，至少实现 `__init__(self, config)` 与 `act()`。
2. 在 `source/security/adapters/__init__.py` 用 `@register_adapter("my_adapter")` 注册（可选；YAML 也可直接用全限定路径）。
3. 多 GPU / 分布式推理在适配器内部封装；对外仍是同步 `act()`。
4. **严禁**在适配器里直接调用 task 或仿真器接口；所需信息都从 `Observation` 拿。
5. 远端 API 适配器必须实现超时与重试，并在 `Action.meta` 记录 `latency_ms`、`retry_count`。
6. 若策略本身需要随机性，**只用 `EpisodeContext.seed` 派生子种子**，保证 reproducibility。

---

## 与其他接口的交互

| 接口 | 交互方式 |
| --- | --- |
| [SecurityTask](security_task.md) | 通过 `Observation` 间接消费任务输出 |
| [ObservationHook](observation_hook.md) | obs 在到达 `act()` 之前可能被改写；适配器无感 |
| [ActionHook](action_hook.md) | `act()` 返回的动作可能被改写后再落到仿真器；适配器无感 |
| [AttackModule](attack_module.md) | 白盒攻击会调 `get_logits` / `get_internal_state`；适配器须如实暴露能力声明 |
| [DefenseModule](defense_module.md) | "模型对齐"类防御可能 wrap 适配器（装饰器模式），对外仍是 `PolicyAdapter` |
| [SecurityEvaluator](security_evaluator.md) | 读取 `Action.text` / `Action.meta` 用于 VLM 判定 |
