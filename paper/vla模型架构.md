# VLA 模型架构

此篇笔记来自综述 Vision-Language-Action Models for Robotics: A Review Towards Real-World Applications，综述写于 2025.10.8，请注意时效性。

> 不是严格的分类方法，而是根据模型的核心设计思想进行的分类，实际模型可能会跨越多个类别。
> 几乎所有的 VLA 模型都是感觉运动模型，而世界模型和可供性本身不冲突，它们可以与感觉运动模型结合使用。
> 即 VLA = Sensorimotor + (World Model?) + (Affordance?)

## 感觉运动模型（Sensorimotor Model）

- Transformer + 离散 Action Token
    - 传统的自回归 Transformer 模型
    - VIMA、Gato

- Transformer + diffusion action head
    - 将理解与生成解耦，Transformer 负责理解，Diffusion Head 负责生成**连续动作**
    - Octo、NoMAD、TinyVLA、RoboBERT 和 VidBot

- Diffusion Transformer
    - 强耦合，Transformer 完成理解与生成连续动作
    - RDT-1B、LBM、StructDiffusion、MDT、DexGraspVLA、UVA、FP3、PPL、PPI 和 Dita

- VLM + 离散动作 Token
    - 使用预训练的视觉语言模型（VLM）作为感知模块，输出离散动作 token 进行决策
    - RT-2、LEO、GR-1、RTH、RoboMamba、QUAR-VLA、OpenVLA、LLARA、ECOT、3D-VLA、RoboUniView 和 CoVLA
    
- VLM + Diffusion Action Head
    - 使用 VLM 作为感知模块，扩散动作头输出连续动作
    - Diffusion-VLA、DexVLA、ChatVLA、ObjectVLA、GO-1（AgiBot World Colosseo）、PointVLA、MoLe-VLA、Fis-VLA 和 CronusVLA
    - HybridVLA 混合了自回归和扩散方式
    
- VLM + 流匹配动作头
    - 使用 VLM 进行感知，流匹配动作头生成连续动作
    - π0、GraspVLA、OneTwoVLA、Hume 和 SwitchVLA
    - π0.5 集成了流匹配和离散动作两种方式，在统一的框架内支持离散 token 和流匹配
    
- VLM + Diffusion Transformer
    - 使用 VLM 作为感知模块，Diffusion Transformer 输出连续动作
    - GR00T N1、CogACT、TrackVLA、SmolVLA 和 MinD

> Sensorimotor Model 类别下的架构方法：
> VL（vision-language）理解 = Transformer / VLM
> action generation = AR / diffusion / flow
> VLA 是这些模块的组合

## 世界模型（World Model）
- 基于世界模型的动作生成
    - 世界模型会生成未来的视觉观察，并用此来指导动作生成。即模型持续进行rollout，计算使未来 reward 最大化的动作表示。
    - UniPi、DreamGen、GeVRM、Hip、Dreamitate、SuSIE、CoT-VLA、LUMOS，AVDC、ATM、Track2Act、LangToMo、MindD、PPI
    
- 通过世界模型生成潜在动作🌟🌟🌟
    - 从人类演示中学习潜在动作表示。与上一类别相比，模型直接通过当前的视觉观察生成潜在动作表示，而不是基于未来 reward 最大化等目标进行动作生成。
    - GR00T N1、DreamGen、GO-1、Moto、UniVLA、UniSkill
    
- 具有隐式世界模型的感觉运动模型
    - 联合输出未来观测到的行动和预测，以提高性能的VLA（不是显式世界模型）
    - GR系列、3D-VLA、FLARE、UVA、WorldVLA 和 ViSA-Flow

## 基于可供性的模型（Affordance-based Models）
- 基于 VLM 的可供性预测和行为生成
    - 利用预训练的 VLM 估计可供性并生成相应的操作。
    - VoxPoser、KAGI、LERFTOGO

- 从人类数据集中提取可供性
    - 从人类运动视频中提取可供性（通常没有注释），以实现机器人动作生成的可扩展学习。
    - HRP、VidBot
    
- 感觉运动模型和基于可供性的模型的集成
    - 将可供性预测融入感觉运动模型中，以增强模型表现和泛化能力。
    - CLIPort、RoboPoint、想·RoboGround、RT-Affordance、A0、RoboBrain、Chain-of-Affordance  
