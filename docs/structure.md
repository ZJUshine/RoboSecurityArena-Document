## 🗂️ 仓库整体架构

### 一、顶层目录

| 目录 | 说明 |
| --- | --- |
| `source/` | 所有源代码主目录 |
| `scripts/` | 根级启动脚本（Docker 入口、GUI、无头模式等） |
| `output/` | 基准测试结果输出 |
| `docs/` | 文档和图片资源 |
| `3rdparty/` | 第三方文件 |

---

### 二、`source/` 核心子模块

| 子模块 | 路径 | 主要功能 |
| --- | --- | --- |
| **GenieSim 仿真平台** | `source/geniesim/` | 高保真机器人仿真、场景生成、评估 |
| **数据采集系统** | `source/data_collection/` | 合成数据采集、任务规划、传感器仿真 |
| **场景重建** | `source/scene_reconstruction/` | 3DGS 三维重建、USD 格式转换 |
| **遥操作系统** | `source/teleop/` | VR 遥操作控制与数据记录 |

---

### 三、`source/geniesim/` — 仿真平台核心

| 子模块 | 路径 | 功能说明 |
| --- | --- | --- |
| **仿真应用** | `geniesim/app/` | Isaac Sim 主程序，集成 cuRobo 运动规划、ROS 通信 |
| **基准评估** | `geniesim/benchmark/` | 200+ 任务 / 100,000+ 场景，自动化评测 |
| **LLM 场景生成** | `geniesim/generator/` | 由 Claude/OpenAI 驱动的场景与评估自动生成 |
| **VLM 评估器** | `geniesim/evaluator/` | 视觉语言模型自动打分 |
| **3D 资产库** | `geniesim/assets/` | 5,140 个验证三维模型（家居、工业、餐饮等） |
| **插件系统** | `geniesim/plugins/` | 可扩展插件（ADER 任务、TGS 场景生成、GUI） |
| **机器人模型** | `geniesim/robot/` | G1/G2 机器人 URDF 及工具函数 |
| **全局配置** | `geniesim/config/` | 主配置 YAML、12 个 ICRA 任务配置、遥操参数 |
| **遥操作** | `geniesim/teleop/` | 仿真侧遥操模块，支持 VR 输入与回放 |
| **工具库** | `geniesim/utils/` | IK-SDK、URDF 求解器、USD 工具 |

---

### 四、`source/data_collection/` — 数据采集系统

| 子模块 | 路径 | 功能说明 |
| --- | --- | --- |
| **客户端** | `client/` | 自主 Agent、2D 布局生成、任务规划协调 |
| ↳ Agent | `client/agent/` | 自主决策代理（OmniAgent） |
| ↳ 布局生成 | `client/layout/` | 场景对象放置与 2D 布局求解 |
| ↳ 动作规划 | `client/planner/` | 抓取/放置/旋转等动作编排 |
| **服务器** | `server/` | Isaac Sim 仿真后端，gRPC 通信 |
| ↳ 运动控制 | `server/controllers/` | 运动学求解、夹爪控制、Ruckig 轨迹 |
| ↳ 运动生成 | `server/motion_generator/` | cuRobo 运动规划集成 |
| ↳ 传感器发布 | `server/ros_publisher/` | RGB-D、IMU、LiDAR 等 ROS2 发布 |
| ↳ 数据记录 | `server/recording/` | ROS Bag 提取、仿真数据格式转换 |
| **任务库** | `tasks/geniesim_2025/` | 200+ 任务模板，支持 G1/G2 双机器人版本 |
| **通用库** | `common/aimdk/` | 硬件抽象层（HAL）、通信协议定义 |
| **执行脚本** | `scripts/` | 服务器启动、数据采集、Docker 入口脚本 |

---

### 五、`source/scene_reconstruction/` — 场景重建

| 组件 | 说明 |
| --- | --- |
| **重建管道** | COLMAP + PGSR 高保真三维重建 |
| **gsplat 库** | 3D Gaussian Splatting，支持 CUDA 加速与分布式训练 |
| **USD 转换** | 将 3DGS 结果导出为 Isaac Sim 可用的 USD 格式 |
| **补丁集** | COLMAP、PGSR、hloc 定制补丁 |
| **Docker** | 独立重建容器 `Dockerfile` |

---

### 六、根级脚本 (`scripts/`)

| 脚本 | 功能 |
| --- | --- |
| `entrypoint.sh` | Docker 主入口 |
| `start_gui.sh` / `start_headless.sh` | GUI / 无头模式启动 |
| `start_generator.sh` | 启动 LLM 场景生成服务 |
| `run_tasks.sh` / `run_icra_tasks.sh` | 运行评估任务 / ICRA 任务 |
| `start_auto_record.sh` | 自动数据采集记录 |
| `collect_scores.sh` | 汇总评估分数 |
| `stat_score.py` / `stat_average.py` | 统计与分析评分 |
| `plot_operation.py` / `plot_cognition.py` | 操作/认知能力可视化 |
| `autoteleop.sh` | 自动遥操作 |