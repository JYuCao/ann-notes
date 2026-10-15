# SiT：随机插值、速度场与 ODE / SDE 采样

## 阅读导航与符号约定

前置知识见 [DDPM](diffusion.md) 和 [DiT 架构](DiT.md)。本文先给出 SiT 主线，再展开数学推导与实验；DDPM 到 VP-SDE 的连续化推导保留在文末附录。

- 快速复习：读第 1 节总览和第 3 节实验。
- 理解 ODE / SDE 的关系：读第 2 节，尤其 2.3、2.4。
- 补连续时间基础：读附录 A，再回到正文。

本文统一采用 $t=0$ 为数据、$t=1$ 为噪声；生成沿 $1\rightarrow0$ 积分。$x_t$ 表示端点随机采样后构造的插值变量，$X_t$ 表示 ODE / SDE 的动态过程。两者可以具有相同边缘分布，但无需具有相同轨迹。

| 含义 | DDPM 离散记号 | 本文连续记号 |
|---|---|---|
| 单步信号保留量 | $\alpha_k=1-\beta_k$ | 不等同于 $\alpha_t$ |
| 累计信号幅度 | $\sqrt{\bar\alpha_k}$ | $\alpha_t$ |
| 累计噪声幅度 | $\sqrt{1-\bar\alpha_k}$ | $\sigma_t$ |
| 连续噪声率 | $\beta_k\approx\beta(t_k)\Delta t$ | $\beta(t)$ |
| 采样扩散强度 | 由单步方差描述 | $w_t$，SDE 噪声系数为 $\sqrt{w_t}$ |

除非另行说明，$x_*\sim p_{\mathrm{data}}$ 与 $\epsilon\sim\mathcal N(0,I)$ 独立；公式省略类别条件。图像实验实际在 VAE latent 空间中建模，下面用 $x$ 统一表示建模变量。


## 1. 总览

SiT（Scalable Interpolant Transformers）的核心思想是：
先定义一条从数据分布到噪声分布的连续概率路径，再学习描述这条路径如何演化的速度场。这个速度场既可以用于确定性的 ODE 生成，也可以结合 Score 构造随机的 SDE 生成。

它通过随机插值统一了 Flow Matching 与 Diffusion Model，使同一个生成模型能够使用 ODE 或 SDE 进行采样。

### 1.1 随机插值

定义数据分布与高斯噪声之间的连续路径：

$$
\boxed{x_t=\alpha_t x_*+\sigma_t\epsilon}
$$

其中 $x_*\sim p_{\mathrm{data}}$ 与 $\epsilon\sim\mathcal N(0,I)$ 独立，$t\in[0,1]$，并满足：

$$
\alpha_0=1,\ \sigma_0=0,\qquad
\alpha_1=0,\ \sigma_1=1
$$

因此，$t=0$ 为真实数据，$t=1$ 为纯高斯噪声。

与传统 DDPM 不同，SiT 可以灵活设计插值路径：

- **Linear：** $\alpha_t=1-t,\ \sigma_t=t$
- **GVP：** $\alpha_t=\cos(\pi t/2),\ \sigma_t=\sin(\pi t/2)$

随机性来自端点采样；固定 $x_*$ 和 $\epsilon$ 后，插值轨迹是确定性的。

### 1.2 Velocity Matching

对插值路径求时间导数：

$$
\dot x_t=\dot\alpha_t x_*+\dot\sigma_t\epsilon
$$

由于同一位置可能对应不同插值轨迹，真实速度场定义为条件期望：

$$
\boxed{v(x,t)=\mathbb E[\dot x_t\mid x_t=x]}
$$

利用 MSE 训练神经网络预测速度：

$$
\mathcal L_v=
\mathbb E\left[
\|v_\theta(x_t,t)-\dot x_t\|^2
\right]
$$

对于 Linear 插值：

$$
x_t=(1-t)x_*+t\epsilon,\qquad
\dot x_t=\epsilon-x_*
$$

这就是 Flow Matching 的基本训练形式。

### 1.3 Velocity 与 Score 的联系

Score：

$$
s(x,t)=\nabla_x\log p_t(x)=-\frac{\mathbb E[\epsilon\mid x_t=x]}{\sigma_t}
$$

联立以下三个式子：

$$
\begin{aligned}
x&=\alpha_t\mathbb E[x_*\mid x_t=x]
+\sigma_t\mathbb E[\epsilon\mid x_t=x]\\
v(x,t)&=\dot\alpha_t\mathbb E[x_*\mid x_t=x]
+\dot\sigma_t\mathbb E[\epsilon\mid x_t=x]\\
s(x,t)&=-\frac1{\sigma_t}\mathbb E[\epsilon\mid x_t=x]
\end{aligned}
$$

前两个式子构成了关于两个条件期望的线性方程组。
消去它们，就能得到：
$$
\boxed{
s(x,t)=
\frac{1}{\sigma_t}
\frac{
\alpha_t v(x,t)-\dot\alpha_t x
}{
\dot\alpha_t\sigma_t-\alpha_t\dot\sigma_t
}
}
$$

这是 SiT 论文公式（9），适用于 $\sigma_t\ne0$ 且 $\dot\alpha_t\sigma_t-\alpha_t\dot\sigma_t\ne0$ 的时间点。本文讨论的 Linear / GVP 路径在 $0<t<1$ 满足这些条件；端点需要取极限或单独处理，不能直接代入含零分母的表达式。

该公式本质是：
**给定插值路径后，我们只需要训练 Velocity 或 Score 中的一个，便可以计算另一个。**

这建立了 Flow Matching 与 Score-based Diffusion 的联系。

例如 Linear 插值：

$$
\boxed{
s(x,t)=-\frac{x+(1-t)v(x,t)}{t}
}
$$

### 1.4 ODE 与 SDE 生成

SiT 可以使用同一个训练好的速度场构造两种采样过程。

**确定性 ODE：**

$$
\boxed{dX_t=v_\theta(X_t,t)dt}
$$

**随机性 SDE：**

$$
\boxed{
dX_t=
\left[v_\theta(X_t,t)-\frac12w_t s_\theta(X_t,t)\right]dt
+\sqrt{w_t}\,d\bar W_t
}
$$

两者均从 $t=1$ 的高斯噪声反向求解至 $t=0$。

其中 $w_t\geq0$ 控制采样时的随机扩散强度：

- $w_t=0$：退化为 ODE。
- $w_t>0$：使用 SDE，加入 Score 修正与随机扰动。

在速度场和 Score 准确、数值求解精确的理想条件下，ODE 与 SDE 虽然样本轨迹不同，却具有相同的边缘分布演化 $p_t$。

### 1.5 框架的三个组成部分

**SiT 将生成建模分成三个相对独立的部分：**

1. **Interpolation：** 定义数据与噪声之间的概率路径。
2. **Learning：** 通过 Velocity Matching 学习路径对应的速度场。
3. **Sampling：** 利用 ODE 或 SDE 将高斯噪声转换为真实数据。

这三部分通过连续性方程联系起来，下面给出推导。

## 2. 数学推导

### 2.1 连续性方程（Continuity Equation）

假设大量随机初始化的粒子 $X_0\sim p_0(x)$ 遵循相同的速度场运动：

$$
\boxed{\frac{dX_t}{dt}=v(X_t,t)}
$$

其中 $X_t$ 表示粒子在 $t$ 时刻的位置，$v(x,t)$ 表示位置 $x$、时刻 $t$ 对应的速度。

这些粒子在每个时刻形成概率分布 $X_t\sim p_t(x)$。我们希望研究 **粒子的运动如何影响整体概率分布**。

#### 2.1.1 从概率守恒推导

考虑一维区间 $[a,b]$，落在该区间内的总概率为：

$$
P_t([a,b])=\int_a^b p_t(x)\,dx
$$

定义概率流量（Probability Flux）：

$$
J(x,t)=p_t(x)v(x,t)
$$

它表示单位时间内通过位置 $x$ 的有符号概率流量。

根据概率守恒，区间内概率的变化等于流入减去流出：

$$
\frac{d}{dt}\int_a^b p_t(x)\,dx
=J(a,t)-J(b,t)
$$

将右边改写为积分：

$$
\int_a^b\frac{\partial p_t}{\partial t}\,dx
=
-\int_a^b\frac{\partial J}{\partial x}\,dx
$$

由于区间 $[a,b]$ 任意：

$$
\boxed{
\frac{\partial p_t}{\partial t}
=-\frac{\partial}{\partial x}(p_tv)
}
$$

推广到高维空间，得到连续性方程：

$$
\boxed{
\frac{\partial p_t}{\partial t}
=-\nabla\cdot(p_tv)
}
$$

其中 $\nabla\cdot$ 表示散度（Divergence）。

将右边展开：

$$
\frac{\partial p_t}{\partial t}
=-v\cdot\nabla p_t-p_t\nabla\cdot v
$$

- $v\cdot\nabla p_t$：描述概率密度随粒子运动发生的空间变化。
- $p_t\nabla\cdot v$：描述局部扩张或压缩对概率密度的影响。

例如，$\nabla\cdot v>0$ 表示局部速度场向外扩张，具有降低局部密度的趋势；反之则具有提高局部密度的趋势。

**连续性方程的本质：确定性粒子流通过概率的搬运与重新分布，使整体概率密度随时间演化。**

#### 2.1.2 为什么随机插值满足连续性方程？

SiT 定义随机插值：

$$
x_t=\alpha_t x_*+\sigma_t\epsilon
$$

其轨迹速度为：

$$
\dot x_t=\dot\alpha_t x_*+\dot\sigma_t\epsilon
$$

定义条件平均速度场：

$$
\boxed{
v(x,t)=\mathbb E[\dot x_t\mid x_t=x]
}
$$

下面证明该速度场能够描述随机插值的边缘概率分布演化。

任取光滑测试函数 $\phi(x)$：

$$
\mathbb E[\phi(x_t)]
=\int\phi(x)p_t(x)\,dx
$$

对时间求导，并利用条件期望：

$$
\begin{aligned}
\frac{d}{dt}\mathbb E[\phi(x_t)]
&=\mathbb E[\nabla\phi(x_t)\cdot\dot x_t]\\
&=\mathbb E\left[
\nabla\phi(x_t)\cdot
\mathbb E[\dot x_t\mid x_t]
\right]\\
&=\int\nabla\phi(x)\cdot v(x,t)p_t(x)\,dx
\end{aligned}
$$

利用分部积分，假设边界项消失：

$$
=-\int\phi(x)\nabla\cdot(p_tv)\,dx
$$

另一方面：

$$
\frac{d}{dt}\mathbb E[\phi(x_t)]
=\int\phi(x)\frac{\partial p_t}{\partial t}\,dx
$$

比较两边：

$$
\int\phi(x)
\left[
\frac{\partial p_t}{\partial t}
+\nabla\cdot(p_tv)
\right]dx=0
$$

由于 $\phi$ 任意，得到：

$$
\boxed{
\frac{\partial p_t}{\partial t}
=-\nabla\cdot(p_tv)
}
$$

**结论：随机插值形成的边缘分布 $p_t$，满足以条件平均速度 $v$ 为速度场的连续性方程。**

因此，在适当正则性和唯一性条件下，可以通过确定性的 ODE：

$$
\boxed{dX_t=v(X_t,t)dt}
$$

生成与随机插值相同的边缘分布演化，但不要求单条粒子轨迹相同。

---

### 2.2 Fokker–Planck 方程

连续性方程描述确定性粒子运动对概率分布的影响。如果粒子在运动过程中还受到随机扰动，就需要使用随机微分方程（SDE）。

#### 2.2.1 从 ODE 到 SDE

一般形式：

$$
\boxed{
dX_t=f(X_t,t)dt+g(t)dW_t
}
$$

其中：

- $f(x,t)$：漂移（Drift），描述确定性的运动趋势。
- $g(t)$：扩散系数，控制随机扰动的强度。
- $W_t$：标准布朗运动（Wiener Process）。

布朗运动在正时间间隔 $\Delta t>0$ 内的增量满足：

$$
\Delta W_t\sim\mathcal N(0,\Delta t I)
$$

因此，SDE 的离散近似为：

$$
\boxed{
X_{t+\Delta t}
\approx X_t+f(X_t,t)\Delta t
+g(t)\sqrt{\Delta t}\epsilon
}
$$

其中 $\epsilon\sim\mathcal N(0,I)$。

直观上，**Drift 负责推动粒子运动，Diffusion 负责引入随机扩散。**

#### 2.2.2 为什么随机扩散对应二阶导数？

先考虑一维纯随机运动：

$$
dX_t=g\,dW_t
$$

为理解其概率演化，可以使用对称随机游走近似：每个时间步以相等概率向左或向右移动距离 $h$，其中：

$$
h=g\sqrt{\Delta t}
$$

位置 $x$ 的新概率密度由两侧粒子贡献：

$$
p_{t+\Delta t}(x)
=\frac12p_t(x-h)+\frac12p_t(x+h)
$$

对两项进行泰勒展开：

$$
p_t(x\pm h)
=p_t(x)\pm hp_t'(x)
+\frac{h^2}{2}p_t''(x)+O(h^3)
$$

代入后，一阶导数项抵消：

$$
p_{t+\Delta t}(x)-p_t(x)
=\frac{h^2}{2}p_t''(x)+O(h^4)
$$

由于 $h^2=g^2\Delta t$，两边除以 $\Delta t$ 并取极限：

$$
\boxed{
\frac{\partial p_t}{\partial t}
=\frac12g^2\frac{\partial^2p_t}{\partial x^2}
}
$$

这就是扩散方程（Diffusion Equation）。

二阶导数反映概率密度的局部曲率：随机扩散通常会降低密度峰值，并使分布逐渐变宽。

#### 2.2.3 Fokker–Planck 方程

对于一般 SDE：

$$
dX_t=f(X_t,t)dt+g(t)dW_t
$$

其概率分布同时受到漂移和随机扩散影响：

$$
\boxed{
\frac{\partial p_t}{\partial t}
=
-\nabla\cdot(p_tf)
+\frac12g^2(t)\Delta p_t
}
$$

这就是 Fokker–Planck 方程（也称 Kolmogorov Forward Equation）。

其中：

$$
\Delta p_t
=\sum_i\frac{\partial^2p_t}{\partial x_i^2}
$$

为拉普拉斯算子（Laplacian）。

两部分分别对应：

$$
\underbrace{-\nabla\cdot(p_tf)}_{\text{确定性漂移}}
+
\underbrace{\frac12g^2(t)\Delta p_t}_{\text{随机扩散}}
$$

**连续性方程描述确定性运动造成的分布变化，而 Fokker–Planck 方程进一步考虑了布朗运动带来的随机扩散。**

以上推导使用各向同性、扩散系数仅依赖时间的情形；更一般的状态依赖扩散需要使用扩散矩阵。

---

### 2.3 Probability Flow ODE

现在考虑一个问题：

**能否构造确定性 ODE，使它与随机 SDE 具有相同的边缘概率分布？**

连续性方程：

$$
\frac{\partial p_t}{\partial t}
=-\nabla\cdot(p_tv)
\tag{1}
$$

Fokker–Planck 方程：

$$
\frac{\partial p_t}{\partial t}
=-\nabla\cdot(p_tf)+\frac12g^2\Delta p_t
\tag{2}
$$

希望找到一个速度场 $v$，使两个方程描述相同的分布演化。

#### 2.3.1 引入 Score

Score 定义为：

$$
\boxed{s(x,t)=\nabla_x\log p_t(x)}
$$

利用对数求导：

$$
\nabla p_t=p_t\nabla\log p_t=p_ts
$$

因此：

$$
\Delta p_t
=\nabla\cdot(\nabla p_t)
=\nabla\cdot(p_ts)
$$

将其代入 Fokker–Planck 方程：

$$
\begin{aligned}
\partial_t p_t
&=-\nabla\cdot(p_tf)
+\frac12g^2\nabla\cdot(p_ts)\\
&=-\nabla\cdot\left[
p_t\left(f-\frac12g^2s\right)
\right]
\end{aligned}
$$

与连续性方程比较：

$$
\partial_t p_t=-\nabla\cdot(p_tv)
$$

可以选择：

$$
\boxed{
v(x,t)=f(x,t)-\frac12g^2(t)s(x,t)
}
$$

于是得到 Probability Flow ODE：

$$
\boxed{
dX_t=
\left[
f(X_t,t)-\frac12g^2(t)s(X_t,t)
\right]dt
}
$$

**结论：通过 Score 修正漂移，可以把 SDE 的随机扩散效应等价地表示为确定性概率流。**

在相同初始分布和适当数学条件下，SDE 与对应的 Probability Flow ODE 具有相同的边缘分布 $p_t$，但单条运动轨迹不同。

#### 2.3.2 高斯例子

考虑纯布朗运动：

$$
dX_t=dW_t,\qquad X_0\sim\mathcal N(0,1)
$$

其边缘分布为：

$$
p_t=\mathcal N(0,1+t)
$$

对应 Score：

$$
s(x,t)=-\frac{x}{1+t}
$$

由于 $f=0,g=1$，Probability Flow 速度为：

$$
v(x,t)=-\frac12s(x,t)=\frac{x}{2(1+t)}
$$

因此 ODE：

$$
\frac{dX_t}{dt}=\frac{X_t}{2(1+t)}
$$

解为：

$$
X_t=X_0\sqrt{1+t}
$$

所以：

$$
X_t\sim\mathcal N(0,1+t)
$$

与原始布朗运动得到的分布完全相同。

区别在于：

- SDE 通过持续加入随机噪声，使分布变宽。
- ODE 通过确定性地向外推动粒子，使分布变宽。

**Probability Flow ODE 的本质不是复现随机轨迹，而是复现随机过程的边缘分布演化。**

---

### 2.4 SiT 的逆时 SDE

SiT 与传统 Score-based Diffusion 的一个重要区别在于构造顺序：

- 传统 Diffusion 通常先定义前向 SDE，再推导 Probability Flow ODE。
- SiT 先通过随机插值学习速度场 $v$，再构造具有相同边缘分布的 SDE。

#### 2.4.1 构造前向 SDE

SiT 已经通过随机插值得到：

$$
\partial_t p_t=-\nabla\cdot(p_tv)
$$

我们希望引入随机扩散，同时保持原来的 $p_t$ 不变。

设：

$$
dX_t=f_+(X_t,t)dt+\sqrt{w_t}dW_t
$$

其中 $w_t\geq0$ 是可选择的扩散强度。

根据 Probability Flow ODE 的关系：

$$
v=f_+-\frac12w_ts
$$

移项得到前向漂移：

$$
\boxed{
f_+=v+\frac12w_ts
}
$$

因此：

$$
\boxed{
dX_t=
\left(v+\frac12w_ts\right)dt
+\sqrt{w_t}dW_t
}
$$

这个前向 SDE 与原始随机插值具有相同的边缘分布路径。

#### 2.4.2 构造逆时 SDE

生成时需要从噪声 $t=1$ 走向数据 $t=0$。

为了避免逆向时间的符号问题，定义：

$$
\tau=1-t,\qquad q_\tau(x)=p_{1-\tau}(x)
$$

由连续性方程及链式法则：

$$
\boxed{
\partial_\tau q_\tau=\nabla\cdot(q_\tau v)
}
\tag{3}
$$

假设反向生成过程满足：

$$
dY_\tau=b(Y_\tau,\tau)d\tau+\sqrt{w_t}dW_\tau
$$

其 Fokker–Planck 方程为：

$$
\partial_\tau q_\tau
=-\nabla\cdot(q_\tau b)
+\frac12w_t\Delta q_\tau
$$

利用 $q_\tau s=\nabla q_\tau$：

$$
\partial_\tau q_\tau
=-\nabla\cdot\left[
q_\tau\left(b-\frac12w_ts\right)
\right]
$$

为了使它与目标方程 (3) 一致，可以令：

$$
b-\frac12w_ts=-v
$$

得到：

$$
\boxed{
b=-v+\frac12w_ts
}
$$

这是新时间变量 $\tau$ 下的正向漂移。

改回原时间变量 $t$，由于 $d\tau=-dt$，漂移项符号反转，得到 SiT 的逆时 SDE：

$$
\boxed{
dX_t=
\left[
v(X_t,t)-\frac12w_ts(X_t,t)
\right]dt
+\sqrt{w_t}\,d\bar W_t
}
$$

其中 $\bar W_t$ 表示逆时间布朗运动，该方程沿 $t=1\rightarrow0$ 求解。上面的 $\tau$ 表达式中，$v$、$s$、$w$ 均在原时间 $t=1-\tau$ 处取值；不能将 $dt<0$ 直接代入正时间布朗增量的方差公式。

使用 Euler–Maruyama 离散化，令步长 $h>0$：

$$
\boxed{
X_{t-h}
\approx X_t-hv(X_t,t)
+\frac12hw_ts(X_t,t)
+\sqrt{hw_t}\epsilon
}
$$

其中 $\epsilon\sim\mathcal N(0,I)$。

可以理解为三部分：

- $-hv$：沿原概率流的反方向移动。
- $+\frac12hw_ts$：利用 Score 调整漂移。
- $\sqrt{hw_t}\epsilon$：引入随机扰动。

Score 修正项与随机扩散项共同作用，使整体分布遵循目标的逆向概率路径。

#### 2.4.3 为什么扩散强度可以自由选择？

将逆时生成的漂移 $b=-v+\frac12w_ts$ 代入 Fokker–Planck 方程：

$$
\begin{aligned}
\partial_\tau q_\tau
&=\nabla\cdot(q_\tau v)
-\frac12w_t\nabla\cdot(q_\tau s)
+\frac12w_t\Delta q_\tau\\
&=\nabla\cdot(q_\tau v)
-\frac12w_t\Delta q_\tau
+\frac12w_t\Delta q_\tau\\
&=\nabla\cdot(q_\tau v)
\end{aligned}
$$

可以看到，**Score 修正项与随机扩散项对概率密度的额外影响恰好抵消**。

而这个抵消关系与 $w_t$ 的具体取值无关。

因此，在理论上，SiT 可以选择不同的非负扩散强度：

$$
\begin{cases}
w_t=0 &\Rightarrow \text{确定性 ODE}\\
w_t>0 &\Rightarrow \text{随机 SDE}
\end{cases}
$$

这使插值路径 $(\alpha_t,\sigma_t)$ 与采样扩散强度 $w_t$ 可以相对独立地设计。

需要注意，理论等价依赖准确的 Velocity、Score 及适当的数学条件；实际采样时，模型预测误差和数值离散误差可能导致不同 $w_t$ 的生成效果不同。

---

### 2.5 动力学公式速查

**三个基础方程：**

连续性方程：

$$
\boxed{
\partial_t p_t=-\nabla\cdot(p_tv)
}
$$

Fokker–Planck 方程：

$$
\boxed{
\partial_t p_t
=-\nabla\cdot(p_tf)+\frac12g^2\Delta p_t
}
$$

Probability Flow ODE 的速度关系：

$$
\boxed{
v=f-\frac12g^2s
}
$$

**SiT 的三种生成动力学：**

| 过程 | 漂移或速度 | 方向 |
|---|---|---|
| Probability Flow ODE | $v$ | 双向均可 |
| Forward SDE | $v+\frac12ws$ | 数据 → 噪声 |
| Reverse-time SDE | $v-\frac12ws$ | 噪声 → 数据 |

其中 $f_-$ 是原时间 $t$ 递减时的漂移系数；若改用递增的 $\tau=1-t$，漂移为 $-v+\frac12ws$。

$$
\boxed{
f_+=v+\frac12ws,\qquad
f_-=v-\frac12ws
}
$$

因此：

$$
v=\frac{f_++f_-}{2}
$$

## 3. 模型设计与实验分析

SiT 沿用 DiT 的 Transformer Backbone（包括 VAE Latent Space、Patch Embedding 和 adaLN-Zero），主要改进并非网络结构，而是**基于随机插值框架，重新设计生成模型的训练目标、概率路径与采样过程**。

论文系统研究了四个设计维度：

1. Time Discretization：离散时间与连续时间。
2. Model Prediction：Score Prediction 与 Velocity Prediction。
3. Interpolant Design：不同的数据—噪声插值路径。
4. Sampler Design：ODE、SDE 以及扩散系数的选择。

本节实验来自 [SiT 原论文](https://arxiv.org/html/2401.08740v2) 的表 1—7。统一条件如下：

| 实验范围 | 条件 |
|---|---|
| 3.1—3.5 过渡消融 | ImageNet 256×256，B 规模，latent patch 大小 2，训练 400K 步，不使用 CFG |
| 采样预算 | ODE 使用二阶 Heun，SDE 使用 Euler–Maruyama，预算均为 250 NFE |
| 评价指标 | FID-50K，越低越好，以 ImageNet 训练集为参考 |
| 3.6 规模比较 | 模型 S / B / L / XL，patch 大小均为 2；训练步数和 CFG 单独标注 |

NFE 是网络评估次数，不等同于积分步数；例如标准 Heun 一步通常需要两次评估。相同 NFE 也不必然意味着完全相同的实际耗时。

### 3.1 连续时间训练（Time Discretization）

传统 DDPM 使用离散时间步，而 SiT 采用连续时间：

$$
t\sim U(0,1)
$$

连续时间训练可以将训练时的时间表示与推理时的数值离散化解耦，使模型在训练完成后灵活选择采样步数和求解器。

实验（论文表 2，FID）：

| 方法 | FID ↓ |
|---|---:|
| DDPM（Discrete） | 44.2 |
| SBDM-VP（Continuous） | 43.6 |

**结论：连续时间本身带来的性能提升有限，主要优势是提高采样策略的灵活性。**

### 3.2 预测目标（Model Prediction）

SiT 比较了 Score Matching、加权 Score Matching 和 Velocity Matching。

Score Matching：

$$
\mathcal L_s=
\mathbb E\left[
\|\sigma_t s_\theta(x_t,t)+\epsilon\|^2
\right]
$$

加权 Score Matching（论文附录 B 中与 Velocity Loss 对应的权重）：

$$
\lambda_t=\dot\sigma_t-\frac{\dot\alpha_t\sigma_t}{\alpha_t}.
$$


$$
\mathcal L_s^\lambda=
\mathbb E\left[
\lambda_t^2\|\sigma_t s_\theta(x_t,t)+\epsilon\|^2
\right]
$$

Velocity Matching：

$$
\mathcal L_v=
\mathbb E\left[
\|v_\theta(x_t,t)-\dot x_t\|^2
\right]
$$

这里的期望还包括 $t\sim U(0,1)$。若按

$$
v_\theta(x,t)=\frac{\dot\alpha_t}{\alpha_t}x-\lambda_t\sigma_t s_\theta(x,t)
$$

转换参数化，加权 Score Loss 与 Velocity Loss 在代数上相等。但直接用网络输出 score 或 velocity，会改变参数化与优化的数值性质；理论目标之间的转换不保证训练结果相同。

实验（论文表 3，SBDM-VP 插值，ODE 采样）：

| 预测目标 | FID ↓ |
|---|---:|
| Score | 43.6 |
| Weighted Score | **39.1** |
| Velocity | 39.8 |

普通 Score Loss 在某些时间区域可能存在训练信号不平衡的问题，加权损失能够调整不同时间的贡献。

**结论：Velocity Prediction 并非始终优于 Score Prediction。预测目标的数值性质与时间权重同样重要，应当结合插值路径进行设计。**

### 3.3 插值路径（Interpolant Design）

SiT 比较三种插值：

**SBDM-VP：**

$$
\alpha_t=e^{-\frac12\int_0^t\beta(s)ds},
\qquad
\sigma_t=\sqrt{1-\alpha_t^2}
$$

**Linear：**

$$
\alpha_t=1-t,\qquad \sigma_t=t
$$

**GVP：**

$$
\alpha_t=\cos(\pi t/2),\qquad
\sigma_t=\sin(\pi t/2)
$$

保持 Velocity Prediction 不变（论文表 4，ODE 采样）：

| 插值路径 | FID ↓ |
|---|---:|
| SBDM-VP | 39.8 |
| Linear | 34.8 |
| GVP | **34.6** |

论文给出两个可能的原因：

**1. 传输代价（Transport Cost）**

定义：

$$
\boxed{
\mathcal C(v)=
\int_0^1\mathbb E[\|v(x_t,t)\|^2]dt
}
$$

该指标衡量概率流的平均平方速度积分。

实验发现：

$$
\mathcal C_{\mathrm{GVP}}
<
\mathcal C_{\mathrm{Linear}}
<
\mathcal C_{\mathrm{SBDM-VP}}
$$

较低的传输代价与更好的生成质量相关，也可能意味着更容易进行数值积分。

需要注意，Linear Interpolant 并不意味着学到的 ODE 轨迹必然是直线。

**2. 数值稳定性**

对于 SBDM-VP：

$$
\dot\sigma_t=
\frac{\beta(t)e^{-\int_0^t\beta(s)ds}}{2\sigma_t}
$$

当 $t\rightarrow0$ 时，可能产生奇异性，影响训练稳定性。

而 Linear 和 GVP 的 $\dot\sigma_t$ 都不存在这种端点奇异性。

此外，Linear 和 GVP 能够在 $t=1$ 精确到达标准高斯噪声，而有限时间的常规 VP-SDE 通常存在终点分布偏差。

**结论：插值路径不仅决定数据与噪声的混合方式，还直接影响速度场的复杂程度、数值稳定性和生成质量。**

### 3.4 ODE 与 SDE 采样

SiT 允许同一个训练好的模型使用两种采样过程：

**Probability Flow ODE：**

$$
dX_t=v_\theta(X_t,t)dt
$$

**Reverse-time SDE：**

$$
dX_t=
\left(v_\theta-\frac12w_ts_\theta\right)dt
+\sqrt{w_t}d\bar W_t
$$

理论上，精确的 ODE 与 SDE 具有相同的边缘分布，但实际模型存在预测误差与数值积分误差。

实验（论文表 5，SDE 使用 $w_t^{KL}$）：

| 插值路径 | ODE FID ↓ | SDE FID ↓ |
|---|---:|---:|
| SBDM-VP | 39.8 | 37.8 |
| Linear | 34.8 | 33.6 |
| GVP | 34.6 | **32.9** |

论文发现：

- **ODE：** 通常能以较少的 NFE 快速获得较好的生成质量。
- **SDE：** 在较充足的计算预算下，可以取得更低的最终 FID。

其中 NFE（Number of Function Evaluations）表示采样过程中神经网络被调用的次数。

SDE 的理论优势之一，是可以利用训练误差建立生成分布 KL 散度的上界；但这并不意味着 SDE 在所有场景下必然优于 ODE。

### 3.5 扩散系数（Diffusion Coefficient）

SiT 的一个重要特点是：**插值路径与采样扩散强度可以解耦。**

插值系数 $\alpha_t,\sigma_t$ 需要在训练前确定，而扩散系数 $w_t$ 可以在训练后调整，无须重新训练模型。

论文比较了：

$$
w_t=\sigma_t
$$

$$
w_t=\sin^2(\pi t)
$$

以及通过优化 KL 散度上界得到的：

$$
\boxed{
w_t^{KL}=
2\left(
\dot\sigma_t\sigma_t
-\frac{\dot\alpha_t\sigma_t^2}{\alpha_t}
\right)
}
$$

代入各路径可得：

$$
\begin{aligned}
\text{SBDM-VP:}\quad&w_t^{KL}=\beta(t),\\
\text{Linear:}\quad&w_t^{KL}=\frac{2t}{1-t},\\
\text{GVP:}\quad&w_t^{KL}=\pi\tan(\pi t/2).
\end{aligned}
$$

因此 Linear 和 GVP 的 $w_t^{KL}$ 在 $t\rightarrow1$ 时发散。它在开区间上定义明确，但从精确端点启动数值求解时需要截断或正则化。

因此，论文进一步设计了考虑积分成本的正则化版本：

$$
w_t^{KL,\eta}
=w_t^{KL}\sqrt{\frac{\mathcal L_t}
{\mathcal L_t+2\eta(w_t^{KL})^2}},\qquad \eta\geq0.
$$

其中 $\mathcal L_t=\mathbb E[\|v_\theta(x_t,t)-\dot x_t\|^2]$ 是固定时间的训练损失，通过样本估计；$\eta$ 控制正则化强度。$\eta=0$ 时恢复 $w_t^{KL}$，$\eta>0$ 可限制噪声端的扩散强度。定义见论文公式（14），KL 上界及其假设见附录 A.5；这一选择优化的是带假设的上界，并不保证 FID 最优。

实验表明，扩散系数的最优选择取决于插值路径、预测目标和模型规模。例如（论文表 6）：

- SBDM-VP + Velocity：$w_t^{KL}$ 表现较好。
- GVP + Velocity：$w_t^{KL}$ 表现较好。
- Linear + Velocity：正则化的 $w_t^{KL,\eta}$ 表现更好，FID 达到 **33.0**。

**结论：Sampler 不应被视为固定的训练附属组件，而是可以在训练完成后独立优化的设计维度。**

### 3.6 最终性能对比

**论文表 1：ImageNet 256×256，训练 400K 步，不使用 CFG，patch 大小为 2，SDE 采样：**

| 模型规模 | DiT FID ↓ | SiT FID ↓ |
|---|---:|---:|
| Small | 68.4 | 57.6 |
| Base | 43.5 | 33.0 |
| Large | 23.3 | 18.8 |
| XL | 19.5 | 17.2 |

在相同模型结构、参数规模和训练计算预算下，SiT 在所有规模上均优于 DiT。

进一步结合 Classifier-free Guidance（CFG）：

$$
v_{\mathrm{CFG}}
=
v_{\mathrm{uncond}}
+
\zeta(v_{\mathrm{cond}}-v_{\mathrm{uncond}})
$$

其中 $\zeta$ 为 Guidance Scale。

最终实验（论文表 1、表 7，XL/2，CFG = 1.5）：

| 图像分辨率 | DiT-XL FID ↓ | SiT-XL FID ↓ |
|---|---:|---:|
| 256×256 | 2.27 | **2.06** |
| 512×512 | 3.04 | **2.62** |

其中 256×256 使用 7M 训练步，512×512 使用 3M 训练步。

**SiT 的性能提升主要来自生成动力学与训练、采样策略的优化，而不是引入更复杂的 Transformer Backbone。**

### 3.7 复习要点

SiT 的贡献可以概括为三个方面：

1. **Interpolation Design：** 不同概率路径对应不同的速度场，直接影响学习难度与数值求解效率。
2. **Objective Design：** Velocity 和 Score 可以相互转换，但不同预测目标及损失权重会影响优化效果。
3. **Sampler Design：** ODE、SDE 及其扩散系数可以在训练后灵活选择，进一步改善生成效率和质量。

这些实验说明，在本文比较的模型与计算预算下，概率路径、学习目标和采样策略的选择，能够在保持 Backbone 不变时改善生成质量。


## 附录 A：从 DDPM 到连续时间 SDE

### A.1 正向 SDE（Forward SDE）

**核心目标：证明 DDPM 的离散前向扩散可以在连续时间下表示为 VP-SDE。**

#### A.1.1 离散扩散连续化

DDPM 的前向扩散：

$$
x_{k+1}=\sqrt{1-\beta_k}x_k+\sqrt{\beta_k}\epsilon_k
$$

引入连续时间 $t\in[0,1]$，令：

$$
\beta_k=\beta(t)\Delta t
$$

代入并对平方根进行一阶泰勒展开：

$$
\Delta x_t\approx
-\frac12\beta(t)x_t\Delta t
+\sqrt{\beta(t)}\sqrt{\Delta t}\epsilon
$$

由于维纳过程满足：

$$
\Delta W_t\sim\mathcal N(0,\Delta t I)
$$

因此可以使用 $\Delta W_t=\sqrt{\Delta t}\epsilon$ 构造离散维纳增量。

当 $\Delta t\to0$ 时，得到：

$$
\boxed{
dX_t=-\frac12\beta(t)X_tdt+\sqrt{\beta(t)}dW_t
}
$$

这就是 VP-SDE。其中：

- Drift：$-\frac12\beta(t)X_t$，控制信号衰减。
- Diffusion：$\sqrt{\beta(t)}$，控制随机扰动强度。

#### A.1.2 VP-SDE 与 DDPM 的对应关系

使用积分因子法求解：

$$
X_t=\alpha_tX_0+
\alpha_t\int_0^t\frac{\sqrt{\beta(s)}}{\alpha_s}dW_s
$$

其中：

$$
\alpha_t=\exp\left(-\frac12\int_0^t\beta(s)ds\right)
$$

随机积分项服从零均值高斯分布，其协方差为：

$$
\left(1-e^{-\int_0^t\beta(s)ds}\right)I
$$

定义：

$$
\sigma_t=\sqrt{1-e^{-\int_0^t\beta(s)ds}}
$$

得到：

$$
\boxed{
X_t\mid X_0=x_0\sim
\mathcal N(\alpha_t x_0,\sigma_t^2I)
}
$$

因此，在固定时间步的条件分布意义下：

$$
\boxed{
X_t\overset d=\alpha_tX_0+\sigma_t\epsilon
}
$$

且满足：

$$
\alpha_t^2+\sigma_t^2=1
$$

**结论：VP-SDE 与 DDPM 的方差保持加噪过程具有一致的连续时间描述。**

---

### A.2 逆时 SDE（Reverse-Time SDE）

**核心目标：推导为什么反向扩散需要 Score，并将其与 DDPM 噪声预测联系起来。**

#### A.2.1 从正向转移到反向转移

考虑一般正时 SDE：

$$
dX_t=f(X_t,t)dt+g(t)dW_t
$$

在很小的时间间隔 $h>0$ 内：

$$
X_{t+h}\approx X_t+f(X_t,t)h+g(t)\sqrt h\,\epsilon
$$

因此，正向转移近似服从高斯分布：

$$
p(y\mid x)\approx
\mathcal N(y;x+f(x,t)h,g^2(t)hI)
$$

根据贝叶斯公式：

$$
\boxed{
p(x\mid y)\propto p(y\mid x)p_t(x)
}
$$

**关键观察：反向分布不仅取决于正向转移概率，还取决于当前数据的边缘概率密度 $p_t(x)$。**

#### A.2.2 Score 为什么会出现？

定义 Score Function：

$$
\boxed{
s(x,t)=\nabla_x\log p_t(x)
}
$$

对 $\log p_t(x)$ 在 $y$ 处进行一阶泰勒展开：

$$
\log p_t(x)\approx
\log p_t(y)+s(y,t)\cdot(x-y)
$$

将其代入贝叶斯公式，对反向条件概率的高斯指数部分配方。

令 $\delta=x-y$，在小时间步近似下可以得到：

$$
\boxed{
\delta\mid y\approx
\mathcal N(
[-f(y,t)+g^2(t)s(y,t)]h,\,
g^2(t)hI
)
}
$$

于是反向更新为：

$$
X_t\approx X_{t+h}
+[-f+g^2s]h+g\sqrt h\,\epsilon
$$

沿用原始时间变量，从大时间向小时间积分：

$$
\boxed{
dX_t=[f(X_t,t)-g^2(t)s(X_t,t)]dt
+g(t)d\bar W_t
}
$$

其中逆向积分时 $dt<0$。

**核心结论：反向漂移不仅要逆转原始漂移，还必须通过 Score 修正，以考虑真实数据分布的影响。**

#### A.2.3 与 DDPM 噪声预测的联系

对于 VP-SDE：

$$
f(x,t)=-\frac12\beta(t)x,\qquad
g^2(t)=\beta(t)
$$

代入逆时 SDE：

$$
\boxed{
dX_t=
[-\frac12\beta(t)X_t-\beta(t)s(X_t,t)]dt
+\sqrt{\beta(t)}d\bar W_t
}
$$

根据高斯加噪模型：

$$
X_t=\alpha_tX_0+\sigma_t\epsilon
$$

可以证明：

$$
\boxed{
s(x,t)=-\frac{\mathbb E[\epsilon\mid X_t=x]}{\sigma_t}
}
$$

由于 MSE 最优噪声预测为：

$$
\epsilon_\theta^*(x,t)=\mathbb E[\epsilon\mid X_t=x]
$$

因此：

$$
s_\theta(x,t)\approx-\frac{\epsilon_\theta(x,t)}{\sigma_t}
$$

代入得到：

$$
\boxed{
dX_t=
\left[
-\frac12\beta(t)X_t
+\frac{\beta(t)}{\sigma_t}\epsilon_\theta(X_t,t)
\right]dt
+\sqrt{\beta(t)}d\bar W_t
}
$$

其离散化与 DDPM 的反向采样公式具有一致的一阶漂移结构。

**结论：DDPM 的噪声预测实际上可以转化为 Score 估计，从而用于连续时间逆时 SDE 的采样。**

---

### A.3 从传统扩散到 SiT

传统 DDPM / VP-SDE 的基本路线：

$$
\text{前向 SDE}
\rightarrow(\alpha_t,\sigma_t)
\rightarrow\text{逆时 SDE}
$$

SiT 则允许直接定义随机插值：

$$
\boxed{
x_t=\alpha_t x_*+\sigma_t\epsilon
}
$$

通过插值构造概率路径，再学习 Score 或 Velocity Field，最终使用 SDE 或 ODE 生成数据。

因此，**SiT 的重要变化不是更换 Transformer 架构，而是让概率路径、学习目标和采样过程具有更高的设计自由度。**

正文已依次讨论：

1. 连续性方程与概率分布的演化。
2. Fokker–Planck 方程与 Probability Flow ODE。
3. SiT 为什么可以构造 $v-\frac12ws$ 的逆时漂移。


## Reference

- [SiT: Exploring Flow and Diffusion-based Generative Models with Scalable Interpolant Transformers](https://arxiv.org/abs/2401.08740)：本文按 v2 整理；第 2 节框架，表 1—7 实验，附录 A / B 推导。
- [SiT 官方代码](https://github.com/willisma/SiT)：训练、transport 与采样实现。
- [Score-Based Generative Modeling through Stochastic Differential Equations](https://arxiv.org/abs/2011.13456)：逆时 SDE 与 Probability Flow ODE 的背景。
- [DDPM 基础笔记](diffusion.md) 与 [DiT 架构笔记](DiT.md)。
