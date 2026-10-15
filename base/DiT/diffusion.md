# Diffusion Model

## Overview

阅读顺序建议：[本文](diffusion.md) → [DiT 架构](DiT.md) → [SiT 连续时间框架](SiT.md)。本文聚焦 DDPM，使用离散时间下标 $t=1,\ldots,T$。

核心思想是：先人为构造一个容易计算的加噪过程，再训练神经网络学习其逆过程，使模型能够从随机噪声中生成符合真实数据分布的样本。

## 前向扩散（Forward Diffusion）

逐渐加噪：

$$
q(x_t | x_{t-1})=\mathcal{N}(\sqrt{1-\beta_t}x_{t-1},\beta_t I)
$$

其中，$q$ 表示人为定义的前向概率分布；$\beta_t$ 是第 $t$ 步的噪声方差，由预先设定的 noise schedule 决定；$x_{t-1}$ 是上一步的数据，$x_t$ 是当前带噪数据。

定义 $\alpha_t=1-\beta_t$，重参数化后（**前向公式形式**）：

$$
x_t = \sqrt{\alpha_t}x_{t-1}+\sqrt{1-\alpha_t}\epsilon_t
$$

其中，$\epsilon_t\sim\mathcal N(0,I)$，不同时间步独立采样。

这条公式的含义十分直接：**每一步将原本的数据稍微缩小，再加入一部分随机噪声**。缩放因子目的是防止加噪可能导致的整体方差持续增长。

定义累计信号保留量：

$$
\bar\alpha_t=\prod_{s=1}^{t}\alpha_s,\qquad \bar\alpha_0=1.
$$

经过递推，对应的条件分布为：

$$
q(x_t|x_0)=\mathcal N(\sqrt{\bar\alpha_t}x_0,(1-\bar\alpha_t)I)
$$

它意味着：即使扩散过程有 1000 步，在训练时我们也不需要先计算 $x_1,x_2,\ldots,x_{t-1}$，而是可以直接从原始数据 $x_0$ 得到任意时间步的 $x_t$。
当 $t$ 足够大，且设计的 schedule 使得 $\bar\alpha_T\approx 0$，就有：

$$
x_T\approx \epsilon\sim\mathcal N(0,I)
$$

也就是说，真实数据最终变成了近似标准高斯噪声。

## 反向扩散（Reverse Diffusion）

反向过程不能直接简单求逆，因为添加的具体噪声未知，且噪声破坏了部分信息。

DDPM 使用神经网络参数化高斯反向分布：

$$
p_\theta(x_{t-1}\mid x_t)
=\mathcal N\bigl(\mu_\theta(x_t,t),\Sigma_\theta(x_t,t)\bigr).
$$

真实反向条件分布 $q(x_{t-1}\mid x_t)$ 一般不是简单高斯；小步长加噪使逐步高斯近似成为可用的建模方式。

### 噪声预测

根据先前递推推导得到的公式，可以得知，知道噪声 $\epsilon$ 后，结合当前状态 $x_t$ 和时间步 $t$，就可以反推原始数据。于是可以直接让网络预测从 $x_0$ 到 $x_t$ 过程中的等效噪声 $\hat{\epsilon}$。

训练时已知 $x_0$，因此可以通过贝叶斯公式解析计算：

$$
q(x_{t-1}\mid x_t,x_0)
=\mathcal N\bigl(\tilde\mu_t(x_t,x_0),\tilde\beta_t I\bigr),
\qquad
\tilde\beta_t=\frac{1-\bar\alpha_{t-1}}{1-\bar\alpha_t}\beta_t.
$$

这里必须区分 $q(x_{t-1}\mid x_t,x_0)$ 与 $q(x_{t-1}\mid x_t)$。后者需要对未知的原始数据积分：

$$
q(x_{t-1}\mid x_t)
=\int q(x_{t-1}\mid x_t,x_0)q(x_0\mid x_t)\,dx_0,
$$

通常无法直接解析计算。生成时也没有已知的 $x_0$，因此需要学习 $p_\theta$。

将构造 $x_t$ 时使用的等效噪声 $\epsilon$ 代入，条件后验的均值可以写为：

$$
\tilde\mu_t=
\frac{1}{\sqrt{\alpha_t}}
\left(
x_t-
\frac{\beta_t}{\sqrt{1-\bar\alpha_t}}
\epsilon
\right)
$$

这个条件后验均值可以写成 $x_t$ 和真实等效噪声 $\epsilon$ 的函数。采样时用网络预测 $\epsilon_\theta(x_t,t)$ 替换 $\epsilon$，从而构造反向模型的均值 $\mu_\theta$。

MSE 的最优预测是条件期望 $\mathbb E[\epsilon\mid x_t,t]$，因此网络学习的是噪声的统计估计；它不需要精确恢复某个样本当初使用的噪声。

## Training

从真实数据采样 $x_0\sim q_{\mathrm{data}}(x)$，随机选择时间步 $t\sim\mathcal U(\{1,\ldots,T\})$，随机采样噪声 $\epsilon\sim\mathcal N(0,I)$，最后根据前向公式构造：

$$
x_t=\sqrt{\bar\alpha_t}x_0+
\sqrt{1-\bar\alpha_t}\epsilon
$$

DDPM 常用的简化噪声预测目标为：

$$
\mathcal L_{\mathrm{simple}}(\theta)=
\mathbb E_{x_0,t,\epsilon}
\left[
\left\|
\epsilon-
\epsilon_\theta(x_t,t)
\right\|_2^2
\right]
$$

它来自对变分训练目标的简化与重加权。若模型还学习反向方差（例如原始 DiT），需加入方差学习对应的变分项。

## Sampling

采样阶段从标准高斯噪声初始化：

$$
x_T\sim\mathcal N(0,I)
$$

接下来需要从 $T$ 一直迭代到 $1$。
对每个时间步，调用网络：

$$
\hat\epsilon=
\epsilon_\theta(x_t,t)
$$

计算反向分布均值：

$$
\mu_\theta(x_t,t)=
\frac{1}{\sqrt{\alpha_t}}
\left(
x_t-\frac{\beta_t}{\sqrt{1-\bar\alpha_t}}
\hat\epsilon
\right)
$$

然后采样：

$$
\boxed{
x_{t-1}=
\mu_\theta(x_t,t)+\sigma_tz
}
$$

其中 $z\sim\mathcal N(0,I)$，每一步独立采样。固定方差的一种常见选择是 $\sigma_t^2=\tilde\beta_t$；也可以取 $\beta_t$，或使用学习的方差。这里的 $\sigma_t$ 表示**单次反向更新的噪声标准差**，与 SiT 中累计加噪幅度 $\sigma(t)$ 的含义不同。

最后一步 $t=1$ 不再添加随机噪声，可令 $z=0$。

这样就得到：

$$
x_T\rightarrow x_{T-1}\rightarrow
\cdots\rightarrow x_1\rightarrow x_0
$$

最终生成一个样本。

采样的目的不是找回某个特定图像，而是从学到的数据分布中产生一个新的样本。

## 与连续时间记号的对应

| 含义 | 本文的 DDPM 离散记号 | SiT / VP-SDE 连续记号 |
|---|---|---|
| 单步信号保留量 | $\alpha_k=1-\beta_k$ | 不直接对应连续信号幅度 |
| 累计信号幅度 | $\sqrt{\bar\alpha_k}$ | $\alpha(t)$ |
| 累计噪声幅度 | $\sqrt{1-\bar\alpha_k}$ | $\sigma(t)$ |
| 反向单步噪声标准差 | $\sigma_k$ | 由 SDE 扩散强度和步长共同决定 |

注意：离散 $\beta_k$ 是单步噪声方差，连续 $\beta(t)$ 是噪声率；连续化时对应 $\beta_k\approx\beta(t_k)\Delta t$。

## Reference

- [Denoising Diffusion Probabilistic Models](https://arxiv.org/abs/2006.11239)：第 3 节、公式（6）—（12）及训练、采样算法。

- [Hugging Face Diffusion Course — Unit 1](https://huggingface.co/learn/diffusion-course/en/unit1/1)
