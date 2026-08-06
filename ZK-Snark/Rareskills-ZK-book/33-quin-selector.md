# Quin Selector（选择器）

Quin Selector 是一种设计模式，让我们可以使用一个信号作为信号数组的索引。

阅读本章前，假定读者已经读过“Circom 中的条件语句”一章。

下面的代码无法通过编译，但它展示了我们想实现的效果：

```jsx
template ArraySelect(n) {

  signal input in[n];
  signal input index;

  signal output out;

  // won't compile -- non-quadratic constraints
  out <== in[index];
}
```

要在 Circom 中表达条件逻辑，需要将目标分支乘以一，将其他分支乘以零，再对所有分支求和。被归零的分支不会影响总和。Quin Selector 遵循同样的逻辑：将目标索引对应的值乘以 1，其余值乘以零，然后对结果求和。

例如，假设输入数组是 `in = [5,9,14,20]`。选择索引 2 处的项，意味着计算：

$$
5\cdot0+9\cdot0+\boxed{14\cdot1}+20\cdot0=14
$$

换句话说，我们计算 `[5,9,14,20]` 与 `[0,0,1,0]` 的内积，结果为 14。

当 `index` 等于目标索引时，对应的每个“开关”为 1，否则为 0。

```jsx
include "./node_modules/circomlib/comparators.circom";

template ArraySelect(n) {

  signal input in[n];
  signal input index;
  signal output out;

  component eqs[n];

  // prod keeps a running product
  signal prod[n];

  // prod = 1 * in[i] if i == index else 0
  for (var i = 0; i < n; i++) {
    eqs[i] = IsEqual();
    eqs[i].in[0] <== i;
    eqs[i].in[1] <== index;

    prod[i] <== eqs[i].out * in[i];
  }

  // sum the result
  var sum;
  for (var i = 0; i < n; i++) {
    sum += prod[i];
  }

  out <== sum;
}
```

上面的代码没有约束索引必须小于数组长度。如果索引越界，代码会返回 0。[Dark Forest 中的 Quin Selector 实现](https://github.com/darkforest-eth/circuits/blob/master/perlin/QuinSelector.circom)对 `index` 加入了范围检查；上面的示例也以该模板为基础，读者可以参考它：

```jsx
// out is the sum of in
template CalculateTotal(n) {
  signal input in[n];
  signal output out;

  signal sums[n];

  sums[0] <== in[0];

  for (var i = 1; i < n; i++) {
      sums[i] <== sums[i-1] + in[i];
  }

  out <== sums[n-1];
}

template QuinSelector(choices) {
  signal input in[choices];
  signal input idx;
  signal output out;

  // Ensure that idx < choices
  component lessThan = LessThan(252);
  lessThan.in[0] <== idx;
  lessThan.in[1] <== choices;
  lessThan.out === 1;

  component calcTotal = CalculateTotal(choices);
  component eqs[choices];

  // For each item, check whether its index equals the input idx.
  for (var i = 0; i < choices; i ++) {
    eqs[i] = IsEqual();
    eqs[i].in[0] <== i;
    eqs[i].in[1] <== idx;

    // eqs[i].out is 1 if the idx matches. As such, at most one input to
    // calcTotal is not 0.
    calcTotal.in[i] <== eqs[i].out * in[i];
  }

  // Returns 0 + 0 + 0 + item
  out <== calcTotal.out;
}
```

作为一种优化，步骤 `component lessThan = LessThan(252);` 不需要使用 252 位来确保 `idx` 小于 `choices`。可以根据具体应用使用小得多的位数来完成比较，从而减少底层生成的约束数量。

## circomlib 中的 Quin Selector 实现

circomlib 库中的 [multiplexer（多路复用器）](https://github.com/iden3/circomlib/blob/master/circuits/multiplexer.circom) 实现了与 Quin Selector 相同的功能。不过，它索引一个二维数组，并返回一个一维数组。例如，给定数组 `in = [[5,5],[6,6],[7,7]]` 和 `idx = 1`，它会返回 `[6, 6]`。

该组件有以下输入和输出：

```jsx
template Multiplexer(wIn, nIn) {
  signal input inp[nIn][wIn];
  signal input sel;
  signal output out[wIn];

  // ...
}
```

沿用 `in = [[5,5],[6,6],[7,7]]` 这个例子，`wIn` 为 2，`nIn` 为 3。信号 `sel` 是要选取的索引；例如，当 `sel = 1` 时，`out = [6,6]`。

Multiplexer 不会遍历数组并检查索引是否通过 `IsEqual` 与 `sel` 的值相等，而是生成一个“掩码”：目标索引位置为 1，其余位置全为零，再将输入与该掩码相乘。例如，如果 `sel = 1`，它会生成掩码 `[0,1,0]`，并将输入与之相乘。

下面是使用 circomlib multiplexer 的示例：

```jsx
include "circomlib/multiplexer.circom";

template MultiplexerExample(n) {
  signal input in[n];
  signal input k;
  signal output out;

  component mux = Multiplexer(1, n);

  for (var i = 0; i < n; i++) {
    mux.inp[i][0] <== in[i];
  }
  mux.sel <== k;

  out <== mux.out[0];
}

component main = MultiplexerExample(4);

/* INPUT = {
  "in": [3, 7, 9, 11],
  "k": "1"
} */
```

## 历史说明

这个算法在早于以太坊 Dark Forest 实现的 [xjsnark 论文](https://akosba.github.io/papers/xjsnark.pdf)中被称为“Linear Scan（线性扫描）”。感谢 [0xerhant](https://x.com/0xerhant/status/1873831895055950247) 指出这一点。
