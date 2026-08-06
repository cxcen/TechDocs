# 将代数电路转换为 R1CS（秩一约束系统）

本文介绍如何把一组算术约束转换为秩一约束系统（Rank-1 Constraint System，R1CS）。

本文侧重实现：相较其他资料，我们会介绍更多转换时的边界情况，讨论优化方法，并解释 Circom 库如何完成这一转换。

## 前置知识
- 假定读者了解如何使用[算术电路（ZK 电路）](https://www.rareskills.io/post/arithmetic-circuit)表示某项计算是否有效。
- 读者应熟悉[模运算](https://www.rareskills.io/post/finite-fields)。本文所有运算都发生在有限域中，因此 $-5$ 实际表示 $5$ 模 $p$ 的加法逆元，而 $2/3$ 表示 $3$ 模 $p$ 的乘法逆元再乘以 $2$。

## 秩一约束系统概述
秩一约束系统（R1CS）是一种算术电路，它要求每个等式约束恰好包含一次乘法，而加法次数不受限制。

这样可以让算术电路的表示与双线性配对兼容。配对的输出 $G_1 \bullet G_2 \rightarrow G_T$ 无法再次参与配对，因为 $G_T$ 中的元素不能作为另一次配对的输入。因此，每个约束只允许一次乘法。

## 见证向量
在算术电路中，见证是对所有信号的一组赋值，并且这组赋值满足等式约束。

在秩一约束系统中，见证向量（witness vector）是一个 $1 \times n$ 向量，其中包含所有输入变量、输出变量和中间值。它表明你知道输入、输出以及全部中间值，并已从头到尾执行了该电路。

按照惯例，第一个元素始终为 1，以便简化一些计算，稍后会演示这一点。

例如，假设有约束：

$$
z = x^2y
$$

并声称知道它的解，这意味着我们必须知道 $x$、$y$ 和 $z$。因为秩一约束系统要求每个约束恰好包含一次乘法，所以上面的多项式约束必须写成：

$$
\begin{align*}
v₁ = xx \\
z = v₁y
\end{align*}
$$

拥有见证并不只是知道 $x$、$y$ 和 $z$，还必须知道展开形式中的每个中间变量。具体而言，见证是向量：

$$
[1, z, x, y, v₁]
$$

其中每一项都具有一个满足上述约束的值。

例如：

$$
[1, 18, 3, 2, 9]
$$

是一个有效见证，因为把这些值代入：

$$
[\text{常量} = 1, z = 18, x = 3, y = 2, v₁ = 9]
$$
便可满足约束：

$$
\begin{align*}
v_1 = x*x \rightarrow 9 = 3\cdot3\\
z = v_1*y \rightarrow 18 = 9\cdot2
\end{align*}
$$

本例没有使用额外的 1，稍后会解释它所提供的便利。

## 示例 1：把 $z = x \cdot y$ 转换为秩一约束系统

在本例中，我们要证明 $41 \times 103 = 4223$。

因此，见证向量是 $[1, 4223, 41, 103]$，对应对 $[1, z, x, y]$ 的赋值。

创建 R1CS 之前，约束需要采用以下形式：

```
result = left_hand_side × right_hand_side
```

幸运的是，当前约束已经符合该形式：
$$
\underbrace{z}_\text{结果} = \underbrace{x}_\text{左侧} \times \underbrace{y}_\text{右侧}
$$

这显然是一个非常简单的例子，但从简单示例入手通常最合适。

要创建有效的 R1CS，需要一组恰好包含一次乘法的公式。

稍后会讨论如何处理 $z = x³$ 或 $z = x³ + y$ 这类并非恰好包含一次乘法的情况。

我们的目标是创建如下形式的方程组：

$$
\mathbf{O}\mathbf{a} = \mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a}
$$

其中 $O$、$L$、$R$ 都是大小为 $n$ x $m$ 的矩阵，即 $n$ 行、$m$ 列。

矩阵 $\mathbf{L}$ 编码乘法左侧的变量，$\mathbf{R}$ 编码乘法右侧的变量，$\mathbf{O}$ 编码结果变量。向量 $\mathbf{a}$ 是见证向量。

具体来说，$\mathbf{L}$、$\mathbf{R}$、$\mathbf{O}$ 的列数与见证向量 $\mathbf{a}$ 的元素数相同，每一列都表示对应索引位置上的同一个变量。

在本例中，见证包含 4 个元素 $(1, z, x, y)$，因此每个矩阵都有 4 列，即 $m = 4$。

行数对应电路中的约束数。本例只有一个约束 $z = x * y$，因此只有一行，即 $n = 1$。

先直接给出答案，再解释它是如何得到的。

$$
\mathbf{O}\mathbf{a} = \mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a}
$$
$$
\underbrace{\begin{bmatrix}
0 & 1 & 0 & 0 \\
\end{bmatrix}}_{\mathbf{O}}\mathbf{a} =
\underbrace{\begin{bmatrix}
0 & 0 & 1 & 0 \\
\end{bmatrix}}_{\mathbf{L}}\mathbf{a}\circ
\underbrace{\begin{bmatrix}
0 & 0 & 0 & 1 \\
\end{bmatrix}}_{\mathbf{R}}\mathbf{a}
$$

$$
\begin{bmatrix}
0 & 1 & 0 & 0 \\
\end{bmatrix}
\begin{bmatrix}
1 \\
4223 \\
41 \\
103 \\
\end{bmatrix}=
\begin{bmatrix}
0 & 0 & 1 & 0 \\
\end{bmatrix}
\begin{bmatrix}
1 \\
4223 \\
41 \\
103 \\
\end{bmatrix}\circ
\begin{bmatrix}
0 & 0 & 0 & 1 \\
\end{bmatrix}
\begin{bmatrix}
1 \\
4223 \\
41 \\
103 \\
\end{bmatrix}
$$

在这个例子中，矩阵中的每一项都充当指示值，用来表示该列所对应的变量是否出现。（严格来说，它是变量的系数，稍后会介绍这一点。）

对左侧项而言，$x$ 是乘法左侧唯一出现的变量。因此，如果各列表示 $[1, z, x, y]$，那么：

$\mathbf{L}$ 是 $[0, 0, 1, 0]$，因为出现了 $x$，而其他变量都没有出现。

$\mathbf{R}$ 是 $[0, 0, 0, 1]$，因为乘法右侧唯一的变量是 $y$。

$\mathbf{O}$ 是 $[0, 1, 0, 0]$，因为乘法的“输出”中只有变量 $z$。

这里没有任何常数，因此表示 1 的列处处为零（稍后会讨论它何时非零）。

该等式是正确的，可以用 Python 验证：

```python
import numpy as np

# define the matrices
O = np.matrix([[0,1,0,0]])
L = np.matrix([[0,0,1,0]])
R = np.matrix([[0,0,0,1]])

# witness vector
a = np.array([1, 4223, 41, 103])

# Multiplication `*` is element-wise, not matrix multiplication.
# Result contains a bool indicating an element-wise indicator that the equality is true for that element.
result = np.matmul(O, a) == np.matmul(L, a) * np.matmul(R, a)

# check that every element-wise equality is true
assert result.all(), "result contains an inequality"
```

你可能会疑惑这样做有什么意义：这不就是用更不紧凑的方式表达 $41 \times 103 = 4223$ 吗？

确实如此。

R1CS 可能非常冗长，但它能自然映射为[二次算术程序（QAP）](https://www.rareskills.io/post/quadratic-arithmetic-program)，而 QAP 可以变得十分简洁。不过，本文暂不讨论 QAP。

但这正是 R1CS 的一个重点：R1CS 传达的信息与原始算术约束完全相同，只是每个等式约束仅包含一次乘法。本例只有一个约束，下一例将加入更多约束。

## 示例 2：转换 r = x * y * z * u
这个稍复杂的例子需要处理中间变量。每行计算只能包含一次乘法，因此必须把等式拆分如下：

$$
\begin{align*}
v_1 &= xy \\
v_2 &= zu \\
r &= v_1v_2
\end{align*}
$$

没有规则要求必须这样拆分，下面的写法同样有效：

$$
\begin{align*}
v_1 &= xy \\
v_2 &= v_1z \\
r &= v_2u
\end{align*}
$$

本例采用第一种转换。

### $\mathbf{L}$、$\mathbf{R}$ 和 $\mathbf{O}$ 的大小
这里涉及 7 个变量 $(r, x, y, z, u, v_1, v_2)$，因此见证向量包含八个元素（第一个是常量 1），矩阵有八列。

因为一共有三个约束，所以矩阵有三行。

### 左侧项与右侧项
本例将进一步强化“左侧项”和“右侧项”的概念。具体而言，$x$、$z$、$v_1$ 是左侧项，$y$、$u$、$v_2$ 是右侧项。

$$
\underbrace{
    \begin{matrix}
        v_1 \\
        v_2 \\
        r \\
    \end{matrix}
}_\text{输出项}
\begin{matrix}
=\\
=\\
=
\end{matrix}
\underbrace{
    \begin{matrix}
        x \\
        z \\
        v_1 \\
    \end{matrix}
}_\text{左侧项}
\begin{matrix}
\times\\
\times\\
\times
\end{matrix}
\underbrace{
    \begin{matrix}
        y \\
        u \\
        v_2 \\
    \end{matrix}
}_\text{右侧项}
$$

### 根据左侧项构造矩阵 $\mathbf{L}$
下面构造矩阵 A。已知它有三行（因为有三个约束）和八列（因为有八个变量）。

$$
\begin{bmatrix}
& l_{1,2} & l_{1,3} & l_{1,4} & l_{1,5} & l_{1,6} &  l_{1,7} & l_{1,8} \\
& l_{2,2} & l_{2,3} & l_{2,4} & l_{2,5} & l_{2,6} &  l_{2,7} & l_{2,8} \\
& l_{3,2} & l_{3,3} & l_{3,4} & l_{3,5} & l_{3,6} &  l_{3,7} & l_{3,8} \\
\end{bmatrix}
$$

见证向量将与它相乘，因此把见证向量的布局定义为：

$$
\begin{bmatrix}
1 & r &x & y & z & u & v_1 & v_2
\end{bmatrix}
$$

由此可以知道 $\mathbf{L}$ 的各列表示什么：

$$
\mathbf{L} =
\begin{bmatrix}
l_{1, 1} & l_{1, r} & l_{1, x} & l_{1, y} & l_{1, z} & l_{1, u} & l_{1, v_1} & l_{1, v_2} \\
l_{2, 1} & l_{2, r} & l_{2, x} & l_{2, y} & l_{2, z} & l_{2, u} & l_{2, v_1} & l_{2, v_2} \\
l_{3, 1} & l_{3, r} & l_{3, x} & l_{3, y} & l_{3, z} & l_{3, u} & l_{3, v_1} & l_{3, v_2} \\
\end{bmatrix}
$$


#### $\mathbf{L}$ 的第一行
第一行对应第一个左侧变量，其中 $v₁ = xy$：

$$
\begin{matrix}
    v_1 \\
    v_2 \\
    r \\
\end{matrix}
\begin{matrix}
=\\
=\\
=
\end{matrix}
\underset{\mathbf{L}}{\boxed{
    \begin{matrix}
        \color{red}{x} \\
        z \\
        v_1 \\
    \end{matrix}
}}
\begin{matrix}
\times\\
\times\\
\times
\end{matrix}
\begin{matrix}
    y \\
    u \\
    v_2 \\
\end{matrix}
$$

这意味着，就左侧而言，出现了变量 $x$，其他变量都没有出现。因此，把第一行转换为：

$$
\mathbf{L}=\begin{bmatrix}
0 & 0 & \color{red}{1} & 0 & 0 & 0 & 0 & 0 \\
l_{2, 1} & l_{2, r} & l_{2, x} & l_{2, y} & l_{2, z} & l_{2, u} & l_{2, v_1} & l_{2, v_2} \\
l_{3, 1} & l_{3, r} & l_{3, x} & l_{3, y} & l_{3, z} & l_{3, u} & l_{3, v_1} & l_{3, v_2} \\
\end{bmatrix}
$$

回顾一下，$\mathbf{L}$ 的各列标签如下：

$$
\begin{bmatrix}
1 & r &x & y & z & u & v_1 & v_2\\
\end{bmatrix}
$$

可以看到，$1$ 位于 $x$ 列。

#### $\mathbf{L}$ 的第二行
继续向下，可以看到方程组第二行的左侧只有 $z$。

$$
\begin{matrix}
    v_1 \\
    v_2 \\
    r \\
\end{matrix}
\begin{matrix}
=\\
=\\
=
\end{matrix}
\underset{\mathbf{L}}{\boxed{
    \begin{matrix}
        x \\
        \color{green}{z} \\
        v_1 \\
    \end{matrix}
}}
\begin{matrix}
\times\\
\times\\
\times
\end{matrix}
    \begin{matrix}
        y \\
        u \\
        v_2 \\
    \end{matrix}
$$

因此，该行除表示 $z$ 的列之外全部设为零：

$$
\mathbf{L}=\begin{bmatrix}
0 & 0 & \color{red}{1} & 0 & 0 & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & \color{green}{1} & 0 & 0 & 0 \\
l_{3, 1} & l_{3, r} & l_{3, x} & l_{3, y} & l_{3, z} & l_{3, u} & l_{3, v_1} & l_{3, v_2} \\
\end{bmatrix}
$$

#### $\mathbf{L}$ 的第三行
最后，第三行左侧项中唯一出现的变量是 $v₁$：

$$
\begin{matrix}
    v_1 \\
    v_2 \\
    r \\
\end{matrix}
\begin{matrix}
=\\
=\\
=
\end{matrix}
\underset{\mathbf{L}}{\boxed{
    \begin{matrix}
        x \\
        z \\
        \color{violet}{v_1} \\
    \end{matrix}
}}
\begin{matrix}
\times\\
\times\\
\times
\end{matrix}
    \begin{matrix}
        y \\
        u \\
        v_2 \\
    \end{matrix}
$$

这样就完成了矩阵 $\mathbf{L}$：

$$
\mathbf{L}=\begin{bmatrix}
0 & 0 & \color{red}{1} & 0 & 0 & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & \color{green}{1} & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & \color{violet}{1} & 0 \\
\end{bmatrix}
$$

下图可以更清楚地展示这种映射：

$$
\begin{matrix}
    v_1 \\
    v_2 \\
    r \\
\end{matrix}
\begin{matrix}
=\\
=\\
=
\end{matrix}
\underset{\mathbf{L}}{\boxed{
    \begin{matrix}
        \color{red}{x} \\
        \color{green}{z} \\
        \color{violet}{v_1} \\
    \end{matrix}
}}
\begin{matrix}
\times\\
\times\\
\times
\end{matrix}
    \begin{matrix}
        y \\
        u \\
        v_2 \\
    \end{matrix}
\space\space\space\space
\begin{array}{c}
\begin{array}{cc}
\begin{matrix}
1 & r & x & y & z & u & v_1 & v_2 \\
\end{matrix}
\end{array} \\[10pt]
\begin{array}{cc}
\begin{bmatrix}
0 & 0 & \color{red}{1} & 0 & 0 & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & \color{green}{1} & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & \color{violet}{1} & 0 \\
\end{bmatrix}
\end{array}
\end{array}
$$

#### $\mathbf{L}$ 的另一种转换方法
也可以通过展开下式的左侧值来完成同样的操作：

$$
\begin{matrix}
    v_1 \\
    v_2 \\
    r \\
\end{matrix}
\begin{matrix}
=\\
=\\
=
\end{matrix}
{
    \begin{matrix}
        x \\
        z \\
        v_1 \\
    \end{matrix}
}
\begin{matrix}
\times\\
\times\\
\times
\end{matrix}
    \begin{matrix}
        y \\
        u \\
        v_2 \\
    \end{matrix}
$$

展开为：

$$
\begin{align*}
v_1 &= (0\cdot 1 + 0\cdot r + \boxed{1\cdot x} + 0\cdot y + 0\cdot z + 0\cdot u + 0\cdot v_1 + 0\cdot v_2) \times y\\
v_2 &= (0\cdot 1 + 0\cdot r + 0\cdot x + 0\cdot y + \boxed{1\cdot z} + 0\cdot u + 0\cdot v_1 + 0\cdot v_2) \times u\\
r &= (0\cdot 1 + 0\cdot r + 0\cdot x + 0\cdot y + 0\cdot z + 0\cdot u + \boxed{1\cdot v_1} + 0\cdot v_2) \times v_2 \\
\end{align*}
$$

这是可行的，因为加上值为零的项不会改变结果。只需注意，要按照见证向量的定义，以相同的“列”展开值为零的变量。

然后，从上述展开式中取出系数（方框中的值）：

$$
\begin{align*}
v_1 &= (\boxed{0}\cdot 1 + \boxed{0}\cdot r + \boxed{1}\cdot x + \boxed{0}\cdot y + \boxed{0}\cdot z + \boxed{0}\cdot u + \boxed{0}\cdot v_1 + \boxed{0}\cdot v_2) \times y\\
v_2 &= (\boxed{0}\cdot 1 + \boxed{0}\cdot r + \boxed{0}\cdot x + \boxed{0}\cdot y + \boxed{1}\cdot z + \boxed{0}\cdot u + \boxed{0}\cdot v_1 + \boxed{0}\cdot v_2) \times u\\
r &= (\boxed{0}\cdot 1 + \boxed{0}\cdot r + \boxed{0}\cdot x + \boxed{0}\cdot y + \boxed{0}\cdot z + \boxed{0}\cdot u + \boxed{1}\cdot v_1 + \boxed{0}\cdot v_2) \times v_2 \\
\end{align*}
$$

这样得到的 $\mathbf{L}$，与刚才生成的矩阵相同：

$$
\mathbf{L}=\begin{bmatrix}
0 & 0 & 1 & 0 & 0 & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & 1 & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & 1 & 0 \\
\end{bmatrix}
$$


### 根据右侧项构造矩阵 $\mathbf{R}$
$$
R = \begin{bmatrix}
r_{1,1} & r_{1,r} & r_{1,x} & r_{1,y} & r_{1,z} & r_{1,u} & r_{1,v_1} & r_{1,v_2} \\
r_{2,1} & r_{2,r} & r_{2,x} & r_{2,y} & r_{2,z} & r_{2,u} & r_{2,v_1} & r_{2,v_2} \\
r_{3,1} & r_{3,r} & r_{3,x} & r_{3,y} & r_{3,z} & r_{3,u} & r_{3,v_1} & r_{3,v_2} \\
\end{bmatrix}
$$

矩阵 $\mathbf{R}$ 表示等式中的右侧项：

$$
\begin{matrix}
    v_1 \\
    v_2 \\
    r \\
\end{matrix}
\begin{matrix}
=\\
=\\
=
\end{matrix}
{
    \begin{matrix}
        x \\
        z \\
        v_1 \\
    \end{matrix}
}
\begin{matrix}
\times\\
\times\\
\times
\end{matrix}
    \underset{\mathbf{R}}{\boxed{
    \begin{matrix}
        y \\
        u \\
        v_2 \\
    \end{matrix}}}
$$

矩阵 $\mathbf{R}$ 中必须有若干个 1，分别表示 $y$、$u$ 和 $v_2$。矩阵的行与算术约束的行对应，也就是说，可以给约束（各行）编号如下：

$$
\begin{matrix}
    (1) \\
    (2) \\
    (3) \\
\end{matrix}
\space\space
\begin{matrix}
    v_1 \\
    v_2 \\
    r \\
\end{matrix}
\begin{matrix}
=\\
=\\
=
\end{matrix}
{
    \begin{matrix}
        x \\
        z \\
        v_1 \\
    \end{matrix}
}
\begin{matrix}
\times\\
\times\\
\times
\end{matrix}
\underset{\mathbf{R}}{\boxed{
\begin{matrix}
    y \\
    u \\
    v_2 \\
\end{matrix}}}
$$

因此，第一行在 $y$ 列为 1，第二行在 $u$ 列为 1，第三行在 $v_2$ 列为 1，其余位置均为零。

由此得到矩阵 $\mathbf{R}$：

$$
\begin{array}{c}
\begin{array}{cc}
\begin{matrix}
1 & r & x & y & z & u & v_1 & v_2 \\
\end{matrix}
\end{array} \\[10pt]
\begin{array}{cc}
\begin{bmatrix}
0 & 0 & 0 & 1 & 0 & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 1 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & 0 & 1 \\
\end{bmatrix}
\end{array}
\end{array}
$$

下图展示了这一转换：

$$
\begin{matrix}
    v_1 \\
    v_2 \\
    r \\
\end{matrix}
\begin{matrix}
=\\
=\\
=
\end{matrix}

    \begin{matrix}
        x \\
        z \\
        v_1 \\
    \end{matrix}

\begin{matrix}
\times\\
\times\\
\times
\end{matrix}
\underset{\mathbf{R}}
{\boxed{
    \begin{matrix}
        \color{red}{y} \\
        \color{green}{u} \\
        \color{violet}{v_2} \\
    \end{matrix}
}}
\space\space\space\space
\begin{array}{c}
\begin{array}{cc}
\begin{matrix}
1 & r & x & y & z & u & v_1 & v_2 \\
\end{matrix}
\end{array} \\[10pt]
\begin{array}{cc}
\begin{bmatrix}
0 & 0 & 0 & \color{red}{1} & 0 & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & \color{green}{1} & 0 & 0  \\
0 & 0 & 0 & 0 & 0 & 0 & 0 &\color{violet}{1}  \\
\end{bmatrix}
\end{array}
\end{array}
$$

### 构造矩阵 $\mathbf{O}$
请读者自行推导矩阵 $\mathbf{O}$：

$$
\mathbf{O}=\begin{bmatrix}
0 & 0 & 0 & 0 & 0 & 0 & 1 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & 0 & 1  \\
0 & 1 & 0 & 0 & 0 & 0 & 0 & 0  \\
\end{bmatrix}
$$

其列标签与前面的矩阵一致。

再提醒一次，$\mathbf{C}$ 根据乘法结果得到：

$$
\underset{\mathbf{C}}{
    \boxed{\begin{matrix}
        v_1 \\
        v_2 \\
        r \\
    \end{matrix}}}

\begin{matrix}
=\\
=\\
=
\end{matrix}

    \begin{matrix}
        x \\
        z \\
        v_1 \\
    \end{matrix}

\begin{matrix}
\times\\
\times\\
\times
\end{matrix}

    \begin{matrix}
        y \\
        u \\
        v_2 \\
    \end{matrix}
$$

列标签如下：

$$
\begin{array}{c}
\begin{array}{cc}
\begin{matrix}
1 & r & x & y & z & u & v_1 & v_2 \\
\end{matrix}
\end{array} \\[10pt]
\begin{array}{cc}
\begin{bmatrix}
0 & 0 & 0 & 0 & 0 & 0 & 1 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & 0 & 1  \\
0 & 1 & 0 & 0 & 0 & 0 & 0 & 0  \\
\end{bmatrix}
\end{array}
\end{array}
$$

下面检查 $r = x \cdot y \cdot z \cdot u$ 的转换结果：

```python
import numpy as np

# enter the A B and C from above
L = np.matrix([[0,0,1,0,0,0,0,0],
              [0,0,0,0,1,0,0,0],
              [0,0,0,0,0,0,1,0]])

R = np.matrix([[0,0,0,1,0,0,0,0],
              [0,0,0,0,0,1,0,0],
              [0,0,0,0,0,0,0,1]])

O = np.matrix([[0,0,0,0,0,0,1,0],
              [0,0,0,0,0,0,0,1],
              [0,1,0,0,0,0,0,0]])

# random values for x, y, z, and u
import random
x = random.randint(1,1000)
y = random.randint(1,1000)
z = random.randint(1,1000)
u = random.randint(1,1000)

# compute the algebraic circuit
r = x * y * z * u
v1 = x*y
v2 = z*u

# create the witness vector
a = np.array([1, r, x, y, z, u, v1, v2])

# element-wise multiplication, not matrix multiplication
result = np.matmul(O, a) == np.multiply(np.matmul(L, a), np.matmul(R, a))

assert result.all(), "system contains an inequality"
```

## 示例 3：与常数相加
如果要为下式构造秩一约束系统，该怎么做？

$$
z = x * y + 2
$$

这正是值为 1 的列发挥作用的地方。

### 加法是免费的
你可能听过 ZK-SNARK 语境中的说法：“加法是免费的。”它的含义是，遇到加法运算时，不必额外创建一个约束。

可以把上述公式写成：

$$
\begin{align*}
v_1 = xy \\
z = v_1 + 2 \\
\end{align*}
$$

但这会让 R1CS 大于实际所需的大小。

可以改写为：

$$
-2 + z = xy
$$

这样，当 $\mathbf{a}$ 与 $\mathbf{O}$ 相乘时，变量 $z$ 和常数 $-2$ 就会在见证向量中自动“组合”。

见证向量的形式为 `[1, z, x, y]`，因此矩阵 $\mathbf{L}$、$\mathbf{R}$ 和 $\mathbf{O}$ 如下：

$$
\mathbf{L} = \begin{bmatrix}
0 & 0 & 1 & 0 \\
\end{bmatrix}
$$

$$
\mathbf{R} = \begin{bmatrix}
0 & 0 & 0 & 1 \\
\end{bmatrix}
$$

$$
\mathbf{O} = \begin{bmatrix}
-2 & 1 & 0 & 0 \\
\end{bmatrix}
$$

每当出现加法常数时，只需把它放入表示 $1$ 的列；按照惯例，这就是第一列。

同样，下面用单元测试检查计算：

```python
import numpy as np
import random

# Define the matrices
L = np.matrix([[0,0,1,0]])
R = np.matrix([[0,0,0,1]])
O = np.matrix([[-2,1,0,0]])

# pick random values to test the equation
x = random.randint(1,1000)
y = random.randint(1,1000)
z = x * y + 2 # witness vector
a = np.array([1, z, x, y])

# check the equality
result = O.dot(a) == np.multiply(np.matmul(L, a), R.dot(a))
assert result.all(), "result contains an inequality"
```

## 示例 4：与常数相乘
前面的例子从未把变量乘以常数，因此 R1CS 中的项始终为 1。根据上一例或许已经可以猜到：矩阵中的项，就是变量所乘常数的值，下面的例子将说明这一点。

求解：

$$
z = 2x^2 + y
$$

请注意，“每个约束一次乘法”指的是两个变量相乘。与常数相乘不算“真正的”乘法，因为它其实是同一个变量的重复加法。

下面的方案有效，但会创建不必要的行：

$$
\begin{align*}
v_1 &= xx \\\\
z &= 2v_1 + y\\\\
\end{align*}
$$

更优的方案如下：

$$
-y + z = 2xx
$$

采用更优方案后，见证向量的形式为 `[1, out, x, y]`。

矩阵定义如下：

$$
\begin{align*}
\mathbf{L} &= \begin{bmatrix}
0 & 0 & 2 & 0 \\
\end{bmatrix} \\
\mathbf{R} &= \begin{bmatrix}
0 & 0 & 1 & 0 \\
\end{bmatrix} \\
\mathbf{O} &= \begin{bmatrix}
0 & 1 & 0 & -1 \\
\end{bmatrix} \\
\end{align*}
$$

把上述矩阵与 `[1, z, x, y]` 按 R1CS 形式进行符号乘法，会重新得到原始等式：

$$
\begin{bmatrix}
0 & 1 & 0 & -1 \\
\end{bmatrix}
\begin{bmatrix}
1 \\
z \\
x \\
y \\
\end{bmatrix}
=
\begin{bmatrix}
0 & 0 & 2 & 0 \\
\end{bmatrix}
\begin{bmatrix}
1 \\
z \\
x \\
y \\
\end{bmatrix}\circ
\begin{bmatrix}
0 & 0 & 1 & 0 \\
\end{bmatrix}
\begin{bmatrix}
1 \\
z \\
x \\
y \\
\end{bmatrix}
$$
$$
z - y = 2xx
$$
$$
z = 2x^2 + y
$$

因此可以确认 $\mathbf{L}$、$\mathbf{R}$ 和 $\mathbf{O}$ 设置正确。

这里只有一行（一个约束）和一次“真正的”乘法。一般而言：

秩一约束系统中的约束数，应等于非常数乘法的次数。

## 示例 5：大型约束
下面处理一个不那么简单的例子，综合前面学到的全部内容。

假设有约束：

$$
z = 3x^2y + 5xy - x - 2y + 3
$$

将其拆分为：

$$
\begin{align*}
v_1 &= 3xx \\
v_2 &= v_1y \\
-v_2 + x + 2y - 3 + z  &= 5xy \\
\end{align*}
$$

注意，所有加法项都被移到了左侧（加法示例中也是这样处理的，但在这里更加明显）。

第三行右侧保留 $5xy$ 是任意选择。也可以把等式两边同时除以 5，使最后一个约束变成：

$$
\frac{-v_2}{5} + \frac{x}{5} + \frac{2y}{5} - \frac{3}{5} + \frac{z}{5}= xy
$$

这不会改变见证，因此两种写法都有效。由于所有计算都发生在有限域中，该操作相当于让等式左右两侧都乘以 5 的乘法逆元。

见证向量的形式为：

$$
[1, z, x, y, v_1, v_2]
$$

由于有三个约束，矩阵也有三行：

$$
\begin{align*}
\color{red}{v_1} &= \color{green}{3x}\color{violet}{x} \\
\color{red}{v_2} &= \color{green}{v_1}\color{violet}{y} \\
\color{red}{-v_2 + x + 2y - 3 + z}  &= \color{green}{5x}\color{violet}{y} \\
\end{align*}
$$

我们用<span style="color:red">红色</span>标记输出 $\mathbf{O}$，用<span style="color:green">绿色</span>标记左侧 $\mathbf{L}$，用<span style="color:violet">紫色</span>标记右侧 $\mathbf{R}$。由此得到以下矩阵：

$$
L = \begin{bmatrix}
0 & 0 & \textcolor{green}{3} & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & \textcolor{green}{1} & 0 \\
0 & 0 & \textcolor{green}{5} & 0 & 0 & 0 \\
\end{bmatrix}
$$

$$
R = \begin{bmatrix}
0 & 0 & \textcolor{violet}{1} & 0 & 0 & 0 \\
0 & 0 & 0 & \textcolor{violet}{1} & 0 & 0 \\
0 & 0 & 0 & \textcolor{violet}{1} & 0 & 0 \\
\end{bmatrix}
$$

$$
O = \begin{bmatrix}

0 & 0 & 0 & 0 & \textcolor{red}{1} & 0 \\
0 & 0 & 0 & 0 & 0 & \textcolor{red}{1} \\
\textcolor{red}{-3} & \textcolor{red}{1} & \textcolor{red}{1} & \textcolor{red}{2} & 0 & \textcolor{red}{-1} \\
\end{bmatrix}
$$

列标签为：

$$
\begin{bmatrix}1 & \text{out} & x & y & v_1 & v_2\end{bmatrix}
$$

照例检查转换结果：

```python
import numpy as np
import random

# Define the matrices
L = np.array([[0,0,3,0,0,0],
               [0,0,0,0,1,0],
               [0,0,5,0,0,0]])

R = np.array([[0,0,1,0,0,0],
               [0,0,0,1,0,0],
               [0,0,0,1,0,0]])

O = np.array([[0,0,0,0,1,0],
               [0,0,0,0,0,1],
               [-3,1,1,2,0,-1]])

# pick random values for x and y
x = random.randint(1,1000)
y = random.randint(1,1000)

# this is our orignal formula
out = 3 * x * x * y + 5 * x * y - x - 2 * y + 3 # the witness vector with the intermediate variables inside
v1 = 3*x*x
v2 = v1 * y
w = np.array([1, out, x, y, v1, v2])

result = O.dot(w) == np.multiply(L.dot(w),R.dot(w))
assert result.all(), "result contains an inequality"
```

## 秩一约束系统不要求从单个多项式开始
为简单起见，前面的例子都采用 $z = xy + ...$ 形式，但现实中的算术约束通常是一组约束，而不是单个约束。

例如，假设要证明数组 $[x₁, x₂, x₃, x₄]$ 是二进制数组，并且 $v$ 小于 16。约束组为：

$$
\begin{align*}
x₁² &= x₁\\
x₂² &= x₂\\
x₃² &= x₃\\
x₄² &= x₄ \\
v &= 8x₄ + 4x₃ + 2x₂ + x₁
\end{align*}
$$

为了把它转换为秩一约束系统，可以注意到最后一行没有乘法，因此把 $x_1$ 代入第一个约束：

$$
\begin{align*}
x₁² &= v - 8x₄ - 4x₃ - 2x₂\\
x₂² &= x₂\\
x₃² &= x₃\\
x₄² &= x₄ \\
\end{align*}
$$

假设见证向量为 $[1, v, x_1, x_2, x_3, x_4]$，可按以下方式创建 R1CS：

$$
\begin{align*}
\mathbf{L} &= \begin{bmatrix}
0 & 0 & 1 & 0 & 0 & 0 \\
0 & 0 & 0 & 1 & 0 & 0 \\
0 & 0 & 0 & 0 & 1 & 0 \\
0 & 0 & 0 & 0 & 0 & 1 \\
\end{bmatrix} \\
\mathbf{R} &= \begin{bmatrix}
0 & 0 & 1 & 0 & 0 & 0 \\
0 & 0 & 0 & 1 & 0 & 0 \\
0 & 0 & 0 & 0 & 1 & 0 \\
0 & 0 & 0 & 0 & 0 & 1 \\
\end{bmatrix} \\
\mathbf{O} &= \begin{bmatrix}
0 & 1 & 0 & -2 & -4 & -8 \\
0 & 0 & 0 & 1 & 0 & 0 \\
0 & 0 & 0 & 0 & 1 & 0 \\
0 & 0 & 0 & 0 & 0 & 1 \\
\end{bmatrix} \\
\end{align*}
$$

严格来说，并非必须进行代入，但它可以为 R1CS 节省一行。后文会展示一个不进行代入也仍然有效的 R1CS。

## R1CS 中的所有运算都对素数取模
前面的例子为了简单而使用普通算术，但现实实现使用的是模运算。

原因很简单：编码 2/3 之类的数会产生行为不佳的浮点数，不仅计算开销大，而且容易出错。

如果所有计算都对一个素数取模，例如 23，那么编码 $2/3$ 就很直接。它等于 $2 \cdot 3^{-1}$；在模运算中，乘以二以及取负一次幂都很容易。

## Circom 实现
Circom 是一种用于构造秩一约束系统的语言，其有限域使用素数 $21888242871839275222246405745257275088548364400416034343698204186575808495617$（它等于[有限域上的椭圆曲线](https://www.rareskills.io/post/elliptic-curves-finite-fields)一章所讨论 BN128 曲线的阶）。

这意味着，在该表示中，$-1$ 是：

```python
p = 21888242871839275222246405745257275088548364400416034343698204186575808495617

# 1 - 2 = -1
(1 - 2) % p

# 21888242871839275222246405745257275088548364400416034343698204186575808495616
```

### out = x * y 的 Circom 实现
如果在 Circom 中编写 `out = x * y`，代码如下：

```javascript
pragma circom 2.0.0;

template Multiply2() {
    signal input x;
    signal input y;
    signal output out;

    out <== x * y;
 }

component main = Multiply2();
```

把它转换为 R1CS 文件并打印：

```bash
circom multiply2.circom --r1cs --sym
snarkjs r1cs print multiply2.r1cs
```

得到以下输出：

![Circom 编译的控制台输出](https://static.wixstatic.com/media/935a00_ce8574af090e4b4d8465fd45d7dda8ff~mv2.png/v1/fill/w_1480,h_496,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_ce8574af090e4b4d8465fd45d7dda8ff~mv2.png)

它看起来与我们的 R1CS 解法很不一样，但实际上编码了相同的信息。

Circom 的实现存在以下差异：
- 不打印值为零的列。
- Circom 不把 $\mathbf{O}\mathbf{a} = \mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a}$ 原样写出，而是写成 $\mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a}  - \mathbf{O}\mathbf{a} = \mathbf{0}$。

那么，实际上等于 -1 的 21888242871839275222246405745257275088548364400416034343698204186575808495616 又是怎么回事？

Circom 的解为：

$$
\begin{align*}
A &= \begin{bmatrix}
0 & 0 & -1 & 0
\end{bmatrix}\\
B &= \begin{bmatrix}
0 & 0 & 0 & 1
\end{bmatrix}\\
C &= \begin{bmatrix}
0 & -1 & 0 & 0
\end{bmatrix}
\end{align*}
$$

虽然这些负一可能出乎意料，但对见证向量 `[1 out x y]` 而言，它确实符合 $\mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a} - \mathbf{O}\mathbf{a} = \mathbf{0}$ 的形式。（马上会看到，Circom 确实采用了这种列分配。）

可以代入 $x$、$y$ 和 out 的值，检查等式 $\mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a} - \mathbf{O}\mathbf{a} = \mathbf{0}$ 是否成立。

下面查看 Circom 的变量—列分配。使用 wasm 求解器重新编译电路：

```bash
circom multiply2.circom --r1cs --wasm --sym

cd multiply2_js/
```

创建 `input.json` 文件：

```bash
echo '{"x": "11", "y": "9"}' > input.json
```

然后计算见证：

```bash
node generate_witness.js multiply2.wasm input.json witness.wtns

snarkjs wtns export json witness.wtns witness.json

cat witness.json
```

得到以下结果：

![计算见证后的终端输出](https://static.wixstatic.com/media/935a00_6ffb172a2e7649cab8cc6db8cace8de8~mv2.png/v1/fill/w_1428,h_316,al_c,lg_1,q_90,enc_auto/935a00_6ffb172a2e7649cab8cc6db8cace8de8~mv2.png)

显然，Circom 使用了与本文相同的列布局 `[1, out, x, y]`，因为在 `input.json` 中，$x$ 被设为 $11$，$y$ 被设为 $9$。

如果使用 Circom 生成的见证，并为便于阅读而把那个巨大的数替换为 -1，就可以看到 Circom 的 R1CS 是正确的：

$$
\begin{align*}
\mathbf{a} &= \begin{bmatrix}
1 & 99 & 11 & 9
\end{bmatrix}\\
\mathbf{L} &= \begin{bmatrix}
0 & 0 & -1 & 0
\end{bmatrix} \rightarrow \mathbf{L}\mathbf{a} = -11\\
\mathbf{R} &= \begin{bmatrix}
0 & 0 & 0 & 1
\end{bmatrix} \rightarrow \mathbf{R}\mathbf{a} = 9\\
\mathbf{O} &= \begin{bmatrix}
0 & -1 & 0 & 0
\end{bmatrix} \rightarrow \mathbf{O}\mathbf{a} = -99
\end{align*}
$$

$$
\begin{align*}
Aw \cdot Bw - Cw &= 0\\
(-11)(9) - (-99) &= 0 \\
-99 + 99 &= 0
\end{align*}
$$

$\mathbf{L}$ 在 $x$ 的位置有一个系数 $-1$，$\mathbf{R}$ 在 $y$ 的位置有一个系数 $+1$，$\mathbf{O}$ 在 $\text{out}$ 的位置有 $-1$。在模形式下，这与上面的终端输出完全相同：

![R1CS 的终端输出](https://static.wixstatic.com/media/935a00_36651c70d5aa49d89059cbae553be7e9~mv2.png/v1/fill/w_1480,h_114,al_c,lg_1,q_85,enc_auto/935a00_36651c70d5aa49d89059cbae553be7e9~mv2.png)

### 检查其余转换
回顾一下，前面研究过以下公式：

$$
\begin{align}
z &= x * y \\
z &= x * y * z * u \\
z &= x * y + 2 \\
z &= 3x^2 y + 5xy - x - 2y + 3
\end{align}
$$

上一节已经处理了公式 (1)。本节将说明一个原则：非常数乘法的次数就是约束数。

公式 (2) 的电路为：
```javascript
pragma circom 2.0.8;

template Multiply4() {
    signal input x;
    signal input y;
    signal input z;
    signal input u;

    signal v1;
    signal v2;

    signal out;

    v1 <== x * y;
    v2 <== z * u;

    out <== v1 * v2;
}

component main = Multiply4();
```

结合目前所讨论的内容，Circom 输出及其注释应该已经不言自明：

![Multiply4() 约束生成注释图](https://static.wixstatic.com/media/935a00_21b46f6f9ffd4a80b1a2a374be0de279~mv2.png/v1/fill/w_990,h_420,al_c,q_90,enc_auto/935a00_21b46f6f9ffd4a80b1a2a374be0de279~mv2.png)

由此可知，其他公式应分别具有如下数量的约束：

$$
\begin{align}
z &= x * y && \text{1 个约束} \\
z &= x * y * z * u && \text{3 个约束} \\
z &= x * y + 2 && \text{1 个约束} \\
z &= 3x^2 y + 5xy - x - 2y + 3 && \text{3 个约束}
\end{align}
$$

请读者编写 Circom 电路并验证上述结果。

### 计算 R1CS 不需要见证
请注意，在 Circom 代码中，计算 R1CS 之前从未提供见证。前面提供见证，是为了让示例不那么抽象，并便于检查结果；但它并非必要。这一点很重要：如果验证者需要见证才能构造 R1CS，那么证明者就不得不泄露隐藏的解！

这里所说的“见证”是已经填入具体值的向量。验证者知道见证的“结构”，也就是变量与列之间的分配，但不知道其中的值。

## R1CS 即使未经优化也仍然有效
从多项式到 R1CS 的有效转换并不唯一。可以用更多约束编码同一个问题，只是效率较低。下面给出一个例子。

某些 R1CS 教程会把如下公式：

$$
z = x² + y
$$

转换为：

$$
\begin{align*}
v₁ &= x  x \\
z &= v₁ + y
\end{align*}
$$

正如前面指出的，这种方法效率不高。不过，仍可使用本文的方法为它创建有效的 R1CS，只需加入一次虚拟乘法：

$$
\begin{align*}
v₁ &= x  x \\
z &= (v₁ + y) * 1
\end{align*}
$$

见证向量的形式为 $[1, z, x, y, v1]$，$\mathbf{L}$、$\mathbf{R}$、$\mathbf{O}$ 定义如下：

$$
\mathbf{L} = \begin{bmatrix}
0 & 0 & 1 & 0 & 0 \\
0 & 0 & 0 & 1 & 1
\end{bmatrix}
$$

$$
\mathbf{R} = \begin{bmatrix}
0 & 0 & 1 & 0 & 0 \\
1 & 0 & 0 & 0 & 0
\end{bmatrix}
$$

$$
\mathbf{O} = \begin{bmatrix}
0 & 0 & 0 & 0 & 1 \\
0 & 1 & 0 & 0 & 0
\end{bmatrix}
$$

$\mathbf{L}$ 的第二行完成加法，而乘以一则通过使用 $\mathbf{R}$ 第二行的第一个元素实现。

这种方案完全有效，但比实际所需多一行和一列。

## 如果没有乘法怎么办？
如果要编码如下电路，该怎么办？

$$
z = x + y
$$

它在实践中几乎没有用处，但为了完整起见，可以通过一次乘以一的虚拟乘法来解决。

$$
out = (x + y)*1
$$

采用通常的见证向量布局 $[1, z, x, y]$，可得到以下矩阵：

$$
\begin{align*}
\mathbf{L} &= \begin{bmatrix}
0 & 0 & 1 & 1 \\
\end{bmatrix} \\
\mathbf{R} &= \begin{bmatrix}
1 & 0 & 0 & 0 \\
\end{bmatrix} \\
\mathbf{O} &= \begin{bmatrix}
0 & 1 & 0 & 0 \\
\end{bmatrix}
\end{align*}
$$

## 秩一约束系统是为了实现方便
[Groth16 原始论文](https://eprint.iacr.org/2016/260.pdf)没有提到“秩一约束系统”这一术语。从实现角度看，R1CS 很方便；但从纯数学角度看，它只是在显式标记并分组不同变量的系数。因此，阅读相关学术论文时通常看不到 R1CS，因为它只是更抽象概念的一项实现细节。

## 实用资源
- 这个[网页工具可以为一组约束计算 R1CS](https://asecuritysite.com/zero/go_r1cs)（但它只支持一个输入变量和一个输出变量）。

- [Vitalik 著名的 x**3 + x + 5 == 35 示例](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649)

- [Zero Knowledge Blog 的 R1CS 教程](https://www.zeroknowledgeblog.com/index.php/the-pinocchio-protocol/r1cs)

## 通过 RareSkills 继续学习
本文摘自我们的[零知识课程](https://www.rareskills.io/zk-bootcamp)学习资料。
