# Scalable Diffusion Models with Transformers

## 概览

本文探索了一类新的基于 Transformer 的扩散模型 Diffusion Transformers (DiTs)。本文训练 latent diffusion models 时，使用 Transformer 架构替换常用的 UNet 架构，且 Transformer 作用于 latent patches 上。

作者探索了 DiT 的缩放性，发现具有较高 GFLOPs 的 DiT 模型，通过增加 Transformer 宽度或者深度或者输入 token 数量，始终有更好的 FID 值。最大的 DiT-XL/2 模型在 ImageNet 512×512 和 256×256 的测试中优于所有先前的扩散模型，实现了 2.27 的 FID 值。

## DiT 架构

### Patchify

1. 预训练 VAE Encoder：$ x_0 \in \mathbb{R}^{256\times 256\times 3} \rightarrow z_0 \in \mathbb{32\times 32\times 4} $

2. Patch Embedding：和 ViT 类似，将图片 Patchify，经过 linear Embedding，最终变为 $T$ 个 $d$ 维 tokens。再应用基于 ViT 频率的位置编码（sine-cosine）

其中 $T$ 和 $p$ 满足 $T=(1/p)^2$.

### DiT Block

为了解决 Diffusion Model 有时会处理额外的条件信息的情况（噪声时间步长 $t$，类标签 $c$，自然语言等），论文探索了四种不同的结构。

#### In-Context Conditioning

以 ViT 为基础，将时间步长 $t$ 和类标签 $c$ 作为两个额外的 token 附加到输入序列。

#### Cross-Attention Block

将 $t$ 和 $c$ 的 Embedding 连接成一个长度为2的 Sequence，且与 image token 序列分开。添加了一个 cross-attention block，额外增添了 15% 的 GFLOPs 开销。

#### Adaptive Layer Norm(adaLN) Block

Adaptive Layer Norm (adaLN) Block 遵循 GAN 中的自适应归一化层，希望探索这个东西在扩散模型里面好不好用。

本文作者没有直接学习缩放和移位参数 $\gamma$ 和 $\beta$ ，*而是改用噪声时间步长 $t$ 和类标签 $c$ 得到*。adaLN 带来的额外的 GFLOPs 是最少的。

#### adaLN-Zero Block

有工作发现在有监督学习中，对每个 Block 的第一个 Batch Norm 操作的缩放因子进行 Zero-Initialization 可以加速其大规模训练。基于 U-Net 的扩散模型使用类似的初始化策略，对每个 Block 的第一个卷积进行 Zero-Initialization。

本文作者作了一些改进：除了回归计算缩放和移位参数 $\gamma$ 和 $\beta$ 之外，还回归缩放系数 
 。

作者初始化 MLP 使其输出的缩放系数 $\alpha$ 全部为0，这样一来，DiT Block 就初始化为了 Identity Function。adaLN-Zero Block 带来的额外的 GFLOPs 可以忽略不计。

### Transformer Decoder

经过最后一个 DiT Block 后，需要将 image 的序列解码为输出噪声以及对角的协方差矩阵的预测结果。这两个输出的形状都与原始的空间输入一致。

作者在这个环节使用标准的线性解码器，将每个 token 线性解码为 $p\times p\times 2C$ 的张量，其中 $C$ 是空间输入中到 DiT 的通道数。最后将解码的 tokens 重新排列到其原始空间布局中，得到预测的噪声和协方差。

## Reference
* [Scalable Diffusion Models with Transformers](https://arxiv.org/abs/2212.09748)

* [扩散模型解读 (一)：DiT 详细解读](https://zhuanlan.zhihu.com/p/685867473)
