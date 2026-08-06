# Circom 中的条件语句

Circom 对 if 语句的使用限制非常严格，必须遵守以下规则：

- 不能使用信号改变 if 语句的行为。
- 不能在 if 语句内为信号赋值。

下面的示例电路同时违反了这两条规则：

```jsx
template Foo() {

  signal input in;
  signal input cond;

  signal output out;

  // if-statements cannot depend on
  // values not known at compile time
  if (in == 3) {
    // assigning a value inside an if-statement
    // whose value is unknown at compile time
    // is not allowed
    out <== 4;
  }
}
```

如果 if 语句不受任何信号影响，也不会影响任何信号，那么可以使用它。

实际上，这种语句并不属于底层的秩一约束系统（Rank-1 Constraint System，R1CS）。

例如，如果要计算列表中的最大值（但不生成约束），可以使用下面这种常见写法。由于其中不涉及信号，Circom 会接受它：

```jsx
var max;
for (var i = 0; i < n; i++) {
  if (arr[i] > max) {
    max = arr[i];
  }
}
```

这项计算不会创建任何约束，只是为了方便编程。

## Circom 中的分支

Circom 看起来似乎无法进行条件分支，但事实并非如此。要在 Circom 中创建条件分支，必须执行一条语句的所有分支：把“不需要的”分支乘以零，把“正确的”分支乘以一。

## 带分支的计算示例

假设我们要为下面的计算建模：

```python
def foo(x):

  if x == 5:
    out = 14
  elif x == 9:
    out = 22
  elif x == 10:
    out = 23
  else
    out = 45

  return out
```

由于 `x` 和 `out` 之间没有明显的数学关系，最好尽可能直接地为这个条件逻辑建模。可以用下面的数学形式描述它：

$$
\texttt{out} = \texttt{x\_eq\_5}\cdot14+\texttt{x\_eq\_9}\cdot22+\texttt{x\_eq\_10}\cdot23+\texttt{otherwise}\cdot45\\
$$

- `x_eq_5` 在 `x` 等于 5 时取 1，否则为零；这可以通过 `IsEqual()([x, 5])` 实现
- `x_eq_9` 在 `x` 等于 9 时取 1，否则为零
- `x_eq_10` 在 `x` 等于 10 时取 1，否则为零
- `otherwise` 在上述信号（`x_eq_5`、`x_eq_9`、`x_eq_10`）全为 0 时取 1。

可以为信号 `x_eq_5`、`x_eq_9`、`x_eq_10` 和 `otherwise` 赋值，使用的是 circomlib 的 `IsEqual()` 模板——这也会强制它们只能为 0 或 1。为了保证恰好一个信号为 1，其余信号都为零，可以使用下面的约束：

$$
\begin{align*}
1===\texttt{x\_eq\_5}+\texttt{x\_eq\_9}+\texttt{x\_eq\_10}+\texttt{otherwise}

\end{align*}
$$

一般来说，我们会创建一些“二进制开关”：特定分支激活时对应开关为 1，否则为 0。然后对所有分支的计算结果求和，每个结果都要乘以相应的开关。

由于 $\texttt{out = }...$ 中只有一个分支会激活，其余分支的计算结果都会乘以 0，因此不会影响结果。

完整电路如下：

```jsx
include "./node_modules/circomlib/circuits/comparators.circom";

template MultiBranchConditional() {
  signal input x;

  signal output out;

  signal x_eq_5;
  signal x_eq_9;
  signal x_eq_10;
  signal otherwise;

  x_eq_5 <== IsEqual()([x, 5]);
  x_eq_9 <== IsEqual()([x, 9]);
  x_eq_10 <== IsEqual()([x, 10]);
  otherwise <== IsZero()(x_eq_5 + x_eq_9 + x_eq_10);

  signal branches_5_9;
  signal branches_10_otherwise;

  branches_5_9 <== x_eq_5 * 14 + x_eq_9 * 22;
  branches_10_otherwise <== x_eq_10 * 23 + otherwise * 45;

  out <== branches_5_9 + branches_10_otherwise;
}

component main = MultiBranchConditional();
```

为了让代码更整洁，最好把四路分支单独放到一个组件中——这样就能复用这个分支模板。

```jsx
include "./node_modules/circomlib/circuits/comparators.circom";

template Branch4(cond1, cond2, cond3, branch1, branch2, branch3, branch4) {
  signal input x;
  signal output out;

  signal switch1;
  signal switch2;
  signal switch3;
  signal otherwise;

  switch1 <== IsEqual()([x, cond1]);
  switch2 <== IsEqual()([x, cond2]);
  switch3 <== IsEqual()([x, cond3]);
  otherwise <== IsZero()(switch1 + switch2 + switch3);

  signal branches_1_2 <== switch1 * branch1 + switch2 * branch2;
  signal branches_3_4 <== switch3 * branch3 + otherwise * branch4;

  out <== branches_1_2 + branches_3_4;
}

template MultiBranchConditional() {
  signal input x;

  signal output out;

  component branch4 = Branch4(5,9,10,14,22,23,45);

  branch4.x <== x;
  branch4.out ==> out; // same as out <== branch4.out
}

component main = MultiBranchConditional();
```

## 涉及大量分支时的代码

在上面的代码中，我们必须显式写出 `switch1`、`switch2`、……、`otherwise`；如果代码中有很多分支，这会非常繁琐。

可以改为把计算理解成开关向量与分支向量的内积（广义点积）：

$$
\begin{align*}
\text{out}&===\langle[\text{switch}_1, \text{switch}_2,...,\text{switch}_n],[\text{branch}_1, \text{branch}_2,...,\text{branch}_n]\rangle\\
&=\text{switch}_1\cdot\text{branch}_1+\text{switch}_2\cdot\text{branch}_2+\dots+\text{switch}_n\cdot\text{branch}_n\\
1&===\text{switch}_1+\text{switch}_2+...+\text{switch}_n\\
0&===\text{switch}_i*(\text{switch}_i-1),\text{i = 1...n}
\end{align*}
$$

上述形式可以保证恰好一个开关激活（等于 1），其他开关都为 0，从而让对应分支成为输出。

为了在 Circom 中高效实现它，我们使用 [multiplexer.circom](https://github.com/iden3/circomlib/blob/master/circuits/multiplexer.circom) 中的 `EscalarProduct` 模板。该模板接收两个长度为 n 的向量，将对应元素相乘，再对结果求和。在下面的代码块中，我们使用 `EscalarProduct` 将每个开关与对应分支相乘。请注意，最后一个开关和分支的处理方式略有不同，因为最后一个条件是兜底的 else 分支。

```jsx
include "./node_modules/circomlib/circuits/comparators.circom";
include "./node_modules/circomlib/circuits/multiplexer.circom";

template BranchN(n) {
  assert(n > 1); // too small

  signal input x;

  // conds n - 1 is otherwise
  signal input conds[n - 1];

  // branch n - 1 is the otherwise branch
  signal input branches[n];
  signal output out;

  signal switches[n];

  component EqualityChecks[n - 1];

  // only compute IsEqual up to the second-to-last switch
  for (var i = 0; i < n - 1; i++) {
    EqualityChecks[i] = IsEqual();

    EqualityChecks[i].in[0] <== x;
    EqualityChecks[i].in[1] <== conds[i];
    switches[i] <== EqualityChecks[i].out;
  }

  // check the last condition
  var total = 0;
  for (var i = 0; i < n - 1; i++) {
    total += switches[i];
  }

  // if none of the first n - 1 switches
  // are active, then `otherwise` must be 1
  switches[n - 1] <== IsZero()(total);

  component InnerProduct = EscalarProduct(n);
  for (var i = 0; i < n; i++) {
    InnerProduct.in1[i] <== switches[i];
    InnerProduct.in2[i] <== branches[i];
  }

  out <== InnerProduct.out;
}

template MultiBranchConditional() {
	signal input x;

	signal output out;

	component branchn = BranchN(4);

  var conds[3] = [5, 9, 10];
  var branches[4] = [14, 22, 23, 45];
  for (var i = 0; i < 4; i++) {
    if (i < 3) {
        branchn.conds[i] <== conds[i];
    }

    branchn.branches[i] <== branches[i];
  }

  branchn.x <== x;
  branchn.out ==> out; // same as out <== branch4.out
}

component main = MultiBranchConditional();
```

## 什么时候可以使用 if 语句？

假设我们要创建一个模板，让它根据电路参数返回完全不同的电路。例如，创建一个 `Max` 组件，让它接收数组 `in[n]` 并返回最大值时，如果 `n` 等于 1，直接返回索引为 0 的项会更高效。

下面展示了在定义约束时正确使用 if 语句的例子。这里的 if 语句在编译时执行，因此模板会生成定义明确的电路：

```jsx
include "./node_modules/circomlib/circuits/comparators.circom";

template Max(n) {
  signal input in[n];
  signal output out;

  assert(n > 0);

  if (n == 1) {
    out <== in[0];
  }

  // it is okay to declare signals inside
  // the if-statement because the evaluation
  // of the if-statement is known at compile time
  else if (n == 2) {
    signal zeroGtOne;
    signal branch0;
    signal branch1;

    zeroGtOne <== GreaterThan(252)([in[0], in[1]]);
    branch0 <== zeroGtOne * in[0];
    branch1 <== (1 - zeroGtOne) * in[1];

    out <== branch0 + branch1;
  }
  else {
    // case for n > 2
  }
}

component main = Max(2);
```

## 条件语句对 ZK 并不友好

一个关键的设计结论是：Circom 电路中的每个条件都会让电路规模翻倍，因为分支无法“短路”。与传统编程不同，所有分支都会被计算。

使用 ZK 证明一项计算时，应当从以下方面优化：

1. 尽量减少分支数量，因为每个分支都会增加证明者的工作量。
2. 尽量降低所有分支的总计算成本，而不只是根据分支概率计算的期望成本。
3. 尽可能避免条件语句。
