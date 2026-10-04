## DeiT

DeiT 的问题意识是：原始 ViT 很依赖超大数据集和超大算力，DeiT 试图让 ViT 只用 ImageNet-1K 也能训练好。它最有辨识度的设计是 distillation token。

DIST token：
* CLS token 学真实标签，DIST token 学 teacher 的输出。
* DeiT 的 Teacher 通常是 CNN，DIST token 可以让 ViT 学到 CNN 的 inductive bias。
* 这代表 Transformer 可以给不同 special token 分配不同监督职责。

## MAE

遮掉大量 image patches，只把可见 patch 输入 encoder，再让 decoder 恢复被遮住的像素。

1. encoder 只处理 visible patches。masked token 不进入 encoder，大幅减少计算量。
2. mask ratio 很高，通常 75%。因为自然图像有很强的空间冗余，mask 大部分 patch 能更好地逼迫模型学习语义表征，如果 mask 太少，模型可能只学到低级特征。
3. 和 iBOT 的区别是，MAE 直接重建像素，而 iBOT 重建语义特征。
4. 非对称结构：heavy encoder + light decoder。训练后只保留 encoder，decoder 丢掉。因为 decoder 只是为了训练时的 reconstruction loss，训练后不需要。

## Swin

将全局 self-attention 改成局部窗口 attention，并通过窗口平移实现跨窗口信息交互，同时引入类似 CNN 的层级结构。

1. Window Attention：只在局部窗口内计算 self-attention，而不是所有 patch 两两计算，大幅降低高分辨率图像下的计算量。
2. Shifted Window：相邻 Transformer block 交替使用普通窗口和位移窗口，使原本属于不同窗口的 patch 能在下一层发生信息交互。
3. Hierarchical Structure：通过 Patch Merging 逐层降低空间分辨率、增加通道维度，形成类似 CNN 的多尺度特征：$
   56\times56 \rightarrow 28\times28 \rightarrow 14\times14 \rightarrow 7\times7
   $
4. 和原始 ViT 的区别：ViT 使用全局 attention、基本保持单尺度特征；Swin 使用局部 attention + 层级特征，因此更适合 detection、segmentation 等 dense prediction 任务。
5. 核心理解：Swin 相当于给 ViT 加回了 CNN 的 locality 和 hierarchical inductive bias，同时保留 Transformer 的 attention 机制。

## CLIP

用 image-text pairs 做对比学习，让图像和文本进入同一个语义 embedding space。

```
image → Image Encoder → z_img
                           ↓
                        similarity
                           ↑
text  → Text Encoder  → z_txt
```

* CLIP 的训练方法会自然产生 zero-shot classification 能力、image-text retrieval 能力、open-vocabulary understanding 能力。
* 但 CLIP 本身更偏 global sematics，并非专为几何、局部定位和动态预测设计。

## SigLIP2

在 SigLIP 的 image-text alignment 基础上，进一步加入 self-distillation、masked prediction、captioning pretraining 和 online data curation，使视觉语言表征不仅有全局语义对齐能力，也有更强的 localization 和 dense feature。[arXiv](https://arxiv.org/abs/2502.14786?utm_source=chatgpt.com)

1. 和 CLIP 一样，核心目标仍然是让图像和文本进入对齐的 embedding space，从而支持 zero-shot classification、image-text retrieval 和 open-vocabulary understanding。
2. SigLIP 使用 sigmoid loss，将每个 image-text pair 看成独立的匹配问题，而不是像 CLIP 一样在整个 batch 内做 softmax 对比。
3. SigLIP2 不再只强调 global image-text alignment，而是加入 self-distillation 和 masked prediction，明显增强局部视觉表征和 dense prediction 能力。[arXiv](https://arxiv.org/abs/2502.14786?utm_source=chatgpt.com)
4. 支持 multi-resolution 和保持原始 aspect ratio，更适合作为现代 VLM/VLA 的视觉 encoder。[arXiv](https://arxiv.org/abs/2502.14786?utm_source=chatgpt.com)
5. 核心理解：SigLIP2 可以看成 **CLIP-style language alignment + 更强的 dense/local visual representation**，因此比原始 CLIP 更适合需要空间 grounding 的具身任务。

## JEPA / I-JEPA

JEPA 的核心思想是不重建原始输入，而是在 representation space 中预测被遮挡区域的 latent representation。

```text
context → context encoder → z_context
                         ↓
                     predictor
                         ↓
                   predicted z_target

target → target encoder → z_target
```

1. 和 MAE 最大的区别是：MAE 预测 masked pixels，而 JEPA 预测 masked region 的 latent feature。
2. I-JEPA 从一张图像中的 context block 出发，预测多个 target blocks 的 representation，而不是恢复这些区域的 RGB 像素。[arXiv](https://arxiv.org/abs/2301.08243?utm_source=chatgpt.com)
3. 这样模型不需要花能力预测纹理、颜色噪声等低层细节，而可以更关注 object、structure 和 semantic information。
4. target region 要足够大，context 也要覆盖足够分散的信息，否则预测任务可能过于局部和简单。[arXiv](https://arxiv.org/abs/2301.08243?utm_source=chatgpt.com)
5. 核心理解：JEPA 的思想是 **predict in latent space rather than reconstruct in input space**，即预测“有意义的抽象状态”，而不是还原所有观察细节。

可以把演进关系记成：

$
\text{MAE: pixel prediction}
\rightarrow
\text{iBOT: feature prediction}
\rightarrow
\text{JEPA: structured latent prediction}
$

## V-JEPA

将 JEPA 的 latent prediction 从单张图像扩展到视频，通过预测被遮挡时空区域的 representation 学习运动和动态信息。

1. 输入不再只是 image patches，而是时空 video tokens。
2. 模型根据可见的视频 context，预测被 mask 的时空区域对应的 latent feature，而不是视频像素。[arXiv](https://arxiv.org/abs/2404.08471?utm_source=chatgpt.com)
3. 训练只使用 feature prediction，不需要文本、负样本、pixel reconstruction 或预训练图像 encoder。[arXiv](https://arxiv.org/abs/2404.08471?utm_source=chatgpt.com)
4. 因为目标存在于 representation space，模型更容易忽略不可预测的像素级细节，而学习 motion、object state 和 temporal structure。
5. 核心理解：V-JEPA 开始从静态视觉 representation learning 走向 **动态世界 representation learning**。

## V-JEPA2

在 V-JEPA 的视频 representation learning 基础上进一步规模化，并通过少量机器人交互数据得到 action-conditioned world model，使模型从“理解视频”进一步走向“预测动作后果和规划”。[arXiv](https://arxiv.org/abs/2506.09985?utm_source=chatgpt.com)

1. 第一阶段使用超过百万小时的互联网视频和图像进行 action-free self-supervised pretraining，学习通用视觉和动态 representation，不需要动作标签。[arXiv](https://arxiv.org/abs/2506.09985?utm_source=chatgpt.com)
2. V-JEPA2 本身主要学习：
   $
   \text{visual context}
   \rightarrow
   \text{future / hidden latent representation}
   $
   从大量观察数据中学习世界如何变化。
3. 在机器人数据上进一步 post-train 后得到 **V-JEPA2-AC**，其中 AC 表示 action-conditioned：
   $
   (z_t,a_t)\rightarrow \hat z_{t+1}
   $
   即根据当前 latent state 和 action，预测动作执行后的未来 latent state。[arXiv](https://arxiv.org/abs/2506.09985?utm_source=chatgpt.com)
4. V-JEPA2-AC 只使用不到 62 小时的机器人视频进行 post-training，就可以利用预测出的 future latent state 做 image-goal planning。[arXiv](https://arxiv.org/abs/2506.09985?utm_source=chatgpt.com)
5. 和普通 VLA 的区别是：VLA 通常直接学习
   $
   (vision,language)\rightarrow action
   $
   而 V-JEPA2-AC 显式学习
   $
   (state,action)\rightarrow future\ state
   $
   因此更接近 latent world model。
6. 核心理解：V-JEPA2 展示了一条很重要的具身路线：
   $
   \boxed{
   \text{大量无动作视频学 world representation}
   \rightarrow
   \text{少量机器人数据学 action dynamics}
   \rightarrow
   \text{planning}
   }
   $

