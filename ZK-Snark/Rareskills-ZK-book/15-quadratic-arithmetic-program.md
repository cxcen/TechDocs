# 二次算术程序（QAP）

二次算术程序（Quadratic Arithmetic Program，QAP）是一种[算术电路](https://www.rareskills.io/post/arithmetic-circuit)，更具体地说，是用一组多项式表示的[秩一约束系统（Rank-1 Constraint System，R1CS）](https://www.rareskills.io/post/rank-1-constraint-system)。它通过对秩一约束系统进行拉格朗日插值得到。与 R1CS 不同，借助 Schwartz–Zippel 引理，可以在 $\mathcal{O}(1)$ 时间内测试二次算术程序的等式是否成立。

## 核心思想
在介绍 Schwartz–Zippel 引理的章节中，我们看到：把两个向量转换成多项式，再对多项式执行 Schwartz–Zippel 测试，就能在 $\mathcal{O}(1)$ 时间内检查两个向量是否相等。（更准确地说，*测试*耗时为 $\mathcal{O}(1)$；把向量转换为多项式仍会产生额外开销。）

因为秩一约束系统完全由向量运算组成，所以我们的目标是在 $\mathcal{O}(1)$ 而不是 $\mathcal{O}(n)$ 时间内检查：

$$
\mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a}\stackrel{?}{=}\mathbf{O}\mathbf{a}
$$

是否成立，其中 $n$ 是 $\mathbf{L}$、$\mathbf{R}$、$\mathbf{O}$ 的行数。

不过在此之前，需要先理解向量与表示这些向量的多项式之间的一些关键性质。

本文所有数学运算都假定发生在[有限域](https://www.rareskills.io/post/finite-fields)中，但为了简洁，省略 $\mod p$ 记号。

## 向量加法与多项式加法之间的同态
### 向量加法同态于多项式加法

取两个向量，分别用多项式进行插值，再把两个多项式相加，所得多项式与先把两个向量相加、再对和向量进行插值得到的多项式相同。

用更数学化的语言表示：令 $\mathcal{L}(\mathbf{v})$ 表示使用 $(1, 2, ..., n)$ 作为 $x$ 值，对向量 $\mathbf{v}$ 进行拉格朗日插值得到的多项式，其中 $n$ 是 $\mathbf{v}$ 的长度。下式成立：

$$
\mathcal{L}(\mathbf{v} + \mathbf{w}) = \mathcal{L}(\mathbf{v}) + \mathcal{L}(\mathbf{w})
$$

换句话说，分别对向量 $\mathbf{v}$、$\mathbf{w}$ 进行插值后所得多项式之和，等于对向量 $\mathbf{v} + \mathbf{w}$ 进行插值得到的多项式。

#### 推导示例
令 $f_1(x) = x^2$、$f_2(x) = x^3$。$f_1$ 对 $(1, 1), (2, 4), (3, 9)$，也就是向量 $[1,4,9]$ 进行插值；$f_2$ 对 $[1,8,27]$ 进行插值。

两个向量的和为 $[2,12,36]$，显然 $x^3 + x^2$ 对它进行插值。令 $f_3(x) = f_1(x) + f_2(x) = x^3 + x^2$。

$$
\begin{align*}
f_3(1) &= 1 + 1 = 2\\
f_3(2) &= 8 + 4 = 12\\
f_3(3) &= 27 + 9 = 36
\end{align*}
$$

#### 使用 Python 检查计算

对一个数学恒等式进行单元测试，并不能证明它成立，但可以展示实际发生了什么。建议读者尝试几个不同的向量，观察该恒等式如何成立。

```python
import galois
import numpy as np

p = 17
GF = galois.GF(p)

xs = GF(np.array([1,2,3]))

# two arbitrary vectors
v1 =  GF(np.array([4,8,2]))
v2 =  GF(np.array([1,6,12]))

def L(v):
    return galois.lagrange_poly(xs, v)

assert L(v1 + v2) == L(v1) + L(v2)
```

### 标量乘法
令 $\lambda$ 为标量，更具体地说，是有限域中的域元素。则：

$$
\mathcal{L}(\lambda \mathbf{v}) = \lambda \mathcal{L}(\mathbf{v})
$$

#### 推导示例
假设三个点为 $[3, 6, 11]$，对其进行插值的多项式是 $f(x) = x^2 + 2$。向量乘以 3 后得到 $[9, 18, 33]$，对它进行插值的多项式为：

```python
from scipy.interpolate import lagrange

x_values = [1, 2, 3]
y_values = [9, 18, 33]

print(lagrange(x_values, y_values))

#    2
# 3 x + 6
```

即 $3x^2 + 6$，它等于 $3 \cdot (x^2 + 2)$。

#### 代码示例
```python
import galois
import numpy as np

p = 17
GF = galois.GF(p)

xs = GF(np.array([1,2,3]))

# arbitrary vector
v =  GF(np.array([4,8,2]))

# arbitrary constant
lambda_ =  GF(15)

def L(v):
    return galois.lagrange_poly(xs, v)

assert L(lambda_ * v) == lambda_ * L(v)
```

### 标量乘法其实就是向量加法
所谓“把向量乘以 3”，其实就是“把该向量与自身相加三次”。因为本文只处理有限域，所以不必考虑如何解释“0.5”之类的标量。

可以把有限域中逐元素加法下的向量和有限域中加法下的多项式都视为[群](https://www.rareskills.io/post/group-theory)。

本章最重要的结论是：

**有限域中加法下的向量群，同态于有限域中加法下的多项式群。**

这一点至关重要，因为**测试向量是否相等需要 $\mathcal{O}(n)$ 时间，而测试多项式是否相等只需 $\mathcal{O}(1)$ 时间。**

因此，测试 R1CS 等式原本需要 $\mathcal{O}(n)$ 时间，现在可以利用这种同态，在 $\mathcal{O}(1)$ 时间内完成测试。

这就是*二次算术程序*。

## 用多项式表示秩一约束系统

矩形矩阵与向量之间的矩阵乘法，可以用向量加法和标量乘法表示。

例如，给定一个 $3 \times 4$ 矩阵 $A$ 和一个四维向量 $v$，矩阵乘法可以写成：

$$
\mathbf{A} \cdot \mathbf{v} = \begin{bmatrix}
a_{11} & a_{12} & a_{13} & a_{14}\\
a_{21} & a_{22} & a_{23} & a_{24}\\
a_{31} & a_{32} & a_{33} & a_{34}
\end{bmatrix}
\begin{bmatrix}
v_1\\
v_2\\
v_3\\
v_4
\end{bmatrix}
$$

通常可以把它理解为向量 $v$ “翻转”后，分别与矩阵的每一行进行内积（广义点积），即：

$$
\mathbf{A}\cdot \mathbf{v} =
\begin{bmatrix}
a_{11}\cdot v_1 + a_{12}\cdot v_2 + a_{13}\cdot v_3 + a_{14}\cdot v_4\\
a_{21}\cdot v_1 + a_{22}\cdot v_2 + a_{23}\cdot v_3 + a_{24}\cdot v_4\\
a_{31}\cdot v_1 + a_{32}\cdot v_2 + a_{33}\cdot v_3 + a_{34}\cdot v_4
\end{bmatrix}
$$

不过，也可以把矩阵 $A$ 拆分为如下多个向量：

$$
\mathbf{A} = \begin{bmatrix}
a_{11} \\
a_{21} \\
a_{31}
\end{bmatrix}
,
\begin{bmatrix}
a_{12} \\
a_{22} \\
a_{32}
\end{bmatrix}
,
\begin{bmatrix}
a_{13} \\
a_{23} \\
a_{33}
\end{bmatrix}
,
\begin{bmatrix}
a_{14} \\
a_{24} \\
a_{34}
\end{bmatrix}
$$

然后让每个向量分别乘以向量 $\mathbf{v}$ 中对应的标量：

$$
\mathbf{A}\cdot \mathbf{v} = \begin{bmatrix}
a_{11} \\
a_{21} \\
a_{31}
\end{bmatrix}\cdot v_1
+
\begin{bmatrix}
a_{12} \\
a_{22} \\
a_{32}
\end{bmatrix}\cdot v_2
+
\begin{bmatrix}
a_{13} \\
a_{23} \\
a_{33}
\end{bmatrix}\cdot v_3
+
\begin{bmatrix}
a_{14} \\
a_{24} \\
a_{34}
\end{bmatrix}\cdot v_4
$$

这样就完全使用向量加法和标量乘法，表示了 $\mathbf{A}$ 与 $\mathbf{v}$ 之间的矩阵乘法。

前面已经证明，有限域中加法下的向量群同态于有限域中加法下的多项式群，因此可以用表示这些向量的多项式来表达上述计算。

## 简洁测试 $\mathbf{A}\mathbf{v}_1 = \mathbf{B}\mathbf{v}_2$

假设有矩阵 $\mathbf{A}$ 和 $\mathbf{B}$：

$$
\begin{align*}
\mathbf{A} = \begin{bmatrix}
6 & 3\\
4 & 7\\
\end{bmatrix}\\
\mathbf{B} = \begin{bmatrix}
3 & 9 \\
12 & 6\\
\end{bmatrix}
\end{align*}
$$

以及向量 $\mathbf{v}_1$ 和 $\mathbf{v}_2$：

$$
\begin{align*}
\mathbf{v}_1 = \begin{bmatrix}
2 \\
4 \\
\end{bmatrix}\\
\mathbf{v}_2 = \begin{bmatrix}
2 \\
2 \\
\end{bmatrix}
\end{align*}
$$

要测试：

$$
\mathbf{A}\mathbf{v}_1 = \mathbf{B}\mathbf{v}_2
$$

是否成立。

显然，可以直接执行矩阵运算，但最终检查需要进行 $n$ 次比较，其中 $n$ 是 $\mathbf{A}$ 和 $\mathbf{B}$ 的行数。我们的目标是在 $\mathcal{O}(1)$ 时间内完成。

首先，把矩阵乘法 $\mathbf{A}\mathbf{v}_1$ 和 $\mathbf{B}\mathbf{v}_2$ 转换到加法下的向量群：

$$
\begin{align*}
\mathbf{A} &= \begin{bmatrix}
6 \\
4 \\
\end{bmatrix}
,
\begin{bmatrix}
3 \\
7 \\
\end{bmatrix}\\
\mathbf{B} &= \begin{bmatrix}
3 \\
12 \\
\end{bmatrix}
,
\begin{bmatrix}
9 \\
6 \\
\end{bmatrix}
\end{align*}
$$

现在，要在多项式群中寻找下式的同态等价形式：

$$
\begin{bmatrix}
6 \\
4 \\
\end{bmatrix}\cdot 2+
\begin{bmatrix}
3 \\
7 \\
\end{bmatrix}\cdot 4\stackrel{?}{=}
\begin{bmatrix}
3 \\
12 \\
\end{bmatrix}\cdot 2+
\begin{bmatrix}
9 \\
6 \\
\end{bmatrix}\cdot 2
$$

把每个向量都转换为使用 $x$ 值 $[1,2]$ 插值得到的多项式：

$$
\underbrace{
\begin{bmatrix}
6 \\
4 \\
\end{bmatrix}}_{p_1(x)}\cdot 2+
\underbrace{
\begin{bmatrix}
3 \\
7 \\
\end{bmatrix}}_{p_2(x)}\cdot 4\stackrel{?}{=}
\underbrace{
\begin{bmatrix}
3 \\
12 \\
\end{bmatrix}}_{q_1(x)}\cdot 2+
\underbrace{
\begin{bmatrix}
9 \\
6 \\
\end{bmatrix}}_{q_2(x)}\cdot 2
$$

下面用 Python 计算拉格朗日插值：

```python
import galois
import numpy as np

p = 17
GF = galois.GF(p)

x_values = GF(np.array([1, 2]))

def L(v):
    return galois.lagrange_poly(x_values, v)

p1 = L(GF(np.array([6, 4])))
p2 = L(GF(np.array([3, 7])))
q1 = L(GF(np.array([3, 12])))
q2 = L(GF(np.array([9, 6])))

print(p1)
# 15x + 8 (mod 17)
print(p2)
# 4x + 16 (mod 17)
print(q1)
# 9x + 11 (mod 17)
print(q2)
# 14x + 12 (mod 17)
```

最后，可以应用 Schwartz–Zippel 引理检查：

$$
p_1(x) \cdot 2+ p_2(x) \cdot 4 \stackrel{?}= q_1(x) \cdot 2 + q_2(x) \cdot 2
$$

是否成立：

```python
import random
u = random.randint(0, p)
tau = GF(u) # a random point

left_hand_side = p1(tau) * GF(2) + p2(tau) * GF(4)
right_hand_side = q1(tau) * GF(2) + q2(tau) * GF(2)

assert left_hand_side == right_hand_side
```

**最后的 assert 语句只需一次比较，而不是 $n$ 次，就能测试 $\mathbf{A}\mathbf{v}_1 = \mathbf{B}\mathbf{v}_2$ 是否成立。**

## 从 R1CS 到 QAP：简洁测试 $\mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a}=\mathbf{O}\mathbf{a}$
既然已经知道如何简洁测试 $\mathbf{A}\mathbf{v}_1 = \mathbf{B}\mathbf{v}_2$，能否同样简洁地测试 $\mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a}=\mathbf{O}\mathbf{a}$？

这些矩阵有 $m$ 列，因此把每个矩阵拆分成 $m$ 个列向量，再使用 $(1, 2, ..., n)$ 对每个列向量进行插值，使每个矩阵分别得到 $m$ 个多项式。

令 $u_1(x), u_2(x), ..., u_m(x)$ 表示对 $\mathbf{L}$ 的列向量进行插值得到的多项式。

令 $v_1(x), v_2(x), ..., v_m(x)$ 表示对 $\mathbf{R}$ 的列向量进行插值得到的多项式。

令 $w_1(x), w_2(x), ..., w_m(x)$ 表示对 $\mathbf{O}$ 的列向量进行插值得到的多项式。

不失一般性，假设有 4 列（$m = 4$）和三行（$n = 3$）。

可以直观表示为：
$$
\begin{array}{c}
\mathbf{L} = \begin{bmatrix}
\quad l_{11} \quad& l_{12} \quad& l_{13} \quad& l_{14} \quad\\
\quad l_{21} \quad& l_{22} \quad& l_{23} \quad& l_{24} \quad\\
\quad l_{31} \quad& l_{32} \quad& l_{33} \quad& l_{34} \quad
\end{bmatrix} \\
\\
\qquad u_1(x) \quad u_2(x) \quad u_3(x) \quad u_4(x)
\end{array}
\begin{array}{c}
\mathbf{R} = \begin{bmatrix}
\quad r_{11} \quad& r_{12} \quad& r_{13} \quad& r_{14} \quad\\
\quad r_{21} \quad& r_{22} \quad& r_{23} \quad& r_{24} \quad\\
\quad r_{31} \quad& r_{32} \quad& r_{33} \quad& r_{34} \quad
\end{bmatrix} \\
\\
\qquad v_1(x) \quad v_2(x) \quad v_3(x) \quad v_4(x)
\end{array}
$$

$$
\begin{array}{c}
\mathbf{O} = \begin{bmatrix}
\quad o_{11} \quad& o_{12} \quad& o_{13} \quad& o_{14} \quad\\
\quad o_{21} \quad& o_{22} \quad& o_{23} \quad& o_{24} \quad\\
\quad o_{31} \quad& o_{32} \quad& o_{33} \quad& o_{34} \quad
\end{bmatrix} \\
\\
\qquad w_1(x) \quad w_2(x) \quad w_3(x) \quad w_4(x)
\end{array}
$$

列向量乘以标量，同态于多项式乘以标量，因此每个多项式都可以乘以见证中的对应元素。

例如：

$$
\mathbf{L}\mathbf{a} = \begin{bmatrix}
\quad l_{11} \quad& l_{12} \quad& l_{13} \quad& l_{14} \quad\\
\quad l_{21} \quad& l_{22} \quad& l_{23} \quad& l_{24} \quad\\
\quad l_{31} \quad& l_{32} \quad& l_{33} \quad& l_{34} \quad
\end{bmatrix}
\begin{bmatrix}
a_1 \\
a_2 \\
a_3 \\
a_4
\end{bmatrix}
$$

转换为：

$$
\begin{align*}
&=\begin{bmatrix}
u_1(x) & u_2(x) & u_3(x) & u_4(x)
\end{bmatrix}
\begin{bmatrix}
a_1 \\
a_2 \\
a_3 \\
a_4
\end{bmatrix}\\
&=a_1u_1(x) + a_2u_2(x) + a_3u_3(x) + a_4u_4(x)\\
&=\sum_{i=1}^4 a_iu_i(x)
\end{align*}
$$

注意，最终结果是一个次数至多为 $n - 1$ 的多项式，因为 $u_1(x), \ldots, u_n(x)$ 各自的次数都至多为 $n - 1$。
这是由其构造方式决定的：$\mathbf{L}$ 的每一列都有 $n$ 个元素，通过拉格朗日插值构造经过 $n$ 个点的多项式，所得多项式次数至多为 $n - 1$。

一般情况下，把 $m$ 个列向量分别转换为多项式后，$\mathbf{L}\mathbf{a}$ 可以写成：

$$
\sum_{i=1}^m a_iu_i(x)
$$

采用相同步骤，R1CS $\mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a} = \mathbf{O}\mathbf{a}$ 中的每个矩阵—见证乘积可转换为：

$$
\begin{align*}
\mathbf{L}\mathbf{a} \rightarrow \sum_{i=1}^m a_iu_i(x) \\
\mathbf{R}\mathbf{a} \rightarrow \sum_{i=1}^m a_iv_i(x) \\
\mathbf{O}\mathbf{a} \rightarrow \sum_{i=1}^m a_iw_i(x)
\end{align*}
$$

因为每个求和项都会得到一个多项式，可以把它们写成：

$$
\begin{align*}
\mathbf{L}\mathbf{a} &\rightarrow \sum_{i=1}^m a_iu_i(x) = u(x)\\
\mathbf{R}\mathbf{a} &\rightarrow \sum_{i=1}^m a_iv_i(x) = v(x)\\
\mathbf{O}\mathbf{a} &\rightarrow \sum_{i=1}^m a_iw_i(x) = w(x)
\end{align*}
$$

### 为什么要对所有列进行插值？
根据同态关系 $\mathcal{L}(\mathbf{v}_1) + \mathcal{L}(\mathbf{v}_2) = \mathcal{L}(\mathbf{v}_1 + \mathbf{v}_2)$ 和 $\mathcal{L}(\lambda \mathbf{v}) = \lambda \mathcal{L}(\mathbf{v})$，如果把 $u(x)$ 计算为 $\mathcal{L}(\mathbf{L}\mathbf{a})$，所得结果与先对 $\mathbf{L}$ 的各列进行拉格朗日插值，再让每个多项式分别乘以 $\mathbf{a}$ 中的对应元素并求和相同。

换一种说法：

$$
\sum_{i=1}^m a_iu_i(x) = \mathcal{L}(\mathbf{L}\mathbf{a}) \mid u_i(x) \text{ 是第 } i \text{ 列的拉格朗日插值，该列来自 } \mathbf{L}
$$

既然如此，为什么不只计算一次拉格朗日插值，而要计算 $m$ 次？

这里必须区分 QAP 的*使用者*。验证者以及稍后将介绍的可信设置并不知道见证 $\mathbf{a}$，因此无法计算 $\mathcal{L}(\mathbf{L}\mathbf{a})$。证明者可以采用这种优化，但 ZK 协议中的其他参与方无法利用它。

在执行任何证明和验证之前，所有参与方都必须对 QAP 达成共同约定，也就是对各矩阵进行多项式插值的结果。

### 多项式次数不平衡

不过，不能简单地把最终结果表示为：

$$
u(x)v(x) = w(x)
$$

因为两侧的次数不匹配。

两个多项式相乘时，乘积多项式的次数等于两个因子多项式次数之和。

由于 $u(x)$、$v(x)$、$w(x)$ 的次数都为 $n - 1$，$u(x)v(x)$ 的次数通常为 $2n - 2$，而 $w(x)$ 的次数为 $n - 1$。因此，即使它们相乘所对应的底层向量相等，这两个多项式也不会相等。

这是因为前面建立的同态只涉及向量加法，并不涉及逐元素乘积（Hadamard product）。

不过，$u(x)v(x)$ 所插值的向量，即：

$$
((1, u(1)v(1)), (2, u(2)v(2)), ..., (n, u(n)v(n)))
$$

与 $w(x)$ 所插值的向量相同，即：

$$
((1, w(1)), \quad (2, w(2)), \quad ..., \quad (n, w(n)))
$$

换句话说：

$$
((1, u(1)v(1)), (2, u(2)v(2)), ..., (n, u(n)v(n))) = ((1, w(1)), (2, w(2)), ..., (n, w(n)))
$$

虽然“底层”向量相等，但对它们进行插值的多项式并不相等。

### 底层相等示例
假设 $u(x)$ 是对以下点进行插值的多项式：

$$
(1,\boxed{2}), (2,\boxed{4}), (3,\boxed{8})
$$

而 $v(x)$ 是对以下点进行插值的多项式：

$$
(1,\boxed{4}), (2,\boxed{2}), (3,\boxed{8})
$$

如果把 $u(x)$ 看作对向量 $[2,4,8]$ 进行插值，把 $v(x)$ 看作对向量 $[4,2,8]$ 进行插值，就可以看到，它们的乘积多项式对两个向量的逐元素乘积进行插值。$[2,4,8]$ 与 $[4,2,8]$ 的逐元素乘积是 $[8,8,64]$。

把 $u(x)$ 与 $v(x)$ 相乘，得到 $w(x) = 4x^4 - 18x^3 + 36x^2 - 42x + 28$。

下图显示，乘积多项式对两个向量的逐元素乘积 $[8, 8, 64]$ 进行插值。

![u、v、w 在三个点相交](https://r2media.rareskills.io/qap-3-point-cross.png)

如果 $w(x)$ 与 $u(x)v(x)$ 在 $(1,2,...,n)$ 上插值得到相同的 $y$ 值，如何“让”二者相等？

### 对 $\mathbf{0}$ 向量进行插值
如果 $\mathbf{v_1} \circ \mathbf{v_2} = \mathbf{v_3}$，那么 $\mathbf{v_1} \circ \mathbf{v_2} = \mathbf{v_3} + \mathbf{0}$。

与其使用拉格朗日插值对 $\mathbf{0}$ 进行插值并得到 $f(x) = 0$——请记住，拉格朗日插值寻找的是次数最低的插值多项式——不如使用一个次数更高的多项式来平衡次数差异。

例如，下图中的黑色多项式 $b(x)$ 对 $[(1,0), (2,0), (3,0)]$ 进行插值：

![零向量插值多项式图](https://r2media.rareskills.io/qap-zero-polynomial.png)

现在，由于 $4x^4 -18x^3 + 8x^2 + 42x - 36$ 是对 $[0,0,0]$ 的有效插值，可以把原式写成：

$u(x)v(x) = w(x) + b(x)$

这样等式就成立了！

$b(x)$ 只是通过 $u(x)v(x) - w(x)$ 计算得到，即<span style="color:#008aff">蓝色</span>多项式减去<span style="color:red">红色</span>多项式。

不过，不能让证明者任意选择 $b(x)$，否则即使 $u(x)v(x)$ 和 $w(x)$ 没有对相同向量进行插值——本例中为 $[8, 8, 64]$——证明者也可以选出一个让等式成立的 $b(x)$。证明者在选择 $b(x)$ 时拥有过大的自由度。具体而言，需要强制 $b(x)$ 在 $x = 1,2,\dots,n$ 处具有根，也就是让它对 $\mathbf{0}$ 向量进行插值。这样，$\mathbf{v}_1 \circ \mathbf{v}_2 = \mathbf{v}_3 + \mathbf{0}$ 的多项式转换仍然尊重底层向量关系。

为了限制 $b(x)$ 的选择，可以使用以下定理：

#### 多项式乘积的根之并集
**定理**：如果 $h(x) = f(x)g(x)$，且 $f(x)$ 的根集合为 $\set{r_f}$，$g(x)$ 的根集合为 $\set{r_g}$，那么 $h(x)$ 的根为 $\set{r_f} \cup \set{r_g}$。

##### 示例
令 $f(x) = (x - 3)(x - 4)$、$g(x) = (x - 5)(x - 6)$，则 $h(x) = f(x)g(x)$ 的根为 $\set{3,4,5,6}$。

可以使用上述定理，强制 $b(x)$ 在 $x = 1,2,\dots,n$ 处具有根。

#### 强制 $b(x)$ 表示零向量
把 $b(x)$ 分解为 $b(x) = h(x)t(x)$，其中 $t(x)$ 为：

$$
t(x) = (x-1)(x-2)\dots(x-n)
$$

任何多项式与 $t(x)$ 相乘后都仍表示零向量，因为所得多项式必然在 $x = 1,2,\dots,n$ 处具有根。

因此，在等式中用 $h(x)t(x)$ 替换 $b(x)$。

等式将变为：

$$
u(x)v(x) = w(x) + h(x)t(x)
$$

可以用基本代数计算 $h(x)$：

$$
h(x) = \frac{u(x)v(x) - w(x)}{t(x)}
$$

## QAP 端到端推导
假设有一个 R1CS，其矩阵为 $\mathbf{L}$、$\mathbf{R}$、$\mathbf{O}$，见证向量为 $\mathbf{a}$。

$$
\mathbf{L}\mathbf{a}\circ\mathbf{R}\mathbf{a} = \mathbf{O}\mathbf{a}
$$

这些矩阵有 $n$ 列、$m$ 行，其中 $n = 4$、$m = 3$。

也就是说，$\mathbf{L}$、$\mathbf{R}$ 和 $\mathbf{O}$ 如下：

$$
\begin{array}{c}
\mathbf{L} = \begin{bmatrix}
\quad l_{11} \quad& l_{12} \quad& l_{13} \quad& l_{14} \quad\\
\quad l_{21} \quad& l_{22} \quad& l_{23} \quad& l_{24} \quad\\
\quad l_{31} \quad& l_{32} \quad& l_{33} \quad& l_{34} \quad
\end{bmatrix} \\
\\
\mathbf{R} = \begin{bmatrix}
\quad r_{11} \quad& r_{12} \quad& r_{13} \quad& r_{14} \quad\\
\quad r_{21} \quad& r_{22} \quad& r_{23} \quad& r_{24} \quad\\
\quad r_{31} \quad& r_{32} \quad& r_{33} \quad& r_{34} \quad
\end{bmatrix} \\
\\
\mathbf{O} = \begin{bmatrix}
\quad o_{11} \quad& o_{12} \quad& o_{13} \quad& o_{14} \quad\\
\quad o_{21} \quad& o_{22} \quad& o_{23} \quad& o_{24} \quad\\
\quad o_{31} \quad& o_{32} \quad& o_{33} \quad& o_{34} \quad
\end{bmatrix}
\end{array}
$$

见证向量 $\mathbf{a}$ 为：

$$
\mathbf{a} = \begin{bmatrix}
a_1 \\ a_2 \\ a_3 \\ a_4
\end{bmatrix}
$$

把每个矩阵拆分为 $m$ 个列向量，并使用 $(1, 2, ..., n)$ 对它们进行插值，使每个矩阵分别得到 $m$ 个多项式。

$$
\begin{array}{c}
\mathbf{L} = \underbrace{\begin{bmatrix}
l_{11} \\l_{12} \\ l_{13} \\ l_{14} \\
\end{bmatrix}}_{u_1(x)}
\quad
\underbrace{\begin{bmatrix}
l_{21} \\ l_{22} \\ l_{23} \\ l_{24}
\end{bmatrix}}_{u_2(x)}
\quad
\underbrace{\begin{bmatrix}
l_{31} \\ l_{32} \\ l_{33} \\ l_{34} \\
\end{bmatrix}}_{u_3(x)}
\quad
\underbrace{\begin{bmatrix}
l_{41} \\ l_{42} \\ l_{43} \\ l_{44}
\end{bmatrix}}_{u_4(x)}
\end{array}
$$

$$
\begin{array}{c}
\mathbf{R} = \underbrace{\begin{bmatrix}
r_{11} \\r_{12} \\ r_{13} \\ r_{14} \\
\end{bmatrix}}_{v_1(x)}
\quad
\underbrace{\begin{bmatrix}
r_{21} \\ r_{22} \\ r_{23} \\ r_{24}
\end{bmatrix}}_{v_2(x)}
\quad
\underbrace{\begin{bmatrix}
r_{31} \\ r_{32} \\ r_{33} \\ r_{34} \\
\end{bmatrix}}_{v_3(x)}
\quad
\underbrace{\begin{bmatrix}
r_{41} \\ r_{42} \\ r_{43} \\ r_{44}
\end{bmatrix}}_{v_4(x)}
\end{array}
$$

$$
\begin{array}{c}
\mathbf{O} = \underbrace{\begin{bmatrix}
o_{11} \\o_{12} \\ o_{13} \\ o_{14} \\
\end{bmatrix}}_{w_1(x)}
\quad
\underbrace{\begin{bmatrix}
o_{21} \\ o_{22} \\ o_{23} \\ o_{24}
\end{bmatrix}}_{w_2(x)}
\quad
\underbrace{\begin{bmatrix}
o_{31} \\ o_{32} \\ o_{33} \\ o_{34} \\
\end{bmatrix}}_{w_3(x)}
\quad
\underbrace{\begin{bmatrix}
o_{41} \\ o_{42} \\ o_{43} \\ o_{44}
\end{bmatrix}}_{w_4(x)}
\end{array}
$$

矩阵—向量乘积 $\mathbf{L}\mathbf{a}$、$\mathbf{R}\mathbf{a}$、$\mathbf{O}\mathbf{a}$ 分别同态等价于以下多项式：

$$
\begin{align*}
\sum_{i=1}^4 a_iu_i(x) &= a_1u_1(x) + a_2u_2(x) + a_3u_3(x) + a_4u_4(x) = u(x) \\
\sum_{i=1}^m a_iv_i(x) &= a_1v_1(x) + a_2v_2(x) + a_3v_3(x) + a_4v_4(x) = v(x) \\
\sum_{i=1}^m a_iw_i(x) &= a_1w_1(x) + a_2w_2(x) + a_3w_3(x) + a_4w_4(x) = w(x) \\
\end{align*}
$$

在本例中，$t(x)$ 为：

$$
t(x) = (x - 1)(x - 2)(x - 3)
$$

而 $h(x)$ 为：

$$
h(x) = \frac{u(x)v(x) - w(x)}{t(x)}
$$

原 R1CS 的 QAP 表示最终为：

$$
\sum_{i=1}^4 a_iu_i(x)\sum_{i=1}^4 a_iv_i(x) = \sum_{i=1}^4 a_iw_i(x) + h(x)t(x)
$$

## QAP 的最终公式
QAP 是以下公式：

$$
\sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x) = \sum_{i=1}^m a_iw_i(x) + h(x)t(x)
$$

其中，$u_i(x)$、$v_i(x)$、$w_i(x)$ 分别是对 $\mathbf{L}$、$\mathbf{R}$、$\mathbf{O}$ 各列进行插值得到的多项式；$t(x)$ 是 $(x - 1)(x - 2)...(x - n)$，其中 $n$ 是 $\mathbf{L}$、$\mathbf{R}$、$\mathbf{O}$ 的行数；而 $h(x)$ 为：

$$
h(x) = \frac{\sum_{i=1}^ma_iu_i(x)\sum_{i=1}^ma_iv_i(x) - \sum_{i=1}^ma_iw_i(x)}{t(x)}
$$

## 使用二次算术程序构建简洁零知识证明
假设验证者可以向证明者发送一个随机值 $\tau$，而证明者返回：

$$
\begin{align*}
A &= u(\tau)\\
B &= v(\tau)\\
C &= w(\tau) + h(\tau)t(\tau)
\end{align*}
$$

验证者可以检查 $AB = C$，并据此接受证明者拥有满足 R1CS 与 QAP 的有效见证 $\mathbf{a}$。

不过，这要求验证者相信证明者正确地对多项式进行了求值，而当前没有机制强制证明者这样做。

下一章将根据本章讨论，用 Python 代码演示如何[把 R1CS 转换为 QAP](https://www.rareskills.io/post/r1cs-to-qap)。

之后将讨论可信设置，开始解决如何让证明者诚实地对多项式求值这一问题。

*最初发表于 2023 年 8 月 23 日*
