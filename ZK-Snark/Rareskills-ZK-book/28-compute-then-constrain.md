# 先计算，后约束

“先计算，后约束”是一种 ZK 电路设计模式：先在不生成约束的情况下计算算法的正确输出，再通过强制满足与该算法有关的不变量来验证结果是否正确。

## 为什么要先计算，后约束

如果只能使用加法、乘法以及赋值并约束（`==>`），很多计算将极难建模，并且需要数量极其庞大的加法和乘法运算。

例如，计算一个数的平方根需要进行多轮迭代估算，这会让电路规模显著增大。

因此，更实用的做法通常是在电路外运行计算——也就是在求解期间不生成任何约束——然后设置一组约束，使其当且仅当计算结果正确时才成立。

我们会大量使用 circomlib 中 [Bitify](https://github.com/iden3/circomlib/blob/master/circuits/bitify.circom) 和 [Comparator](https://github.com/iden3/circomlib/blob/master/circuits/comparators.circom) 库的示例，因为它们广泛采用了这种模式。

## Circom 中的“细箭头” `<--` 运算符及其与 `<==` 的区别

`<==` 运算符会为信号赋值并创建约束。由于约束必须是二次的，我们无法执行下面这样的操作：

```jsx
template InputEqualsZero() {
  signal input in;
  signal output out;

  // out = 1 if in == 0
  out <== in == 0 ? 1 : 0;
}

component main = InputEqualsZero();
```

编译上述电路会产生非二次约束错误，因为三元运算符无法直接表示为信号之间的一次乘法。事实上，Circom 会直接拒绝对信号进行乘法和加法以外的任何运算。

乍看之下，Circom 似乎无法替我们计算 `out`，只能要求把它作为公开输入提供。但如果涉及大量信号，这会非常不便。

我们需要一种机制来告诉 Circom：“根据其他信号计算这个信号的值并为其赋值，但不要创建约束。”这项操作使用 `<--` 运算符：

```jsx
template InputEqualsZero() {
  signal input in;
  signal output out;

  // out = 1 if in == 0
  out <-- in == 0 ? 1 : 0;
}

component main = InputEqualsZero();
```

`in == 0 ? 1 : 0;` 这项操作有时称为“电路外计算”或“hint”。

上面的代码可以通过编译，**但 `out` 和 `in` 都没有受到任何约束**。

`<--` 运算符非常方便：它允许我们在不生成约束的情况下计算值，从而不必手动提供某些信号值。不过，它也一直是安全漏洞的来源之一。

**Circom 不会强制开发者在使用 `<--` 后创建适当的约束，而这正是 Circom 中严重和高危漏洞的常见来源。即使开发者完全没有添加约束，写出了非常危险的代码，编译器也不会发出任何警告或提示。未受约束的信号可以取任意值，使电路能够为毫无意义的陈述生成 ZK 证明。**

后续的[利用欠约束电路](https://www.rareskills.io/post/underconstrained-circom)教程会讲解如何利用误用 `<--` 运算符的 Circom 代码。目前可以把它理解为：这项操作免去了我们亲自提供某个信号值的麻烦，但仍要求我们之后对该信号施加约束。

通过示例最容易理解先计算、后约束，本章余下内容将给出多个例子。

## 示例 1：模平方根

数 $q$ 的模平方根是一个数 $r$，满足 $r^2=q\pmod p$。不过，并非每个域元素都有模平方根。对平方根的正确计算进行建模，其约束很直接（尽管计算平方根本身并不简单）。

请看下面的代码，它证明 `out` 是 `in` 的模平方根：

```jsx
function sqrt(n) {
  // do some magic (see the next code block)
  return r;
}

template ValidSqrt() {
  signal input in;
  signal output out; // sqrt(in)

  out <-- sqrt(in);
  out * out === in; // ensure sqrt was correct
  // the `*` is implicity done modulo p
}
```

这里，`out <-- sqrt(in)` 会将平方根赋给 `out`，但不会添加约束。

circomlib 的 [pointbits](https://github.com/iden3/circomlib/blob/master/circuits/pointbits.circom) 文件提供了计算模平方根的函数。请注意，函数必须声明在 Circom 模板之外。Circom 中的“函数”只是一种便利写法，用于把相关代码放到一个独立代码块中。

```jsx
function sqrt(n) {

    if (n == 0) {
        return 0;
    }

    // Test that have solution
    var res = n ** ((-1) >> 1);
//        if (res!=1) assert(false, "SQRT does not exists");
    if (res!=1) return 0;

    var m = 28;
    var c = 19103219067921713944291392827692070036145651957329286315305642004821462161904;
    var t = n ** 81540058820840996586704275553141814055101440848469862132140264610111;
    var r = n ** ((81540058820840996586704275553141814055101440848469862132140264610111+1)>>1);
    var sq;
    var i;
    var b;
    var j;

    while ((r != 0)&&(t != 1)) {
        sq = t*t;
        i = 1;
        while (sq!=1) {
            i++;
            sq = sq*sq;
        }

        // b = c ^ m-i-1
        b = c;
        for (j=0; j< m-i-1; j ++) b = b*b;

        m = i;
        c = b*b;
        t = t*c;
        r = r*b;
    }

    if (r < 0 ) {
        r = -r;
    }

    return r;
}
```

模平方根有两个解：平方根本身和它的加法逆元。因此，可以像下面这样生成两个解：

```jsx
template ValidSqrt() {
  signal input in;
  signal output out1; // sqrt(in)
  signal output out2; // -sqrt(in)

  out1 <-- sqrt(in);
  out2 <-- out1 * -1; // Computation Step (Unconstrained)
  out1 * out1 === in; // Verification Step (Constraint-Based):
  out2 * out2 === in; // Verification Step
}
```

**警告：**这里给出的代码将 Circom 的默认有限域大小写死在了代码中。如果将 Circom 配置为使用其他有限域，结果可能出错！

上面的例子说明，不考虑约束时，计算平方根要简单得多——如果只用乘法和加法计算平方根，电路会大得不切实际。计算完毕后，再通过约束保证结果正确。

这说明 Circom 既可以作为传统编程语言，也可以作为生成约束的 DSL。代码中的函数 `sqrt(n)` 属于传统编程，而约束 `in === out * out` 则会生成约束。

## 示例 2：数独

如果一项计算难以或需要高昂成本才能通过约束建模——也就是需要大量门和中间信号——那么可以直接把结果作为输入，并假定证明者通过其他方式得到了答案。

真正求解数独需要运行搜索算法来寻找可能的解，通常会使用深度优先搜索。但我们无须直接证明自己运行过搜索算法——给出一个有效解，就足以证明我们完成了求解。网上已经有许多 Circom [数独教程](https://github.com/nalinbhardwaj/snarky-sudoku/blob/main/circuits/sudoku.circom)，因此这里不再给出示例。

## 示例 3：模逆元

假设我们要计算信号 `in` 的乘法逆元，也就是寻找一个信号 `out`，使 `out * in === 1`。

计算乘法逆元的一种方法是使用费马小定理：

$$
x^{-1}=x^{p-2}\pmod p
$$

然而，使用这么大的指数（Circom 的默认值为 $p\approx2^{254}$）会导致大量乘法运算，并产生非常大的电路。更好的做法是在电路*外*计算乘法逆元，然后证明得到的逆元确实正确。例如：

```jsx
template MulInv() {
  signal input in;
  signal output out;

  // Fermat's little theorem
  // compute:
  // note that -2 = p - 2 mod p
  var inv = in ** (-2);
  out <-- inv;

  // then constrain
  out * in === 1;
}

component main = MulInv();
```

这里只有一个约束：`out * in === 1`，因此效率很高。

### Circom 中的模除法

Circom 将 `/` 运算符解释为模除法，因此值 `n` 的逆元可以这样计算：

```jsx
inv <-- 1 / n;
```

上面的模板可以写得更简洁一些：

```solidity
template MulInv() {
  signal input in;
  signal output out;

  // compute
  out <-- 1 / in;

  // then constrain
  out * in === 1;
}

component main = MulInv();
```

模除法是非二次运算，因此只能用于变量或细箭头赋值——也就是说，必须在电路外计算。

## 示例 4：IsZero

### 动机

IsZero 电路非常适合组合到规模更大的计算中。例如，假设我们要证明 `x` 小于 16，或者 `x` 等于 42。

下面这组约束无法实现这一点：

```jsx
// equal 42
x === 42

// less than 16
x === b_0 + 2*b_1 + 4*b_2 + 8*b_3
0 === b_0 * (b_0 - 1)
0 === b_1 * (b_1 - 1)
0 === b_2 * (b_2 - 1)
0 === b_3 * (b_3 - 1)
```

如果 `x` 是 42，它就会违反下面那些约束；如果它小于 16，又会违反 `x === 42`。

因此，我们真正需要的是让子电路*指示*某个条件成立（即 `x` 等于 42 或小于 16），而不是*强制*某个条件成立。之后可以对这些*指示信号*施加约束。例如，假设有指示信号 `x_eq_42` 和 `x_lt_16`，可以使用下面的约束要求其中至少一个为真：

```jsx
// at least one of the two signals is not zero
x_eq_42 * x_lt_16 === 1;
```

要创建一个指示 `x` 等于 42 的信号，我们需要判断 `x - 42` 的值是否恰好为零。

### 设计一个指示某个值为零的电路

这里我们要设计一个电路：返回 `1` 表示输入为 `0`，否则返回 `0`（如果你好奇，这个函数称为 [Kronecker Delta 函数](https://en.wikipedia.org/wiki/Kronecker_delta)）。

如果完全使用加法和乘法来编写这样的函数，得到的函数就是一个多项式，而多项式可以取 0 的位置数量是有限的。换句话说，如果希望函数在有限域中的*每个位置*都为零，多项式的次数就会接近有限域的阶，这在实践中不可行。

我们改为设计一组约束，使 `in` 和 `out` 具有以下性质：

| in | out | 约束结果 |
| --- | --- | --- |
| 0 | 0 | 违反 |
| 0 | 1 | 接受 |
| 非 0 | 0 | 接受 |
| 非 0 | 1 | 违反 |

我们需要一组约束：要求 `out` 在 `in` 为 0 时取 1，并要求 `out` 在 `in` 非零时取 0。也可以这样理解这层关系：“`in` 和 `out` 中至少有一个必须非零，但二者不能同时为零，也不能同时非零。”

要求 `in` 和 `out` 中至少有一个为零，可以用约束 `in * out === 0` 建模。

从下表可以看到，`in * out === 0` 会接受“`in` 和 `out` 中恰好一个为零”的情况，也能正确拒绝 `in` 和 `out` 都非零的情况：

| in | out | 约束结果 | in * out === 0 |
| --- | --- | --- | --- |
| 0 | 0 | 违反 | 接受 |
| 0 | 1 | 接受 | 接受 |
| 非 0 | 0 | 接受 | 接受 |
| 非 0 | 1 | 违反 | 违反 |

约束 `in * out === 0` 的问题是，它不能阻止 `in` 和 `out` 同时为 0 的情况（在上表中标为红色）。

我们还缺少的性质是：`in` 和 `out` 不能同时为零。

最直接的做法是使用 `in + out === 1`。这意味着如果 `in` 是 1，`out` 就必须是 0，反之亦然。但规范要求 `in` 可以是任意非零值，例如 100，而 `100 + out` 不可能等于 1。

不过，如果能“把 100 变成 1”，约束就可以成立。可以先在电路外计算 `in` 的乘法逆元，再施加约束 `in * inv + out === 1`。如果 `in` 为零，我们就令 `inv` 为零，因为零没有乘法逆元。现在得到以下约束：

```jsx
in * inv + out === 1;
in * out === 0;
```

请注意，`inv` 本身没有受到约束，但在这里不会造成影响。

第一个约束 `in * inv + out === 1;` 仅用于禁止 `in` 和 `out` 同时为零。如果 `in` 和 `out` 都是零，无论 `inv` 取何值，都会违反该约束。

总结一下在电路外完成的计算：

- 判断 `in` 是否为零。
- 计算 `in` 的乘法逆元。

circomlib 中的 [IsZero](https://github.com/iden3/circomlib/blob/0a045aec50d51396fcd86a568981a5a0afb99e95/circuits/comparators.circom#L24) 组件实现了本节所述的约束：

```jsx
template IsZero() {
  signal input in;
  signal output out;

  signal inv;

  inv <-- in!=0 ? 1/in : 0;

  out <== -in*inv +1;
  in*out === 0;
}
```

它先在电路外计算 `inv`，再约束 `out`：`in` 为零时必须为 1，否则把 0 赋给 `out`。

### 非确定性输入

在电路外计算、使我们能够使用更简洁约束的值，称为“建议输入（advice input）”或“非确定性输入”。上面电路中的 `inv` 信号就是建议输入（也就是非确定性输入）的一个例子。

## 示例 5：IsEqual

circomlib 中的 [IsEqual](https://github.com/iden3/circomlib/blob/0a045aec50d51396fcd86a568981a5a0afb99e95/circuits/comparators.circom#L37) 组件与 `IsZero` 密切相关——它检查两个输入之差是否为零（如果为零，二者就一定相等）：

```solidity
template IsEqual() {
  signal input in[2];
  signal output out;

  component isz = IsZero();

  in[1] - in[0] ==> isz.in;

  isz.out ==> out;
}
```

## 示例 6：Num2Bits

circomlib 中的 [Num2Bits](https://github.com/iden3/circomlib/blob/252f8130105a66c8ae8b4a23c7f5662e17458f3f/circuits/bitify.circom#L25) 模板按照模板参数的指定，将一个信号分解为 `n` 个比特：

```jsx
template Num2Bits(n) {
  signal input in; // number
  signal output out[n]; // binary output
  var lc1=0;

  var e2=1;
  for (var i = 0; i<n; i++) {
    out[i] <-- (in >> i) & 1;
    out[i] * (out[i] -1 ) === 0;
    lc1 += out[i] * e2;
    e2 = e2+e2;
  }

  lc1 === in;
}
```

*请注意，对于上面代码中的 `n`，如果 $2^n$ 大于有限域，就可能出现 [alias 漏洞](https://www.rareskills.io/post/circom-aliascheck)。该章节会进一步解释这一问题。*

本质上，这段代码从最低有效位开始，遍历二进制表示中的每一位。在每轮循环中，我们把值 `[1,2,4,8,…,2^i]` 存入变量 `e2`，它就是该比特所代表的值。如果这个比特为 1（`out[i] <-- (in >> i) & 1;`），就把该值加到累加器 `lc1` 中。在每轮循环中，我们都会约束读到的比特确实为 0 或 1（使用 `out[i] * (out[i] -1 ) === 0;`）。最后，再约束计算得到的二进制值与原始值相等（`lc1 === in;`）。

用动画最容易展示它计算二进制数组的方式，如下：

<video src="https://r2media.rareskills.io/ComputeThenConstrain/Num2Bits.mp4" type="video/mp4" autoplay loop muted controls></video>

与前面的例子类似，二进制值在电路外计算，之后再施加约束以确保二进制数组正确。

`Num2Bits` 模板是 `LessThan` 模板以及其他用于比较信号相对大小的模板中的核心组件。

域元素（有限域中的数）不能直接相互比较——必须先转换成二进制数。

要了解如何在电路中高效比较二进制数的大小，请先阅读[算术电路](https://www.rareskills.io/post/arithmetic-circuit#:~:text=%F0%9D%91%A0-,Compute%20%E2%89%A5%20in%20binary,-If%20we%20are)一章中的相关部分，再将其中的讨论与 [circomlib 的 LessThan 模板](https://github.com/iden3/circomlib/blob/master/circuits/comparators.circom#L89)进行对照。

## 示例 7：IsMax

要证明某一项是列表中的最大值，必须证明：1）它大于或等于每一个元素；2）它本身也在列表中。要理解第二项要求，请注意，尽管 100 大于或等于列表 [4,5,6] 中的每一项，但 100 并不是该列表的最大值。

下面的电路在电路外使用传统的 for 循环计算最大值，然后使用 `GreaterEqThan` 组件确保 `out` 大于或等于列表中的所有其他项。

为确保 `out` 至少等于列表中的一项，电路会把它与其他每个信号进行 `IsEqual` 比较，再将比较结果求和。如果总和为零，就说明 `out` 不在列表中。因此，我们约束该总和不能为零：

```jsx

template IsMax() {
  signal input in[3];
  signal output out;

  // compute the max as usual
  var maxx = in[0];
  for (var i = 1; i < 3; i++) {
    if (in[i] > maxx) {
      maxx = in[i];
    }
  }

  // propose the max, but do not constrained it yet
  out <-- maxx;

  // max must be ≥ every other element
  signal gte0;
  signal gte1;
  signal gte2;

  // gte0 <== GreaterEqThan(252)([out, in[0]]);
  // is shorthand for
  // component gte0 = GreaterEqThan(252);
  // gte0[0] <== out;
  // gte0[1] <== in[0];
  // 252 is to ensure we don't have enough
  // bits to encode numbers larger than what
  // fits in the default finite field, which
  // would lead to aliasing issues
  gte0 <== GreaterEqThan(252)([out, in[0]]);
  gte1 <== GreaterEqThan(252)([out, in[1]]);
  gte2 <== GreaterEqThan(252)([out, in[2]]);
  gte0 === 1;
  gte1 === 1;
  gte2 === 1;

  // max must be equal to at least one element
  signal eq0;
  signal eq1;
  signal eq2;
  eq0 <== IsEqual()([out, in[0]]);
  eq1 <== IsEqual()([out, in[1]]);
  eq2 <== IsEqual()([out, in[2]]);

  signal iz;
  iz <== IsZero()(eq0 + eq1 + eq2);
  // if IsZero is 1, we have a violation
  iz === 0;
}
```

目前这个电路写死为只支持长度为 3 的数组。如果能让模板支持任意长度的输入会更理想，这正是后续章节的主题。

## 练习题

编写一个 Circom 函数，使用求根公式求解二次多项式的根。请记住，所有运算都在有限域上完成，因此需要使用第一个例子中的模平方根。

然后编写约束，使两个根（如果存在）满足该多项式。将这个多项式作为由三个系数组成的数组传入 Circom 模板。
