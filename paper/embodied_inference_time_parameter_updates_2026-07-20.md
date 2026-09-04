# 具身智能中的推理期参数更新：Test-Time Adaptation、Online Learning 与 Continual Robot Learning

> 调研日期：2026-07-20（Asia/Shanghai）
> 检索窗口：2018–2026；另补充少量更早的奠基工作
> 核心问题：具身智能体能否在部署/推理期间利用当前观测、交互反馈或经验流更新模型参数？这个方向到底应称为“在线学习”吗？

## 结论先行

如果你强调的是“**模型在测试/部署/推理阶段，根据测试输入或环境交互做梯度更新**”，最准确的主关键词是：

- **Test-Time Adaptation (TTA，测试时自适应)**：上位词；可能更新权重、归一化统计量、适配器，也可能只做非参数适应。
- **Test-Time Training (TTT，测试时训练)**：通常更明确地指在推理阶段构造自监督/反馈损失，并通过梯度下降更新参数。
- **Online Test-Time Adaptation / Continual Test-Time Adaptation (OTTA/CTTA)**：测试样本按流到达，模型持续更新，且要处理误差累积、分布连续变化和灾难性遗忘。

“**Online Learning（在线学习）**”并没有错，但它太宽：凡是数据逐步到来并随到随学，都可以叫 online learning；它不保证发生在测试阶段，也不保证是具身智能，更不保证更新的是神经网络参数。若研究目标是机器人长期部署后不断获得新技能并保留旧技能，则更常叫 **Continual/Lifelong Robot Learning**；若是通过少量在线轨迹快速适配新动力学/新任务，则还会落入 **online adaptation、meta-RL、adaptive control、online system identification**。

一句话建议：把方向暂定为 **“Test-Time Learning/Adaptation for Embodied Agents（具身智能体的测试时学习与自适应）”**，并把 **online/continual robot learning** 作为相邻大方向；检索时不要只搜 online learning。

## 概念边界：同叫 adaptation，更新的东西可能完全不同

| 标签             | 部署时变化的对象                          | 是否更新学习参数 | 典型名称                       | 本报告记号  |
| ---------------- | ----------------------------------------- | ---------------: | ------------------------------ | ----------- |
| 梯度式测试时训练 | 主干、适配器、价值函数或策略的权重        |               是 | TTT、gradient-based TTA        | **P** |
| 在线/持续训练    | episode 间或数据流上持续优化策略/世界模型 |               是 | online RL、continual learning  | **O** |
| 统计/小状态更新  | BN 统计量、卡尔曼参数、GP 后验、快速权重  |      部分/视定义 | online adaptation              | **S** |
| 隐变量适配       | 环境编码、belief、latent dynamics         |       否（通常） | rapid adaptation、meta-RL      | **L** |
| 外部记忆/检索    | 成功轨迹、技能库、世界模型库              |               否 | memory-based adaptation        | **M** |
| 推理搜索/采样    | action samples、规划候选、denoising path  |               否 | test-time scaling/optimization | **I** |

因此，RMA、in-context robot learning、检索增强 VLA、test-time sampling 虽然都能“推理时变强”，却不一定属于你要找的“推理时更新参数”。

## 一、最匹配：具身场景中确实更新权重/适配器/在线模型参数

| #   | 论文                                                                                                                                                                                                                                                            | 年份/发表                         | 更新类型 | 一句话总结                                                                                                                                                  |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| P1  | [Fast-Slow Test-Time Adaptation for Online Vision-and-Language Navigation](https://proceedings.mlr.press/v235/gao24p.html)                                                                                                                                       | 2024, ICML                        | P        | FSTTA 在在线 VLN 中联合分解和累积梯度与参数，用快/慢两种更新频率缓解逐步决策中的漂移，是“具身 + 推理时梯度更新”最直接的代表作之一。                       |
| P2  | [Test-Time Adaptation for Online Vision-Language Navigation with Feedback-based Reinforcement Learning](https://proceedings.mlr.press/v267/kim25ad.html)                                                                                                         | 2025, ICML                        | P/O      | FeedTTA 只用每个导航 episode 的成功/失败二值反馈做在线 RL 更新，并以梯度正则平衡可塑性与稳定性。                                                            |
| P3  | [VITA: Zero-Shot Value Functions via Test-Time Adaptation of Vision–Language Models](https://chziakas.github.io/vita/)                                                                                                                                          | 2026, ICLR                        | P        | 每个时间步用元学习得到的自监督损失对轻量适配模块做一次梯度更新，把轨迹历史编码进参数以改善机器人零样本价值估计。                                            |
| P4  | [RoboTTT: Context Scaling for Robot Policies](https://arxiv.org/abs/2607.15275)                                                                                                                                                                                  | 2026, arXiv                       | P/S      | 把梯度下降更新的 fast weights 当作 VLA 策略的循环状态，在训练和推理时持续把长达 8K 步的历史压缩进权重，是目前最贴近“边推理边改权重”的机器人基础模型工作。 |
| P5  | [Search-TTA: A Multimodal Test-Time Adaptation Framework for Visual Search in the Wild](https://search-tta.github.io/)                                                                                                                                           | 2025, CoRL                        | P        | 机器人搜索过程中根据过去测量做不确定性加权的在线梯度更新，动态修正卫星图像编码器产生的目标概率图。                                                          |
| P6  | [Embodied Perception for Test-time Grasping Detection Adaptation with Knowledge Infusion](https://arxiv.org/abs/2504.04795)                                                                                                                                      | 2025, arXiv                       | P/O      | 机器人主动换视角并用抓取可行性筛选自标注样本，再在新场景中优化抓取检测网络，实现无人工标注的具身 TTA。                                                      |
| P7  | [Test-time Model Adaptation for Robotic Grasping with Weakly Supervised Affordance Grounding](https://lemonqc.github.io/WeaklyTestAffordance/)                                                                                                                   | 预印本/项目信息不完整             | P        | 用 teacher–student、显著知识转移和熵约束在在线感知数据上优化 affordance grounding 模型，再驱动真实抓取。                                                   |
| P8  | [TTT-Parkour: Rapid Test-Time Training for Perceptive Robot Parkour](https://arxiv.org/abs/2602.02331)                                                                                                                                                           | 2026, arXiv                       | P/O      | 把新地形 RGB-D 重建成高保真仿真网格，并在部署前约十分钟内快速微调跑酷策略，属于“场景级 TTT”而非每步在线更新。                                             |
| P9  | [MixTBN: A Fully Test-Time Adaptation Method for Visual Reinforcement Learning on Robotic Manipulation](https://www.researchgate.net/publication/376565127_MixTBN_A_Fully_Test-Time_Adaptation_Method_for_Visual_Reinforcement_Learning_on_Robotic_Manipulation) | 2023, ICCASIT                     | S/P      | 仅凭目标域无标签在线数据混合/更新归一化相关参数，以缓解视觉机器人强化学习策略的域偏移；论文影响力和验证规模相对有限。                                       |
| P10 | [Online Continual Learning for Interactive Instruction Following Agents](https://openreview.net/forum?id=7M0EzjugaN)                                                                                                                                             | 2024, ICLR                        | O        | 为具身指令跟随定义行为增量和环境增量两种在线持续学习设定，并用无需任务边界的置信度移动平均缓解遗忘。                                                        |
| P11 | [Adaptive Robot Traversability Estimation Based on Self-Supervised Online Continual Learning in Unstructured Environments](https://doi.org/10.1109/LRA.2024.3386451)                                                                                             | 2024, RA-L                        | O        | Husky 机器人利用自身经验在线自监督训练可通行性模型，并通过 uncertainty-aware replay 应对环境变化和灾难性遗忘。                                              |
| P12 | [Efficient Continual Adaptation of Pretrained Robotic Policy with Online Meta-Learned Adapters](https://arxiv.org/abs/2503.18684)                                                                                                                                | 2025, arXiv                       | O/P      | OMLA 冻结预训练策略主体、顺序更新小型适配器，并以在线元学习目标促进新旧任务间迁移。                                                                         |
| P13 | [Online Model Adaptation with Feedforward Compensation](https://proceedings.mlr.press/v229/abuduweili23a.html)                                                                                                                                                   | 2023, CoRL                        | O/P      | 从记忆缓冲区选择关键旧样本而非只用最新误差，实时更新预测/动力学模型并减轻慢变系统中的遗忘。                                                                 |
| P14 | [Grow Your Limits: Continuous Improvement with Real-World RL for Robotic Locomotion](https://arxiv.org/abs/2310.17634)                                                                                                                                           | 2024, ICRA                        | O/P      | APRL 在真实四足机器人上持续更新策略，通过基于动力学预测误差的探索正则实现分钟级学会行走并随训练继续提升。                                                   |
| P15 | [Dynamic Rank Adjustment in Diffusion Policies for Efficient and Flexible Training](https://arxiv.org/abs/2502.03822)                                                                                                                                            | 2025, RSS                         | O/P      | DRIFT 用 SVD 动态调节扩散策略的可训练秩，使 DRIFT-DAgger 能从离线启动平滑进入在线交互模仿学习。                                                             |
| P16 | [Meta-Learning Online Dynamics Model Adaptation in Off-Road Autonomous Driving](https://www.roboticsproceedings.org/rss21/p139.html)                                                                                                                             | 2025, RSS                         | S/O      | 离线元学习可适配的动力学基函数，部署时用卡尔曼滤波实时调整车载动力学模型参数以提高越野驾驶安全性。                                                          |
| P17 | [Action Flow Matching for Continual Robot Learning](https://arxiv.org/abs/2504.18471)                                                                                                                                                                            | 2025, RSS                         | O/P      | 用 flow matching 修正计划动作以主动采集更有信息的转移，同时在线更新动力学模型，在 UGV 和四旋翼的变化动力学下加速重对齐。                                    |
| P18 | [Preserving and Combining Knowledge in Robotic Lifelong Reinforcement Learning](https://doi.org/10.1038/s42256-025-00983-2)                                                                                                                                      | 2025, Nature Machine Intelligence | O        | 用非参数贝叶斯知识空间、任务推断和策略学习持续积累、组合机器人技能，重点是跨任务长期更新与抗遗忘。                                                          |
| P19 | [Getting Robots Back on Track by Reconstituting Control in Unexpected Situations with Online Learning](https://www.nature.com/articles/s41467-026-70256-y)                                                                                                       | 2026, Nature Communications       | S/O      | FLAIR 在嵌入式平台上用高斯过程在线学习扰动到控制修正的映射，使受损或受扰机器人无需任务专用预训练即可恢复可操作性。                                          |
| P20 | [Self-Supervised Online Robot-Agnostic Traversability Estimation for Open-World Environments](https://arxiv.org/abs/2605.28442)                                                                                                                                  | 2026, arXiv                       | O/P      | COTRATE 从多模态无标签机器人经验持续训练视觉可通行性模型，针对开放世界部署中的平台迁移和遗忘问题。                                                          |

### 这一组中最值得优先读的 6 篇

1. **FSTTA (ICML 2024)**：最清楚地定义了在线 VLN 中“每步/跨样本参数更新”的特殊困难。
2. **FeedTTA (ICML 2025)**：把测试时更新从无监督熵最小化推进到真实交互反馈 RL。
3. **VITA (ICLR 2026)**：轻量适配模块 + 元学习自监督损失，机制干净、具身任务明确。
4. **RoboTTT (2026)**：把 fast weights 直接作为机器人策略的长时记忆，代表最新 VLA/TTT 结合趋势。
5. **Search-TTA (CoRL 2025)**：真实户外搜索、在线梯度反馈、规划器无关，具身闭环很完整。
6. **Online Continual Learning for Interactive Instruction Following (ICLR 2024)**：如果你更关心长期跨任务学习而不是单场景 TTA，这是最好的概念桥梁。

## 二、强相关但通常不更新主网络参数：不要和目标方向混为一谈

| #   | 论文                                                                                                                                        | 年份/发表                   | 更新类型 | 一句话总结                                                                                                       |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------- |
| R1  | [RMA: Rapid Motor Adaptation for Legged Robots](https://www.roboticsproceedings.org/rss17/p011.html)                                         | 2021, RSS                   | L        | 用历史观测估计环境隐变量并条件化固定策略，部署时主网络不做梯度更新，却奠定了机器人“实时适配”范式。             |
| R2  | [Deep Whole-Body Control: Learning a Unified Policy for Manipulation and Locomotion](https://proceedings.mlr.press/v205/fu23a.html)          | 2023, CoRL                  | L        | Regularized Online Adaptation 在训练期联合学习隐变量预测模块，部署时只前向推断环境 latent，并不在线优化权重。    |
| R3  | [Rapid Motor Adaptation for Robotic Manipulator Arms](https://doi.org/10.1109/CVPR52733.2024.01552)                                          | 2024, CVPR                  | L        | 把 RMA 式历史条件快速适配从腿足扩展到机械臂，重点仍是 latent inference 而不是推理时 SGD。                        |
| R4  | [RoboPack: Learning Tactile-Informed Dynamics Models for Dense Packing](https://www.roboticsproceedings.org/rss20/p130.html)                 | 2024, RSS                   | L/S      | 通过视觉触觉历史在线推断物体状态和潜在物理属性，再以 MPC 控制，适配主要发生在状态/latent 层。                    |
| R5  | [Offline Meta Reinforcement Learning with In-Distribution Online Adaptation](https://proceedings.mlr.press/v202/wang23au.html)               | 2023, ICML                  | L        | IDAQ 在测试任务上生成分布内上下文并推断 task belief，属于上下文适配而非直接改策略权重。                          |
| R6  | [ADPro: a Test-time Adaptive Diffusion Policy for Robot Manipulation](https://arxiv.org/abs/2508.06266)                                      | 2025, arXiv                 | I        | 通过流形约束和任务感知噪声初始化改变扩散策略的推理轨迹，明确无需再训练，因此是 test-time optimization 而非 TTT。 |
| R7  | [Retrieve-then-Steer: Online Success Memory for Test-Time Adaptation of Generative VLAs](https://arxiv.org/abs/2605.10094)                   | 2026, arXiv                 | M/I      | 冻结 VLA，把成功动作片段存入长期记忆并在生成时检索引导，论文明确强调 non-parametric adaptation。                 |
| R8  | [World Model Implanting for Test-time Adaptation of Embodied Agents](https://arxiv.org/abs/2509.03956)                                       | 2025, arXiv                 | M        | 测试时检索并组合独立训练的领域世界模型，以模块植入实现跨域适配，不修改基础策略参数。                             |
| R9  | [Learn from What We HAVE: History-Aware Verifier that Reasons about Past Interactions Online](https://proceedings.mlr.press/v305/li25e.html) | 2025, CoRL                  | M/I      | 用历史感知 verifier 从扩散策略候选动作中在线筛选，利用交互历史但不进行权重学习。                                 |
| R10 | [Inference-Time Enhancement of Generative Robot Policies via Predictive World Modeling](https://computationalrobotics.seas.harvard.edu/GPC/) | 2025/2026, project/preprint | I        | 用预测世界模型在推理时评估和优化生成策略候选，属于 planning-time policy improvement 而非参数更新。               |
| R11 | [Behavior Prompting Policy: Demonstrations as Prompts for Manipulation](https://arxiv.org/abs/2606.30457)                                    | 2026, arXiv                 | M/L      | 把一次人类示范当作行为 prompt，靠 in-context visuomotor conditioning 学新任务，无需微调。                        |
| R12 | [RoboMonkey: Scaling Test-Time Sampling and Verification for Vision-Language-Action Models](https://proceedings.mlr.press/v305/)             | 2025, CoRL                  | I        | 通过扩大测试时动作采样并用 verifier 选择来提高 VLA 成功率，增加的是推理算力而非学习参数。                        |

## 三、方法学基础：具身 TTA 论文经常引用

| #   | 论文                                                                                                                                                        | 年份/发表                     | 一句话总结                                                                                                   |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------ |
| F1  | [Test-Time Training with Self-Supervision for Generalization under Distribution Shifts](https://proceedings.mlr.press/v119/sun20b.html)                      | 2020, ICML                    | TTT 奠基作：把每个无标签测试样本变成自监督训练样本，预测前更新模型参数。                                     |
| F2  | [Tent: Fully Test-Time Adaptation by Entropy Minimization](https://openreview.net/forum?id=uXl3bZLkr3c)                                                      | 2021, ICLR                    | 只用测试数据最小化预测熵并更新归一化仿射参数，成为大量在线 TTA 方法的默认基线。                              |
| F3  | [TTT++: When Does Self-Supervised Test-Time Training Fail or Thrive?](https://openreview.net/forum?id=86NHK__yFDl)                                           | 2021, NeurIPS                 | 分析 TTT 在严重分布偏移下可能反而退化，并以特征对齐等机制提高稳定性。                                        |
| F4  | [Continual Test-Time Domain Adaptation](https://openaccess.thecvf.com/content/CVPR2022/html/Wang_Continual_Test-Time_Domain_Adaptation_CVPR_2022_paper.html) | 2022, CVPR                    | CoTTA 用权重/增强平均和随机恢复源权重抑制长期在线适配中的误差累积与遗忘。                                    |
| F5  | [Towards Stable Test-Time Adaptation in Dynamic Wild World](https://openreview.net/forum?id=g2YraF75Tj)                                                      | 2023, ICLR                    | SAR 针对小 batch、混合 shift 和类别不平衡提出可靠熵筛选与 sharpness-aware 更新，和真实机器人流数据非常相关。 |
| F6  | [Parameter-free Online Test-time Adaptation](https://doi.org/10.1109/CVPR52688.2022.00816)                                                                   | 2022, CVPR                    | 展示无需反向传播的在线 TTA 路线，提醒“test-time adaptation”并不等于“参数更新”。                          |
| F7  | [Test-Time Training on Video Streams](https://openreview.net/forum?id=orbnZE-4UvD)                                                                           | 2023, ICLR 投稿后撤稿         | 利用最近视频帧做 masked-autoencoder 自监督更新，直接讨论连续流上的 TTT 与有益遗忘。                          |
| F8  | [A Comprehensive Survey on Test-Time Adaptation under Distribution Shifts](https://arxiv.org/abs/2303.15361)                                                 | 2023, arXiv                   | 按 test-time domain/batch/online adaptation 系统梳理 TTA，适合建立术语与方法 taxonomy。                      |
| F9  | [Continual Learning for Robotics: Definition, Framework, Learning Strategies, Opportunities and Challenges](https://arxiv.org/abs/1907.00182)                | 2019/2020, Information Fusion | 机器人持续学习经典综述，强调数据流、目标变化、知识积累、算力约束和真实具身评测。                             |
| F10 | [A Survey of Continual Learning for Robotics in the Foundation Model Era](https://doi.org/10.36227/techrxiv.176972367.76460794/v2)                           | 2026, TechRxiv                | 从 foundation model 时代重新梳理机器人 CL 的任务、方法、基准和遗忘问题，是连接 VLA 与持续更新的最新入口。    |

## 四、研究脉络与空白

### 主要趋势

1. **2020–2022：视觉 TTT/TTA 方法学成形。** 目标是无源数据、无标签的分布偏移，主流损失是自监督和熵最小化。
2. **2021–2023：机器人快速适配主要走 latent/context inference。** RMA 等方法追求毫秒级稳健控制，但通常冻结网络，因此不是严格的推理时参数学习。
3. **2024–2025：具身闭环 TTA 出现。** VLN、户外视觉搜索、抓取和可通行性估计开始直接利用交互数据与反馈更新模型。
4. **2025–2026：VLA、fast weights 与持续记忆汇合。** VITA、RoboTTT、FeedTTA 开始把“历史写入参数”或反馈驱动策略更新做成模型核心机制；同时 non-parametric memory 和 test-time sampling 成为竞争路线。
5. **评测正在从静态 dataset shift 转向持续部署。** 新问题包括更新延迟、边缘算力、安全探索、失败反馈稀疏、遗忘、参数污染和能否回滚。

### 值得做的研究问题

- **安全的 embodied TTT**：机器人失败会有物理代价，不能照搬图像分类上的“每批熵最小化”。
- **VLA 的参数高效在线更新**：只更新 LoRA/adapters/fast weights，并限制延迟、显存与遗忘。
- **交互反馈如何构造自监督损失**：成功/失败、接触、可达性、视觉进度、语言纠错、动力学预测误差都可成为信号。
- **何时更新、何时不更新**：检测 domain shift 和置信度，避免错误样本把策略带崩。
- **参数记忆 vs 外部记忆**：RoboTTT/VITA 的 weight-space memory 与 Retrieve-then-Steer/HAVE 的显式 memory 应在统一预算下比较。
- **跨 embodiment 的持续适配**：变化的不只是场景和任务，还有相机、机械臂、动力学与 action space。
- **统一 benchmark**：应报告在线 regret/样本效率、灾难性遗忘、恢复速度、计算/能耗、真实机器人事故率，而不只是最终 success rate。

## 五、检索统计与技能要求的摘要

### Overview

运行了 6 组互补查询，时间范围为 2018–2026，每个 API 来源每组最多返回 10 篇；脚本共返回 **120 条原始 source hits**（包含重复和明显误检），再结合论文/会议官方页面核验并整理出上面 **42 篇核心或强相关论文**。查询为：

1. `embodied AI robot test-time adaptation parameter update`
2. `robot learning online adaptation policy test time`
3. `rapid motor adaptation robot locomotion`
4. `test-time adaptation vision language navigation embodied`
5. `continual lifelong robot learning embodied agent online adaptation`
6. `test-time training vision language action robot policy`

### Trends

检索结果在 2024–2026 明显增多；ICML/ICLR/CoRL/RSS/RA-L 是最相关的高信号来源。方法从静态视觉域适配，发展到 VLN 的参数流更新、交互反馈 RL、机器人自监督在线学习，再到 VLA fast weights 与长期部署记忆。

### Key themes

- **在线 VLN 的梯度适配**：P1、P2。
- **VLM/VLA 的 fast weights 与轻量模块**：P3、P4、P12。
- **真实机器人自监督在线学习**：P6、P11、P20。
- **动力学/控制模型持续适配**：P13、P16、P17、P19。
- **长期技能积累与抗遗忘**：P10、P18。
- **非参数推理时适配**：R6–R12。

### Keywords frequency

对 42 篇核心论文的标题与摘要做词干级、不区分大小写的概念计数：

| Keyword                       | Count |
| ----------------------------- | ----: |
| robot / robotic / robotics    |    36 |
| adapt / adaptation / adaptive |    31 |
| learn / learning              |    28 |
| test-time                     |    19 |
| online / continual / lifelong |    18 |

### Most cited by accepted paper

以下计数取本次 API 返回值中的较高记录；不同索引更新时间不一致，只适合粗排，不能视为精确实时引用数。

| Rank | Title                                                                     | Year | Citations |
| ---: | ------------------------------------------------------------------------- | ---: | --------: |
|    1 | RMA: Rapid Motor Adaptation for Legged Robots                             | 2021 |       445 |
|    2 | Multi-expert Learning of Adaptive Legged Locomotion                       | 2020 |       194 |
|    3 | Efficient Test-Time Adaptation of Vision-Language Models                  | 2024 |       165 |
|    4 | Parameter-free Online Test-time Adaptation                                | 2022 |       129 |
|    5 | Real-world Embodied AI through a Morphologically Adaptive Quadruped Robot | 2021 |       114 |

### Most cited by first author

按标题去重后，以本次结果集中的最高 citation count 统计：

| Rank | Author           | Papers in set | Total citations |
| ---: | ---------------- | ------------: | --------------: |
|    1 | Ashish Kumar     |             1 |             445 |
|    2 | Chuanyu Yang     |             1 |             194 |
|    3 | Adilbek Karmanov |             1 |             165 |
|    4 | Malik Boudiaf    |             1 |             129 |
|    5 | Tønnes Nygaard  |             1 |             114 |

### Recommendations for reading

1. **TTT (ICML 2020)**：先理解“测试样本本身就是训练信号”的原始范式。
2. **RMA (RSS 2021)**：理解机器人领域为什么大量工作选择 latent adaptation 而非在线反向传播。
3. **FSTTA (ICML 2024)**：看具身多步决策如何改变 TTA 的梯度与参数更新设计。
4. **FeedTTA (ICML 2025)**：看环境反馈如何转化为在线策略更新。
5. **RoboTTT + VITA (2026)**：看 fast weights、元学习自监督损失和机器人基础模型的最新融合。

## 六、所有 API 检索结果（未按相关性删减）

说明：以下严格保留技能脚本返回的全部 120 条 source hits；其中包含重复项、无年份项及明显偏离具身主题的误检。核心正文已做去重和相关性筛选。

### Semantic Scholar（30 条）

|  # | Title                                                                                                                                                                                     |  Date | Venue                       | Citations |
| -: | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----: | --------------------------- | --------: |
|  1 | [VITA: Zero-Shot Value Functions via Test-Time Adaptation of Vision-Language Models](https://www.semanticscholar.org/paper/49c193405f73ba371384c8c62eef2cf7f8e1ccf7)                       |  2025 | —                          |         1 |
|  2 | [UL-VIO: Ultra-lightweight Visual-Inertial Odometry with Noise Robust Test-time Adaptation](https://www.semanticscholar.org/paper/4870d2f769653ec11873060d1d5cd063d70fc78b)                |  2024 | ECCV                        |         8 |
|  3 | [Retrieve-then-Steer: Online Success Memory for Test-Time Adaptation of Generative VLAs](https://www.semanticscholar.org/paper/996bf7b2754f10be9f0bb4470c42f20461318c85)                   |  2026 | arXiv                       |         1 |
|  4 | [Get a GRIP on Test Time Adaptation!](https://www.semanticscholar.org/paper/e1e518ab46264555f9434b79a66b70b0d6652d8b)                                                                      |  2025 | CVPRW                       |         0 |
|  5 | [Fast-Slow Test-Time Adaptation for Online Vision-and-Language Navigation](https://www.semanticscholar.org/paper/a6946bdaf38f3b6e34d38a1a57483978777b3e66)                                 | 2023* | ICML                        |        29 |
|  6 | [Real-world Embodied AI through a Morphologically Adaptive Quadruped Robot](https://www.semanticscholar.org/paper/bd5cd9d680bd4bea13c7d1e0433e862f3bfb8411)                                |  2021 | Nature Machine Intelligence |       114 |
|  7 | [Test-Time Adaptation for Point Cloud Upsampling Using Meta-Learning](https://www.semanticscholar.org/paper/dd7a5e767dae0b9dbd402b693e0b606048dfcf06)                                      |  2023 | IROS                        |        12 |
|  8 | [Real-world Embodied AI: A Morphologically Adaptive Quadruped Robot in the Wild](https://www.semanticscholar.org/paper/e075e589b1a60a0af65fb6f17fe7272ade6079a2)                           |  2021 | —                          |         0 |
|  9 | [ISCS: Parameter-Guided Feature Pruning for Resource-Constrained Embodied Perception](https://www.semanticscholar.org/paper/9bb15c6188a86e1d8c04f6aef56ac71e8ce9b52d)                      |  2025 | —                          |         2 |
| 10 | [PhysMem: Scaling Test-Time Memory for Embodied Physical Reasoning](https://www.semanticscholar.org/paper/1c3a27b73367d84fef9cdf045645c732c049cdf6)                                        |  2026 | —                          |         0 |
| 11 | [Fast-Slow Test-Time Adaptation for Online Vision-and-Language Navigation](https://www.semanticscholar.org/paper/a6946bdaf38f3b6e34d38a1a57483978777b3e66)                                 | 2023* | ICML                        |        29 |
| 12 | [Test-Time Adaptation for Online Vision-Language Navigation with Feedback-based Reinforcement Learning](https://www.semanticscholar.org/paper/94455bf7dead863cd4c9281b6b2ac97102012546)    |  2025 | ICML                        |        11 |
| 13 | [Bayesian Test-Time Adaptation for Vision-Language Models](https://www.semanticscholar.org/paper/843603e514f51d714e1498538460525d55577cf7)                                                 |  2025 | CVPR                        |        24 |
| 14 | [Realistic Test-Time Adaptation of Vision-Language Models](https://www.semanticscholar.org/paper/d9a639523324e0cce9376cee7e190e503cecbe1c)                                                 |  2025 | CVPR                        |        15 |
| 15 | [Efficient Test-Time Adaptation of Vision-Language Models](https://www.semanticscholar.org/paper/390b2aa03672d33c760a151bb056b1397bf40cc0)                                                 |  2024 | CVPR                        |       165 |
| 16 | [Embodied4C: Measuring What Matters for Embodied Vision-Language Navigation](https://www.semanticscholar.org/paper/7fd7ec886aeb87c2f3031351023cefe3df9b34c5)                               |  2025 | arXiv                       |         3 |
| 17 | [Aux-Think: Exploring Reasoning Strategies for Data-Efficient Vision-Language Navigation](https://www.semanticscholar.org/paper/6681c165ac6626408c48ee3a4c868a760db2f5e2)                  |  2025 | arXiv                       |        19 |
| 18 | [Latte: Collaborative Test-Time Adaptation of Vision-Language Models in Federated Learning](https://www.semanticscholar.org/paper/44ff20337f1cfed26664db0048ea104dcfcece7b)                |  2025 | ICCV                        |        12 |
| 19 | [Noisy Test-Time Adaptation in Vision-Language Models](https://www.semanticscholar.org/paper/e101b28580679f6c9e5a29327c9f68b2c2cddfdc)                                                     |  2025 | ICLR                        |        10 |
| 20 | [FlexVLN: Flexible Adaptation for Diverse Vision-and-Language Navigation Tasks](https://www.semanticscholar.org/paper/1d72f5f6695bb270c3c12d06938228f53e24704f)                            |  2025 | IEEE TMM                    |        18 |
| 21 | [Lifelong Robot Library Learning](https://www.semanticscholar.org/paper/931d1a4e3f28829225cc1bc83897f97bc38a156b)                                                                          |  2024 | ICRA                        |        23 |
| 22 | [Dynamic Mixture of Progressive Parameter-Efficient Expert Library for Lifelong Robot Learning](https://www.semanticscholar.org/paper/a3276f846034a2ed8003a99e5f21516cb239e463)            |  2025 | TMLR                        |         4 |
| 23 | [Efficient Continual Adaptation of Pretrained Robotic Policy with Online Meta-Learned Adapters](https://www.semanticscholar.org/paper/98c4e11770e07adad1447193526a5d82776a139e)            |  2025 | arXiv                       |         7 |
| 24 | [Continual Harness: Online Adaptation for Self-Improving Foundation Agents](https://www.semanticscholar.org/paper/6a9320f6e52a64a977fbb253481fb2f53fe8c24a)                                |  2026 | arXiv                       |         4 |
| 25 | [Online Continual Learning for Interactive Instruction Following Agents](https://www.semanticscholar.org/paper/1802572390d9b3b9ca23bcee01ab12d273916d7b)                                   |  2024 | ICLR                        |        22 |
| 26 | [Arcadia: Toward a Full-Lifecycle Framework for Embodied Lifelong Learning](https://www.semanticscholar.org/paper/21e1fef0e6ba79a4cb0bf76daa80c3faa651a165)                                |  2025 | arXiv                       |         1 |
| 27 | [Multi-Agent Continual Deep Reinforcement Learning with Artificial Potential Fields and Switching Control](https://www.semanticscholar.org/paper/36510abaf6d90c52989964d5c417827f1ed39109) |  2026 | JIRS                        |         1 |
| 28 | [Preserving and Combining Knowledge in Robotic Lifelong Reinforcement Learning](https://www.semanticscholar.org/paper/4f55bbd7e6f9ca2013b471705cc7c655ee9a56c6)                            |  2025 | Nature Machine Intelligence |        80 |
| 29 | [Efficient Continual Imitation Learning with Online Meta-Adapters](https://www.semanticscholar.org/paper/9dc99999cf1eb4288d612b5c5eacd004ae8ccd68)                                         |  2026 | RA-L                        |         1 |
| 30 | [Self-adapting Robotic Agents through Online Continual Reinforcement Learning with World Model Feedback](https://www.semanticscholar.org/paper/72d13098873da103d0870ef9fe924bec5defdbb6)   |  2026 | arXiv                       |         0 |

\* Semantic Scholar 的年份字段与正式会议年份不一致；FSTTA 正式发表为 ICML 2024。

### OpenAlex（20 条）

|  # | Title                                                                                                                                             | Date | Venue                              | Citations |
| -: | ------------------------------------------------------------------------------------------------------------------------------------------------- | ---: | ---------------------------------- | --------: |
|  1 | [Multi-expert Learning of Adaptive Legged Locomotion](https://doi.org/10.1126/scirobotics.abb2174)                                                 | 2020 | Science Robotics                   |       194 |
|  2 | [Exploration-based Learning of a Stabilizing Controller Predicts Locomotor Adaptation](https://doi.org/10.1038/s41467-024-53416-w)                 | 2024 | Nature Communications              |        36 |
|  3 | [Locomotion Control with Frequency and Motor Pattern Adaptations](https://doi.org/10.3389/fncir.2021.743888)                                       | 2021 | Frontiers in Neural Circuits       |        13 |
|  4 | [Meta-Learning for Fast Adaptive Locomotion with Uncertainties in Environments and Robot Dynamics](https://doi.org/10.1109/IROS51168.2021.9635840) | 2021 | IROS                               |        17 |
|  5 | [RMA: Rapid Motor Adaptation for Legged Robots](https://doi.org/10.48550/arXiv.2107.04034)                                                         | 2021 | arXiv                              |        16 |
|  6 | [Exploration-based Learning of a Stabilizing Controller Predicts Locomotor Adaptation](https://doi.org/10.1101/2021.03.18.435986)                  | 2021 | bioRxiv                            |        12 |
|  7 | [Learning Gait-conditioned Bipedal Locomotion with Motor Adaptation](https://doi.org/10.1109/Humanoids57100.2023.10375167)                         | 2023 | Humanoids                          |        14 |
|  8 | [Fast and Slow Adaptations of Interlimb Coordination](https://doi.org/10.3389/frobt.2021.697612)                                                   | 2021 | Frontiers in Robotics and AI       |         9 |
|  9 | [Rapid Motor Adaptation for Robotic Manipulator Arms](https://doi.org/10.1109/CVPR52733.2024.01552)                                                | 2024 | CVPR                               |         6 |
| 10 | [Locomotor Adaptations: Paradigms, Principles and Perspectives](https://doi.org/10.1088/2516-1091/ac91b6)                                          | 2022 | Progress in Biomedical Engineering |        12 |
| 11 | [Policy and Value Transfer in Lifelong Reinforcement Learning](https://openalex.org/W2887671224)                                                   | 2018 | ICML                               |        39 |
| 12 | [Preserving and Combining Knowledge in Robotic Lifelong Reinforcement Learning](https://doi.org/10.1038/s42256-025-00983-2)                        | 2025 | Nature Machine Intelligence        |        34 |
| 13 | [Lifelong Robot Learning](https://doi.org/10.1007/978-3-642-41610-1_203-1)                                                                         | 2021 | Encyclopedia chapter               |        10 |
| 14 | [Adaptive Robot Traversability Estimation Based on Self-Supervised Online Continual Learning](https://doi.org/10.1109/LRA.2024.3386451)            | 2024 | RA-L                               |        13 |
| 15 | [Grow Your Limits](https://doi.org/10.1109/ICRA57147.2024.10610485)                                                                                | 2024 | ICRA                               |        12 |
| 16 | [Preserving and Combining Knowledge in Robotic Lifelong Reinforcement Learning](https://doi.org/10.21203/rs.3.rs-4353532/v1)                       | 2024 | Research Square                    |         2 |
| 17 | [Evaluations of the Gap between Supervised and Reinforcement Lifelong Learning on Robotic Manipulation Tasks](https://openalex.org/W3215163890)    | 2021 | CoRL                               |         1 |
| 18 | [Online Continual Learning for Interactive Instruction Following Agents](https://doi.org/10.48550/arXiv.2403.07548)                                | 2024 | arXiv                              |         0 |
| 19 | [Building Embodied AI Systems](https://doi.org/10.1007/978-3-031-68256-8)                                                                          | 2024 | Book                               |         6 |
| 20 | [A Survey of Continual Learning for Robotics in the Foundation Model Era](https://doi.org/10.36227/techrxiv.176972367.76460794/v1)                 | 2026 | TechRxiv                           |         0 |

### arXiv（10 条）

|  # | Title                                                                                                                                             | Date | Venue | Citations |
| -: | ------------------------------------------------------------------------------------------------------------------------------------------------- | ---: | ----- | --------: |
|  1 | [VLP: A Survey on Vision-Language Pre-training](http://arxiv.org/abs/2202.09061v4)                                                                 | 2022 | arXiv |         0 |
|  2 | [VLS: Steering Pretrained Robot Policies via Vision-Language Models](http://arxiv.org/abs/2602.03973v1)                                            | 2026 | arXiv |         0 |
|  3 | [Vision-Language Pre-training: Basics, Recent Advances, and Future Trends](http://arxiv.org/abs/2210.09263v1)                                      | 2022 | arXiv |         0 |
|  4 | [End-to-End Test-Time Training for Long Context](http://arxiv.org/abs/2512.23675v2)                                                                | 2025 | arXiv |         0 |
|  5 | [Your Vision-Language-Action Model Already Has Attention Heads for Path Deviation Detection](http://arxiv.org/abs/2603.13782v1)                    | 2026 | arXiv |         0 |
|  6 | [Imitation Learning of Robot Policies by Combining Language, Vision and Demonstration](http://arxiv.org/abs/1911.11744v1)                          | 2019 | arXiv |         0 |
|  7 | [Compositional Context Fine-Tuning Vision-Language Model for Complex Assembly Action Understanding from Videos](http://arxiv.org/abs/2607.10797v1) | 2026 | arXiv |         0 |
|  8 | [Revisiting Realistic Test-Time Training](http://arxiv.org/abs/2303.10856v1)                                                                       | 2023 | arXiv |         0 |
|  9 | [Exploring Large Language Models to Facilitate Variable Autonomy for Human-Robot Teaming](http://arxiv.org/abs/2312.07214v3)                       | 2023 | arXiv |         0 |
| 10 | [EasyARC: Evaluating Vision Language Models on True Visual Reasoning](http://arxiv.org/abs/2506.11595v1)                                           | 2025 | arXiv |         0 |

### OpenReview（0 条）

脚本的 6 组查询均返回 0；随后通过官方网页检索补到 ICLR 2024 的 Online Continual Learning、ICLR 2026 的 VITA 等论文，见正文。

### Crossref（60 条）

|  # | Title                                                                                 | Date | Venue                            | Citations |
| -: | ------------------------------------------------------------------------------------- | ---: | -------------------------------- | --------: |
|  1 | From Real-time Adaptation to Social Learning in Robot Ecosystems                      | 2023 | Frontiers in Robotics and AI     |         2 |
|  2 | Emergent Test-Time Adaptation via the Genesis-Integration Principle                   |   — | TechRxiv                         |         0 |
|  3 | Mismatch Minimization in AI and Cognition                                             |   — | TechRxiv                         |         0 |
|  4 | Bio-inspired Cognitive Robotics vs. Embodied AI for Socially Acceptable Robots        | 2026 | Frontiers in Robotics and AI     |         0 |
|  5 | Parameter-free Online Test-time Adaptation                                            | 2022 | CVPR                             |       129 |
|  6 | Embodied AI in Mobile Robot Simulation with EyeSim                                    | 2025 | SIMULTECH                        |         6 |
|  7 | Embodied Intelligence in Soft Robotics Through Hardware Multifunctionality            | 2021 | Frontiers in Robotics and AI     |        23 |
|  8 | Gödelian Embodied Self-referential Genomic Intelligence                              | 2025 | Frontiers in Robotics and AI     |         0 |
|  9 | Embodied AI Agent for Co-creation Ecosystem (OSF 54de2)                               |   — | OSF                              |         0 |
| 10 | Embodied AI Agent for Co-creation Ecosystem (OSF krxp6)                               |   — | OSF                              |         0 |
| 11 | Robot Learning and Adaptation for Intelligent Behavior                                | 2023 | TOJQI                            |         0 |
| 12 | Robust Online Test-Time Adaptation via a Multilayer Generative-Integrative Framework  |   — | TechRxiv                         |         0 |
| 13 | Online Meta-Learning for Real-Time Adaptation in Dynamical System Modeling            |   — | SSRN                             |         0 |
| 14 | Learning Less Generalizable Patterns for Better Test-Time Adaptation                  | 2023 | VISAPP                           |         0 |
| 15 | When Not to Adapt: Conservative Slack-Aware Online TTA                                |   — | SSRN                             |         0 |
| 16 | Imagined Rehearsal as Test-Time Adaptation and Policy Repair for World-Model Agents   |   — | SSRN                             |         0 |
| 17 | From Real-time Adaptation to Social Learning in Robot Ecosystems                      | 2023 | Frontiers in Robotics and AI     |         2 |
| 18 | Online Policy Adaptation for Personalized Lane-Keeping                                | 2026 | RA-L                             |         0 |
| 19 | Test-Time Adaptation for Graph Learning: A Systematic Survey                          |   — | TechRxiv                         |         0 |
| 20 | Cloud-Edge Test-Time Adaptation for Cross-Domain Online Machinery Fault Diagnosis     | 2024 | Advanced Engineering Informatics |        19 |
| 21 | Learning Modular Robot Visual-motor Locomotion Policies                               | 2023 | ICRA                             |         1 |
| 22 | A Wearable System for Experimental Knee Pain during Real-world Locomotion             |   — | TechRxiv                         |         0 |
| 23 | Rapid Motor Adaptation via Population-level Modulation of Cerebellar Error Signals    |   — | bioRxiv                          |         2 |
| 24 | Robotic Locomotion through Active and Passive Morphological Adaptation                | 2025 | Science Robotics                 |        35 |
| 25 | SCRMA: Snake-Like Robot Curriculum Rapid Motor Adaptation                             | 2022 | ICIRA                            |         0 |
| 26 | RL2AC: Reinforcement Learning-based Rapid Online Adaptive Control                     | 2024 | RSS                              |        14 |
| 27 | RMA: Rapid Motor Adaptation for Legged Robots                                         | 2021 | RSS                              |       445 |
| 28 | Locomotion Adaptation in Heavy Payload Transportation Tasks with CENTAURO             | 2021 | ICRA                             |        14 |
| 29 | STANCE: Locomotion Adaptation over Soft Terrain                                       | 2020 | IEEE T-RO                        |        63 |
| 30 | Reinforcement Learning with Model-Based Step Adaptation for Robust Bipedal Locomotion | 2026 | IJHR                             |         0 |
| 31 | Uniformity First: Uniformity-aware TTA of Vision-language Models                      |   — | SSRN                             |         0 |
| 32 | Reliable and Global Gaussian Alignment for TTA of Vision-Language Models              |   — | SSRN                             |         0 |
| 33 | EMKG-VLN: Embodied Multimodal Knowledge Graph VLM for Open-World Navigation           |   — | SSRN                             |         0 |
| 34 | Realistic Test-Time Adaptation of Vision-Language Models                              | 2025 | CVPR                             |         3 |
| 35 | Efficient Test-Time Adaptation of Vision-Language Models                              | 2024 | CVPR                             |        65 |
| 36 | SwapPrompt: Test-Time Prompt Adaptation for Vision-Language Models                    | 2023 | NeurIPS                          |         1 |
| 37 | Online Gaussian Test-Time Adaptation of Vision-Language Models                        | 2025 | CVPRW                            |         4 |
| 38 | Frustratingly Easy Test-Time Adaptation of Vision-Language Models                     | 2024 | NeurIPS                          |         8 |
| 39 | Latte: Collaborative Test-Time Adaptation of VLMs in Federated Learning               | 2025 | ICCV                             |         1 |
| 40 | Test-Time Low Rank Adaptation via Confidence Maximization                             | 2025 | WACV                             |         6 |
| 41 | FEDL: A Federated Continual Learning Framework (SSRN 5638746)                         |   — | SSRN                             |         0 |
| 42 | FEDL: A Federated Continual Learning Framework (SSRN 6177802)                         |   — | SSRN                             |         0 |
| 43 | Learning on the Job: Online Lifelong and Continual Learning                           | 2020 | AAAI                             |        30 |
| 44 | Enabling Lifelong Learning in AI with Continual Learning Methods                      |   — | SSRN                             |         0 |
| 45 | Lifelong Robot Library Learning                                                       | 2024 | ICRA                             |         8 |
| 46 | Continual Learning of Conversational Skills                                           | 2024 | Book chapter                     |         0 |
| 47 | Open-World Continual Learning: A Framework                                            | 2024 | Book chapter                     |         2 |
| 48 | Continual Learning in Chit-Chat Systems                                               | 2024 | Book chapter                     |         0 |
| 49 | Robot Learning and Adaptation for Intelligent Behavior                                | 2023 | TOJQI                            |         0 |
| 50 | AdaER: An Adaptive Experience Replay Approach for Continual Lifelong Learning         |   — | SSRN                             |         1 |
| 51 | Vision-Language-Action and Vision Language Models for Robot Manipulation: A Review    |   — | Preprints.org                    |         0 |
| 52 | Test-Time Prompt Tuning for Vision-Language Models                                    | 2026 | Book chapter                     |         0 |
| 53 | Token Expand-Merge: Training-Free Token Compression for VLA Models                    | 2026 | RA-L                             |         0 |
| 54 | GenACT: Towards Unified Vision-Language-Action Co-training                            | 2026 | PRCV                             |         0 |
| 55 | Graph-Fused VLA Models for Semantically Safe Dual-Robot Control                       |   — | engrXiv                          |         0 |
| 56 | AU-TTT: Vision Test-Time Training for Facial Action Unit Detection                    | 2025 | ICME                             |         4 |
| 57 | ActionX: Pre-training Action Experts with RL for VLA Models                           | 2026 | Frontiers in Neurorobotics       |         0 |
| 58 | Introduction                                                                          | 2024 | Language Policy in Action        |         0 |
| 59 | Generating Robot Action Sequences with Visual Prompts                                 | 2024 | IWIS                             |         2 |
| 60 | Improved Self-Training for Test-Time Adaptation                                       | 2024 | CVPR                             |        17 |

### DBLP（0 条）

5 组查询返回 0；第 1 组发生 HTTP 500，错误见下一节。

### Model Knowledge（5 条，均已用官方/论文页面核验）

这些论文未在脚本 API 的对应查询中稳定返回，但对理解方向十分关键；不提供记忆式 citation count。

| # | Title                                                                                                                                                       | Year | Venue | Notes                                    |
| -: | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ---: | ----- | ---------------------------------------- |
| 1 | [Test-Time Training with Self-Supervision](https://proceedings.mlr.press/v119/sun20b.html)                                                                   | 2020 | ICML  | TTT 奠基工作。                           |
| 2 | [Tent](https://openreview.net/forum?id=uXl3bZLkr3c)                                                                                                          | 2021 | ICLR  | Fully TTA 的经典熵最小化基线。           |
| 3 | [Continual Test-Time Domain Adaptation](https://openaccess.thecvf.com/content/CVPR2022/html/Wang_Continual_Test-Time_Domain_Adaptation_CVPR_2022_paper.html) | 2022 | CVPR  | CTTA 奠基工作之一。                      |
| 4 | [Deep Whole-Body Control](https://proceedings.mlr.press/v205/fu23a.html)                                                                                     | 2023 | CoRL  | 机器人 latent online adaptation 的代表。 |
| 5 | [Online Model Adaptation with Feedforward Compensation](https://proceedings.mlr.press/v229/abuduweili23a.html)                                               | 2023 | CoRL  | 实时更新动力学/预测模型并抑制遗忘。      |

## 七、检索错误与局限（原样保留）

初次聚合式调用因个别无超时 API 长时间阻塞，被中断；之后按来源设置 70 秒进程超时运行，未修改技能脚本。主要错误如下：

```text
[open_alex] Error: 504 Server Error: Gateway Timeout for url: https://api.openalex.org/works?search.semantic=embodied+AI+robot+test-time+adaptation+parameter+update&filter=publication_year%3A2018-2026&sort=relevance_score%3Adesc&page=1&per-page=10

[dblp] Error: 500 Server Error: Internal Server Error for url: https://dblp.org/search/publ/api?q=embodied+AI+robot+test-time+adaptation+parameter+update&format=json&h=10&f=0

[arxiv] Error: The read operation timed out

Rate limited. Waiting 3 seconds...
```

同类 OpenAlex 504 出现在查询 2、4、6；arXiv read timeout 出现在查询 1–5；Semantic Scholar 在查询 2、3、6 多次 rate-limit 后由外层 `timeout` 以 exit code 124 终止。OpenReview API 六组查询均返回 0，而网页核验能找到相关论文，说明该脚本的 OpenReview 关键词召回不足。

局限：

- citation count 来自不同 API，刷新时间和合并规则不同；同一论文可能出现 65/165、34/80 等不同值。
- 2026 年论文很多仍是预印本，接受状态和标题可能继续变化。
- Crossref 的语义检索精度较低，完整表中保留了明显无关项，以满足全量结果可审计性。
- “参数更新”有灰区：BN running statistics、GP/Kalman 后验、fast weights 是否算“模型参数”取决于论文定义；正文已显式标注。

## 建议继续检索的关键词串

```text
"test-time training" robot policy
"test-time adaptation" embodied agent
"online test-time adaptation" vision-language navigation
"continual test-time adaptation" robotics
"online policy adaptation" robot manipulation
"continual robot learning" vision-language-action
"fast weights" robot policy
"online meta-learning" robot dynamics
"self-supervised online learning" robot deployment
"lifelong robot learning" catastrophic forgetting
```
