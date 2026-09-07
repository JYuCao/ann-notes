# FSTTA：Fast Update 与 Slow Update 详解

> Junyu Gao, Xuan Yao, Changsheng Xu. *Fast-Slow Test-Time Adaptation for Online Vision-and-Language Navigation*. ICML 2024.  
> 论文：[PMLR](https://proceedings.mlr.press/v235/gao24p.html) ｜ [arXiv](https://arxiv.org/abs/2311.13209) ｜ [官方代码](https://github.com/Feliciaxyao/ICML2024-FSTTA)

## 1. 核心思想

FSTTA 使用同一个熵最小化信号，在两个时间尺度上处理不同对象：

- **Fast Update：** 分析一个 episode 内连续动作产生的梯度，快速适应当前环境；
- **Slow Update：** 分析多个 episodes 结束后形成的参数轨迹，将主要变化方向写入稳定模型。

可以概括为：

$$
\boxed{
\text{Fast：梯度一致性}
+
\text{Slow：参数轨迹巩固}
}
$$

默认配置为 $M=3$、$N=4$，整体时间线如下：

```text
Episode 1:
  step 1 ─ step 2 ─ step 3 → Fast Update
  step 4 ─ step 5 ─ step 6 → Fast Update
  episode 结束 → 保存最终 Fast 参数 Θ₁,J₁

Episode 2:
  每 3 步做一次 Fast Update
  episode 结束 → 保存 Θ₂,J₂

Episode 3:
  ... → 保存 Θ₃,J₃

Episode 4:
  ... → 保存 Θ₄,J₄
  上一 Slow 状态 + 4 个 Fast 终态 → Slow Update

Episode 5:
  从新的 Slow 状态继续 Fast Adaptation
```

## 2. 共同的无监督适应信号

第 $t$ 个动作步，VLN 模型根据语言指令 $I$、视觉特征 $R_t$、物体特征 $O_t$ 和历史 $H_t$ 输出动作概率：

$$
\mathbf{s}_t
=
\phi(I,R_t,O_t,H_t;\Theta),
\qquad
\mathbf{s}_t\in\mathbb{R}^{|V_t|}.
$$

测试阶段没有真实动作标签，因此论文使用预测熵：

$$
\mathcal{L}_t
=
-\sum_i s_{t,i}\log s_{t,i},
\qquad
\mathbf{g}_t
=
\nabla_\Theta\mathcal{L}_t.
$$

最小化熵会让动作概率更加尖锐，例如：

$$
[0.4,0.35,0.25]
\longrightarrow
[0.8,0.15,0.05].
$$

但置信度不等于正确性，模型也可能变得“更自信地犯错”。因此 FSTTA 不直接相信单步梯度，而是分析多个梯度和多个参数状态。实验中只更新最后四个 LayerNorm 的仿射参数，其余参数冻结。

## 3. Fast Update：从近期梯度中寻找一致方向

### 3.1 收集短窗口梯度

每 $M$ 个动作步执行一次 Fast Update。第 $j$ 个窗口中的梯度为：

$$
\tilde{\mathbf{g}}_{j,m}
=
\mathbf{g}_{M(j-1)+m},
\qquad
m=1,\ldots,M.
$$

将它们堆叠成矩阵：

$$
G_j
=
\begin{bmatrix}
\tilde{\mathbf{g}}_{j,1}^{\top}\\
\vdots\\
\tilde{\mathbf{g}}_{j,M}^{\top}
\end{bmatrix}
\in\mathbb{R}^{M\times D}.
$$

平均梯度为：

$$
\bar{\mathbf{g}}_j
=
\frac{1}{M}
\sum_{m=1}^{M}\tilde{\mathbf{g}}_{j,m}.
$$

直接使用平均梯度会同时混入：

- 多个动作都支持的共同适应方向；
- 由特定视角、动作或错误预测产生的瞬时噪声。

Fast Update 的目标是降低第二类成分的影响。

### 3.2 用协方差度量梯度一致性

先对梯度矩阵中心化：

$$
\hat{G}_j
=
G_j-\mathbf{1}\bar{\mathbf{g}}_j^\top.
$$

再计算协方差：

$$
C_j
=
\frac{1}{M-1}
\hat{G}_j^\top\hat{G}_j.
$$

对其进行特征分解：

$$
C_j\mathbf{u}_{j,d}
=
\lambda_{j,d}\mathbf{u}_{j,d}.
$$

其中：

- $\mathbf{u}_{j,d}$ 是参数空间中的一个正交方向；
- $\lambda_{j,d}$ 是近期梯度沿该方向的方差。

如果三个梯度在某个方向上的投影为

$$
[1.0,1.1,0.9],
$$

说明该方向方差小、多个动作意见一致。若投影为

$$
[2.0,-1.0,1.0],
$$

则说明不同动作在该方向上存在明显分歧。

### 3.3 重组一致梯度

平均梯度在第 $d$ 个特征方向上的分量为：

$$
\left\langle
\bar{\mathbf{g}}_j,\mathbf{u}_{j,d}
\right\rangle
\mathbf{u}_{j,d}.
$$

论文使用特征值倒数加权：

$$
\nabla_j^{\mathrm{fast}}
=
\sum_{d=1}^{D}
\frac{1}{\lambda_{j,d}}
\left\langle
\bar{\mathbf{g}}_j,\mathbf{u}_{j,d}
\right\rangle
\mathbf{u}_{j,d}.
$$

因此：

- 小方差、一致性高的方向被增强；
- 大方差、分歧较大的方向被抑制；
- 去掉 $1/\lambda_{j,d}$ 后，各正交分量之和就是普通平均梯度。

从矩阵角度看，它近似于逆协方差预条件：

$$
\nabla_j^{\mathrm{fast}}
\approx
C_j^\dagger\bar{\mathbf{g}}_j.
$$

### 3.4 梯度范数校准

$1/\lambda$ 可能使更新幅度失控，因此将处理后的梯度缩放回平均梯度的长度：

$$
\nabla_j^{\mathrm{fast}}
\leftarrow
\frac{\|\bar{\mathbf{g}}_j\|_2}
{\|\nabla_j^{\mathrm{fast}}\|_2}
\nabla_j^{\mathrm{fast}}.
$$

这样，SVD 主要决定更新方向，平均梯度则控制总体更新强度。

### 3.5 Dynamic Learning Rate Scaling

Fast Update 还计算协方差矩阵的迹：

$$
\sigma_j
=
\operatorname{Tr}(C_j)
=
\sum_d\lambda_{j,d}.
$$

$\sigma_j$ 表示当前窗口的总体梯度波动。历史基准通过 EMA 维护：

$$
\bar{\sigma}
\leftarrow
\rho\bar{\sigma}
+
(1-\rho)\sigma_j.
$$

学习率为：

$$
\gamma_j^{\mathrm{fast}}
=
\operatorname{Trunc}_{[a,b]}
\left(
1+\tau-|\sigma_j-\bar{\sigma}|
\right)
\hat{\gamma}^{\mathrm{fast}}.
$$

含义是：

- 当前梯度波动接近历史水平时，允许正常或稍大的更新；
- 当前波动突然异常时，减小学习率；
- 截断函数防止学习率变化过大。

DUET 默认配置为：

$$
M=3,
\quad
\hat{\gamma}^{\mathrm{fast}}=6\times10^{-4},
\quad
\rho=0.95,
\quad
\tau=0.7,
\quad
[a,b]=[0.9,1.1].
$$

最终更新为：

$$
\Theta_j
=
\Theta_{j-1}
-
\gamma_j^{\mathrm{fast}}
\nabla_j^{\mathrm{fast}}.
$$

### 3.6 数值实现问题

中心化矩阵满足：

$$
\operatorname{rank}(\hat{G}_j)\le M-1.
$$

默认 $M=3$ 时，无论参数维度 $D$ 多大，协方差最多只有两个非零特征值。因此论文直接写出的 $1/\lambda_{j,d}$ 会遇到零特征值。实际实现需要采用：

$$
\frac{1}{\max(\lambda_{j,d},\varepsilon)},
$$

或者只在非零低秩子空间中计算。官方仓库当前实现使用 $\varepsilon=10^{-6}$。

## 4. Slow Update：从参数轨迹中提取长期趋势

Fast Update 能快速适应，但频繁使用无监督梯度仍可能导致参数漂移、错误累积和遗忘。Slow Update 不再分析瞬时梯度，而是分析多个 episodes 结束后的参数状态。

### 4.1 收集 $N+1$ 个参数状态

第 $o$ 个 episode 结束后，保存最终 Fast 状态：

$$
\Theta_{o,J_o},
$$

其中 $J_o$ 是该 episode 内最后一次 Fast Update 的编号。

第 $l$ 个 Slow 窗口包含：

$$
\mathcal{M}_l
=
\left\{
\tilde{\Theta}_{l,0},
\tilde{\Theta}_{l,1},
\ldots,
\tilde{\Theta}_{l,N}
\right\}
\in\mathbb{R}^{(N+1)\times D}.
$$

其中：

$$
\tilde{\Theta}_{l,0}
=
\Theta^{(l-1)}
$$

是上一次 Slow Update 得到的稳定锚点，其余 $N$ 个状态是最近 episodes 的 Fast 终态。因此默认 $N=4$ 时，PCA 分析的是一个 Slow 锚点和四个 Fast 终态。

### 4.2 对参数轨迹进行 PCA

参数均值为：

$$
\bar{\Theta}_l
=
\frac{1}{N+1}
\sum_{n=0}^{N}\tilde{\Theta}_{l,n}.
$$

中心化参数矩阵：

$$
\hat{\mathcal{M}}_l
=
\mathcal{M}_l
-
\mathbf{1}\bar{\Theta}_l^\top.
$$

参数协方差为：

$$
P_l
=
\frac{1}{N}
\hat{\mathcal{M}}_l^\top
\hat{\mathcal{M}}_l.
$$

特征分解得到：

$$
P_l\mathbf{z}_{l,d}
=
\epsilon_{l,d}\mathbf{z}_{l,d}.
$$

其中：

- $\mathbf{z}_{l,d}$ 是参数轨迹的一个主方向；
- $\epsilon_{l,d}$ 是参数沿该方向的变化幅度。

假设最近几个 Fast 状态主要沿 $\mathbf{z}_1$ 移动：

$$
0.1\mathbf{z}_1,
\quad
0.2\mathbf{z}_1,
\quad
0.3\mathbf{z}_1,
\quad
0.4\mathbf{z}_1,
$$

而其他方向只有小幅随机抖动，那么 $\epsilon_{l,1}$ 会比较大。Slow Update 将 $\mathbf{z}_1$ 解释为值得巩固的主要适应路径。

### 4.3 用参考方向解决特征向量符号问题

PCA 的特征向量存在符号不确定性：

$$
\mathbf{z}_{l,d}
\quad\text{和}\quad
-\mathbf{z}_{l,d}
$$

表示同一个主轴，但参数更新必须知道应该往哪一侧移动。因此论文构造参考方向：

$$
\mathbf{h}_l
=
\frac{1}{\sum_{i=0}^{N-1}q^i}
\sum_{n=1}^{N}
q^{N-n}
\left(
\tilde{\Theta}_{l,0}
-
\tilde{\Theta}_{l,n}
\right).
$$

它是“Slow 锚点减去各个 Fast 状态”的加权平均。默认 $q=0.1$、$N=4$ 时，未归一化权重大致为：

$$
0.001,
\quad
0.01,
\quad
0.1,
\quad
1.
$$

因此越新的 Fast 状态权重越大，最近一个 episode 占主导。

### 4.4 重组 Slow 方向

先根据参考方向确定每个主轴的正负号：

$$
\operatorname{sign}
\left(
\left\langle
\mathbf{h}_l,\mathbf{z}_{l,d}
\right\rangle
\right).
$$

再根据参数特征值分配权重：

$$
\Psi_d(\boldsymbol{\epsilon}_l,\mathbf{h}_l)
=
\epsilon_{l,d}
\frac{\|\mathbf{h}_l\|_2}
{\|\boldsymbol{\epsilon}_l\|_2}.
$$

最终 Slow 方向为：

$$
\nabla_l^{\mathrm{slow}}
=
\sum_d
\Psi_d(\boldsymbol{\epsilon}_l,\mathbf{h}_l)
\operatorname{sign}
\left(
\left\langle
\mathbf{h}_l,\mathbf{z}_{l,d}
\right\rangle
\right)
\mathbf{z}_{l,d}.
$$

这里有三个重要细节：

1. $\epsilon_{l,d}$ 决定哪些参数主轴更重要；
2. $\mathbf{h}_l$ 提供各主轴的符号和总体尺度；
3. Slow 不使用 $\langle\mathbf{h}_l,\mathbf{z}_{l,d}\rangle$ 的投影大小，只使用其符号。

最后从上一 Slow 锚点更新：

$$
\Theta^{(l)}
=
\Theta^{(l-1)}
-
\gamma^{\mathrm{slow}}
\nabla_l^{\mathrm{slow}}.
$$

由于

$$
\mathbf{h}_l
\approx
\Theta^{(l-1)}
-
\Theta^{\mathrm{fast}},
$$

减去与 $\mathbf{h}_l$ 同向的 Slow 梯度，相当于让 Slow 参数向近期 Fast 状态的主要轨迹移动。DUET 默认使用：

$$
N=4,
\qquad
q=0.1,
\qquad
\gamma^{\mathrm{slow}}=10^{-3}.
$$

## 5. 为什么 Fast 抑制大方差，Slow 却强化大方差？

| 维度 | Fast Update | Slow Update |
| --- | --- | --- |
| 数据对象 | 相邻动作的梯度 | 多个 episodes 的参数状态 |
| 大方差含义 | 梯度意见不一致 | 参数轨迹的主要变化轴 |
| 大方差处理 | 抑制 | 强化 |
| 目标 | 找共同梯度 | 找主要迁移路径 |

Fast 的假设是：

> 同一局部时间窗口中，可靠梯度应该相互一致，因此高方差更像噪声。

Slow 的假设是：

> 跨 episode 的参数状态构成适应轨迹，变化最大的主轴代表主要迁移趋势。

这不是数学上的必然结论，而是论文的建模假设。高梯度方差也可能代表真实环境突变；参数轨迹的大方差也可能由异常 episode 引起。

## 6. 简化伪代码

```text
初始化：
    Slow 锚点 Θslow ← 预训练参数
    当前参数 Θ ← Θslow

对每个 episode o：

    清空当前 Fast 梯度窗口

    对每个动作步 t：
        计算动作概率 s_t
        根据 s_t 执行动作
        计算熵损失和梯度 g_t
        保存 g_t

        如果已收集 M 个梯度：
            计算梯度协方差
            抑制高方差方向
            校准梯度范数
            根据历史方差调整学习率
            更新当前参数 Θ

    保存 episode 最终参数 Θ_{o,J_o}

    如果已收集 N 个 episodes：
        加入上一 Slow 锚点 Θslow
        对 N+1 个参数状态做 PCA
        构造近期加权参考方向 h
        强化参数轨迹的主方向
        从旧锚点计算新的 Θslow
        将其用于后续 Fast Adaptation
        清空参数轨迹
```

## 7. 最终理解

Fast Update 解决的问题是：

> 最近几个动作产生的无监督梯度中，哪些方向比较一致，适合立即更新？

Slow Update 解决的问题是：

> 最近几个 episodes 造成的临时参数变化中，哪些主方向值得写入稳定模型？

从线性代数角度，可以近似理解为：

$$
\text{Fast}
\approx
\text{inverse-covariance gradient filtering},
$$

$$
\text{Slow}
\approx
\text{recency-oriented signed PCA consolidation}.
$$

FSTTA 真正有价值的是双时间尺度的组织方式；其主要风险则来自两个未经保证的假设：高梯度方差未必是噪声，大参数方差也未必是可靠经验。
