# World Models for Robot Learning: A Comprehensive Survey 2026.4.30

## 两个维度WM

- Action-Conditioned WM
    
    - 三个核心能力：foresight、imagination-driven planning、data amplification
        
- Video Generative Model
    
    - 主要是生成视频以学习时空规律
        

两者的根本区别在于WM的优化目标是否由action 主导。

本文对世界模型的定义，不只是能预测一个合理的未来，而是预测未来在机器人相关行动下如何变化，以支持具身决策。所以本文的世界模型不包括广义下的视频生成模型（优化目标不以action为主导）。

但这不表示可以生成视频的 WM 不是 ACWM，由于当前具身系统中最常见且可扩展的状态实现是 observation stream，所以基于未来观测定义的视频生成模型也属于 ACWM，且在该领域意义重大。

## 世界模型的本质

世界模型本质上是一个未来轨迹（观测+动作）的联合分布：

$p(o_{t+1:t+k},a_{t+1:t+k} | o_t,l)$

即通过当前时间步的观测 和 语言/任务描述，生成一个**未来观测的中间潜在变量** 和 一系列动作表示。

- Policy Model：$p(a_{t+1:t+k} | o_t,l)=\int p(o_{t+1:t+k},a_{t+1:t+k} | o_t,l)d_o$
    
- Passive World Model：$p(o_{t+1:t+k} | o_t,l)=\int p(o_{t+1:t+k},a_{t+1:t+k} | o_t,l)d_a$
    
- Controllable World Model：$p(o_{t+1:t+k} | o_t,a_{t+1:t+k})$
    
- Inverse Dynamics Model：$p(a_{t+1:t+k}|o_{t:t+k})$
    

比如，策略模型就是边缘化了中间的未来观测，主要去使用动作表示；逆动力学模型则是解码未来观测，反推动作。

## 机器人policy

> **visuomotor policies vs. VLA models**
> 
> 我认为VLA必须要存在有一种结构化中间状态（如世界模型、对象中心表示方式、分层策略等）来支持稳定的空间表示、因果预测和规划能力（非端到端），才能更加泛化。
> 
> 端到端VLA的所谓泛化本质上只是前者（Visuomotor Policies）的从观察到行为的映射规模扩大，没有实现真正的泛化（详见LIBERO-PLUS）。
> 
> 那么**为什么要引入世界模型？**
> 
> 端到端VLA直接将观测+描述映射为动作，所以会有弱 allocentric representation 的缺陷，世界模型可以维护一个 latent future observation 的中间状态，一定程度可以弥补这方面的缺陷增强泛化能力。
> 
> “**Instead of learning a direct, monolithic mapping from the current observation to actions**, the model reasons about future observations as auxiliary predictive variables that inform or constrain action selection.” ([Hou 等, 2026, p. 7](zotero://select/library/items/CSDL3ZVZ)) ([pdf](zotero://open-pdf/library/items/PYW5GXV2?page=7))

将世界模型集成进机器人policy主要有两大方向，文中称为 **WM for policy（紧密耦合预测模型和行动生成）** 和 **WM for simulator（世界模型作为验证、训练后和RL模拟器）**，详见 Figure 2.

## WM for policy

目前的趋势是从**解耦的 predict-then-act 流水线**迁移至**联合的预测控制模式。**

- **解耦方式（IDM-style）**：未来预测和动作生成由两个不同的模块实现。主流方法是先使用一个图像生成模型或VGM预测未来观测序列，再训练一个单独的策略模块（可能使用IDM），以推断可执行动作
    
- **联合方式（Single-Backbone-Style）**：使用单一生成骨干（大部分使用 VGM）来联合建模未来的视觉变化和行动。主要思想是从带噪声的输入（视觉、动作 latent 表示）中重建真实未来并生成动作。融合方式类似于使用 VLM 等 backbone 的 VLA 方法，将骨干模型换成了 VGM，详见综述 Vision-Language-Action Models for Robotics: A Review Towards Real-World Applications（2025.8）
    
- **MoT-Style**：多分支设计，专家之间通过共享注意力、交叉注意力或交错自回归序列进行交互。这种方法会把视频分支充当时间预测的潜在流，并将其预测注入动作分支获得策略，这和完全独立的下游动作头获得策略不同。（主要分三类，见12-13页）
    

MoT方式相当于解耦与联合方式的折中方案。比起解耦方式，MoT通过联合层（共享、交错注意力等方式）将视频分支的预测流注入动作分支，从而产生联系；比起联合方式，MoT**没有将参数在层间完全传递，而是通过联合层让专家进行互动（信息注入），保持了专家的参数特化(参数分工）**。

### Unified VLA Models

没有采用明确的世界模型，但模型中也学习了面向未来的预测结构，如 future-image prediction, visual foresight, or structured world knowledge。

与上述 VGA 为主干的模型不同，它们**对未来的建模是内化在模型中，而不是通过一个单独的预测模块引入的**。

> 与 Vision-Language-Action Models for Robotics: A Review Towards Real-World Applications（2025.8）不同，该论文将带 WM 的 VLA（WM for Policy）与 VLA 严格分类了。区分的**核心在于其中是否有显式用于预测的模块。**

1. 端到端：直接基于明确图像或视频进行未来预测
    
2. 非端到端：用隐式或潜在未来建模代替像素级预测
    
3. 由多专家或多系统（multi-expert or multi-system unified models）组成：训练和任务级别保持统一，但在架构内保留明确的功能特化。

### Policies with Latent-Space World Modeling

这部分说明了 WM for policy 的趋势：把未来预测主要放在表示空间完成，而不依赖于显式图像或视频生成。不再将重心放在像素级的表示和预测上，而是更加关注在潜在空间中学习未来感知的表示，该表示捕获环境如何以直接用于控制的形式演变。

因此，此类方法通过将预测结构注入动作生成来保留世界建模的核心优势，同时避免显式生成解码的计算开销和冗余。

除了 neural latent representations，还出现了一种方法，即将世界建模外化为谓词、对象关系、可供性、运算符或因果过程的抽象转换模型，然后由符号或任务和运动规划器查询以产生高级技能序列。

## WM as Simulator

世界模型不只是能模拟未来的演化，还可以代表环境本身：给定当前的观察、任务指令和候选动作，该模型可以推出未来状态，提供反馈信号，并通过想象的交互支持下游决策。

* 作为强化学习的学习模拟器
* 作为决策时验证的评估器

## Benchmarks, Datasets, and Results

### Benchmarks

机器人领域评估世界模型的关键在于，**世界模型是否能生成与真是物理动力学保持一致的动作条件未来状态**。

> 论文强调：评估世界模型的核心在于它是否能生成与真实物理动力学一致的动作条件未来状态，**而不仅仅是视觉预测的合理性。**

Benchmarks 可以分为三类：
* **动作条件生成和开环预测质量**：给定当前的观察结果以及动作序列、语言指令或任务规范，要求模型生成未来的观察结果，而无需嵌入到规划器或控制循环中。（预测合理性）
* **闭环任务效用和策略评估**：开环基准测试评估世界模型是否可以生成以动作为条件的未来，而闭环基准测试则询问这些预测在交互式决策循环中是否仍然有用。（决策效用）
* **物理一致性**：预测的未来是否保留执行所需的物理和行动相关结构，包括动态的一致性、对行动干预的响应性以及有效控制信号的可恢复性。

### Datasets

数据集的价值不仅仅由规模来定义，而是由它是否提供足够丰富的动作条件转换、长视野任务结构、跨场景和实施例的多样性以及与操作相关的物理信号的覆盖范围来定义。

维度：general trajectory pretraining, long-horizon modeling, cross-embodiment scaling, human-prior transfer, contact- and physics-aware modeling, and synthetic or recipe-driven data scaling

## 挑战

1. 因果错位
2. 效率评价
3. 多模态感知瓶颈
4. 经典控制集成
5. 符号结构整合
6. 评估指标的开放性

## 说明

关于 WM for Robotic VG，以及其他应用的部分我没有细看，主要关注了 WM for policy 的部分。

Benchmarks, Datasets, and Results 和挑战是略读。

## 思考

1. WM 在 VLA 中究竟带来了什么？

我原本以为 WM 主要是提供了一个稳定的空间表征，弥补了 VLA 中 allocentric representation 的缺陷，但从综述来看，现在的主流方法基本都有一层 latent spatial representation。

WM 的核心优势其实在于额外提供了一个 **predictive structure（可用于时间演化的结构）**，能够在未来的观测和动作之间建立联系，从而支持更好的因果预测和规划能力。

2. latent world model 是 representation learning 还是 planning substitute？

latent world model 本质上是 representation learning，但它之所以存在是为了引入可用于控制的未来结构，因此在不同研究中既可以被视为 planning substitute（model-based RL），也可以被视为 representation learning with predictive inductive bias（modern VLA / JEPA），具体取决于是否显式用于 rollout decision making。

**新的问题与下一步**：

进一步思考，如果 VLA 已经有 latent state，WM 还在干嘛？这部分我决定深入阅读几篇 WM 和 VLA 相关的论文，再来看这个问题。围绕三个主线：

1. latent state is what?
encoder feature?
token space?
object-centric?
2. is there explicit temporal constraint?
transition model?
predictive loss?
JEPA-style target?
3. does it affect action selection directly?
rollout planning?
or only representation shaping?

同时，最近关注到两篇英伟达的论文：longlive 和 locateAnything3D。似乎是VGM和VLM。主要解决的是实时性和资源占用的问题，而非 performance，以解决实际部署中的问题，引起了我的兴趣，后续准备看看这两篇论文。
