# ViT (Vision Transformer)

以 `vit-base-patch16-224` 为例进行分析，`patch16` 代表输入图像会被切分为 `16x16` 的小块，`224` 代表输入图像的大小为 `224x224`。

## Patch Embedding

该过程利用一个卷积核为 $16\times16$ 的 Conv2d 卷积层将输入图像切分为小块，图像被分割成 $(224/16) \times (224/16) = 14 \times 14 = 196$ 个互不重叠的小块，每个小块的大小为 $16\times16$。

每个 filter 的操作：

$$
3 \times 16 \times 16 \rightarrow 1 \times 14 \times 14
$$

同时，该卷积层有 $768$ 个 filter，因此整张图会被映射为一个 $768\times14\times14$ 的张量。

后续对输出的空间维进行展平，将其转换为一个 $768\times196$ 的张量，最终进行转置得到一个形状为 $(196, 768)$ 的张量。最前面加上一个 [CLS] token，最终得到一个形状为 $(197, 768)$ 的张量。

这在 Transformer 等价于 197 个 768 维的 token（单张 $224\times224$ 的图片）。

## Pooler

[CLS] token 已经包含了整张图片的全局信息，Pooler 通过一个线性层将 [CLS] token 映射为一个 $768$ 维的向量。最终该向量会被送入下游任务的分类器、任务头中。

$$
Pooler = \tanh(Linear(768\times768))
$$

## 其他与传统 Transformer 的区别

1. 激活函数函数

$$
GeLU = 0.5x(1 + tanh[\sqrt{\frac{2}{\pi}}(x + 0.044715x^3)])
$$

2. Position Embedding：ViT 使用了**可学习**的位置编码，即

$$
PE = Embedding(197, 768)
$$

3. 只有 Encoder，没有 Decoder。

## Inductive Bias

CNN 的 inductive bias 是局部性和空间不变性，而 ViT 对于局部性的 inductive bias 较弱，ViT 需要更多的数据来学习图像的空间结构。
