# Transformer

---

## Transformer Architecture

1. Embedding：文本部分被分为 token，这些 token 会被转换为称为 embedding 的向量表示，embedding 包含了 token 的语义信息和位置信息。
   - **Embedding** -> Tokenization -> Token Embedding -> Positional Encoding(主流 RoPE) -> **Final Embedding**
2. Transformer Block
   - Multi-Head Self-Attention：允许 token 和其他 token 进行交互，捕捉上下文信息。
     - Step 1: Query, Key, and Value Matrices
     - Step 2: Multi-Head Splitting
     - Step 3: Masked Self-Attention
     - Step 4: Output
   - MLP: Multi-Layer Perceptron
3. Output Probabilities：最终的线性层和 softmax 层将处理后的嵌入转换为概率，使模型能够对序列中的下一个 token 进行预测。

## Auxiliary Architectural Features

- Layer Normalization：稳定训练并帮助模型更快地收敛。
- Dropout：通过随机禁用神经元来防止过拟合。
- Residual Connections：允许梯度直接通过网络，并有助于防止梯度消失问题。

## Thinking

1. FFN/MLP的作用是什么？

本质上 FFN 是对信息进行了一次升维和降维的过程，如果只是单纯的线性变换，只是对于原始信息的重排，但加入非线性变换之后，是基于已有信息的基础上进行了一次新的信息提取和重组，能够捕捉到更复杂的模式和关系。

总的来说，Attention 部分机制的核心功能是建模 token 之间的依赖关系，并进行信息聚合；FFN 部分对每个token的表示进行独立的、更深层次的变换。

2. 位置编码与 RoPE

## Flash Attention

- 核心原理：将输入 QKV 分块，并保证每个块能够在 SRAM（一级缓存）上完成注意力操作，并将结果更新回 HBM，从而降低对高带宽内存（HBM）的读写操作。
- safe softmax：在计算 softmax 时，先找到每个块的最大值，并将其从每个块中减去，以避免数值不稳定性。
- **online softmax**：在计算 softmax 时，保持一个全局的最大值，并在每个块中更新它，以确保数值稳定性。

## KV Cache

- 核心思想：推理阶段历史 token 不变，所以不再重复让历史 token 的 KV 参与计算，而是缓存起来。
- KV Cache 在上下文过长时，会占用大量内存，导致性能下降。

## MoE

- 替换 FFN 为 **门控网络或路由**与**稀疏 MoE 层**，路由决定哪些 token 被发送到哪些专家 FFN。
- 简单总结

  - 本质上是同等训练成本换取更大的模型容量。
  - 与稠密模型相比，**预训练速度更快**
  - 与具有相同参数数量的模型相比，具有更快的**推理速度**
  - 需要**大量显存**，因为所有专家系统都需要加载到内存中
  - 在**微调方面**存在诸多挑战，但[近期的研究](https://arxiv.org/pdf/2305.14705)表明，对混合专家模型进行**指令调优**具有很大的潜力。
- Top-k MoE(一般 k 大于等于2）：每个 token 只被路由到 k 个专家系统中，减少了计算量和内存占用。
- 负载均衡：确保每个专家系统处理的 token 数量相对均衡，以避免某些专家系统过载而其他专家系统闲置。

  - 随机路由：在 Top-2 设置中，我们始终选择排名最高的专家，但第二个专家是根据其权重比例随机选择的。
  - 专家容量：$ \text{Expert Capacity}= \frac{\text{tokens per batch} }{\text{number of experts}} \times \text{capacity factor} $
- Switch Transformers

  - Switch Transformer 层：接收两个输入 (两个不同的 token) 并拥有四个专家。且采用 top-1 路由机制。
  - 辅助损失：在训练过程中，添加一个辅助损失项来鼓励路由器在专家之间进行负载均衡。这个损失项计算每个专家系统处理的 token 数量，并惩罚那些处理过多或过少 token 的专家系统，以促进更均衡的负载分配。
  - 混合精度：仅对专家使用 bf16 混合精度训练，以减少内存占用和加速训练过程。
  - 一些结论
    - 在相同的预训练困惑度下，稀疏模型在下游任务中的表现不如对应的稠密模型，特别是在重理解任务 (如 SuperGLUE) 上。
    - 对于知识密集型任务 (如 TriviaQA)，稀疏模型的表现很好。
    - 在微调过程中，较少的专家的数量有助于改善性能。
    - 泛化性：模型在小型任务上表现较差，但在大型任务上表现良好。
- Router z-loss
- MoE 训练

  - 预训练：增加更多专家可以提升处理样本的效率和加速模型的运算速度，但这些优势随着专家数量的增加而递减。
  - 微调
    - 稀疏模型更易于出现过拟合现象，因此在处理这些模型时，尝试更强的内部正则化措施是有益的，比如使用更高比例的 dropout。
    - 辅助损失：关闭辅助损失后，token 被丢弃没有对模型性能产生显著影响，这代表丢弃 token 也许是一种有效的正则化方法，能防止过拟合。
    - 冻结非专家层权重的微调效果和完全微调的效果相当，且能减少训练时间和计算资源的消耗。
    - 稀疏模型往往更适合使用较小的批量大小和较高的学习率，这样可以获得更好的训练效果。

## SFT

SFT (Supervised Fine-Tuning) 是一种**微调**方法，使用带有标签的数据集来训练模型，使其在特定任务上表现更好。SFT 的核心思想是通过**监督学习**的方式，让模型学习从输入到输出的映射关系，从而提高模型在特定任务上的性能。

通常来说 SFT 比模型训练成本更低。

### 关键技术点
1. Teacher Forcing
• 训练时将前一步的真实 token 作为输入，促使模型快速收敛。
2. Label Smoothing
• 在标签上加小噪声，防止模型过度自信，从而提升泛化能力。
3. 批量梯度 vs. 在线学习
• 对大规模数据可采用小批次（batch）训练；对新鲜反馈可通过在线微调快速落地。

### 关键环节

| 环节 | 输入 | 输出 | 常见工具 |
| --- | --- | --- | --- |
| 数据准备 | 年报 PDF、原始领域文档 | 清洗并格式化的 SFT 数据集 | Apache Tika / PDFPlumber、OCR 清洗、Label Studio |
| 模型选择 | 场景需求 & 资源预算 | 基础模型（如 Qwen-7B、Baichuan-13B、LLaMA2-7B） | Hugging Face Model Hub |
| 配置训练 | 数据集 + 基础模型 | 微调后的领域适配模型 | PyTorch、DeepSpeed、PEFT |
| 验证评估 | 测试集 + 自动化评测用例 | 模型性能报告 | OpenAI Evals、EleutherAI Evals、MLflow |
| 灰度发布 & 在线监控 | 服务请求日志、监控指标 | 实时偏差与延迟报警 | Prometheus、Grafana、Sentry |
| 部署上线 | 训练好的模型 | API 服务或应用 | vLLM、Triton Inference |


## Reference

- [Transformer Visualization: ](https://poloclub.github.io/transformer-explainer/)https://poloclub.github.io/transformer-explainer/
- [Flash Attention: ](https://mp.weixin.qq.com/s/jMgNZ1xpEdMpwLvclFV9Mg)https://mp.weixin.qq.com/s/jMgNZ1xpEdMpwLvclFV9Mg
- [MoE: ](https://huggingface.co/blog/zh/moe)https://huggingface.co/blog/zh/moe
