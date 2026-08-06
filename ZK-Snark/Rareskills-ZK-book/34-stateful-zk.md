# ZK 中的有状态计算简介

执行幂、阶乘或 Fibonacci 数列等迭代计算时，我们需要在某个时点“停止计算”。

例如，计算 $x^7$ 时，需要将 $x$ 与自身相乘七次。然而，算术电路无法有条件地停止。由于电路大小固定（底层 R1CS 的行数固定，无法改变），电路必须足够大，能够涵盖我们关心的每个指数。

因此，解决办法是计算直到某个上限为止的所有可能值，这个上限要高于实际预计会计算到的值。然后使用 Quin Selector 选出所需的值。

本章将分别用阶乘和 Fibonacci 数列展示这种做法，并把幂运算留作练习。

可以把这些计算都理解为状态机：它按照固定的状态转移执行一定次数，而迭代次数在证明时确定，并没有固化在电路中。

这些数列在每一步都只有一种可能的计算方式（例如，把前两个状态相加，或者将前一个状态乘以某个数）。不过，如果在每个状态处加入条件分支，就具备了有状态计算所需的全部组件。

本章只展示状态变化方式唯一、但状态变化次数可变的例子。后续章节将介绍如何让状态转移本身也带有条件，也就是允许多种不同的状态转移。

## 阶乘示例

下面展示如何编写一个电路，证明我们正确计算了

$$
y =x!\pmod p
$$

其中 $!$ 表示阶乘，$p$ 是默认的有限域模数。

在 Python 等传统编程语言中，可以用下面的代码计算阶乘：

```python
def factorial_mod_p(x, p):
  if x == 0:
    return 1

  acc = 1
  for i in range(1, x+1):
    acc = (acc * i) % p

  return acc
```

但如果直接把上述代码翻译成 Circom，会遇到几个问题：

- Circom 不支持 if 语句，因此 `if x == 0: return 0` 这一行无法编译。
- Circom 不支持迭代次数未知的循环。由于 `x` 决定循环次数，这段代码同样无法编译。Circom 底层会编译为 R1CS，而 R1CS 必须具有固定大小，不能根据输入值改变规模。

为了让代码兼容 R1CS 这样的算术化表示，需要从零开始计算阶乘，一直计算到我们希望支持的某个上限。

例如，如果确定永远不需要计算超过 99 的阶乘，就必须计算从 0 到 99（含）的每个阶乘。如果要为 80 的阶乘创建证明，仍然需要计算从 0 到 99 的所有阶乘，但可以使用 Quin Selector 返回 80 对应的结果。

下面是一个没有 if 语句、循环长度固定的 Python 示例：

```python
def factorial_mod_p(x, p):

  assert x < 100
  # allocate the array
  ans = [0] * 100
  ans[0] = 1 # 0! = 1

  for i in range(1, 100):
      ans[i] = (ans[i-1] * i) % p

  return ans[x]
```

从某种意义上说，我们创建了一个长度为 100 的数组，并将每个索引位置填入该索引的阶乘。然后使用 Quin Selector “选出”所关心的阶乘。

将它翻译成 Circom 很直接：

```jsx
include "./node_modules/circomlib/circuits/multiplexer.circom";
include "./node_modules/circomlib/circuits/comparators.circom";

template factorial(n) {
  signal input in;
  signal output out;

  // precompute factorials from 0 to n
  signal factorials[n+1];

  // compute the factorials
  factorials[0] <== 1;
  for (var i = 1; i <= n; i++) {
    factorials[i] <== factorials[i - 1] * i;
  }

  // ensure that in < n
  signal inLTn;
  inLTn <== LessThan(252)([in, n]);
  inLTn === 1;

  // select the factorial of interest
  component mux = Multiplexer(1, n);
  mux.sel <== in;

  // assign factorials into the multiplexer
  for (var i = 0; i < n; i++) {
    mux.inp[i][0] <== factorials[i];
  }

  out <== mux.out[0];
}

component main = factorial(100);

/*
  INPUT = { "in": "3" }
*/
```

### 一种不安全的实现

许多刚接触 Circom 的工程师经常采用一种“直观”的解决方案，以避开二次约束相关的问题，并写出下面这样的代码：

```jsx
pragma circom 2.1.8;

include "./node_modules/circomlib/circuits/comparators.circom";

template factorial(n) {
  signal input in;
  signal output out;

  signal factorials[n + 1];

  // compute the factorials
  var acc = 1;
  for (var i = 1; i < n; i++) {
    acc = acc * i;
  }

  out <-- acc;
}

component main = factorial(100);
```

虽然 `out` 会得到正确答案，但代码完全没有 `<==` 或 `===` 运算符，这意味着电路中没有任何约束。

**在上面的代码中，程序员编写了能正确计算阶乘的代码，却没有对结果施加约束。**

## 模 p 的 Fibonacci 数列示例

在阶乘示例中，我们必须将 `factorials` 的第 0 项“写死”为 1，因为 0! = 1。在 Fibonacci 数列中，前两个数是 1，之后的每个数都是数列中前两个数之和。因此，在 Fibonacci 代码中，我们会写死前两个值，再计算其余值。

下面的电路计算模 p 意义下直到第 n 个数的 Fibonacci 数列，然后输出所关注的第“in”个 Fibonacci 数。

```jsx

include "./node_modules/circomlib/circuits/multiplexer.circom";
include "./node_modules/circomlib/circuits/comparators.circom";

template Fibonacci(n) {
  assert(n >= 2); // so we don't break the hardcoding

  signal input in; // compute the kth Fibonacci number
  signal output out;

  // precompute Fibonacci sequence from
  // 0 to n
  signal fib[n + 1];

  // compute the factorials
  fib[0] <== 1;
  fib[1] <== 1;

  for (var i = 2; i < n; i++) {
    fib[i] <== fib[i - 1] + fib[i - 2];
  }

  // ensure that in < n
  signal inLTn;
  inLTn <== LessThan(252)([in, n]);
  inLTn === 1;

  // select the fibonacci number
  // of interest
  component mux = Multiplexer(1, n);
  mux.sel <== in;

  // assign Fibonacci into
  // the Quin Selector
  for (var i = 0; i < n; i++) {
    mux.inp[i][0] <== fib[i];
  }

  out <== mux.out[0];
}

component main = Fibonacci(99);

/*
  INPUT = {"in": 5}
*/
```

与往常一样，务必显式约束 Fibonacci 数列的每次更新，不能只在一个不受约束的循环中计算结果。

### 练习：

完成 Circom Puzzles 中的 [pow 练习](https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/Power/pow.circom)。
