# ZK 友好型哈希函数

ZK 友好型哈希函数，是指相比传统密码学哈希函数，证明和验证所需约束少得多的哈希函数。

SHA256 或 keccak256 等哈希函数会大量使用 XOR、比特旋转等按位运算符。要证明 XOR 或比特旋转执行正确，需要把一个数表示成 32 个比特，也就是使用 32 个独立信号。传统哈希函数的默认机器字宽度是 32 位，因此对这种数据类型进行运算需要 32 个信号。

ZK 友好型哈希函数使用原生域元素作为默认数据类型，并避开会把域元素分解成比特的操作。域元素的原生运算只有加法和乘法，因此 ZK 友好型哈希函数只能使用模加法和模乘法。

我们关心哈希函数的以下性质：

1. **原像抗性**——给定一个哈希输出，计算出输入应当不可行。
2. **抗碰撞性**——给定一组输入与输出，找到另一个产生相同输出的输入，在计算上应当不可行。
3. **伪随机性**——输出看起来应当是随机的，输入与输出之间不应存在统计关系。

我们将从较高层次介绍两种最流行的 ZK 友好型哈希函数：Minimal Multiplicative Complexity（MiMC）和 Poseidon。不过，分析它们为什么安全超出了本文范围。事实上，尽管这些哈希函数是同类方案中经受实践检验最多的，其安全性仍在一定程度上是一个开放问题。

## MiMC

这个哈希函数接收单个域元素作为输入，也输出单个域元素。

MiMC 会初始化 91 个随机常量并存入数组 C。这些常量可以用确定且透明的方式计算，例如对字符串“MiMC”连续执行 91 次 SHA256 哈希，每次哈希结果都用作一个随机数。这些常量固定、公开，并为各方所知。按照惯例，`C[0] = 0`。然后，MiMC 接收域元素 $t_0$ 作为输入，并迭代计算：

$$
\begin{align*}
\texttt{let } u &= k+t_i+C_i\\
t_{i+1} &= i\space<90\space\space?\space\space u^e : \space u^e + k
\end{align*}
$$

其中 $e$ 是某个固定指数，通常为 3 或 7。$k$ 是一个被设为 0 的常量（为什么要提供输入 $k$，却又将它设为零，稍后会讨论）。

要使 MiMC 安全，必须满足 `gcd(e, p - 1) == 1`，其中 `gcd` 是最大公约数。对于 Circom 的默认有限域大小，`gcd(3, p - 1) ≠ 1`，但 `gcd(7, p - 1) = 1`。

```python
from math import gcd
p = 21888242871839275222246405745257275088548364400416034343698204186575808495617
gcd(3, p - 1)
# 3
gcd(7, p - 1)
# 1
```

因此，circomlib 提供 MiMC7 作为哈希函数（其中 7 是指数）。不过，使用其他有限域大小的库可能会采用 `e = 3`（要了解原因，请参阅文末链接的资料）。

下面是使用单个输入调用 MiMC7 的最小示例：

```jsx
include "circomlib/mimc.circom";

template ExampleMiMC() {
  signal input a;
  signal output out;

  component hash = MiMC7(91);

  hash.x_in <== a;
  hash.k <== 0;
  out <== hash.out;
}
```

如果希望向哈希函数传入多个域元素并输出单个域元素，可以采用以下方法：

1. 对第一个输入元素进行哈希。
2. 该哈希的输出成为下一个哈希的 `k` 值
3. 下一个哈希的输入是输入数据的下一部分。

可以用下图表示：

![MultiMiMC 示意图](https://r2media.rareskills.io/ZKFriendlyHashFunction/multi-mimc.png)


MultiMiMC7 模板替我们完成了这项工作：

```jsx
include "circomlib/mimc.circom";

template ExampleMultiMiMC(n) {
  signal input in[n];
  signal output out;

  component hash = MultiMiMC7(n, 91);

  for (var i = 0; i < n; i++) {
    hash.in[i] <== in[i];
  }
  hash.k <== 0;

  out <== hash.out;
}
```

## Poseidon

Poseidon 与 MiMC 类似，但增加了矩阵乘法步骤。也就是说，如果输入是单个元素，就先将其扩展为 `[0, input]`，再让这个向量乘以一系列经过精心调整的 2 × 2 矩阵。这里的“精心调整”是指它们具备某种密码学性质，本文不作深入讨论。

因此，在 MiMC 中，单个元素会经历一系列加法和幂运算；而在 Poseidon 中，一个向量会经历一系列逐元素加法、矩阵乘法（产生相同维度的向量）和幂运算。

虽然矩阵乘法会增加约束，但它能在哈希中产生更强的“扩散”，因此 Poseidon 不需要像 MiMC 那样执行那么多轮。

下面是使用单个输入调用 Poseidon 的最小示例：

```jsx
include "circomlib/poseidon.circom";

template Example(n) {
  signal input in[n];
  signal output out;

  component hash = Poseidon(n);

  for (var i = 0; i < n; i++) {
    hash.inputs[0] <== in[i];
  }
  out <== hash.out;
}

component main = Example(1);

/* INPUT = {
  "in": [5]
} */
```

要使用多个输入信号，需要把 Poseidon 的模板参数 `n` 改为所需值，并提供大小正确的数组。

## Poseidon 与 MiMC 的性能对比

对于接收单个输入的哈希，MiMC 底层 R1CS 有 364 个约束：

![MiMC 约束](https://r2media.rareskills.io/ZKFriendlyHashFunction/mimc-constraints.png)

而 Poseidon 有 213 个：

![Poseidon 约束](https://r2media.rareskills.io/ZKFriendlyHashFunction/Poseidon-single.png)


现在比较两个输入时生成的约束数量。

对于 MiMC7，两个输入会让约束数量翻倍：

![多输入 MiMC 约束](https://r2media.rareskills.io/ZKFriendlyHashFunction/multi-mimc.png)

而 Poseidon 的约束数量几乎没有增加：

![多输入 Poseidon 约束](https://r2media.rareskills.io/ZKFriendlyHashFunction/multi-poseidon.png)

这种优异性能的主要原因是：与 MiMC 不同，Poseidon 不会对输入中的每个域元素重新执行一次哈希。它会使用更大的矩阵与输入向量相乘，从而对更大的输入进行哈希。circomlib 的 Poseidon 不支持超过 17 个域元素的输入。如果需要对大型数据集执行哈希，这可能成为问题。不过，构建 Merkle 树时，只需对两个输入进行哈希。

## 寻求更强的安全性

如前所述，人们对 Poseidon 和 MiMC 安全性质的理解，不如对 SHA-256 等历经充分实战检验的哈希函数那样深入。

还有一种 ZK 友好型哈希函数，其安全假设比 Poseidon 和 MiMC 更强，并且以椭圆曲线为基础。人们普遍认为，没有量子计算机就无法计算椭圆曲线点的离散对数。Pedersen 哈希是一种 ZK 友好型哈希函数，以椭圆曲线运算作为计算哈希的核心子程序。在电路中执行椭圆曲线算术不如 Poseidon 或 MiMC 高效，但仍比传统密码学哈希函数更高效。

## MiMC 与 Poseidon 漏洞赏金

目前，以太坊基金会提供了 [20,000 美元赏金](https://crypto.ethereum.org/bounties/mimc-hash-challenge)，用于寻找本文所述 MiMC 哈希某个变体的哈希碰撞。

以太坊基金会目前还通过 [Poseidon Cryptanalysis Initiative](https://www.poseidon-initiative.info) 资助 Poseidon 安全性研究。以太坊创始人曾[表示](https://x.com/VitalikButerin/status/1894681713613164888)，以太坊可能改用 Poseidon 作为首选哈希函数，以便使用 ZK 更容易地证明网络状态。

## 致谢

撰写本文时参考了 [Risc Zero](https://risczero.com) 的以下资料：

[https://www.youtube.com/watch?v=_MIxjDs70W8](https://www.youtube.com/watch?v=_MIxjDs70W8)
