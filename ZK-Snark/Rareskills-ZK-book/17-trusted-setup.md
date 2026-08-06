# 可信设置
可信设置（trusted setup）是 ZK-SNARK 用于在秘密值处对多项式求值的一种机制。

注意，多项式 $f(x)$ 可以通过计算其系数与 $x$ 的连续幂之间的内积来求值：

例如，若 $f(x)=3x^3+2x^2+5x+10$，其系数为 $[3,2,5,10]$，可以按如下方式计算：

$$
f(x)=\langle[3,2,5,10],[x^3,x^2,x, 1]\rangle
$$

换言之，我们通常会把上面这个多项式的 $f(2)$ 写成：

$$
f(2)=3(2)^3+2(2)^2+5(2)+10
$$

但也可以这样求值：

$$
f(2)=\langle[3,2,5,10],[8,4,2,1]\rangle = 3\cdot8+2\cdot4+5
\cdot2+10\cdot1
$$

现在假设某人选择一个秘密标量 $\tau$，并计算：

$$
[\tau^3,\tau^2,\tau,1]
$$

然后将其中每个值分别乘以一个密码学椭圆曲线群的生成元点，结果如下：

$$
[\Omega_3, \Omega_2, \Omega_1, G_1]=[\tau^3G_1,\tau^2G_1,\tau G_1,G_1]
$$

此时，任何人都可以使用*结构化参考字符串*（structured reference string，SRS）$[\Omega_3, \Omega_2, \Omega_1, G_1]$，在 $\tau$ 处对次数不超过三次的多项式求值。

例如，给定二次多项式 $g(x)=4x^2+7x+8$，可以计算结构化参考字符串与多项式系数的内积，从而得到 $g(\tau)$：

$$
\langle[0,4,7,8],[\Omega_3, \Omega_2, \Omega_1, G_1]\rangle=4\Omega_2+7\Omega_1+8G_1
$$

这样，我们便在*不知道 $\tau$ 是多少的情况下*计算出了 $g(\tau)$！

这也被称为*可信设置*：虽然*我们*不知道 $g(\tau)$ 的离散对数，但创建结构化参考字符串的人知道。这可能在后续造成信息泄露，因此我们需要*信任*执行可信设置的实体会删除 $\tau$，且不会以任何方式保留它。

## Python 示例

```python
from py_ecc.bn128 import G1, multiply, add
from functools import reduce

def inner_product(points, coeffs):
    return reduce(add, map(multiply, points, coeffs))

## Trusted Setup
tau = 88
degree = 3

# tau^3, tau^2, tau, 1
srs = [multiply(G1, tau**i) for i in range(degree,-1,-1)]

## Evaluate
# p(x) = 4x^2 + 7x + 8
coeffs = [0, 4, 7, 8]

poly_at_tau = inner_product(srs, coeffs)
```

## 验证可信设置是否正确生成

给定一个结构化参考字符串，我们怎么知道它确实遵循 $[x^d, x^{d-1},\dots,x,1]$ 的结构，而不是随意选择出来的？

如果执行可信设置的人还提供 $\Theta=\tau G_2$，我们就可以验证结构化参考字符串是否确实由 $\tau$ 的连续幂构成。

$$
e(\Theta, \Omega_i)\stackrel{?}=e(G_2,\Omega_{i+1})
$$

其中，$e$ 是[双线性配对](https://www.rareskills.io/post/bilinear-pairing)。直观地说，等式左侧计算的是 $\tau\cdot\tau^i$，右侧计算的是 $1\cdot\tau^{i+1}$。

为了验证 $\Theta$ 与 $\Omega_1$ 具有相同的离散对数（$\Omega_1$ 应为 $\tau G_1$），可以检查：

$$
e(\Theta,G_1)\stackrel{?}=e(G_2,\Omega_1)
$$

## 通过多方计算生成结构化参考字符串

假定结构化参考字符串的生成者确实删除了 $\tau$，并不是一个稳妥的信任假设。

下面介绍一种由多方协作创建结构化参考字符串的算法：只要其中至少有一方诚实（即删除了 $\tau$），结构化参考字符串的离散对数就无人知晓。

Alice 生成结构化参考字符串 $([\Omega_n,...,\Omega_2,\Omega_1, G_1],\Theta)$，并将其交给 Bob。

Bob 使用上一节的检查方法验证 SRS 是否“正确”。然后，Bob 选择自己的秘密参数 $\gamma$ 并计算：

$$
([\gamma^n\Omega_n,...,\gamma^2\Omega_2,\gamma\Omega_1,G_1],\gamma\Theta)
$$

注意，此时 SRS 的离散对数为：

$$
([(\tau\gamma)^n,...,(\tau\gamma)^2,(\tau\gamma),1],\tau\gamma)
$$

只要 Alice 或 Bob 中任意一人删除自己的 $\tau$ 或 $\gamma$，最终 SRS 的离散对数就无法恢复。

当然，参与者不必局限于两人，可以有任意数量的参与者。

这种多方计算通常被非正式地称为 *Powers of Tau 仪式*（powers of tau ceremony）。

## 可信设置在 ZK-SNARK 中的作用
在结构化参考字符串上对多项式求值，不会向验证者泄露多项式的信息；证明者也不知道自己具体在哪个点上求值。后面我们会看到，这一方案既有助于防止证明者作弊，也有助于保持见证的零知识性。
