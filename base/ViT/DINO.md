## 现代主流 ViT

### 计划*

| 路线                        | 代表                         | 你要抓的核心                                                         |
| ------------------------- | -------------------------- | -------------------------------------------------------------- |
| 基础架构                      | ViT                        | image → patch tokens → Transformer                             |
| 层级 / 局部结构                 | Swin                       | window attention、shifted window、hierarchical feature map       |
| Masked visual modeling    | BEiT / MAE                 | 把 BERT-style masking 搬到视觉；重建被遮挡信息                              |
| Self-distillation SSL     | DINO → DINOv2 → **DINOv3** | 不依赖标签，让 ViT 学出通用语义和 dense features                             |
| Image-text representation | CLIP / SigLIP2             | 图像表示与语言语义对齐，VLM/VLA 很常见                                        |
| 视频 / 世界模型                 | V-JEPA2                    | 不重建 pixel，而是在 latent space 预测未来状态，和你做 embodied/world model 很接近 |

重心放在 DINO 系列，然后 MAE，再迅速看 SigLIP2 / V-JEPA2；Swin 只需要知道思想。

## DINO 系列（SSL）

### 1. 总体脉络：
```
对比式学习
MoCo-v3
    ↓
非对比式 / Siamese 学习
BYOL、SimSiam
    ↓
自蒸馏
DINO
    ↓
Patch-level / Masked Prediction
iBOT
    ↓
大规模视觉 Foundation SSL
DINOv2、CAPI、DINOv3
```

这些方法虽然具体 loss 和训练策略不同，但逐渐形成了一套比较统一的现代视觉自监督框架：
\[
\boxed{
\text{Multi-view Augmentation}
+
\text{Teacher-Student}
+
\text{Stop-Gradient}
+
\text{稳定 Target}
+
\text{Anti-Collapse}
+
\text{Global + Patch-level Representation}
}
\]

### 2. 核心思想
1. Multi-view Augmentation：对同一张图像进行多种，改变图像外观和局部观察范围（不改变图像的主要语义），让模型学习对数据增强不敏感的语义表征。
2. Teacher-Student：用 Teacher 的输出作为 Student 的 Pseudo Label，让模型本身产生训练目标。
3. Stop-Gradient：Teacher 的参数不参与梯度下降，避免 Loss collapse。一般使用 EMA 更新 Teacher 参数。
4. 动量更新/指数滑动平均（EMA）：
    * $\theta_t = m \theta_{t-1} + (1-m) \theta_s$
    * EMA 很常见，但不是理论上的必要条件。SimSiam 不使用 EMA，仅靠 predictor 和 stop-gradient 也能训练成功。

### 3. 对比式与非对比式 SSL

* 对比式：以 MoCo-v3 为代表，同时存在正负样本，通过将两者的相似度最大化/最小化来训练模型。
* 非对比式：以 BYOL、SimSiam、DINO、iBOT 为代表，不显式使用负样本，而是关心 Student Prediction 与 Teacher Representation 的相似度。

### 4. 从 Embedding Matching 到 Distribution Matching

* MoCo：识别同一实例（Sample-level discrimination）
* BYOL：直接匹配 Embedding（Embedding-level matching）
* DINO：将 Teacher 和 Student 的输出由 Softmax 映射为概率分布，通过交叉熵匹配两个分布（Distribution-level matching）
* iBOT：在 DINO 的基础上增加了 MIM（Masked Image Modeling）技术，让模型在 patch-level 上也能学习到语义表征

### 5. CLS 与 Patch-level Representation

* DINO 更偏向对 CLS token 做全局 self-distillation，这主要强化 Global Sematic Representation。
* iBOT 在 DINO 的基础上加入 MIM，让模型在 patch-level 上也能学习到语义表征。

### 6. Vision Transformers Need Registers（2024年4月12日）

该论文发现，**大规模、长时间**训练后的 ViT 会在少量低信息 patch 上产生高范数异常 token，这些 token 不再保留原本的局部位置和像素信息，而被模型“回收”成存储全局信息的临时工作区，从而造成特征图伪影并影响 dense prediction。作者因此在 patch token 之外显式加入若干可学习的 register tokens，让模型把这类全局中间信息存到专门的 token 中，而不是占用 patch token；训练结束后丢弃 registers，只保留 CLS 和 patch features。这样可以显著减少伪影，使局部特征更平滑、空间语义更干净，并改善目标定位等密集视觉任务。

### 参考

* [万字长文超详解读之DINO全系列—视觉表征对比学习的高峰](https://zhuanlan.zhihu.com/p/1933583851923439816)

## DINOv3



### 参考

* [万字长文超详解之DINO-V3（DINO全系列之补充篇）](https://zhuanlan.zhihu.com/p/1940400858836742367)
