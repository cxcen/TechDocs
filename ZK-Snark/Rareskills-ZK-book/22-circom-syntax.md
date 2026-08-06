# Circom 模板参数、变量、循环、if 语句与断言

本章介绍大多数 Circom 程序都会用到的基础语法。借助 Circom，可以通过代码定义秩一约束系统（R1CS），而不必逐条显式定义约束。本章将介绍这些工具。

## **模板参数**

前面介绍了一个电路 `IsBinary`，用于验证所提供的输入是否确实为二进制数。该电路被硬编码为只接受两个输入。

```jsx
template IsBinary() {

  signal input in[2];

  in[0] * (in[0] - 1) === 0;
  in[1] * (in[1] - 1) === 0;
}

component main = IsBinary();
```

上述代码适用于两个输入，但如果要支持大量 `n` 个输入，就必须手工添加约束，开发体验很差。

因此，Circom 允许使用以下模式约束任意数量的信号，自动生成相应约束：

```jsx
template IsBinary(n) {

  // array of n inputs
  signal input in[n];

  // n loops: n constraints
  for (var i = 0; i < n; i++) {
    in[i] * (in[i] - 1) === 0;
  }
}

// instantiated w/ 4 inputs & 4 constraints
component main = IsBinary(4);
```

注意，模板声明发生了变化：圆括号中加入了 `n`。

- 这里的 `n` 称为模板参数；
- `n` 在电路中用于指定数组 `in` 的大小；
- 实例化模板时，必须指定 `n` 的值。

## Circom 中的电路与约束必须具有固定且已知的结构

虽然约束可以用程序生成，但**约束是否存在以及如何配置，不能根据某个信号有条件地变化。**

模板可以使用参数，但电路必须是静态且定义明确的。Circom 不支持“动态长度”电路或约束，一切都必须从一开始就固定下来。

设想一个 R1CS 的结构会随输入信号值变化。由于约束数量并不固定，证明者和验证者都无法正常工作。

`n` 的值必须在编译时确定。

## for 循环与变量：`for`、`var`

下面解释刚才使用的 `for` 循环。

```jsx
template IsBinary(n) {

  // array of n inputs
  signal input in[n];

  // n loops: n constraints
  for (var i = 0; i < n; i++) {
    in[i] * (in[i] - 1) === 0;
  }
}

// instantiated with 4 inputs & 4 constraints
component main = IsBinary(4);
```

- 输入数量和循环次数都由 `n` 决定；
- 对每个输入生成一条约束，用于验证该输入是 `0` 或 `1`。

电路中引入了两个新关键字：`for` 和 `var`。

- `for` 的用法与常见编程语言相同。
- `var` 关键字声明一个**变量**；在上述循环定义中，该变量是 `i`。
- 等号 `=` 把右侧的值赋给左侧变量。

这里，变量 `i` 用于在创建约束时以程序方式引用输入数组中的不同信号。用程序生成约束非常实用；如果涉及数百或数千条约束，手工完成极易出错。

### 变量

变量保存非信号数据，并且可以修改。下面是在循环外声明变量的示例：

```jsx
template VariableExample(n) {
  var acc = 2;
  signal s;
}
```

- 默认情况下，变量**不属于** R1CS 约束系统。
    - 稍后会看到，变量可以作为 R1CS 中的加法或乘法常量。
- 变量用于在 R1CS 外计算值，以帮助定义 R1CS。
- 操作变量时，Circom 的行为与普通编程语言相同。
- 数学运算均对 `p` 取模。完整运算符列表参见 [Circom 文档](https://docs.circom.io/circom-language/basic-operators/#arithmetic-operators)。熟悉类 C 语言的读者会觉得这些运算符很眼熟，例如 `++`、`**`、`<=`。不过要注意，`/` 表示乘以乘法逆元，而 `\` 表示整数除法。
- 信号只能使用 `+`、`*`、`===`、`<--` 和 `<==`。后续文章会讨论 `<--` 与 `<==`。

## if 语句

Circom 允许使用 `if` 语句有条件地创建约束，但条件必须是确定性的，并且在编译时已知。示例如下。

### 示例：强制偶数索引处的元素相等

假设有两个数组，可以使用以下模板生成约束，强制偶数索引处的元素相等，而不检查奇数索引：

```jsx
template EqualOnEven(n) {
  signal input in1[n];
  signal input in2[n];

  for (var i = 0; i < n; i++) {
    if (i % 2 == 0) {
      in1[i] === in2[i];
    }
    // otherwise no constraint is generated
  }
}
```

注意，变量 `i` 决定生成哪些约束。

### 信号不能作为 if 语句或 for 循环中的分支条件

以下代码不合法，因为信号 `a` 被用作 `if` 语句的条件：

```jsx
template IfStatementViolation() {
  signal input a;
  signal input b;

  if (a == 2) {
    b === 3;
  }
  else {
    b === 4;
  }
}
```

在 R1CS 中，信号之间只能进行加法与乘法。Circom 只是 R1CS 之上的一层薄封装，因此无法把 if 语句“转换”为加法和乘法。

Circom 仍然可以根据某个信号执行条件运算（if 语句），后续章节会专门讨论。但目前只需记住：`if` 语句无法“直接”转换成一次乘法。

### 在约束中使用变量

变量可以成为约束的一部分。下面的示例强制输入数组 `in[n]` 是 Fibonacci 数列。变量数组的语法为 `var varName[size]`：

```jsx
template IsFib(n) {
  assert(n > 1);

  signal input in[n];

  // generate the Fibonacci sequence
  var correctFibo[n];
  correctFibo[0] = 0;
  correctFibo[1] = 1;

  for (var i = 2; i < n; i++) {
    correctFibo[i] = correctFibo[i - 1] + correctFibo[i - 2];
  }


  // assert that the input is a Fibonacci sequence
  for (var i = 0; i < n; i++) {
    in[i] === correctFibo[i];
  }
}
```

需要注意：

- `assert(n > 1)` 不会生成任何约束。当模板参数不满足条件时，它会阻止模板实例化。
- 可以使用 `signal === var` 强制信号取某个值，这与 `signal === 5` 或其他常量约束相同。

### Circom 没有 constant 关键字

可以改用变量为魔数命名，从而提高可读性。例如：

```jsx
template Equality() {
  signal input in[2];

  var left = 0;
  var right = 1;

  // require the inputs
  // to be equal
  in[left] === in[right];
}
```

## 变量可以与其他信号相加或相乘

在 Circom 中，变量和常量一样，可以与信号相加或相乘。下面的示例要求 `in2[]` 等于 `in1[]` 中各元素乘以其索引。

例如，如果 `in1[] = [3,5,6]`，那么 `in2[] = [0,5,12]`，因为 `[3,5,6]` 与 `[0,1,2]` 逐元素相乘。

```jsx
template IsIndexMultiplied(n) {
  signal input in1[n];
  signal input in2[n];

  for (var i = 0; i < n; i++) {
    in1[i] * i === in2[i];
  }
}

component main = IsIndexMultiplied(3);

/* INPUT = {"in1": [0,1,2], "in2": [0,1,4]} */

// accept
// in1[] = [0,1,2]
// in2[] = [0,1,4]

// reject
// in1[] = [0,1,2]
// in2[] = [0,0,2]
```

可以在[这里](https://zkrepl.dev)测试代码。

## 要点总结

- 在底层实现中，如果变量与信号相加或相乘，该变量会被编译成 R1CS 中的常量。
- 信号不能执行加、减、乘以外的运算，因为 R1CS 只能包含加法或与常量的乘法。底层的减法其实就是加上加法逆元。
- 如果信号除以常量，或除以保存常量的变量，Circom 会让信号乘以该常量的乘法逆元。常量为 0 时，代码无法编译。

## 练习题

尝试完成 ZK Puzzles 中的以下问题，并运行测试检查答案。

- [https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/AllBinary/AllBinary.circom](https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/AllBinary/AllBinary.circom)
- [https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/MultiANDNoOut/MultiANDNoOut.circom](https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/MultiANDNoOut/MultiANDNoOut.circom)
- [https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/IncreasingDistance/IncreasingDistance.circom](https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/IncreasingDistance/IncreasingDistance.circom)
