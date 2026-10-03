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

