# SiT 前置：从 DDPM 到连续时间 SDE

## 1. 正时 SDE（Forward SDE）

**核心目标：证明 DDPM 的离散前向扩散可以在连续时间下表示为 VP-SDE。**

### 1.1 离散扩散连续化

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

### 1.2 VP-SDE 与 DDPM 的对应关系

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

## 2. 逆时 SDE（Reverse-Time SDE）

**核心目标：推导为什么反向扩散需要 Score，并将其与 DDPM 噪声预测联系起来。**

### 2.1 从正向转移到反向转移

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

### 2.2 Score 为什么会出现？

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

### 2.3 与 DDPM 噪声预测的联系

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

## 3. 从传统扩散到 SiT

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

后续需要研究：

1. 连续性方程与概率分布的演化。
2. Fokker–Planck 方程与 Probability Flow ODE。
3. SiT 为什么可以构造 $v-\frac12ws$ 的逆时漂移。

# SiT 的 Stochastic Interpolant 框架

TODO
