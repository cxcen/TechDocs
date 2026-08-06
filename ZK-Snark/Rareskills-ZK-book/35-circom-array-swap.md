# 在 Circom 中交换数组里的两个元素

本章介绍如何交换信号列表中的两个信号。这是排序算法的一项重要子程序。更一般地说，列表是构造哈希函数、为 CPU 内存建模等更复杂功能的基础，因此我们必须学习如何更新列表中的值。

交换列表中的两个元素是程序员最早学习的操作之一，典型解法如下：

```python
# s and t are indexes of array arr
def swap(arr, s, t):
  temp = arr[s];
  arr[s] = arr[t];
  arr[t] = temp;
  return arr
```

然而，在 ZK 电路中，这项操作可能出人意料地棘手。

- 首先，不能直接索引信号数组。为此需要使用 Quin Selector。
- 其次，信号不可变，所以不能向信号数组中的某个信号“写入”数据。

我们需要创建一个新数组，把旧值复制到新数组，并遵守以下条件：

- 如果当前位于索引 `s`，写入 `arr[t]` 处的值
- 如果当前位于索引 `t`，写入 `arr[s]` 处的值
- 否则写入原值

向新数组执行的每次写入都是条件操作。

前一章已经解释过 Quin Selector；为了节省篇幅，这里不再重复其代码。

## 在 Circom 中交换元素

下面的组件会将索引 `s` 处的元素与索引 `t` 处的元素交换，并返回一个新数组。（以下代码存在一个漏洞，试着找出它！答案将在后文给出。）

```jsx
template Swap(n) {
  signal input in[n];
  signal input s;
  signal input t;
  signal output out[n];

  // we do not check that
  // s < n or t < n
  // because the Quin selector
  // does that

  // get the value at s
  component qss = QuinSelector(n);
  qss.idx <== s;
  for (var i = 0; i < n; i++) {
    qss.in[i] <== in[i];
  }

  // get the value at t
  component qst = QuinSelector(n);
  qst.idx <== t;
  for (var i = 0; i < n; i++) {
    qst.in[i] <== in[i];
  }

  // qss.out holds in[s]
  // qst.out holds in[t]

  component IdxEqS[n];
  component IdxEqT[n];
  component IdxNorST[n];
  signal branchS[n];
  signal branchT[n];
  signal branchNorST[n];
  for (var i = 0; i < n; i++) {
    IdxEqS[i] = IsEqual();
    IdxEqS[i].in[0] <== i;
    IdxEqS[i].in[1] <== s;

    IdxEqT[i] = IsEqual();
    IdxEqT[i].in[0] <== i;
    IdxEqT[i].in[1] <== t;

    // if IdxEqS[i].out + IdxEqT[i].out
    // equals 0, then it is not i ≠ s and i ≠ t
    IdxNorST[i] = IsZero();
    IdxNorST[i].in <== IdxEqS[i].out + IdxEqT[i].out;

    // if we are at index s,
    // write in[t]
    // if we are at index t,
    // write in[s]
    // else write in[i]
    branchS[i] <== IdxEqS[i].out * qst.out;
    branchT[i] <== IdxEqT[i].out * qss.out;
    branchNorST[i] <== IdxNorST[i].out * in[i];
    out[i] <==  branchS[i] + branchT[i] + branchNorST[i];
  }
}
```

请注意，最后的条件语句

```jsx
branchS[i] <== IdxEqS[i].out * qst.out;
branchT[i] <== IdxEqT[i].out * qss.out;
branchNorST[i] <== IdxNorST[i].out * in[i];
out[i] <==  branchS[i] + branchT[i] + branchNorST[i];
```

不能写成

```jsx
out[i] <==  IdxEqS[i].out * qst.out + IdxEqT[i].out * qss.out + IdxNorST[i].out * in[i]
```

因为这样会产生非二次约束错误（该约束中存在不止一次乘法）。

## 找出漏洞

上面的代码中有一个漏洞——继续往下读之前，你能找出它吗？

## 代码中的漏洞

上面代码的问题是，它没有考虑 `s` 处的值可能等于 `t` 处的值（`s == t`）。在这种情况下，写入该索引的值会是这个索引原有值与其自身之和。

## 修复问题

为防止这种情况，需要显式检测 `s == t`，并将 `branchS` 或 `branchT` 中的一个乘以零，避免把值翻倍。换句话说，如果 `s` 和 `t` 的开关都处于激活状态，结果值将是 `s + t`。但我们不希望这样；我们要任意选择 `branchS` 或 `branchT`，从而让值实际上保持不变（二者的值相同）：

```jsx
template Swap(n) {
  signal input in[n];
  signal input s;
  signal input t;
  signal output out[n];

  // NEW CODE to detect if s == t
  signal sEqT;
  sEqT <== IsEqual()([s, t]);

  // get the value at s
  component qss = QuinSelector(n);
  qss.idx <== s;
  for (var i = 0; i < n; i++) {
    qss.in[i] <== in[i];
  }

  // get the value at t
  component qst = QuinSelector(n);
  qst.idx <== t;
  for (var i = 0; i < n; i++) {
    qst.in[i] <== in[i];
  }

  component IdxEqS[n];
  component IdxEqT[n];
  component IdxNorST[n];
  signal branchS[n];
  signal branchT[n];
  signal branchNorST[n];
  for (var i = 0; i < n; i++) {
    IdxEqS[i] = IsEqual();
    IdxEqS[i].in[0] <== i;
    IdxEqS[i].in[1] <== s;

    IdxEqT[i] = IsEqual();
    IdxEqT[i].in[0] <== i;
    IdxEqT[i].in[1] <== t;

    // if IdxEqS[i].out + IdxEqT[i].out
    // equals 0, then it is not i ≠ s and i ≠ t
    IdxNorST[i] = IsZero();
    IdxNorST[i].in <== IdxEqS[i].out + IdxEqT[i].out;

    // if we are at index s, write in[t]
    // if we are at index t, write in[s]
    // else write in[i]
    branchS[i] <== IdxEqS[i].out * qst.out;
    branchT[i] <== IdxEqT[i].out * qss.out;
    branchNorST[i] <== IdxNorST[i].out * in[i];

    // multiply branchS by zero if s equals T
    out[i] <==  (1-sEqT) * (branchS[i]) + branchT[i] + branchNorST[i];
  }
}
```

## 结论

在 Circom 中操作数组时，需要创建一个新数组，把旧值复制到新数组中，只在需要更新的位置写入不同的值。

在循环中使用这种模式，可以实现列表排序、栈和队列等数据结构建模，甚至改变 CPU 或 VM 的状态。后续章节会看到这些例子。
