# 接口 5：`ObservationHook` —— 观测中间件

## 角色与定位

`ObservationHook` 是仿真器观测路径上的**可链式中间件**，挂在 `data_courier` 与 `api_core` 之间：

```
sim ──► data_courier ──► [ObservationHook 1] ──► [ObservationHook 2] ──► api_core ──► PolicyAdapter
                                          ▲                                   │
                                          │                                   ▼
                                       AttackModule / DefenseModule 注册的 hook
                                                                              SecurityEvaluator
                                                                              (与 policy 同看同一份)
```

设计目标：

- **传感器攻击**（Phantom Menace、对抗补丁、雾化、噪声等）和**输入异常检测**（OOD、对抗扰动检测器）走同一个抽象，避免每个攻击/防御自己叉链路；
- **可链式 + 可重排**，runner 根据 YAML 顺序构造；
- **可旁路**：hook 可声明 `read_only=True`，仅观察不改写。

---

## 类签名

```python
# source/security/hooks/observation.py
from abc import ABC, abstractmethod
from source.security.types import Observation, EpisodeContext

class ObservationHook(ABC):
    hook_id: str
    read_only: bool = False

    def setup(self, ctx: "RunContext") -> None: ...
    def pre_episode(self, ep_ctx: EpisodeContext) -> None: ...
    def post_episode(self, ep_ctx: EpisodeContext) -> None: ...
    def teardown(self) -> None: ...

    @abstractmethod
    def on_observation(self,
                       obs: Observation,
                       ep_ctx: EpisodeContext) -> Observation:
        """处理（或观察）一次观测。read_only=True 时返回值会被 runner 丢弃。"""
```

约束：

1. `on_observation` **必须返回一个 `Observation`**（即便不改也要 return）；
2. **不要**原地修改图像数组（`obs.images["head"][:] = ...`）—— runner 后续把同一份 obs 同时交给 evaluator；应深拷贝或新建 dict；
3. 任何修改都应在 `obs.meta["hooks"]` 中追加一条记录：

```python
obs.meta.setdefault("hooks", []).append({
    "hook_id": self.hook_id,
    "step": obs.step,
    "ts": time.time(),
    "summary": "added gaussian noise σ=0.02",
})
```

`meta.hooks` 是排查"为什么策略看到了奇怪输入"的关键证据，写入轨迹文件。

---

## 链式调用顺序

runner 按以下顺序构造观测链：

1. **AttackModule 注册的 obs hook**（按 `arena.yaml` 中 `attack` 列表顺序）；
2. **DefenseModule 注册的 obs hook**（按 `defense` 列表顺序）；
3. **`hooks.observation` 中显式声明的 hook**（追加到尾部）。

> 默认顺序：**先攻击后防御**。语义上：攻击注入扰动 → 防御尝试检测/还原 → 策略消费。
>
> 若要改变默认顺序，可在每个 hook 的 `config` 中显式设置 `priority`（int，越小越靠前）。

---

## 输入与输出

### 输入

- `obs: Observation`：上一个 hook（或仿真器）的输出；
- `ep_ctx: EpisodeContext`：可读 `attack_id` / `defense_id` / `shared` 字段，做模块间通信。

### 输出

- `obs': Observation`：交给下一个 hook 或 policy。

副作用建议：

| 副作用 | 推荐做法 |
| --- | --- |
| 想标记"被攻击" | `obs.meta["attack_active"] = self.hook_id` |
| 想给评估器留证据 | `ep_ctx.shared.setdefault(f"{self.hook_id}.frames", []).append(idx)` |
| 想终止 episode | **不允许**直接终止；可以把 `obs.meta["abort"] = reason`，runner 在下一拍读取 |

---

## 配置（在 ArenaConfig 中的对应块）

```yaml
hooks:
  observation:
    - module: source.security.hooks.obs.GaussianNoise
      config:
        sigma: 0.02
        targets: ["images.head"]
        priority: 100
    - module: source.security.hooks.obs.OODGuard
      config:
        threshold: 0.7
        log_only: true             # 等价于 read_only
        priority: 200
```

也支持简写（无 config 时）：

```yaml
hooks:
  observation:
    - source.security.hooks.obs.LatencyLogger
```

### 字段说明

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `module` | — | hook 的导入路径 |
| `config.priority` | `1000` | 越小越靠前；当与 attack/defense 注册的 hook 冲突时按数值排序 |
| `config.read_only` / `log_only` | `false` | 仅观察 |
| `config.<其他>` | — | 透传给 hook 构造函数 |

> `attack` / `defense` 中声明的 hook **不在** `hooks.observation` 里显式列出，由对应模块在 `setup()` 时自行注册（见 [AttackModule](attack_module.md) / [DefenseModule](defense_module.md)）。

---

## 最小可用示例

### 高斯噪声注入（攻击/调试）

```python
# source/security/hooks/obs/noise.py
import numpy as np
from source.security.hooks.observation import ObservationHook
from source.security.types import Observation
from copy import deepcopy

class GaussianNoise(ObservationHook):
    def __init__(self, sigma: float = 0.02, targets=("images.head",)):
        self.hook_id = f"gaussian_noise_{sigma}"
        self.sigma = sigma
        self.targets = targets
        self._rng = None

    def pre_episode(self, ep_ctx):
        self._rng = np.random.default_rng(ep_ctx.seed)

    def on_observation(self, obs, ep_ctx):
        new = deepcopy(obs)
        for path in self.targets:
            group, key = path.split(".")
            arr = getattr(new, group)[key].astype(np.float32) / 255.0
            arr = np.clip(arr + self._rng.normal(0, self.sigma, arr.shape), 0, 1)
            getattr(new, group)[key] = (arr * 255).astype(np.uint8)
        new.meta.setdefault("hooks", []).append(
            {"hook_id": self.hook_id, "step": obs.step,
             "summary": f"noise σ={self.sigma} on {list(self.targets)}"})
        return new
```

### OOD 检测（防御 + 只读）

```python
class OODGuard(ObservationHook):
    read_only = True

    def __init__(self, threshold: float = 0.7, model_path: str = ".../ood.pt"):
        self.hook_id = "ood_guard"
        self.detector = load_ood_model(model_path)
        self.threshold = threshold

    def on_observation(self, obs, ep_ctx):
        score = self.detector.score(obs.images["head"])
        if score > self.threshold:
            ep_ctx.logger.warning(f"OOD score {score:.3f} at step {obs.step}")
            ep_ctx.shared.setdefault("ood_guard.flagged_steps", []).append(obs.step)
        return obs    # read_only：返回值被丢弃，但仍要 return
```

---

## 自定义扩展

1. 继承 `ObservationHook`，至少实现 `__init__(self, **config)` 与 `on_observation`；
2. 用 `@register_obs_hook("my_hook")` 注册（可选；YAML 也接受全限定路径）；
3. 若 hook 内部带有可训练状态（如对抗补丁生成器），可在 `pre_episode` 重置；
4. 性能：单步预算 **< 10 ms**（CPU）或 **< 20 ms**（GPU），超出请异步化并把结果写入下一步；
5. 调试：runner 在 `--debug` 模式下会保存每个 hook 的 in/out 图像 diff 到 `episodes/ep#/hook_diff/`。

---

## 与其他接口的交互

| 接口 | 关系 |
| --- | --- |
| [ActionHook](action_hook.md) | 对偶结构，作用于动作路径 |
| [AttackModule](attack_module.md) | 通过 `register_observation_hook()` 把扰动器注入到链路头部 |
| [DefenseModule](defense_module.md) | 通过同名 API 注入检测器；防御 hook 默认排在攻击 hook 之后 |
| [PolicyAdapter](policy_adapter.md) | 看到链路最末端的 obs，无法感知中间是否被改 |
| [SecurityEvaluator](security_evaluator.md) | 看到与 policy **一致**的 obs，以保证"判它看到了什么" |
| [SecurityTask](security_task.md) | 产生原始 obs，不感知 hook |
