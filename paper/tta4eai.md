# Test-Time Learning/Adaptation for Embodied Agents

## 概览

* RSS / 检索关键词：Test-Time Adaptation (TTA，测试时自适应)、Test-Time Training (TTT，测试时训练)、Online Test-Time Adaptation / Continual Test-Time Adaptation (OTTA/CTTA)、Continual/Lifelong Robot Learning、online adaptation、meta-RL、adaptive control、online system identification、online/continual robot learning。

* 当前方向暂定为 **Test-Time Learning/Adaptation for Embodied Agents（具身智能体的测试时学习与自适应）**，并将 **online/continual robot learning** 视为相邻的大方向。检索时不要只搜索 online learning，否则容易被传统监督式在线学习文献淹没。

核心问题可以统一表述为：

> **Embodied Agent 在 deployment/test time 如何利用在线产生的信息，改变自身内部状态，从而改善之后的感知、评价、决策和行为？**

---

## 阅读每篇论文时固定回答的问题

1. **Learning signal 从哪里来？**
   是无标签 observation、自监督目标、环境反馈、reward、success/failure signal，还是模型自身预测误差？

2. **What is adapted？更新什么？**
   Representation、Policy、Value Function、Reward Model、World Model、Adapter、LoRA、Prompt、Fast Weights，还是显式 Memory？

3. **Adaptation timescale 是什么？**
   每个 observation、每个 action step、每个 trajectory、episode 之间，还是跨任务持续更新？

4. **经验如何跨时间积累？**
   是保存在 context、replay buffer、gradient statistics、显式 memory，还是直接编码进不断变化的参数？

5. **如何控制 adaptation risk？**
   怎样避免错误反馈、自强化错误、catastrophic forgetting、parameter drift，以及 plasticity-stability 冲突？

6. **Adaptation 最终改善的是哪一层能力？**
   感知、状态表征、价值判断、规划、策略执行、环境建模，还是长期记忆？

建议给每篇论文额外写一句：

> **Adaptation changes ___ using ___ at timescale ___, and stores experience in ___.**

如果这句话还无法明确填写，通常意味着对论文核心机制还没有压缩完成。

---

# TTA 方法分类

当前可以先按照 **deployment 时究竟什么东西发生自适应** 来分类。

## 1. Policy-level Adaptation

直接更新负责 action prediction / navigation policy 的模型参数。

代表：

* FSTTA
* FeedTTA

核心目标：

$$
\text{Observation / Feedback}
\rightarrow
\Delta \theta_{\text{policy}}
\rightarrow
\text{Better Action}
$$

---

## 2. Evaluation-level Adaptation

不一定直接修改 policy，而是首先调整智能体对当前状态、目标完成程度或未来收益的判断。

代表：

* VITA：Value Function / Value Estimation TTA

核心形式：

$$
\text{Online Experience}
\rightarrow
\Delta \theta_{\text{value}}
\rightarrow
\text{Better State Evaluation}
\rightarrow
\text{Better Planning / Behavior}
$$

注意：**VITA 更准确地属于 Value / Evaluation TTA，而不是严格意义上的 World Model TTA。**

典型 World Model 应进一步建模：

$$
p(s_{t+1}\mid s_t,a_t)
$$

或者 latent dynamics，而 VITA 主要解决的是 test-time value estimation 与 temporal reasoning。

---

## 3. Memory / Fast-Weight Adaptation

将参数更新本身视为一种长期或快速记忆机制：

$$
\theta_t = f(x_1,x_2,\dots,x_{t-1})
$$

此时参数不再只是固定模型参数，而可以被理解为 trajectory-dependent hidden state。

代表方向：

* RoboTTT
* Fast weights / test-time memory 类方法

这是后续重点关注方向。

---

# 最值得优先读的 6 篇

1. **FSTTA (ICML 2024)**
   最清楚地定义在线 VLN 中同时存在的 intra-sample 与 inter-sample adaptation 问题，以及高频更新和稳定性之间的矛盾。

2. **FeedTTA (ICML 2025)**
   将 TTA 从依赖无监督 surrogate loss 推进到利用真实 embodied interaction feedback，通过 episodic binary feedback 产生 adaptation signal。

3. **VITA (ICLR 2026)**
   轻量 adaptation module + meta-learned self-supervised loss，通过 trajectory 中连续 test-time updates 改善 VLM value estimation，并隐式编码 temporal history。

4. **RoboTTT (2026)**
   将 fast weights / parameter updates 更直接地作为机器人策略的 test-time memory，代表 VLA 与 TTT 融合的重要趋势。

5. **Search-TTA (CoRL 2025)**
   面向真实户外搜索任务，将在线梯度反馈与 embodied closed-loop interaction 结合，并强调 planner-agnostic adaptation。

6. **Online Continual Learning for Interactive Instruction Following (ICLR 2024)**
   如果研究兴趣进一步从单 episode / 单 environment adaptation 扩展到跨任务、长期持续学习，它是 TTA 与 continual embodied learning 之间的重要概念桥梁。

---

# 论文及其问题的答案

## VITA (ICLR 2026)

VITA 是一种基于 VLM 的、可测试时自适应的 goal-conditioned value function 方法，用来评估当前视觉状态相对于语言任务目标的未来任务价值。

1. **Learning signal 从哪里来？**
2. **What is adapted？更新什么？**
    - Training: $f_{adapt}$ + value head + SSL learning rule
    - Test Time: $f_{adapt}$
3. **Adaptation timescale 是什么？**
4. **经验如何跨时间积累？**
5. **如何控制 adaptation risk？**
6. **Adaptation 最终改善的是哪一层能力？**
