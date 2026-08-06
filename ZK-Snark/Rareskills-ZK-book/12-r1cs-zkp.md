# 从 R1CS 构建零知识证明

给定一个用[算术电路](https://www.rareskills.io/post/arithmetic-circuit)表示、并编码为[秩一约束系统（Rank-1 Constraint System，R1CS）](https://www.rareskills.io/post/rank-1-constraint-system)的计算，可以为“拥有见证（witness）”这一命题创建 ZK 证明，尽管这种证明并不简洁。本文将介绍如何实现这一点。

为 R1CS 构建零知识证明，需要把见证向量转换成[有限域上的椭圆曲线点](https://www.rareskills.io/post/elliptic-curves-finite-fields)，并把每一行的逐元素乘积（Hadamard product）替换为[双线性配对](https://www.rareskills.io/post/bilinear-pairing)。

给定一个秩一约束系统，其中每个矩阵都有 $n$ 行、$m$ 列，可以写成：

$$
\mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a}=\mathbf{O}\mathbf{a}
$$

其中 $\mathbf{L}$、$\mathbf{R}$、$\mathbf{O}$ 是 $n$ 行 $m$ 列的矩阵，$\mathbf{a}$ 是见证向量，其中包含算术电路所有信号的一组满足赋值（satisfying assignment）。向量 $\mathbf{a}$ 有 1 列、$m$ 行，$\circ$ 表示逐元素乘积。

展开后如下：

$$
\left[ \begin{array}{ccc}
l_{1,1} & \cdots & l_{1,m} \\
\vdots & \ddots & \vdots \\
l_{n,1} & \cdots & l_{n,m}
\end{array} \right]
\left[ \begin{array}{c}
a_1 \\
\vdots \\
a_m
\end{array} \right]
\circ
\left[ \begin{array}{ccc}
r_{1,1} & \cdots & r_{1,m} \\
\vdots & \ddots & \vdots \\
r_{n,1} & \cdots & r_{n,m}
\end{array} \right]
\left[ \begin{array}{c}
a_1 \\
\vdots \\
a_m
\end{array} \right]
=
\left[ \begin{array}{ccc}
o_{1,1} & \cdots & o_{1,m} \\
\vdots & \ddots & \vdots \\
o_{n,1} & \cdots & o_{n,m}
\end{array} \right]
\left[ \begin{array}{c}
a_1 \\
\vdots \\
a_m
\end{array} \right]
$$

$$
=
\left[ \begin{array}{ccc}
a_1 l_{1,1} + \cdots + a_m l_{1,m} \\
\vdots \\
a_1 l_{n,1} + \cdots + a_m l_{n,m}
\end{array} \right]
\circ
\left[ \begin{array}{ccc}
a_1 r_{1,1} + \cdots + a_m r_{1,m} \\
\vdots \\
a_1 r_{n,1} + \cdots + a_m r_{n,m}
\end{array} \right]
=
\left[ \begin{array}{ccc}
a_1 o_{1,1} + \cdots + a_m o_{1,m} \\
\vdots \\
a_1 o_{n,1} + \cdots + a_m o_{n,m}
\end{array} \right]
$$

$$
=
\left[ \begin{array}{ccc}
\sum_{i=1}^m a_i l_{1,i} \\
\sum_{i=1}^m a_i l_{2,i} \\
\vdots \\
\sum_{i=1}^m a_i l_{n,i}
\end{array} \right]
\circ
\left[ \begin{array}{ccc}
\sum_{i=1}^m a_i r_{1,i} \\
\sum_{i=1}^m a_i r_{2,i} \\
\vdots \\
\sum_{i=1}^m a_i r_{n,i}
\end{array} \right]
=
\left[ \begin{array}{ccc}
\sum_{i=1}^m a_i o_{1,i} \\
\sum_{i=1}^m a_i o_{2,i} \\
\vdots \\
\sum_{i=1}^m a_i o_{n,i}
\end{array} \right]
$$

$$
=
\begin{array}{ccc}
\sum_{i=1}^m a_i l_{1,i}  \sum_{i=1}^m a_i r_{1,i} = \sum_{i=1}^m a_i o_{1,i} \\
\sum_{i=1}^m a_i l_{2,i}  \sum_{i=1}^m a_i r_{2,i} = \sum_{i=1}^m a_i o_{2,i} \\
\vdots \\
\sum_{i=1}^m a_i l_{n,i}  \sum_{i=1}^m a_i r_{n,i} = \sum_{i=1}^m a_i o_{n,i}
\end{array}
$$

在这种设置下，只要把见证向量 $\mathbf{a}$ 交给验证者，就能证明自己拥有满足 R1CS 的见证向量 $\mathbf{a}$；但显而易见，这并不是零知识证明！

## R1CS 的零知识证明算法
如果把见证向量的每一项分别乘以 $G_1$ 或 $G_2$ 来“加密”，数学关系仍能正常成立！

为了理解这一点，考虑矩阵乘法：

$$
\begin{bmatrix}
1 & 2 \\
3 & 4 \\
\end{bmatrix}
\begin{bmatrix}
4 \\
5
\end{bmatrix}
= \begin{bmatrix}
14 \\
32
\end{bmatrix}
$$

以及：

$$
\begin{bmatrix}
1 & 2 \\
3 & 4 \\
\end{bmatrix}
\begin{bmatrix}
4G_1 \\
5G_1
\end{bmatrix}
= \begin{bmatrix}
14G_1 \\
32G_1
\end{bmatrix}
$$

第二个矩阵乘法所得两个椭圆曲线点的离散对数，与第一个矩阵乘法所得元素具有相同的值。

换句话说，每当用方阵的一行乘以列向量时，都会执行两次椭圆曲线标量乘法和一次椭圆曲线点加法。

### 椭圆曲线记号
$[aG_1]_1$ 表示由域元素 $a$ 乘以 $G_1$ 得到的 $\mathbb{G}_1$ 椭圆曲线点。$[aG_2]_2$ 表示由 $a$ 乘以生成元 $G_2$ 得到的 $\mathbb{G}_2$ 椭圆曲线点。由于离散对数问题，给定 $[aG_1]_1$ 或 $[aG_2]_2$ 时，无法提取 $a$。给定点 $A \in \mathbb{G}_1$ 和 $B \in \mathbb{G}_2$，用 $A\bullet B$ 表示这两个点的配对。

### 证明者步骤
把 $\mathbf{a}$ 向量的每一项乘以生成元点 $G_1$，得到椭圆曲线点 $[a_iG_1]_1$，从而对向量进行加密。

对矩阵 $\mathbf{L}$，执行以下运算：

$$
\left[ \begin{array}{ccc}
l_{1,1} & \cdots & l_{1,m} \\
\vdots & \ddots & \vdots \\
l_{n,1} & \cdots & l_{n,m}
\end{array} \right]
\left[ \begin{array}{c}
[a_1 G_1]_1 \\
\vdots \\
[a_m G_1]_1
\end{array} \right]
=
\left[ \begin{array}{ccc}
l_{1,1}[a_1 G_1]_1 & + \cdots + & l_{1,m}[a_m G_1]_1 \\
\vdots & \ddots & \vdots \\
l_{n,1}[a_1 G_1]_1 & + \cdots + & l_{n,m}[a_m G_1]_1
\end{array} \right]
=
\left[ \begin{array}{c}
\sum_{i=1}^m l_{1,i}[a_i G_1]_1 \\
\sum_{i=1}^m l_{2,i}[a_i G_1]_1 \\
\vdots \\
\sum_{i=1}^m l_{n,i}[a_i G_1]_1
\end{array} \right]
$$

考虑到逐元素乘积将变成一组椭圆曲线配对，也可以使用 $G_2$ 点加密 $\mathbf{a}$ 向量，使验证者能够执行配对：

$$
\left[ \begin{array}{ccc}
r_{1,1} & \cdots & r_{1,m} \\
\vdots & \ddots & \vdots \\
r_{n,1} & \cdots & r_{n,m}
\end{array} \right]
\left[ \begin{array}{c}
[a_1 G_2]_2 \\
\vdots \\
[a_m G_2]_2
\end{array} \right]
=
\left[ \begin{array}{ccc}
r_{1,1}[a_1 G_2]_2 & + \cdots + & r_{1,m}[a_m G_2]_2 \\
\vdots & \ddots & \vdots \\
r_{n,1}[a_1 G_2]_2 & + \cdots + & r_{n,m}[a_m G_2]_2
\end{array} \right]
=
\left[ \begin{array}{c}
\sum_{i=1}^m r_{1,i}[a_i G_2]_2 \\
\sum_{i=1}^m r_{2,i}[a_i G_2]_2 \\
\vdots \\
\sum_{i=1}^m r_{n,i}[a_i G_2]_2
\end{array} \right]
$$

完成该运算后，会得到由乘法 $\mathbf{L}\mathbf{a}$ 产生的单列 $G_1$ 椭圆曲线点，以及由 $\mathbf{R}\mathbf{a}$ 产生的单列 $G_2$ 点。

朴素的下一步是用 $G_{12}$ 点加密 $\mathbf{a}$ 向量，使验证者能够对 $\mathbf{L}\mathbf{a}$ 和 $\mathbf{R}\mathbf{a}$ 的结果执行配对，再检查它是否等于 $\mathbf{O}\mathbf{a}$。但 $G_{12}$ 点非常庞大，因此更适合让验证者先处理 $\mathbf{O}\mathbf{a}$ 在 $G_1$ 中的椭圆曲线点，再把每一项与一个 $G_2$ 点配对。从某种意义上说，与 $G_2$ 点配对相当于“乘以一”，但会把 $G_1$ 点转换成 $G_{12}$ 点。

随后，证明者把 $G_1$ 向量和 $G_2$ 向量交给验证者。

### 验证步骤
于是，验证步骤变为：

$$

\left[ \begin{array}{c}
\sum_{i=1}^m l_{i,1}[a_i G_1]_1 \\
\sum_{i=1}^m l_{i,1}[a_i G_1]_1 \\
\vdots \\
\sum_{i=1}^m l_{i,1}[a_i G_1]_1
\end{array} \right]
\begin{matrix}
\bullet \\
\bullet \\
\vdots \\
\bullet
\end{matrix}
\left[ \begin{array}{c}
\sum_{i=1}^m r_{i,1}[a_i G_2]_2 \\
\sum_{i=1}^m r_{i,1}[a_i G_2]_2 \\
\vdots \\
\sum_{i=1}^m r_{i,1}[a_i G_2]_2
\end{array} \right]

\stackrel{?}{=}
\left[ \begin{array}{c}
\sum_{i=1}^m o_{i,1}[a_i G_1]_1 \\
\sum_{i=1}^m o_{i,1}[a_i G_1]_1 \\
\vdots \\
\sum_{i=1}^m o_{i,1}[a_i G_1]_1
\end{array} \right]
\begin{matrix}
\bullet \\
\bullet \\
\vdots \\
\bullet
\end{matrix}
\left[ \begin{array}{c}
G_2 \\
G_2 \\
\vdots \\
G_2
\end{array} \right]
$$

$$
=
\begin{array}{c}
 \sum_{i=1}^m l_{i,1}[a_i G_1]_1\bullet \sum_{i=1}^m r_{i,1}[a_i G_2]_2 \\
 \sum_{i=1}^m l_{i,2}[a_i G_1]_1\bullet \sum_{i=1}^m r_{i,2}[a_i G_2]_2  \\
\vdots \\
\sum_{i=1}^m l_{i,n}[a_i G_1]_1\bullet \sum_{i=1}^m r_{i,n}[a_i G_2]_2
\end{array}
\stackrel{?}{=}
\begin{array}{c}
\sum_{i=1}^m o_{i,1}[a_i G_1]_1\bullet G_2 \\
\sum_{i=1}^m o_{i,2}[a_i G_1]_1\bullet G_2 \\
\vdots \\
\sum_{i=1}^m o_{i,n}[a_i G_1]_1 \bullet G_2
\end{array}
$$

当且仅当证明者提供了有效见证时，上述两个 $G_{12}$ 元素向量才会逐元素相等。

更准确地说，几乎如此。后文会讨论剩下的问题。

首先需要说明一个重要的实现细节。


### 公开输入
假设知识声明是：“我知道 $x$，满足 $x³ + 5x + 5 = y$，其中 $y = 155$。”见证向量可能如下：

$$
[1, y, x, v]
$$

其中 $v = x^2$。在这种情况下，$[1, y]$ 需要公开。只需不加密见证的前两个元素即可。验证者会检查公开输出，然后把公开输入分别乘以 $G_1$ 或 $G_2$ 点进行加密，从而保持验证公式不变。

### 应对恶意证明者
由于向量已经加密，验证者无法立即知道 $\mathbb{G}₁$ 点向量与 $\mathbb{G}₂$ 点向量是否加密了相同的值。

也就是说，证明者提供的是 $\mathbf{a}G_1$ 和 $\mathbf{a}G_2$。验证者不知道这些点向量的离散对数，那么如何确认 $\mathbb{G}₁$ 点向量与 $\mathbb{G}₂$ 点向量具有相同的离散对数？

验证者可以分别把两个点向量与由*另一群*生成元组成的向量配对，再检查所得 $\mathbb{G}_{12}$ 点是否相等，从而在不知道离散对数的情况下检查它们是否相等。具体来说：

$$
\begin{bmatrix}
a_1G_1 \\
a_2G_1 \\
\vdots \\
a_mG_1
\end{bmatrix}
\begin{matrix}
\bullet \\
\bullet \\
\vdots \\
\bullet
\end{matrix}
\begin{bmatrix}
G_2 \\
G_2 \\
\vdots \\
G_2
\end{bmatrix}
\stackrel{?}{=}
\begin{bmatrix}
a_1G_2 \\
a_2G_2 \\
\vdots \\
a_mG_2
\end{bmatrix}
\begin{matrix}
\bullet \\
\bullet \\
\vdots \\
\bullet
\end{matrix}
\begin{bmatrix}
G_1 \\
G_1 \\
\vdots \\
G_1
\end{bmatrix}
$$


## 该算法主要具有学术意义
该算法对验证者而言效率很低。如果 R1CS 中的矩阵很大——对于有意义的算法，矩阵确实会很大——验证者就必须执行大量配对和椭圆曲线点加法。椭圆曲线点加法相当快，但椭圆曲线配对很慢，而且在以太坊上会消耗大量 gas。

不过，看到零知识证明在这个阶段已经成为可能，仍然很有意义。如果很好地掌握了椭圆曲线运算，并且还没有忘记矩阵运算，理解它并不困难。

### 让该技术真正具备零知识性
在当前方案中，见证向量无法被解密，却可以被猜出。如果攻击者（试图找出未加密见证的人）利用辅助信息对见证作出有根据的猜测，就可以把猜测的见证向量乘以椭圆曲线生成元点，再检查结果是否与证明者的见证向量相同，从而验证猜测。

介绍 [Groth16](https://www.rareskills.io/post/groth16) 时，将学习如何抵御对见证的猜测。

还要记住，现实中没有人使用上述算法，因为它效率太低。不过，实现该算法可以帮助你练习有实际意义的椭圆曲线运算，并构建一个端到端、可以运行的（近似）零知识证明。

Obront 在[这个仓库](https://github.com/zobront/homerolled-zk)中提供了本文算法的示例实现。

## 通过 RareSkills 继续学习
本文内容来自我们的[零知识课程](https://www.rareskills.io/zk-bootcamp)。

*最初发表于 2023 年 8 月 26 日*
