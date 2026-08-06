# 在可信设置上对二次算术程序求值

在可信设置（trusted setup）上对[二次算术程序（Quadratic Arithmetic Program，QAP）](https://www.rareskills.io/post/quadratic-arithmetic-program)求值，可以让证明者用恒定大小的证明表明 QAP 等式成立，同时不泄露见证（witness）。

具体而言，需要在一个未知点 $\tau$ 处对 QAP 多项式求值。QAP 等式

$$
\sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x) = \sum_{i=1}^m a_iw_i(x) + h(x)t(x)
$$

在向量 $\mathbf{a}$ 满足等式时成立；否则，它将以压倒性概率不成立。

这里展示的方案并不是安全的零知识证明，但它是理解 Groth16 工作原理的一个过渡步骤。

## 具体示例
为了让问题不那么抽象，假设[秩一约束系统（Rank-1 Constraint System，R1CS）](https://www.rareskills.io/post/rank-1-constraint-system)的矩阵 $\mathbf{L}$、$\mathbf{R}$ 和 $\mathbf{O}$ 都有 3 行 4 列。

$$
\mathbf{L}\mathbf{a} \circ \mathbf{R}\mathbf{a} = \mathbf{O}\mathbf{a}
$$

因为有 3 行，所以插值多项式的次数为 2。又因为有 4 列，每个矩阵会产生 4 个多项式，总计 12 个多项式。

对应的 QAP 为：

$$
\sum_{i=1}^4a_iu_i(x)\sum_{i=1}^4a_iv_i(x) = \sum_{i=1}^4a_iw_i(x) + h(x)t(x)
$$

## 记号与预备知识
我们将群 $\mathbb{G}_1$ 和 $\mathbb{G}_2$ 中的椭圆曲线生成元点分别记为 $G_1$ 和 $G_2$。$\mathbb{G}_1$ 中的元素记作 $[X]_1$，$\mathbb{G}_2$ 中的元素记作 $[X]_2$。当下标可能与列表索引混淆时，则写作 $X \in \mathbb{G}_1$ 或 $X \in \mathbb{G}_2$。两个点之间的[椭圆曲线配对](https://www.rareskills.io/post/bilinear-pairing)记作 $[X]_1 \bullet [Y]_2$。

令 $\mathbf{L}_{(*,j)}$ 表示 $\mathbf{L}$ 的第 $j$ 列。在本例中，行对应 $(1,2,3)$，列对应 $(1,2,3,4)$。令 $\mathcal{L}(\mathbf{L}_{(*,j)})$ 表示对 $\mathbf{L}$ 的第 $j$ 列执行拉格朗日插值得到的多项式，其中 $x$ 值为 $(1,2,3)$，$y$ 值取自第 $j$ 列。

由于有 4 列，从 $\mathbf{L}$ 得到四个多项式：

$$
\begin{align*}
u_1(x) = \mathcal{L}(\mathbf{L}_{(*,1)}) =u_{12}x^2 + u_{11}x+u_{10}\\
u_2(x) = \mathcal{L}(\mathbf{L}_{(*,2)}) =u_{22}x^2 + u_{21}x+u_{20}\\
u_3(x) = \mathcal{L}(\mathbf{L}_{(*,3)}) =u_{32}x^2 + u_{31}x+u_{30}\\
u_4(x) = \mathcal{L}(\mathbf{L}_{(*,4)}) =u_{42}x^2 + u_{41}x+u_{40}\\
\end{align*}
$$

从 $\mathbf{R}$ 得到四个多项式：

$$
\begin{align*}
v_1(x) = \mathcal{L}(\mathbf{R}_{(*,1)}) =v_{12}x^2 + v_{11}x+v_{10}\\
v_2(x) = \mathcal{L}(\mathbf{R}_{(*,2)}) =v_{22}x^2 + v_{21}x+v_{20}\\
v_3(x) = \mathcal{L}(\mathbf{R}_{(*,3)}) =v_{32}x^2 + v_{31}x+v_{30}\\
v_4(x) = \mathcal{L}(\mathbf{R}_{(*,4)}) =v_{42}x^2 + v_{41}x+v_{40}\\
\end{align*}
$$

从 $\mathbf{O}$ 得到四个多项式：

$$
\begin{align*}
w_1(x) = \mathcal{L}(\mathbf{O}_{(*,1)}) =w_{12}x^2 + w_{11}x+w_{10}\\
w_2(x) = \mathcal{L}(\mathbf{O}_{(*,2)}) =w_{22}x^2 + w_{21}x+w_{20}\\
w_3(x) = \mathcal{L}(\mathbf{O}_{(*,3)}) =w_{32}x^2 + w_{31}x+w_{30}\\
w_4(x) = \mathcal{L}(\mathbf{O}_{(*,4)}) =w_{42}x^2 + w_{41}x+w_{40}\\
\end{align*}
$$

多项式记号 $p_{ij}$ 表示第 $i$ 个多项式的第 $j$ 个系数（幂次）。例如，$j=2$ 表示与 $x^2$ 对应的系数。

本例的 QAP 为：

$$
\sum_{i=1}^4a_iu_i(x)\sum_{i=1}^4a_iv_i(x) = \sum_{i=1}^4a_iw_i(x) + h(x)t(x)
$$
其中 $t(x) = (x - 1)(x - 2)(x - 3)$，而 $h(x)$ 为：

$$
h(x)=\frac{\sum_{i=1}^4a_iu_i(x)\sum_{i=1}^4a_iv_i(x) - \sum_{i=1}^4a_iw_i(x)}{t(x)}
$$

### QAP 中各多项式的次数与 R1CS 大小之间的关系
对于一般情形下的多项式次数，可以作出以下观察：
- $u(x)$ 和 $v(x)$ 的次数最高可达 $n - 1$，因为它们对 $n$ 个点插值，而 $n$ 是 R1CS 的行数。
- 如果多项式之和 $\sum_{i=0}^m a_iw_i(x)$ 等于零多项式，即各系数相加后彼此抵消，那么 $w(x)$ 的次数最低可以为 0。
- 根据定义，$t(x)$ 的次数为 $n$。
- 多项式相乘时次数相加，多项式相除时次数相减。

因此，h(x) 的次数最高为 $n - 2$，因为：

$$
\underbrace{n - 1}_{
\deg{u(x)}} + \underbrace{n - 1}_{\deg{v(x)}} - \underbrace{n}_{\deg{t(x)}} = n - 2
$$

## 展开各项

将前面示例中的求和展开，可得：

$$
\begin{align*}
\sum_{i=1}^4 a_iu_i(x) &= a_1(u_{12}x^2 + u_{11}x+u_{10}) + a_2(u_{22}x^2 + u_{21}x+u_{20}) + a_3(u_{32}x^2 + u_{31}x+u_{30}) + a_4(u_{42}x^2 + u_{41}x+u_{40})\\
&= (a_1u_{12}+a_2u_{22}+a_3u_{32}+a_4u_{42})x^2 + (a_1u_{11}+a_2u_{21}+a_3u_{31}+a_4u_{41})x + (a_1u_{10}+a_2u_{20}+a_3u_{30}+a_4u_{40})\\
&=u_{2a}x^2+u_{1a}x+u_{0a}\\
\sum_{i=1}^4 a_iv_i(x) &= a_1(v_{12}x^2 + v_{11}x+v_{10}) + a_2(v_{22}x^2 + v_{21}x+v_{20}) + a_3(v_{32}x^2 + v_{31}x+v_{30}) + a_4(v_{42}x^2 + v_{41}x+v_{40})\\
&= (a_1v_{12}+a_2v_{22}+a_3v_{32}+a_4v_{42})x^2 + (a_1v_{11}+a_2v_{21}+a_3v_{31}+a_4v_{41})x + (a_1v_{10}+a_2v_{20}+a_3v_{30}+a_4v_{40})\\
&=v_{2a}x^2+v_{1a}x+v_{0a}\\
\sum_{i=1}^4 a_iw_i(x) &= a_1(w_{12}x^2 + w_{11}x+w_{10}) + a_2(w_{22}x^2 + w_{21}x+w_{20}) + a_3(w_{32}x^2 + w_{31}x+w_{30}) + a_4(w_{42}x^2 + w_{41}x+w_{40})\\
&= (a_1w_{12}+a_2w_{22}+a_3w_{32}+a_4w_{42})x^2 + (a_1w_{11}+a_2w_{21}+a_3w_{31}+a_4w_{41})x + (a_1w_{10}+a_2w_{20}+a_3w_{30}+a_4w_{40})\\
&=w_{2a}x^2+w_{1a}x+w_{0a}\\
\end{align*}
$$

在每种情形下，我们都将 4 个二次多项式相加，因此结果仍是一个二次多项式。

一般表达式 $\sum_{i=1}^m a_ip_i(x)$ 所产生多项式的最高幂次不会超过 $p(x)$（也可能更低，例如 $(a_1w_{12}+a_2w_{22}+a_3w_{32}+a_4w_{42})x^2$ 的和为 0 时）。为方便起见，这里引入系数 $p_{ia}$：$i$ 表示该系数对应的幂次，$_a$ 表示这些多项式已与见证 $\mathbf{a}$ 组合。

按这种方式合并之后，多项式如下：

$$
\begin{align*}
\sum_{i=1}^4 a_iu_i(x) &= u_{2a}x^2+u_{1a}x+u_{0a}\\
\sum_{i=1}^4 a_iv_i(x) &= v_{2a}x^2+v_{1a}x+v_{0a}\\
\sum_{i=1}^4 a_iw_i(x) &= w_{2a}x^2+w_{1a}x+w_{0a}\\
\end{align*}
$$

## 将可信设置与 QAP 结合
现在可以使用可信设置生成的结构化参考字符串对这些多项式求值。

也就是说，给定结构化参考字符串：

$$
[\Omega_2, \Omega_1, G_1], [\Theta_2, \Theta_1, G_2], \space\Omega_i \in \mathbb{G}_1, \space\Theta_i \in \mathbb{G}_2
$$

它在可信设置中按如下方式计算：

$$
\begin{align*}
[\Omega_2, \Omega_1, G_1] &= [\tau^2G_1, \tau G_1, G_1], \space\Omega_i \in \mathbb{G}_1\\
[\Theta_2, \Theta_1, G_2] &= [\tau^2G_2, \tau G_2, G_2], \space\Theta_i \in \mathbb{G}_2
\end{align*}
$$

可以计算：

$$
\begin{align*}
[A]_1 &=\sum_{i=1}^4 a_iu_i(\tau) = \langle[u_{2a}, u_{1a}, u_{0a}],[\Omega_2, \Omega_1, G_1]\rangle\\
[B]_2 &=\sum_{i=1}^4 a_iv_i(\tau) = \langle[v_{2a}, v_{1a}, v_{0a}],[\Theta_2, \Theta_1, G_2]\rangle\\
[C]_1 &=\sum_{i=1}^4 a_iw_i(\tau) = \langle[v_{2a}, v_{1a}, v_{0a}],[\Omega_2, \Omega_1, G_1]\rangle \\
\end{align*}
$$

这里的 $u_i(\tau), v_i(\tau), w_i(\tau)$ 表示使用可信设置中由 $\tau$ 生成的结构化参考字符串对多项式求值，并不是“代入 $\tau$ 后对多项式求值”。可信设置结束后 $\tau$ 已被销毁，所以 $\tau$ 的值无人知晓。

我们已经使用 SRS 计算了 QAP 的大部分内容，但还没有计算 $h(x)t(x)$：

$$
\underbrace{\sum_{i=1}^m a_iu_i(x)}_{[A]_1}\underbrace{\sum_{i=1}^m a_iv_i(x)}_{[B]_2} = \underbrace{\sum_{i=1}^m a_iw_i(x)}_{[C]_1} + \underbrace{h(x)t(x)}_{??}
$$

## 计算 $h(x)t(x)$

回顾一下，$t(x)$ 的次数为 3（一般为 $n$），而 $h(x)$ 的次数为 1（一般为 $n - 2$）。两者相乘，最高可能得到三次多项式，超出了 Powers of Tau 仪式所提供的幂次范围。因此，必须调整 Powers of Tau 仪式，为 $h(x)t(x)$ 提供专用的结构化参考字符串。

执行可信设置的人知道 $t(x)$，因为它就是 $(x - 1)(x - 2)...(x - n)$。但是，$h(x)$ 由证明者计算，并且会随 $\mathbf{a}$ 的值变化，因此在可信设置期间无法预先知道它。

注意，不能使用结构化参考字符串分别对 $h(\tau)$ 与 $t(\tau)$ 求值，再把它们配对。那样无法得到我们所需的 $\mathbb{G}_1$ 元素。

### 多项式乘积的 SRS
注意，以下计算都会得到相同的值：
- 在 $u$ 处对多项式 $h(x)t(x)$ 求值，即 $(h(x)t(x))(u)$
- $h(u)$ 与 $t(u)$ 相乘，即 $h(u)t(u)$（在 $u$ 处对 $h$ 求值，并在 $u$ 处对 $t$ 求值）
- 先将 $h(x)$ 乘以求值结果 $t(u)$，再在 $u$ 处求值，即 $(h(x)t(u))(u)$

我们将使用第三种方法计算 $h(\tau)t(\tau)$。不失一般性，假设 $h(x)$ 为 $3x^2 + 6x + 2$，且 $t(u) = 4$。计算如下：

$$
h(x)t(u) = (3x^2 + 6x + 2) \cdot 4 = 12x^2 + 24x + 8
$$

如果把 $u$ 代入 $12x^2 + 24x + 8$，结果就是 $h(u)t(u)$。

但是，要在 $\tau$ 处对这个多项式求值，证明者就必须知道 $\tau$。这里的关键洞见是，上面的计算可以写成：

$$
h(u)t(u) = \langle[3, 6, 2], [4u^2, 4u, 4]\rangle=12u^2+24u+8
$$

如果可信设置提供 $[4u^2, 4u, 4]$，而证明者提供 $[3, 6, 2]$，证明者就能在不知道 $u$ 的情况下计算 $h(u)t(u)$，因为所有包含 $u$ 的量都位于内积的右侧向量中。

### 用于 $h(\tau)t(\tau)$ 的结构化参考字符串
为了创建用于 $h(\tau)t(\tau)$ 的结构化参考字符串，需要生成 $n - 1$ 个值：将 $t(\tau)$ 分别乘以 $\tau$ 的连续幂。

$$
[\Upsilon_{n-2}, \Upsilon_{n-3}, ..., \Upsilon_1, \Upsilon_0] = [\tau^{n-2}t(\tau)G_1, \tau^{n-3}t(\tau)G_1, ..., \tau t(\tau)G_1, t(\tau)G_1]
$$

（这里容易混淆：一个 $k$ 次多项式有 $k+1$ 项，因此要为 $k-2$ 次多项式生成 $k-1$ 个求值。注意，Upsilon 从 ${n-1}$ 开始，到 0 结束。）

这里的 $n$ 是 R1CS 的行数，并且我们已经证明 $h$ 的次数不会超过 $n - 2$。

证明者使用该结构化参考字符串，按如下方式计算 $h(\tau)t(\tau)$：

$$
h(\tau)t(\tau) = \langle[h_{n-2}, h_{n-3}, ..., h_1, h_0], [\Upsilon_{n-2}, \Upsilon_{n-3}, ..., \Upsilon_1, \Upsilon_0] \rangle
$$

## 在可信设置上对 QAP 求值
现在把前面的内容组合起来。假设一个 R1CS 的矩阵有 $n$ 行、$m$ 列。对它应用拉格朗日插值，可以将其转换为 QAP：

$$
\sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x) = \sum_{i=1}^m a_iw_i(x) + h(x)t(x)
$$

各求和项都会产生一个 $n - 1$ 次多项式（拉格朗日多项式的次数比插值点数量少 1），$h(x)$ 的次数最高为 $n - 2$，而 $t(x)$ 的次数为 $n$。

可信设置生成一个随机域元素 $\tau$，并计算：

$$
\begin{align*}
[\Omega_{n-1},\Omega_{n-2},..., \Omega_1, G_1] &= [\tau^{n-1}G_1, \tau^{n-2}G_1,\dots, \tau G_1, G_1]\\
[\Theta_{n-1}, \Theta_{n-2}, ..., \Theta_1, G_2] &= [\tau^{n-1}G_2, \tau^{n-2}G_2,\dots, \tau G_2, G_2]\\
[\Upsilon_{n-2}, \Upsilon_{n-3}, ..., \Upsilon_1, \Upsilon_0] &= [\tau^{n-2}t(\tau)G_1, \tau^{n-3}t(\tau)G_1, ..., \tau t(\tau)G_1, t(\tau)G_1]
\end{align*}
$$

注意，结构化参考字符串必须包含足够多的项，才能容纳 QAP 中的多项式。

随后，可信设置销毁 $\tau$，并发布这些结构化参考字符串：

$$
([\Omega_2, \Omega_1, G_1], [\Theta_2, \Theta_1, G_2], [\Upsilon_{n-2}, \Upsilon_{n-3}, ..., \Upsilon_1, \Upsilon_0])
$$

证明者按如下方式对 QAP 的各个组成部分求值：

$$
\underbrace{\sum_{i=1}^m a_iu_i(x)}_{A}\underbrace{\sum_{i=1}^m a_iv_i(x)}_B = \underbrace{\sum_{i=1}^m a_iw_i(x) + h(x)t(x)}_{C}
$$

$$
\begin{align*}
[A]_1 &=\sum_{i=1}^m a_iu_i(\tau) = \langle[u_{{n-1}a}, u_{{n-2}a}, \dots, u_{1a}, u_{0a}],[\Omega_{n-1}, \Omega_{n-2}, \dots, \Omega_1, G_1]\rangle\\
[B]_2 &=\sum_{i=1}^m a_iv_i(\tau) = \langle[v_{{n-1}a}, v_{{n-2}a}, \dots, v_{1a}, v_{0a}],[\Theta_{n-1}, \Theta_{n-2}, \dots, \Theta_1, G_2]\rangle\\
[C]_1 &=\sum_{i=1}^m a_iw_i(\tau) + h(\tau)t(\tau) = \langle[w_{{n-1}a}, w_{{n-2}a}, \dots, w_{1a}, w_{0a}],[\Omega_{n-1}, \Omega_{n-2}, \dots, \Omega_1, G_1]\rangle \\
&+\langle[h_{n-2}, h_{n-3}, \dots, h_1, h_0], [\Upsilon_{n-2}, \Upsilon_{n-3}, \dots, \Upsilon_1, \Upsilon_0] \rangle\\
\end{align*}
$$

证明者发布 $([A]_1, [B]_2, [C]_1)$，验证者则检查：

$$
[A]_1 \bullet [B]_2 \stackrel{?}= [C]_1 \bullet G_2
$$

如果见证 $\mathbf{a}$ 满足 QAP，上式就会成立。不过，等式成立并不能保证证明者确实知道一个满足约束的 $\mathbf{a}$：证明者可以发布任意椭圆曲线点，而验证者无法判断这些点是否真的由 QAP 推导而来。

## 证明非常小
注意，证明只包含三个椭圆曲线点。如果一个 $\mathbb{G}_1$ 元素占 64 字节，一个 $\mathbb{G}_2$ 元素占 128 字节，那么整个证明只有 256 字节。无论 R1CS 的规模多大，证明大小*始终如此*！

R1CS 越大，证明者的工作量越大，但验证者的工作量保持不变。

下一章的 [Groth16 协议](https://www.rareskills.io/post/groth16)将说明如何解决上述问题。

从 Tornado Cash 源代码中的 [struct](https://github.com/tornadocash/tornado-core/blob/master/contracts/Verifier.sol#L167-L171)
 `Proof` 可以看出，Groth16 的证明仍保持恒定大小。

*最初发布于 2023 年 8 月 28 日*
