# Diffusion Model

## Overview

* 核心思想是：先人为构造一个容易计算的加噪过程，再训练神经网络学习其逆过程，使模型能够从随机噪声中生成符合真实数据分布的样本。

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

其中，$\epsilon_t\sim\mathcal{N}$为正态分布。

这条公式的含义十分直接：**每一步将原本的数据稍微缩小，再加入一部分随机噪声**。缩放因子目的是防止加噪可能导致的整体方差持续增长。

经过递推推导，对应的条件分布为：

$$
q(x_t|x_0)=\mathcal{N}(\sqrt{\bar{\alpha}}x_0,(1-\hat{\alpha})I)
$$

它意味着：即使扩散过程有 1000 步，在训练时我们也不需要先计算 $x_1,x_2,\ldots,x_{t-1}$，而是可以直接从原始数据 $x_0$ 得到任意时间步的 $x_t$。
当 $t$ 足够大，且设计的 schedule 使得 $\bar\alpha_T\approx 0$，就有：

$$
x_T\approx \epsilon\sim\mathcal N(0,I)
$$

也就是说，真实数据最终变成了近似标准高斯噪声。

## 反向扩散（Reverse Diffusion）

反向过程不能直接简单求逆，因为添加的具体噪声未知，且噪声破坏了部分信息。

所以使用 DDPM 进行近似反向建模。其建模的分布是一个高斯分布。

### 噪声预测

根据先前递推推导得到的公式，可以得知，知道噪声 $\epsilon$ 后，结合当前状态 $x_t$ 和时间步 $t$，就可以反推原始数据。于是可以直接让网络预测从 $x_0$ 到 $x_t$ 过程中的等效噪声 $\hat{\epsilon}$。

在已知 $x_0$ 的基础上，可以通过高斯分布的贝叶斯公式解析解以及 $q(x_{t-1}|x_t,x_0)$ 求出 $q(x_{t-1}|x_t)$。

其均值经过化简，并代入前向扩散中的噪声表达式后，可以写成：

$$
\tilde\mu_t=
\frac{1}{\sqrt{\alpha_t}}
\left(
x_t-
\frac{\beta_t}{\sqrt{1-\bar\alpha_t}}
\epsilon
\right)
$$

反向均值实际上可以表示成当前带噪状态 $x_t$ 与等效噪声 $\epsilon$ 的函数。

## Training

从真实数据采样 $x_0\sim q_{\mathrm{data}}(x)$，随机选择时间步 $t\sim\mathcal U(\{1,\ldots,T\})$，随机采样噪声 $\epsilon\sim\mathcal N(0,I)$，最后根据前向公式构造：

$$
x_t=\sqrt{\bar\alpha_t}x_0+
\sqrt{1-\bar\alpha_t}\epsilon
$$

直接使用 MSE 作为 loss：

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

## Sampling

采样阶段没有真实数据,只有一个随机初始化的：

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

其中通常在最后一步不再添加随机噪声。

这样就得到：

$$
x_T\rightarrow x_{T-1}\rightarrow
\cdots\rightarrow x_1\rightarrow x_0
$$

最终生成一个样本。

采样的目的不是找回某个特定图像，而是从学到的数据分布中产生一个新的样本。

## Reference

* [Hugging Face Diffusion Course — Unit 1](https://huggingface.co/learn/diffusion-course/en/unit1/1)
