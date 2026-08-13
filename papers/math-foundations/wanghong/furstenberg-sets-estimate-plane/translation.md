# 平面中 Furstenberg 集的估计（Furstenberg Sets Estimate in the Plane）

**作者**: Kevin Ren, Hong Wang（王虹）
**翻译日期**: 2026-07-19

---

## 摘要（Abstract）

我们完全解决了 $\mathbb{R}^2$ 中的 Furstenberg 集猜想：一个 $(s, t)$-Furstenberg 集具有 Hausdorff 维数 $\ge \min\left(s+t, \frac{3s+t}{2}, s+1\right)$。作为结果，我们获得了离散和积问题的 Elekes 界限的类比，并解决了 Oberlin 的一个正交投影问题。

## 1. 引言（Introduction）

对于 $s \in (0, 1]$ 和 $t \in (0, 2]$，一个 $(s, t)$-Furstenberg 集是具有以下性质的集合 $E \subset \mathbb{R}^2$：存在一个直线族 $\mathcal{L}$，满足 $\dim_H \mathcal{L} \ge t$，且对所有 $\ell \in \mathcal{L}$ 有 $\dim_H (E \cap \ell) \ge s$。Hausdorff 维数 $\dim_H \mathcal{L}$ 定义为 $\mathcal{L}$ 在 $\mathbb{R}^2$ 中所有仿射直线构成的空间 $\mathcal{A}(2,1)$ 中的 Hausdorff 维数（参见 [26, 第 3.16 节]）。Furstenberg 集问题问：$\dim_H (E)$ 的最小可能维数是多少？

受其关于 $\times 2, \times 3$-不变集工作 [15] 的启发，Furstenberg 考虑了具有以下性质的集合 $E$：存在一个直线族 $\mathcal{L}$，每个方向一条直线，使得对所有 $\ell \in \mathcal{L}$ 有 $\dim_H (E \cap \ell) \ge s$。他猜想（在未发表的工作中，但在 Wolff [43, 注 1.5] 中被回忆）这样的集合 $E$（这是一个特殊的 $(s, 1)$-Furstenberg 集）具有 Hausdorff 维数 $\ge \frac{3s+1}{2}$。

Furstenberg 的猜想后来被推广到 $(s, t)$-Furstenberg 集：任何 $(s, t)$-Furstenberg 集 $E \subset \mathbb{R}^2$ 具有 Hausdorff 维数

$$\dim_H E \ge \min\left\{s + t, \frac{3s + t}{2}, s + 1\right\}.$$

正如 Wolff [43] 所观察到的，猜想的界 $\frac{3s+t}{2}$ 对于 Szemerédi-Trotter 定理 [40] 的连续版本是尖锐的。界 $s + t$ 对于 $0 \le t \le s \le 1$ 由 Lutz-Stull [23] 和 Héra-Shmerkin-Yavicoli [19] 证明，界 $s + 1$ 对于 $s + t \ge 2$ 的情况由 Fu 和第一作者 [14] 最近证明。Wolff [43] 在 $t = 1$ 时给出了 $\frac{3s+t}{2}$ 的尖锐构造（基于网格例子，可轻松推广到其他 $t$ 值），$s + t$ 的尖锐构造可取为 $t$-维 Cantor 集与 $s$-维 Cantor 集的乘积，Fu 和第一作者 [14] 给出了 $s + 1$ 的尖锐构造（区间与 $s$-维 Cantor 集的乘积）。

主要情况 $s < t < 2 - s$ 仍然开放。Wolff [43] 在 $t = 1$ 情况下的初等界、Molter-Rela [28]、Héra [18] 和 Lutz-Stull [23] 给出

$$\dim_H E \ge \max(t/2 + s, s + \min(s, t)).$$

同时，Katz-Tao [21] 和 Bourgain [1] 在特殊情况 $(s, t) = (1/2, 1)$ 下证明了 $\varepsilon$-改进 $\dim_H (E) \ge 2s + \varepsilon(s, t)$，然后 Héra-Shmerkin-Yavicoli [19] 对一般 $(s, 2s)$ 证明了这一点。Benedetto-Zahl [7] 获得了 $(s, 2s)$-Furstenberg 集的显式改进。

Orponen-Shmerkin [30] 的突破证明了对一般 $s < t$ 的 $(s, t)$ 超过 $2s$ 的 $\varepsilon$-改进（关于此结果如何导致近期进展参见第 1.3 节）。后来，Shmerkin 和第二作者 [38] 使用 [30] 中的思想提供了超过界 $s + 2t$ 的 $\varepsilon$-改进。最近，Orponen 和 Shmerkin [31] 取得了另一个突破，证明了 $\dim_H E \ge \max(t/2 + s, s + \min(s, t)) + C(t, s)$，其中 $C(t, s)$ 是仅依赖于 $s$ 和 $t$ 的良好显式常数，接近于猜想。在本文中，我们证明了 $(s, t)$-Furstenberg 集的 Hausdorff 维数的 Furstenberg 猜想。

**定理 1.1**. 固定 $s \in (0, 1]$ 和 $t \in (0, 2]$。任何 $(s, t)$-Furstenberg 集 $F \subset \mathbb{R}^2$ 具有 Hausdorff 维数

$$\dim_H F \ge \min\left\{s + t, \frac{3s + t}{2}, s + 1\right\}.$$

$(s, t)$-Furstenberg 问题已知与离散和积问题及投影问题相关，如第 1.1 节、第 1.2 节和 [31, 第 6 节] 所示。

最近，Wang 和 Wu [42] 证明了我们定理 1.1 的"两端"推广，并将其与 Bourgain-Demeter 隔离理论结合，在 Stein 的 Fourier 限制猜想上取得了进展。他们的证明将定理 1.1 作为黑箱，并结合多尺度分析和尺度归纳；我们请读者参阅该论文和 [6] 了解更多细节。

### 1.1. 正交投影的例外集估计

设 $K \subset \mathbb{R}^2$ 是任意 Borel 集，$0 \le u \le \min\{\dim_H K, 1\}$。正交投影的例外集问题要求对以下集合给出最优上界

$$\dim_H \{\theta \in S^1 : \dim_H \pi_\theta(K) < u\}.$$

1968 年，Kaufman [22] 证明了

$$\dim_H \{\theta \in S^1 : \dim_H \pi_\theta(K) < u\} \le u,$$

改进了 Marstrand [24] 的结果。2000 年，Peres-Schlag [33] 证明了当 $\dim_H K > 1$ 时，可以改进估计为

$$\dim_H \{\theta \in S^1 : \dim_H \pi_\theta(K) < u\} \le \max\{u + 1 - \dim_H K, 0\}.$$

Kaufman 的界在 $u = \dim_H K$ 时是尖锐的，Peres-Schlag 界在 $u = 1$ 时是尖锐的。事实上，情况 $u = 1$ 之前由 Falconer [11] 得到。对于一般情况，Oberlin [29] 猜想右边应该是 $\max\{2u - \dim_H K, 0\}$。

例外集估计的进一步改进源于 Bourgain 的投影定理 [2]，它说

$$\dim_H \{\theta \in S^1 : \dim_H \pi_\theta(K) \le \dim_H K/2\} = 0. \tag{2}$$

Bourgain 的投影定理涉及实数的深层性质，已被用于 Bourgain-Furman-Lindenstrauss-Mozes [3] 中的环面上随机游走和 Katz-Zahl [20] 中的 $\mathbb{R}^3$ 中 Kakeya 集的 Hausdorff 维数。例外集估计的近期改进 [30, 31] 与 Furstenberg 集问题的改进相关。

作为定理 1.1（实际上是其离散版本定理 4.1）的推论，我们证明了 Oberlin 关于正交投影例外集的猜想 [29, (1.8)]，其 AD-正则版本已在 [31, 定理 1.13] 中建立。这个推导类似于用于建立 [31, 定理 1.13] 的过程（参见 [31, 第 6.1 节]），所以我们省略细节。

**定理 1.2**. 设 $K \subset \mathbb{R}^2$ 是任意 Borel 集。则对所有 $0 \le u \le \min\{\dim_H(K), 1\}$，我们有

$$\dim_H \{\theta \in S^1 : \dim_H \pi_\theta(K) < u\} \le \max\{2u - \dim_H K, 0\}.$$

### 1.2. 离散和积

设 $F$ 是一个域，$A, B \subset F$，和集定义为 $A + B := \{a + b : a \in A, b \in B\}$，积集定义为 $A \cdot B := \{a \cdot b : a \in A, b \in B\}$。

和积问题问 $A + A$ 或 $A \cdot A$ 的大小是否可以比 $A$ 的大小大得多。当 $A$ 是有限集时，我们用基数度量 $A$ 的大小，Erdös 和 Szemerédi 猜想当 $F = \mathbb{R}$ 时，对任意 $\varepsilon > 0$，存在 $C_\varepsilon > 0$ 使得

$$\max\{|A + A|, |A \cdot A|\} \ge C_\varepsilon |A|^{2-\varepsilon}.$$

关于 Erdös-Szemerédi 和积猜想的一些最新结果参见 [39, 27, 34, 35, 36, 37]。当 $F = \mathbb{R}$ 时，离散和积问题的"分形"类比涉及 $(\delta, s, C)$-集，这是 $\mathbb{R}$ 中 Hausdorff 维数 $\ge s$ 的集合的离散化。对于有界集 $A \subset \mathbb{R}^d$，令 $|A|_\delta$ 表示 $A$ 的二进 $\delta$-覆盖数。这个概念将在定义 2.2 中精确定义，但现在可以认为 $|A|_\delta$（相差一个常数因子）与覆盖 $A$ 所需的最少 $\delta$-球数相同。

**定义 1.3**（$(\delta, s, C)$-集）. 对于 $\delta \in 2^{-\mathbb{N}}$，$s \in [0, d]$ 和 $C > 0$，非空有界集 $A \subset \mathbb{R}^d$ 称为 $(\delta, s, C)$-集，如果

$$|A \cap B(x, r)|_\delta \le C r^s |A|_\delta, \quad \forall x \in \mathbb{R}^d, r \in [\delta, 1].$$

如果 $A$ 是 $(\delta, s, C)$-集，则 $|A|_\delta \ge C^{-1} \delta^{-s}$，如果 $A' \subset A$ 满足 $|A'|_\delta \ge C'^{-1} |A|_\delta$，则 $A'$ 是 $(\delta, s, CC')$-集。

定义 1.3 与 $(\delta, s, C)$-Katz-Tao 集的概念形成对比，后者要求界

$$|A \cap B(x, r)|_\delta \le C \left(\frac{r}{\delta}\right)^s, \quad \forall x \in \mathbb{R}^d, r \in [\delta, 1].$$

**定义 1.4**（AD-正则集）. 一个 $(\delta, s, C)$-集 $A$ 称为在尺度 $\delta$ 上是 Alfors-David 正则的，如果 $C > 0$ 是不依赖于 $\delta$ 的常数，且

$$C^{-1} r^s |A|_\delta \le |A \cap B(x, r)|_\delta \le C r^s |A|_\delta, \quad \forall x \in A, r \in [\delta, 1].$$

我们说 $A$ 是 $(s, \varepsilon)$-几乎 AD-正则的，如果 $C = \delta^{-\varepsilon}$。注意在上述条件中，我们要求 $x \in A$，而不是 $x \in \mathbb{R}^d$。

Katz 和 Tao [21] 提出了以下离散环猜想，并证明它等价于 $(1/2, 1)$-Furstenberg 集问题和 Hausdorff 维数为 1 的平面集的 Falconer 距离集问题的 $\varepsilon$-改进。当 $\varepsilon > 0$ 足够小时，如果 $A \subset [1, 2]$ 是 $s = 1/2$ 的 $(\delta, s, \delta^{-\varepsilon})$-集，则

$$\max\{|A + A|_\delta, |A \cdot A|_\delta\} \ge C_{s,\varepsilon} \delta^{-s-\varepsilon}.$$

这个猜想在 2003 年由 Bourgain [1] 证明。（在相关工作中，Edgar 和 Miller [9] 证明了不存在 $\mathbb{R}$ 的 $s$-维 Borel 子环的定性陈述。）后来 Bourgain 将猜想推广到上述投影估计 (2)。最近，Guth-Katz-Zahl [16] 使用受有限域中和积问题 Gaarev 方法启发的初等方法证明了一个定量结果，将 $\varepsilon$ 替换为依赖于 $s$ 的显式常数。

作为定理 1.2 的推论，我们为离散和积问题证明了一个新估计，这补充了 Elekes [10] 对 $\mathbb{R}$ 和 Mohammadi-Stevens [27] 对有限域 $\mathbb{F}_p$ 的类似结果。这个推导类似于用于建立 [31, 定理 1.26] 的过程（参见 [31, 第 5.2 节]），所以我们省略细节。

**定理 1.5**. 给定 $s \in (0, \frac{2}{3})$ 和 $\eta > 0$，存在 $\varepsilon = \varepsilon(s, \eta) > 0$ 使得以下对足够小的 $\delta > 0$ 成立。设 $A \subset [1, 2]$ 是 $(\delta, s, \delta^{-\varepsilon})$-集。则

$$\max\{|A + A|_\delta, |A \cdot A|_\delta\} \ge \delta^{-(5/4)s+\eta}.$$

定理 1.5 改进了最近 Máthé-O'Regan [25, 定理 B] 和 Orponen-Shmerkin [31, 定理 1.26] 的结果，其中 $5/4$ 被替换为 $7/6$，并将 [31, 定理 1.26] 对几乎 AD-正则集的结果推广到一般 $(\delta, s, C)$-集。

**注 1.6**. 如果 $s \ge \frac{3}{2}$，则 [14, 推论 1.7] 给出下界 $\delta^{-\frac{s+1}{2}+\eta}$。这个界是尖锐的，参见 [25, 注 1.13]。

虽然和积定理可以视为 Furstenberg 集问题的特殊情况，但不对称和积，即不同集合 $A, B, C \subset [0, 1]$ 的 $A + B \cdot C$，在 Orponen 和 Shmerkin 最近关于 Furstenberg 集猜想的工作 [31] 中起着重要作用。

### 1.3. 路线图

我们的证明建立在先前关于投影问题和管之间关联的工作基础上。我们说明下面不同结果之间的关系。

Bourgain 的离散和积定理 [1]/Bourgain 的投影定理 [2] $\Rightarrow$ Furstenberg 集估计的 $\varepsilon$-改进 [30] $\Rightarrow$ 平面中尖锐的径向投影定理 [32] $\Rightarrow$ ABC-和积定理 [31], [8] $\Rightarrow$ 几乎 AD-正则集的尖锐例外集估计 [31] $\Rightarrow$ 几乎 AD-正则集的尖锐 Furstenberg 集估计 [31] + 良好间隔集的尖锐 Furstenberg 集估计 [17] $\Rightarrow$ 尖锐的 Furstenberg 集估计。

在 [31] 中，Furstenberg 猜想对几乎 AD-正则集得到解决。另一方面，在 [17] 中，Furstenberg 猜想对良好间隔集得到解决（相关结果另见 [13]）。在本文中，我们将推广 [17] 中的主要结果，并对一类半良好间隔集解决 Furstenberg 猜想。然后我们将此结果与几乎 AD-正则结果结合，使用源于 [30] 的尺度归纳方案完成 Furstenberg 集猜想的证明。

## 参考文献（References）

*参考文献略*