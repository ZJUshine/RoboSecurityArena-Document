# Supported List

具身智能安全攻防竞技场支持的被测模型、攻击方法、防御方法和安全基准。

状态说明：

- **支持**：平台已复现。
- **部分支持**：平台尚未完整复现。
- **支持中**：已经确认复现条件，但正在复现。
- **未支持**：需要进一步确认复现条件。

## 被测模型

### LLM / VLM

| 模型名字 | 论文链接 | 代码链接 | 是否支持 |
| --- | --- | --- | --- |
| VoxPoser | [Paper](https://arxiv.org/abs/2307.05973) | [Code](https://github.com/huangwl18/VoxPoser) | 支持中 |
| ReKep | [Paper](https://arxiv.org/abs/2409.01652) | [Code](https://github.com/huangwl18/ReKep) | 支持中 |

### VLA

| 模型名字 | 论文链接 | 代码链接 | 是否支持 |
| --- | --- | --- | --- |
| OpenVLA | [Paper](https://arxiv.org/abs/2406.09246) | [Code](https://github.com/openvla/openvla) | 支持中 |
| OpenVLA-OFT | [Paper](https://arxiv.org/abs/2502.19645) | [Code](https://github.com/moojink/openvla-oft) | 支持中 |
| π0          | [Paper](https://arxiv.org/abs/2410.24164) | [Code](https://github.com/Physical-Intelligence/openpi) | 支持     |
| π0.5 | [Paper](https://arxiv.org/abs/2504.16054) | [Code](https://github.com/Physical-Intelligence/openpi) | 支持中 |
| GR00T-N1.5 | [Project](https://research.nvidia.com/labs/gear/gr00t-n1_5/) | [Code](https://github.com/NVIDIA/Isaac-GR00T) | 支持中 |
| GR00T-N1.6 | [Project](https://research.nvidia.com/labs/gear/gr00t-n1_6/) | [Code](https://github.com/NVIDIA/Isaac-GR00T) | 支持中 |

### WAM

| 模型名字 | 论文链接 | 代码链接 | 是否支持 |
| --- | --- | --- | --- |
| DreamZero | [Paper](https://arxiv.org/abs/2602.15922) | [Code](https://github.com/dreamzero0/dreamzero) | 未支持 |
| LingBot-VA | [Paper](https://arxiv.org/abs/2601.21998) | [Code](https://github.com/robbyant/lingbot-va) | 未支持 |
| RynnVLA-002 | [Paper](https://arxiv.org/abs/2511.17502) | [Code](https://github.com/alibaba-damo-academy/RynnVLA-002) | 未支持 |

### Brain

| 模型名字 | 论文链接 | 代码链接 | 是否支持 |
| --- | --- | --- | --- |
| RoboBrain 2.0 | [Paper](https://arxiv.org/abs/2507.02029) | [Code](https://github.com/FlagOpen/RoboBrain2.0) | 支持中 |
| RoboBrain 2.5 | [Paper](https://arxiv.org/abs/2601.14352) | [Code](https://github.com/FlagOpen/RoboBrain2.5) | 支持中 |
| Cosmos-Reason1 | [Paper](https://arxiv.org/abs/2503.15558) | [Code](https://github.com/nvidia-cosmos/cosmos-reason1) | 支持中 |
| Cosmos-Reason2 | [Blog](https://docs.nvidia.com/cosmos/latest/reason2/index.html) | [Code](https://github.com/nvidia-cosmos/cosmos-reason2) | 支持中 |

## 攻击方法

### 越狱攻击

这类方法尝试绕过安全策略、系统提示、权限边界或任务约束，让模型执行危险、越权或与安全规则冲突的动作。

| 方法名字 | 论文链接 | 开源链接 | 是否支持 |
| --- | --- | --- | --- |
| BadRobot: Jailbreaking Embodied LLMs in the Physical World | [Paper](https://arxiv.org/abs/2407.20242) | [Code](https://github.com/Rookie143/BadRobot) | 支持中 |
| RoboPAIR: Jailbreaking LLM-Controlled Robots | [Paper](https://arxiv.org/abs/2410.13691) | [Code](https://github.com/arobey1/robopair) | 支持中 |
| POEX: Policy Executable Jailbreak Attacks against Embodied AI | [Paper](https://arxiv.org/abs/2412.16633) | [Project](https://poex-eai-jailbreak.github.io/) | 支持中 |

### 对抗攻击

这类方法构造对抗样本、对抗纹理或物理扰动，使视觉和语言在输入看似合理时模型产生错误判断。

| 方法名字 | 论文链接 | 开源链接 | 是否支持 |
| --- | --- | --- | --- |
| Exploring the Adversarial Vulnerabilities of Vision-Language-Action Models in Robotics | [Paper](https://arxiv.org/abs/2411.13587) | [Code](https://github.com/William-wAng618/roboticAttack) | 支持中 |
| Adversarial Attacks on Robotic Vision-Language-Action Models | [Paper](https://arxiv.org/abs/2506.03350) | [Code](https://github.com/eliotjones1/robogcg) | 支持中 |

### 后门攻击

这类方法在模型、视觉编码器、策略网络或机器人技能中植入隐蔽触发器，使系统在特定触发条件下执行异常行为。

| 方法名字 | 论文链接 | 开源链接 | 是否支持 |
| --- | --- | --- | --- |
| Can We Trust Embodied Agents? Exploring Backdoor Attacks against Embodied LLM-based Decision-Making Systems | [Paper](https://arxiv.org/abs/2405.20774) | [Code](https://github.com/Daniel-xsy/BALD) | 支持中 |
| BadVLA: Towards Backdoor Attacks on Vision-Language-Action Models via Objective-Decoupled Optimization | [Paper](https://arxiv.org/abs/2505.16640) | [Code](https://github.com/Zxy-MLlab/BadVLA) | 支持中 |

### 其他

其他攻击可能包括提示词注入攻击、传感器攻击等

| 方法名字 | 论文链接 | 开源链接 | 是否支持 |
| --- | --- | --- | --- |
| Phantom Menace: Exploring and Enhancing the Robustness of VLA Models Against Physical Sensor Attacks | [Paper](https://arxiv.org/abs/2511.10008) | [Code](https://github.com/ZJUshine/Phantom-Menace) | 支持 |

## 防御方法

### 输入过滤

这类方法在模型推理前过滤危险指令、恶意上下文和越权请求。

| 方法名字 | 论文链接 | 开源链接 | 是否支持 |
| --- | --- | --- | --- |
| Finetune Llama-Guard-3 | [Blog](https://www.llama.com/docs/model-cards-and-prompt-formats/llama-guard-3/) | [Code](https://github.com/meta-llama/llama-cookbook/blob/main/getting-started/responsible_ai/llama_guard/llama_guard_finetuning_multiple_violations_with_torchtune.ipynb) | 支持中 |

### 提示词约束

这类方法把安全规则和动作约束显式加入任务规划的提示词。

| 方法名字                                                     | 论文链接                                  | 开源链接                                               | 是否支持 |
| ------------------------------------------------------------ | ----------------------------------------- | ------------------------------------------------------ | -------- |
| COT of SafeAgentBench                                        | [Paper](https://arxiv.org/abs/2412.13178) | [Code](https://github.com/shengyin1224/SafeAgentBench) | 支持中  |
| Don’t Let Your Robot be Harmful: Responsible Robotic Manipulation via Safety-as-Policy | [Paper](https://arxiv.org/abs/2411.18289) | [Code](https://github.com/kodenii/Responsible-Robotic-Manipulation/) | 支持中 |
| Generating Robot Constitutions & Benchmarks for Semantic Safety | [Paper](https://arxiv.org/abs/2503.08663) | [Project](https://asimov-benchmark.github.io/) | 支持中 |

### 模型对齐

这类方法通过训练、微调、偏好对齐或遗忘学习，让模型在面对危险指令时主动拒绝或选择更安全的行为。

| 方法名字 | 论文链接 | 开源链接 | 是否支持 |
| --- | --- | --- | --- |
| VLA-Forget: Vision-Language-Action Unlearning for Embodied Foundation Models | [Paper](https://arxiv.org/abs/2604.03956) | [Code](https://github.com/raviranjan-ai/VLA-Forget) | 支持中 |
| SafeVLA: Towards Safety Alignment of Vision-Language-Action Model via Constrained Learning | [Paper](https://arxiv.org/abs/2503.03480) | [Code](https://github.com/PKU-Alignment/SafeVLA) | 支持中 |

### 安全检测

这类方法在运行时通过内部状态或者外部行为检测异常轨迹和危险动作。

| 方法名字 | 论文链接 | 开源链接 | 是否支持 |
| --- | --- | --- | --- |
| Advancing Embodied Agent Security: From Safety Benchmarks to Input Moderation | [Paper](https://arxiv.org/abs/2504.15699) | [Code](https://github.com/ZihanYan-CQU/EAsafetyBench) | 支持中 |
| SafePlan: Leveraging Formal Logic and Chain-of-Thought Reasoning for Enhanced Safety in LLM-based Robotic Task Planning | [Paper](https://arxiv.org/abs/2503.06892) | [Project](https://ikeobie.com/safeplan.html) | 支持中 |

## 基准

| 基准名字 | 论文链接 | 开源链接 | 是否支持 |
| --- | --- | --- | --- |
| SafeAgentBench | [Paper](https://arxiv.org/abs/2412.13178) | [Code](https://github.com/shengyin1224/SafeAgentBench) | 支持中 |
| BadRobot Benchmark | [Paper](https://arxiv.org/abs/2407.20242) | [Code](https://github.com/Rookie143/BadRobot) | 支持中 |
| EARBench | [Paper](https://arxiv.org/abs/2408.04449) | [Code](https://github.com/zihao-ai/EARBench) | 支持中 |
| Harmful-RLBench | [Paper](https://arxiv.org/abs/2412.16633) | [Project](https://poex-eai-jailbreak.github.io/) | 支持中 |
