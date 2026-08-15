# 线性代数的艺术——《Linear Algebra for Everyone》图解笔记（The Art of Linear Algebra – Graphic Notes on "Linear Algebra for Everyone"）

**作者**: Kenji Hiranabe，Gilbert Strang 协助
**翻译日期**: 2026-08-15

---

## 摘要（Abstract）

我尝试为 Gilbert Strang 在书籍《Linear Algebra for Everyone》中介绍的矩阵的重要概念进行可视化图释，以促进从矩阵分解的角度对向量、矩阵计算和算法的理解。它们包括矩阵分解（Column-Row, CR）、高斯消去法（Gaussian Elimination, LU）、格拉姆-施密特正交化（Gram-Schmidt Orthogonalization, QR）、特征值和对角化（Eigenvalues and Diagonalization, QΛQᵀ）、和奇异值分解（Singular Value Decomposition, UΣVᵀ）。

## 序言（Preface）

我很高兴能看到 Kenji Hiranabe 的线性代数中的矩阵运算的图片！这样的图片是展示代数的绝佳方式。我们当然可以通过行·列的点乘来想象矩阵乘法，但那绝非全部——它是"线性组合"与"秩 1 矩阵"组成的代数与艺术。我很感激能看到日文翻译的书籍和 Kenji 的图片中的想法。

——Gilbert Strang，麻省理工学院数学教授

## 1. 理解矩阵——4 个视角（Understanding Matrices – 4 Views）

一个矩阵（m × n）可以被视为 1 个矩阵、mn 个数、n 个列和 m 个行。

图 1：从四个角度理解矩阵

$$A = \begin{bmatrix} a_{11} & a_{12} \\ a_{21} & a_{22} \\ a_{31} & a_{32} \end{bmatrix} = \begin{bmatrix} | & | \\ a_1 & a_2 \\ | & | \end{bmatrix} = \begin{bmatrix} — a_1^* — \\ — a_2^* — \\ — a_3^* — \end{bmatrix}$$

在这里，列向量被标记为粗体 $a_1$。行向量则有一个 $*$ 号，标记为 $a_1^*$。转置向量和矩阵则用 $T$ 标记为 $a^T$ 和 $A^T$。

## 2. 向量乘以向量——2 个视角（Vector × Vector – 2 Views）

后文中，我将介绍一些概念，同时列出《Linear Algebra for Everyone》一书中的相应部分（部分编号插入如下）。详细的内容最好看书，这里我也添加了一个简短的解释，以便您可以通过这篇文章尽可能多地理解。此外，每个图都有一个简短的名称，例如 v1（数字 1 表示向量的乘积）、Mv1（数字 1 表示矩阵和向量的乘积），以及如下图（v1）所示的彩色圆圈。如你所见，随着讨论的进行，该名称将被交叉引用。

- 1.1 节（p.2）Linear combination and dot products
- 1.3 节（p.25）Matrix of Rank One
- 1.4 节（p.29）Row way and column way

图 2：向量乘以向量 - (v1), (v2)

(v1) 是两个向量之间的基础运算，而 (v2) 将列乘以行并产生一个秩 1 矩阵。理解 (v2) 的结果（秩 1）是接下来章节的关键。

## 3. 矩阵乘以向量——2 个视角（Matrix × Vector – 2 Views）

一个矩阵乘以一个向量将产生三个点积组成的向量（Mv1）和一种 A 的列向量的线性组合。

- 1.1 节（p.3）Linear combinations
- 1.3 节（p.21）Matrices and Column Spaces

图 3：矩阵乘以向量 - (Mv1), (Mv2)

往往你会先学习 (Mv1)。但当你习惯了从 (Mv2) 的视角看待它，会理解 $Ax$ 是 $A$ 的列的线性组合。矩阵 $A$ 的列向量的所有线性组合生成的子空间记为 $C(A)$。$Ax = 0$ 的解空间则是零空间，记为 $N(A)$。

同理，由 (vM1) 和 (vM2) 可见，行向量乘以矩阵也是同一种理解方式。

图 4：向量乘以矩阵 - (vM1), (vM2)

$A$ 的行向量的所有线性组合生成的子空间记为 $C(A^T)$。$yA = 0$ 的解空间是 $A$ 的左零空间，记为 $N(A^T)$。

本书的一大亮点即为四个基本子空间：在 $\mathbb{R}^n$ 上的 $N(A) + C(A^T)$（相互正交）和在 $\mathbb{R}^m$ 上的 $N(A^T) + C(A)$（相互正交）。

- 3.5 节（p.124）Dimensions of the Four Subspaces

图 5：四个子空间

关于秩 $r$，请见 A = CR（6.1 节）。

## 4. 矩阵乘以矩阵——4 个视角（Matrix × Matrix – 4 Views）

由"矩阵乘以向量"自然延伸到"矩阵乘以矩阵"。

- 1.4 节（p.35）Four ways to multiply AB = C
- 也可以见书的封底

图 6：矩阵乘以矩阵 - (MM1), (MM2), (MM3), (MM4)

## 5. 实用模式（Practical Patterns）

在这里，我展示了一些实用的模式，可以让你更直观地理解接下来的内容。

图 7：图 1, 2 - (P1), (P1)

P1 是 (MM2) 和 (Mv2) 的结合。P2 是 (MM3) 和 (vM2) 的扩展。注意，P1 是列运算（右乘一个矩阵），而 P2 是行运算（左乘一个矩阵）。

图 8：图 1′, 2′ - (P1′), (P2′)

(P1′) 将对角线上的数乘以矩阵的列，而 (P2′) 将对角线上的数乘以矩阵的行。两个分别为 (P1) 和 (P2) 的变体。

图 9：图 3 - (P3)

当解决微分方程和递归方程时也会出现这一模式：

- 6 节（p.201）Eigenvalues and Eigenvectors
- 6.4 节（p.243）Systems of Differential Equations

$$\frac{du(t)}{dt} = Au(t), \quad u(0) = u_0$$
$$u_{n+1} = Au_n, \quad u_0 = u_0$$

在两种问题中，它的解都可以用 $A$ 的特征值（$\lambda_1, \lambda_2, \lambda_3$）、特征向量 $X = \begin{bmatrix} x_1 & x_2 & x_3 \end{bmatrix}$ 和系数 $c = \begin{bmatrix} c_1 & c_2 & c_3 \end{bmatrix}^T$ 表示。其中 $C$ 是以 $X$ 为基底的初始值 $u(0) = u_0$ 的坐标。

$$u_0 = c_1 x_1 + c_2 x_2 + c_3 x_3$$
$$c = \begin{bmatrix} c_1 \\ c_2 \\ c_3 \end{bmatrix} = X^{-1} u_0$$

以上两个问题的通解为：

$$u(t) = e^{At} u_0 = X e^{\Lambda t} X^{-1} u_0 = X e^{\Lambda t} c = c_1 e^{\lambda_1 t} x_1 + c_2 e^{\lambda_2 t} x_2 + c_3 e^{\lambda_3 t} x_3$$
$$u_n = A^n u_0 = X \Lambda^n X^{-1} u_0 = X \Lambda^n c = c_1 \lambda_1^n x_1 + c_2 \lambda_2^n x_2 + c_3 \lambda_3^n x_3$$

见 Figure 9：通过 P3 可以得到 $XDc$。

图 10：Pattern 4 - (P4)

P4 在特征值分解和奇异值分解中都会用到。两种分解都可以表示为三个矩阵之积，其中中间的矩阵均为对角矩阵。且都可以表示为带特征值/奇异值系数的秩 1 矩阵之积。更多细节将在下一节中讨论。

## 6. 矩阵的五种分解（Five Matrix Factorizations）

- 前言 p.vii, The Plan for the Book.

$A = CR$、$A = LU$、$A = QR$、$A = Q\Lambda Q^T$、$A = U\Sigma V^T$ 将一一说明。

Table 1：五种分解

| 分解 | 说明 |
|---|---|
| $A = CR$ | $C$ 为 $A$ 的线性无关列，$R$ 为 $A$ 的行阶梯形矩阵，可推知列秩 = 行秩 |
| $A = LU$ | LU 分解通过高斯消去法（下三角）（上三角） |
| $A = QR$ | QR 分解为格拉姆-施密特正交化中的正交矩阵 $Q$ 和三角矩阵 $R$ |
| $S = Q\Lambda Q^T$ | 对称矩阵 $S$ 可以进行特征值分解，特征向量组成 $Q$，特征值组成 $\Lambda$ |
| $A = U\Sigma V^T$ | 所有矩阵 $A$ 的奇异值分解，奇异值组成 $\Sigma$ |

### 6.1 A = CR

- 1.4 节 Matrix Multiplication and A = CR（p.29）

所有一般的长矩阵 $A$ 都有相同的行秩和列秩。这个分解是理解这一定理最直观的方法。$C$ 由 $A$ 的线性无关列组成，$R$ 为 $A$ 的行阶梯形矩阵（消除了零行）。$A = CR$ 将 $A$ 化简为 $r$ 的线性无关列 $C$ 和线性无关行 $R$ 的乘积。

$$A = CR = \begin{bmatrix} 1 & 2 \\ 2 & 3 \end{bmatrix} \begin{bmatrix} 1 & 2 & 1 & 0 \\ 0 & 1 & 0 & 1 \end{bmatrix}$$

推导过程：从左往右看 $A$ 的列。保留其中线性无关的列，去掉可以由前者线性表出的列。则第 1、2 列被保留，而第三列因为可以由前两列之和表示而被去掉。而要通过线性无关的 1、2 两列重新构造出 $A$，需要右乘一个行阶梯矩阵 $R$。

图 11：CR 中列的秩

现在你会发现行的秩为 2，因为 $C$ 中只有 2 个线性无关列。而 $A$ 中所有的列都可以由 $C$ 中的 2 列线性表出。

图 12：CR 中行的秩

同样，列秩也为 2，因为 $R$ 中只有 2 个线性无关行，且 $A$ 中所有的行都可以由 $R$ 中的 2 行线性表出。

### 6.2 A = LU

用高斯消除法求解 $Ax = b$ 也被称为 LU 分解。通常，是 $A$ 左乘一个初等行变换矩阵（$E$）来得到一个上三角矩阵 $U$。

$$EA = U$$
$$A = E^{-1}U$$
$$\text{令 } L = E^{-1}, \quad A = LU$$

现在，求解 $Ax = b$ 有 2 步：（1）求解 $Lc = b$，（2）代回 $Ux = c$。

- 2.3 节（p.57）Matrix Computations and A = LU

在这里，我们直接通过 $A$ 计算 $L$ 和 $U$。

$$A = \begin{bmatrix} | \\ l_1 \\ | \end{bmatrix} —u_1^* — + \begin{bmatrix} 0 & 0 & 0 \\ 0 \\ 0 \end{bmatrix} = \begin{bmatrix} | \\ l_1 \\ | \end{bmatrix} —u_1^* — + \begin{bmatrix} | \\ l_2 \\ | \end{bmatrix} —u_2^* — + \begin{bmatrix} 0 & 0 & 0 \\ 0 & 0 & 0 \\ 0 & 0 & A_3 \end{bmatrix} = LU$$

图 13：$A$ 的递归秩 1 矩阵分离

要计算 $L$ 和 $U$，首先分离出由 $A$ 的第一行和第一列组成的外积。余下的部分为 $A_2$。递归执行此操作，将 $A$ 分解为秩 1 矩阵之和。

图 14：由 LU 重新构造 $A$

由 $L$ 乘以 $U$ 来重新构造 $A$ 则相对简单。

### 6.3 A = QR

$A = QR$ 是在保持 $C(A) = C(Q)$ 的条件下，将 $A$ 转化为正交矩阵 $Q$。

- 4.4 节 Orthogonal matrices and Gram-Schmidt（p.165）

在格拉姆-施密特正交化中，首先，单位化的 $a_1$ 被用作 $q_1$，然后求出 $a_2$ 与 $q_1$ 正交所得到的 $q_2$，以此类推。

$$q_1 = a_1 / \|a_1\|$$
$$q_2 = a_2 - (q_1^T a_2) q_1, \quad q_2 = q_2 / \|q_2\|$$
$$q_3 = a_3 - (q_1^T a_3) q_1 - (q_2^T a_3) q_2, \quad q_3 = q_3 / \|q_3\|$$

或者你也可以写作 $r_{ij} = q_i^T a_j$：

$$a_1 = r_{11} q_1$$
$$a_2 = r_{12} q_1 + r_{22} q_2$$
$$a_3 = r_{13} q_1 + r_{23} q_2 + r_{33} q_3$$

原本的 $A$ 就可以表示为 QR：正交矩阵乘以上三角矩阵。

$$A = \begin{bmatrix} | & | & | \\ q_1 & q_2 & q_3 \\ | & | & | \end{bmatrix} \begin{bmatrix} r_{11} & r_{12} & r_{13} \\ & r_{22} & r_{23} \\ & & r_{33} \end{bmatrix} = QR$$

$$QQ^T = Q^T Q = I$$

图 15：A = QR

$A$ 的列向量就可以转化为一个正交集合：$Q$ 的列向量。$A$ 的每一个列向量都可以用 $Q$ 和上三角矩阵 $R$ 重新构造出。图释可以回头看 P1。

### 6.4 S = QΛQᵀ

所有对称矩阵 $S$ 都必须有实特征值和正交特征向量。特征值是 $\Lambda$ 的对角元素，特征向量在 $Q$ 中。

- 6.3 节（p.227）Symmetric Positive Definite Matrices

$$S = Q\Lambda Q^T = \begin{bmatrix} | & | & | \\ q_1 & q_2 & q_3 \\ | & | & | \end{bmatrix} \begin{bmatrix} \lambda_1 & & \\ & \lambda_2 & \\ & & \lambda_3 \end{bmatrix} \begin{bmatrix} — q_1^T — \\ — q_2^T — \\ — q_3^T — \end{bmatrix}$$

$$= \lambda_1 \begin{bmatrix} | \\ q_1 \\ | \end{bmatrix} — q_1^T — + \lambda_2 \begin{bmatrix} | \\ q_2 \\ | \end{bmatrix} — q_2^T — + \lambda_3 \begin{bmatrix} | \\ q_3 \\ | \end{bmatrix} — q_3^T —$$

$$= \lambda_1 P_1 + \lambda_2 P_2 + \lambda_3 P_3$$

$$P_1 = q_1 q_1^T, \quad P_2 = q_2 q_2^T, \quad P_3 = q_3 q_3^T$$

图 16：S = QΛQᵀ

一个对称矩阵 $S$ 通过一个正交矩阵 $Q$ 和它的转置矩阵，对角化为 $\Lambda$。然后被分解为一阶投影矩阵 $P = qq^T$ 的组合。这就是谱定理。

注意，这里的分解用到了 P4。

$$S = S^T = \lambda_1 P_1 + \lambda_2 P_2 + \lambda_3 P_3$$
$$QQ^T = P_1 + P_2 + P_3 = I$$
$$P_1 P_2 = P_2 P_3 = P_3 P_1 = O$$
$$P_1^2 = P_1 = P_1^T, \quad P_2^2 = P_2 = P_2^T, \quad P_3^2 = P_3 = P_3^T$$

### 6.5 A = UΣVᵀ

- 7.1 节（p.259）Singular Values and Singular Vectors

包括长方阵在内的所有矩阵都具有奇异值分解（SVD）。$A = U\Sigma V^T$ 中，有 $A$ 的奇异向量 $U$ 和 $V$。奇异值则排列在 $\Sigma$ 的对角线上。下图就是"简化版"的 SVD。

图 17：A = UΣVᵀ

你可以发现，$V$ 是 $\mathbb{R}^n$（$A^T A$ 的特征向量）的标准正交基，而 $U$ 是 $\mathbb{R}^m$（$AA^T$ 的特征向量）的标准正交基。它们共同将 $A$ 对角化为 $\Sigma$。这也可以表示为秩 1 矩阵的线性组合。

$$A = U\Sigma V^T = \begin{bmatrix} | & | & | \\ u_1 & u_2 & u_3 \\ | & | & | \end{bmatrix} \begin{bmatrix} \sigma_1 & & \\ & \sigma_2 & \\ & & \end{bmatrix} \begin{bmatrix} — v_1^T — \\ — v_2^T — \end{bmatrix}$$

$$= \sigma_1 \begin{bmatrix} | \\ u_1 \\ | \end{bmatrix} — v_1^T — + \sigma_2 \begin{bmatrix} | \\ u_2 \\ | \end{bmatrix} — v_2^T —$$

$$= \sigma_1 u_1 v_1^T + \sigma_2 u_2 v_2^T$$

注意：

$$UU^T = I_m$$
$$VV^T = I_n$$

图释见 P4。

## 总结和致谢（Summary and Acknowledgements）

我展示了矩阵/向量乘法的系统可视化与它们在五种矩阵分解中的应用。我希望你能够喜欢它们、通过它们加深对线性代数的理解。

Ashley Fernandes 在排版时帮我美化了这篇论文，使它更加一致和专业。在结束这篇论文之前，我要感谢 Gilbert Strang 教授出版了《Linear Algebra for Everyone》一书。它引导我们通过新的视角去了解线性代数中这些美丽的风景。其中介绍了当代和传统的数据科学和机器学习，每个人都可以通过实用的方式对它的基本思想进行基本理解。矩阵世界的重要组成部分。

参考文献略
