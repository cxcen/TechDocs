# 使用 Python 将 R1CS 转换为有限域上的 QAP

为了让从秩一约束系统（Rank-1 Constraint System，R1CS）到二次算术程序（Quadratic Arithmetic Program，QAP）的转换不那么抽象，我们来看一个具体示例。

假设要编码如下[算术电路](https://www.rareskills.io/post/arithmetic-circuit)：

$$
z = x⁴ - 5y²x²
$$

将其转换为 [R1CS](https://www.rareskills.io/post/rank-1-constraint-system)，得到：

$$
\begin{align*}
v_1 &= xx \\
v_2 &= v_1 * v_1 && //x^4\\
v_3 &= -5yy \\
-v_2 + z &= v_3 * v_1 && //-5y^2 * x^2\\
\end{align*}
$$

我们需要为承载这些运算的[有限域](https://www.rareskills.io/post/finite-fields)选择一个特征。当后续把它与[椭圆曲线](https://www.rareskills.io/post/elliptic-curves-finite-fields)结合时，素域的阶必须等于椭圆曲线的阶（两者不匹配是一个非常常见的错误）。

不过目前先选择一个较小的数，以便手工处理。这里选用素数 79。

首先定义矩阵 $\mathbf{L}$、$\mathbf{R}$ 和 $\mathbf{O}$：

```python
import numpy as np

# 1, out, x, y, v1, v2, v3
L = np.array([
    [0, 0, 1, 0, 0, 0, 0],
    [0, 0, 0, 0, 1, 0, 0],
    [0, 0, 0, -5, 0, 0, 0],
    [0, 0, 0, 0, 0, 0, 1],
])

R = np.array([
    [0, 0, 1, 0, 0, 0, 0],
    [0, 0, 0, 0, 1, 0, 0],
    [0, 0, 0, 1, 0, 0, 0],
    [0, 0, 0, 0, 1, 0, 0],
])

O = np.array([
    [0, 0, 0, 0, 1, 0, 0],
    [0, 0, 0, 0, 0, 1, 0],
    [0, 0, 0, 0, 0, 0, 1],
    [0, 1, 0, 0, 0, -1, 0],
])
```

为了确认 R1CS 构造正确（手工构造时很容易出错），我们创建一个有效见证（witness）并执行矩阵乘法：

```python
x = 4
y = -2
v1 = x * x
v2 = v1 * v1        # x^4
v3 = -5*y * y
z = v3*v1 + v2    # -5y^2 * x^2

# witness
a = np.array([1, z, x, y, v1, v2, v3])

assert all(np.equal(np.matmul(L, a) * np.matmul(R, a), np.matmul(O, a))), "not equal"
```
## Python 中的有限域算术
下一步是把它转换成域数组。直接在 NumPy 中做模运算会很繁琐，而使用 galois 库则很直接。我们在有限域一章介绍过这个库，这里快速回顾其用法：

```python
import galois

GF = galois.GF(79)

a = GF(70)
b = GF(10)

print(a + b)
# prints 1
```

不能向它传入 GF(-1) 这样的负值，否则会抛出异常。要把负数转换成域中的同余表示，可以加上曲线的阶。为了避免正值“溢出”，再对曲线的阶取模。

```python
L = (L + 79) % 79
R = (R + 79) % 79
O = (O + 79) % 79
```

新的矩阵为：
```python
## New values of L, R, O
'''
L

[[ 0  0  1  0  0  0  0]
 [ 0  0  0  0  1  0  0]
 [ 0  0  0 74  0  0  0]
 [ 0  0  0  0  0  0  1]]

R

[[ 0  0  1  0  0  0  0]
 [ 0  0  0  0  1  0  0]
 [ 0  0  0  1  0  0  0]
 [ 0  0  0  0  1  0  0]]

O

[[ 0  0  0  0  1  0  0]
 [ 0  0  0  0  0  1  0]
 [ 0  0  0  0  0  0  1]
 [ 0  1  0  0  0 78  0]]
'''
```

现在只需用 GF 包装这些矩阵，就能将其转换成域数组。见证中也包含负值，因此还需要重新计算见证。

```python
L_galois = GF(L)
R_galois = GF(R)
O_galois = GF(O)

x = GF(4)
y = GF(-2 + 79) # we are using 79 as the field size, so 79 - 2 is -2
v1 = x * x
v2 = v1 * v1         # x^4
v3 = GF(-5 + 79)*y * y
out = v3*v1 + v2    # -5y^2 * x^2

witness = GF(np.array([1, out, x, y, v1, v2, v3]))

assert all(np.equal(np.matmul(L_galois, witness) * np.matmul(R_galois, witness), np.matmul(O_galois, witness))), "not equal"
```

## 有限域中的多项式插值
现在需要把各矩阵的每一列转换成一个 galois 多项式列表，这些多项式分别对相应列进行插值。由于矩阵有 4 行，插值点为 `x = [1,2,3,4]`。

```python
def interpolate_column(col):
    xs = GF(np.array([1,2,3,4]))
    return galois.lagrange_poly(xs, col)

# axis 0 is the columns.
# apply_along_axis is the same as doing a for loop over the columns and collecting the results in an array
U_polys = np.apply_along_axis(interpolate_column, 0, L_galois)
V_polys = np.apply_along_axis(interpolate_column, 0, R_galois)
W_polys = np.apply_along_axis(interpolate_column, 0, O_galois)
```

再次查看矩阵内容可知，`U_polys` 和 `V_polys` 的前两个多项式应为零，`W_polys` 的第一个多项式也应为零。

运行以下合理性检查：

```python
print(U_polys[:2])
print(V_polys[:2])
print(W_polys[:1])

# [Poly(0, GF(79)) Poly(0, GF(79))]# [Poly(0, GF(79)) Poly(0, GF(79))]# [Poly(0, GF(79))]
```

`Poly(0, GF(79))` 表示所有系数均为零的多项式。

建议读者在 R1CS 使用的各个取值处对这些多项式求值，以确认它们确实正确插值了矩阵中的值。

## 计算 h(x)
由于矩阵有四行，我们已经知道 $t(x)$ 是 $(x - 1)(x - 2)(x - 3)(x - 4)$。

回顾一下，二次算术程序的公式如下，其中向量 $\mathbf{a}$ 是见证：

$$
\underbrace{\sum_{i=1}^{m} a_i u_i(x)}_\text{term 1} \underbrace{\sum_{i=1}^m a_i v_i(x)}_\text{term 2} = \underbrace{\sum_{i=1}^{m} a_i w_i(x)}_\text{term 3} + h(x)t(x)
$$

每一项都在计算见证与列插值多项式之间的内积。换言之，每个求和项实际上都是 $[a₁, …, aₘ]$ 与 $[u₁(x), ..., uₘ(x)]$ 的内积。

```python
def inner_product_polynomials_with_witness(polys, witness):
    mul_ = lambda x, y: x * y
    sum_ = lambda x, y: x + y
    return reduce(sum_, map(mul_, polys, witness))

term_1 = inner_product_polynomials_with_witness(U_polys, witness)

term_2 = inner_product_polynomials_with_witness(V_polys, witness)

term_3 = inner_product_polynomials_with_witness(W_polys, witness)
```

要求出 $h(x)$，只需移项求解。注意：除非见证有效，否则除法会产生余数，无法正确得到 $h(x)$。

```python
# t = (x - 1)(x - 2)(x - 3)(x - 4)
t = galois.Poly([1, 78], field = GF) * galois.Poly([1, 77], field = GF) * galois.Poly([1, 76], field = GF) * galois.Poly([1, 75], field = GF)

h = (term_1 * term_2 - term_3) // t
```

与 NumPy 的 [poly1d](https://numpy.org/doc/stable/reference/generated/numpy.poly1d.html) 不同，galois 库不会提示除法是否存在余数，因此需要检查 QAP 公式是否仍然成立。

```python
assert term_1 * term_2 == term_3 + h * t, "division has a remainder"
```

上面的检查与验证者将执行的检查非常相似。

当我们在可信设置生成的隐藏点上对多项式求值时，上述方案将无法直接使用。不过，执行可信设置的计算机仍然需要完成上面的许多计算。

## 总结
本文给出了将 R1CS 转换为 QAP 的 Python 代码。

## 通过 RareSkills 深入学习
本文内容来自我们的[零知识课程](https://www.rareskills.io/zk-bootcamp)。
