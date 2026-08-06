# Groth16 详解

Groth16 算法允许证明者在可信设置生成的椭圆曲线点上计算二次算术程序，并由验证者快速检查。它使用可信设置生成的辅助椭圆曲线点来防止伪造证明。

## 前置知识
本文是 [RareSkills 零知识证明教程](https://www.rareskills.io/zk-book)中的一章，假定读者已经熟悉前面的章节。

## 记号
属于椭圆曲线群 $\mathbb{G}_1$ 的[椭圆曲线点](https://www.rareskills.io/post/elliptic-curves-finite-fields)记作 $[x]_1$，属于椭圆曲线群 $\mathbb{G}_2$ 的点记作 $[x]_2$。$[x]_1$ 与 $[x]_2$ 之间的[配对](https://www.rareskills.io/post/bilinear-pairing)记作 $[x]_1\bullet[x]_2$，结果是 $\mathbb{G}_{12}$ 中的元素。$\mathbf{a}$ 这样的粗体变量表示向量，$\mathbf{L}$ 这样的大写粗体字母表示矩阵，而域元素（有时非正式地称为“标量”）用 $d$ 这样的小写字母表示。所有算术运算均在[有限域](https://www.rareskills.io/post/finite-fields)中进行，该域的特征等于椭圆曲线群的阶。

给定一个[算术电路（ZK 电路）](https://www.rareskills.io/post/arithmetic-circuit)，我们将其转换为[秩一约束系统（Rank-1 Constraint System，R1CS）](https://www.rareskills.io/post/rank-1-constraint-system) $\mathbf{L}\mathbf{a}\circ \mathbf{R}\mathbf{a} = \mathbf{O}\mathbf{a}$。其矩阵有 $n$ 行、$m$ 列，并带有见证向量（witness vector）$\mathbf{a}$。然后，把各矩阵的列作为 $y$ 值，在 $x$ 值 $[1,2,...,n]$ 上进行插值，即可将 R1CS 转换成[二次算术程序（Quadratic Arithmetic Program，QAP）](https://www.rareskills.io/post/quadratic-arithmetic-program)。由于 $\mathbf{L}$、$\mathbf{R}$ 和 $\mathbf{O}$ 都有 $m$ 列，最终会得到三组多项式，每组有 $m$ 个：

$$
\begin{array}{}
u_1(x),...,u_m(x) & m \text{ 个多项式，对 }m \text{ 列插值得到，矩阵为 } \mathbf{L}\\
v_1(x),...,v_m(x)& m \text{ 个多项式，对 }m \text{ 列插值得到，矩阵为 } \mathbf{R}\\
w_1(x),...,w_m(x)& m \text{ 个多项式，对 }m \text{ 列插值得到，矩阵为 } \mathbf{O}\\
\end{array}
$$

据此可以构造二次算术程序（QAP）：

$$
\sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x) = \sum_{i=1}^m a_iw_i(x) + h(x)t(x)
$$
其中
$$
t(x) = (x - 1)(x - 2)\dots(x - n)
$$
并且
$$
h(x) = \frac{\sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x) - \sum_{i=1}^m a_iw_i(x)}{t(x)}
$$

如果第三方通过 Powers of Tau 仪式创建结构化参考字符串（SRS），证明者就能在隐藏点 $\tau$ 处对 QAP 中的求和项（即 $\sum a_if_i(x)$ 形式的项）求值。令结构化参考字符串按如下方式计算：

$$
\begin{align*}
[\Omega_{n-1}, \Omega_{n-2},\dots,\Omega_2,\Omega_1,G_1] &= [\tau^nG_1,\tau^{n-1}G_1,\dots,\tau G_1,G_1] && \text{SRS，所属群为 } G_1 \\
[\Theta_{n-1}, \Theta_{n-2},\dots,\Theta_2,\Theta_1,G_2] &= [\tau^nG_2,\tau^{n-1}G_2,\dots,\tau G_2,G_2] && \text{SRS，所属群为 } G_2\\
[\Upsilon_{n-2},\Upsilon_{n-3},\dots,\Upsilon_1,\Upsilon_0]&=[\tau^{n-2}t(\tau)G_1,\tau^{n-3}t(\tau)G_1,\dots,\tau t(\tau)G_1,t(\tau)G_1] && \text{用于该项的 SRS：}h(\tau)t(\tau)\\
\end{align*}
$$

我们用 $f(\tau)$ 表示通过内积，在结构化参考字符串 $[\tau^dG_1,...,\tau^2G_1,\tau G_1,G_1]$ 上对多项式求值：

$$
f(\tau) = \sum_{i=1}^d f_i\Omega_i=\langle[f_d, f_{d-1},...,f_1,f_0],[\Omega_d,\Omega_{d-1},...,G_1]\rangle
$$

对于 $\mathbb{G}_2$ 中的 SRS，则为：

$$
f(\tau) = \sum_{i=1}^d f_i\Theta_i=\langle[f_d, f_{d-1},...,f_1,f_0],[\Theta_d,\Theta_{d-1},...,G_2]\rangle
$$

$f(\tau)$ 是上述表达式的简写，其结果是一个椭圆曲线点；它并不表示证明者知道 $\tau$。

证明者可以通过以下计算，在可信设置上对 QAP 求值：

$$
\begin{align*}
[A]_1 &= \sum_{i=1}^m a_iu_i(\tau)\\
[B]_2 &= \sum_{i=1}^m a_iv_i(\tau)\\
[C]_1 &= \sum_{i=1}^m a_iw_i(\tau) + h(\tau)t(\tau)
\end{align*}
$$

具体计算过程参见教程[椭圆曲线上的二次算术程序](https://www.rareskills.io/post/elliptic-curve-qap)。

如果 QAP 等式成立，那么下式成立：

$$
[A]_1\bullet[B]_2 \stackrel{?}= [C]_1\bullet G_2
$$

## 动机
仅仅给出 $([A]_1, [B]_2, [C]_1)$，并不能令人信服地证明证明者知道某个使 QAP 等式成立的 $\mathbf{a}$。

证明者可以随意编造满足 $ab = c$ 的 $a$、$b$、$c$，并计算：

$$
\begin{align*}
[A]_1 &= aG_1\\
[B]_2 &= bG_2\\
[C]_1 &= cG_1
\end{align*}
$$

然后把这些椭圆曲线点作为 $[A]_1$、$[B]_2$、$[C]_1$ 交给验证者。


因此，验证者完全无法判断 $([A]_1, [B]_2, [C]_1)$ 是否确实由一个满足 QAP 的见证计算得到。

我们需要迫使证明者诚实行事，同时不能引入过多的计算开销。首个实现这一目标的算法是论文 “[Pinocchio: Nearly Practical Verifiable Computation](https://eprint.iacr.org/2013/279.pdf)”。它已经足够实用，Zcash 的第一版区块链正是以此为基础。

不过，Groth16 用少得多的步骤实现了同样的目标。该算法至今仍被广泛使用，因为此后还没有其他算法在验证步骤上达到同等效率（尽管其他算法已经消除了可信设置，或显著减少了证明者的工作量）。

**2024 年更新：** 密码学领域发表了一篇标题颇为自信的论文 “[Polymath: Groth16 is not the limit](https://eprint.iacr.org/2024/916)”，展示了一种所需验证步骤少于 Groth16 的算法。不过，在本文写作时，该算法尚无已知实现。

## 防止伪造（第一部分）：引入 $\alpha$ 和 $\beta$

### 一个“无解”的验证公式
假设把验证公式更新为：

$$
[A]_1 \bullet [B]_2 \stackrel{?}= [D]_{12} + [C]_1\bullet G_2
$$

*注意：为方便起见，这里对 $G_{12}$ 群采用加法记号。*

这里，$[D]_{12}$ 是 $G_{12}$ 中的元素，其离散对数未知。

下面将说明：如果不知道 $[D]_{12}$ 的离散对数，就无法为该等式提供解 $([A]_1, [B]_2, [C]_1)$。

#### 攻击 1：伪造 A 和 B，再推导 C
假设证明者随机选择 $a'$ 和 $b'$ 来生成 $[A]₁$ 和 $[B]₂$，并尝试推导出一个与验证公式相容的 $[C']$。

$$
[A]_1 \bullet [B]_2 \stackrel{?}= [D]_{12} + [C]_1\bullet G_2
$$

恶意证明者知道 $[A]₁$ 与 $[B]₂$ 的离散对数，于是尝试通过以下计算求出 $[C']$：

$$
\begin{align*}\underbrace{[A]_1\bullet [B]_2 - [D]_{12}}_{\chi_{12}}=[C']_1\bullet G_2\\
[\chi]_{12}=[C']_1\bullet G_2
\end{align*}
$$

最后一行要求证明者求解 $\chi_{12}$ 的离散对数，因此无法为 $[C']_1$ 计算出有效的离散对数。

#### 攻击 2：伪造 C，再推导 A 和 B
在这种攻击中，证明者选择一个随机点 $c'$ 并计算 $[C']_1$。由于知道 $c'$，证明者可以尝试找到一组相容的 $a'$ 与 $b'$，使得：


$$
\begin{align*}[A]_1 \bullet [B]_2 &\stackrel{?}= \underbrace{[D]_{12} + [C]_1\bullet G_2}_{[\zeta]_{12}}\\
[A]_1 \bullet [B]_2 &\stackrel{?}=[\zeta]_{12}
\end{align*}
$$

这要求证明者在给定 $[\zeta]_{12}$ 的情况下，找出一对 $[A]₁$ 与 $[B]₂$，使其配对结果为 $[\zeta]_{12}$。

与离散对数问题类似，我们依赖一个尚未得到证明的密码学假设：将 $\mathbb{G}_{12}$ 中的元素分解成 $\mathbb{G}_1$ 和 $\mathbb{G}_2$ 中的元素在计算上不可行。在这里，无法将 $[\zeta]_{12}$ 分解为 $[A]₁$ 与 $[B]₂$ 的假设称为*双线性 Diffie–Hellman 假设*（Bilinear Diffie-Hellman Assumption）。感兴趣的读者可以参考关于 [判定性 Diffie–Hellman 假设](https://en.wikipedia.org/wiki/Decisional_Diffie–Hellman_assumption)的相关讨论。

（尚未证明并不等于不可靠。如果你能证明或推翻这个假设，名声与财富都在等着你！实践中，目前没有已知方法可以把 $[\zeta]_{12}$ 分解为 $[A]₁$ 和 $[B]₂$，并且普遍认为这一计算不可行。）

### $\alpha$ 和 $\beta$ 的用法
在实际的 Groth16 中，并不直接使用 $[D]_{12}$。可信设置会生成两个随机标量 $\alpha$ 和 $\beta$，并发布按如下方式计算的椭圆曲线点 $([\alpha]_1,[\beta]_2)$：

$$
\begin{align*}
[α]_1 &= α G_1 \\
[β]_2 &= β G_2
\end{align*}
$$

前文的 $[D]_{12}$ 就是 $[\alpha]_1 \bullet [\beta]_2$。

### 重新推导证明与验证公式
为了让验证公式 $[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1\bullet[\beta]_2 + [C]_1\bullet G_2$ “有解”，需要修改 QAP 公式，将 $\alpha$ 和 $\beta$ 纳入其中。

$$
\sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x) = \sum_{i=1}^m a_iw_i(x) + h(x)t(x)
$$

现在考察在等式左侧引入 $\theta$ 与 $\eta$ 后会发生什么：

$$
\left(\boxed{\theta}+\sum_{i=1}^m a_iu_i(x)\right)\left(\boxed{\eta} +\sum_{i=1}^m a_iv_i(x)\right) =
$$
$$
=\boxed{\theta\eta} + \boxed{\theta}\sum_{i=1}^m a_iv_i(x) + \boxed{\eta}\sum_{i=1}^m a_iu_i(x) + \sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x)
$$

使用原始 QAP 定义替换最右侧的项：
$$
=\theta\eta + \theta\sum_{i=1}^m a_iv_i(x) + \eta\sum_{i=1}^m a_iu_i(x) + \boxed{\sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x)}
$$

$$
=\theta\eta + \theta\sum_{i=1}^m a_iv_i(x) + \eta\sum_{i=1}^m a_iu_i(x) + \boxed{\sum_{i=1}^m a_iw_i(x) + h(x)t(x)}
$$

现在可以定义如下“展开后的”QAP：

$$
\left(\theta+\sum_{i=1}^m a_iu_i(x)\right)\left(\eta +\sum_{i=1}^m a_iv_i(x)\right) =\theta\eta + \theta\sum_{i=1}^m a_iv_i(x) + \eta\sum_{i=1}^m a_iu_i(x) + \sum_{i=1}^m a_iw_i(x) + h(x)t(x)
$$

先简要展示一下后续方向：若把 $\theta$ 替换为 $[\alpha]_1$，把 $\eta$ 替换为 $[\beta]_2$，就会得到前面更新后的验证公式：

$$
[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + [C]_1\bullet G_2
$$

其中

$$
\underbrace{\left([\alpha]_1+\sum_{i=1}^m a_iu_i(\tau)\right)}_{[A]_1}\underbrace{\left([\beta]_2 +\sum_{i=1}^m a_iv_i(\tau)\right)}_{[B]_2} =[\alpha]_1\bullet[\beta]_2 + \underbrace{\left(\alpha\sum_{i=1}^m a_iv_i(\tau) + \beta\sum_{i=1}^m a_iu_i(\tau) + \sum_{i=1}^m a_iw_i(\tau) + h(\tau)t(\tau)\right)}_{[C]_1} \bullet G_2
$$

证明者无需知道 $\tau$、$\alpha$ 或 $\beta$，就能计算 $[A]_1$ 与 $[B]_2$。给定结构化参考字符串（$\tau$ 的连续幂）以及椭圆曲线点 $([α]_1,[β]_2)$，证明者按如下方式计算 $[A]_1$ 和 $[B]_2$：

$$
\begin{align*}
[A]_1 &= [\alpha]_1 + \sum_{i=1}^m a_iu_i(\tau)\\
[B]_2 &= [\beta]_2 + \sum_{i=1}^m a_iv_i(\tau)\\
\end{align*}
$$

这里，$a_iu_i(\tau)$ 并不表示证明者知道 $\tau$。证明者使用结构化参考字符串 $[\tau^{n-1}G_1,\tau^{n-2}G_1,\dots,\tau G_1,G_1]$ 计算 $i=1,2,\dots,m$ 时的 $u_i(\tau)$，并使用 $G_2$ 中的 SRS 计算 $[B]_2$。

不过，在不知道 $\alpha$ 和 $\beta$ 的情况下，目前还无法计算 $[C]_1$。证明者不能将 $[\alpha]_1$ 与 $\sum a_iu_i(\tau)$ 配对，也不能将 $[\beta]_2$ 与 $\sum a_iv_i(\tau)$ 配对，因为那样会产生 $\mathbb{G}_{12}$ 中的点，而证明者需要为 $[C]_1$ 得到一个 $\mathbb{G}_1$ 中的点。

因此，可信设置需要为展开后 QAP 中难以处理的 $C$ 项预计算 $m$ 个多项式。

$$
\alpha\sum_{i=1}^m a_iv_i(\tau) + \beta\sum_{i=1}^m a_iu_i(\tau) + \sum_{i=1}^m a_iw_i(\tau)
$$

通过一些代数变换，可以把各求和项合并成一个求和：

$$
=\sum_{i=1}^m (\alpha a_iv_i(\tau)+\beta a_iu_i(\tau) + a_iw_i(\tau))
$$

再提取公因子 $a_i$：

$$
=\sum_{i=1}^m a_i\boxed{(\alpha v_i(\tau)+\beta u_i(\tau) + w_i(\tau))}
$$

可信设置可以根据上面的方框项，创建 $m$ 个在 $\tau$ 处求值后的多项式，证明者便可用它们计算该求和。具体细节见下一节。

### 当前算法小结
#### 可信设置步骤
具体来说，可信设置执行以下计算：
$$
\begin{align*}
\alpha,\beta,\tau &\leftarrow \text{随机标量}\\
[\tau^{n-1}G_1,\tau^{n-2}G_1,\dots,\tau G_1,G_1] &\leftarrow \text{SRS，所属群为 } \mathbb{G}_1\\
[\tau^{n-1}G_2,\tau^{n-2}G_2,\dots,\tau G_2,G_2] &\leftarrow \text{SRS，所属群为 } \mathbb{G}_2\\
[\tau^{n-2}t(\tau)G_1,\tau^{n-3}t(\tau)G_1,\dots,\tau t(\tau)G_1,t(\tau)G_1] &\leftarrow \text{用于该项的 SRS：}h(\tau)t(\tau)\\
[\Psi_1]_1 &= (\alpha v_1(\tau) + \beta u_1(\tau) + w_1(\tau))G_1\\
[\Psi_2]_1 &= (\alpha v_2(\tau) + \beta u_2(\tau) + w_2(\tau))G_1\\
&\vdots\\
[\Psi_m]_1 &= (\alpha v_m(\tau) + \beta u_m(\tau) + w_m(\tau))G_1\\
\end{align*}
$$

可信设置发布：

$$
([\alpha]_1,[\beta]_2,\text{srs}_{G_1},\text{srs}_{G_2},\text{srs for }h(\tau)t(\tau),[\Psi_1]_1,[\Psi_2]_1,\dots,[\Psi_m]_1)
$$

#### 证明者步骤
证明者计算：

$$
\begin{align*}
[A]_1 &= [\alpha]_1 + \sum_{i=1}^m a_iu_i(\tau)\\
[B]_2 &= [\beta]_2 + \sum_{i=1}^m a_iv_i(\tau)\\
[C]_1 &= \sum_{i=1}^m a_i[\Psi_i]_1 + h(\tau)t(\tau)\\
\end{align*}
$$

注意，我们将这个包含 $\alpha$ 与 $\beta$、因而“难以处理”的多项式

$$
=\sum_{i=1}^m a_i\boxed{(\alpha v_i(\tau)+\beta u_i(\tau) + w_i(\tau))}
$$

替换成：

$$
\sum_{i=1}^m a_i[\Psi_i]_1
$$

#### 验证者步骤

验证者检查：

$$
[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + [C]_1\bullet G_2
$$

## 支持公开输入

到目前为止，验证公式还不支持公开输入，也就是不能将见证的一部分公开。

按照惯例，见证的公开部分是向量 $\mathbf{a}$ 的前 $\ell$ 个元素。要公开这些元素，证明者只需将它们披露出来：

$$
[a_1, a_2, \dots, a_\ell]
$$

为了检查这些值是否确实参与了计算，验证者必须接手原本由证明者完成的一部分计算。

具体而言，证明者计算：

$$
\begin{align*}
[A]_1 &= [\alpha]_1 + \sum_{i=1}^m a_iu_i(\tau)\\
[B]_2 &= [\beta]_2 + \sum_{i=1}^m a_iv_i(\tau)\\
[C]_1 &= \sum_{i=\ell+1}^m a_i[\Psi_i]_1 + h(\tau)t(\tau)\\
\end{align*}
$$

注意，只有 $[C]_1$ 的计算发生了变化——证明者仅使用从 $\ell + 1$ 到 $m$ 的 $a_i$ 与 $\Psi_i$ 项。

验证者计算求和的前 $\ell$ 项：
$$
[X]_1=\sum_{i=1}^\ell a_i\Psi_i
$$

验证等式为：

$$
[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + [X]_1\bullet G_2 + [C]_1\bullet G_2
$$

## 第二部分：使用 $\gamma$ 或 $\delta$ 分离公开输入与私有输入
### 利用 $i\leq\ell$ 时的 $\Psi_i$ 伪造证明

上式假定证明者只使用 $\Psi_{\ell+1}$ 到 $\Psi_m$ 计算 $[C]_1$，但没有任何机制阻止不诚实的证明者使用 $\Psi_1$ 到 $\Psi_{\ell}$ 计算 $[C]_1$，从而伪造证明。

例如，当前验证等式为：

$$
[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + \sum_{i=1}^\ell a_i\Psi_i +  [C]_1\bullet G_2
$$

如果展开其中的 C 项，可得：

$$
[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + \sum_{i=1}^\ell a_i\Psi_i + \underbrace{\left(\sum_{i=\ell+1}^m a_i[\Psi_i]_1 + h(\tau)t(\tau)\right)}_C \bullet G_2
$$

不失一般性，假设 $\mathbf{a} = [1,2,3,4,5]$ 且 $\ell=3$。此时，见证的公开部分为 $[1,2,3]$，私有部分为 $[4,5]$。

最终等式如下：

$$
[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + (1\Psi_1+2\Psi_2+3\Psi_3)\bullet G2 + \underbrace{(4\Psi_4 + 5\Psi_5  + h(\tau)t(\tau))}_C \bullet G_2
$$

但是，没有任何机制阻止证明者构造一个值为 [1,2,0] 的有效公开见证部分，并把公开部分中被置零的值移到私有计算中：

$$
[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + (1\Psi_1+2\Psi_2+\boxed{0\Psi_3})\bullet G2 + \underbrace{(\boxed{3\Psi_3}+4\Psi_4 + 5\Psi_5  + h(\tau)t(\tau))}_C \bullet G_2
$$

上式成立，但该见证不一定满足原始约束。

因此，需要阻止证明者在 $[C]_1$ 的计算中使用 $\Psi_1$ 到 $\Psi_{\ell}$。

### 引入 $\gamma$ 和/或 $\delta$
为避免上述问题，可信设置引入新的标量 $\gamma$ 或 $\delta$，强制将 $\Psi_{\ell+1}$ 到 $\Psi_m$ 与 $\Psi_1$ 到 $\Psi_{\ell}$ 分离。具体做法是：可信设置将私有项（构成 $[C]_1$）除以 $\delta$，也就是乘以其模逆；并且/或者将公开项（构成验证者所计算之和 $[X]_1$）除以 $\gamma$。

由于 $h(\tau)t(\tau)$ 项嵌入在 $[C]_1$ 中，这些项也需要除以 $\delta$。只要 $\delta$ 和 $\gamma$ 中任意一个的离散对数未知，就能避免前述伪造以及其他可能的伪造方法。Zcash 基于 Sapling 的[可信设置](https://github.com/ebfull/phase2/blob/master/src/lib.rs#L808)采用了这种方法：$\gamma$ 在 $G_2$ 上保持不变，而 $\delta$ 会在可信设置的后续阶段从 $G_2$ 更新为随机值。

$$
\begin{align*}
\alpha,\beta,\tau,\gamma,\delta &\leftarrow \text{随机标量}\\
[\tau^{n-1}G_1,\tau^{n-2}G_1,\dots,\tau G_1,G_1] &\leftarrow \text{SRS，所属群为 } \mathbb{G}_1\\
[\tau^{n-1}G_2,\tau^{n-2}G_2,\dots,\tau G_2,G_2] &\leftarrow \text{SRS，所属群为 } \mathbb{G}_2\\
\left[\frac{\tau^{n-2}t(\tau)}{\delta}G_1,\frac{\tau^{n-3}t(\tau)}{\delta}G_1,\dots,\frac{\tau t(\tau)}{\delta}G_1,
\frac{t(\tau)}{\delta}G_1\right] &\leftarrow \text{用于该项的 SRS：}h(\tau)t(\tau)\\
\\
&\text{见证的公开部分}\\
[\Psi_1]_1 &= \frac{\alpha v_1(\tau) + \beta u_1(\tau) + w_1(\tau)}{\gamma}G_1\\
[\Psi_2]_1 &= \frac{\alpha v_2(\tau) + \beta u_2(\tau) + w_2(\tau)}{\gamma}G_1\\
&\vdots\\
[\Psi_\ell]_1 &= \frac{\alpha v_\ell(\tau) + \beta u_\ell(\tau) + w_\ell(\tau)}{\gamma}G_1\\
\\
&\text{见证的私有部分}\\
[\Psi_{\ell+1}]_1 &= \frac{\alpha v_{\ell+1}(\tau) + \beta u_{\ell+1}(\tau) + w_{\ell+1}(\tau)}{\delta}G_1\\
[\Psi_{\ell+2}]_1 &= \frac{\alpha v_{\ell+2}(\tau) + \beta u_{\ell+2}(\tau) + w_{\ell+2}(\tau)}{\delta}G_1\\
&\vdots\\
[\Psi_{m}]_1 &= \frac{\alpha v_{m}(\tau) + \beta u_{m}(\tau) + w_{m}(\tau)}{\delta}G_1\\
\end{align*}
$$

可信设置发布：
$$
([\alpha]_1,[\beta]_2,[\gamma]_2,[\delta]_2,\text{srs}_{G_1},\text{srs}_{G_2},\text{srs for }h(\tau)t(\tau),[\Psi_1]_1,[\Psi_2]_1,\dots,[\Psi_m]_1)
$$

证明者的步骤与之前相同：

$$
\begin{align*}
[A]_1 &= [\alpha]_1 + \sum_{i=1}^m a_iu_i(\tau)\\
[B]_2 &= [\beta]_2 + \sum_{i=1}^m a_iv_i(\tau)\\
[C]_1 &= \sum_{i=\ell+1}^m a_i[\Psi_i]_1 + h(\tau)t(\tau)\\
\end{align*}
$$

验证者的步骤现在还包含与 $[\gamma]_2$ 和/或 $[\delta]_2$ 的配对，用来消去分母：

$$
[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + [X]_1\bullet [\gamma]_2 + [C]_1\bullet [\delta]_2
$$

## 第三部分：使用 r 和 s 实现真正的零知识性
当前方案还不具备真正的零知识性。如果攻击者能够猜出见证向量（当有效输入范围很小时，这是有可能的，例如仅允许特权地址参与的秘密投票），就可以把自己构造的证明与原始证明比较，从而验证猜测是否正确。

来看一个简单示例：假设我们声称 $x_1$ 与 $x_2$ 都只能取 $0$ 或 $1$。对应的算术电路为：

$$
\begin{align*}
x_1 (x_1 - 1) = 0\\
x_2 (x_2 - 1) = 0
\end{align*}
$$

攻击者只需猜测四种组合，就能确定见证。也就是说，攻击者猜出一个见证、生成证明，再检查结果是否与原始证明相同。

为防止这种猜测，证明者需要为证明“加盐”，同时修改验证等式以容纳这些盐值。

证明者采样两个随机域元素 $r$ 与 $s$，并分别把它们加入 $A$ 和 $B$，使见证无法被猜出——攻击者必须同时猜中见证以及盐值 $r$ 和 $s$：

$$
\begin{align*}
[A]_1 &= [\alpha]_1 + \sum_{i=1}^m a_iu_i(\tau) + r[\delta]_1\\
[B]_2 &= [\beta]_2 + \sum_{i=1}^m a_iv_i(\tau) + s[\delta]_2\\
[B]_1 &= [\beta]_1 + \sum_{i=1}^m a_iv_i(\tau) + s[\delta]_1\\
[C]_1 &= \sum_{i=\ell+1}^m a_i[\Psi_i]_1 + h(\tau)t(\tau) + As+Br-rs[\delta]_1\\
\end{align*}
$$

为了推导最终验证公式，先暂时忽略我们并不知道希腊字母所代表各项的离散对数，直接计算验证等式左侧的 $AB$：

$$
\underbrace{\left(\alpha + \sum_{i=1}^m a_iu_i(x) + r\delta\right)}_A \underbrace{\left(\beta + \sum_{i=1}^m a_iv_i(x) + s\delta\right)}_B
$$

展开各项，得到：

$$
\alpha\beta+\alpha\sum_{i=1}^m a_iv_i(x)+\alpha s\delta + \beta\sum_{i=1}^m a_iu_i(x) + \sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x)+\sum_{i=1}^m a_iu_i(x) s\delta + r\delta\beta + r\delta\sum_{i=1}^m a_iv_i(x) + r\delta s\delta
$$

从中选出原来属于 $C$ 的各项：

$$
\alpha\beta+\boxed{\alpha\sum_{i=1}^m a_iv_i(x)}+\alpha s\delta + \boxed{\beta\sum_{i=1}^m a_iu_i(x)} + \boxed{\sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x)}+\sum_{i=1}^m a_iu_i(x) s\delta + r\delta\beta + r\delta\sum_{i=1}^m a_iv_i(x) + r\delta s\delta
$$

将原有各项放在左侧组合起来，把新引入的项留在右侧：

$$
\alpha\beta + \boxed{\alpha\sum_{i=1}^m a_iv_i(x) + \beta\sum_{i=1}^m a_iu_i(x) + \sum_{i=1}^m a_iu_i(x)\sum_{i=1}^m a_iv_i(x)}+ \underline{\alpha s\delta + \sum_{i=1}^m a_iu_i(x) s\delta + r\delta\beta + r\delta\sum_{i=1}^m a_iv_i(x) + r\delta s\delta}
$$

进一步重排带下划线的各项，将其写成 $As\delta$ 与 $Br\delta$ 的形式。同时，把 $r\delta s\delta$ 拆成 $rs\delta^2 + rs\delta^2 - rs\delta^2$：

$$
=\alpha s\delta + \sum_{i=1}^m a_iu_i(x) s\delta + rs\delta^2 + r\delta\beta + r\delta\sum_{i=1}^m a_iv_i(x) + rs\delta^2 - rs\delta^2
$$

把包含 $s$ 与 $r$ 的项分别组合：
$$
=\left(\alpha s\delta + \sum_{i=1}^m a_iu_i(x) s\delta + rs\delta^2\right) + \left(r\delta\beta + r\delta\sum_{i=1}^m a_iv_i(x) + rs\delta^2\right) - rs\delta^2
$$

分别提取 $s\delta$ 与 $r\delta$：
$$
=\underbrace{\left(\alpha+ \sum_{i=1}^m a_iu_i(x) + r\delta\right)s\delta}_{As\delta} + \underbrace{\left(\beta + \sum_{i=1}^m a_iv_i(x) + s\delta\right)r\delta}_{Br\delta} - rs\delta^2
$$

代入 $A$ 与 $B$：
$$
=As\delta + Br\delta - rs\delta^2
$$

因此，最终等式为：

$$
\left(\alpha + \sum_{i=1}^m a_iu_i(x) + r\delta\right)\left(\beta + \sum_{i=1}^m a_iv_i(x) + s\delta\right)=\alpha\beta+\sum_{i=1}^m a_i(\alpha v_i(x) + \beta u_i(x)+w_i(x)) + h(x)t(x) + As\delta + Br\delta - rs\delta^2
$$

现在将它拆分为公开部分与私有部分：

$$
\left(\alpha + \sum_{i=1}^m a_iu_i(x) + r\delta\right)\left(\beta + \sum_{i=1}^m a_iv_i(x) + s\delta\right)=\alpha\beta+\underbrace{\sum_{i=1}^\ell a_i(\alpha v_i(x) + \beta u_i(x)+w_i(x))}_\text{公开} + \underbrace{\sum_{i=\ell+1}^m a_i(\alpha v_i(x) + \beta u_i(x)+w_i(x)) + h(x)t(x) + As\delta + Br\delta - rs\delta^2}_\text{私有}
$$

我们希望分别用 $\gamma$ 和 $\delta$ 隔离公开部分与私有部分：

$$
(\alpha + \sum_{i=1}^m a_iu_i(x) + r\delta)(\beta + \sum_{i=1}^m a_iv_i(x) + s\delta)=\alpha\beta+\gamma\frac{\sum_{i=1}^\ell a_i(\alpha v_i(x) + \beta u_i(x)+w_i(x))}{\gamma} + \delta\frac{\sum_{i=\ell+1}^m a_i(\alpha v_i(x) + \beta u_i(x)+w_i(x)) + h(x)t(x)}{\delta} + As\delta + Br\delta - rs\delta^2
$$

其中一些项的 $\delta$ 可以约去：

$$
(\alpha + \sum_{i=1}^m a_iu_i(x) + r\delta)(\beta + \sum_{i=1}^m a_iv_i(x) + s\delta)=\alpha\beta+\gamma\frac{\sum_{i=1}^\ell a_i(\alpha v_i(x) + \beta u_i(x)+w_i(x))}{\gamma} + \delta\left(\frac{\sum_{i=\ell+1}^m a_i(\alpha v_i(x) + \beta u_i(x)+w_i(x)) + h(x)t(x)}{\delta} + As + Br - rs\delta\right)
$$

现在把这个等式拆分为验证者部分与证明者部分。方框内各项属于验证者部分，带下括号的各项由证明者提供：

$$
\underbrace{(\alpha + \sum_{i=1}^m a_iu_i(x) + r\delta)}_{[A]_1}\underbrace{(\beta + \sum_{i=1}^m a_iv_i(x) + s\delta)}_{[B]_2}=\boxed{\alpha\beta}+\boxed{\gamma}\boxed{\frac{\sum_{i=1}^\ell a_i(\alpha v_i(x) + \beta u_i(x)+w_i(x))}{\gamma}} + \boxed{\delta}\underbrace{\left(\frac{\sum_{i=\ell+1}^m a_i(\alpha v_i(x) + \beta u_i(x)+w_i(x)) + h(x)t(x)}{\delta} + As + Br - rs\delta\right)}_{[C]_1}
$$

## Groth16 证明算法
现在可以完整给出 Groth16 算法。可信设置和验证步骤与前面引入 $\gamma$、$\delta$ 的示例保持不变；只有证明者的计算发生变化，以纳入 $r$ 和 $s$。

### 可信设置

$$
\begin{align*}
\alpha,\beta,\tau,\gamma,\delta &\leftarrow \text{随机标量}\\
[\tau^{n-1}G_1,\tau^{n-2}G_1,\dots,\tau G_1,G_1] &\leftarrow \text{SRS，所属群为 } \mathbb{G}_1\\
[\tau^{n-1}G_2,\tau^{n-2}G_2,\dots,\tau G_2,G_2] &\leftarrow \text{SRS，所属群为 } \mathbb{G}_2\\
\left[\frac{\tau^{n-2}t(\tau)}{\delta}G_1,\frac{\tau^{n-3}t(\tau)}{\delta}G_1,\dots,\frac{\tau t(\tau)}{\delta}G_1,
\frac{t(\tau)}{\delta}G_1\right] &\leftarrow \text{用于该项的 SRS：}h(\tau)t(\tau)\\
\\
&\text{见证的公开部分}\\
[\Psi_1]_1 &= \frac{\alpha v_1(\tau) + \beta u_1(\tau) + w_1(\tau)}{\gamma}G_1\\
[\Psi_2]_1 &= \frac{\alpha v_2(\tau) + \beta u_2(\tau) + w_2(\tau)}{\gamma}G_1\\
&\vdots\\
[\Psi_\ell]_1 &= \frac{\alpha v_\ell(\tau) + \beta u_\ell(\tau) + w_\ell(\tau)}{\gamma}G_1\\
\\
&\text{见证的私有部分}\\
[\Psi_{\ell+1}]_1 &= \frac{\alpha v_{\ell+1}(\tau) + \beta u_{\ell+1}(\tau) + w_{\ell+1}(\tau)}{\delta}G_1\\
[\Psi_{\ell+2}]_1 &= \frac{\alpha v_{\ell+2}(\tau) + \beta u_{\ell+2}(\tau) + w_{\ell+2}(\tau)}{\delta}G_1\\
&\vdots\\
[\Psi_{m}]_1 &= \frac{\alpha v_{m}(\tau) + \beta u_{m}(\tau) + w_{m}(\tau)}{\delta}G_1\\
\end{align*}
$$

可信设置发布：
$$
([\alpha]_1,[\beta]_1[\beta]_2,[\gamma]_2,[\delta]_1[\delta]_2,\text{srs}_{G_1},\text{srs}_{G_2},\text{srs for }h(\tau)t(\tau),[\Psi_1]_1,[\Psi_2]_1,\dots,[\Psi_m]_1)
$$

### 证明者步骤
证明者持有见证 $\mathbf{a}$，并生成随机标量 $r$ 与 $s$。
$$
\begin{align*}
[A]_1 &= [\alpha]_1 + \sum_{i=1}^m a_iu_i(\tau)+r[\delta]_1\\
[B]_1 &= [\beta]_1 + \sum_{i=1}^m a_iv_i(\tau)+s[\delta]_1\\
[B]_2 &= [\beta]_2 + \sum_{i=1}^m a_iv_i(\tau)+s[\delta]_2\\
[C]_1 &= \sum_{i=\ell+1}^m a_i[\Psi_i]_1 + h(\tau)t(\tau)+[A]_1s+[B]_1r-rs[\delta]_1\\
\end{align*}
$$

证明者发布 $([A]_1, [B]_2, [C]_1, [a_1,...,a_\ell])$。

### 验证者步骤
验证者检查：

$$
\begin{align*}
[X]_1&=\sum_{i=1}^\ell a_i\Psi_i\\
[A]_1\bullet[B]_2 &\stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + [X]_1\bullet [\gamma]_2 + [C]_1\bullet [\delta]_2
\end{align*}
$$

## 在 Solidity 中验证 Groth16
至此，你已经具备理解 Solidity 证明验证代码所需的知识。这里是 [Tornado Cash 的证明验证代码](https://github.com/tornadocash/tornado-core/blob/master/contracts/Verifier.sol#L192)。建议读者仔细阅读其源代码。如果熟悉 Solidity 汇编编程，由于源代码中的变量名与本文一致，理解起来并不困难。


此外，还有支持 [Solana 上 Groth16](https://lib.rs/crates/groth16-solana) 的库。

## 需要注意的安全问题
### Groth16 具有可塑性
Groth16 证明具有可塑性。给定一个有效证明

$([A]_1, [B]_2, [C]_1)$，攻击者可以分别计算 $[A]_1$ 和 $[B]_2$ 的点取负，并提交一个新证明 $([A']_1, [B']_2, [C]_1)$，其中 $[A']_1 = \mathsf{neg}([A]_1)$，$[B']_2 = \mathsf{neg}([B]_2)$。

要理解为什么 $[A]_1\bullet[B]_2 = [A']_1\bullet[B']_2$，请看以下代码：
```python
from py_ecc.bn128 import G1, G2, multiply, neg, eq, pairing

# chosen arbitrarily
x = 10
y = 100
A = multiply(G1, x)
B = multiply(G2, y)

A_p = neg(A)
B_p = neg(B)

assert eq(pairing(B, A), pairing(B_p, A_p))
```

直观地说，攻击者将 $A$ 和 $B$ 都乘以 $-1$，而在配对中 $(-1)\times(-1)$ 会相互抵消。

因此，如果验证公式接受：
$$
[A]_1\bullet[B]_2 \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + [X]_1\bullet [\gamma]_2 + [C]_1\bullet [\delta]_2
$$

那么它也会接受：

$$
\mathsf{neg}([A]_1)\bullet\mathsf{neg}([B]_2) \stackrel{?}= [\alpha]_1 \bullet [\beta]_2 + [X]_1\bullet [\gamma]_2 + [C]_1\bullet [\delta]_2
$$

下一节将说明如何防御这种攻击。

这篇[文章](https://medium.com/@cryptofairy/exploring-vulnerabilities-the-zksnark-malleability-attack-on-the-groth16-protocol-8c80d13751c5)给出了该攻击的概念验证。

### 证明者可以为同一个见证创建任意多个证明
严格来说，这并不是“安全问题”——它是实现零知识性所必需的。不过，应用需要一种机制来跟踪哪些事实已经证明，不能依赖证明的唯一性来实现这一点。

## 通过 RareSkills 深入学习
我们能够免费发布此类资料，有赖于学员的持续支持。欢迎报名我们的[零知识训练营](https://www.rareskills.io/zk-bootcamp)、[Web3 训练营](https://www.rareskills.io/web3-blockchain-bootcamps)，或在 [RareTalent](https://www.raretalent.xyz) 上求职。

*最初发布于 2023 年 8 月 31 日*
