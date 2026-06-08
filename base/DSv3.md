# DeepSeek-v3_MLA机制

## DSv3主要创新

- 结构为：MLA + DeepSeekMoE（671B）

1. MLA（Miti-Latent Attention）
2. DeepSeekMoE
3. 辅助无损负载均衡（auxiliary-loss-free load balancing）
4. 多标记预测训练（multi-token prediction training）

## MHA（Multi-Head Attention）

### RMS（Root Mean Square） Norm

原始的 Transformer 使用 LayerNorm（post-norm）。LN 的作用是将样本分布标准化，使其具有零均值和单位方差。LN 的公式如下：

$$
\text{LayerNorm}(x) = \frac{x - \mu}{\sqrt{\sigma^2 + \epsilon}} \cdot \gamma + \beta
$$

RMSNorm（pre-norm） 是一种替代 LayerNorm 的方法，它只计算均方根（RMS）而不进行中心化。RMSNorm 的公式如下：

$$
\text{RMSNorm}(x) = \frac{x}{\sqrt{\frac{1}{d} \sum_{i=1}^{d} x_i^2 + \epsilon}} \cdot \gamma
$$

抽象来说，RMSNorm 只是对输入进行缩放，不改变均值，只统一向量的长度。其效果等价甚至优于 LayerNorm，且计算更简单。

为什么会这样？因为在 Transformer 中，样本分布的均值会漂移，hidden state 可以看成 $ x=Mean Shift+Direction Component $，其中 Direction Component 是主要信息载体。这样看来 LN 统一均值的意义不大，而RMSNorm 只关注方向成分，忽略均值成分，因此会有上述效果。

## MLA（Miti-Latent Attention）

核心在于通过低秩联合压缩来减少注意力键（keys）和值（values）在推理过程中的缓存，从而提高推理效率。

> MHA, MQA, GQA, MLA

流程：
1. 处理 Quary 标记：

$$ c_t^Q = W^{DQ} h_t $$

$$ [q^C_1; q^C_2; ...; q^C_{n_h}] = q^C_t = W^{UQ} c_t^Q $$

$$ [q^R_1; q^R_2; ...; q^R_{n_h}] = q^R_t = RoPE(W^{QR}c_t^Q) $$

$$ q_{t,i} = [q^C_{t,i}; q^R_{t,i}] $$

2. 处理 Key tokens：

$$ \mathbf{c_t^{KV}} = W^{DKV} h_t $$

$$ [k^C_{t,1}; k^C_{t,2}; ...; k^C_{t,n_h}] = k^C_t = W^{UKV} c_t^{KV} $$

3. 为 RoPE 使用额外的共享 Key：

$$ \mathbf{k^R_{t}} = RoPE(W^{KR} h_t) $$

$$ k_{t,i} = [k^C_{t,i}; k^R_{t}] $$

4. 处理 Value tokens：

$$ [v^C_{t,1}; v^C_{t,2}; ...; v^C_{t,n_h}] = v^C_t = W^{UV} c_t^{KV} $$

5. 计算注意力输出：

$$ o_{t,i} = \sum_{j=1}^{t} \text{softmax}_j\left(\frac{q_{t,i}^T k_{j,i}}{\sqrt{d_h+d^R_h}}\right) v^C_{j,i} $$

$$ u_t = W^O [o_{t,1}; o_{t,2}; ...; o_{t,n_h}] $$

## 参考
- [【知乎】一文搞懂DeepSeek-V3_MLA注意力机制](https://zhuanlan.zhihu.com/p/24040056236)
- [【知乎】DeepSeek-V3 关键点解读-架构篇](https://zhuanlan.zhihu.com/p/15057106396)
