---
marp: true
author: CaoJiayu
theme: gaia
math: katex
paginate: true
style: |
  section { font-size: 26px; }
  h1 { font-size: 52px; }
  h2 { font-size: 35px; }
  section p { margin: 0.45em 0; }
  section li { margin-bottom: 0.25em; }
---

# Flow Matching

1. Flow ODE & Continuity Equation
2. Conditional & Marginal Probability Paths
3. Conditional & Marginal Vector Fields
4. Flow Matching & Conditional FM Loss
5. Gaussian Probability Path & Target ODE
6. Training & Sampling

---

## Probability Path

给定初始噪声分布 $p_{\mathrm{init}}$ 和目标数据分布 $p_{\mathrm{data}}$。

希望构造一条随时间演化的概率路径 $p_t$，满足：

$$
p_0=p_{\mathrm{init}},\qquad p_1=p_{\mathrm{data}}
$$

**核心问题：** 如何构造这条概率路径，并找到驱动其演化的速度场？

---

## Flow ODE

$$
\frac{d}{dt}\phi_t(x_0)=u_t\bigl(\phi_t(x_0)\bigr),
\qquad \phi_0(x_0)=x_0
$$

- $u_t(x)$：位置 $x$、时刻 $t$ 的**瞬时速度场**，包含方向和大小。
- $\phi_t(x_0)$：从 $x_0$ 出发的粒子在时刻 $t$ 的位置。
- $\{\phi_t\}_{t\in[0,1]}$：由 ODE 的解构成的一族映射（Flow）。

若 $X_0\sim p_{\mathrm{init}}$，则 $X_t=\phi_t(X_0)$ 诱导出分布 $p_t$。

---

## Continuity Equation

粒子的运动与概率密度的变化必须相容，即满足**连续性方程**：

$$
\boxed{\frac{\partial p_t(x)}{\partial t}
=-\nabla_x\cdot\bigl(p_t(x)u_t(x)\bigr)}
$$

- $p_t(x)u_t(x)$：**概率流**，描述概率质量的流动。
- $\nabla_x\cdot(p_tu_t)$：概率流的散度，表示局部净流出。
- $\partial_t p_t(x)$：固定位置处的概率密度变化率。

**直观理解：** 某一区域流出的概率质量越多，其密度就下降得越快。

---

## Conditional & Marginal Probability Paths

先从数据分布采样 $Z$，再从对应的条件概率路径采样：

$$
Z\sim p_{\mathrm{data}},\qquad
X_t\mid Z=z\sim p_t(\cdot\mid z)
$$

得到边缘概率路径：

$$
\boxed{p_t(x)=\int p_t(x\mid z)p_{\mathrm{data}}(z)\,dz}
$$

**理解：** 每个样本 $z$ 对应一条条件概率路径；按数据分布混合后，得到整体的 $p_t$。

---

## Conditional & Marginal Vector Fields

边缘速度场是条件速度场的**后验加权平均**：

$$
\boxed{u_t(x)=\int u_t(x\mid z)
\frac{p_t(x\mid z)p_{\mathrm{data}}(z)}{p_t(x)}\,dz}
$$

等价地：

$$
u_t(x)=\mathbb E\bigl[u_t(x\mid Z)\mid X_t=x\bigr]
$$

权重 $p_t(z\mid x)$ 表示：已知 $X_t=x$，它来自样本 $z$ 的后验概率密度。

**困难：** 边缘分布与边缘速度场通常难以直接计算。

---

## Flow Matching & Conditional FM Loss

直接学习边缘速度场的 FM Loss：

$$
\mathcal L_{\mathrm{FM}}(\theta)=
\mathbb E_{t,X_t}\bigl[\|v_\theta(X_t,t)-u_t(X_t)\|^2\bigr]
$$

使用可计算的条件速度场构造 CFM Loss：

$$
\mathcal L_{\mathrm{CFM}}(\theta)=
\mathbb E_{t,Z,X_t\mid Z}
\bigl[\|v_\theta(X_t,t)-u_t(X_t\mid Z)\|^2\bigr]
$$

$$
\boxed{\mathcal L_{\mathrm{CFM}}=\mathcal L_{\mathrm{FM}}+C}
\quad(C\text{ 与 }\theta\text{ 无关})
$$

因此，两者具有**相同的最优解**，可以用 CFM 代替难以计算的 FM。

---

## Gaussian Probability Path

人为构造高斯条件概率路径：

$$
p_t(\cdot\mid z)=\mathcal N(\alpha_t z,\beta_t^2I_d)
$$

可以直接通过重参数化采样：

$$
\boxed{X_t=\alpha_t Z+\beta_t\epsilon},\qquad
Z\sim p_{\mathrm{data}},\ \epsilon\sim\mathcal N(0,I_d)
$$

- 固定 $z$：得到条件路径 $p_t(x\mid z)$；对 $z$ 边缘化：得到 $p_t(x)$。
- $\alpha_t,\beta_t$：人为设计的插值系数，控制数据与噪声的比例。

---

## Target ODE for Gaussian Paths

对插值表达式沿时间求导：

$$
\frac{dX_t}{dt}=\dot\alpha_t z+\dot\beta_t\epsilon
$$

由 $\epsilon=(X_t-\alpha_tz)/\beta_t$，得到**条件速度场**：

$$
\boxed{u_t(x\mid z)=\dot\alpha_tz+
\frac{\dot\beta_t}{\beta_t}(x-\alpha_tz)}
$$

因此，给定高斯条件路径，即可计算对应的条件速度场（$\beta_t>0$ 时）。

**关键作用：** 为 CFM 训练提供可计算的监督信号。

---

## Training

以线性插值 $\alpha_t=t,\ \beta_t=1-t$ 为例：

1. 采样 $Z\sim p_{\mathrm{data}}$、$\epsilon\sim\mathcal N(0,I_d)$。
2. 随机采样 $t\sim\mathcal U(0,1)$。
3. 构造 $X_t=(1-t)\epsilon+tZ$。
4. 条件速度目标为 $U_t=Z-\epsilon$。
5. 最小化 $\mathbb E\|v_\theta(X_t,t)-U_t\|^2$，更新参数。

**结果：** 虽然监督信号是条件速度，网络最优预测的是边缘速度场。

---

## Sampling

从 $X_0\sim p_{\mathrm{init}}$ 出发，用训练好的网络求解 Flow ODE。

以 **Euler Method** 为例，令 $\Delta t=1/N,\ t_k=k\Delta t$：

$$
\boxed{X_{t_{k+1}}=X_{t_k}+\Delta t\,v_\theta(X_{t_k},t_k)}
$$

不断迭代，最终得到生成样本 $X_1$。

理想情况下：

$$
p_1=(\phi_1)_\#p_{\mathrm{init}}=p_{\mathrm{data}}
$$

实际模型与数值求解存在误差，因此 $p_1^\theta\approx p_{\mathrm{data}}$。
