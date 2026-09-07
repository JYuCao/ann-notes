# The Annotated Transformer

---

## Encoder

### Step

1. Input Embedding -> Positional Encoding
2. PreNorm -> Self-Attention -> Dropout -> Residual -> PreNorm -> FFN -> Dropout -> Residual
    * PreNorm: $x_{i+1} = x_i + Dropout(Sublayer(Norm(x_i)))$
    * PostNorm: $x_{i+1} = Norm(x_i + Dropout(Sublayer(x_i)))$，其中 $i$ 为当前层的索引，Sublayer 为 Self-Attention 或 FFN
3. Layer Normalization -> Output Z

> 1. **PostNorm 和 PreNorm 在性质上有什么区别？**
> 答：PreNorm 梯度传播通常更稳定，尤其适合深层 Transformer；PostNorm 是原始 2017 Transformer 论文描述的结构，但深层训练通常更难一些。Annotated Transformer 代码是 PreNorm，最终额外 LayerNorm。
> 2. 要求每个 Sublayer 的输出维度与输入维度相同，以便进行残差连接。
> 3. 论文中的 layers 有 6 层，注意力头数为 8。

### Multi-Head Self-Attention

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

> **为什么要使用多个 Head 而非一个？**
> 答：每个 Head 对不同 token 的 value 进行不同的加权组合（$ Z = AV $），能够捕捉到不同的语义信息。单个 head 只产生一套 attention distribution，并在一个投影子空间中将多个位置的 Value 加权混合；多个 head 使用不同的 \(W_i^Q,W_i^K,W_i^V\)，可以并行地在不同表示子空间中建立不同的注意力模式，从而提高对多种关系的表示能力。

#### MHA 维度
1. 输入维度：$X \in \mathbb{R}^{n \times d_{model}}$，其中 $n$ 为序列长度，$d_{model}$ 为模型维度。
2. W 矩阵维度：
   * $W_i^Q \in \mathbb{R}^{d_{model} \times d_k}$
   * $W_i^K \in \mathbb{R}^{d_{model} \times d_k}$
   * $W_i^V \in \mathbb{R}^{d_{model} \times d_v}$
   * 输出维度：$Z \in \mathbb{R}^{n \times d_v}$
3. 多头注意力：
   * 将 $d_{model}$ 分为 $h$ 个 head，每个 head 的维度为 $d_k = d_v = d_{model} / h$。（通常选择 \(d_k=d_v=d_{\text{model}}/h\)，使得 \(h\) 个 head 拼接后维度恰好回到 \(d_{\text{model}}\)。理论上 $ d_k $可以不等于 $ d_v $）
   * 每个 head 的输出维度为 $Z_i \in \mathbb{R}^{n \times d_v}$，将所有 head 的输出拼接后再通过线性变换得到最终输出：
     * $Z = [Z_1, Z_2, ..., Z_h]W^O \in \mathbb{R}^{n \times d_{model}}$，其中 $W^O \in \mathbb{R}^{hd_v \times d_{model}}$。

> **$ W^O $ 的作用**是混合不同 head 的维度，学习这 8 个 head 的信息应该怎样组合成新的 \(d_{\text{model}}\) 维 token representation。

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

* 两层全连接网络，第一层使用 ReLU 激活函数，第二层不使用激活函数。
* 输入和输出维度相同，为 512，隐藏层维度为 2048。
* 作用：对每个 token 的特征进行非线性重组。

> 为什么要先升维再降维？<br>
> 答：主要是为了给非线性变换更大的中间表示空间。原来的 512 维里混合着语法、语义、指代、实体属性等信息，升到 2048 后，网络有更大的容量去形成新的非线性特征。降维是为了保持输出维度与输入维度一致，以便进行残差连接。

## Decoder

