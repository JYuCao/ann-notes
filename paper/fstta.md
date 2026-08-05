# Fast-Slow Test-Time Adaptation for Online Vision-and-Language Navigation

> Junyu Gao, Xuan Yao, Changsheng Xu. ICML 2024.
>
> 论文：[PMLR](https://proceedings.mlr.press/v235/gao24p.html) ｜ [arXiv](https://arxiv.org/abs/2311.13209) ｜ [官方代码](https://github.com/Feliciaxyao/ICML2024-FSTTA)

**关键词：** Online VLN、Test-Time Adaptation、Fast-Slow Update、Stability-Plasticity、Continual Adaptation

## 1. 一句话总结

FSTTA 用两个时间尺度处理在线 VLN 的稳定性—可塑性矛盾：在一个 episode 内，每隔若干动作聚合近期梯度，快速适应当前环境；跨多个 episodes，则分析 Fast 模型的参数轨迹，将反复出现的变化方向写入较稳定的 Slow 模型。

其核心可以概括为：

$$
\boxed{\text{Fast：梯度一致性} + \text{Slow：参数轨迹巩固}}
$$

## 2. 问题设定与适应信号

在线 VLN 同时包含两种序列结构：

- **样本内（intra-sample）：** 一条指令需要连续执行多步动作；
- **样本间（inter-sample）：** 不同指令或 episodes 按时间顺序持续到达。

频繁更新有较强可塑性，但容易造成错误累积和参数漂移；更新太慢又难以及时适应新环境。FSTTA 因此分别在 action-step 和 sample/episode 两个尺度上更新。

在第 $t$ 个动作步，基础 VLN 模型根据指令 $I$、全景视觉特征 $R_t$、物体特征 $O_t$ 和历史 $H_t$，输出当前可导航节点（含 STOP）的概率：

$$
\mathbf{s}_t = \phi(I,R_t,O_t,H_t;\Theta),
\qquad
\mathbf{s}_t \in \mathbb{R}^{|V_t|}.
$$

测试阶段没有动作标签，论文使用预测熵作为无监督目标：

$$
\mathcal{L}(\mathbf{s}_t;\Theta)
= -\sum_i s_{t,i}\log s_{t,i},
\qquad
\mathbf{g}_t = \nabla_{\Theta}\mathcal{L}(\mathbf{s}_t;\Theta).
$$

熵最小化只会提高置信度，并不能保证动作正确；FSTTA 的重点不是改变该 loss，而是从多个梯度和参数状态中提取相对可靠的更新方向。实验只更新基础模型最后四个 LayerNorm 的仿射参数，其余参数冻结。

## 3. Fast Update：样本内梯度一致性

### 3.1 梯度分解—累积

每隔 $M$ 个动作执行一次 Fast Update。第 $j$ 次更新收集最近 $M$ 个梯度：

$$
G_j =
\begin{bmatrix}
\tilde{\mathbf{g}}_{j,1}^{\top}\\
\vdots\\
\tilde{\mathbf{g}}_{j,M}^{\top}
\end{bmatrix}
\in \mathbb{R}^{M\times D},
\qquad
\bar{\mathbf{g}}_j = \frac{1}{M}\sum_{m=1}^{M}\tilde{\mathbf{g}}_{j,m}.
$$

中心化后计算梯度协方差并分解：

$$
\hat{G}_j = G_j - \mathbf{1}\bar{\mathbf{g}}_j^{\top},
$$

$$
(\lambda_{j,d},\mathbf{u}_{j,d})
= \operatorname{SVD}_d\!\left(
\frac{1}{M-1}\hat{G}_j^{\top}\hat{G}_j
\right).
$$

$\lambda_{j,d}$ 越大，表示最近梯度沿 $\mathbf{u}_{j,d}$ 的变化越大、步间一致性越低。论文用特征值倒数压低这些高方差方向：

$$
\nabla_j^{\mathrm{fast}}
= \sum_{d=1}^{D}
\frac{1}{\lambda_{j,d}}
\left\langle \bar{\mathbf{g}}_j,\mathbf{u}_{j,d}\right\rangle
\mathbf{u}_{j,d}.
$$

再将梯度范数校准到平均梯度的范数，使该操作主要改变方向组成，而不任意放大更新：

$$
\nabla_j^{\mathrm{fast}}
\leftarrow
\frac{\|\bar{\mathbf{g}}_j\|_2}
{\|\nabla_j^{\mathrm{fast}}\|_2}
\nabla_j^{\mathrm{fast}}.
$$

直觉是：同一指令和相邻观测产生的共同梯度方向更可能反映当前分布，变化剧烈的方向更可能包含 step-specific noise。不过这只是建模假设；环境真正发生突变时，高方差也可能是有用信号。

> **数值实现注意：** 由于 $\operatorname{rank}(\hat{G}_j)\le M-1$，当论文默认 $M=3$ 时，协方差至多有两个非零特征值，直接计算 $1/\lambda_{j,d}$ 会遇到零特征值。实际实现需使用低秩分解、截断或 $1/\max(\lambda_{j,d},\varepsilon)$；官方仓库当前代码使用 $\varepsilon=10^{-6}$。

### 3.2 Dynamic Learning Rate Scaling（DLR）

用协方差的迹衡量当前窗口的总体梯度变化：

$$
\sigma_j
= \operatorname{Tr}\!\left(
\frac{1}{M-1}\hat{G}_j^{\top}\hat{G}_j
\right)
= \sum_d \lambda_{j,d}.
$$

历史基准通过 EMA 更新：

$$
\bar{\sigma}
\leftarrow
\rho\bar{\sigma}+(1-\rho)\sigma_j.
$$

当前方差偏离历史水平越多，Fast 学习率越小：

$$
\gamma_j^{\mathrm{fast}}
= \operatorname{Trunc}_{[a,b]}\!\left(
1+\tau-|\sigma_j-\bar{\sigma}|
\right)\hat{\gamma}^{\mathrm{fast}}.
$$

最终更新为：

$$
\Theta_j
= \Theta_{j-1}
- \gamma_j^{\mathrm{fast}}\nabla_j^{\mathrm{fast}}.
$$

默认 DUET 配置为 $M=3$、$\tau=0.7$、$\rho=0.95$、$[a,b]=[0.9,1.1]$、$\hat{\gamma}^{\mathrm{fast}}=6\times10^{-4}$。因此 DLR 只是一个较保守的步长调节器。

## 4. Slow Update：样本间参数轨迹巩固

Fast Update 能快速适应，但长期依赖无监督梯度仍会累积误差。为此，每个样本 $o$ 结束后，FSTTA 保存其最终 Fast 状态 $\Theta_{o,J_o}$；每处理 $N$ 个样本，执行一次 Slow Update。

第 $l$ 个 Slow 窗口包含 $N+1$ 个状态：上一 Slow 状态作为锚点，加上 $N$ 个 Fast 终态：

$$
\mathcal{M}_l
= \{\tilde{\Theta}_{l,n}\}_{n=0}^{N}
\in \mathbb{R}^{(N+1)\times D},
$$

其中

$$
\tilde{\Theta}_{l,0}=\Theta^{(l-1)},
\qquad
\tilde{\Theta}_{l,n}
= \Theta_{N(l-1)+n,\,J_{N(l-1)+n}}
\quad(n\ge 1).
$$

对参数状态中心化，并分解参数协方差：

$$
\bar{\Theta}_l
= \frac{1}{N+1}\sum_{n=0}^{N}\tilde{\Theta}_{l,n},
\qquad
\hat{\mathcal{M}}_l
= \mathcal{M}_l-\mathbf{1}\bar{\Theta}_l^{\top},
$$

$$
(\epsilon_{l,d},\mathbf{z}_{l,d})
= \operatorname{SVD}_d\!\left(
\frac{1}{N}\hat{\mathcal{M}}_l^{\top}\hat{\mathcal{M}}_l
\right).
$$

与 Fast 阶段相反，Slow 阶段认为大特征值对应反复出现的主要参数变化，小特征值更可能是噪声，因此强化大方差主轴。

由于特征向量 $\mathbf{z}_{l,d}$ 与 $-\mathbf{z}_{l,d}$ 等价，论文用偏重近期样本的参考方向确定符号：

$$
\mathbf{h}_l
= \frac{1}{\sum_{i=0}^{N-1}q^i}
\sum_{n=1}^{N}
q^{N-n}
\left(\tilde{\Theta}_{l,0}-\tilde{\Theta}_{l,n}\right),
\qquad q\in(0,1).
$$

Slow 方向为：

$$
\nabla_l^{\mathrm{slow}}
= \sum_d
\Psi_d(\boldsymbol{\epsilon}_l,\mathbf{h}_l)
\operatorname{sign}\!\left(
\left\langle\mathbf{h}_l,\mathbf{z}_{l,d}\right\rangle
\right)
\mathbf{z}_{l,d},
$$

$$
\Psi_d(\boldsymbol{\epsilon}_l,\mathbf{h}_l)
= \epsilon_{l,d}
\frac{\|\mathbf{h}_l\|_2}{\|\boldsymbol{\epsilon}_l\|_2}.
$$

最后从上一 Slow 状态出发更新，而不是直接保留最新 Fast 状态：

$$
\Theta^{(l)}
= \Theta^{(l-1)}
- \gamma^{\mathrm{slow}}\nabla_l^{\mathrm{slow}}.
$$

默认 DUET 配置为 $N=4$、$q=0.1$、$\gamma^{\mathrm{slow}}=10^{-3}$。$q=0.1$ 意味着越新的 Fast 终态权重越大。

## 5. Fast 与 Slow 的区别

| 维度 | Fast Update | Slow Update |
| --- | --- | --- |
| 时间尺度 | action steps | samples / episodes |
| 分析对象 | 梯度 | Fast 后的参数状态 |
| 默认周期 | $M=3$ | $N=4$ |
| 大特征值的解释 | 梯度分歧较大 | 主要参数变化轨迹 |
| 对大特征值方向 | 抑制 | 强化 |
| 作用 | 快速适应 | 长期稳定与经验巩固 |
| 学习率 | 动态缩放 | 固定 |

两阶段形式上都是“中心化 $\rightarrow$ 协方差 $\rightarrow$ SVD $\rightarrow$ 方向重组”，但对大方差方向的解释相反。

## 6. 实验设置与结果

### 6.1 设置

- 数据集：REVERIE、R2R、SOON、连续环境 R2R-CE；
- 基础模型：DUET、HM3D，R2R-CE 还使用 WS-MGMap 和 BEVBert；
- 在线模拟：batch size 为 1，每个动作只前向一次；随机打乱测试样本后顺序输入，TTA 方法运行 5 次并报告均值；
- 常用指标：SR（成功率）、SPL（兼顾成功与路径效率）、TL（路径长度）、NE（终点误差）、OSR（轨迹曾进入成功区域的比例）；REVERIE 另有 RGS / RGSPL 衡量远程目标定位。

### 6.2 Ablation：REVERIE Val Unseen

| 方法 | SR $\uparrow$ | SPL $\uparrow$ |
| --- | ---: | ---: |
| DUET | 46.98 | 33.73 |
| Tent（每 $M$ 步取平均梯度） | 48.60 | 34.65 |
| Fast | 49.74 | 34.91 |
| Fast + DLR | 49.82 | 35.34 |
| Fast + DLR + Slow | **54.15** | **36.41** |

结论：Fast 的梯度分解优于简单平均；DLR 带来的增益较小；加入 Slow 后 SR 从 49.82 提升到 54.15，是该消融中最大的单步增益，说明跨样本参数巩固十分关键。

### 6.3 主要观察

- 在 REVERIE Val Unseen 上，DUET-FSTTA 相比 DUET 将 SR 从 46.98 提升到 54.15（$+7.17$ 个百分点），SPL 从 33.73 提升到 36.41（$+2.68$ 个百分点）。
- FSTTA 的单指令平均耗时为 135.61 ms，DUET 为 104.84 ms，约增加 29%；但它比 SAR 的 145.53 ms 略快。性能提升并非没有测试时计算代价。
- R2R、SOON 和 R2R-CE 上，多种基础模型总体也获得提升，说明 Fast-Slow 思路具有一定的跨数据集、跨模型泛化性；个别指标并非都改善。
- 路径长度 TL 有时增加。论文推测在线更新改变了原有动作模式，使 agent 更可能探索或回退；这也说明策略更新会改变后续观测分布。
- 遗忘实验中，模型先在 REVERIE Unseen 上适应，再回到 Seen 且关闭 TTA，SR 为 71.78，与论文表 3 所列基础模型 71.15 接近。但这只是一次有限 benchmark 检查，不能证明长期真实部署中不会遗忘。

## 7. 局限与研究启发

### 论文明确承认的局限

1. 只更新 normalization layers，不适用于没有这类层的模型；
2. “在线”环境由打乱并顺序输入现有测试集模拟，不是真实长期机器人数据流；
3. 测试时反向传播和 SVD 带来额外计算开销；
4. Fast / Slow 更新频率固定，尚未根据场景变化自适应触发。

### 更一般的启发

- **学习信号：** 熵最小化可能让错误预测变得更自信；可进一步结合时序一致性、多模态一致性、奖励、失败检测或 world-model prediction error。
- **时间层级：** 真实具身系统可能同时包含 frame、action、subtask、episode、environment 和 lifetime 等尺度，不应只设计单一更新周期。
- **更新载体：** 在线学习不一定要修改主干参数，也可以更新 adapter、LoRA、policy head、外部记忆或检索状态。
- **闭环分布漂移：** 参数影响动作，动作改变未来观测，观测又决定下一次更新：

$$
\Theta_t
\rightarrow a_t
\rightarrow o_{t+1}
\rightarrow \text{learning signal}
\rightarrow \Theta_{t+1}.
$$

FSTTA 尚未显式建模这一闭环。尤其当环境发生真实 transition 时，高梯度方差可能代表重要新信息，而非噪声；因此 event-triggered update 是自然的后续方向。

## 8. 阅读同类工作的四个问题

1. **学习信号从哪里来？** FSTTA：动作分布的预测熵。
2. **更新什么？** FSTTA：最后四个 LayerNorm 的仿射参数。
3. **经验怎样跨时间积累？** FSTTA：样本内梯度聚合 + 样本间参数轨迹巩固。
4. **怎样抑制错误累积和遗忘？** FSTTA：低方差梯度方向加权 + Slow 主参数轨迹更新。

## 9. 最终评价

FSTTA 更准确的定位是 **面向在线 VLN 的 continual test-time parameter adaptation**，而不是完整的 lifelong embodied learning，也不能直接等同于连续控制的 VLA 在线学习。它最有价值之处是把“短期适应”和“长期巩固”放进统一的梯度—参数分解框架；最明显的短板则是弱监督信号、固定时间窗口、理想化在线协议和额外计算成本。
