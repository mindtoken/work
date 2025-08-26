<link rel="stylesheet" type="text/css" href="../css/auto-number-title.css" />

# flash_rl

Your Efficient RL Framework Secretly Brings You Off-Policy RL Training

问题：
- [Q: Can the output of a prompt vary across runs in vLLM?](https://docs.vllm.ai/en/latest/usage/faq.html)： batching changes...vLLM does not guarantee stable log probabilities (logprobs) for the output tokens. 但从RL角度看，batch变化不大。
- [PyTorch Numerical accuracy](https://docs.pytorch.org/docs/stable/notes/numerical_accuracy.html)： Batched computations or slice computations,SPDA
- MiniMax: Computational Precision Mismatch in Generation and Training. This discrepancy arose from a precision mismatch between the training and inference kernels. The issue was detrimental and prevented reward growth in our experiments. Interestingly, this issue
did not appear in smaller, dense models with softmax attention. Through layer-by-layer analysis, we identified high-magnitude activations in the LM head at the output layer as the primary source of error. To address this, we increased the precision of the LM output head to FP32


## REINFORCE algorithm

$$
\theta \gets \theta + \mu \cdot  \mathbb{E}_{\underbrace{a \sim{\pi}(\theta)}_{rollout}} [R(a)\cdot \underbrace{\nabla_\theta \log {\pi}(a, \theta)}_{\tiny{training}}].
$$


**符号说明**

- $\theta$：策略网络的参数（例如神经网络权重）。
- $\pi(a|\theta)$：策略函数，表示在状态 $s$ 下选择动作 $a$ 的概率，依赖于参数 $\theta$。
- $R(a)$：动作 $a$ 的回报（reward），可以是即时奖励或累积回报（如GAE等）。
- $\nabla_\theta \log \pi(a|\theta)$：策略对数似然的梯度，也称为“得分函数（score function）”。
- $\mathbb{E}_{a \sim \pi(\theta)}[\cdot]$：期望值，表示对从当前策略 $\pi(\theta)$ 中采样的动作 $a$ 求平均。
- $\mu$：学习率（step size），控制更新步长。

---

### **公式的含义**

这个公式表示：

> 将策略参数 $\theta$ 沿着期望回报的梯度方向进行更新，以最大化长期回报。
> 这个公式通过“回报 × 对数策略梯度”的期望来更新策略参数，使策略倾向于选择带来更高回报的动作。

具体来说：

- $\log \pi(a|\theta)$ 是动作 $a$ 的对数概率。
- $\nabla_\theta \log \pi(a|\theta)$ 是该对数概率关于参数 $\theta$ 的梯度。
- $R(a)$ 是动作 $a$ 带来的回报（比如累计折扣回报）。
- 整个表达式 $R(a) \cdot \nabla_\theta \log \pi(a|\theta)$ 是“**回报加权的策略梯度**”。

通过取期望 $\mathbb{E}_{a \sim \pi(\theta)}$，我们得到了一个无偏的梯度估计。

---

### **为什么这样更新？**

这是基于**策略梯度定理（Policy Gradient Theorem）**，其核心思想是：

> 策略的期望回报 $J(\theta) = \mathbb{E}_{\tau \sim \pi(\theta)}[R(\tau)]$ 对 $\theta$ 的梯度为：
>
> $$
> \nabla_\theta J(\theta) = \mathbb{E}_{a \sim \pi(\theta)}[R(a) \cdot \nabla_\theta \log \pi(a|\theta)]
> $$

因此，为了最大化 $J(\theta)$，我们使用梯度上升法：

$$
\theta \leftarrow \theta + \mu \cdot \nabla_\theta J(\theta)
$$

即上述公式。


### **实际应用中的变体**

在实践中，这个基本公式有多种改进版本，例如：

- **REINFORCE**：使用蒙特卡洛回报 $R(a)$（整个轨迹的总回报）。
- **Actor-Critic**：用价值函数 $V(s)$ 或优势函数 $A(s,a)$ 替代 $R(a)$，减少方差。
- **PPO, TRPO**：加入约束防止更新过大。

例如，更常见的形式是：

$$
\theta \leftarrow \theta + \mu \cdot \mathbb{E}_{a \sim \pi(\theta)}[A(s,a) \cdot \nabla_\theta \log \pi(a|\theta)]
$$

其中 $A(s,a)$ 是优势函数，表示动作 $a$ 相对于平均表现的优劣。

## Mismatch
As shown in Figure 1, despite  $\textcolor{blue}{\pi_{\text{fsdp}}}$ and $\textcolor{red}{\pi_{\text{vllm}}}$ sharing the same model parameters $\theta$, they can produce significantly different token probabilities. For certain tokens $a$, they even yield contradictory predictions —  $\textcolor{red}{\pi_{\text{vllm}}}(a, \theta)\!=\!1$ and $\textcolor{blue}{\pi_{\text{fsdp}}}(a, \theta)\!=\!0$. This unexpected behavior implicitly breaks the on-policy assumption, secretly making the RL training become off-policy.


system-level mismatch
- forces vLLM to return the actual probabilities used for sampling
-  provides the option to force vLLM casting lm_head to fp32.
  
algorithm-level fix: truncated importance sampling
$$\mathbb{E}_{a \sim \textcolor{red}{\pi_{\text{vllm}}}(\theta)} [R(a)\cdot \nabla_\theta \log \textcolor{blue}{\pi_{\text{fsdp}}}(a, \theta)]$$
->
$$\mathbb{E}_{a \sim \textcolor{red}{\pi_{\text{vllm}}}(\theta)} \Bigl[\underbrace{\min\Bigl(\frac{\textcolor{blue}{\pi_{\text{fsdp}}}(a, \theta)}{\textcolor{red}{\pi_{\text{vllm}}}(a, \theta)}, C\Bigr)}_{\text{truncated importance ratio}} \cdot R(a) \cdot \nabla_\theta \log \textcolor{blue}{\pi_{\text{fsdp}}}(a, \theta)\Bigr]$$
where C is a hyper parameter.

### Extension to General Case: PPO
To improve throughput, hybrid RL systems adopt vLLM engine for rollout generation —sampling tokens $a$ from $\pi_{\theta_{\text{old}}}$, while using FSDP backend both to sample from $\pi_\theta$ and to recompute the token probabilities for $\pi_{\theta_{\mathrm{old}}}$ for gradient computation: 
$$
\small{
\mathbb{E}_{a\sim\textcolor{red}{\pi_{\text{vllm}}}(\theta_{\mathrm{old}})}
\Bigl[
\nabla_\theta \min\Bigl(
\frac{\textcolor{blue}{\pi_{\text{fsdp}}}(a, \theta)}{\textcolor{blue}{\pi_{\text{fsdp}}}(a, \theta_{\mathrm{old}})}\,\hat A,
\;\mathrm{clip}\bigl(\frac{\textcolor{blue}{\pi_{\text{fsdp}}}(a, \theta)}{\textcolor{blue}{\pi_{\text{fsdp}}}(a, \theta_{\mathrm{old}})},\,1-\epsilon,\,1+\epsilon\bigr)\,\hat A
\Bigr)
\Bigr]
}
$$
->
Similar to the analysis above, the gap between $\textcolor{blue}{\pi_{\text{fsdp}}}$ and $\textcolor{red}{\pi_{\text{vllm}}}$ shows up again, and we fix it with truncated importance sampling:
$$
\small{\mathbb{E}_{a\sim\textcolor{red}{\pi_{\mathrm{vllm}}}(\theta_{\mathrm{old}})}\Bigl[\underbrace{\min\Bigl(  \frac{\textcolor{blue}{\pi_{\mathrm{fsdp}}}(a,\theta_{\mathrm{old}})}{\textcolor{red}{\pi_{\mathrm{vllm}}}(a,\theta_{\mathrm{old}})},  C\Bigr)}_{\text{truncated importance ratio}}\cdot\nabla_{\theta}\,\min\Bigl(  \frac{\textcolor{blue}{\pi_{\mathrm{fsdp}}}(a,\;\theta)}{\textcolor{blue}{\pi_{\mathrm{fsdp}}}(a,\;\theta_{\mathrm{old}})}\,\hat{A},  \mathrm{clip}\Bigl(    \frac{\textcolor{blue}{\pi_{\mathrm{fsdp}}}(a,\;\theta)}{\textcolor{blue}{\pi_{\mathrm{fsdp}}}(a,\;\theta_{\mathrm{old}})},    1-\epsilon,\;1+\epsilon  \Bigr)\,\hat{A}\Bigr)\Bigr]}
$$
where C is a hyper-parameter.

Decoupled PPO is a special case of using importance sampling to bridge the gap between the rollout generation and gradient computation, which has been adopted in async-RL frameworks like AReaL. It is worth mentioning that AReaL didn’t implement the truncated importance ratio as we discussed here. Instead, AReaL will drop the training sample entirely, if the importance ratio exceeds a pre-defined threshold. 

### Does TIS always help? 
- The gap can be amplified in MoE RL: Dynamic Routing, Specially Optimized Kernels.
- TIS is orthogonal and compatible with existing GxPOs

