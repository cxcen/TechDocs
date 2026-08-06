# 有限域上的椭圆曲线

有限域中的椭圆曲线是什么样子？

光滑的椭圆曲线很容易可视化，但有限域上的椭圆曲线看起来如何？

下图绘制了 $y² = x³ + 3 \pmod {23}$：

![y² = x³ + 3 \pmod 23 的图](https://static.wixstatic.com/media/935a00_9b8594bdeb9b4eb580847f1d5ffcd6c0~mv2.png/v1/fill/w_614,h_678,al_c,lg_1,q_90,enc_auto/935a00_9b8594bdeb9b4eb580847f1d5ffcd6c0~mv2.png)

因为只允许整数输入——更准确地说，只允许有限域元素——所以不会得到光滑的曲线图。

求解方程时，并非每个 $x$ 值都会产生整数 $y$ 值，因此某些 $x$ 值上不会出现点。上图中可以看到这些空隙。

稍后会提供生成该图的代码。

### 素数会显著改变图形
下面分别是在模 11、23、31 和 41 上绘制的 $y² = x³ + 3$。模数越大，包含的点越多，图形看起来也越复杂。

![模 11、23 的椭圆曲线图](https://static.wixstatic.com/media/935a00_e592040ff7174e81a1f32ed7ed70a150~mv2.png/v1/fill/w_1053,h_565,al_c,q_90,enc_auto/935a00_e592040ff7174e81a1f32ed7ed70a150~mv2.png)

![模 23、31 的椭圆曲线图](https://static.wixstatic.com/media/935a00_382bd8455deb45efba13fdb7d77517b4~mv2.png/v1/fill/w_1053,h_565,al_c,q_90,enc_auto/935a00_382bd8455deb45efba13fdb7d77517b4~mv2.png)

上一篇文章已经说明，椭圆曲线点在“连线并翻转”运算下构成群。在有限域上执行该运算时，它仍然构成群，而且会成为循环群，这对我们的应用极其有用。遗憾的是，解释它为什么是循环群需要非常复杂的数学，所以现在只能暂且接受这一点。不过，这应该不会太令人意外。点的数量是有限的，因此通过执行 $(x + 1)G, (x + 2)G, … (x + \text{阶} - 1)G$ 生成每个点，至少看起来是可行的。

在密码学应用中，$p$ 必须很大。实践中，它会超过 200 位。后文会再次讨论这一点。

## 背景知识
### 域元素
本文会反复使用“域元素”一词。希望通过此前的教程，尤其是[有限域](https://www.rareskills.io/post/finite-fields)一章，你已经熟悉该术语。如果还不熟悉，可以把它理解为取模运算中的非负整数。

也就是说，如果执行模 11 加法，该集合中的有限域元素为 $\set{0,1,…,9,10}$。虽然 Python 示例会使用整数这一数据类型，但把这些对象直接称为“整数”并不准确。对素数取模进行加法时，域元素不能为负数，尽管整数可以为负。有限域中的“负数”只是另一个数的加法逆元，即二者相加得到零的数。例如，在模 11 的有限域中，可以把 4 看作“-7”，因为 $7 + 4 \pmod {11}$ 为 $0$；这类似于普通数中的 7 + (-7) 为零。

在有理数域上，乘法单位元是 1，一个数的逆元只需交换分子与分母。例如，500/303 是 303/500 的逆元，因为二者相乘得到 1。

在有限域中，一个元素的逆元是与它相乘得到有限域元素 1 的数。例如，在模 23 下，6 是 4 的逆元，因为二者相乘再模 23 得到 1。当域的阶为素数时，除零外每个数都有逆元。

### 循环群
循环[群](https://www.rareskills.io/group-theory)是这样一种群：从一个生成元开始反复应用群的二元运算，可以计算出每个元素。

一个非常简单的例子是加法下的模 11 整数。如果生成元为 1，不断把生成元与自身相加，就能生成群中从 0 到 10 的每个元素。

说椭圆曲线点在椭圆曲线加法下构成循环群，意味着可以把有限域中的每个数表示为一个椭圆曲线点，并像对有限域中的普通整数一样把它们相加。

也就是说：

$5 + 7 \pmod p$ 与 $5G + 7G$ 同态。

其中 G 是椭圆曲线循环群的生成元。

只有点数为素数的有限域椭圆曲线才具有这一性质，而实践中使用的正是这类曲线。稍后会再次讨论这一点。

## BN128 公式
以太坊预编译合约使用 BN128 曲线验证 ZK 证明，其定义如下：

$$
p = 21888242871839275222246405745257275088696311157297823662689037894645226208583
$$

$$
y² = x³ + 3 \pmod p
$$

这里，$p$ 是域模数。

不要把 `field_modulus` 与曲线的阶混淆；曲线的阶是曲线上的点数。

对 bn128 曲线，其曲线的阶如下：

```python
from py_ecc.bn128 import curve_order
# 21888242871839275222246405745257275088548364400416034343698204186575808495617
print(curve_order)
```

域模数非常大，直接用它做实验很不方便。下一节会使用相同公式和更小的模数，逐步建立对有限域中椭圆曲线点的直觉。

## 生成椭圆曲线循环群 $y² = x³ + 3 \pmod {11}$
要解上面的方程并确定哪些 $(x, y)$ 点位于曲线上，需要计算 $\sqrt{(x³ + 3)} \pmod {11}$。

### 模平方根
我们使用 [Tonelli–Shanks 算法](https://en.wikipedia.org/wiki/Tonelli%E2%80%93Shanks_algorithm)计算模平方根。如果感兴趣，可以自行了解算法原理；现在只需把它当作黑盒：它能在给定模数下计算域元素的数学平方根，或者告诉你平方根不存在。

例如，5 模 11 的平方根是 4 $(4 \times 4 \mod 11 = 5)$，但 6 模 11 没有平方根。（建议读者通过暴力枚举验证这一点。）

平方根通常有两个解，一个为正，一个为负。虽然有限域中没有带负号的数，但在逆元意义下仍然有“负数”的概念。

可以在网上找到上述算法的实现代码；为避免在教程中放入大段代码，本文改为安装一个 Python 库。


```bash
python3 -m pip install libnum
```

安装 [libnum](https://pypi.org/project/libnum/) 后，可以运行以下代码演示其用法。

```python
from libnum import has_sqrtmod_prime_power, sqrtmod_prime_power

# the functions take arguments# has_sqrtmod_prime_power(n, field_mod, k), where n**k,
# but we aren't interested in powers in modular fields, so we set k = 1
# check if sqrt(8) mod 11 exists
print(has_sqrtmod_prime_power(8, 11, 1))
# False

# check if sqrt(5) mod 11 exists
print(has_sqrtmod_prime_power(5, 11, 1))
# True

# compute sqrt(5) mod 11
print(list(sqrtmod_prime_power(5, 11, 1)))
# [4, 7]

assert (4 ** 2) % 11 == 5
assert (7 ** 2) % 11 == 5

# we expect 4 and 7 to be additive inverses of each other, because in "regular" math, the two solutions to a square root are sqrt and -sqrt
assert (4 + 7) % 11 == 0
```

现在已经知道如何计算模平方根，可以遍历 $x$ 的取值，并根据公式 $y² = x³ + b$ 计算 $y$。求解 $y$ 只需对等式两侧取模平方根（如果存在），并保存得到的 $(x, y)$ 有序对，以便稍后绘图。

下面绘制一个简单的椭圆曲线：

$$
y² = x³ + 3 \pmod {11}
$$

```python
import libnum
import matplotlib.pyplot as plt

def generate_points(mod):
    xs = []
    ys = []
    def y_squared(x):
        return (x**3 + 3) % mod

    for x in range(0, mod):
        if libnum.has_sqrtmod_prime_power(y_squared(x), mod, 1):
            square_roots = libnum.sqrtmod_prime_power(y_squared(x), mod, 1)

            # we might have two solutions
            for sr in square_roots:
                ys.append(sr)
                xs.append(x)
    return xs, ys


xs, ys = generate_points(11)
fig, (ax1) = plt.subplots(1, 1);
fig.suptitle('y^2 = x^3 + 3 (mod p)');
fig.set_size_inches(6, 6);
ax1.set_xticks(range(0,11));
ax1.set_yticks(range(0,11));
plt.grid()
plt.scatter(xs, ys)
plt.plot();
```

绘图结果如下：

![y² = x³ + 3（mod 11）的图](https://static.wixstatic.com/media/935a00_2355ae79d450498eb3ee5b6721634b43~mv2.png/v1/fill/w_614,h_678,al_c,lg_1,q_90,enc_auto/935a00_2355ae79d450498eb3ee5b6721634b43~mv2.png)

可以观察到：

- 不会出现大于或等于所用模数的 x 或 y 值；
- 与实数值曲线一样，模运算下的图形“看起来对称”。

## 椭圆曲线点加法
更有趣的是，用于计算椭圆曲线点的“连线并翻转”运算仍然有效！


不过，考虑到我们是在有限域上执行运算，这并不奇怪。实数上的公式使用普通域运算，即加法和乘法。虽然判断一个点是否位于曲线上时使用平方根，而平方根并不是有效的域运算，但计算点加法和倍点时并不使用平方根。


读者可以从上图中任取两个点，代入下面的代码进行点加法，验证结果总会落在另一个点上；如果二者互为逆点，则结果为无穷远点。这些公式来自 Wikipedia 的[椭圆曲线标量乘法](https://en.wikipedia.org/wiki/Elliptic_curve_point_multiplication)页面。

```python
def double(x, y, a, p):
    lambd = (((3 * x**2 + a) % p ) *  pow(2 * y, -1, p)) % p
    newx = (lambd**2 - 2 * x) % p
    newy = (-lambd * newx + lambd * x - y) % p
    return (newx, newy)

def add_points(xq, yq, xp, yp, p, a=0):
    if xq == yq == None:
        return xp, yp
    if xp == yp == None:
        return xq, yq

    assert (xq**3 + 3) % p == (yq ** 2) % p, "q not on curve"
    assert (xp**3 + 3) % p == (yp ** 2) % p, "p not on curve"

    if xq == xp and yq == yp:
        return double(xq, yq, a, p)
    elif xq == xp:
        return None, None

    lambd = ((yq - yp) * pow((xq - xp), -1, p) ) % p
    xr = (lambd**2 - xp - xq) % p
    yr = (lambd*(xp - xr) - yp) % p
    return xr, yr
```

下面的可视化展示了有限域中的“连线并翻转”：

![有限域中的椭圆曲线加法示例](https://r2media.rareskills.io/elliptic-curve-finite-fields-example.jpg)

## 循环群中的每个椭圆曲线点都有一个“编号”
按照定义，循环群可以通过不断把生成元与自身相加来生成。

使用一个实际例子：$y² = x³ + 3 \pmod {11}$，生成元点为 $(4, 10)$。

利用上面的 Python 函数，可以从点 $(4, 10)$ 开始生成群中的每个点：

```python
# for our purposes, (4, 10) is the generator point G
next_x, next_y = 4, 10
print(0, 4, 10)
points = [(next_x, next_y)]
for i in range(1, 13):
    # repeatedly add G to the next point to generate all the elements
    next_x, next_y = add_points(next_x, next_y, 4, 10, 11)
    print(i, next_x, next_y)
    points.append((next_x, next_y))
```

输出为：
```bash
0 4 10
1 7 7
2 1 9
3 0 6
4 8 8
5 2 0
6 8 3
7 0 5
8 1 2
9 7 4
10 4 1
11 None None
12 4 10 # note that this is the same point as the first one
```
请注意，$(\text{阶} + 1)G = G$。与模加法一样，当发生“上溢”时，循环会重新开始。

这里，`None` 表示无穷远点，它确实属于该群。把无穷远点与生成元相加会返回生成元，这正是单位元应有的行为。

可以根据为了到达某个点而把生成元与自身相加的次数，为每个点指定一个“编号”。

可以使用以下代码绘制曲线，并在每个点旁标出编号：

```python
xs11, ys11 = generate_points(11)

fig, (ax1) = plt.subplots(1, 1);
fig.suptitle('y^2 = x^3 + 3 (mod 11)');
fig.set_size_inches(13, 6);

ax1.set_title("modulo 11")
ax1.scatter(xs11, ys11, marker='o');
ax1.set_xticks(range(0,11));
ax1.set_yticks(range(0,11));
ax1.grid()

for i in range(0, 11):
    plt.annotate(str(i+1), (points[i][0] + 0.1, points[i][1]), color="red");
```

红色文字可以理解为：从单位元开始，需要把生成元加上多少次才能得到该点。

![带点编号的 y^2 = x^3 + 3（mod 11）图](https://static.wixstatic.com/media/935a00_3e9b90d38e9c4c4a82b34d138fa9f49c~mv2.png/v1/fill/w_1063,h_565,al_c,q_90,enc_auto/935a00_3e9b90d38e9c4c4a82b34d138fa9f49c~mv2.png)

### 逆点仍然关于竖直方向对称
有一个有趣的观察：具有相同 x 值的点，其编号之和为 12，对应单位元 $(12 \mod 12 = 0)$。如果把图中编号 11 的点 $(4, 1)$ 与 $(4, 10)$ 相加，会得到无穷远点，也就是群中的第 12 个元素。

### 群的阶不是模数
在这个例子中，群的阶为 12，即群中椭圆曲线点的总数，尽管椭圆曲线公式使用模 11。本文会反复强调：绝对不要假设椭圆曲线的模数就是群的阶。不过，可以使用 [Hasse 定理](https://en.wikipedia.org/wiki/Hasse%27s_theorem_on_elliptic_curves)，根据域模数估计曲线阶的范围。

### 如果点数为素数，点加法的行为就类似有限域

上图中有 12 个点（包括 0）。模 12 加法不构成有限域，因为 12 不是素数。

不过，如果仔细选择曲线参数，就能创建一条点与有限域元素相对应的椭圆曲线，也就是让曲线的阶等于有限域的阶。

例如，$y^2 = x^3 + 7 \pmod {43}$ 会创建一条总共有 31 个点的曲线，如下图所示：

![包含 31 个点的椭圆曲线](https://r2media.rareskills.io/elliptic-curve-finitie-order-31.jpg)

当曲线的阶与有限域的阶相同时，**在有限域中执行的每一种运算，在椭圆曲线上都有对应的同态运算。**

要从有限域映射到椭圆曲线，可以任意选取一个点作为生成元，再用有限域中的元素对生成元执行标量乘法。

## 乘法其实是重复加法
不存在椭圆曲线点之间的乘法。所谓“标量乘法”，实际是重复加法。不能把两个椭圆曲线点直接相乘（严格来说，使用[双线性配对](https://www.rareskills.io/post/bilinear-pairing)某种程度上可以做到，但稍后才会讨论）。

使用 Python 库执行 `multiply(G1, x)`，实际等价于把 `G1 + G1 + … + G1` 相加 `x` 次。内部并不会真的逐次自加，而是使用倍点等快捷方法，在对数时间内完成运算。

例如，要计算 135G，可以利用倍点高效计算并缓存以下值：

```
G, 2G, 4G, 8G, 16G, 32G, 64G, 128G
```
……然后计算 128G + 4G + 2G + G = 135G。

说 `5G + 6G = 11G`，本质上只是把 G 与自身相加 11 次。借助上面的快捷方法，可以用对数数量的计算得到 11G；但归根结底，它只是重复加法。

## Python bn128 库
EVM 实现 pyEVM 用于椭圆曲线预编译合约的库是 `py_ecc`，后文会大量使用它。下面的代码展示生成元点的样子，以及一些加法和标量乘法。

G1 点如下：

```python
from py_ecc.bn128 import G1, multiply, add, eq, neg

print(G1)
# (1, 2)

print(add(G1, G1))
# (1368015179489954701390400359078579693043519447331113978918064868415326638035, 9918110051302171585080402603319702774565515993150576347155970296011118125764)

print(multiply(G1, 2))
#(1368015179489954701390400359078579693043519447331113978918064868415326638035, 9918110051302171585080402603319702774565515993150576347155970296011118125764)

# 10G + 11G = 21G
assert eq(add(multiply(G1, 10), multiply(G1, 11)), multiply(G1, 21))
```

虽然这些数字很大，难以阅读，但可以看到，把一个点与自身相加，与用 2 对该点执行“乘法”得到相同的值。上面的两个点显然相同。这个元组仍是 $(x, y)$ 有序对，只是定义域非常大。

上面打印出的数如此巨大是有原因的。我们不希望攻击者取得一个椭圆曲线点后，计算出生成它的域元素。如果循环群的阶太小，攻击者只需暴力枚举即可。

下面绘制了前 1000 个点：

![bn128 的前 1000 个点](https://static.wixstatic.com/media/935a00_d9bd567f47c247f588061305dc97940e~mv2.png/v1/fill/w_656,h_514,al_c,lg_1,q_85,enc_auto/935a00_d9bd567f47c247f588061305dc97940e~mv2.png)

生成上图的代码如下：

```python
import matplotlib.pyplot as plt
from py_ecc.bn128 import G1, multiply, neg
import math
import numpy as np
xs = []
ys = []
for i in range(1,1000):
    xs.append(i)
    ys.append(int(multiply(G1, i)[1]))
    xs.append(i)
    ys.append(int(neg(multiply(G1, i))[1]))
plt.scatter(xs, ys, marker='.')
```

它可能看起来很吓人，但与上一节所做工作的唯一区别，是使用了大得多的模数和不同的生成元点。

### 库中的加法
`py_ecc` 库让点加法变得很方便，语法应该不言自明：

```python
from py_ecc.bn128 import G1, multiply, add, eq

# 5 = 2 + 3
assert eq(multiply(G1, 5), add(multiply(G1, 2), multiply(G1, 3)));
```

有限域中的加法与椭圆曲线点之间的加法同态（当二者的阶相等时）。由于离散对数问题，另一方可以在不知道哪些域元素生成了这些点的情况下，把椭圆曲线点相加。

到这里，希望读者已经从理论和实践两个方面对椭圆曲线点加法建立了良好直觉，因为现代零知识算法*高度*依赖这一运算。

### 模加法与椭圆曲线加法之同态的实现细节
这里必须仔细区分几个术语：

**域模数**是定义曲线时使用的模数。**曲线的阶**是曲线上的点数。

如果从点 $R$ 开始，加上曲线的阶 $o$，会重新得到 $R$。如果加上域模数，则会得到不同的点。

```python
from py_ecc.bn128 import curve_order, field_modulus, G1, multiply, eq

x = 5 # chosen randomly
# This passes
assert eq(multiply(G1, x), multiply(G1, x + curve_order))

# This fails
assert eq(multiply(G1, x), multiply(G1, x + field_modulus))
```

这意味着 `(x + y) mod curve_order == xG + yG`。

```python
x = 2 ** 300 + 21
y = 3 ** 50 + 11

# (x + y) == xG + yG
assert eq(multiply(G1, (x + y)), add(multiply(G1, x), multiply(G1, y)))
```

尽管 `x + y` 运算显然会“上溢”曲线的阶，但这没有关系。与有限域一样，这正是预期行为。椭圆曲线标量乘法会隐式执行与先取模再乘法相同的运算。

事实上，如果只关心正数，甚至无需显式取模，以下恒等式同样成立：

```python
x = 2 ** 300 + 21
y = 3 ** 50 + 11

assert eq(multiply(G1, (x + y) % curve_order), add(multiply(G1, x), multiply(G1, y)))
```

但是，如果有限域数学使用错误的模数——曲线阶以外的某个数——那么一旦“上溢”，等式就会被破坏：

```python
x = 2 ** 300 + 21
y = 3 ** 50 + 11 # these values are large enough to overflow:

assert eq(multiply(G1, (x + y) % (curve_order - 1)), add(multiply(G1, x), multiply(G1, y))), "this breaks"
```

#### 编码有理数
取模后，就可以编码除法概念。

例如，普通整数无法执行下面的运算：

```python
# this throws an exception
eq(add(multiply(G1, 5 / 2), multiply(G1, 1 / 2), multiply(G1, 3)
```
但在有限域中，1/2 可以有意义地计算为 2 的乘法逆元。因此，5 / 2 可以编码为 $5 \cdot \mathsf{inv}(2)$。

在 Python 中可以这样实现：

```python
five_over_two = (5 * pow(2, -1, curve_order)) % curve_order
one_half = pow(2, -1, curve_order)

# Essentially 5/2 = 2.5# 2.5 + 0.5 = 3
# but we are doing this in a finite field
assert eq(add(multiply(G1, five_over_two), multiply(G1, one_half)), multiply(G1, 3))
```

#### 结合律
我们知道群满足结合律，因此一般情况下应有以下恒等式：
```python
x = 5
y = 10
z = 15

lhs = add(add(multiply(G1, x), multiply(G1, y)), multiply(G1, z))

rhs = add(multiply(G1, x), add(multiply(G1, y), multiply(G1, z)))

assert eq(lhs, rhs)
```

建议读者自行尝试不同的 `x`、`y` 和 `z` 值。

#### 每个元素都有逆元
`py_ecc` 库提供 `neg` 函数，它会在有限域中把给定元素关于 x 轴翻转，从而得到其逆元。该库使用 Python 的 `None` 编码“无穷远点”。

```python
from py_ecc.bn128 import G1, multiply, neg, is_inf, Z1

# pick a field element
x = 12345678
# generate the point
p = multiply(G1, x)

# invert
p_inv = neg(p)

# every element added to its inverse produces the identity element assert is_inf(add(p, p_inv))

# Z1 is just None, which is the point at infinity
assert Z1 is None

# special case: the inverse of the identity is itself
assert eq(neg(Z1), Z1)
```

与实数上的椭圆曲线一样，一个椭圆曲线点的逆点具有相同的 x 值，而 y 值互为逆元。

```python
from py_ecc.bn128 import G1, neg, multiply

field_modulus = 21888242871839275222246405745257275088696311157297823662689037894645226208583
for i in range(1, 4):
    point = multiply(G1, i)
    print(point)
    print(neg(point))
    print('----')

    # x values are the same
    assert int(point[0]) == int(neg(point)[0])

    # y values are inverses of each other, we are adding y values
    # not ec points
    assert int(point[1]) + int(neg(point)[1]) == field_modulus
```
##### 每个元素都可以由生成元生成
当点数超过 $2^{200}$ 时，无法通过暴力枚举验证这一点。不过，请考虑 `eq(multiply(G1, x), multiply(G1, x + order))` 始终为真这一事实。这意味着，我们可以生成多达阶所规定数量的点，然后循环回到起点。

#### `optimized_bn128` 又是什么？
查看该库时，会看到一个名为 optimized_bn128 的实现。对执行时间进行基准测试，会发现这个版本运行快得多，pyEVM 使用的也是该实现。不过，出于教学目的，更适合使用未优化版本，因为它以更直观的方式组织点，也就是通常的 x、y 元组。优化版本则把椭圆曲线点组织为三元组，更难解释。

```python
from py_ecc.optimized_bn128 import G1, multiply, neg, is_inf, Z1
print(G1)
# (1, 2, 1)
```

## 使用椭圆曲线的基础零知识证明
考虑下面这个相当简单的例子：

声明：“我知道两个值 $x$ 和 $y$，满足 $x + y = 15$。”

证明：我用 `x` 对 `G1` 执行标量乘法，再用 `y` 对 `G1` 执行标量乘法，把得到的点作为 `A` 和 `B` 交给你。

验证者：你用 15 对 G1 执行标量乘法，并检查 `A + B == 15G1`。

对应的 Python 代码如下：
```python
from py_ecc.bn128 import G1, multiply, add

# Prover
secret_x = 5
secret_y = 10

x = multiply(G1, 5)
y = multiply(G1, 10)

proof = (x, y, 15)

# verifier
if multiply(G1, proof[2]) == add(proof[0], proof[1]):
    print("statement is true")
else:
    print("statement is false")
```

虽然验证者不知道 `x` 和 `y` 的值，但可以验证 `x` 和 `y` 在椭圆曲线空间中相加得到 15，因此 `secret_x` 和 `secret_y` 作为有限域元素相加也得到 15。

留给读者一个练习：完成更复杂的证明，例如证明自己知道线性方程组的一个解。

一个非常重要的提示是：一个数乘以常量等价于重复加法，而重复加法等价于椭圆曲线标量乘法。因此，如果 x 是椭圆曲线点，可以用 `multiply(x, 9)` 将其乘以标量 9。这与“不能把椭圆曲线点相乘”的说法一致——这里实际上是用标量乘以椭圆曲线点，而不是让它与另一个点相乘。

你能证明自己知道满足 $23x = 161$ 的 $x$ 吗？能把它推广到更多变量吗？

另一个提示：你（证明者）和验证者需要事先就公式达成一致，因为验证者会运行你所声称知道其解的原公式之相同“结构”。

### 安全假设
为了让上述方案安全，我们假设：如果公开一个如 `multiply(G1, x)` 的点，攻击者无法根据所生成的 $(x, y)$ 值推断原始的 $x$。这就是离散对数假设。计算公式所用的素数必须很大，原因也在于此，这样攻击者便无法暴力猜测。

还有一些比暴力搜索更高效的算法，例如 [baby-step giant-step](https://en.wikipedia.org/wiki/Baby-step_giant-step) 算法。

注意：BN128 这一名称来自它具有 128 位安全性的假设。椭圆曲线在一个 254 位有限域中计算，但由于存在比朴素暴力搜索更好的离散对数算法，人们认为它具有 128 位安全性。

### 真正的零知识
还需要指出，`A + B = 15G` 示例并不是真正的零知识。如果攻击者猜出 `a` 和 `b`，就可以把生成的椭圆曲线点与我们的点进行比较，从而验证猜测。

后续章节会解决这个问题。

## 把有限域上的椭圆曲线视为魔法黑盒
正如使用哈希函数时无需知道其内部原理一样，使用椭圆曲线点加法和标量乘法时，也无需了解实现细节。

不过，必须知道它们遵循的规则。即使这时听起来像在重复播放同一张唱片，它们仍然遵循循环群的规则：
- 椭圆曲线点加法封闭：结果是另一个椭圆曲线点；
- 椭圆曲线点加法满足结合律；
- 存在单位元；
- 每个元素都有一个逆元，与之相加会得到单位元。

只要理解这些规则，就可以尽情执行加法、标量乘法和求逆，而不会进行无效操作。`py_ecc` 库为每项运算都提供了对应函数。

这是本课最重要的结论：

**有限域上的椭圆曲线对有限域中的加法进行同态加密。**

### 登月数学：如何知道曲线的阶？
读者可能会疑惑：既然无法数出公式的全部有效解，我们如何确定 bn128 曲线的阶？有效点的数量多到任何计算机都无法枚举，那么曲线阶是如何得到的？

这正是我们试图避开的数学类型，因为它相当高级。事实证明，可以使用 [Schoof 算法](https://en.wikipedia.org/wiki/Schoof%27s_algorithm)在多项式时间内计算点数。本文不要求你理解该算法如何工作，只需知道它存在。从实现角度看，曲线阶如何推导并不重要；只需相信设计者正确计算了它。

RareSkills 的这些材料经过精心设计，尽量避开此类数学雷区。

## 通过 RareSkills 继续学习
这就是我们的[零知识课程](https://www.rareskills.io/zk-bootcamp)如此重视抽象代数基础的原因。理解椭圆曲线的实现细节难如噩梦；但理解循环群的行为，虽然起初有些陌生，却完全在大多数程序员的理解范围内。一旦掌握这些知识，即使点加法难以可视化，其整体行为也会变得直观。

*最初发表于 2023 年 9 月 19 日*
