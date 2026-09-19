# The Annotated Transformer 笔记

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
> 4. Norm 除了 LayerNorm 之外，还有 RMSNorm、QKNorm 等。
> 其他机制记录：滑动窗口注意力、分组查询注意力、MLA、无位置嵌入、$\mu$子优化器

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

> 直接使用 FFN 的 Transformer 被称为稠密 Transformer，使用 MoE 的 Transformer 被称为稀疏 Transformer。

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
* Encoder 的输入 Embedding 和 Decoder 的输入/输出 Embedding 共享权重矩阵 $E$，操作在 softmax 前进行

## Position Encoding

* $PE_{(pos,2i)}=\sin{\frac{pos}{10000^{\frac{2i}{d_{model}}}}}$
* $PE_{(pos,2i+1)}=\cos{\frac{pos}{10000^{\frac{2i}{d_{model}}}}}$
* 位置编码过后要进行一次 Dropout
* 相对位置编码直接建模 token 间的距离与方向，使注意力更容易学习位置关系，并具有更好的平移不变性与长度泛化能力。

## Training

1. Batch：保存 src，构造 src_mask；将完整 tgt 错一位切成 Decoder 输入 tgt 和监督标签 tgt_y，构造 tgt_mask，统计非 `<PAD>` 的 ntokens。
2. forward：src 和 tgt 经过 embedding、Encoder、Decoder，得到每个 target 位置的 hidden representation。
3. SimpleLossCompute：将 Decoder 输出经 Generator 从 \(d_{model}\) 映射到词表维度 \(V\)，再将 $[B,L,V]$ flatten 为 $[B\times L,V]$。
4. label smoothing / Criterion：将目标 token ID 构造成平滑后的 soft target，并用 KLDivLoss 计算所有有效 token 的总损失，再除以 ntokens 得到平均 token loss。
5. Backward / Optimizer / Scheduler：backward() 计算梯度；Adam 根据梯度更新参数；清空梯度；scheduler 根据 warmup + decay 规则调整后续 learning rate。

### Batches and Masking

* 将完整目标序列 tgt 错位切分为 Decoder 输入 tgt = tgt[:, :-1] 和监督标签 tgt_y = tgt[:, 1:]，使模型能够在 causal mask 的约束下并行完成所有位置的 next-token prediction。
* 构造 src_mask 和 tgt_mask
   * src_mask 用于屏蔽输入序列中的 `<PAD>` token，防止 Encoder self-attention 和 Decoder cross-attention 关注这些无效位置。
   * tgt_mask 同时屏蔽输出序列中的 `<PAD>` token 和未来位置，使 Decoder self-attention 只能访问当前位置及之前的 token。
* 统计 tgt_y 中非 <PAD> token 的数量 ntokens，用于后续 loss 归一化。

### SimpleLossCompute

假设 Decoder 的输出为 $X \in \mathbb{R}^{B\times L\times d_{model}}$，其中 $B$ 为 batch size，$L$ 为序列长度，$d_{model}$ 为模型维度.

1. 将 \(X\) 经 Generator 的 Linear(d_model,V) 和 log_softmax 映射为 $ \mathbb{R}^{B\times L\times V} $，其中 $V$ 为词表大小. 表示每个序列位置对词表中 \(V\) 个 token 的 log probability.
2. 将模型输出 reshape 为 $ \mathbb{R}^{(B*L)\times V} $，将目标 token 的索引 reshape 为 $ \mathbb{R}^{(B*L)} $，然后计算交叉熵损失. 本质上是将每个有效序列位置视为一个 next-token prediction 样本，根据该位置的 contextual hidden state，在大小为 \(V\) 的词表中预测下一个 token，因此每个位置对应一个 \(V\) 类分类任务。
3. LabelSmoothing 根据目标 token 构造 soft target distribution，并通过 KLDivLoss 计算损失；所有有效 token 的 loss 求和后除以 ntokens，得到平均 token loss。

### Backward / Optimizer

```python
# 核心训练循环
out = model.forward(
    batch.src,
    batch.tgt,
    batch.src_mask,
    batch.tgt_mask,
)

loss, loss_node = loss_compute(
    out,
    batch.tgt_y,
    batch.ntokens,
)

loss_node.backward()

optimizer.step()

optimizer.zero_grad(set_to_none=True)

scheduler.step()
```

* forward: 前向传播，计算模型输出并建立计算图。
* loss.backward: 根据 loss 进行反向传播，通过链式法则计算所有可训练参数的梯度，但不更新参数。
* optimizer.step: 更新参数
   * Adam: 根据当前梯度的一阶矩、二阶矩估计以及当前 learning rate 更新模型参数：
$$
\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{\hat{v}_t} + \epsilon}\hat{m}_t
$$
* zero_grad: 清空梯度，因为 PyTorch 默认会累加梯度。若故意延迟清空并经过多个 micro-batch 后再执行则构成 gradient accumulation，可用于模拟更大的 effective batch size。
* scheduler.step()：更新 learning rate schedule。

### Warmup / Decay

$$
lr = d_{model}^{-0.5} \cdot \min(step^{-0.5}, step \cdot warmup^{-1.5})
$$

为什么一开始不用正常的学习率，而是先 warmup？

因为一开始模型参数是随机初始化的，梯度可能非常大，直接使用较大的学习率可能导致梯度爆炸或训练不稳定。通过 warmup，学习率从一个较小的值逐渐增加，使模型在训练初期能够更稳定地收敛。

### Label Smoothing

* 将 one-hot target 转换为 soft target distribution，避免模型被要求将正确类别概率推向绝对的 1，从而起到 regularization、缓解过度自信的作用。
* 若 smoothing 系数为 \(\epsilon\)，正确 token 分配概率 \(1-\epsilon\)，其余非 <PAD>、非正确类别共享剩余的 \(\epsilon\)。Annotated Transformer 中 <PAD> 类别概率始终设为 0，并且 target 本身为 <PAD> 的位置不参与 loss。
* Generator 提供模型的 log probability \(\log p\)，Label Smoothing 提供目标分布 \(q\)，KLDivLoss 最小化：
$$ D_{KL}(q\|p) = \sum_i q_i\log\frac{q_i}{p_i} $$

* 原文指出，Label Smoothing 可能使 perplexity 略差，因为模型不再追求极端高置信度，但可以改善 accuracy 和 BLEU。

## Inference：Greedy Decoding

1. 将源序列经过 Encoder 得到固定的 memory，随后以 <BOS> 作为 Decoder 的初始输入。

2. 在第 \(t\) 个生成步骤中，将当前已经生成的序列 $y_{<t}$ 输入 Decoder，并结合 Encoder 的 memory 计算输出，仅取最后一个位置的 hidden state out[:, -1]，再经过 Generator 映射到词表空间，得到下一个 token 的条件概率分布 \(p(y_t\mid y_{<t},x)\)。

3. Greedy Decoding 每一步直接选择概率最大的 token，即 \(y_t=\arg\max_y p(y\mid y_{<t},x)\)，并将其拼接到当前序列中，重复上述过程直到生成 <EOS> 或达到最大长度。

4. 与训练阶段不同，训练时由于真实 target 已知，可以通过 teacher forcing 和 causal mask 并行计算所有位置；而推理阶段未来 token 尚未生成，因此必须逐步串行解码。

5. Annotated Transformer 的教学实现每一步都会重新计算完整的 target prefix，而实际高性能 Transformer 通常使用 KV Cache 保存历史 Key/Value，避免重复计算.


