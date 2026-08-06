# Circom 循环中的组件

Circom 不允许直接在循环中实例化组件。例如，编译下面的代码会产生随后所示的错误。

```jsx
include "./node_modules/circomlib/circuits/comparators.circom";

template IsSorted(n) {
  signal input in[n];

  for (var i = 0; i < n; i++) {
    component lt = LessEqThan(252); // error here
    lt.in[0] <== in[0];
    lt.in[1] <== in[1];
    lt.out === 1;
  }
}

component main = IsSorted(8);
```

```
Signal or component declaration inside While scope. Signal and component can only be defined in the initial scope or in If scopes with known condition
```

解决办法是先声明一个组件数组，但不指定组件类型：

```jsx
pragma circom 2.1.8;
include "./node_modules/circomlib/circuits/comparators.circom";

template IsSorted(n) {

  signal input in[n];

  // declare array of components
  // but do not specify the component type
  component lessThan[n];

  for (var i = 0; i < n - 1; i++) {
    lessThan[i] = LessEqThan(252); // specify type in the loop
    lessThan[i].in[0] <== in[i];
    lessThan[i].in[1] <== in[i+1];
    lessThan[i].out === 1;
  }
}

component main = IsSorted(8);

```

以这种方式声明组件时，不能像下面这样用“一行赋值”为信号赋值：

```jsx
pragma circom 2.1.8;
include "./node_modules/circomlib/circuits/comparators.circom";

template IsSorted() {

  signal input in[4];
  signal leq1;
  signal leq2;
  signal leq3;

  // one line assignment to the signal
  leq1 <== LessEqThan(252)([in[0], in[1]]);
  leq2 <== LessEqThan(252)([in[1], in[2]]);
  leq3 <== LessEqThan(252)([in[2], in[3]]);

  leq1 === 1;
  leq2 === 1;
  leq3 === 1;
}

component main = IsSorted();
```

在循环外，可以用一行代码为信号赋值。但在循环内，我们必须分成更多步骤来写赋值，就像 `lessThan[i] = LessEqThan(252); // specify type in the loop` 那样。

## 示例 1：数组的最大值

为了展示在循环中声明组件的实用示例，下面说明如何证明 `k` 是一个数组的最大值。为此，需要约束 `k` 大于或等于其他每个元素，并且至少等于其中一个元素。相等性检查之所以必要，是因为尽管 18 大于或等于 [7, 8, 15] 中的所有元素，它却不是这个数组的最大值。

下面的 Circom 代码在不生成约束的情况下计算数组的最大值。接着，它运行 `n` 个 [GreaterEqualThan](https://github.com/iden3/circomlib/blob/master/circuits/comparators.circom#L131) 组件，约束候选 `max` 确实是最大值；同时使用一组 [IsEqual](https://github.com/iden3/circomlib/blob/master/circuits/comparators.circom#L37) 组件，检查是否至少有一个元素等于 `k`。

```jsx
include "./node_modules/circomlib/circuits/comparators.circom";

template Max(n) {
  signal input in[n];
  signal output out;

  // no constraints here, just a computation
  // to find the max

  var max = 0;
  for (var i = 0; i < n; i++) {
    max = in[i] > max ? in[i] : max;
  }

  out <-- max;

  // for each element in the array, assert that
  // max ≥ that element
  component GTE[n];
  component EQ[n];
  var acc;
  for (var i = 0; i < n; i++) {
        GTE[i] = GreaterEqThan(252);
        GTE[i].in[0] <== out;
        GTE[i].in[1] <== in[i];
        GTE[i].out === 1;

        // this is used in the
        // next code block to ensure
        // that out equals at
        // least one of the inputs
        EQ[i] = IsEqual();
        EQ[i].in[0] <== out;
        EQ[i].in[1] <== in[i];

        // acc is greater than zero
        // (acc != 0) if EQ[i].out
        // equals 1 at least one time
        acc += EQ[i].out;
  }

  // assert that out is
  // equal to at least one of the
  // inputs. if acc = 0 then
  // none of the inputs equals
  // out
  signal allZero;
  allZero <== IsEqual()([0, acc]);
  allZero === 0;
}

component main = Max(8);
```

**练习：**创建一个实现相同功能、但用于求 `min` 的电路。

## 示例 2：数组已排序

检查每个元素都小于或等于后一个元素，就可以断言数组已经排序。上一个例子需要 `n` 个组件，而这里比较的是相邻值，所以只需要 `n - 1` 个组件。因为有 `n` 个元素，所以要进行 `n - 1` 次比较。

下面的模板约束输入数组 `in[n]` 必须有序。请注意，只有一个元素的数组按照定义也是有序的，下面的电路同样适用于这种情况：

```jsx
pragma circom 2.1.6;

include "circomlib/comparators.circom";

template IsSorted(n) {
  signal input in[n];

  component lt[n - 1];

  // loop goes up to n - 1, not n
  for (var i = 0; i < n - 1; i++) {
    lt[i] = LessThan(252);
    lt[i].in[0] <== in[i];
    lt[i].in[1] <== in[i+1];
    lt[i].out === 1;
  }
}

component main = IsSorted(3);
```

## 示例 3：所有项都不重复

检查列表中所有项都不重复，最直接的办法是使用哈希映射，但算术电路中没有哈希映射。效率次优的办法是先对列表排序，但在电路内排序很棘手，所以暂且不这样做。剩下的就是暴力解法：将每个元素与其他所有元素逐一比较。这需要嵌套的 for 循环。

所做的计算可以表示为：

$$
\begin{array}{c|c|c|c|c|}
&a_1&a_2&a_3&a_4\\
\hline
a_1&&\neq&\neq&\neq\\
\hline
a_2&&&\neq&\neq\\
\hline
a_3&&&&\neq\\
\hline
a_4\\
\hline
\end{array}
$$

一般来说，需要进行

$$
\frac{n(n-1)}{2}
$$

次不等性检查，因此也需要同样数量的组件。

下面展示如何实现：

```jsx
pragma circom 2.1.8;

include "./node_modules/circomlib/comparators.circom";

template ForceNotEqual() {
  signal input in[2];

  component iseq = IsEqual();
  iseq.in[0] <== in[0];
  iseq.in[1] <== in[1];
  iseq.out === 0;
}

template AllUnique (n) {
  signal input in[n];

  // the nested loop below will run
  // n * (n - 1) / 2 times
  component Fneq[n * (n-1)/2];

  // loop from 0 to n - 1
  var index = 0;
  for (var i = 0; i < n - 1; i++) {
    // loop from i + 1 to n
    for (var j = i + 1; j < n; j++) {
      Fneq[index] = ForceNotEqual();
      Fneq[index].in[0] <== in[i];
      Fneq[index].in[1] <== in[j];
      index++;
    }
  }
}

component main = AllUnique(5);
```

## 小结

要在循环中使用 Circom 组件，需要先在循环外声明一个组件数组，但不指定类型。

然后在循环内声明具体组件，并约束组件的输入和输出。
