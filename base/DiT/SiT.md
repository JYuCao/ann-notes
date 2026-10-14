# SiT: Exploring Flow and Diffusion-based Generative Models with Scalable Interpolant Transformers

## 正时 SDE

### DDPM 前向扩散

DDPM 离散前向扩散：
$$
q(x_t | x_{t-1}) = 
\mathcal{N}(\sqrt{1-\beta_t}x_{t-1},\beta_t I)
$$

高斯重参数化：
$$
x_t =
\sqrt{1-\beta_t}x_{t-1}+\sqrt{\beta_t}\epsilon_t \quad \epsilon_t\sim\mathcal{N}(0,I) 
$$

### 离散时间连续化

总时间区间归一化到：$t\in[0, 1]$

每一离散时间步对应时间间隔：$\Delta t = \frac{1}{N}$，其中 $N$ 是总的离散时间步数。因此，$t_k = k \Delta t = \frac{k}{N}$。

当 $N\rightarrow\infty$ 时，离散时间步 $\Delta t \rightarrow 0$ 逼近连续时间过程。

#### 重新定义噪声调度

在连续时间框架下，噪声调度可以表示为一个函数 $\beta(t)$，它定义了在每个时间点添加的噪声强度。这种连续化的噪声调度使得模型能够更灵活地处理不同时间尺度上的变化。

假设 $\beta_t = \beta(t_k)\Delta t$。

>为什么要乘上 $\Delta t$？<br>
> 因为随着时间间隔越来越小，每个时间间隔内添加的噪声方差也应该越来越小，这样才能得到一个合理的连续随机过程。

代入前向公式得：
$$
x_{t+\Delta t} = \sqrt{1-\beta(t)\Delta t}x_t + \sqrt{\beta(t)\Delta t}\epsilon
$$

### 泰勒展开

对第一项进行泰勒展开：
$$
x_{t+\Delta t} 
\approx (1-\frac{1}{2}\beta(t)\Delta t)x_t + \sqrt{\beta(t)\Delta t}\epsilon_t \\
\approx x_t - \frac{1}{2}\beta(t)x_t\Delta t + \sqrt{\beta(t)\Delta t}\epsilon
$$

两边同时减去 $x_t$：
$$
\Delta x_t = x_{t+\Delta t} - x_t \approx -\frac{1}{2}\beta(t)x_t\Delta t + \sqrt{\beta(t)\Delta t}\epsilon
$$

### 引入维纳过程

维纳过程（Wiener process）重要性质：任意时间间隔内的增量服从高斯分布，其方差等于时间间隔。即：
$$
\Delta W_t \sim \mathcal{N}(0, \Delta tI)
$$
此外，它还满足 $W_0=0$、不相交时间区间上的增量相互独立，并且样本轨迹连续。

由于 $\sqrt{\Delta t}\epsilon \sim \mathcal{N}(0, \Delta tI)$，我们可以将 $\Delta x_t$ 表示为：

$$
\Delta x_t \approx -\frac{1}{2}\beta(t)x_t\Delta t + \sqrt{\beta(t)}\Delta W_t
$$

### 连续时间 SDE

当 $\Delta t \rightarrow 0$ 时，上述离散形式的 SDE 可以转换为连续时间的 SDE：

$$
\boxed{
dX_t = 
-\frac{1}{2}\beta(t)X_t dt + \sqrt{\beta(t)}dW_t
}
$$

上式为 VP-SDE（Variance Preserving Stochastic Differential Equation），VP 指的是加噪过程中样本方差保持在单位方差附近。

它是一个伊藤随机微分方程（Itô SDE），严格含义是：
$$
X_t = X_0 + \int_0^t -\frac{1}{2}\beta(s)X_s ds + \int_0^t \sqrt{\beta(s)}dW_s
$$

### 验证 VP-SDE 是否符合 DDPM 的前向扩散

#### 求解确定性部分

忽略随机部分，得到：
$$
\frac{dX_t}{dt} = -\frac{1}{2}\beta(t)X_t
$$

解得：
$$
X_t = X_0\exp\left(-\frac{1}{2}\int_0^t\beta(s)ds\right)
$$

定义
$$
\alpha_t = \exp\left(-\frac{1}{2}\int_0^t\beta(s)ds\right)
$$

#### 求解随机部分

用积分因子法求解 SDE，由于噪声系数只依赖时间，得
$$
X_t = \alpha_t X_0 + \alpha_t\int_0^t \sqrt{\beta(s)}dW_s
$$
右边第二项是伊藤随机积分。

其服从均值为0的高斯分布，协方差为：
$$
\Sigma_t
=\alpha_t^2
\int_0^t\frac{\beta(s)}{\alpha_s^2}ds\,I\\
=\left(1-e^{-\int_0^t\beta(s)ds}\right)I
$$

定义
$$
\sigma_t = \sqrt{1-e^{-\int_0^t\beta(s)ds}}
$$

最终，给定 $X_0 = x_0$时：
$$
\boxed{
X_t\overset{d}{=}\alpha_t x_0+\sigma_t\epsilon
} \quad \epsilon \in \mathcal{N}(0,I)
$$

### VP 重要性质

$$
\alpha_t^2 + \sigma_t^2 = 1
$$

若初始数据满足 $Cov(X_0)=I$，则整个扩散过程有
$$
Cov(X_t) = \alpha_t^2I+\sigma_t^2I = I
$$

### SiT 的随机插值（Stochastic Interpolant）

由于 DDPM 加噪过程可以推导出
$$
\boxed{
X_t\overset{d}{=}\alpha_t x_0+\sigma_t\epsilon
} \quad \epsilon \in \mathcal{N}(0,I)
$$

所以，SiT 允许反过来直接设计 $\alpha_t$ 和 $\sigma_t$。

## 逆时 SDE

### 正时 SDE

正时 SDE 的一般形式：
$$
dX_t = f(X_t,t)dt + g(t)dW_t
$$

其中：
- $f(X_t,t)$：Drift，确定性漂移。
- $g(t)$：Diffusion Coefficient，随机扩散强度。
- $dW_t$：维纳过程增量。

对应一个很小的正向时间步 $h>0$：
$$
X_{t+h}\approx X_t+f(X_t,t)h+g(t)\sqrt h\,\epsilon \quad \epsilon\sim\mathcal N(0,I)
$$

因此，我们可以直接得到正向条件概率：
$$
\boxed{
p(X_{t+h}=y\mid X_t=x)
\approx
\mathcal N(y;x+f(x,t)h,g^2(t)hI)
}
$$

### 贝叶斯求反向分布

由贝叶斯公式：
$$
\boxed{
p(X_t=x\mid X_{t+h}=y)
=
\frac{
p(X_{t+h}=y\mid X_t=x)p_t(x)
}{
p_{t+h}(y)
}
}
$$

这里：
- $p(X_{t+h}=y\mid X_t=x)$：正向转移概率，已经知道。
- $p_t(x)$：时间 $t$ 时数据的边缘概率密度。
- $p_{t+h}(y)$：归一化因子。

> Score 出现的原因：反向分布不仅由转移概率决定，还依赖当前数据的概率分布 $p_t(x)$。

#### 代入正向高斯分布

为简化推导，暂时考虑 $g(t)$ 不依赖状态 $x$。
正向转移密度为：
$$
p(y\mid x)\propto
\exp\left(
-\frac{\|y-x-f(x,t)h\|^2}{2g^2(t)h}
\right)
$$

根据贝叶斯公式，忽略与 $x$ 无关的归一化常数：
$$
p(x\mid y)\propto
\exp\left(
-\frac{\|y-x-f(x,t)h\|^2}{2g^2(t)h}
\right)p_t(x)
$$

当 $h\to0$ 时，扩散增量趋近于零，所以可以围绕 $y$ 对函数展开。

### Score Function

处理 $p_t(x)$，对其对数概率密度进行一阶泰勒展开：
$$
\log p_t(x) \approx
\log p_t(y) + \nabla_x \log p_t(y) \cdot (x-y)
$$

定义 Score：
$$
\boxed
{
    s(y,t) = \nabla_y \log p_t(y)
}
$$
则有：
$$
p_t(x) \approx p_t(y) \exp
\left[ 
s(y,t) \cdot (x-y)
\right]
$$

### 计算反向高斯分布的均值

令 $\delta=x-y$。由于时间间隔很小，可以在主要项中将 $f(x,t)$ 近似为 $f(y,t)$。
将刚才的泰勒展开代入贝叶斯公式：
$$
p(\delta\mid y)\propto
\exp\left[
-\frac{\|\delta+f(y,t)h\|^2}{2g^2(t)h}
+s(y,t)\cdot\delta
\right]
$$

为了方便，记
$$
a=g^2(t)h,\qquad f=f(y,t),\qquad s=s(y,t)
$$

展开指数部分：
$$
-\frac{\|\delta+fh\|^2}{2a}+s\cdot\delta = -\frac{\|\delta\|^2}{2a}
-\frac{fh\cdot\delta}{a}
+s\cdot\delta
-\frac{\|fh\|^2}{2a}
\\ =
-\frac{
\|\delta-(-fh+as)\|^2
}{2a}
+\text{常数}
$$

因此，反向增量的近似分布为：
$$
\boxed{
\delta\mid y
\approx
\mathcal N\left(
-f(y,t)h+g^2(t)h\,s(y,t),
g^2(t)hI
\right)
}
$$

由于：
$$
\delta=X_t-X_{t+h}
$$

所以得到**反向更新公式**：
$$
\boxed{
X_t\approx X_{t+h}
+\left[-f+g^2s\right]h
+g\sqrt h\,\epsilon
}
$$

可以看到，反向过程的漂移包含两部分：
$$
\underbrace{-f}_{\text{反转原来的漂移}}
+
\underbrace{g^2s}_{\text{根据数据概率密度修正}}
$$

这说明：
**反向扩散不只是把正向漂移取负，还必须根据当前边缘分布的 Score 修正反向运动方向。**

### 从离散反向更新得到逆时 SDE

现在我们得到：
$$
X_t\approx X_{t+h}
+\left[-f+g^2s\right]h
+g\sqrt h\,\epsilon
$$

为了严格区分时间方向，定义新的逆向时间变量：
$$
\tau=T-t
$$

令：
$$
Y_\tau=X_{T-\tau}
$$

这样，当 $\tau$ 从 0 增加到 $T$ 时，对应原来的时间 $t$ 从 $T$ 减少到 0。
因此：
$$
\boxed{
dY_\tau=
\left[
-f(Y_\tau,T-\tau)
+g^2(T-\tau)s(Y_\tau,T-\tau)
\right]d\tau
+g(T-\tau)d\widetilde W_\tau
}
$$

这就是使用正向增加的逆时间变量 $\tau$ 写出的逆时 SDE。

### 代入 VP-SDE，与 DDPM 对应

之前推导的 VP-SDE 是：
$$
dX_t=-\frac12\beta(t)X_tdt
+\sqrt{\beta(t)}dW_t
$$

因此：
$$
f(X_t,t)=-\frac12\beta(t)X_t
$$

$$
g^2(t)=\beta(t)
$$

代入逆时 SDE：
$$
\boxed{
dX_t=
\left[
-\frac12\beta(t)X_t
-\beta(t)s(X_t,t)
\right]dt
+\sqrt{\beta(t)}d\bar W_t
}
$$

这就是经典 VP-SDE 的逆时形式。

## Reference

* [Diffusion Model与SDE、ODE之间的联系](https://zhuanlan.zhihu.com/p/698737562)
* [SiT: Exploring Flow and Diffusion-based Generative Models with Scalable Interpolant Transformers](https://arxiv.org/abs/2401.08740)

