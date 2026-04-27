# Dreamer 2020

## model structure

* CNN + RSSM -> $s_t$
  * CNN：编码器处理原始图像
  * RSSM：
    * 生成 $prior \widehat{s}_t$ 和 $posterior s_t$。其中 $\widehat{s}_t$ 用于 imagination，$s_t$ 用于训练。训练时需要最小化 KL 散度 $D_{KL}( q(s_t | o_t, s_{t-1}, a_{t-1}) \;\|\; p(s_t | s_{t-1}, a_{t-1}) )$，使 prior distribution 与 posterior distribution 对齐，使 prior 可以在无观测的情况下稳定进行 long-horizon rollout。
    * $prior \widehat{s}_t$：$$ p(s_t | s_{t-1}, a_{t-1}) $$
    * $posterior s_t$：$$ q(s_t | o_t, s_{t-1}, a_{t-1}) $$

* $s_t$ -> observation model -> $\widehat{o}_t$
    * observation model：transposed CNN，作为解码器，将状态 $s_t$ 重建图像 $\widehat{o}_t$。

* $s_t$ -> reward model -> $\widehat{r}_t$
    * reward model：MLP，将状态 $s_t$ 映射为奖励 $\widehat{r}_t$。

* $s_t$ -> value model -> $\widehat{v}_t$（critic）
    * value model：MLP，将状态 $s_t$ 映射为价值 $\widehat{v}_t$。
    * 训练方式：
        * 在 latent space 中 rollout imagined trajectory
        * return 作为监督信号训练 value model（MSE）
    * return 计算公式：
        * $$ \mathrm{V}_{\mathrm{R}}\left(s_{\tau}\right) \doteq \mathrm{E}_{q_{\theta}, q_{\phi}}\left(\sum_{n=\tau}^{t+H} r_{n}\right) （用于消融实验）$$
        * $$ \mathrm{V}_{\mathrm{N}}^{k}\left(s_{\tau}\right) \doteq \mathrm{E}_{q_{\theta}, q_{\phi}}\left(\sum_{n=\tau}^{h-1} \gamma^{n-\tau} r_{n}+\gamma^{h-\tau} v_{\psi}\left(s_{h}\right)\right) \quad \text { with } \quad h=\min (\tau+k, t+H) $$
        * $$ \mathrm{V}_{\lambda}\left(s_{\tau}\right) \doteq(1-\lambda) \sum_{n=1}^{H-1} \lambda^{n-1} \mathrm{V}_{\mathrm{N}}^{n}\left(s_{\tau}\right)+\lambda^{H-1} V_N^k(s_{\tau}) $$

* $s_t$ -> policy -> $a_t$（actor）
    * policy：MLP，基于状态 $s_t$ 生成动作 $a_t$。
    * 训练方式：
        * imagination rollout 过程中，使用 reward model + value model 计算 return
        * 最大化 expected return，反向传播更新 policy

## training pipeline

1. environment interaction：agent 与环境交互，收集数据 $(o_t, a_t, r_t)$。
2. dynamics learning：使用收集的数据训练 CNN + RSSM，最小化重建误差和 KL 散度。
3. behavior learning：基于 latent world model，在 latent space 中进行 imagination rollout，使用 reward model 和 value model 计算 return，训练 policy 最大化 expected return。
4. repeat：重复上述步骤，持续改进模型和策略。

## Objective / Loss view
* World Model:
    * reconstruction loss
    * reward prediction loss
    * KL (posterior vs prior)
* Value:
    * λ-return regression
* Policy:
    * maximize imagined return

## data flow vs gradient flow
* data flow：env → replay buffer → model → imagination → policy
* losses -> update:
  * CNN + RSSM
  * reward model
  * value model
  * policy

核心：data flow 和 gradient flow 完全解耦

## design motivation
1. **Why world model（RSSM）**?
* 传统 model-free 方法在高维观测空间（如图像）中效率低下，难以学习有效的策略。
* world model 通过学习环境的动态模型，可以在 latent space 中进行 imagination rollout，提升数据效率和长期规划能力。

2. **Why latent space imagination（RSSM Rollout）**?
* world model 本身只能预测 state，进行 imagination 可以优化 world model。
* 直接在高维观测空间中进行 imagination 计算成本高，且不稳定。

3. **Why value function（bootstrap）**?
* 如果直接做 rollout，会导致 horizon 很短，rollout 越长误差越大。
* value function 可以提供更稳定的 long-horizon，减少 rollout 的长度，降低 return 的方差。

4. **Why latent space (RSSM instead of pixel)**?
* 在 latent space 中进行 computation 更加高效，因为状态空间的维度通常比观测空间低得多。
* latent space 中的表示更加抽象和语义化，有助于学习更有效的策略。

5. **Why Actor-Critic + Model-based hybrid**?
* 纯 model-based: planning 太慢，模型对误差敏感。
* 纯 actor-critic: 需要大量交互数据，效率低。

6. **Why KL loss is crucial**?
* KL loss 使得 prior distribution 与 posterior distribution 对齐，使得在无观测的情况下，prior distribution 也能稳定进行 imagination rollout。
* 没有 KL loss，prior distribution 可能会偏离 posterior distribution，导致 imagination rollout 的质量下降，影响策略的学习。

## Summary

Dreamer 将强化学习问题重新表述为在一个学习到的 latent world model 中进行可微分的想象优化问题。通过 CNN + RSSM 学习环境的潜在动态模型，agent 能够在 latent space 中进行 long-horizon rollout，从而避免直接与真实环境频繁交互。

在该框架中，reward model 提供即时反馈，value model 负责对长时序回报进行 bootstrap 估计，policy 则通过在 imagined trajectories 上最大化 expected return 进行优化。KL 约束保证 prior dynamics 能够在无观测情况下稳定进行长时序预测，从而支撑 imagination rollout。

整体上，Dreamer 的核心是将强化学习从“基于采样的环境交互优化”转化为“基于可微分世界模型的想象空间优化”，实现了 sample-efficient and planning-capable reinforcement learning。
