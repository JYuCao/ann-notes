# ViT (Vision Transformer)

## 简介

ViT 是 2020 年由 Google 团队提出的将 Transformer 架构应用于计算机视觉任务的模型。

该论文最核心的结论是：当训练数据足够大时，ViT 的表现可以超过 CNN，突破 Transformer 缺少归纳偏置（inductive bias）的限制。但如果训练数据较小，ViT 的表现会比同等大小的 ResNets 差，因为 Transformer 和 CNN 相比缺少归纳偏置。

> **归纳偏置**：指模型在学习过程中对数据结构的先验假设。Transformer 的归纳偏置较弱，意味着它需要更多的数据来学习数据的结构和模式，而 CNN 具有更强的归纳偏置，能够更有效地捕捉图像中的局部特征。
> CNN具有两种归纳偏置，一种是局部性（locality/two-dimensional neighborhood structure），即图片上相邻的区域具有相似的特征；一种是平移不变形（translation equivariance），$ g(f(x)) = f(g(x)) $，其中$g$代表卷积操作，$f$代表平移操作。当 CNN 具有以上两种归纳偏置，就有了很多先验信息，需要相对少的数据就可以学习一个比较好的模型。

## ViT 结构


