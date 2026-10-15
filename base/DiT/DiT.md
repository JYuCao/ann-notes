# Scalable Diffusion Models with Transformers

## 概览

DiT（Diffusion Transformer）在 latent diffusion 中使用 Transformer 替换 U-Net，处理带噪 latent 的 patch tokens。扩散训练和反向采样的基本过程见 [Diffusion Model](diffusion.md)，连续概率路径与采样设计见 [SiT](SiT.md)。

论文的关键问题是：**Transformer 能否作为扩散模型的 Backbone，并通过增加模型计算量持续改善生成质量？**

在论文考察的模型和训练设置中，增加深度、宽度或输入 token 数量，通常能改善 FID；参数量本身不足以解释全部性能变化。DiT-XL/2 使用 CFG = 1.5 时，在 ImageNet 256×256 上达到 FID 2.27，在 512×512 上达到 3.04。这些是论文当时的结果，比较时需要同时注明分辨率、训练预算和采样设置。

## 1. 完整数据流

训练时，先用预训练 VAE 编码真实图像，再在 latent 空间加噪：

$$
x_0\xrightarrow{\mathrm{VAE\ Encoder}}z_0,
\qquad
z_t=\sqrt{\bar\alpha_t}z_0+\sqrt{1-\bar\alpha_t}\epsilon,
\quad \epsilon\sim\mathcal N(0,I).
$$

DiT 接收的是带噪的 $z_t$，以及时间步 $t$ 和类别标签 $y$：

$$
z_t\xrightarrow{\mathrm{Patchify+Embedding}}
H\in\mathbb R^{N\times d}
\xrightarrow{\mathrm{DiT\ Blocks}(t,y)}
H'\xrightarrow{\mathrm{Linear+Unpatchify}}
(\hat\epsilon,\hat r).
$$

其中 $\hat r$ 是反向过程的方差参数，不能直接理解为任意取值的协方差矩阵。采样时先从高斯 latent 开始，反向迭代得到 $\hat z_0$，最后通过 VAE Decoder 得到图像。

## 2. Patchify 与位置编码

对于 256×256 图像，VAE 的空间下采样倍率为 8：

$$
x_0\in\mathbb R^{256\times256\times3}
\longrightarrow z_0\in\mathbb R^{32\times32\times4}.
$$

设 latent 尺寸为 $H_z\times W_z\times C$，patch 边长为 $p$，则 token 数为：

$$
\boxed{N=\frac{H_zW_z}{p^2}}.
$$

每个 patch 展平后有 $p^2C$ 个元素，通过线性映射得到 $d$ 维 token，再加上固定的二维 sine-cosine 位置编码。这里用 $N$ 表示 token 数，避免与扩散总步数 $T$ 混淆。

例如，$32\times32$ latent 使用 $p=2$ 时，$N=256$；使用 $p=4$ 时，$N=64$。DiT-XL/2 中的 XL 表示模型规模，/2 表示 **latent patch** 的边长为 2。

缩小 patch 会增加 token 数与计算量；自注意力的 token 两两交互部分随 $N^2$ 增长，因此模型规模和 patch 大小应放在同一计算预算下比较。

## 3. DiT Block：如何注入条件？

论文比较了四种时间和类别条件的注入方式。这里的实验是 ImageNet 类别条件生成；自然语言等更长条件序列需要另行设计和验证。

### 3.1 In-Context Conditioning

将时间步 $t$ 和类别标签 $y$ 分别编码为两个额外 token，附加到图像 token 序列，通过 self-attention 进行交互。

### 3.2 Cross-Attention Block

将时间和类别 embedding 组成长度为 2 的条件序列，与图像 token 分开。图像 token 通过额外的 cross-attention 读取条件，论文设置下增加约 15% 的 GFLOPs。

### 3.3 Adaptive Layer Norm（adaLN）

先组合时间与类别 embedding：

$$
c=\mathrm{Embed}_{t}(t)+\mathrm{Embed}_{y}(y).
$$

通过条件网络预测 shift 和 scale，对归一化后的特征进行调制：

$$
\mathrm{adaLN}(h,c)
=(1+\gamma(c))\odot\mathrm{LN}(h)+\beta(c).
$$

其中 LayerNorm 不使用自身可学习的 affine 参数；条件网络输出的参数在 token 维度广播。这里 $\gamma,\beta$ 是条件调制参数，与扩散噪声 schedule 中的 $\beta_t$ 含义不同。

### 3.4 adaLN-Zero

adaLN-Zero 进一步为两个残差分支预测 gate：

$$
\begin{aligned}
h'&=h+g_{\mathrm{attn}}(c)\odot
\mathrm{Attention}\!\left(
(1+\gamma_{\mathrm{attn}}(c))\odot\mathrm{LN}(h)
+\beta_{\mathrm{attn}}(c)\right),\\
h''&=h'+g_{\mathrm{mlp}}(c)\odot
\mathrm{MLP}\!\left(
(1+\gamma_{\mathrm{mlp}}(c))\odot\mathrm{LN}(h')
+\beta_{\mathrm{mlp}}(c)\right).
\end{aligned}
$$

**每个 block 共预测六组 $d$ 维参数：两个 shift、两个 scale、两个 gate。** gate 对应论文中的残差缩放系数 $\alpha$；这里改用 $g$，避免与扩散信号系数混淆。

官方实现将条件调制网络最后一个线性层的权重和偏置初始化为零，使六组输出初始都为零。于是两个残差分支的 gate 为零，整个 block 初始为恒等映射：

$$
h''=h.
$$

这是 block 级别的性质；完整 DiT 还包含输入投影、最终调制与输出投影。adaLN-Zero 在论文的条件注入消融中表现最好，并被用作主要配置。

## 4. 输出层与训练目标

最后一个 block 之后，先做条件调制的 LayerNorm，再将每个 token 线性映射为 $p^2\times2C$ 个数，重新排列为 $H_z\times W_z\times2C$ 的输出。

前 $C$ 个通道预测噪声，后 $C$ 个通道参数化对角反向方差。例如 $C=4$ 时，输出共 8 个通道。

**这两个分支的学习目标不同：**噪声预测使用 DDPM 的简化 MSE；学习方差还需要变分下界相关的训练项。因此，理解原始 DiT 时，不能只用噪声 MSE 概括完整训练目标。

## 5. 阅读时需要记住的结论

- DiT 改变的是去噪网络 Backbone，仍在 VAE latent 空间执行扩散。
- adaLN-Zero 通过条件调制与零初始化残差 gate 注入时间和类别信息。
- 增加 token 数也会显著增加计算量；模型缩放需要同时考察参数量、GFLOPs 和训练预算。
- SiT 沿用这一 Backbone，进一步研究概率路径、预测目标和采样方式。

## Reference

- [Scalable Diffusion Models with Transformers](https://arxiv.org/abs/2212.09748)：架构、条件注入消融与 scaling 实验。
- [DiT 官方实现：models.py](https://github.com/facebookresearch/DiT/blob/main/models.py)：`DiTBlock`、`FinalLayer` 与零初始化细节。
- [DiT 官方训练实现](https://github.com/facebookresearch/DiT/blob/main/diffusion/gaussian_diffusion.py)：噪声和方差分支的训练目标。
- [扩散模型解读（一）：DiT 详细解读](https://zhuanlan.zhihu.com/p/685867473)：辅助阅读。
