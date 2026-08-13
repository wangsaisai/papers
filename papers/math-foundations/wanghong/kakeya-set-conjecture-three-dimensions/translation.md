# 凸集并集的体积估计与三维 Kakeya 集猜想（Volume Estimates for Unions of Convex Sets, and the Kakeya Set Conjecture in Three Dimensions）

**作者**: Hong Wang（王虹）, Joshua Zahl
**翻译日期**: 2026-07-19

---

## 摘要（Abstract）

我们研究 $\mathbb{R}^3$ 中 $\delta$ 管的集合，其性质是不可能有太多管同时包含在同一个凸集 $V$ 中。我们证明，来自这样的集合的管之并集必然具有几乎最大的体积。作为推论，我们证明 $\mathbb{R}^3$ 中的每个 Kakeya 集都具有 Minkowski 和 Hausdorff 维数 3。

## 1. 引言（Introduction）

Kakeya 集是 $\mathbb{R}^n$ 中包含每个方向上单位线段的紧致子集。Kakeya 集猜想断言 $\mathbb{R}^n$ 中的每个 Kakeya 集都具有 Minkowski 和 Hausdorff 维数 $n$。当 $n = 2$ 时，这个猜想由 Davies [5] 证明，但在三维及更高维中仍然开放。关于 Kakeya 猜想的介绍和历史进展综述参见 [17, 28]。关于三维及更高维中猜想的当前进展参见 [15, 16, 18, 20, 21, 27, 31]。

本文的目的是获得 $\mathbb{R}^3$ 中满足某些非聚集条件的 $\delta$-管（即单位线段的 $\delta$-邻域）并集体积的下界。作为推论，我们解决了三维中的 Kakeya 集猜想。

**定理 1.1**. $\mathbb{R}^3$ 中的每个 Kakeya 集都具有 Minkowski 和 Hausdorff 维数 3。

定理 1.1 是以下稍微更技术性结果的推论。

**定理 1.2**. 对所有 $\varepsilon > 0$，存在 $K > 1$ 使得以下对所有足够小的 $\delta > 0$ 成立。设 $\mathcal{T}$ 是包含在 $\mathbb{R}^3$ 单位球中的 $\delta$-管集合，并假设每个尺寸为 $a \times b \times 2$ 的长方棱柱包含来自 $\mathcal{T}$ 的最多 $100ab\delta^{-2}$ 个管（例如，如果 $\mathcal{T}$ 中的管指向 $\delta$-分离的方向，则这是真的）。对每个 $T \in \mathcal{T}$，设 $Y(T) \subset T$ 是可测集，满足 $|Y(T)| \ge \lambda|T|$。则

$$\left|\bigcup_{T \in \mathcal{T}} Y(T)\right| \ge \delta^\varepsilon \lambda^K \sum_{T \in \mathcal{T}} |T|. \tag{1.1}$$

Kakeya 极大函数猜想断言对每个 $\varepsilon > 0$，不等式 (1.1) 对 $K = 3$ 成立。$\mathbb{R}^2$ 中的 Kakeya 极大函数猜想由 Cordoba [6] 证明。虽然我们没有解决 $\mathbb{R}^3$ 中的 Kakeya 极大函数猜想，但定理 1.2 中给出的较弱陈述足以得到定理 1.1。

每个 $a \times b \times 2$ 长方棱柱包含来自 $\mathcal{T}$ 的最多 $100ab\delta^{-2}$ 个管的假设是一种非聚集条件。这个假设的一个密切变体首先由 Wolff 在 [27] 中引入，满足这个假设的管集合被称为满足 Wolff 公理。

### 1.1. 定理 1.2 与多尺度分析

在 [25, 26] 中，作者证明了当集合 $\mathcal{T}$ 具有称为粘性（stickiness）的性质时，定理 1.2 成立（参见图 1（左））。粗略地说，如果 $\mathcal{T}$ 满足定理 1.2 的非聚集条件；基数约为 $\delta^{-2}$；并且对每个中间尺度 $\delta \le \rho \le 1$，$\mathcal{T}$ 中的管可以被一组 $\rho$-管覆盖，这些 $\rho$-管满足定理 1.2 中用 $\rho$ 替换 $\delta$ 的非聚集条件，则 $\mathcal{T}$ 是粘性的。

不幸的是，并非每个管集合都是粘性的——图 1（右）给出了一个例子。图 1（右）中说明的排列难以分析，因为 $\rho$-管以大的重数相交（即许多 $\rho$-管通过一个典型点），但每个 $\rho$-管内部的 $\delta$-管是稀疏的（即每个 $\rho$-管内部 $\delta$-管的并集只填充该 $\rho$-管的一小部分）。为了帮助我们分析这种类型的排列，在第 1.2 节中，我们将引入定理 1.2 中非聚集假设的两个变体，以及体积估计 (1.1) 的两个变体。

### 1.2. 凸集的并集与非聚集

在下文中，我们说一对集合 $U, V \subset \mathbb{R}^n$ 是本质不同的，如果 $|U \cap V| \le \frac{1}{2} \max(|U|, |V|)$。

**定义 1.3**（凸集公理）. 设 $\mathcal{T}$ 是 $\mathbb{R}^3$ 中 $\delta$-管的集合。我们说 $\mathcal{T}$ 满足凸集公理，如果对每个凸集 $V \subset \mathbb{R}^3$，

$$|\{T \in \mathcal{T} : T \subset V\}| \le K_V \frac{|V|}{\delta^2}.$$

我们说 $\mathcal{T}$ 满足强凸集公理，如果对每个凸集 $V \subset \mathbb{R}^3$ 和每个 $T \in \mathcal{T}$，

$$|\{T' \in \mathcal{T} : T' \subset V, T' \text{ 与 } T \text{ 平行}\}| \le K_V \frac{|V|}{\delta^2}.$$

**定义 1.4**（Frostman 板公理）. 设 $\mathcal{T}$ 是 $\mathbb{R}^3$ 中 $\delta$-管的集合。我们说 $\mathcal{T}$ 满足 Frostman 板公理，如果对每个 $\delta$-板 $P$（即尺寸为 $a \times b \times 2$ 的长方棱柱，其中 $a, b \ge \delta$），

$$|\{T \in \mathcal{T} : T \subset P\}| \le K_P \frac{|P|}{\delta^2}.$$

**定义 1.5**（本质不同的管）. 设 $\mathcal{T}$ 是 $\mathbb{R}^3$ 中 $\delta$-管的集合。我们说 $\mathcal{T}$ 中的管是本质不同的，如果对每个 $\delta$-板 $P$，$\mathcal{T}$ 中包含在 $P$ 中的管的数量最多为 $K_P \frac{|P|}{\delta^2}$。

**命题 1.6**（断言 D 和 E 等价）. 设 $\mathcal{T}$ 是 $\mathbb{R}^3$ 中 $\delta$-管的集合。则以下断言等价：

(D) $\mathcal{T}$ 满足凸集公理，且对每个 $T \in \mathcal{T}$，有 $|T| \ge \lambda |\mathcal{T}| \delta^2$。

(E) 对每个 $T \in \mathcal{T}$，存在可测集 $Y(T) \subset T$ 满足 $|Y(T)| \ge \lambda |T|$，使得

$$\left|\bigcup_{T \in \mathcal{T}} Y(T)\right| \ge \delta^\varepsilon \lambda^K \sum_{T \in \mathcal{T}} |T|.$$

**命题 1.7**（从断言 D 和 E 到 Kakeya 集猜想）. 设 $\mathcal{T}$ 是 $\mathbb{R}^3$ 中 $\delta$-管的集合，满足凸集公理。则

$$\left|\bigcup_{T \in \mathcal{T}} T\right| \ge \delta^\varepsilon |\mathcal{T}| \delta^2.$$

作为推论，$\mathbb{R}^3$ 中的每个 Kakeya 集都具有 Minkowski 和 Hausdorff 维数 3。

**命题 1.8**（Wolff 的刷子论证）. 设 $\mathcal{T}$ 是 $\mathbb{R}^3$ 中 $\delta$-管的集合，满足凸集公理。则

$$\left|\bigcup_{T \in \mathcal{T}} T\right| \ge \delta^\varepsilon |\mathcal{T}| \delta^2.$$

### 1.3. 从断言 D 和 E 到 Kakeya 集猜想

在本节中，我们概述从定理 1.2 到定理 1.1 的推导。这个推导相对标准，基于 Wolff [27] 和 Katz-Tao [14] 的先前工作。

设 $K \subset \mathbb{R}^3$ 是一个 Kakeya 集。对每个 $\delta > 0$，设 $\mathcal{T}_\delta$ 是 $\delta$-管的集合，使得每个管包含在 $K$ 的某个平移中，并且管指向 $\delta$-分离的方向。则 $|\mathcal{T}_\delta| \approx \delta^{-2}$。

为了证明 $K$ 具有 Hausdorff 维数 3，我们需要证明对每个 $\varepsilon > 0$，存在常数 $C_\varepsilon > 0$ 使得

$$|K|_\delta \ge C_\varepsilon \delta^{-3+\varepsilon},$$

其中 $|K|_\delta$ 表示 $K$ 的 $\delta$-覆盖数。

设 $\mathcal{T}_\delta$ 是满足定理 1.2 假设的 $\delta$-管集合。则对每个 $T \in \mathcal{T}_\delta$，我们有 $|T| \approx \delta^2$。应用定理 1.2（取 $\lambda = 1$），我们得到

$$|K|_\delta \ge \left|\bigcup_{T \in \mathcal{T}_\delta} T\right| \ge \delta^\varepsilon \sum_{T \in \mathcal{T}_\delta} |T| \approx \delta^\varepsilon |\mathcal{T}_\delta| \delta^2 \approx \delta^{-3+\varepsilon}.$$

这完成了推导。

### 1.4. 证明哲学与先前工作

我们的证明建立在 Kakeya 集猜想的先前工作基础上。特别是，我们受到了以下工作的启发：

- **Wolff [27]**：引入了非聚集假设（Wolff 公理），并证明了三维中 Kakeya 集的 Hausdorff 维数至少为 $5/2 + \varepsilon$。
- **Katz-Tao [14]**：改进了 Wolff 的结果，证明了 Hausdorff 维数至少为 $5/2 + 1/10^{13}$。
- **Guth [12]**：引入了"颗粒分解"技术，用于分析管的分布。
- **Katz-Zahl [15]**：将 Guth 的技术与多尺度分析结合，进一步改进了维数下界。

我们的证明结合了这些技术，并引入了新的想法：

1. **多尺度分析**：将问题分解为不同尺度上的子问题，通过归纳法从大尺度到小尺度逐步分析管的分布。
2. **凸集分解**：将空间分解为一系列凸集（称为"颗粒"），每个凸集包含一定数量的管。通过分析这些颗粒的几何结构，证明管的并集必须具有足够大的体积。
3. **非聚集条件的强化**：引入更强的非聚集条件（强凸集公理），使得分析更加精细。

### 1.5. 证明概要

在本节中，我们概述证明的主要步骤。详细的证明将在后续章节中给出。

**步骤 1：多尺度分解**

设 $\delta = 2^{-n}$，我们将问题分解为 $n$ 个尺度上的子问题。对每个尺度 $2^{-k}$（$k = 0, 1, \ldots, n$），我们分析管在该尺度上的分布。

**步骤 2：颗粒分解**

在每个尺度上，我们将空间分解为一系列凸集（颗粒），每个颗粒包含一定数量的管。颗粒的尺寸和形状取决于管的分布。

**步骤 3：归纳论证**

我们使用归纳法，从大尺度到小尺度逐步证明体积估计。归纳假设是：在尺度 $2^{-k}$ 上，管的并集具有足够大的体积。

**步骤 4：非聚集条件的应用**

在每个尺度上，我们应用非聚集条件（凸集公理）来控制管的分布。关键观察是：如果管的并集体积很小，那么必然存在某个凸集包含了太多的管，这与非聚集条件矛盾。

**步骤 5：体积估计的组合**

最后，我们将所有尺度上的体积估计组合起来，得到最终的体积估计。

### 1.6. 管加倍与 Keleti 的线段扩展猜想

在本节中，我们讨论与 Kakeya 集猜想相关的两个问题：管加倍和 Keleti 的线段扩展猜想。

**管加倍问题**：设 $\mathcal{T}$ 是 $\mathbb{R}^3$ 中 $\delta$-管的集合。管加倍问题问：$\mathcal{T}$ 中管的并集的最小可能体积是多少？

这个问题与 Kakeya 集猜想密切相关。如果 $\mathcal{T}$ 满足非聚集条件，那么管加倍问题的答案就是定理 1.2 中给出的体积估计。

**Keleti 的线段扩展猜想**：设 $K \subset \mathbb{R}^3$ 是一个紧致集，具有 Hausdorff 维数 $> 2$。Keleti 猜想 [13] 断言：存在一个包含 $K$ 的 Kakeya 集。

这个猜想如果为真，将直接推出 Kakeya 集猜想。因为如果存在 Hausdorff 维数 $> 2$ 的 Kakeya 集，那么 Kakeya 集的 Hausdorff 维数必须至少为 3。

### 1.7. 致谢

作者感谢 Larry Guth、Nets Katz 和 Pablo Shmerkin 的有益讨论。这项工作部分受到 NSF 的支持。

## 参考文献（References）

*参考文献略*