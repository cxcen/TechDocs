# 先指示，再约束

如果要表达“`x` 可以等于 5 或 6”，只需使用下面的约束：

```jsx
(x - 6) * (x - 5) === 0
```

但假设我们要表达“`x` 小于 5，或者 `x` 大于 17”。此时不能直接把两个条件组合起来，因为当 `x` 小于 5 时，它会违反“`x` 大于 17”的约束，反之亦然。

解决办法是为不同条件创建*指示*信号（例如指示 `x` 小于 5，或大于 17），然后再对这些指示信号施加约束。

## circomlib 比较器库

[circomlib 比较器库](https://github.com/iden3/circomlib/blob/master/circuits/comparators.circom)包含一个 `LessThan` 组件：当 `in[0]` 小于 `in[1]` 时，它返回 1，否则返回 0。[算术电路](https://www.rareskills.io/post/arithmetic-circuit)一章介绍了这个组件的工作原理。简而言之，假设 `x` 和 `y` 最大都是 3 位数。如果计算 `z = 2^3 + (x - y)`，那么 `x` 小于 `y` 时，`z` 就会小于 2^3，反之亦然（2^3 = 8）。由于 `z` 是一个 4 位数，只查看最高有效位，就能高效判断 `z` 是否小于 2^3。2^3 的二进制表示是 `1000₂`。所有大于或等于 2^3 的 4 位数，其最高有效位都是 1；所有小于 2^3 的 4 位数，其最高有效位都是 0。

| 数值 | 二进制表示 | 是否大于或等于 2^3 |
| --- | --- | --- |
| 15 | 1111 | 是 |
| 14 | 1110 | 是 |
| 13 | 1101 | 是 |
| … |  |  |
| 10 | 1010 | 是 |
| 9 | 1001 | 是 |
| 8 (2^3) | 1000 | 是 |
| 7 | 0111 | 否 |
| 6 | 0110 | 否 |
| … |  |  |
| 2 | 0010 | 否 |
| 1 | 0001 | 否 |
| 0 | 0000 | 否 |

对于一般的 n 位数，可以通过检查最高有效位是否为 1，判断 `x` 是否大于或等于 2^n。因此可以推广得到：如果 `x` 和 `y` 都是 `n-1` 位数，要判断 `x < y` 是否成立，只需检查 `2^(n-1) + (x - y)` 的最高有效位是否为 1。

下面是使用 LessThan 模板的最小示例：

```jsx
include "circomlib/comparators.circom";

template Example () {
  signal input a;
  signal input b;
  signal output out;

  // 252 will be explained in the next section
  out <== LessThan(252)([a, b]);
}

component main = Example();

/* INPUT = {
  "a": "9",
  "b": "10"
} */
```

## 252 从何而来

[有限域](https://www.rareskills.io/post/finite-fields)中的数（Circom 使用的正是有限域）不能按“小于”或“大于”相互比较，因为不等式通常遵循的代数规律在有限域中并不成立。

例如，如果 $x > y$，且 $c$ 为正数，那么 $x+c>y+c$ 应该始终成立。然而，这在有限域中并不成立。我们可以选取一个 $c$，使其成为 $x$ 的加法逆元，即 $x + c=0\mod p$。这样最终会得到“0 大于一个非零数”这种荒谬结论。例如，当 $p = 7$、$x=2$、$y=1$ 时，确有 $x>y$。但如果分别给 $x$ 和 $y$ 加上 $5$，就会得到 $0>1$。

252 指定了 `LessThan` 组件中的位数，以限制 `x` 和 `y` 可以有多大，从而使比较有意义（上一节以 4 位为例）。

Circom 的有限域最多可以容纳 253 位的数。出于 [Alias Check](https://www.rareskills.io/post/circom-aliascheck) 一章所讨论的安全原因，我们不应将域元素转换为能够编码大于该有限域的数的二进制表示。因此，Circom 不允许以超过 252 位的参数实例化比较模板（[源代码](https://github.com/iden3/circomlib/blob/252f8130105a66c8ae8b4a23c7f5662e17458f3f/circuits/comparators.circom#L90)）。

不过请回想，对于 `LessThan(n)`，我们需要计算 `z = 2^n + (x - y)`，而 `2^n` 必须比 `x` 或 `y` 多一位。因此，`x` 和 `y` 最大只能是 $2^{n-1}$ 位。由于 Circom 支持的数最大为 253 位，`x` 和 `y` 最多只能是 252 位。

## x 小于 5，或者 x 大于 17

好在 circomlib 库会替我们完成大部分工作。我们将使用 LessThan 和 GreaterThan 组件的输出信号，*指示* x 是小于 5 还是大于 17。

然后使用 OR 组件（其底层就是 `out <== a + b - a * b`）*约束*二者中至少有一个等于 1。

```jsx
pragma circom 2.1.6;

include "circomlib/comparators.circom";
include "circomlib/gates.circom";

template DisjointExample1() {
  signal input x;

  signal indicator1;
  signal indicator2;

  indicator1 <== LessThan(252)([x, 5]);
  indicator2 <== GreaterThan(252)([x, 17]);

  component or = OR();
  or.a <== indicator1;
  or.b <== indicator2;

  or.out === 1;
}

component main = DisjointExample1();

/* INPUT = {
  "x": "18"
} */
```

**约束 `or.out === 1;` 非常重要，否则当 `indicator1` 和 `indicator2` 都为零时，电路也会接受这些信号。本章末尾会更详细地讨论这一点。**

### 简化代码

可以隐式使用指示信号来简化上面的代码，如下所示：

```jsx
pragma circom 2.1.6;

include "circomlib/comparators.circom";
include "circomlib/gates.circom";

template DisjointExample1() {
  signal input x;

  component or = OR();
  or.a <== LessThan(252)([x, 5]);
  or.b <== GreaterThan(252)([x, 17]);
  or.out === 1;
}

component main = DisjointExample1();

/* INPUT = {
  "x": "18"
} */
```

## 并非 x < 100 且 y < 100

要表达上面“`x` < 100 且 `y` < 100 并非同时成立”的情况，可以使用 NAND 门。除两个输入都为 1 的情况外，NAND 门对所有输入组合都返回 1，其真值表如下：

| a | b | out |
| --- | --- | --- |
| 0 | 0 | 1 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

因此，可以分别创建指示 `x` 小于 100 和指示 `y` 小于 100 的信号，然后在它们之间施加 NAND 关系约束。

```jsx
pragma circom 2.1.6;

include "circomlib/comparators.circom";
include "circomlib/gates.circom";

template DisjointExample2() {
  signal input x;
  signal input y;

  component nand = NAND();
  nand.a <== LessThan(252)([x, 100]);
  nand.b <== LessThan(252)([y, 100]);
  nand.out === 1;
}

component main = DisjointExample2();

/* INPUT = {
  "x": "18",
  "y": "100"
} */
```

## k 至少大于 x、y、z 中的两个

在这个例子中，我们要表达：`k` 大于 `x` 和 `y`，或者 `k` 大于 `x` 和 `z`，或者 `k` 大于 `y` 和 `z`。`k` 也可以同时大于 `x`、`y` 和 `z`，但这不是必要条件。

直接写出上面如此复杂的逻辑表达式会很冗长，所以更简单的做法是统计 `k` 大于多少个信号，再检查这个数量是否不小于 2。

```jsx
pragma circom 2.1.6;

include "circomlib/comparators.circom";
include "circomlib/gates.circom";

template DisjointExample3(n) {
  signal input k;
  signal input x;
  signal input y;
  signal input z;

  signal totalGreaterThan;

  signal greaterThanX;
  signal greaterThanY;
  signal greaterThanZ;

  greaterThanX <== GreaterThan(252)([k, x]);
  greaterThanY <== GreaterThan(252)([k, y]);
  greaterThanZ <== GreaterThan(252)([k, z]);

  totalGreaterThan = greaterThanX + greaterThanY + greaterThanZ;

  signal atLeastTwo;
  atLeastTwo <== GreaterEqThan(252)([totalGreaterThan, 2]);
  atLeastTwo === 1;
}

component main = DisjointExample3();

/* INPUT = {
  "k": 20
  "x": 18,
  "y": 100,
  "z": 10
} */
```

## 不要忘记约束组件的输出！

有时开发者会忘记约束组件的输出，**这可能造成严重的安全漏洞！**例如，下面的代码看起来像是在强制 `x` 和 `y` 都等于 `1`，事实却并非如此。`x` 和 `y` 可以为零（或任意其他值）。如果 `x` 和 `y` 都为零，AND 门的*输出*就是零，但没有任何约束要求这个输出必须是什么值。

```jsx
template MissingConstraint1() {
  signal input x;
  signal input y;

  component and = AND();
  and.a <== x;
  and.b <== y;

  // and.out is not constrained, so x and y can have any values!
}
```

同样，下面的电路不会强制 `x` 小于 100。当 `x` 小于 100 时，LessThan 的输出为 1，但代码没有约束该输出，以确保这个条件确实成立。

```jsx
template MissingConstraint2() {
  signal input x;

  component lt = LessThan(252);
  lt.in[0] <== x;
  lt.in[1] <== 100;

  // x could be ≥ 100 since lt.out is allowed to be 0 or any other arbitrary value
}
```
