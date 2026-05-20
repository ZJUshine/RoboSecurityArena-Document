# 接口 6：`ActionHook` —— 动作中间件

## 角色与定位

`ActionHook` 是 `PolicyAdapter` 输出动作到仿真器之间的**可链式中间件**：

```
PolicyAdapter.act() ──► [ActionHook 1] ──► [ActionHook 2] ──► SecurityTask.step()
                                     ▲                  │
                                     │                  ▼
                              AttackModule / DefenseModule 注册的 hook
                              SecurityEvaluator (看到链路末端动作)
```

常见场景：

| 用途 | 类别 |
| --- | --- |
| **安全门控**：检测到危险动作直接替换为 `noop` 或紧急停止 | 防御 |
| **动作平滑/限幅**：clip 到关节/力矩边界，避免硬件损坏 | 通用 |
| **动作扰动注入**：模拟执行器抖动或对抗扰动 | 攻击 |
| **后门触发器**：在特定 obs 条件下覆盖动作 | 攻击（后门） |
| **审计日志**：记录每步动作，不修改 | 防御 / 监控 |

`ActionHook` 与 `ObservationHook` 是镜像设计：相同的生命周期、相同的链式约定。

---

## 类签名

```python
# source/security/hooks/action.py
from abc import ABC, abstractmethod
from source.security.types import Observation, Action, EpisodeContext

class ActionHook(ABC):
    hook_id: str
    read_only: bool = False

    def setup(self, ctx: "RunContext") -> None: ...
    def pre_episode(self, ep_ctx: EpisodeContext) -> None: ...
    def post_episode(self, ep_ctx: EpisodeContext) -> None: ...
    def teardown(self) -> None: ...

    @abstractmethod
    def on_action(self,
                  action: Action,
                  obs: Observation,
                  ep_ctx: EpisodeContext) -> Action:
        """处理（或观察）策略返回的动作。read_only=True 时返回值被丢弃。"""
```

约束：

1. 必须返回 `Action`；如果决定"什么都不做"，**显式**返回 `Action(type="noop", payload=None)` 而不是原 action；
2. 不要原地修改 `payload`；用 `dataclasses.replace(action, payload=new_payload)`；
3. 任何修改在 `action.meta["hooks"]` 里追加：

```python
new_action = replace(action, payload=clipped)
new_action.meta.setdefault("hooks", []).append({
    "hook_id": self.hook_id,
    "step": obs.step,
    "summary": "clipped to joint limits",
    "diff_norm": float(np.linalg.norm(action.payload - clipped)),
})
return new_action
```

---

## 链式调用顺序

runner 构造动作链的默认顺序：

1. **AttackModule 注册的 action hook**（按 `attack` 列表顺序）；
2. **DefenseModule 注册的 action hook**（按 `defense` 列表顺序）；
3. **`hooks.action` 显式声明的 hook**（追加到尾部）；
4. **runner 内置的安全 clip**（始终最后一个，不可关闭）。

> 默认意图：攻击先改 → 防御过滤 → 全局兜底硬限幅。
>
> 同样支持 `priority` 数字覆盖顺序，越小越靠前。

---

## 输入与输出

### 输入

| 字段 | 含义 |
| --- | --- |
| `action` | 上一个 hook（或 policy）输出 |
| `obs` | **与 policy 看到的同一份观测**（policy 的输入），方便上下文判定 |
| `ep_ctx` | 含 `attack_id` / `defense_id` / `shared` |

### 输出

- `action': Action`：交给下一 hook 或 `task.step()`；
- 如果是 read-only：必须 `return action`，但返回值不被采用。

允许的副作用：

| 副作用 | 推荐做法 |
| --- | --- |
| 紧急停止 | 返回 `Action(type="noop")`，并 `action.meta["safety_stop"] = self.hook_id` |
| 标记拒答 | 返回 `Action(type="text_only", payload="(blocked) ..." , meta={"refused_by": self.hook_id})` |
| 留下证据 | `ep_ctx.shared.setdefault(f"{self.hook_id}.blocked_steps", []).append(obs.step)` |

> 把"拒答"显式编码进 `Action.type="text_only" + meta.refused`，evaluator 才能正确计入 RR / ORR。

---

## 配置（在 ArenaConfig 中的对应块）

```yaml
hooks:
  action:
    - module: source.security.hooks.action.JointLimitClip
      config:
        limits_path: configs/robots/g1_dual_arm.yaml
        priority: 100
    - module: source.security.hooks.action.SafetyGuard
      config:
        forbidden_regions: ["human_zone", "fragile_zone"]
        forbidden_objects: ["knife", "scissors"]
        action: "noop"          # 或 "refusal_text"
        priority: 200
```

### 字段说明

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `module` | — | hook 导入路径 |
| `config.priority` | `1000` | 与 attack/defense 注册的 hook 一起按数值排序 |
| `config.read_only` | `false` | 仅审计、不改动作 |
| `config.action` | — | 当某些 hook 需要决定"被拦截后输出什么"（noop / refusal_text / safety_pose） |
| `config.<其他>` | — | 透传 |

---

## 最小可用示例

### 关节限幅（通用安全）

```python
# source/security/hooks/action/clip.py
import numpy as np
from dataclasses import replace
from source.security.hooks.action import ActionHook
from source.security.types import Action

class JointLimitClip(ActionHook):
    def __init__(self, limits_path: str):
        self.hook_id = "joint_limit_clip"
        cfg = load_yaml(limits_path)
        self.lo = np.array(cfg["lower"])
        self.hi = np.array(cfg["upper"])

    def on_action(self, action, obs, ep_ctx):
        if action.type not in ("joint_pos", "joint_delta"):
            return action
        clipped = np.clip(action.payload, self.lo, self.hi)
        if np.allclose(clipped, action.payload):
            return action
        new = replace(action, payload=clipped)
        new.meta.setdefault("hooks", []).append({
            "hook_id": self.hook_id, "step": obs.step,
            "summary": "clipped to joint limits"})
        return new
```

### 安全门控（防御）

```python
class SafetyGuard(ActionHook):
    def __init__(self, forbidden_regions=(), forbidden_objects=(),
                 action: str = "noop"):
        self.hook_id = "safety_guard"
        self.regions = set(forbidden_regions)
        self.objects = set(forbidden_objects)
        self.fallback = action

    def on_action(self, action, obs, ep_ctx):
        # 简化版：用 obs.extra 里的预测末端位姿做粗判
        ee_target = obs.extra.get("predicted_ee_target")
        if ee_target is None:
            return action
        for region in self.regions:
            if point_in_region(ee_target, obs.extra["regions"][region]):
                blocked = Action(type="noop", payload=None,
                                 meta={"refused_by": self.hook_id,
                                       "reason": f"enter {region}"})
                ep_ctx.shared.setdefault("safety_guard.blocked", []).append(obs.step)
                return blocked
        return action
```

### 审计日志（只读）

```python
class ActionAuditLogger(ActionHook):
    read_only = True
    def __init__(self): self.hook_id = "action_audit"

    def on_action(self, action, obs, ep_ctx):
        ep_ctx.logger.info(
            f"step={obs.step} type={action.type} "
            f"text={action.text[:60] if action.text else ''}")
        return action
```

---

## 自定义扩展

1. 继承 `ActionHook`，至少实现 `__init__(self, **config)` 与 `on_action`；
2. 用 `@register_action_hook("my_hook")` 注册（可选）；
3. 性能预算：单步 **< 5 ms**；想做 VLM 后置过滤（慢）的话，请放到 `DefenseModule` 的 `audit()` 阶段（异步）；
4. 想拦截 + 给出替代动作？返回新 `Action`，并填好 `meta.refused / meta.replaced_by`；evaluator 会读这些字段；
5. 想完全否决（policy 等价于"啥都没做"）？返回 `Action(type="noop")`。

---

## 与其他接口的交互

| 接口 | 关系 |
| --- | --- |
| [ObservationHook](observation_hook.md) | 对偶 |
| [PolicyAdapter](policy_adapter.md) | 上游；hook 看到的是 `act()` 的原始返回 |
| [SecurityTask](security_task.md) | 下游；接收链路末端的 `Action` |
| [AttackModule](attack_module.md) | 注入后门、动作扰动通过此 hook |
| [DefenseModule](defense_module.md) | 注册"后置过滤"类防御通过此 hook |
| [SecurityEvaluator](security_evaluator.md) | 评估器看到的是链路末端的动作，与仿真器实际执行一致 |
