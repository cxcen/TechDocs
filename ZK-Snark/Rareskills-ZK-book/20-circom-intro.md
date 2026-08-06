# Circom 零知识电路简介

Circom 是一种编程语言，用于创建秩一约束系统（Rank-1 Constraint System，R1CS），并填充 R1CS 的见证向量。

R1CS 格式之所以值得关注，是因为它适合用于构建 SNARK，尤其是 [Groth16](https://www.rareskills.io/post/groth16)。SNARK 能够实现可验证计算：我们可以证明某项计算的正确性，而验证者确认结果所需的计算量少于亲自执行该计算所需的计算量。还可以在不泄露底层数据的情况下生成证明，此时称为 ZK-SNARK。

本书第一部分侧重于证明某个见证对给定 R1CS 有效。本部分则介绍如何用程序生成 R1CS，以及如何设计约束系统来建模虚拟机、密码学哈希函数等现实算法。

## 前置知识

读者应已熟悉 ZK Book 中的以下章节：

- [https://rareskills.io/post/p-vs-np](https://rareskills.io/post/p-vs-np)
- [https://rareskills.io/post/arithmetic-circuit](https://rareskills.io/post/arithmetic-circuit)
- [https://rareskills.io/post/finite-fields](https://rareskills.io/post/finite-fields)
- [https://rareskills.io/post/rank-1-constraint-system](https://rareskills.io/post/rank-1-constraint-system)

后文假定读者知道 R1CS 是什么以及它表示什么，上述四章对此有完整说明。

使用 Circom 并不要求完全理解 ZK 背后的数学，但有些基本原理必须真正掌握，否则 Circom 很难讲得通。

不过，如果读者认真考虑从事 ZK 相关工作，学习 ZK 基础仍然必不可少。强烈建议通读 [ZK Book](rareskills.io/zk-book) 的前两个部分，并从头构建 Groth16 证明系统，以巩固所学内容。

如果目标只是快速理解 ZK 应用，则建议先阅读上面列出的四章，再继续本部分。

## Circom 为何出现

Circom 旨在解决 SNARK 约束系统开发中的两个主要问题：

1. 手工设计约束系统既繁琐又容易出错，面对大规模或重复约束时尤其如此。
2. 填充见证同样困难，需要手工计算本可由程序推导的中间值。

因此，Circom 一方面简化约束设计，另一方面自动填充见证。

### 1. 设计约束系统很繁琐

手工设计一组正确约束，再将其转换成 R1CS，既繁琐又容易出错。Circom 可以通过程序生成约束，从而降低这项工作的难度。

例如，若要表达 `x` 只能取 $\set{1,2,3}$ 中的值，可以使用约束：

$$
0 === (x - 1) (x - 2) (x - 3)
$$

然而，一条 R1CS 约束只能包含一次非常数乘法，因此必须把上述约束拆成两条：

$$
\begin{align*}
s &=== (x - 1)(x - 2) &&= x * x - 3 * x + 2\\
0 &=== s(x - 3) &&= x * s - 3 * s
\end{align*}
$$

对于小型系统，手工转换尚可应付。但如果要为 100 甚至 1,000 个变量创建这种约束，手工处理会非常麻烦。当存在成千上万条相似约束时，更合适的方式是为约束创建“模板”，再通过循环生成。Circom 正允许我们用程序完成这些工作。

例如，假设要约束 1,000 个变量只能取 $\set{0,1}$ 中的值。Circom 可以在循环中生成相应约束：

```solidity
template Constrain1000Example() {
  signal input in[1000];

  for (var i = 0; i < 1000; i++) {
    0 === in[i] * (in[i] - 1);
  }
}

component main = Constrain1000Example();
```

后续章节会进一步解释语法。这里的核心思想是：定义约束 `0 === in[i] * (in[i] - 1)`，再重复生成 1,000 次。

### 2. 填充见证很繁琐

在 ZK 语境中，见证是对变量的一组赋值，并且这组赋值满足算术电路中的全部约束。

正如[算术电路](https://www.rareskills.io/post/arithmetic-circuit)一章所述，要证明一个数小于另一个数，需要先把两个数转换成二进制。原因是有限域中的数会回绕，因此“大于”在有限域中没有通常的含义。

假设 $x$ 可以用四位表示，要用二进制表达 $x$，它必须满足以下约束：

$$
\begin{align*}
x&===b_0+2b_1+4b_2+8b_3\\
0&===b_0(b_0 - 1)\\
0&===b_1(b_1 - 1)\\
0&===b_2(b_2 - 1)\\
0&===b_3(b_3 - 1)
\end{align*}
$$

这里，$b_0$ 是最低有效位，$b_3$ 是最高有效位。除 $x$ 本身外，证明者还必须提供 $x$ 的二进制位 $b_0, b_1, b_2, b_3$。

这样一来，证明 $x$ 是四位数就繁琐了五倍：除了 $x$，还要提供 $x$ 的二进制表示，尽管这些值可以直接、确定性地推导出来。Circom 会自动完成这个过程，并允许我们根据其他变量编写代码来填充见证中的变量。例如，可以用以下 Circom 代码填充二进制变量（这段代码缺少一些必要的安全措施，请勿直接照搬）：

```jsx
b_0 <-- x & 1;        // get the first bit of x via bitmask
b_1 <-- (x >> 1) & 1; // get the second bit of x
b_2 <-- (x >> 2) & 1; // get the third bit of x
b_3 <-- (x >> 3) & 1; // get the fourth bit of x
```

上面的代码会*生成*见证，但不会为以下公式创建约束：

$$
\begin{align*}
x&===b_0+2b_1+4b_2+8b_3\\
0&===b_0(b_0 - 1)\\
0&===b_1(b_1 - 1)\\
0&===b_2(b_2 - 1)\\
0&===b_3(b_3 - 1)
\end{align*}
$$

把上述电路写成 Circom 如下（稍后会进一步解释语法）：

```jsx
template BinaryConstraint() {

  // assign the values to b_0,...,b_3
  x === b_0 + 2*b_1 + 4*b_2 + 8*b_3;
  0 === b_0*(b_0 - 1);
  0 === b_1*(b_1 - 1);
  0 === b_2*(b_2 - 1);
  0 === b_3*(b_3 - 1);
}
```

Circom 的一大便利之处在于，其代码与算术电路中的数学表达式很相似，因此很容易把方程组转换成 Circom。

具体思路是：不再向电路提供 $(x,b_0,b_1,b_2,b_3)$，而只提供 $x$。Circom 会替我们计算各个二进制位，再用计算结果填充约束。

除了自动生成约束，Circom 还通过“赋值并约束”运算符 `<==` 改进了见证填充过程。

## Circom 中“赋值并约束”运算符 `<==` 的优势

Circom 使用“赋值并约束”运算符 `<==` 进一步简化见证填充。假设有以下约束：

```bash
z === x * y
```

如果已经提供 `x` 和 `y` 的值，还要求提供 `z` 会有些麻烦，因为 `z` 只有一个可能的解。
在 Circom 中，可以改用 `<==`：

```bash
z <== x * y
```

这样就不再需要把变量 `z` 作为输入提供：Circom 会替我们填充它，并在电路的其余部分将其值约束为 $x\cdot y$。

因此，Circom 免去了用户显式提供见证中每个元素的麻烦，这是它易用性的一大优势。

## Circom 既是 DSL，也是编程语言

Circom 编程最容易令人困惑之处在于，它既是一种类似 JavaScript 的编程语言，也是一种可编译成 R1CS 的领域特定语言（Domain-Specific Language，DSL）。从这个角度看，它有点像 Solidity：Solidity 可以通过转移 Ether 改变底层区块链状态，也可以像普通编程语言一样运行。Circom 中“编程语言”的部分用于辅助自动填充见证。不过，对初学者而言，哪些 Circom 代码会影响底层 R1CS，并不总是一目了然。

例如，下面是一段有效的 Circom 代码，用于计算一个数的幂：

```jsx
function power(base, exp) {
  return base ** exp;
}

template Power() {
  signal input base;
  signal input exp;
  signal output out;

  out <-- power(base, exp);
}

component main = Power();

/* INPUT = {
  "base": "3",
  "exp": "2"
} */
```

但是，上述代码不会生成任何约束，因此不能用于证明任何内容。稍后会看到，`<--` 运算符只负责生成见证，不会生成约束。

## 为什么要学习 Circom

Circom 是最早出现的一批 ZK 领域特定语言之一，拥有最多可供学习的**库**和**项目**，也经历了充分的实战检验。

我们认为，如果先学习 Circom，再学习 [Halo2](https://github.com/privacy-scaling-explorations/halo2)、[Plonky3](https://github.com/Plonky3/Plonky3) 等较新的 ZK DSL，会容易得多。因此，本书先从 Circom 开始。

为了说明这一点，可以看看 [Halo2 中计算 Fibonacci 数列的代码](https://github.com/Divide-By-0/halo2-examples/blob/master/src/fibonacci/example1.rs)，以及 [Plonky3 中计算 Fibonacci 数列的代码](https://github.com/BrianSeong99/Plonky3_Fibonacci/blob/master/src/main.rs)。粗略浏览这些示例，应该足以看出它们并不是初学者最合适的起点。下面则是证明 `out` 为正确第 n 个 Fibonacci 数的 Circom 代码，相比之下容易理解得多：

```jsx
pragma circom 2.1.6;

// proves `out` is the nth
// fibonnaci number
template Fibonacci(n) {
  var offset = n + 1;
  assert(n > 2);

  signal fib[offset];
  signal output out;

  fib[0] <== 0;
  fib[1] <== 1;

  for (var i = 2; i < offset; i++) {
    fib[i] <== fib[i-1] + fib[i - 2];
  }

  out <== fib[n];
}

// 5th fibonnaci number is 5
// 0 1 1 2 3 5
component main = Fibonacci(5);
```

相比之下，Circom 的学习曲线较为平缓，更适合刚进入 ZK 开发领域的初学者。

## Noir、Cairo 和 Leo 不是已经抽象掉约束编写了吗？

可以使用 Noir、Cairo、Leo 等类 Rust 语言，为 ZK 区块链或 Layer 2 编写智能合约。这些语言旨在向程序员“隐藏”约束生成过程。如果目标只是为这些区块链开发应用，并不一定需要理解 ZK 约束的底层工作方式。

不过，认真从事 Solidity 开发的人通常都对以太坊虚拟机（EVM）有相当了解，也能编写基础汇编。理解幕后机制有助于写出更高效的代码，本部分正是为了实现这一目标。

此外，底层 ZK 执行模型会使这些执行环境产生许多特有缺陷。要理解哪些内容真正私密、控制流存在哪些限制、使用域时常见哪些错误，或者要安全使用 Noir 中的无约束函数和 [o1js](https://docs.minaprotocol.com/zkapps/o1js) 中的自定义约束，都需要具备底层知识。

### 本系列的目标

高级 ZK 语言并没有让约束编写过时；恰恰相反，它们增加了对真正理解底层原理的专家的需求。本部分旨在帮助高级开发者和安全审计人员入门，使他们能够开发和保护这些高级 ZK 语言所依赖的底层区块链、虚拟机与编译器环境。

## 本部分的组织方式

本部分分为两个主要部分：

1) 第一部分介绍 Circom 语法，具体包括如何编写约束，以及如何让 Circom 自动填充大部分见证值。

2) 第二部分介绍如何为一般 ZK 应用设计约束。示例使用 Circom，但相关内容也适用于 Halo2、Plonky3 等其他 ZK DSL。

期间还会穿插介绍 ZK 应用中的安全问题。

## 学习不仅需要阅读，也需要练习

许多章节包含明确的练习，或留有一些“交给读者完成”的未完成代码。**如果亲自解决这些问题，学习效果会好得多**。这些问题用于复习刚刚读过的内容并巩固学习。只要正确理解正文，它们就不要求任何特殊“洞见”或“巧思”。我们希望读完材料后，文末练习会显得比较“显然”（如果并非如此，请在练习仓库中提交 issue 或 pull request）。

## 安装 Circom

Circom 的安装说明见：[https://docs.circom.io/getting-started/installation/#installing-dependencies](https://docs.circom.io/getting-started/installation/#installing-dependencies)

也可以使用 Circom 在线 IDE：[https://zkrepl.dev/](https://zkrepl.dev/)

## 补充说明：Circom 中的 Plonk 与 Groth16

熟悉 Plonk 证明系统的读者需要注意：Plonk 证明系统和 Groth16 证明系统使用相同的电路。

Groth16 允许一条约束中包含任意数量的加法，但只允许一次非常数乘法（R1CS 的每一行恰好有一次乘法）。相比之下，Plonk 的每条约束只允许一次乘法或一次加法，二者不能同时出现。随着后续探索 Circom，“每条约束一次乘法”的限制会越来越明显。

不过，兼容 Groth16 的 Circom 电路也可以用于 Plonk。以 R1CS 为输入的 snarkjs 库可以按开发者的需要，将其转换为 Plonk 约束系统。

因此，无论目标底层证明系统是 Groth16 还是 Plonk，Circom 都与之无关。只要电路兼容 Groth16，无需开发者额外修改，也可以兼容 Plonk。

## 作者与致谢

Calnix 撰写了本书第一部分，并对全书结构产生了重要影响。欢迎[在 X 上关注 Calnix](https://x.com/cal_nix)，也可以向他表达感谢。

感谢 [Veridise](https://veridise.com)、[Privacy Scaling Explorations](https://pse.dev/en)、来自 [zkSecurity](https://www.zksecurity.xyz) 的 Marco Besier，以及 [Chainlight](https://chainlight.io) 对本文提供的细致审阅。
