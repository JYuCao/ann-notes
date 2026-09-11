# The Annotated Transformer

## Encoder

### Step

1. Input Embedding -> Positional Encoding
2. PreNorm -> Self-Attention -> Dropout -> Residual -> PreNorm -> FFN -> Dropout -> Residual
    * PreNorm: $x_{i+1} = x_i + Dropout(Sublayer(Norm(x_i)))$
    * PostNorm: $x_{i+1} = Norm(x_i + Dropout(Sublayer(x_i)))$，其中 $i$ 为当前层的索引，Sublayer 为 Self-Attention 或 FFN
3. Layer Normalization -> Output Z

> 1. **PostNorm 和 PreNorm 在性质上有什么区别？**
> 答：PreNorm 梯度传播通常更稳定，尤其适合深层 Transformer；PostNorm 是原始 2017 Transformer 论文描述的结构，但深层训练通常更难一些。Annotated Transformer 代码是 PreNorm，最终额外 LayerNorm。(https://medium.com/@ashutoshs81127/why-pre-norm-became-the-default-in-transformers-4229047e2620)
> 2. 要求每个 Sublayer 的输出维度与输入维度相同，以便进行残差连接。
> 3. 论文中的 layers 有 6 层，注意力头数为 8。

### Multi-Head Self-Attention
> scaled dot-product attention，除此之外还有 dot-product attention、additive attention 等。
$$
Z = \text{softmax}(\frac{QK^T}{\sqrt{d_k}})V
$$

1. 计算 Query、Key 和 Value 矩阵：
   * $Q = XW^Q$
   * $K = XW^K$
   * $V = XW^V$
2. 对 Q 和 K 矩阵进行点积，除以 Key 向量维度的平方根进行缩放，然后应用 softmax 函数得到一组注意力权重：
   * $ S = \frac{QK^T}{\sqrt{d_k}} $
   * $ A = \text{softmax}(S) $
3. 通过加权计算 head output：
   * $ Z = AV $
4. 最终输出为所有 head 的输出拼接后再通过线性变换：
   * $ Z = [Z_1, Z_2, ..., Z_h]W^O $

> **为什么要使用多个 Head 而非一个？**
> 答：每个 Head 对不同 token 的 value 进行不同的加权组合（$ Z = AV $），能够捕捉到不同的语义信息。单个 head 只产生一套 attention distribution，并在一个投影子空间中将多个位置的 Value 加权混合；多个 head 使用不同的 $W_i^Q,W_i^K,W_i^V$，可以并行地在不同表示子空间中建立不同的注意力模式，从而提高对多种关系的表示能力。<br>
> *原文：Multi-head attention allows the model to jointly attend to information from different representation subspaces at different positions. With a single attention head, averaging inhibits this.*

#### MHA 维度
1. 输入维度：$X \in \mathbb{R}^{n \times d_{model}}$，其中 $n$ 为序列长度，$d_{model}$ 为模型维度。
2. W 矩阵维度：
   * $W_i^Q \in \mathbb{R}^{d_{model} \times d_k}$
   * $W_i^K \in \mathbb{R}^{d_{model} \times d_k}$
   * $W_i^V \in \mathbb{R}^{d_{model} \times d_v}$
   * 输出维度：$Z \in \mathbb{R}^{n \times d_v}$
> 实现上，一般会先进行 $XW$ 的矩阵乘法，然后再进行切分。也就是说 $W$ 的维度是 $d_{model} \times d_{model}$，算出 $ Q, K, V $ 后再使用`view`进行切分。
3. 多头注意力：
   * 将 $d_{model}$ 分为 $h$ 个 head，每个 head 的维度为 $d_k = d_v = d_{model} / h$。（通常选择 $d_k=d_v=d_{\text{model}}/h$，使得 $h$ 个 head 拼接后维度恰好回到 $d_{\text{model}}$。理论上 $ d_k $可以不等于 $ d_v $）
   * 每个 head 的输出维度为 $Z_i \in \mathbb{R}^{n \times d_v}$，将所有 head 的输出拼接后再通过线性变换得到最终输出：
     * $Z = [Z_1, Z_2, ..., Z_h]W^O \in \mathbb{R}^{n \times d_{model}}$，其中 $W^O \in \mathbb{R}^{hd_v \times d_{model}}$。

> **$ W^O $ 的作用**是混合不同 head 的维度，学习这 8 个 head 的信息应该怎样组合成新的 $d_{\text{model}}$ 维 token representation。

例：
1. 输入维度：$X \in \mathbb{R}^{n \times 512}$，其中 $n$ 为序列长度。
2. W 矩阵维度：
   * $W_i^Q \in \mathbb{R}^{512 \times 64}$
   * $W_i^K \in \mathbb{R}^{512 \times 64}$
   * $W_i^V \in \mathbb{R}^{512 \times 64}$
   * 输出维度：$ A_i = softmax(\frac{XW_i^Q(XW_i^K)^T)}{\sqrt{d_k}}) \in \mathbb{R}^{n \times 64}$
3. $Z_i = A_i(XW_i^V) \in \mathbb{R}^{n \times 64}$
4. Concat 后的输出维度：
   * $Z = [Z_1, Z_2, ..., Z_8]W^O \in \mathbb{R}^{n \times 512}$，其中 $W^O \in \mathbb{R}^{512 \times 512}$。

### FFN

* 两层全连接网络，第一层使用 ReLU 激活函数，第二层不使用激活函数。（两次 Kernal Size 为 1 的卷积，$d_{model}=512,d_{ff}=2048$）
* 输入和输出维度相同，为 512，隐藏层维度为 2048。
* 实现上，第一个 ReLU 后跟一次 Dropout
* 作用：对每个 token 的特征进行非线性重组。

> 为什么要先升维再降维？<br>
> 答：主要是为了给非线性变换更大的中间表示空间。原来的 512 维里混合着语法、语义、指代、实体属性等信息，升到 2048 后，网络有更大的容量去形成新的非线性特征。降维是为了保持输出维度与输入维度一致，以便进行残差连接。

## Decoder

### Step
* Output(Shifted Right) -> Output Embedding -> Positional Encoding
* Masked MHA -> Cross Attention -> FFN（省略了残差连接和 LayerNorm，和 Encoder 类似）

### Masked MHA

* 与 Encoder 的 MHA 类似，但在计算注意力权重时，使用了一个 mask 来阻止模型在预测下一个 token 时看到未来的 token

### Encoder-Decoder Attention(Cross Attention)

* 接收 Encoder 的输出作为 Key 和 Value，Decoder Masked MHA 的输出作为 Query，计算注意力权重并生成新的表示
* Decoder Cross Attention 和 Encoder MHA 都使用了一个 mask 屏蔽 BOS 和 PAD token 的注意力权重，防止模型在生成时关注到这些无效的 token

> **为什么要使用 Cross Attention？**<br>
答：让生成过程受到输入内容的条件约束。Self-Attention 关注已经生成的目标序列，而 Cross Attention 关注输入序列的编码表示。

> **mask 在什么时候进行？**<br>
答：在计算 attention 时，softmax前，将 mask 对应位置的 score 设置为 1e-9，使得 softmax 后的注意力权重为 0，从而阻止模型关注这些位置。

### FFN

* 与 Encoder 的 FFN 相同，对每个 token 的特征进行非线性重组

## Embedding

* X = $\sqrt{d_{model}}Emb(tokens)+PE$
* $\sqrt{d_{model}}$用于将 embedding 调整到合适的尺度，使它和 positional encoding 相加时处于合理的数值尺度
* 其中`Emb()`是一个矩阵$E\in\mathbb{R}^{V\times d_{model}}$，其中 $V$ 为词表大小。可以推测 input token 的维度为 $L\times V$，但实际实现是使用一个 one-hot 向量，向量每个元素是对应词表中 token 的索引，经查找对应后得到 $1\times d_{model}$ 的向量表示。
* Encoder 的输入 Embedding 和 Decoder 的输入/输出 Embedding 共享权重矩阵 $E$，若最终输出有 softmax 层，则操作在 softmax 前进行
