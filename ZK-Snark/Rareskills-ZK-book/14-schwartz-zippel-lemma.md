# Schwartz–Zippel 引理及其在零知识证明中的应用

几乎所有 ZK 证明算法都依赖 Schwartz–Zippel 引理实现简洁性。

Schwartz–Zippel 引理指出：给定两个多项式 $p(x)$ 和 $q(x)$，其次数分别为 $d_p$ 和 $d_q$；如果 $p(x) \neq q(x)$，那么 $p(x)$ 与 $q(x)$ 的交点数量不超过 $\mathsf{max}(d_p, d_q)$。

下面观察几个例子。

## 多项式示例与 Schwartz–Zippel 引理
### 直线与抛物线相交

考虑多项式 $p(x) = x$ 和 $q(x) = x^2$。它们在 $x = 0$ 和 $x = 1$ 处相交。
![y = x 与 y = x^2 的图](https://r2media.rareskills.io/schwartz-zippel-x-x2-example.png)

它们有两个交点，恰好等于多项式 $y = x$ 与 $y = x^2$ 的最大次数。

### 三次多项式与一次多项式

考虑多项式 $p(x) = x^3$ 和 $q(x) = x$。它们在 $x = -1$、$x = 0$、$x = 1$ 处相交，除此之外没有其他交点。交点数量受两个多项式的最大次数约束，本例中该次数为 3。

![y = x^3 与 y = x 的图](https://r2media.rareskills.io/schwartz-zippel-x-x3-example.png)


## 有限域中的多项式与 Schwartz–Zippel 引理
Schwartz–Zippel 引理同样适用于[有限域](https://www.rareskills.io/post/finite-fields)中的多项式，即所有计算都对素数 $p$ 取模。

## 多项式相等性测试
可以通过检查两个多项式的所有系数是否相等来判断它们是否相等，但这需要 $\mathcal{O}(d)$ 时间，其中 $d$ 是多项式次数。

也可以在随机点 $u$ 处对两个多项式求值，再用 $\mathcal{O}(1)$ 时间比较求值结果。

具体而言，在有限域 $\mathbb{F}_{p}$ 中，从 $[0,p)$ 随机选取 $u$，然后计算 $y_f=f(u)$ 和 $y_g=g(u)$。如果 $y_f = y_g$，必然存在以下两种情况之一：

1. $f(x) = g(x)$；
2. $f(x) \neq g(x)$，而我们恰好选中了它们相交的 $d$ 个点之一，其中 $d = \mathsf{max}(\deg(f), \deg(g))$。

如果 $d \ll p$，第二种情况发生的概率极低，可以忽略不计。

例如，假设域 $\mathbb{F}_{p}$ 中的 $p \approx 2^{254}$，略小于一个 [uint256](https://www.rareskills.io/post/uint-max-value-solidity)；如果多项式次数不超过一百万，那么选中交点的概率为：

$$
\frac{1\times 10^6}{2^{254}} \approx \frac{2^{20}}{2^{254}} \approx \frac{1}{2^{234}} \approx \frac{1}{10^{70}}
$$

为了直观理解这一数量级，宇宙中的原子数大约为 $10^{78}$ 到 $10^{82}$。因此，如果两个多项式不相等，随机选中其交点的概率极低。

## 使用 Schwartz–Zippel 引理测试两个向量是否相等

可以结合拉格朗日插值与 Schwartz–Zippel 引理，测试两个向量是否相等。

通常需要逐一比较两个向量的 $n$ 个分量。

另一种方法是使用一组相同的 $x$ 值（例如 $[1,2,..,n]$）对向量进行插值：

1. 分别为两个向量插值得到多项式 $f(x)$ 和 $g(x)$；
2. 随机选取一点 $u$；
3. 在 $u$ 处对多项式求值；
4. 检查 $f(u) = g(u)$ 是否成立。

虽然计算多项式需要更多工作，但最终检查的成本低得多。

下面是用 Python 执行该计算的示例：

```python
import galois
import numpy as np

p = 103
GF = galois.GF(p)

xs = GF(np.array([1,2,3]))

# arbitrary vectors
v1 =  GF(np.array([4,8,19]))
v2 =  GF(np.array([4,8,19]))


def L(v):
    return galois.lagrange_poly(xs, v)

p1 = L(v1)
p2 = L(v2)

import random
u = random.randint(0, p)

lhs = p1(u)
rhs = p2(u)

# only one check required
assert lhs == rhs
```

## 在 ZK 证明中使用 Schwartz–Zippel 引理
最终目标是让证明者向验证者发送一小段数据，并让验证者快速检查它。

大多数时候，ZK 证明本质上就是在一个随机点处求值后的多项式。

需要解决的难题是：无法确定多项式是否被*诚实地*求值——必须通过某种方式确保，证明者在计算 $f(u)$ 时没有撒谎。

不过在此之前，需要先学习如何把整个算术电路表示为一小组在随机点处求值的多项式，这正是学习[二次算术程序](https://www.rareskills.io/post/quadratic-arithmetic-program)的动机。
