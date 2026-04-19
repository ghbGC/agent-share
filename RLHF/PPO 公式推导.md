---
Author: ghbgc
Date: 2026-04-19
---
## 符号定义
### 状态转移概率
$$
P(s_{t+1}|s_t,a_t)
$$
>在状态$s_t$执行动作$a_t$后转移到下一状态$s_{t+1}$的条件概率
>该项由环境决定，与策略参数$\theta$无关
### 策略与轨迹分布
$$
\left\{
\begin{aligned}
&\pi_\theta(a_t|s_t) \\
&\pi_\theta(\tau)=\pi_\theta(s_1,a_1,...,s_T,a_T)=p(s_1)\prod_{t=1}^{T}\pi_\theta(a_t|s_t)p(s_{t+1}|s_t,a_t)
\end{aligned}
\right.
$$
>$\pi_\theta(a_t|s_t)$表示策略在状态$s_t$下选择动作$a_t$的概率
>$\pi_\theta(\tau)$表示完整轨迹$\tau=(s_1,a_1,\dots,s_T,a_T)$在当前策略和环境下出现的概率
### 轨迹回报
$$
R(\tau)
$$
>表示轨迹$\tau$的总回报

## 目标函数$J(\theta)$和$\nabla J(\theta)$
$$
\begin{align}
J(\theta)&=\mathbb{E}_{\tau\sim\pi_{\theta}}[R(\tau)]=\int\pi_{\theta}(\tau)R(\tau)d\tau \\
\\
\nabla J(\theta)
&=\int\nabla_\theta\pi_{\theta}(\tau)R(\tau)d\tau=^{(注释1)}\int\pi_{\theta}(\tau)\nabla_\theta\log[\pi_{\theta}(\tau)]R(\tau)d\tau \\
&=^{(注释2)}\int\pi_{\theta}(\tau)\sum_{t=1}^{T}\nabla_\theta\log[\pi_{\theta}(a_t|s_t)]R(\tau)d\tau \\
&=\mathbb{E}_{\tau\sim\pi_{\theta}(\tau)}(\sum_{t=1}^{T}\nabla_\theta\log[\pi_{\theta}(a_t|s_t)]R(\tau)) \\
&\approx \frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}\nabla_\theta\log[\pi_{\theta}(a_{i,t}|s_{i,t})]R(\tau_i)
\end{align}
$$

### 从奖励$R$到优势函数$A$的演变
- $R(\tau_i)$：整条轨迹共享同一个回报，无法刻画“第$t$步动作”的贡献。
	- 改进点1：第$t$步的奖励与$t$及其之后的步数有关 $\rightarrow R(\tau_{i,t})=\sum_{t'=t}^{T}\gamma^{t'-t}r_{i,t'}$
	- 改进点2：降低梯度估计的方差，使更新更加稳定。通俗来讲，就是在好的局势和坏的局势下奖励应该有正有负，而不是一边倒 $\rightarrow Baseline: B(s_{i,t})$
- $\rightarrow R(\tau_{i,t})-B(s_{i,t})$
	- 存在的问题：$R(\tau_{i,t})$的计算每次都是一次采样，方差很大，不稳定
	- 解决：使用$Q(s,a)^{（注释3:状态价值函数V(s)和动作价值函数Q(s,a)）}$估计状态$s$下做出动作$a$的期望回报
- $\rightarrow$ 优势函数：$A(s,a)=Q(s,a)-V(s)$
	- $V(s)^{（注释3:状态价值函数V(s)和动作价值函数Q(s,a)）}$可视为基线$B(s)$的具体实现
- $\rightarrow 广义优势估计^{(注释4)}：\hat{A}^{GAE}(s,a)$
	- $\hat A_t^{\mathrm{GAE}}=\sum_{l=0}^{\infty}(\gamma\lambda)^l\delta_{t+l}$
		- $\delta_t=r_t+\gamma V(s_{t+1})-V(s_t)$

$$
\begin{align}
\nabla J(\theta)
&\approx \frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}\nabla_\theta\log[\pi_{\theta}(a_{i,t}|s_{i,t})]R(\tau_i) \\
&\rightarrow\frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}\nabla_\theta\log[\pi_{\theta}(a_{i,t}|s_{i,t})]\hat{A}^{GAE}(s_{i,t},a_{i,t}) \\
&=^{（注释5：重要性采样）}\frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}\frac{\pi_{\theta}(a_{i,t}|s_{i,t})}{\pi_{\theta'}(a_{i,t}|s_{i,t})}\nabla_\theta\log[\pi_{\theta}(a_{i,t}|s_{i,t})]\hat{A}^{GAE}(s_{i,t},a_{i,t}) \\
&=^{（注释6）}\frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}\frac{\nabla_\theta\pi_{\theta}(a_{i,t}|s_{i,t})}{\pi_{\theta'}(a_{i,t}|s_{i,t})}\hat{A}^{GAE}(s_{i,t},a_{i,t}) \\
\\
%% L(\theta)&\approx\frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}\frac{\pi_{\theta}(a_{i,t}|s_{i,t})}{\pi_{\theta'}(a_{i,t}|s_{i,t})}\hat{A}^{GAE}(s_{i,t},a_{i,t}) \\
\end{align}
$$
>$J(\theta)\rightarrow L(\theta)$：
> 通过重要性采样，可将期望$\mathbb{E}_{\tau\sim\pi(\theta)}$重写为基于旧策略$\pi_{\theta'}$采样轨迹的加权期望形式，从而构造在旧策略采样分布下定义的替代优化目标（surrogate objective）$L(\theta)$，用于在策略迭代过程中近似优化原始期望回报目标$J(\theta)$。
> 由于该变换依赖重要性加权和有限样本估计，$L(\theta)$与$J(\theta)$在一般情况下并不严格等价。

$$
L(\theta)=\mathbb{E}_{\tau\sim\pi_{\theta'}}\left[
\sum_{t=1}^{T}
\frac{\pi_{\theta}(a_t|s_t)}{\pi_{\theta'}(a_t|s_t)}
\hat{A}^{GAE}(s_t,a_t)
\right]
\approx
\frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}
\frac{\pi_{\theta}(a_{i,t}|s_{i,t})}{\pi_{\theta'}(a_{i,t}|s_{i,t})}
\hat{A}^{GAE}(s_{i,t},a_{i,t})
$$

### 截断函数$\text{clip}$
$$
\begin{align}
\text{clip}(\frac{\pi_{\theta}(a_{i,t}|s_{i,t})}{\pi_{\theta'}(a_{i,t}|s_{i,t})},1-\epsilon,1+\epsilon)
\end{align}
$$

### PPO最终目标函数（最大化）
$$
\begin{align}
L^{CLIP}(\theta)&=\mathbb{E}_{\tau\sim\pi_{\theta'}}\sum_{t=1}^{T_i}\min\left[\frac{\pi_{\theta}(a_{t}|s_{t})}{\pi_{\theta'}(a_{t}|s_{t})}\hat{A}^{GAE}(s_{t},a_{t}), \text{clip}(\frac{\pi_{\theta}(a_{t}|s_{t})}{\pi_{\theta'}(a_{t}|s_{t})},1-\epsilon,1+\epsilon)\hat{A}^{GAE}(s_{t},a_{t})\right] \\
&\approx\frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}\min\left[\frac{\pi_{\theta}(a_{i,t}|s_{i,t})}{\pi_{\theta'}(a_{i,t}|s_{i,t})}\hat{A}^{GAE}(s_{i,t},a_{i,t}), \text{clip}(\frac{\pi_{\theta}(a_{i,t}|s_{i,t})}{\pi_{\theta'}(a_{i,t}|s_{i,t})},1-\epsilon,1+\epsilon)\hat{A}^{GAE}(s_{i,t},a_{i,t})\right] \\
\end{align}
$$

### 状态价值函数损失（最小化）

$$
\begin{align}
L^{VF}(\theta)
&=\mathbb{E}_{s_t\sim \pi_{\theta'}}\left[(V_\theta(s_t)-R_t)^2\right] \\
&\approx\frac{1}{N}\sum_{i=1}^{N}\left(\frac{1}{T_i}\sum_{t=1}^{T_i}\ell_{i,t}\right)\\
\end{align}
$$

>附：模型输出对照（🔥参数更新，❄️参数不更新）

🔥策略模型（Policy Model）输出 $\rightarrow\pi_{\theta}(a_{t}|s_{t})$
❄️参考模型（Reference Model）输出 $\rightarrow\pi_{\theta'}(a_{t}|s_{t})$
🔥价值模型（Value Model）输出 $\rightarrow V_\theta(s_t)$
❄️奖励模型（Reward Model）输出 $\rightarrow r_t$

---

## 注释

### 注释1

使用恒等式$\nabla_\theta \pi_\theta(\tau)=\pi_\theta(\tau)\nabla_\theta\log\pi_\theta(\tau)$

### 注释2

由于$\pi_\theta(\tau)=p(s_1)\prod_{t=1}^{T}\pi_\theta(a_t|s_t)p(s_{t+1}|s_t,a_t)$，且$p(s_1)$与$p(s_{t+1}|s_t,a_t)$不依赖$\theta$
因此$\nabla_\theta\log\pi_\theta(\tau)=\sum_{t=1}^{T}\nabla_\theta\log\pi_\theta(a_t|s_t)$

### 注释3：状态价值函数$V(s)$和动作价值函数$Q(s,a)$

1. 状态价值函数$V(s_t)$：在状态$s_t$下，之后始终遵循当前策略 $\pi_\theta$ 所能获得的期望折扣回报。数学上可表示为：
$$
   V(s_t) = \mathbb{E}_{\tau_{t:} \sim \pi_\theta} \left[ \sum_{t'=t}^{\infty} \gamma^{t'-t} r_{t'} \;\Big|\; s_t \right]
$$
其中$\tau_{t:}$表示从$t$时刻开始的后续轨迹片段

2. 动作价值函数$Q(s_t, a_t)$：在状态$s_t$下，先明确执行了动作$a_t$，之后始终遵循策略$\pi_\theta$所能获得的期望折扣回报。数学上可表示为：
$$
   Q(s_t, a_t) = \mathbb{E}_{\tau_{t+1:} \sim \pi_\theta} \left[ \sum_{t'=t}^{\infty} \gamma^{t'-t} r_{t'} \;\Big|\; s_t, a_t \right]
$$

3. $V(s_t)$和$Q(s_t, a_t)$的关系
$$
\begin{align}
Q(s_t, a_t) &= \mathbb{E}_{\tau_{t+1:} \sim \pi_\theta} \left[ r_t + \gamma \sum_{t'=t+1}^{\infty} \gamma^{t'-(t+1)} r_{t'} \;\Big|\; s_t, a_t \right]\\
& ~~~~ 由于s_{t+1}是条件于s_t,a_t的随机变量\\
& ~~~~ 应用全概率公式：\mathbb{E}[X] = \mathbb{E}_Y\big[\mathbb{E}[X \mid Y]\big]\\
& ~~~~ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ =\sum_y P(Y=y) \cdot \mathbb{E}[X \mid Y=y] \quad (离散条件下) \\ 
& = \mathbb{E}_{s_{t+1} \sim P(\cdot|s_t,a_t)}\left[ r_t + \gamma \cdot \mathbb{E}_{\tau_{t+1:}} \left[ \sum_{t'=t+1}^{\infty} \gamma^{t'-(t+1)} r_{t'} \;\Big|\; s_{t+1}, s_t, a_t \right] \right] \\
& = \sum_{s_{t+1}} P(s_{t+1} \mid s_t, a_t) \left[ r_t + \gamma \cdot \mathbb{E}_{\tau_{t+1:}} \left[ \sum_{t'=t+1}^{\infty} \gamma^{t'-(t+1)} r_{t'} \;\Big|\; s_{t+1}, s_t, a_t \right] \right] (离散条件下)\\
& ~~~~马尔可夫性：在给定s_{t+1}之后，未来的状态和奖励条件独立于更早的历史(s_t, a_t) \\
& = \mathbb{E}_{s_{t+1} \sim P(\cdot|s_t,a_t)}\left[ r_t + \gamma \cdot \mathbb{E}_{\tau_{t+1:}} \left[ \sum_{t'=t+1}^{\infty} \gamma^{t'-(t+1)} r_{t'} \;\Big|\; s_{t+1} \right] \right] \\
& = \sum_{s_{t+1}} P(s_{t+1} \mid s_t, a_t) \left[ r_t + \gamma \cdot \mathbb{E}_{\tau_{t+1:}} \left[ \sum_{t'=t+1}^{\infty} \gamma^{t'-(t+1)} r_{t'} \;\Big|\; s_{t+1} \right] \right] (离散条件下) \\
& ~~~~ (1)~环境是确定的 \\
& = r_t + \gamma \cdot \mathbb{E}_{\tau_{t+1:}} \left[ \sum_{t'=t+1}^{\infty} \gamma^{t'-(t+1)} r_{t'} \;\Big|\; s_{t+1} \right] = r_t + \gamma V(s_{t+1}) \\
& ~~~~ (2)~随机环境下的单次采样近似 (PPO的实际情形) \\
& \approx r_t + \gamma \cdot \mathbb{E}_{\tau_{t+1:}} \left[ \sum_{t'=t+1}^{\infty} \gamma^{t'-(t+1)} r_{t'} \;\Big|\; s_{t+1} \right] = r_t + \gamma V(s_{t+1})
\end{align}
$$
4. $V(s_t)$和$V(s_{t+1})$的关系
根据3的推导，不难看出：
$$
V(s_t) = \mathbb{E}_{a_t \sim \pi}[Q(s_t,a_t)]
$$
代入：
$$
Q(s_t,a_t) = \mathbb{E}_{s_{t+1}}[r_t + \gamma V(s_{t+1})]
$$
可得：
$$
V(s_t) = \mathbb{E}_{a_t \sim \pi_\theta(\cdot|s_t),\; s_{t+1} \sim P(\cdot|s_t,a_t)} \left[ r_t + \gamma V(s_{t+1}) \right]
$$
这个公式满足贝尔曼方程，其含义是状态$s_t$的价值等于先按策略采样动作$a_t$，再按环境转移采样$s_{t+1}$，然后获得即时奖励$r_t$加上折扣后的下一状态价值$V(s_{t+1})$的期望
在t+1时：
$$
V(s_{t+1}) = \mathbb{E}_{a_{t+1} \sim \pi_\theta,\; s_{t+2} \sim P} \left[ r_{t+1} + \gamma V(s_{t+2}) \right]
$$
在PPO算法实际运行时，单次采样近似的形式为：
$$
V(s_{t+1}) \approx r_{t+1} + \gamma V(s_{t+2})
$$
### 注释4：$A_t^{\mathrm{GAE}}=\sum_{l=0}^{\infty}(\gamma\lambda)^l\delta_{t+l},\quad \delta_t=r_t+\gamma V(s_{t+1})-V(s_t)$

$\text{n-step}$优势估计
$$
\begin{align}
&A(s,a)=Q(s,a)-V(s) \\
\\
&Q(s_t,a_t)=r_t+\gamma V(s_{t+1}) \quad (参考注释3) \\
&A(s_t,a_t)=r_t+\gamma V(s_{t+1})-V(s_t) \\
\\
&V(s_{t+1})=r_{t+1}+\gamma V(s_{t+2}) \quad (参考注释3) \\
\\
&\text{n-step优势估计:} \\
&A^1(s_t, a_t) = r_t + \gamma V(s_{t+1}) - V(s_t) \\
&A^2(s_t, a_t) = r_t + \gamma r_{t+1} + \gamma^2 V(s_{t+2}) - V(s_t) \\
&A^3(s_t, a_t) = r_t + \gamma r_{t+1} + \gamma^2 r_{t+2} + \gamma^3 V(s_{t+3}) - V_\theta(s_t) \\
&~~~~~~~~\vdots \\
&A^T(s_t, a_t) = r_t + \gamma r_{t+1} + \gamma^2 r_{t+2} + \gamma^3 r_{t+3} + \cdots + \gamma^T r_T - V(s_t)\end{align}
$$
采样的步数越多，方差越大，偏差越小；采样的步数越少，方差越小，偏差越大
$$
\begin{align}
\text{令：}\\
&\delta_t=r_t+\gamma V(s_{t+1})-V(s_t) ~~~~ \text{（TD误差）}\\
\text{则：}\\
&A^1(s_t, a) = \delta_t \\
&A^2(s_t, a) = \delta_t + \gamma \delta_{t+1} \\
&A^3(s_t, a) = \delta_t + \gamma \delta_{t+1} + \gamma^2 \delta_{t+2} \\
&~~~~~~~~\vdots\\
\end{align}
$$
$GAE\text{（广义优势估计）}$
$$
\begin{align}
A^{GAE}(s_t, a_t) &= (1-\lambda)(A^1 + \lambda A^2 + \lambda^2 A^3 + \cdots) \\
&= (1-\lambda)(\delta_t + \lambda (\delta_t + \gamma \delta_{t+1}) + \lambda^2 (\delta_t + \gamma \delta_{t+1} + \gamma^2 \delta_{t+2}) + \cdots) \\
&= (1-\lambda)(\delta_t (1 + \lambda + \lambda^2 + \cdots) + \gamma \delta_{t+1} (\lambda + \lambda^2 + \cdots) + \cdots) \\
& \\
&~~~~~~~~ \text{无穷级数：等比级数求和公式} \\
&~~~~~~~~ 对于 1 + \lambda + \lambda^2 + \cdots \text{（其中} |\lambda| < 1 \text{），其和为} \frac{1}{1-\lambda} \\
&= (1-\lambda)(\delta_t \frac{1}{1-\lambda} + \gamma \delta_{t+1} \frac{\lambda}{1-\lambda} + \cdots) \\
&= \sum_{l=0}^{\infty} (\gamma \lambda)^l \delta_{t+l}
\end{align}
$$

### 注释5：重要性采样

重要性采样：$\mathbb{E}_{x \sim p}[f(x)] = \mathbb{E}_{x \sim q}\left[ f(x) \cdot \frac{p(x)}{q(x)} \right]$
在PPO中，这里采用逐时间步的概率比值近似，而非整条轨迹的概率比值

>讨论：关于重要性采样前后$A^{GAE}(s_t, a_t)$的计算问题
- 根据重要性采样的定义$\mathbb{E}_{x \sim p}[f(x)] = \mathbb{E}_{x \sim q}\left[ f(x) \cdot \frac{p(x)}{q(x)} \right]$，$f(x)$在重要性采样的前后保持不变，那么$A_{\theta}^{GAE}(s_t, a_t)$不应该变成$A_{\theta'}^{GAE}(s_t, a_t)$
- 但在PPO的实际实现中，优势计算在重要性采样前后从$A_{\theta}^{GAE}(s_t, a_t)$变成了$A_{\theta'}^{GAE}(s_t, a_t)$
	- 重要性采样前：
		- $\nabla J(\theta)=\frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}\nabla_\theta\log[\pi_{\theta}(a_{i,t}|s_{i,t})]\hat{A_\theta}^{GAE}(s_{i,t},a_{i,t})$
		- $A^{GAE}(s_t, a_t)$是基于策略模型$\pi_\theta$计算的，表示为$A_\theta^{GAE}(s_t, a_t)$
	- 重要性采样后：
		- $\nabla J(\theta)=\frac{1}{N}\sum_{i=1}^{N}\sum_{t=1}^{T_i}\frac{\pi_{\theta}(a_{i,t}|s_{i,t})}{\pi_{\theta'}(a_{i,t}|s_{i,t})}\nabla_\theta\log[\pi_{\theta}(a_{i,t}|s_{i,t})]\hat{A}^{GAE}(s_{i,t},a_{i,t})$
		- $A^{GAE}(s_t, a_t)$是基于策略模型$\pi_{\theta'}$计算的，表示为$A_{\theta'}^{GAE}(s_t, a_t)$
- 原因是：实施重要性采样后，轨迹数据是从旧策略$\pi_{\theta'}$​中采样的，而$A^{GAE}(s_t, a_t)$是基于轨迹的即时奖励$r_t$和旧策略的价值估计$V_{\theta'}$计算的固定数值，不参与梯度$\nabla J(\theta)$计算，需要将其作为常数项看待
	- 在代码中体现为 `with torch.no_grad():`
 
### 注释6

使用恒等式$\nabla_\theta\log\pi_\theta(a|s)=\frac{\nabla_\theta\pi_\theta(a|s)}{\pi_\theta(a|s)}$

---

## 参考

1. [Proximal Policy Optimization Algorithms](https://arxiv.org/pdf/1707.06347)
2. [强化学习PPO公式推导 - 知乎](https://zhuanlan.zhihu.com/p/1989469160779055947)
3. [【强化学习】PPO的理论推导 - 知乎](https://zhuanlan.zhihu.com/p/563166533)
4. [零基础学习强化学习算法：ppo_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1iz421h7gb)
5. [[Agentic RL][01] 练习两天半，完全从零开始实现PPO算法...](https://www.bilibili.com/video/BV1CHQDYHEsU)
