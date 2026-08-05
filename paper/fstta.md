# Fast-Slow Test-Time Adaptation for Online Vision-and-Language Navigation

**Junyu Gao, Xuan Yao, Changsheng Xu, ICML 2024**  
**关键词：** Online VLN, Test-Time Adaptation, Fast-Slow Update, Stability–Plasticity, Continual Adaptation

## 1\. 核心问题与主要思想

FSTTA 面向在线 Vision-and-Language Navigation（VLN），关注训练完成的导航模型如何在测试过程中利用持续到来的无标签数据进行适应。与普通图像分类 TTA 不同，VLN 同时存在两个时间尺度：一个 navigation sample / instruction 内包含连续多个 action decisions，即 **intra-sample multi-step structure**；不同导航任务又以 sample 为单位持续到达，构成 **inter-sample sequential structure**。如果过于频繁地根据单步测试数据更新，虽然适应快，但容易产生参数漂移、错误累积和 catastrophic forgetting；如果更新过慢或频繁 reset，又无法积累新的测试经验。因此论文将问题归纳为 **adaptability–stability dilemma**，并设计 Fast-Slow 双时间尺度机制：Fast Update 处理当前 episode 内的快速适应，Slow Update 负责跨 episode 的长期经验整合。

VLN 模型在第 $t$ 步根据语言指令 $I$、视觉特征 $R_t$、物体特征 $O_t$ 和历史 $H_t$ 输出

$$s_t=\phi(I,R_t,O_t,H_t;\Theta),\qquad s_t\in\mathbb R^{|V_t|}$$ 

其中 $s_t$ 是当前 navigable nodes 和 STOP 的动作概率分布。在 graph-based VLN 中，action 可以近似理解为选择下一个 navigable node，但 instruction 本身并不是预先分解成 node sequence，trajectory 是 agent 在执行过程中动态产生的。

测试时没有 ground-truth action，因此论文采用 entropy minimization：

$$L(s_t;\Theta)=-\sum_i s_{t,i}\log s_{t,i}, \qquad g_t=\nabla_\Theta L(s_t;\Theta).$$

该 loss 通过降低动作分布熵使模型更加确定，但它只优化 confidence，并不知道预测是否正确，因此存在“更自信地犯错”的风险。FSTTA 后续的核心并不是重新设计 loss，而是判断这些无监督梯度中哪些部分更值得相信。实验中只更新最后四个 LayerNorm 的 affine parameters，其余网络冻结，以降低测试时计算量和参数漂移风险。

## 2\. Fast Update：短时间尺度的梯度分析

Fast Update 每 $M$ 个 action steps 收集一次梯度，实验中 $M=3$。将最近梯度写成

$$G_j=\begin{bmatrix}g_1^T \\ \vdots \\ g_M^T\end{bmatrix}\in\mathbb R^{M\times D}$$

先计算平均梯度 $\bar{g}_j$，再中心化得到 $\hat{G}_j$，随后构造梯度协方差

$$C_j=\frac{1}{M-1}\hat{G}_j^T\hat{G}_j$$

并进行 SVD / eigendecomposition：

$$\lambda_{j,d},u_{j,d}=\mathrm{SVD}_d(C_j)$$

其中 $u_{j,d}\in\mathbb R^D$ 表示参数空间中的正交变化方向，$\lambda_{j,d}$ 表示最近几个梯度沿该方向的方差。Fast 阶段假设：短时间窗口内，多个 action steps 虽然动作与观测不同，但由于共享同一 instruction、environment 和相邻视觉状态，其中可能存在共同的 test-distribution adaptation direction；因此低方差方向代表较高 gradient agreement，而高方差方向更可能是 step-specific 或 noisy component。这个判断只是建模假设，因为真实的 step-specific adaptation 也可能表现为高方差。

作者将平均梯度投影到各个 SVD 方向，并使用特征值倒数重新加权：

# [  
\nabla_j^{fast}

\sum_d  
\frac{1}{\lambda_{j,d}}  
\langle \bar g_j,u_{j,d}\rangle u_{j,d}.  
]

因此低方差的一致方向被增强，高方差的冲突方向被抑制。为了避免 $1/\lambda_d$ 改变整体梯度尺度，又进行 norm calibration：

[  
\nabla_j^{fast}  
\leftarrow  
\frac{\nabla_j^{fast}|\bar g_j|_2}  
{|\nabla_j^{fast}|_2},  
]

使处理后的梯度保持与平均梯度相近的总体 magnitude，主要改变方向组成而不是任意放大 update size。

从线性代数角度看，因为中心化后的 $M$ 个梯度满足和为零，所以

[  
\mathrm{rank}(\hat G_j)\le M-1.  
]

因此当 $M=3$ 时，即使参数空间 $D$ 很高，当前窗口最多只能估计两个独立的梯度变化方向。可以把 Fast Update 看作在高维参数空间中估计一个极低维的局部 gradient variation subspace。

### Dynamic Learning Rate Scaling

Fast Update 还利用所有特征值之和

[  
\sigma_j=\sum_d\lambda_{j,d}  
]

衡量当前梯度窗口的总体变化强度，并通过 EMA 维护历史基准 $\bar\sigma$。学习率为

# [  
\gamma_j^{fast}

\mathrm{Trunc}  
\left(  
1+\tau-|\sigma_j-\bar\sigma|  
\right)  
\hat\gamma^{fast}.  
]

当当前梯度方差与历史正常水平差异较大时，作者认为该更新更异常，因此降低 learning rate；差异较小时则允许略大的更新。实验中 $\tau=0.7,\rho=0.95$，缩放被截断在 $[0.9,1.1]$，基础 fast learning rate 为 $6\times10^{-4}$，因此 DLR 实际上是一种较保守的步长调节机制。最终：

[  
\Theta_j=\Theta_{j-1}-\gamma_j^{fast}\nabla_j^{fast}.  
]

Fast Update 可以概括为：**利用短窗口梯度一致性判断更新方向是否可靠，再根据当前窗口相对历史的异常程度控制更新幅度。**

## 3\. Slow Update：跨 Sample 的参数轨迹分析

Fast Update 提高了短期适应能力，但如果模型持续依赖无监督 entropy gradient 更新，长期仍可能发生 parameter drift。因此 Slow Update 不再分析瞬时梯度，而是分析一段时间内模型经过 Fast Adaptation 后形成的 **parameter trajectory**。每个 sample 完成后保存最终 fast-adapted state $\Theta_{o,J_o}$，每 $N=4$ 个 samples 进行一次 Slow Update，并额外加入上一次 Slow state (\Theta_{$l-1$}) 作为历史锚点。

对这些 parameter states 求均值、中心化并计算协方差：

[  
\frac1N\hat{\mathcal M}_l^T\hat{\mathcal M}_l,  
]

随后做 SVD：

# [  
$\epsilon_{l,d},z_{l,d}$

\mathrm{SVD}_d  
\left(  
\frac1N\hat{\mathcal M}_l^T\hat{\mathcal M}_l  
\right).  
]

这里 $z_{l,d}$ 表示历史参数轨迹的 principal direction，$\epsilon_{l,d}$ 表示参数沿这一方向的变化幅度。与 Fast 阶段不同，Slow 阶段把 **大方差解释为长期主要变化方向**，因此高 $\epsilon_d$ 会获得更大权重。两阶段虽然形式上都是 Centering → Covariance → SVD → Recombination，但语义相反：Fast 寻找 gradient consensus，Slow 寻找 principal parameter trajectory。

由于 eigenvector 的符号 $z_d$ 与 $-z_d$ 等价，作者额外构造 recency-weighted reference direction：

[  
h_l=  
\frac{1}{\sum_{i=0}^{N-1}q^i}  
\sum_{n=1}^{N}  
q^{N-n}  
(\Theta_{\tilde l,0}-\Theta_{\tilde l,n}),  
]

实验中 $q=0.1$，因此越新的 fast-adapted parameter state 权重越大。最终 Slow direction 为

# [  
\nabla_l^{slow}

\sum_d  
\Psi_d$\epsilon_l,h_l$  
,\mathrm{sign}  
\big(  
\langle h_l,z_{l,d}\rangle  
\big)  
z_{l,d},  
]

其中

# [  
\Psi_d$\epsilon_l,h_l$

\epsilon_{l,d}  
\frac{|h_l|_2}{|\epsilon_l|_2}.  
]

特征值决定各主轴的重要程度，$h_l$ 决定方向符号和整体尺度。Slow Update 最终从上一稳定状态出发：

# [  
\Theta_{$l$}

## \Theta_{$l-1$}

\gamma^{slow}\nabla_l^{slow},  
\qquad  
\gamma^{slow}=10^{-3}.  
]

因此 Slow 并不是简单保留最新 Fast model，而是把最近多个 short-term adaptations 当作候选经验，通过参数轨迹分析决定哪些趋势值得长期巩固。

## 4\. Fast / Slow 的核心区别

| 
 | 

Fast Update

 | 

Slow Update

 |
| --- | --- | --- |
| 

时间尺度

 | 

action steps

 | 

samples / episodes

 |
| 

分析对象

 | 

gradient

 | 

parameter state

 |
| 

周期

 | 

$M=3$

 | 

$N=4$

 |
| 

SVD 大方差含义

 | 

gradient disagreement

 | 

principal parameter trajectory

 |
| 

大方差处理

 | 

抑制

 | 

强化

 |
| 

目标

 | 

快速适应

 | 

长期稳定与经验巩固

 |
| 

Learning rate

 | 

dynamic

 | 

fixed

 |

从 embodied online learning 的角度，可以把两者理解为 **fast plasticity + slow consolidation**。

## 5\. 实验指标

主要指标中，\*\*SR（Success Rate）\*\*表示最终距离目标小于规定阈值的 episode 比例，反映“能否成功到达”；\*\*SPL（Success weighted by Path Length）\*\*同时考虑成功率和路径效率，因此比单独 SR 更能反映成功且高效的导航。\*\*TL（Trajectory Length）\*\*是实际路径长度，本身越短通常越好，但必须和 success 一起解释；\*\*NE（Navigation Error）\*\*是最终位置到目标的距离，越低越好；\*\*OSR（Oracle Success Rate）\*\*判断 trajectory 是否曾经进入成功区域，因此能够区分“走到附近但 STOP 失败”和“根本没走到”；REVERIE 还包括 **RGS / RGSPL**，分别衡量 remote object grounding success 及其路径效率。

## 6\. 关键实验结果

REVERIE Val Unseen 的 ablation 最能说明各模块的贡献：

| 
Method

 | 

SR

 | 

SPL

 |
| --- | --- | --- |
| 

DUET

 | 

46.98

 | 

33.73

 |
| 

TENT

 | 

48.60

 | 

34.65

 |
| 

Fast

 | 

49.74

 | 

34.91

 |
| 

Fast + DLR

 | 

49.82

 | 

35.34

 |
| 

Fast + DLR + Slow

 | 

**54.15**

 | 

**36.41**

 |

普通 entropy-based TTA 已能使 SR 从 46.98 提高到 48.60，说明 test-time adaptation 本身有效；加入 gradient decomposition 后进一步提高到 49.74，支持短窗口 gradient agreement filtering 的价值；DLR 对 SR 的额外提升只有 0.08，更多表现为辅助稳定作用；真正最大的增益来自 Slow Update，SR 从 49.82 提升到 54.15，因此从实验结果看，**跨 sample 的长期参数整合是整套方法最重要的性能来源之一**。

与 TENT、SAR、CoTTA、EATA 等现有 TTA 方法相比，FSTTA 在 REVERIE 上取得更好的 SR/SPL，支持作者“普通 TTA 没有显式处理 VLN 的 intra-/inter-sample sequence structure”的论点。Seen/Unseen 混合实验中 TENT 甚至可能降低 baseline performance，而 FSTTA 仍保持提升，说明 online adaptation 本身并不天然有益，关键在于如何约束持续更新。

在 catastrophic forgetting 实验中，原始 REVERIE Seen SR 为 71.15；模型先在 Unseen 上经历 FSTTA，再回 Seen 且不继续 adaptation 时 SR 为 71.78，说明在该 benchmark 设置下没有出现明显的灾难性遗忘。这个结果只能视为支持性证据，不能推广为真实长期机器人部署中不会遗忘。

FSTTA 还在 REVERIE、R2R、SOON 和 continuous R2R-CE，以及 DUET、HM3D、BEVBert 等不同 base models 上得到总体 favorable results，说明其 Fast-Slow adaptation idea 有一定跨模型、跨数据集泛化性。部分实验中 TL 会增加，表明参数在线变化不仅改变 prediction confidence，也会改变 agent 的 exploration / backtracking 行为，从而进一步改变未来收到的数据。

## 7\. 对具身在线学习最重要的启发

这篇论文最值得带入后续具身在线学习研究的，不是具体的 SVD 公式，而是几个更一般的问题。

首先，**在线学习的核心是 stability–plasticity trade-off**。具身 agent 必须能够根据新环境快速改变，同时又不能因为少量无标签经验破坏已有能力。FSTTA 用 Fast/Slow 两个时间尺度分别承担 adaptation 和 consolidation，这一问题会在 continual learning、lifelong learning、online VLA adaptation 和 online world model 中反复出现。

其次，具身在线学习应显式考虑 **temporal hierarchy**。FSTTA 只区分 action-step 与 sample/episode 两级，但更一般的机器人系统可能同时存在 frame、action、subtask、episode、environment、lifetime 等多个尺度，不同尺度的经验可能适合写入不同类型的 memory 或 parameters。

第三，需要持续追问 **learning signal 从哪里来**。FSTTA 使用 entropy minimization，但 confidence 不等于 correctness，这是其基本弱点。未来方法可以利用 temporal consistency、multimodal consistency、prediction error、reward、human feedback、failure detection 或 world-model error 等更可靠的自监督信号。

第四，需要区分 **在线更新的载体**。FSTTA 只修改少量 LayerNorm parameters，但 embodied online learning 并不一定意味着每一步都做 parameter backpropagation；可更新的对象还可能包括 adapter、LoRA、policy head、external memory、retrieval state、latent world model 或完整模型。更新位置直接决定 adaptation capacity、compute、stability 和 forgetting risk。

第五，真正困难的是 **experience accumulation**。单个 sample 上性能变好并不等于长期在线学习，关键在于过去经验如何影响未来，同时避免错误累积。FSTTA 的 Fast gradient aggregation + Slow parameter consolidation 是一种简单但清晰的实现：短期更新先作为临时经验，经过跨 sample 分析后再选择性写入长期状态。

第六，具身数据具有典型的 **policy-induced distribution shift**。模型参数 $\Theta_t$ 决定动作 $a_t$，动作决定未来观察 $o_{t+1}$，新的观察又产生学习信号并更新 $\Theta_{t+1}$：

[  
\Theta_t\rightarrow a_t\rightarrow o_{t+1}  
\rightarrow\text{learning signal}\rightarrow\Theta_{t+1}.  
]

因此 agent 不是被动从固定数据分布中取样，而是在主动改变自己未来会看到的数据。FSTTA 已经出现了这一现象，例如 online adaptation 会改变 trajectory length，但论文并没有系统建模这一闭环，这是进一步走向真实 embodied online learning 时必须关注的问题。

最后，FSTTA 的固定 $M=3,N=4$ 暗含“固定时间窗口内的数据可以共同聚合”的假设，但环境真正发生 transition 时，高 gradient variance 可能是有效新信息而不是 noise。因此一个自然的后续方向是 **adaptive / event-triggered update**：根据 scene change、gradient similarity、uncertainty、instruction progress 或 failure signal 决定“何时学习”，而不只是研究“如何学习”。论文也将固定 update frequency 列为 limitation。

## 8\. 论文局限与定位

FSTTA 更准确地属于 **online / continual test-time parameter adaptation for VLN**，不能代表完整的 embodied online learning，也不能直接等同于现代 VLA online learning。论文的 online setting 是通过将现有 test samples shuffle 后 sequentially 输入来模拟，并非真实长期机器人部署；方法增加了测试时反向传播开销，只适应 normalization layers，Fast/Slow invocation frequency 也是固定的。

对于 VLA，action 往往是 continuous control 或 action chunk，无法简单照搬 navigable-node entropy，因此 FSTTA 对 VLA 更有价值的是 **双时间尺度 adaptation、稳定性–可塑性、经验巩固、无标签 learning signal reliability** 等组织思想，而不是具体的 entropy loss 或 SVD weighting。

## 9\. 后续阅读时可固定使用的四个问题

以后读 embodied online learning / TTA / TTT / continual VLA 论文，可以固定检查：

1. **Learning signal 从哪里来？** FSTTA：prediction entropy。
    
2. **更新什么？** FSTTA：最后四个 LayerNorm parameters。
    
3. **经验怎样跨时间积累？** FSTTA：短期 gradient aggregation + 长期 parameter consolidation。
    
4. **怎样避免错误累积和遗忘？** FSTTA：gradient agreement filtering + slow parameter trajectory analysis。
    

## 10\. 最终总结

FSTTA 的形式可以概括为

[  
\text{Gradient SVD}  
+  
\text{Parameter SVD},  
]

但从具身在线学习角度，更重要的抽象是

[  
\boxed{  
\text{Fast Plasticity}  
+  
\text{Slow Consolidation}  
}  
]

即具身 agent 不仅需要根据当前交互快速适应，还必须决定哪些短期变化值得长期保留。实验中 Slow Update 带来的增益明显大于 DLR，也进一步说明“**如何把短期在线适应转化为长期稳定经验**”可能比单纯设计更激进的单步更新规则更值得关注。FSTTA 因此适合作为具身在线学习的入门论文：它没有解决真实 lifelong embodied learning 的全部问题，但把 learning signal、序列数据、稳定性–可塑性、经验积累和遗忘问题集中到了一个相对简洁的 VLN 框架中。

