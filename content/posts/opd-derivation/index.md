---
title: "从信息论到 On-Policy Distillation：一次完整的推导"
date: 2026-05-25T11:00:00+08:00
lastmod: 2026-05-25T11:00:00+08:00
author: "ZijieGuo"
description: "从自信息出发推导出 OPD 的完整数学结构：信息论 → KL 方向选择 → SFT/OPD 对应的 KL 方向 → reverse KL 经六步代数恒等变形得到 per-token dense reward 的 policy gradient 形式。"
keywords: ["On-Policy Distillation", "OPD", "信息论", "KL 散度", "策略梯度", "反向 KL", "蒸馏", "强化学习"]

tags: ["LLM", "强化学习", "蒸馏", "OPD", "信息论", "数学推导"]
categories: ["技术"]

draft: false

math: true

showToc: true
TocOpen: false
ShowReadingTime: true
ShowCodeCopyButtons: true
ShowShareButtons: true
---

> 本文从自信息出发，推导出 On-Policy Distillation (OPD) 的完整数学结构，说明它本质上是"老师逐 token 打分的 RL"。
>
> 整条链子分为三段：第一段把信息论基础接到 KL 散度的方向选择；第二段说明 SFT 和 OPD 分别对应哪个 KL 方向；第三段从 reverse KL 出发，通过六步代数推出 per-token dense reward 的形式。
>
> 如果你想先看 OPD 的全景（实验结果、工程实现、与 RL/SFT 的对比），可以读 [On-Policy Distillation：从分布视角理解 SFT、RL 与 OPD](../on-policy-distillation/)。

## 一、信息论的复习与 KL 方向的来源

一个事件 $x$ 的**自信息**定义为 $I(x) = -\log p(x)$，衡量事件发生时携带的"惊喜程度"：概率越低，信息量越大。把自信息对整个分布取期望，就得到**信息熵**：
$$
H(X) = -\sum_x p(x)\log p(x) = \mathbb{E}_{x\sim p}[-\log p(x)]
$$
信息熵是用最优编码方案对 $X$ 进行编码时，平均每个事件所需的最少比特数，是任何编码方案的理论下界。

如果不知道真实分布 $p$，而用一个近似分布 $q$ 来设计编码方案（每个事件编码长度为 $-\log q(x)$），但事件实际仍按 $p$ 出现，那么平均编码长度变成**交叉熵**：
$$
H(p, q) = -\sum_x p(x)\log q(x) = \mathbb{E}_{x\sim p}[-\log q(x)]
$$
由于次优编码的存在，$H(p, q) \geq H(p)$，中间的差距就是 **KL 散度**：
$$
D_{KL}(p \,\|\, q) = H(p, q) - H(p) = \sum_x p(x)\log\frac{p(x)}{q(x)} = \mathbb{E}_{x\sim p}\!\left[\log\frac{p(x)}{q(x)}\right]
$$
KL 散度衡量"用 $q$ 替代 $p$ 时引入的额外编码代价"，恒非负，且当且仅当 $p = q$ 时取 0。

到这里有一个非常关键的细节需要强调：**KL 散度的期望是对第一个参数 $p$ 取的，不是对 $q$ 取的**。这意味着把 $p$ 和 $q$ 的位置交换，得到的是一个完全不同的量：

$$
D_{KL}(p \,\|\, q) = \mathbb{E}_{x\sim p}\!\left[\log\frac{p(x)}{q(x)}\right] \quad\text{vs}\quad D_{KL}(q \,\|\, p) = \mathbb{E}_{x\sim q}\!\left[\log\frac{q(x)}{p(x)}\right]
$$

前者称为 **forward KL**（期望对 $p$ 取），后者称为 **reverse KL**（期望对 $q$ 取）。这种不对称性不是数学瑕疵，而是 KL 散度在不同应用场景下行为差异的根源。

具体来说，forward KL 在 $p(x) \gt 0$ 而 $q(x) \to 0$ 的位置会爆炸为 $+\infty$，这迫使 $q$ 必须在 $p$ 有支持的所有地方都保留概率质量——这种行为称为 **mass-covering**，$q$ 倾向变得宽而平坦。反过来，reverse KL 的期望只在 $q$ 有支持的地方计算，所以 $q$ 可以放弃 $p$ 的某些 mode，只锁定其中一个高概率区域——这种行为称为 **mode-seeking**，$q$ 倾向变得窄而锐利。

> 这条性质决定了：**当我们设计蒸馏目标时，选 forward 还是 reverse KL，就已经决定了学生模型最终的"性格"**。

## 二、SFT 与 OPD 对应的 KL 方向

监督微调（SFT）的训练流程是：老师 $\pi_T$ 提供轨迹 $y$（即训练集），让学生 $\pi_s$ 在这些轨迹上最大化 log-likelihood。这等价于最小化：

$$
\mathcal{L}_{\text{SFT}} = -\mathbb{E}_{y\sim\pi_T}[\log\pi_s(y|x)]
$$

对照交叉熵的定义，这恰好是 $H(\pi_T, \pi_s)$。再利用 $H(p, q) = H(p) + D_{KL}(p\,\|\,q)$ 的关系，且老师 $\pi_T$ 固定、$H(\pi_T)$ 与学生参数无关：

$$
\mathcal{L}_{\text{SFT}} = H(\pi_T) + D_{KL}(\pi_T\,\|\,\pi_s) \;\;\Longleftrightarrow\;\; \min_\theta D_{KL}(\pi_T\,\|\,\pi_s)
$$

所以 **SFT 在数学上等价于最小化 forward KL**，期望对老师取，数据从老师采样。其几何后果是学生被迫去 mass-cover 老师的所有轨迹模式，即使有些模式学生根本学不会——这就是分布漂移的根源：学生在训练时只见过老师的轨迹，推理时却要在自己的轨迹分布上工作，二者并不一致。

OPD 把方向反过来，直接最小化 reverse KL：

$$
\mathcal{L}_{\text{OPD}} = D_{KL}(\pi_s\,\|\,\pi_T) = \mathbb{E}_{y\sim\pi_s}\!\big[\log\pi_s(y|x) - \log\pi_T(y|x)\big]
$$

期望对学生取意味着 **$y$ 必须由学生自己 rollout**——on-policy 不是一个工程选择，而是 reverse KL 在数学层面强制要求的。其几何后果是 mode-seeking：学生不需要覆盖老师的全部能力，只需要在自己能走到的轨迹上向老师靠拢。对一个比老师小很多的学生模型，这正是我们想要的——不指望它什么都会，只指望它在自己能做的事上做得像老师。

到这里我们已经看到 SFT 和 OPD 在信息论上的对立：**两者都在做"分布对齐"，但选择的 KL 方向不同，因此采样分布、几何行为和分布漂移性质都完全不同**。

## 三、从 reverse KL 推导到 per-token dense reward

接下来的目标是把 $\min D_{KL}(\pi_s\,\|\,\pi_T)$ 一路化简成"老师逐 token 打分的 policy gradient"。整个过程是六步恒等变形，没有任何近似。

### 第一步：翻号并拆分

最小化变最大化，两边乘 $-1$ 并翻转括号内的符号：
$$
\min_\theta D_{KL}(\pi_s\,\|\,\pi_T) \;\Longleftrightarrow\; \max_\theta\; J(\theta), \quad J(\theta) = \mathbb{E}_{y\sim\pi_s}\!\big[\log\pi_T(y|x) - \log\pi_s(y|x)\big]
$$
将括号内的两项分开写：
$$
J(\theta) = \underbrace{\mathbb{E}_{y\sim\pi_s}[\log\pi_T(y|x)]}_{(A)\;\text{reward 项}} + \underbrace{\mathbb{E}_{y\sim\pi_s}[-\log\pi_s(y|x)]}_{(B)\;=\,H(\pi_s)\;\text{熵正则项}}
$$

注意 (B) 项恰好是香农熵的定义。所以 OPD 的目标函数本质上是：

$$
\boxed{\;\max_\theta\;\; \mathbb{E}_{y\sim\pi_s}[R(y)] + H(\pi_s), \quad R(y) := \log\pi_T(y|x)\;}
$$

这是教科书里 **entropy-regularized RL** 的标准形式（Soft Actor-Critic 那一套），reward 是老师赋予这条轨迹的 log-likelihood，熵 $H(\pi_s)$ 是自动出现的正则项，鼓励学生保持适度的随机性、避免坍缩到单一输出。

#### 插话：为什么 $\mathbb{E}_{y\sim\pi_s}[\log\pi_T(y|x)]$ 可以叫 reward 项？

(A) 项被称为 reward 项不是一种类比，而是形式上和语义上都成立的恒等。要看清这一点，需要从三个层次理解。

**第一层：与标准 RL 目标函数形式上完全一致。**

标准 RL 的目标函数是 $J_{\text{RL}}(\theta) = \mathbb{E}_{y \sim \pi_\theta}[R(y)]$，含义是在当前策略下采样轨迹，然后用 reward 函数 $R(y)$ 给每条轨迹打分。把它和 (A) 项摆在一起：

$$
\underbrace{\mathbb{E}_{y \sim \pi_\theta}[R(y)]}_{\text{标准 RL}} \quad\text{vs}\quad \underbrace{\mathbb{E}_{y \sim \pi_s}[\log \pi_T(y|x)]}_{\text{OPD 的 (A) 项}}
$$

两者结构一模一样：外层是"在当前策略上采样轨迹的期望"，内层是"一个只跟轨迹 $y$ 有关的标量函数"。**只要定义 $R(y) := \log \pi_T(y|x)$，(A) 项就字面意义上是 RL 的 reward 期望项**。这是形式上的恒等，不是类比。

**第二层：老师 log-prob 在语义上就是 reward。**

reward 衡量"这条轨迹有多好"，而 $\log \pi_T(y|x)$ 是老师赋予轨迹 $y$ 的对数似然——它衡量"老师觉得这条轨迹有多对"。如果老师认为 $y$ 高概率应该出现，$\log \pi_T$ 大 → 这条轨迹"质量高" → 该被奖励；如果老师觉得 $y$ 几乎不会出现，$\log \pi_T$ 极负 → 这条轨迹"质量差" → 该被惩罚。**老师在这里扮演的角色就是 reward model**，只不过这个 reward model 不需要单独训练，直接复用老师本身的 logits。

从信息论角度看，这个解读还能更精确。回忆自信息 $I(y) = -\log p(y)$，那么 $\log \pi_T(y|x) = -I_T(y)$，等价于"老师对 $y$ 的自信息的负数"。直观含义是：用老师的编码方案传输 $y$ 需要的比特数越少 → 老师越熟悉/越认可这条轨迹 → reward 越高。所以 reward 在信息论下的精确含义就是"老师编码学生轨迹的代价的负数"。

**第三层：任何专家分布天然就是一个 reward 函数（RL as inference 视角）。**

更深的视角来自 RL-as-inference 理论：任何分布 $p(y)$ 都可以写成玻尔兹曼形式 $p(y) \propto \exp(R(y))$，等价地 $R(y) = \log p(y) + \text{const}$。这意味着**任何专家分布天然就定义了一个 reward 函数，就是它自己的对数概率**。把 $\pi_T$ 当作"专家分布"，对应的 reward 自然就是 $\log \pi_T(y|x)$。

这也解释了为什么 OPD 和 entropy-regularized RL 在数学上是同一个东西：后者的最优策略恰好是 $\pi^*(y) \propto \exp(R(y))$，反过来就把 reward 读成 log-policy。OPD 和 SAC 是从同一条数学结构的两端进入，一个从"蒸馏"出发，一个从"RL"出发，最后汇合在同一个目标函数上。

**用 token 级例子感受一下。** 假设学生在某个位置 rollout 出 token $y_t = $ "因此"：

- 老师认为 "因此" 在这个 context 下应该出现的概率是 $\pi_T = 0.6$ → $\log \pi_T = -0.51$
- 学生当前认为 "因此" 应该出现的概率是 $\pi_s = 0.1$ → $\log \pi_s = -2.30$
- 该 token 的 dense reward $r_t = -0.51 - (-2.30) = +1.79 \gt 0$

优化方向：提升学生在这个 context 下输出 "因此" 的概率。反过来如果学生 rollout 出 "然而"，老师觉得 $\pi_T = 0.02$ 而学生当前是 $\pi_s = 0.3$，则 $r_t = -3.91 + 1.20 = -2.71 \lt 0$，抑制。**老师 log-prob 在做的事就是 reward 在做的事**：告诉学生"这一步走得对不对、对多少"。所以 (A) 项叫 reward 项是名副其实的——它在形式、语义、和理论结构上都精确对应着 RL 中的奖励信号。

### 第二步：对 $\theta$ 求梯度，处理"期望分布也含 $\theta$"的难点

要做优化就得算 $\nabla_\theta J(\theta)$，但这里期望分布 $\pi_s$ 本身依赖 $\theta$，不能简单地把 $\nabla_\theta$ 塞进期望符号里面。先把期望展开成求和：

$$
J(\theta) = \sum_y \pi_s(y;\theta)\cdot\big[\log\pi_T(y) - \log\pi_s(y;\theta)\big]
$$

用乘法求导法则，$\pi_s$ 和括号内的项都含 $\theta$：

$$
\nabla_\theta J = \underbrace{\sum_y \nabla_\theta\pi_s(y)\cdot[\log\pi_T(y) - \log\pi_s(y)]}_{\text{第一项}} + \underbrace{\sum_y \pi_s(y)\cdot\nabla_\theta[\log\pi_T(y) - \log\pi_s(y)]}_{\text{第二项}}
$$

### 第三步：用 score function 化简第一项

第一项里出现的 $\nabla_\theta\pi_s(y)$ 不太好用（因为它不是期望的形式）。利用恒等式 $\nabla_\theta\pi_s = \pi_s\cdot\nabla_\theta\log\pi_s$（这是 $\nabla\log f = \nabla f / f$ 的直接变形，在 RL 里叫 **score function trick**）：

$$
\text{第一项} = \sum_y \pi_s(y)\cdot\nabla_\theta\log\pi_s(y)\cdot[\log\pi_T(y) - \log\pi_s(y)] = \mathbb{E}_{y\sim\pi_s}\!\big[\nabla\log\pi_s(y)\cdot A(y)\big]
$$

其中我们定义 advantage：

$$
A(y) := \log\pi_T(y|x) - \log\pi_s(y|x)
$$

至此第一项已经化成 policy gradient 的标准形式。

### 第四步：第二项的梯度恒为 0

回到第二项。$\log\pi_T(y)$ 完全不依赖 $\theta$，所以括号内只剩 $-\nabla_\theta\log\pi_s(y)$：

$$
\text{第二项} = -\sum_y \pi_s(y)\cdot\nabla_\theta\log\pi_s(y) = -\mathbb{E}_{y\sim\pi_s}[\nabla_\theta\log\pi_s(y)]
$$

关键的观察是这个期望恒等于 0。证明只用到概率分布的归一化：

$$
\mathbb{E}_{y\sim\pi_s}[\nabla_\theta\log\pi_s(y)] = \sum_y \pi_s(y)\cdot\frac{\nabla_\theta\pi_s(y)}{\pi_s(y)} = \sum_y \nabla_\theta\pi_s(y) = \nabla_\theta\underbrace{\sum_y \pi_s(y)}_{=\,1} = \nabla_\theta(1) = 0
$$

所以**熵正则项的梯度自动消失**，不需要任何手动处理。这其实是 REINFORCE 里 "任意 baseline 可以减去而不改变期望" 的同一个数学结构——概率归一化的副产品。

### 第五步：合并成 policy gradient 的标准形式

两项合并（第二项为 0），最终得到：

$$
\boxed{\;\nabla_\theta J = \mathbb{E}_{y\sim\pi_s}\!\Big[\nabla_\theta\log\pi_s(y|x)\cdot\underbrace{\big(\log\pi_T(y|x) - \log\pi_s(y|x)\big)}_{\text{advantage}\;A(y)}\Big]\;}
$$

对照标准 REINFORCE 公式 $\nabla J = \mathbb{E}_{y\sim\pi_\theta}[\nabla\log\pi_\theta(y)\cdot R(y)]$，形式完全一致。唯一的区别是 reward 不再来自外部环境或 reward model，而是**学生与老师对数概率之差**：老师认为这条轨迹"应该"出现的程度减去学生当前认为它"应该"出现的程度。

### 第六步：利用自回归分解得到 per-token reward

LLM 是自回归生成，任何轨迹的 log-likelihood 都可以按 token 分解：

$$
\log\pi(y|x) = \sum_{t=1}^{T}\log\pi(y_t\mid x, y_{\lt t})
$$

代入 advantage：

$$
A(y) = \sum_{t=1}^{T}\underbrace{\big[\log\pi_T(y_t|x,y_{\lt t}) - \log\pi_s(y_t|x,y_{\lt t})\big]}_{r_t\;:\;\text{第 }t\text{ 个 token 的 dense reward}}
$$

至此 dense reward 的形式自然浮现：**每生成一个 token，老师都基于学生当前位置的 context 给出一个连续标量 $r_t$**。$r_t \gt 0$ 表示"老师认为这个 token 比学生采样的更应该出现"，优化时会强化这个 token；$r_t \lt 0$ 则抑制。整段轨迹的 advantage 就是这些 per-token reward 的累加。

这就是为什么 OPD 看起来"像一个有老师逐 token 打分的 RL"——因为它在数学上**就是**。从信息论一路推下来，没有任何近似，只有恒等变形。

## 四、与 SFT / GRPO 的横向对比

把三种主流后训练方法放在同一个坐标系下：

| 维度 | SFT | OPD | GRPO / RLVR |
|---|---|---|---|
| 信息论本质 | 最小化交叉熵 $H(\pi_T, \pi_s)$ | 最小化 reverse KL $D_{KL}(\pi_s\,\|\,\pi_T)$ | 最大化奖励期望，KL 仅作约束 |
| KL 方向 | Forward KL | Reverse KL | 不直接对应 KL 优化 |
| 采样分布 | $\pi_T$ (off-policy) | $\pi_s$ (on-policy) | $\pi_s$ (on-policy) |
| Reward 粒度 | dense，逐 token CE | dense，逐 token log-ratio | sparse，整条轨迹一个值 |
| 几何行为 | mass-covering | mode-seeking | 取决于 reward 设计 |
| 分布漂移 | 严重 (train/inference mismatch) | 无 | 无 |
| credit assignment | 不需要 (有 ground-truth) | 自动逐 token | 困难，需要 GAE 或分组归一化 |
| Reward 来源 | ground-truth 标签 | 老师 logits 一次 forward | 环境 / verifier / reward model |
| 工程开销 | 1× 学生 forward | 1× 学生 rollout + 1× 老师 forward | 多次 rollout + reward model 调用 |

可以看到 OPD 几乎同时拿掉了 SFT 的分布漂移问题和 RL 的稀疏奖励问题，**位置介于 SFT 和 RL 之间，既不像 SFT 那样脱离学生轨迹，也不像 RL 那样要面对稀疏 reward 的高方差**。代价是要求老师与学生同 tokenizer（因为需要对学生轨迹算 $\log\pi_T$），不能跨词表蒸馏。

## 五、工程实现的几个要点

第一，工程上实际不需要走 REINFORCE。因为 logits 是可微的，可以直接对每个 token 位置算 per-token reverse KL 作为 loss，做监督式 backward：

$$
\mathcal{L}_{\text{OPD,impl}} = \sum_{t=1}^{T} D_{KL}\!\big(\pi_s(\cdot|x,y_{\lt t}) \,\|\, \pi_T(\cdot|x,y_{\lt t})\big)
$$

这本质上和第五步推出来的 policy gradient 等价，但因为不需要 score function 估计，方差更小、训练更稳定。verl 里很多 distillation loss 都是这么实现的。

第二，老师只需要 forward 一次。学生 rollout 出 $y$ 后，把 $(x, y)$ 喂给老师做一次 teacher-forcing forward，就能拿到所有位置的完整 logits → 整段 $\{r_t\}$ 一次性算完。相比 GRPO 一组要 4-16 条 rollout 加 reward model 调用，OPD 的算力开销低很多。

第三，与现有 GRPO 框架兼容。如果已经在 verl 上跑 GRPO，OPD 可以理解为"把 group-relative advantage 换成 teacher-logit advantage"，rollout 和 loss 计算流水线几乎不动，只需替换 advantage 计算函数。Thinking Machines 的 *On-Policy Distillation*、MiniLLM (Gu et al., ICLR 2024) 都属于这个家族。

第四，对老师有硬性要求。OPD 需要老师能给学生轨迹算 $\log\pi_T(y_t|...)$，这意味着老师必须是同 tokenizer / 同词表的模型。跨 tokenizer 的蒸馏（例如 GPT-4 → Qwen）走不通这条路，只能退回 SFT 或 black-box reward distillation。

## 六、总结

整条推导链可以浓缩为一句话：

> **OPD = 选择 reverse KL 强制 on-policy 采样 + 把老师 log-prob 当 dense per-token reward + 学生熵作为自动正则项。**

从信息论的角度看，选择哪个 KL 方向就已经决定了一切：forward KL 给出 SFT，reverse KL 给出 OPD。从 reverse KL 出发，经过翻号拆项、score function、归一化求导，自然得到 entropy-regularized policy gradient，再用自回归分解得到 per-token reward 的形式。整条链子没有任何近似或工程妥协，每一步都是恒等变形。

OPD 真正的位置是 SFT 和 RL 之间的过渡阶段——它继承了 SFT 的稠密监督信号，又继承了 RL 的 on-policy 性质，同时避免了二者各自的痛点。对于一个比老师小得多的学生模型，这通常是性价比最高的对齐方式。

---

> 如果你想看 OPD 在工程实践中的具体表现——Thinking Machines 博客的 5 大实验、THUNLP 的成功/失败机制分析、tinker-cookbook 的源码剖析——可以接着看 [On-Policy Distillation：从分布视角理解 SFT、RL 与 OPD](../on-policy-distillation/)。
