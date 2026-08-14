# Adam：一种随机优化方法（Adam: A Method for Stochastic Optimization）

**作者**: Diederik P. Kingma（阿姆斯特丹大学，OpenAI）、Jimmy Lei Ba（多伦多大学）

**翻译日期**: 2026-08-14

---

## 摘要（Abstract）

我们提出 Adam，一种基于低阶矩的自适应估计、对随机目标函数进行一阶梯度优化的算法。该方法实现简单、计算高效、内存需求小，对梯度的对角缩放（diagonal rescaling）不变，非常适合数据和/或参数规模庞大的问题。该方法同样适用于非平稳目标以及梯度非常嘈杂和/或稀疏的问题。其超参数具有直观的解释，通常只需很少的调参。我们还讨论了一些启发 Adam 的相关算法的联系。我们分析了该算法的理论收敛性质，并在在线凸优化框架下给出了与已知最佳结果相当的遗憾界（regret bound）。经验结果表明 Adam 在实践中表现良好，优于其他随机优化方法。最后，我们讨论了 AdaMax——一种基于无穷范数的 Adam 变体。

## 1. 引言（Introduction）

基于梯度的随机优化在许多科学与工程领域具有核心的实际重要性。这些领域的许多问题都可以归结为对某个以标量参数化的目标函数进行优化——关于其参数求最大值或最小值。如果函数对其参数可微，梯度下降就是一种相对高效的优化方法，因为对所有参数求一阶偏导数的计算复杂度与仅求函数值相当。目标函数常常是随机的。例如，许多目标函数由对数据的不同子样本求值的一系列子函数之和构成；在这种情况下，可以针对单个子函数取梯度步，即随机梯度下降（stochastic gradient descent, SGD）或上升，从而使优化更高效。SGD 已被证明是一种高效且有效的优化方法，它也是许多机器学习成功案例的核心，例如深度学习领域的最新进展（Deng et al., 2013; Krizhevsky et al., 2012; Hinton & Salakhutdinov, 2006; Hinton et al., 2012a; Graves et al., 2013）。目标还可能存在数据子采样以外的噪声来源，例如 dropout（Hinton et al., 2012b）正则化。对于所有这类带噪声的目标，都需要高效的随机优化技术。本文聚焦于参数空间高维的随机目标的优化。在这些情况下，高阶优化方法并不合适，因此本文的讨论仅限于一阶方法。

我们提出 Adam，一种只需要一阶梯度、内存需求很小的高效随机优化方法。该方法利用梯度的一阶矩和二阶矩的估计，为不同参数计算各自的自适应学习率；Adam 这个名字来源于 adaptive moment estimation（自适应矩估计）。我们的方法旨在结合两种近来的流行方法的优势：AdaGrad（Duchi et al., 2011）擅长处理稀疏梯度，RMSProp（Tieleman & Hinton, 2012）擅长在线和非平稳场景；与这些以及其他随机优化方法的重要联系在第 5 节中阐明[^*]。Adam 的一些优点是：参数更新的量级对梯度的缩放不变，其步长近似受步长超参数约束，不要求目标平稳，能处理稀疏梯度，并且天然执行一种步长退火（step size annealing）。

[^*]: 两位作者贡献相等。作者排序由在 Google Hangout 上抛硬币决定。

## 2. 算法（Algorithm）

我们提出的算法 Adam 的伪代码见算法 1。设 f(θ) 是一个带噪声的目标函数：一个对参数 θ 可微的随机标量函数。我们感兴趣的是关于参数 θ 最小化该函数的期望值 E[f(θ)]。用 f_1(θ), ..., f_T(θ) 表示后续时间步 1, ..., T 上随机函数的实现。随机性可能来自对数据点的随机子样本（小批量）求值，也可能来自函数内在的噪声。用 g_t = ∇_θ f_t(θ) 表示在时间步 t 求值的梯度，即 f_t 关于 θ 的偏导数向量。

该算法更新梯度（m_t）和梯度平方（v_t）的指数移动平均（exponential moving averages），其中超参数 β_1, β_2 ∈ [0, 1) 控制这些移动平均的指数衰减率。移动平均本身是梯度的一阶矩（均值）和二阶原始矩（未中心化的方差）的估计。然而，这些移动平均以零向量初始化，导致矩估计偏向于零，尤其在初始时间步，以及衰减率较小（即 β 接近 1）的时候。好消息是，这种初始化偏差很容易被抵消，得到偏置校正后的估计 m̂_t 和 v̂_t。详见第 3 节。

注意，算法 1 的效率可以通过改变计算顺序在牺牲可读性的前提下得到提升，例如把循环中的最后三行替换为以下各行：

$$\alpha_t = \alpha \cdot \sqrt{1 - \beta_2^t} / (1 - \beta_1^t) \text{ 且 } \theta_t \leftarrow \theta_{t-1} - \alpha_t \cdot m_t / (\sqrt{v_t} + \hat{\epsilon})。$$

**算法 1：Adam**，我们提出的随机优化算法。细节见第 2 节；一个计算顺序稍高效（但不够清晰）的版本亦见第 2 节。g_t² 表示 g_t 的逐元素平方 g_t ⊙ g_t。本文测试的机器学习问题中效果良好的默认设置为 α = 0.001、β_1 = 0.9、β_2 = 0.999 和 ε = 10⁻⁸。所有向量运算均为逐元素运算。用 β_1^t 和 β_2^t 表示 β_1 和 β_2 的 t 次幂。

```
Require: α: 步长（Stepsize）
Require: β_1, β_2 ∈ [0, 1): 矩估计的指数衰减率
Require: f(θ): 带参数 θ 的随机目标函数
Require: θ_0: 初始参数向量
  m_0 ← 0 （初始化一阶矩向量）
  v_0 ← 0 （初始化二阶矩向量）
  t ← 0 （初始化时间步）
  while θ_t 未收敛 do
     t ← t + 1
     g_t ← ∇_θ f_t(θ_{t-1}) （在时间步 t 获取关于随机目标的梯度）
     m_t ← β_1 · m_{t-1} + (1 − β_1) · g_t （更新有偏一阶矩估计）
     v_t ← β_2 · v_{t-1} + (1 − β_2) · g_t² （更新有偏二阶原始矩估计）
     m̂_t ← m_t / (1 − β_1^t) （计算偏置校正后的一阶矩估计）
     v̂_t ← v_t / (1 − β_2^t) （计算偏置校正后的二阶原始矩估计）
     θ_t ← θ_{t-1} − α · m̂_t / (√v̂_t + ε) （更新参数）
  end while
  return θ_t （最终参数）
```

### 2.1 Adam 的更新规则（Adam's Update Rule）

Adam 更新规则的一个重要性质是它对步长的谨慎选择。假设 ε = 0，时间步 t 在参数空间中的有效步长为 Δ_t = α · m̂_t / √v̂_t。有效步长有两个上界：在 (1 − β_1) > √(1 − β_2) 的情况下，|Δ_t| ≤ α · (1 − β_1)/√(1 − β_2)；否则 |Δ_t| ≤ α。第一种情况只发生在最严重的稀疏情形：当梯度在所有时间步上均为零、只有当前时间步非零。对于不那么稀疏的情形，有效步长会更小。当 (1 − β_1) = √(1 − β_2) 时，有 |m̂_t / √v̂_t| < 1，因此 |Δ_t| < α。在更常见的场景中，由于 |E[g]/√(E[g²])| ≤ 1，会有 m̂_t / √v̂_t ≈ ±1。每个时间步在参数空间中所取步骤的有效量级近似受步长设置 α 约束，即 |Δ_t| ≲ α。这可以理解为在当前参数值周围建立一个信任域（trust region），越过该信任域当前梯度估计就无法提供足够信息。这通常使人们相对容易预先知道 α 的正确尺度。例如，对许多机器学习模型，我们常常预先知道好的最优点高概率落在参数空间的某个设定区域内；例如，对参数有一个先验分布也并不罕见。由于 α 设定参数空间步长的（上界）量级，我们往往可以推导出 α 的正确数量级，使得从 θ_0 出发在若干次迭代内能够到达最优点。稍微滥用一下术语，我们把比值 m̂_t / √v̂_t 称为信噪比（signal-to-noise ratio, SNR）。SNR 越小，有效步长 Δ_t 越接近零。这是一个可取的性质，因为 SNR 越小意味着 m̂_t 的方向是否对应真实梯度方向的不确定性越大。例如，在接近最优点时 SNR 值通常会趋近于 0，导致参数空间中的有效步长变小：这是一种形式的自动退火。有效步长 Δ_t 还对梯度的尺度不变：把梯度 g 按因子 c 缩放会使 m̂_t 缩放因子 c、v̂_t 缩放因子 c²，两者相互抵消：(c · m̂_t)/(√(c² · v̂_t)) = m̂_t / √v̂_t。

## 3. 初始化偏差校正（Initialization Bias Correction）

如第 2 节所述，Adam 使用初始化偏差校正项。我们在这里推导二阶矩估计的校正项；一阶矩估计的推导完全类似。设 g 为随机目标 f 的梯度，我们希望使用平方梯度的指数移动平均（衰减率为 β_2）估计它的二阶原始矩（未中心化的方差）。设 g_1, ..., g_T 为后续时间步的梯度，每个都从底层梯度分布 g_t ~ p(g_t) 中抽取。设指数移动平均初始化为 v_0 = 0（零向量）。首先注意到时间步 t 指数移动平均的更新 v_t = β_2 · v_{t-1} + (1 − β_2) · g_t²（其中 g_t² 表示逐元素平方 g_t ⊙ g_t）可以写成之前所有时间步梯度的函数：

$$v_t = (1 - \beta_2) \sum_{i=1}^{t} \beta_2^{t-i} \cdot g_i^2 \tag{1}$$

我们想知道 E[v_t]——时间步 t 指数移动平均的期望值——与真实二阶矩 E[g_t²] 的关系，以便校正两者之间的差异。对式 (1) 左右两边取期望：

$$E[v_t] = E\left[ (1 - \beta_2) \sum_{i=1}^{t} \beta_2^{t-i} \cdot g_i^2 \right] \tag{2}$$
$$= E[g_t^2] \cdot (1 - \beta_2) \sum_{i=1}^{t} \beta_2^{t-i} + \zeta \tag{3}$$
$$= E[g_t^2] \cdot (1 - \beta_2^t) + \zeta \tag{4}$$

其中，如果真实二阶矩 E[g_i²] 是平稳的，则 ζ = 0；否则 ζ 可以保持很小，因为指数衰减率 β_1 可以（而且应该）被选择为让指数移动平均对过于久远的梯度赋予很小的权重。剩下的项是 (1 − β_2^t)，它是由运行均值用零初始化造成的。因此在算法 1 中，我们用这一项相除来校正初始化偏差。

在稀疏梯度的情况下，要可靠地估计二阶矩，需要选择较小的 β_2 值在许多梯度上取平均；然而恰恰是在这种 β_2 较小的情形下，缺少初始化偏差校正会导致最初的几步过大。

## 4. 收敛性分析（Convergence Analysis）

我们使用 (Zinkevich, 2003) 提出的在线学习框架分析 Adam 的收敛性。给定一个任意、未知的凸代价函数序列 f_1(θ), f_2(θ), ..., f_T(θ)。在每个时间 t，我们的目标是预测参数 θ_t，并在一个先前未知的代价函数 f_t 上对它求值。由于序列的性质事先未知，我们用遗憾（regret）评估算法——即此前所有步骤在线预测 f_t(θ_t) 与可行集中最佳固定点参数 f_t(θ*) 之差的总和。具体地，遗憾定义为：

$$R(T) = \sum_{t=1}^{T} [f_t(\theta_t) - f_t(\theta^*)] \tag{5}$$

其中 $\theta^* = \arg\min_{\theta \in X} \sum_{t=1}^{T} f_t(\theta)$。我们证明 Adam 具有 O(√T) 的遗憾界，证明见附录。我们的结果与该通用凸在线学习问题已知的最佳界相当。我们还使用一些定义简化记号，其中 g_t = ∇f_t(θ_t)，g_{t,i} 为其第 i 个元素。我们定义 g_{1:t,i} ∈ R^t 为包含到 t 为止所有迭代中梯度的第 i 维的向量，即 g_{1:t,i} = [g_{1,i}, g_{2,i}, · · · , g_{t,i}]。此外，我们定义 $\gamma \triangleq \frac{\beta_1}{\sqrt{\beta_2}}$。我们下面的定理在学习率 α_t 以 t^{-1/2} 的速率衰减、一阶矩运行平均系数 β_{1,t} 以 λ 指数衰减（λ 通常接近 1，如 1 − 10⁻⁸）时成立。

**定理 4.1。** 假设函数 f_t 有有界梯度：对所有 θ ∈ R^d，‖∇f_t(θ)‖₂ ≤ G，‖∇f_t(θ)‖∞ ≤ G_∞；且 Adam 生成的任意 θ_t 之间的距离有界：对任意 m, n ∈ {1, ..., T}，‖θ_n − θ_m‖₂ ≤ D，‖θ_m − θ_n‖∞ ≤ D_∞；并且 β_1, β_2 ∈ [0, 1) 满足 $\frac{\beta_1}{\sqrt{\beta_2}} < 1$。令 $\alpha_t = \frac{\alpha}{\sqrt{t}}$，β_{1,t} = β_1 λ^{t-1}，λ ∈ (0, 1)。则对一切 T ≥ 1，Adam 满足如下保证：

$$R(T) \le \frac{D^2}{2\alpha(1 - \beta_1)} \sum_{i=1}^{d} \sqrt{T \hat{v}_{T,i}} + \frac{\alpha(1 + \beta_1)G_\infty}{(1 - \beta_1)\sqrt{1 - \beta_2}(1 - \gamma)^2} \sum_{i=1}^{d} \|g_{1:T,i}\|_2 + \sum_{i=1}^{d} \frac{D_\infty^2 G_\infty \sqrt{1 - \beta_2}}{2\alpha(1 - \beta_1)(1 - \lambda)^2}$$

我们的定理 4.1 表明，当数据特征稀疏且有界梯度时，求和项可以远小于其上界 $\sum_{i=1}^{d} \|g_{1:T,i}\|_2 \ll dG_\infty \sqrt{T}$ 且 $\sum_{i=1}^{d} \sqrt{T \hat{v}_{T,i}} \ll dG_\infty \sqrt{T}$，特别是当函数类别和数据特征形如 (Duchi et al., 2011) 第 1.2 节所述时。他们对期望值 $E[\sum_{i=1}^{d} \|g_{1:T,i}\|_2]$ 的结果同样适用于 Adam。特别是，Adam 与 Adagrad 这类自适应方法可以实现 O(log d √T)，优于非自适应方法的 O(√(dT))。把 β_{1,t} 向零衰减在我们的理论分析中很重要，也与之前的经验发现一致，例如 (Sutskever et al., 2013) 建议在训练结束时降低动量系数可以改善收敛。

最后，我们可以证明 Adam 的平均遗憾收敛：

**推论 4.2。** 假设函数 f_t 有有界梯度：对所有 θ ∈ R^d，‖∇f_t(θ)‖₂ ≤ G，‖∇f_t(θ)‖∞ ≤ G_∞；且 Adam 生成的任意 θ_t 之间的距离有界：对任意 m, n ∈ {1, ..., T}，‖θ_n − θ_m‖₂ ≤ D，‖θ_m − θ_n‖∞ ≤ D_∞。则对一切 T ≥ 1，Adam 满足如下保证：

$$\frac{R(T)}{T} = O\left(\frac{1}{\sqrt{T}}\right)$$

该结果可以通过定理 4.1 与 $\sum_{i=1}^{d} \|g_{1:T,i}\|_2 \le dG_\infty \sqrt{T}$ 得到。因此，$\lim_{T \to \infty} \frac{R(T)}{T} = 0$。

## 5. 相关工作（Related Work）

与 Adam 直接相关的优化方法有 RMSProp (Tieleman & Hinton, 2012; Graves, 2013) 和 AdaGrad (Duchi et al., 2011)，这些关系在下面讨论。其他随机优化方法包括 vSGD (Schaul et al., 2012)、AdaDelta (Zeiler, 2012) 和来自 Roux & Fitzgibbon (2010) 的自然牛顿方法（natural Newton method），它们都通过一阶信息估计曲率来设定步长。函数求和优化器（Sum-of-Functions Optimizer, SFO）(Sohl-Dickstein et al., 2014) 是一种基于小批量的拟牛顿（quasi-Newton）方法，但（与 Adam 不同）其内存需求与数据集的小批量分区数量成线性关系，在 GPU 等内存受限的系统中往往不可行。与自然梯度下降（natural gradient descent, NGD）(Amari, 1998) 一样，Adam 使用一个自适应于数据几何结构的预处理器（preconditioner），因为 v̂_t 是 Fisher 信息矩阵对角线的近似 (Pascanu & Bengio, 2013)；但与普通 NGD 相比，Adam 的预处理器（和 AdaGrad 一样）在自适应上更保守——它使用对角 Fisher 信息矩阵近似之逆的平方根进行预处理。

RMSProp：与 Adam 密切相关的优化方法是 RMSProp (Tieleman & Hinton, 2012)。有时也使用带动量的版本 (Graves, 2013)。带动量的 RMSProp 与 Adam 之间有几个重要区别：带动量的 RMSProp 用重缩放梯度上的动量生成参数更新，而 Adam 的更新直接用梯度一阶矩和二阶矩的运行平均值估计。RMSProp 也缺少偏置校正项；这在 β_2 接近 1 的值（稀疏梯度情形下需要）时关系最大，因为在这种情况下不校正偏差会导致非常大的步长和经常性的发散，我们也在第 6.4 节中做了经验验证。

AdaGrad：一种对稀疏梯度效果良好的算法是 AdaGrad (Duchi et al., 2011)。它的基本版本把参数更新为 $\theta_{t+1} = \theta_t - \alpha \cdot g_t / \sqrt{\sum_{i=1}^{t} g_i^2}$。注意如果我们把 β_2 选得无限接近 1（从下方），那么 $\lim_{\beta_2 \to 1} \hat{v}_t = t^{-1} \cdot \sum_{i=1}^{t} g_i^2$。AdaGrad 对应 Adam 的一个 β_1 = 0、(1 − β_2) 无穷小、且 α 被替换为退火版本 $\alpha_t = \alpha \cdot t^{-1/2}$ 的版本，即 $\theta_t - \alpha \cdot t^{-1/2} \cdot \hat{m}_t / \lim_{\beta_2 \to 1} \hat{v}_t = \theta_t - \alpha \cdot t^{-1/2} \cdot g_t / (t \cdot \sum_{i=1}^{t} g_i^2) = \theta_t - \alpha \cdot g_t / \sqrt{\sum_{i=1}^{t} g_i^2}$。注意，去掉偏置校正项后，Adam 与 Adagrad 之间的这种直接对应关系不再成立；没有偏置校正（与 RMSProp 一样），无限接近 1 的 β_2 会导致无限大的偏差和无限大的参数更新。

## 6. 实验（Experiments）

为了经验评估所提方法，我们研究了不同的流行机器学习模型，包括逻辑回归（logistic regression）、多层全连接神经网络和深度卷积神经网络。利用大型模型和数据集，我们证明 Adam 能高效解决实际的深度学习问题。

比较不同优化算法时，我们使用相同的参数初始化。学习率和动量等超参数在密集网格上搜索，结果用最佳超参数设置报告。

### 6.1 实验：逻辑回归（Experiment: Logistic Regression）

我们使用 MNIST 数据集评估所提方法在带 L2 正则化的多类逻辑回归上的表现。逻辑回归具有被研究得很透彻的凸目标，适合在不用担心局部最小值问题的前提下比较不同优化器。我们逻辑回归实验中的步长 α 通过 1/√t 衰减调整，即 $\alpha_t = \frac{\alpha}{\sqrt{t}}$，这与我们第 4 节的理论预测一致。逻辑回归直接在 784 维图像向量上分类类别标签。我们用小批量大小 128，把 Adam 与带 Nesterov 动量的加速 SGD 和 Adagrad 进行比较。根据图 1，我们发现 Adam 的收敛性与带动量的 SGD 相似，两者都比 Adagrad 收敛得更快。

如 (Duchi et al., 2011) 所讨论，高效处理稀疏特征和梯度是 Adagrad 的主要理论结果之一，而 SGD 学习稀有特征的能力较差。步长带 1/√t 衰减的 Adam 理论上应该能匹配 Adagrad 的表现。我们用 (Maas et al., 2011) 的 IMDB 电影评论数据集考察稀疏特征问题。我们把 IMDB 电影评论预处理为词袋（bag-of-words, BoW）特征向量，包含前 10,000 个最常用词。每篇评论的 10,000 维 BoW 特征向量高度稀疏。如 (Wang & Manning, 2013) 所建议，训练时可以对 BoW 特征施加 50% 的 dropout 噪声以防过拟合。在图 1 中，Adagrad 在有无 dropout 噪声的情况下都大幅胜过带 Nesterov 动量的 SGD。Adam 的收敛速度与 Adagrad 一样快。Adam 的经验表现与我们第 2、4 节的理论发现一致。与 Adagrad 类似，Adam 可以利用稀疏特征，获得比普通带动量 SGD 更快的收敛速度。

**图 1：** MNIST 图像和带 10,000 维词袋（BoW）特征向量的 IMDB 电影评论上的逻辑回归训练负对数似然。

### 6.2 实验：多层神经网络（Experiment: Multi-Layer Neural Networks）

多层神经网络是带非凸目标函数的强大模型。虽然我们的收敛性分析不适用于非凸问题，但我们经验上发现 Adam 在这类情况下往往优于其他方法。在我们的实验中，我们做出的模型选择与该领域之前的论文一致：本实验使用带两个全连接隐藏层、每层 1000 个隐藏单元、ReLU 激活的神经网络模型，小批量大小为 128。

首先，我们研究使用标准确定性交叉熵目标函数、带 L2 权重衰减防止过拟合时不同优化器的表现。函数求和（SFO）方法 (Sohl-Dickstein et al., 2014) 是最近提出的、配合数据小批量工作的拟牛顿方法，在优化多层神经网络上表现出色。我们使用他们的实现，与 Adam 比较训练这类模型。图 2 表明，无论从迭代次数还是墙钟时间来看，Adam 的进展都更快。由于更新曲率信息的开销，SFO 每迭代比 Adam 慢 5-10 倍，且内存需求与小批量数量成线性关系。

dropout 这类随机正则化方法是防止过拟合的有效方式，并且因其简单而常在实践中使用。SFO 假设子函数是确定性的，在带随机正则化的代价函数上确实无法收敛。我们比较 Adam 与其他随机一阶方法在带 dropout 噪声训练的多层神经网络上的有效性。图 2 显示了我们的结果；Adam 的收敛性优于其他方法。

### 6.3 实验：卷积神经网络（Experiment: Convolutional Neural Networks）

带若干层卷积、池化和非线性单元的卷积神经网络（convolutional neural networks, CNNs）在计算机视觉任务上取得了相当大的成功。与大多数全连接神经网络不同，CNN 中的权共享导致不同层的梯度差异极大。实践中，用 SGD 时通常对卷积层使用较小的学习率。我们展示了 Adam 在深度 CNN 上的有效性。我们的 CNN 架构有三个交替阶段：5x5 卷积滤波器和步长 2 的 3x3 最大池化，后面接一个 1000 个整流线性隐藏单元（ReLU）的全连接层。输入图像经过白化预处理，dropout 噪声施加于输入层和全连接层。小批量大小同样设为 128，与之前的实验一致。

有趣的是，虽然如图 3（左）所示，Adam 和 Adagrad 在训练的初始阶段都迅速降低了代价，但如图 3（右）所示，对 CNN 而言，Adam 和 SGD 最终收敛得比 Adagrad 快得多。我们注意到二阶矩估计 v̂_t 在几个 epoch 后衰减到接近零，并被算法 1 中的 ε 主导。因此，与第 6.2 节的全连接网络相比，二阶矩估计对 CNN 中代价函数几何结构的近似较差。而通过一阶矩降低小批量方差对 CNN 更为重要，并促成了加速。因此，在这个特定实验中 Adagrad 的收敛比其他方法慢得多。虽然 Adam 相比带动量 SGD 只有边际改进，但它为不同层自适应学习率尺度，而不像 SGD 那样需要手工挑选。

**图 2：** 在 MNIST 图像上训练多层神经网络。(a) 使用 dropout 随机正则化的神经网络。(b) 使用确定性代价函数的神经网络。我们与函数求和（SFO）优化器 (Sohl-Dickstein et al., 2014) 比较。

**图 3：** 卷积神经网络训练代价。（左）前三个 epoch 的训练代价。（右）45 个 epoch 的训练代价。CIFAR-10，c64-c64-c128-1000 架构。

### 6.4 实验：偏置校正项（Experiment: Bias-Correction Term）

我们还经验评估了第 2、3 节所述偏置校正项的效果。如第 5 节所述，去掉偏置校正项会得到带动量的 RMSProp 版本 (Tieleman & Hinton, 2012)。训练变分自编码器（variational auto-encoder, VAE）时我们变化 β_1 和 β_2。VAE 的架构与 (Kingma & Welling, 2013) 相同：一个带 500 个隐藏单元、softplus 非线性的隐藏层，以及 50 维球面高斯潜变量。我们遍历了广泛的超参数选择，即 β_1 ∈ [0, 0.9]、β_2 ∈ {0.99, 0.999, 0.9999} 和 log₁₀(α) ∈ [−5, ..., −1]。接近 1 的 β_2 值——为对稀疏梯度稳健所必需——会产生更大的初始化偏差；因此我们预期偏置校正项在这种慢衰减情形下很重要，它可以防止对优化产生不利影响。

在图 4 中，接近 1 的 β_2 值在没有偏置校正项时确实导致训练不稳定，尤其是在训练的最初几个 epoch。最好的结果是在小的 (1 − β_2) 值配合偏置校正时取得的；这在优化后期更为明显，此时随着隐藏单元专门化到特定模式，梯度往往变得更稀疏。总之，无论超参数设置如何，Adam 的表现都与 RMSProp 相当或更好。

**图 4：** 在 (a) 10 个 epoch 后和 (b) 100 个 epoch 后，有偏置校正项（红线）与无偏置校正项（绿线）在损失（y 轴）上的效果，学习对象为变分自编码器（VAE）(Kingma & Welling, 2013)，对步长 α 的不同设置（x 轴）和超参数 β_1、β_2。

## 7. 扩展（Extensions）

### 7.1 AdaMax

在 Adam 中，单个权重的更新规则是把其梯度按其当前和过去梯度的（缩放）L2 范数的倒数比例缩放。我们可以把基于 L2 范数的更新规则推广到基于 Lp 范数的更新规则。这类变体在 p 变大时数值上变得不稳定。然而，在 p → ∞ 的特殊情况下，会出现一个惊人简单而稳定的算法；见算法 2。我们现在推导该算法。在 Lp 范数的情况下，设时间 t 的步长与 $v_t^{1/p}$ 成反比，其中：

$$v_t = \beta_2^p v_{t-1} + (1 - \beta_2^p) |g_t|^p \tag{6}$$
$$= (1 - \beta_2^p) \sum_{i=1}^{t} \beta_2^{p(t-i)} \cdot |g_i|^p \tag{7}$$

**算法 2：AdaMax**，一种基于无穷范数的 Adam 变体。详见第 7.1 节。本文测试的机器学习问题中效果良好的默认设置为 α = 0.002、β_1 = 0.9 和 β_2 = 0.999。用 β_1^t 表示 β_1 的 t 次幂。这里，(α/(1 − β_1^t)) 是带一阶矩偏置校正项的学习率。所有向量运算均为逐元素运算。

```
Require: α: 步长（Stepsize）
Require: β_1, β_2 ∈ [0, 1): 指数衰减率
Require: f(θ): 带参数 θ 的随机目标函数
Require: θ_0: 初始参数向量
   m_0 ← 0 （初始化一阶矩向量）
   u_0 ← 0 （初始化指数加权无穷范数）
   t ← 0 （初始化时间步）
   while θ_t 未收敛 do
     t ← t + 1
     g_t ← ∇_θ f_t(θ_{t-1}) （在时间步 t 获取关于随机目标的梯度）
     m_t ← β_1 · m_{t-1} + (1 − β_1) · g_t （更新有偏一阶矩估计）
     u_t ← max(β_2 · u_{t-1}, |g_t|) （更新指数加权无穷范数）
     θ_t ← θ_{t-1} − (α/(1 − β_1^t)) · m_t / u_t （更新参数）
   end while
   return θ_t （最终参数）
```

注意，衰减项在这里等价地参数化为 β_2^p 而不是 β_2。现在令 p → ∞，并定义 u_t = lim_{p→∞} (v_t)^{1/p}，则：

$$u_t = \lim_{p \to \infty} (v_t)^{1/p} = \lim_{p \to \infty} (1 - \beta_2^p) \sum_{i=1}^{t} \beta_2^{p(t-i)} \cdot |g_i|^p \tag{8}$$
$$= \lim_{p \to \infty} (1 - \beta_2^p)^{1/p} \left( \sum_{i=1}^{t} \beta_2^{p(t-i)} \cdot |g_i|^p \right)^{1/p} \tag{9}$$
$$= \lim_{p \to \infty} \left( \sum_{i=1}^{t} \left( \beta_2^{(t-i)} \cdot |g_i| \right)^p \right)^{1/p} \tag{10}$$
$$= \max \left( \beta_2^{t-1} |g_1|, \beta_2^{t-2} |g_2|, \ldots, \beta_2 |g_{t-1}|, |g_t| \right) \tag{11}$$

这对应一个非常简单的递归公式：

$$u_t = \max(\beta_2 \cdot u_{t-1}, |g_t|) \tag{12}$$

初始值 u_0 = 0。注意，方便的是，这种情况下我们不需要校正初始化偏差。还要注意，使用 AdaMax 时参数更新的量级有比 Adam 更简单的界，即 |Δ_t| ≤ α。

### 7.2 时间平均（Temporal Averaging）

由于随机近似带来的噪声，最后一步迭代是嘈杂的，通过平均往往能获得更好的泛化性能。此前在 Moulines & Bach (2011) 中，Polyak-Ruppert 平均 (Polyak & Juditsky, 1992; Ruppert, 1988) 已被证明能改善标准 SGD 的收敛，其中 $\bar{\theta}_t = \frac{1}{t} \sum_{k=1}^{n} \theta_k$。或者，可以使用参数上的指数移动平均，对更近的参数值赋予更高权重。这可以轻松实现：只需在算法 1 和 2 的内循环中加一行：$\bar{\theta}_t \leftarrow \beta_2 \cdot \bar{\theta}_{t-1} + (1 - \beta_2)\theta_t$，且 θ̄₀ = 0。初始化偏差可以再次通过估计量 $\hat{\theta}_t = \bar{\theta}_t / (1 - \beta_2^t)$ 校正。

## 8. 结论（Conclusion）

我们提出了一种简单且计算高效的、用于随机目标函数基于梯度优化的算法。我们的方法面向具有大规模数据集和/或高维参数空间的机器学习问题。该方法结合了两种近来的流行优化方法的优势：AdaGrad 处理稀疏梯度的能力，以及 RMSProp 处理非平稳目标的能力。该方法实现简单、内存需求小。实验证实了凸问题上收敛速率的分析。总体而言，我们发现 Adam 稳健，适合机器学习领域广泛的各种非凸优化问题。

## 9. 致谢（Acknowledgments）

This paper would probably not have existed without the support of Google Deepmind. We would like to give special thanks to Ivo Danihelka, and Tom Schaul for coining the name Adam. Thanks to Kai Fan from Duke University for spotting an error in the original AdaMax derivation. Experiments in this work were partly carried out on the Dutch national e-infrastructure with the support of SURF Foundation. Diederik Kingma is supported by the Google European Doctorate Fellowship in Deep Learning.

---

**参考文献略**

## 10. 附录（Appendix）

### 10.1 收敛性证明（Convergence Proof）

**定义 10.1。** 如果对任意 x, y ∈ R^d、任意 λ ∈ [0, 1]，都有

$$\lambda f(x) + (1 - \lambda) f(y) \ge f(\lambda x + (1 - \lambda) y)$$

则函数 f : R^d → R 是凸的。

另外注意，凸函数可以以其切点处的超平面作为下界。

**引理 10.2。** 如果函数 f : R^d → R 是凸的，则对任意 x, y ∈ R^d，

$$f(y) \ge f(x) + \nabla f(x)^T (y - x)$$

上面的引理可以用来给出遗憾的上界，我们对主定理的证明就是把超平面替换为 Adam 更新规则构造的。下面两个引理用于支撑我们的主定理。我们还使用一些定义简化记号，其中 g_t = ∇f_t(θ_t)，g_{t,i} 为其第 i 个元素。我们定义 g_{1:t,i} ∈ R^t 为包含到 t 为止所有迭代中梯度的第 i 维的向量，即 g_{1:t,i} = [g_{1,i}, g_{2,i}, · · · , g_{t,i}]。

**引理 10.3。** 设 g_t = ∇f_t(θ_t)，g_{1:t} 按上述定义且有界：‖g_t‖₂ ≤ G，‖g_t‖∞ ≤ G_∞。则

$$\sum_{t=1}^{T} \frac{g_{t,i}^2}{\sqrt{t}} \le 2G_\infty \|g_{1:T,i}\|_2$$

*证明。* 我们将通过对 T 的归纳证明该不等式。

T = 1 的基本情形：我们有 $\sqrt{g_{1,i}^2} \le 2G_\infty \|g_{1,i}\|_2$。

对于归纳步骤，

$$\sum_{t=1}^{T} \frac{g_{t,i}^2}{\sqrt{t}} = \sum_{t=1}^{T-1} \frac{g_{t,i}^2}{\sqrt{t}} + \frac{g_{T,i}^2}{\sqrt{T}} \le 2G_\infty \|g_{1:T-1,i}\|_2 + \frac{g_{T,i}^2}{\sqrt{T}} = 2G_\infty \sqrt{\|g_{1:T,i}\|_2^2 - g_{T,i}^2} + \frac{g_{T,i}^2}{\sqrt{T}}$$

由 $\|g_{1:T,i}\|_2^2 - g_{T,i}^2 + \frac{g_{T,i}^4}{4\|g_{1:T,i}\|_2^2} \ge \|g_{1:T,i}\|_2^2 - g_{T,i}^2$，我们可以对两边开平方根，得到

$$\sqrt{\|g_{1:T,i}\|_2^2 - g_{T,i}^2} \le \|g_{1:T,i}\|_2 - \frac{g_{T,i}^2}{2\|g_{1:T,i}\|_2} \le \|g_{1:T,i}\|_2 - \frac{g_{T,i}^2}{\sqrt{T} \cdot 2 G_\infty^2}$$

重排不等式并代入 $\sqrt{\|g_{1:T,i}\|_2^2 - g_{T,i}^2}$ 项，

$$G_\infty \sqrt{\|g_{1:T,i}\|_2^2 - g_{T,i}^2} + \frac{g_{T,i}^2}{\sqrt{T}} \le 2G_\infty \|g_{1:T,i}\|_2$$

**引理 10.4。** 设 $\gamma \triangleq \frac{\beta_1}{\sqrt{\beta_2}}$。对满足 $\frac{\beta_1}{\sqrt{\beta_2}} < 1$ 的 β_1, β_2 ∈ [0, 1) 以及有界的 g_t（‖g_t‖₂ ≤ G，‖g_t‖∞ ≤ G∞），以下不等式成立：

$$\sum_{t=1}^{T} \frac{\hat{m}_{t,i}^2}{\sqrt{t \hat{v}_{t,i}}} \le \frac{2}{(1 - \gamma)\sqrt{1 - \beta_2}} \|g_{1:T,i}\|_2$$

*证明。* 在上述假设下，$\frac{1 - \beta_2^t}{(1 - \beta_1^t)^2} \le \frac{1}{(1 - \beta_1)^2}$。我们可以用算法 1 中的更新规则展开求和中的最后一项：

$$\sum_{t=1}^{T} \frac{\hat{m}_{t,i}^2}{\sqrt{t \hat{v}_{t,i}}} = \sum_{t=1}^{T-1} \frac{\hat{m}_{t,i}^2}{\sqrt{t \hat{v}_{t,i}}} + \frac{\sqrt{1 - \beta_2^T}}{(1 - \beta_1^T)^2} \frac{\left( \sum_{k=1}^{T} (1 - \beta_1)\beta_1^{T-k} g_{k,i} \right)^2}{\sqrt{T \sum_{j=1}^{T} (1 - \beta_2)\beta_2^{T-j} g_{j,i}^2}}$$

$$\le \sum_{t=1}^{T-1} \frac{\hat{m}_{t,i}^2}{\sqrt{t \hat{v}_{t,i}}} + \frac{\sqrt{1 - \beta_2^T}}{(1 - \beta_1^T)^2} \sum_{k=1}^{T} \frac{T\left( (1 - \beta_1)\beta_1^{T-k} g_{k,i} \right)^2}{\sqrt{T \sum_{j=1}^{T} (1 - \beta_2)\beta_2^{T-j} g_{j,i}^2}}$$

$$\le \sum_{t=1}^{T-1} \frac{\hat{m}_{t,i}^2}{\sqrt{t \hat{v}_{t,i}}} + \frac{\sqrt{1 - \beta_2^T}}{(1 - \beta_1^T)^2} \sum_{k=1}^{T} \frac{T\left( (1 - \beta_1)\beta_1^{T-k} g_{k,i} \right)^2}{\sqrt{T (1 - \beta_2)\beta_2^{T-k} g_{k,i}^2}}$$

$$\le \sum_{t=1}^{T-1} \frac{\hat{m}_{t,i}^2}{\sqrt{t \hat{v}_{t,i}}} + \frac{\sqrt{1 - \beta_2^T}(1 - \beta_1)^2}{(1 - \beta_1^T)^2 \sqrt{T(1 - \beta_2)}} \sum_{k=1}^{T} \left( \frac{\beta_1^2}{\beta_2} \right)^{T-k} \|g_{k,i}\|_2$$

$$\le \sum_{t=1}^{T-1} \frac{\hat{m}_{t,i}^2}{\sqrt{t \hat{v}_{t,i}}} + \frac{1}{\sqrt{T(1 - \beta_2)}} \sum_{k=1}^{T} \gamma^{T-k} \|g_{k,i}\|_2$$

类似地，我们可以给出求和其余各项的上界。

$$\sum_{t=1}^{T} \frac{\hat{m}_{t,i}^2}{\sqrt{t \hat{v}_{t,i}}} \le \sum_{t=1}^{T} \frac{\|g_{t,i}\|_2}{\sqrt{t(1 - \beta_2)}} \sum_{j=0}^{T-t} \gamma^j$$

$$\le \sum_{t=1}^{T} \frac{\|g_{t,i}\|_2}{\sqrt{t(1 - \beta_2)}} \sum_{j=0}^{T} \gamma^j$$

对于 γ < 1，利用算术-几何级数的上界 $\sum_t t\gamma^t < \frac{1}{(1 - \gamma)^2}$：

$$\sum_{t=1}^{T} \frac{\|g_{t,i}\|_2}{\sqrt{t(1 - \beta_2)}} \sum_{j=0}^{T} \gamma^j \le \frac{1}{(1 - \gamma)^2\sqrt{1 - \beta_2}} \sum_{t=1}^{T} \frac{\|g_{t,i}\|_2}{\sqrt{t}}$$

应用引理 10.3，

$$\sum_{t=1}^{T} \frac{\hat{m}_{t,i}^2}{\sqrt{t \hat{v}_{t,i}}} \le \frac{2G_\infty}{(1 - \gamma)^2 \sqrt{1 - \beta_2}} \|g_{1:T,i}\|_2$$

为简化记号，我们定义 $\gamma \triangleq \frac{\beta_1}{\sqrt{\beta_2}}$。直观地说，我们的以下定理在学习率 α_t 以 t^{-1/2} 的速率衰减、一阶矩运行平均系数 β_{1,t} 以 λ 指数衰减（λ 通常接近 1，如 1 − 10⁻⁸）时成立。

**定理 10.5。** 假设函数 f_t 有有界梯度：对所有 θ ∈ R^d，‖∇f_t(θ)‖₂ ≤ G，‖∇f_t(θ)‖∞ ≤ G∞；且 Adam 生成的任意 θ_t 之间的距离有界：对任意 m, n ∈ {1, ..., T}，‖θ_n − θ_m‖₂ ≤ D，‖θ_m − θ_n‖∞ ≤ D∞；并且 β_1, β_2 ∈ [0, 1) 满足 $\frac{\beta_1}{\sqrt{\beta_2}} < 1$。令 $\alpha_t = \frac{\alpha}{\sqrt{t}}$，β_{1,t} = β_1 λ^{t-1}，λ ∈ (0, 1)。则对一切 T ≥ 1，Adam 满足如下保证：

$$R(T) \le \frac{D^2}{2\alpha(1 - \beta_1)} \sum_{i=1}^{d} \sqrt{T \hat{v}_{T,i}} + \frac{\alpha(\beta_1 + 1)G_\infty}{(1 - \beta_1)\sqrt{1 - \beta_2}(1 - \gamma)^2} \sum_{i=1}^{d} \|g_{1:T,i}\|_2 + \sum_{i=1}^{d} \frac{D_\infty^2 G_\infty \sqrt{1 - \beta_2}}{2\alpha(1 - \beta_1)(1 - \lambda)^2}$$

*证明。* 利用引理 10.2，我们有，

$$f_t(\theta_t) - f_t(\theta^*) \le g_t^T (\theta_t - \theta^*) = \sum_{i=1}^{d} g_{t,i}(\theta_{t,i} - \theta_{*,i})$$

由算法 1 中给出的更新规则，

$$\theta_{t+1} = \theta_t - \alpha_t \frac{\hat{m}_t}{\sqrt{\hat{v}_t}} = \theta_t - \frac{\alpha_t}{1 - \beta_1^t} \left( \frac{\beta_{1,t}}{\sqrt{\hat{v}_t}} m_{t-1} + \frac{(1 - \beta_{1,t})}{\sqrt{\hat{v}_t}} g_t \right)$$

我们聚焦于参数向量 θ_t ∈ R^d 的第 i 维。在上面的更新规则两边减去标量 θ_{*,i} 并平方，我们有，

$$(\theta_{t+1,i} - \theta_{*,i})^2 = (\theta_{t,i} - \theta_{*,i})^2 - \frac{2\alpha_t}{1 - \beta_1^t} \left( \frac{\beta_{1,t}}{\sqrt{\hat{v}_{t,i}}} m_{t-1,i} + \frac{(1 - \beta_{1,t})}{\sqrt{\hat{v}_{t,i}}} g_{t,i} \right) (\theta_{t,i} - \theta_{*,i}) + \alpha_t^2 \left( \frac{\hat{m}_{t,i}}{\sqrt{\hat{v}_{t,i}}} \right)^2$$

我们可以重排上述方程并利用 Young 不等式 ab ≤ a²/2 + b²/2。另外，可以证明

$$\sqrt{\hat{v}_{t,i}} = \sqrt{\frac{\sum_{j=1}^{t} (1 - \beta_2)\beta_2^{t-j} g_{j,i}^2}{1 - \beta_2^t}} \le \|g_{1:t,i}\|_2$$

且 β_{1,t} ≤ β_1。于是

$$g_{t,i}(\theta_{t,i} - \theta_{*,i}) = \frac{(1 - \beta_1^t)\sqrt{\hat{v}_{t,i}}}{2\alpha_t(1 - \beta_{1,t})} \left[ (\theta_{t,i} - \theta_{*,i})^2 - (\theta_{t+1,i} - \theta_{*,i})^2 \right] + \frac{\beta_{1,t}}{(1 - \beta_{1,t})} \frac{\sqrt{\hat{v}_{t-1,i}}}{2\alpha_{t-1}} (\theta_{*,i} - \theta_{t,i})^2 + \frac{\alpha_t (1 - \beta_1^t) \hat{v}_{t,i}}{2(1 - \beta_{1,t})} \left( \frac{\hat{m}_{t,i}}{\sqrt{\hat{v}_{t,i}}} \right)^2$$

$$\le \frac{1}{2\alpha_t(1 - \beta_1)} \left[ (\theta_{t,i} - \theta_{*,i})^2 - (\theta_{t+1,i} - \theta_{*,i})^2 \right] \sqrt{\hat{v}_{t,i}} + \frac{\beta_{1,t}}{2\alpha_{t-1}(1 - \beta_{1,t})} (\theta_{*,i} - \theta_{t,i})^2 \sqrt{\hat{v}_{t-1,i}} + \frac{\beta_1 \alpha_{t-1} m_{t-1,i}^2}{2(1 - \beta_1)\sqrt{\hat{v}_{t-1,i}}} + \frac{\alpha_t \hat{m}_{t,i}^2}{2(1 - \beta_1)\sqrt{\hat{v}_{t,i}}}$$

我们对上述不等式应用引理 10.4，并对 f_t(θ_t) − f_t(θ*) 上界中的所有维度 i ∈ 1, ..., d 以及 t ∈ 1, ..., T 的凸函数序列求和，得到遗憾界：

$$R(T) \le \sum_{i=1}^{d} \frac{1}{2\alpha_1(1 - \beta_1)} (\theta_{1,i} - \theta_{*,i})^2 \sqrt{\hat{v}_{1,i}} + \sum_{i=1}^{d} \sum_{t=2}^{T} \frac{1}{2(1 - \beta_1)} (\theta_{t,i} - \theta_{*,i})^2 \left( \frac{\sqrt{\hat{v}_{t,i}}}{\alpha_t} - \frac{\sqrt{\hat{v}_{t-1,i}}}{\alpha_{t-1}} \right)$$

$$+ \frac{\beta_1 \alpha G_\infty}{(1 - \beta_1)\sqrt{1 - \beta_2}(1 - \gamma)^2} \sum_{i=1}^{d} \|g_{1:T,i}\|_2 + \frac{\alpha G_\infty}{(1 - \beta_1)\sqrt{1 - \beta_2}(1 - \gamma)^2} \sum_{i=1}^{d} \|g_{1:T,i}\|_2$$

$$+ \sum_{i=1}^{d} \sum_{t=1}^{T} \frac{\beta_{1,t}}{2\alpha_t(1 - \beta_{1,t})} (\theta_{*,i} - \theta_{t,i})^2 \sqrt{\hat{v}_{t,i}}$$

由假设 ‖θ_t − θ*‖₂ ≤ D、‖θ_m − θ_n‖∞ ≤ D∞，我们有：

$$R(T) \le \frac{D^2}{2\alpha(1 - \beta_1)} \sum_{i=1}^{d} \sqrt{T \hat{v}_{T,i}} + \frac{\alpha(1 + \beta_1)G_\infty}{(1 - \beta_1)\sqrt{1 - \beta_2}(1 - \gamma)^2} \sum_{i=1}^{d} \|g_{1:T,i}\|_2 + \frac{D_\infty^2}{2\alpha} \sum_{i=1}^{d} \sum_{t=1}^{T} \frac{\beta_{1,t}}{(1 - \beta_{1,t})} \sqrt{t \hat{v}_{t,i}}$$

$$\le \frac{D^2}{2\alpha(1 - \beta_1)} \sum_{i=1}^{d} \sqrt{T \hat{v}_{T,i}} + \frac{\alpha(1 + \beta_1)G_\infty}{(1 - \beta_1)\sqrt{1 - \beta_2}(1 - \gamma)^2} \sum_{i=1}^{d} \|g_{1:T,i}\|_2 + \frac{D_\infty^2 G_\infty \sqrt{1 - \beta_2}}{2\alpha} \sum_{i=1}^{d} \sum_{t=1}^{T} \frac{\beta_{1,t}}{(1 - \beta_{1,t})} \sqrt{t}$$

我们可以对最后一项使用算术-几何级数上界：

$$\sum_{t=1}^{T} \frac{\beta_{1,t}}{(1 - \beta_{1,t})} \sqrt{t} \le \sum_{t=1}^{T} \frac{1}{(1 - \beta_1)} \lambda^{t-1} \sqrt{t} \le \sum_{t=1}^{T} \frac{1}{(1 - \beta_1)} \lambda^{t-1} t \le \frac{1}{(1 - \beta_1)(1 - \lambda)^2}$$

因此，我们得到如下遗憾界：

$$R(T) \le \frac{D^2}{2\alpha(1 - \beta_1)} \sum_{i=1}^{d} \sqrt{T \hat{v}_{T,i}} + \frac{\alpha(1 + \beta_1)G_\infty}{(1 - \beta_1)\sqrt{1 - \beta_2}(1 - \gamma)^2} \sum_{i=1}^{d} \|g_{1:T,i}\|_2 + \sum_{i=1}^{d} \frac{D_\infty^2 G_\infty \sqrt{1 - \beta_2}}{2\alpha \beta_1 (1 - \lambda)^2}$$