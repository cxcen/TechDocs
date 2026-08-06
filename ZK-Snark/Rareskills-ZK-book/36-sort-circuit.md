# 选择排序的零知识证明

我们关心的大多数计算通常都是“有状态的”——也就是说，它们需要经历一系列步骤才能得到最终结果。

有时，我们不需要证明自己执行了计算，只需证明结果。例如，若 A 是一个列表，可以通过证明 B 是 A 的一个排列，且 B 中所有元素均按顺序排列，来证明 B 是列表 A 排序后的版本。无须证明排序算法的每一步都执行正确。前面已经介绍过如何证明列表元素有序，但要高效证明一个列表是另一个列表的排列，难度出人意料，我们将在后文介绍这项技术。

一般来说，许多现实中的计算无法只靠证明结果正确来完成验证。尤其是，要证明我们正确执行了 `sha256("RareSkills")`，就必须正确执行哈希函数的每一步。

哈希函数可能有些令人望而生畏，因此我们通过展示如何证明自己正确地对列表执行了选择排序，来介绍有状态计算的概念。如前所述，这种做法有些“用力过猛”，因为只需证明输出列表是输入列表的一个有序排列即可；至于使用什么算法排序并不重要。

不过，我们仍会展示选择排序算法，因为它很适合作为有状态计算的入门示例。

选择排序的工作方式如下：

- 遍历列表
- 在每个索引 `i` 处，将 `i` 处的值与包含 `i` 及其后所有元素的子列表（`i..n-1`，包含两端）进行比较
- 将 `i` 处的元素与子列表 `i..n-1` 中的最小值交换

下面的动画演示了选择排序：

<video src="https://r2media.rareskills.io/SortCircuit/SelectionSortC.mp4" type="video/mp4" autoplay loop muted controls></video>

由于 ZK 电路中的信号不可变，每次交换时都需要创建一个新列表。例如，对 [5,2,3,4] 排序时，状态转移序列为：

1. i = 0, [5,2,3,4] —> 交换 —> [2,5,3,4]
2. i = 1, [2,5,3,4] —> 交换 —> [2,3,5,4]
3. i = 2, [2,3,5,4] —> 交换 —> [2,3,4,5]

为了证明选择排序执行正确，需要证明在第 `i` 轮迭代中，我们将 `i` 处的元素与子列表 `i…n - 1` 的最小值进行了交换。前面几章已经构建了所需的大部分组件：

- 可以证明某个元素是列表的最小值，并且位于某个特定索引。
- 可以证明交换了列表中的两个元素。

本章只需把这些组件组合起来。首先构建一个模板，证明我们正确找到了子列表最小值的索引：

```jsx
template GetMinAtIdx(n) {
  signal input in[n];

  // compute and constrain min and idx
  // to be the min value in the list
  // and the index of the minimum value
  signal output min;
  signal output idx;

  // compute the minimum and its index
  // outside of the constraints
  var minv = in[0];
  var idxv = 0;
  for (var i = 1; i < n; i++) {
    if (in[i] < minv) {
      minv = in[i];
      idxv = i;
    }
  }
  min <-- minv;
  idx <-- idxv;

  // constrain that min is ≤ all others
  component lte[n];
  for (var i = 0 ; i < n; i++) {
    lte[i] = LessEqThan(252);
    lte[i].in[0] <== min;
    lte[i].in[1] <== in[i];
    lte[i].out === 1;
  }

  // assert min is really at in[idx]
  component qs = QuinSelector(n);
  qs.index <== idx;
  for (var i = 0; i < n; i++) {
    qs.in[i] <== in[i];
  }
  qs.out === min;
}
```

## 排序算法的一轮迭代

选择排序的第一步，是将索引 0 处的元素与整个列表中的最小元素交换（最小元素也可能就在索引 0 处）。下面是把某个特定索引处的数与它后方最小元素交换的代码。

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
  qss.index <== s;
  for (var i = 0; i < n; i++) {
    qss.in[i] <== in[i];
  }

  // get the value at t
  component qst = QuinSelector(n);
  qst.index <== t;
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
    dxEqS[i] = IsEqual();
    dxEqS[i].in[0] <== i;
    dxEqS[i].in[1] <== s;

    dxEqT[i] = IsEqual();
    dxEqT[i].in[0] <== i;
    dxEqT[i].in[1] <== t;

    / if IdxEqS[i].out + IdxEqT[i].out
    / equals 0, then it is not i ≠ s and i ≠ t
    dxNorST[i] = IsZero();
    dxNorST[i].in <== IdxEqS[i].out + IdxEqT[i].out;

    / if we are at index s, write in[t]
    / if we are at index t, write in[s]
    / else write in[i]
    ranchS[i] <== IdxEqS[i].out * qst.out;
    ranchT[i] <== IdxEqT[i].out * qss.out;
    ranchNorST[i] <== IdxNorST[i].out * in[i];

    / multiply branchS by zero if s equals t
    ut[i] <==  (1-sEqT) * (branchS[i]) + branchT[i] + branchNorST[i];
  }
}

template Select(n, start) {
  // unsorted list
  signal input in[n];

  // index start swapped with the min
  signal output out[n];

  // we will define GetMinAtIdxStartingAt in the next codeblock
  component minIdx0 = GetMinAtIdxStartingAt(n, start);
  for (var i = 0; i < n; i++) {
      minIdx0.in[i] <== in[i];
  }

  component Swap0 = Swap(n);
  Swap0.s <== start; // swap 0 with the min
  Swap0.t <== minIdx0.idx; // with the min (could be idx 0)
  for (var i = 0; i < n; i++) {
      Swap0.in[i] <== in[i];
  }

  // copy to out
  for (var i = 0; i < n; i++) {
      out[i] <== Swap0.out[i];
  }
}
```

当然，我们应当把它参数化，因为接下来会针对索引 `0…n - 2` 重复这个过程。为此，需要修改 `GetMinAtIdx`，让它只考虑 `start` 索引之后的值：

```jsx
// formerly GetMinAtIdx
template GetMinAtIdxStartingAt(n, start) {
  signal input in[n];
  signal output min;
  signal output idx;

  // only look for values start and later
  var minv = in[start];
  var idxv = start;
  for (var i = start + 1; i < n; i++) {
    if (in[i] < minv) {
      minv = in[i];
      idxv = i;
    }
  }
  min <-- minv;
  idx <-- idxv;

  // only compare to values start and later
  component lt[n];

  // CHANGES HERE: LOOP FROM START TO N-1
  for (var i = start ; i < n; i++) {
    lt[i] = LessEqThan(252);
    lt[i].in[0] <== min;
    lt[i].in[1] <== in[i];
    lt[i].out === 1;
  }

  // Quin Selector -- ensure that
  // assert min is really at in[idx]
  component qs = QuinSelector(n);
  qs.index <== idx;
  for (var i = 0; i < n; i++) {
    qs.in[i] <== in[i];
  }
  qs.out === min;
}
```

## 最终算法

要证明选择排序执行正确，只需将上面的模板重复 `n - 2` 次。

```jsx
include "circomlib/comparators.circom";

// ----QUIN SELECTOR ----
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

// from https://github.com/darkforest-eth/circuits/blob/master/perlin/QuinSelector.circom
template QuinSelector(choices) {
  signal input in[choices];
  signal input index;
  signal output out;

  // Ensure that index < choices
  component lessThan = LessThan(4);
  lessThan.in[0] <== index;
  lessThan.in[1] <== choices;
  lessThan.out === 1;

  component calcTotal = CalculateTotal(choices);
  component eqs[choices];

  // For each item, check whether its index equals the input index.
  for (var i = 0; i < choices; i ++) {
    eqs[i] = IsEqual();
    eqs[i].in[0] <== i;
    eqs[i].in[1] <== index;

    // eqs[i].out is 1 if the index matches. As such, at most one input to
    // calcTotal is not 0.
    calcTotal.in[i] <== eqs[i].out * in[i];
  }

  // Returns 0 + 0 + 0 + item
  out <== calcTotal.out;
}

// Given array in[n]
// swap the items at index
// s and t
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
  qss.index <== s;
  for (var i = 0; i < n; i++) {
    qss.in[i] <== in[i];
  }

  // get the value at t
  component qst = QuinSelector(n);
  qst.index <== t;
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

    // multiply branchS by zero if s equals t
    out[i] <==  (1-sEqT) * (branchS[i]) + branchT[i] + branchNorST[i];
  }
}

// Find the smallest element starting
// at index start
template GetMinAtIdxStartingAt(n, start) {
  signal input in[n];
  signal output min;
  signal output idx;

  // only look for values start and later
  var minv = in[start];
  var idxv = start;
  for (var i = start + 1; i < n; i++) {
    if (in[i] < minv) {
      minv = in[i];
      idxv = i;
    }
  }
  min <-- minv;
  idx <-- idxv;

  // only compare to values start and later
  component lt[n];

  // CHANGES HERE: LOOP FROM START TO N-1
  for (var i = start ; i < n; i++) {
    lt[i] = LessEqThan(252);
    lt[i].in[0] <== min;
    lt[i].in[1] <== in[i];
    lt[i].out === 1;
  }

  // Quin Selector -- ensure that
  // assert min is really at in[idx]
  component qs = QuinSelector(n);
  qs.index <== idx;
  for (var i = 0; i < n; i++) {
    qs.in[i] <== in[i];
  }
  qs.out === min;
}

// Given an array in, swap
// start with the smallest element
// in front of it
template Select(n, start) {
  // unsorted list
  signal input in[n];

  // index 0 swapped with the min
  signal output out[n];

  component minIdx0 = GetMinAtIdxStartingAt(n, start);
  for (var i = 0; i < n; i++) {
    minIdx0.in[i] <== in[i];
  }

  component Swap0 = Swap(n);
  Swap0.s <== start; // swap 0 with the min
  Swap0.t <== minIdx0.idx; // with the min (could be idx 0)
  for (var i = 0; i < n; i++) {
    Swap0.in[i] <== in[i];
  }

  // copy to out
  for (var i = 0; i < n; i++) {
    out[i] <== Swap0.out[i];
  }
}

// ---- CORE ALGORITHM ----
template SelectionSort(n) {
  assert(n > 0);

  signal input in[n];
  signal output out[n];

  signal intermediateStates[n][n];

  component SSort[n - 1];
  for (var i = 0; i < n; i++) {
    // copy the input to the first row of
    // intermediateStates. Note that we can do
    // if(i == 0) because i is not a signal
    // and i is known at compile time
    if (i == 0) {
      for (var j = 0; j < n; j++) {
        intermediateStates[0][j] <== in[j];
      }
    }

    else {
      // select sort n items starting at i - 1
      // for i = 1, we compare item at 0 to
      // the rest of the list
      SSort[i - 1] = Select(n, i - 1);

      // load in the intermediate state i -1
      for (var j = 0; j < n; j++) {
        SSort[i - 1].in[j] <== intermediateStates[i - 1][j];
      }
      // write the sorted result to row i
      for (var j = 0; j < n; j++) {
        SSort[i - 1].out[j] ==> intermediateStates[i][j];
      }
    }
  }

  // write the final state to the ouput
  for (var i = 0; i < n; i++) {
    intermediateStates[n-1][i] ==> out[i];
  }
}

component main = SelectionSort(9);

/* INPUT = {"in": [3,1,8,2,4,0,1,2,4]} */
```

# 结论

“中间状态”以及证明中间状态之间的转换正确，是验证实践中大多数 ZK 算法的核心，尤其是哈希函数和 ZK 虚拟机。本章介绍的选择排序算法为有状态计算提供了一个循序渐进的入门示例。
