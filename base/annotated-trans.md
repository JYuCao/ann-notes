# The Annotated Transformer

---

## Encoder 

### Step
* Input Embedding -> Positional Encoding -> Multi-Head Attention -> Add & Norm -> Feed Forward -> Add & Norm -> Z，其中 Attention 和 FFN 都为 Sublayer
* $x_{i+1} = LayerNorm(x_i + Dropout(Sublayer(x_i)))$，其中 $i$ 为当前层的索引。

> 1. PostNorm：在残差连接后进行归一化；PreNorm：在残差连接前进行归一化。
> 2. 要求每个 Sublayer 的输出维度与输入维度相同，以便进行残差连接。

### Multi-Head Attention

$$ Z = \text{softmax}(\frac{QK^T}{\sqrt{d_k}})V $$

1. Compute the query, key, and value matrices.
    * $Q = XW^Q$
    * $K = XW^K$
    * $V = XW^V$
2. Compute the attention scores by taking the dot product of the query and key matrices, scaling by the square root of the dimension of the key vectors, and applying the softmax function to obtain a probability distribution over the keys.
    * $ S = \frac{QK^T}{\sqrt{d_k}} $
    * $ A = \text{softmax}(S) $
3. Compute the **memory** representation by multiplying the attention weights with the value matrix.
    * $ Z = AV $

> **为什么要使用多个 Head 而非一个？**<br>
> 答：每个 Head 对不同 token 的 value 进行不同的加权组合（$ Z = AV $），能够捕捉到不同的语义信息。如果只使用一个，则只能捕捉到一种语义信息，无法充分利用输入序列中的多样性。
