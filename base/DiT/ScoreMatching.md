---
marp: true
author: CaoJiayu
theme: gaia
math: katex
paginate: true
style: |
  section { font-size: 26px; }
  h2 { font-size: 34px; }
---

# Score Matching

1. Score Function & VF Conversion
2. Gaussian Marginal Score Function
3. Posterior Mean
4. From ODE to SDE
5. Fokker-Plank Equation
6. Score Matching Loss
7. Training

---

## Score Function and Vector Field

高斯概率路径：

$$
p_t(x|z)=\mathcal N(\alpha_tz,\beta_t^2I_d)
$$

Conditional Vector Field：

$$
u_t(x|z)=\frac{\dot\beta_t}{\beta_t}x+
\left(\dot\alpha_t-\frac{\dot\beta_t\alpha_t}{\beta_t}\right)z
$$

Conditional Score Function：

$$
s_t(x|z)=\nabla_x\log p_t(x|z)
=-\frac{x-\alpha_tz}{\beta_t^2}
$$

---

## Vector Field and Score Conversion

由于二者都是 $x,z$ 的线性组合，可以相互转换：

$$
u_t(x|z)=a_ts_t(x|z)+b_tx
$$

其中：

$$
a_t=\beta_t^2\frac{\dot\alpha_t}{\alpha_t}
-\dot\beta_t\beta_t,\quad
b_t=\frac{\dot\alpha_t}{\alpha_t}
$$

边缘化后仍然成立：

$$
\boxed{u_t(x)=a_ts_t(x)+b_tx}
$$

当 $a_t\neq0$ 时，二者可以相互恢复。

---

## Marginal Score Function

边缘概率路径：

$$
p_t(x)=\int p_t(x|z)p_{\mathrm{data}}(z)\,dz
$$

利用 Marginalization Trick：

$$
s_t(x)=\int s_t(x|z)p_t(z|x)\,dz
$$

对于高斯条件路径：

$$
\boxed{
s_t(x)=-\frac{x-\alpha_t\mathbb E[z|x]}{\beta_t^2}
}
$$

条件 Score 关于 $x,z$ 线性；边缘 Score 通常不关于 $x$ 线性。

---

## Posterior Mean

定义后验均值（Marginal Denoiser）：

$$
D_t(x)=\mathbb E[z|x]
$$

Marginal Score：

$$
s_t(x)=\frac{\alpha_tD_t(x)-x}{\beta_t^2}
$$

Marginal Vector Field：

$$
u_t(x)=\frac{\dot\beta_t}{\beta_t}x+
\left(\dot\alpha_t-\frac{\dot\beta_t\alpha_t}{\beta_t}\right)D_t(x)
$$

**结论：** Score 和 VF 都是后验均值的仿射变换。

---

## From ODE to SDE

Flow ODE：

$$
dX_t=u_t(X_t)\,dt
$$

引入布朗运动 $W_t$ 和扩散系数 $\sigma_t\geq0$：

$$
\boxed{
dX_t=\left[u_t(X_t)+
\frac{\sigma_t^2}{2}s_t(X_t)\right]dt
+\sigma_t\,dW_t
}
$$

Score 项用于补偿随机扩散对概率分布的影响。

**结论：** ODE 与该 SDE 具有相同的边缘概率路径 $p_t$。

---

## Fokker–Planck Equation

对于上述 SDE，Fokker–Planck 方程为：

$$
\frac{\partial p_t}{\partial t}
=-\nabla\cdot\left[p_t
\left(u_t+\frac{\sigma_t^2}{2}s_t\right)\right]
+\frac{\sigma_t^2}{2}\Delta p_t
$$

根据 Score 的定义：

$$
p_ts_t=p_t\nabla\log p_t=\nabla p_t
$$

因此：

$$
\begin{aligned}
\frac{\partial p_t}{\partial t}
&=-\nabla\cdot(p_tu_t)
-\frac{\sigma_t^2}{2}\Delta p_t
+\frac{\sigma_t^2}{2}\Delta p_t\\
&=\boxed{-\nabla\cdot(p_tu_t)}
\end{aligned}
$$

**恢复连续性方程，证明边缘概率路径相同。**

---

## Score Matching Loss

直接学习 Marginal Score：

$$
\mathcal L_{\mathrm{SM}}(\theta)=
\mathbb E_{t,x}\left[
\|s_t^\theta(x)-s_t(x)\|^2
\right]
$$

使用 Conditional Score 作为监督信号：

$$
\mathcal L_{\mathrm{CSM}}(\theta)=
\mathbb E_{t,z,x|z}\left[
\|s_t^\theta(x)-s_t(x|z)\|^2
\right]
$$

利用 Marginalization Trick：

$$
\boxed{\mathcal L_{\mathrm{CSM}}=
\mathcal L_{\mathrm{SM}}+C}
$$

其中 $C$ 与 $\theta$ 无关，二者具有相同的最优解。

---

## Training: Denoising Score Matching

高斯采样：

$$
x_t=\alpha_tz+\beta_t\epsilon,\quad
\epsilon\sim\mathcal N(0,I_d)
$$

Conditional Score：

$$
s_t(x_t|z)=-\frac{\epsilon}{\beta_t}
$$

定义 Noise Predictor：$\epsilon_t^\theta(x)=-\beta_ts_t^\theta(x)$

则：

$$
\boxed{
\mathcal L_{\mathrm{CSM}}=
\mathbb E_{t,z,\epsilon}
\left[\frac{1}{\beta_t^2}
\|\epsilon_t^\theta(x_t)-\epsilon\|^2\right]
}
$$

**DDPM 常用无权重 Noise MSE，即对不同时间步重新加权。**
