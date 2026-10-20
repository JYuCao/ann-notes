---
marp: true
author: CaoJiayu
theme: gaia
math: katex
---

# Flow Matching

1. Flow ODE, Continuity Equation
2. Conditional and Marginal Probability Path
3. Conditional and Marginal VF
4. Conditional and Marginal Loss
5. Gaussian Probability Path & Target ODE for Gaussian Path
6. Training & Sampling

---

## Probability Path

有一噪声分布 $p_{init}$，和一目标分布 $p_{data}$. 

我们希望将 $p_{init}$ 演化成 $p_{data}$.


**核心问题**：

如何如何构造一条连接二者的概率路径 $p_t$？

---

## Flow ODE
$$ \frac{d}{dt}\phi_t(x_0) = u_t(\phi_t(x_0)) $$
$$ \phi_0(x_0) = x_0 $$

- $u_t(x)$：描述位置 $x$、时刻 $t$ 的瞬时运动速度，包括方向和大小。
- $\phi_t(x_0)$：表示从 $x_0$ 出发，沿速度场运动的粒子在时刻 $t$ 的位置。
- $\{\phi_t\}$：由 ODE 解构成的一族映射，描述粒子随时间的演化。

---

## Conditional and Marginal Probability Path

$$ z\sim p_{data}, x\sim p_t(\cdot|data) \implies x\sim p_t$$
边缘概率分布：
$$ p_t(x) = \int p_t(x|z)p_{data}(z)dz $$

每个条件概率路径都以一个给定的数据样本 $z$ 为条件，所有条件概率路径按照数据分布进行加权平均，得到边缘概率路径。

---

## Conditional and Marginal VF

边缘向量场：
$$ u_t^{target}(x) = \int u_t^{target}(x|z)\frac{p_t(x|z)p_{data}(z)}{p_{t}(x)}dz $$
$$=\mathbb E_{Z\mid X_t=x}[u_t(x\mid Z)|X_t=x]$$

按照后验分布 $p_t(z\mid x)$ 对条件向量场进行加权。

**但这两个边缘积分都难以直接计算。**

---

## Continuity Equation

粒子的运动必须与概率密度变化相容。即满足**连续性方程**：

$$ \frac{\partial p_t}{\partial t} = -\nabla\cdot(p_tu_t) $$

* $p_t(x)u_t(x)$：概率流，描述概率质量的流动。
* $\nabla\cdot(p_tu_t)$：概率流的散度，衡量某点附近概率质量的净流出程度。
* $\frac{\partial p_t}{\partial t}$：该位置概率密度随时间的变化率。

连续性方程建立了速度场与概率分布演化之间的联系，表明局部概率密度的变化等于概率质量的净流入量。

---

## Conditional and Marginal Loss

使用网络学习一个边缘向量场 $v_\theta$，构造 FM Loss：
$$\mathcal L_{\mathrm{FM}}(\theta)
=
\mathbb E_{t,X_t}
\left[
\|v_\theta(X_t,t)-u_t(X_t)\|^2
\right]$$
$$\mathcal L_{\mathrm{CFM}}(\theta)
=
\mathbb E_{t,Z,X_t\mid Z}
\left[
\|v_\theta(X_t,t)-u_t(X_t\mid Z)\|^2
\right]$$

利用关系：

$$ L_{FM}(\theta) = L_{CFM}(\theta) + C $$

可以得到 $L_{FM}$ 的收敛点，即 $\arg\min_\theta\mathcal L_{\mathrm{CFM}}=\arg\min_\theta\mathcal L_{\mathrm{FM}}$.

---

## Gaussian Probability Path

人为构造高斯条件概率路径：

$$
p_t(\cdot\mid z)=\mathcal N(\alpha_t z,\beta_t^2 I_d)
$$

通过重参数化进行采样：

$$X_t=\alpha_t z+\beta_t\epsilon \quad z\sim p_{data}, \epsilon\sim\mathcal{N}(0,I_d)$$

- 固定 $z$ 可得到 $p_t(x\mid z)$；对 $z$ 边缘化可得到 $p_t(x)$。
- $\alpha_t,\beta_t$：人为设计的插值系数。

---

## Target ODE for Gaussian Paths

对插值过程求时间导数：

$$
\frac{dX_t}{dt}=\dot\alpha_t z+\dot\beta_t\epsilon
$$

利用：$\epsilon=\frac{X_t-\alpha_t z}{\beta_t}$ 得到 **条件速度场**：

$$
u_t(x\mid z)=
\dot\alpha_t z+
\frac{\dot\beta_t}{\beta_t}(x-\alpha_t z)
$$

因此，给定高斯条件路径，可以直接计算其速度场。


---

## Training

**训练过程**：

1. 从数据分布采样 $Z$，从噪声分布采样 $X_0$。
2. 随机采样 $t\sim\mathcal U(0,1)$。
3. 构造 $X_t=(1-t)X_0+tZ$（例子）。
4. 计算条件速度 $U_t=Z-X_0$。
5. 最小化 CFM Loss，更新网络参数。

最终网络拟合的是边缘速度场，和人为指定的条件速度场不同。

---

## Sampling

以 **Euler Method** 为例，将时间区间 $[0,1]$ 划分为 $N$ 步：
$\Delta t=\frac{1}{N},\quad t_k=k\Delta t$

根据网络预测的速度更新样本：

$$
X_{t_{k+1}}=X_{t_k}+\Delta t\,v_\theta(X_{t_k},t_k)
$$

重复迭代，最终得到生成样本 $X_1$。

理想情况下：$p_1=(\phi_1)_\#p_{init}=p_{data}$；实际生成分布 $p_1^\theta\approx p_{data}$。



