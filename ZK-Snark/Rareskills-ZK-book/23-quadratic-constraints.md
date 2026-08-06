# Circom 中的二次约束

## Circom 约束

秩一约束系统的每条约束最多包含一次信号之间的乘法，这称为“二次”约束。任何包含加法、乘法以外运算的约束都会被 Circom 拒绝，并报告“Non quadratic constraints are not allowed”错误。

下面两个示例的每条约束都包含一次以上的信号乘法，因此无法编译。

## 非二次约束示例 1

编译以下代码会得到 `error: [T3001]: Non quadratic constraints are not allowe`：

```jsx
template QuadraticViolation1() {
  signal input a;
  signal input b;
  signal input c;
  signal input d;

  // two multiplications per constraint
  // is not allowed
  a * b === c * d;
}
```

## 非二次约束示例 2

与上一个示例类似，以下约束包含两次信号之间的乘法。

```jsx
template QuadraticViolation2() {
  signal input a;
  signal input b;
  signal input c;
  signal input d;

  // two multiplications per constraint
  // is not allowed
  a * b * c === d;
}
```

## 与常量相乘不计入限制

因此，尽管以下示例包含一次以上的乘法，它们仍能编译：

```jsx
a * b === c;

2*a * 3*b === 4*c; // integer coefficients allowed

a * b + c === d; // addition and one multiplication allowed

a + b + c === d; // multiplication is optional

a * b + c === d + e + f; // no restrictions on number of additions
```

## 二次形式与 R1CS

回顾[算术化](https://www.rareskills.io/post/arithmetic-circuit)过程：我们把验证程序展开成一系列中间步骤，每个中间步骤只包含未知变量之间的一次乘法。

**考虑以下验证示例：**

```python
def someProblem(x, y, out):
  res = y^2 + 4*(x^2)*y -2
  assert out == res, "incorrect inputs";
```

**转换成 R1CS 后得到：**

```jsx
v1  === y * y
v2  === x * x
out === v1 + (4v2 * y) - 2
```

- 为了遵守二次约束限制，R1CS 格式要求把问题重组为多个中间步骤，每个步骤只包含一次信号之间的乘法。
- 这些步骤构成约束系统。

**因此，R1CS 表示为：**

```jsx
//     Cw = Aw * Bw
       v1  = y * y
       v2  = x * x
out -v1 +2 = (4v2 * y)
```

由于前面已经保证每条约束只有一次乘法，因此可以把约束系统表示成向量形式，也就是 R1CS。

## 非乘法运算符导致非二次约束的示例

如果在约束中使用非法运算，即加法和乘法以外的运算，Circom 编译器会报告“Non quadratic constraints are not allowed!”错误。

下面给出一些示例。

### 示例 1：不能使用信号索引信号数组

以下操作会违反二次约束规则。数组索引无法直接转换成加法与乘法。下面的代码会报告错误 `Non-quadratic constraint was detected statically, using unknown index will cause the constraint to be non-quadratic`：

```jsx
template KMustEqual5(n) {

  signal input in[n];
  signal input k;

  // not allowed
  in[k] === 5;
}
```

从技术上说，仍然可以实现数组索引，但需要采用更复杂的方案，后续章节会介绍。

### 示例 2：信号不能使用 %、<< 等运算

以下约束会触发“Non quadratic constraints are not allowed!”错误：

```jsx
template Example() {
  signal input a;
  signal input b;

  // not allowed
  a === b % 5;

  // not allowed
  a === b << 2;
}
```

## Circom 如何处理除法

Circom 允许信号除以常量，这一点不太直观。原因是该操作可以直接替换为乘以这个常量的乘法逆元。因此，下面的代码合法：

```jsx
template Example() {
  signal input a;
  signal input b;

  a === b / 2;
}

component main = Example();
```

但是，信号之间不能相除，因为这意味着计算某个信号的乘法逆元，而该计算无法直接转换为纯粹的加法与乘法。乘法逆元通常使用[扩展欧几里得算法](https://en.wikipedia.org/wiki/Extended_Euclidean_algorithm)计算，它需要循环和条件语句，而这些运算不能原生表示成加法与乘法。

```jsx
template Example() {
  signal input a;
  signal input b;
  signal input c;

  // not allowed
  a === b / c;
}

component main = Example();
```

相比之下，信号相减是允许的，因为它可以直接转换成乘以常量 `-1`：

```jsx
template Example() {
  signal input a;
  signal input b;

  // allowed
  a === b - a;

  // equivalent
  a === b + -1*a
}

component main = Example();
```

与乘以模逆不同，整数除法用 `\` 表示，并且不能应用于信号：

```jsx
template Example() {
  signal input a;
  signal input b;

  // can only use \ with variables
  // not signals
  a === b \ 2;
}

component main = Example();
```

对于变量，既可以使用整数除法，也可以使用“普通”除法，即乘以除数的乘法逆元。

另一方面，信号只允许使用上述意义上的“普通”除法。

## 总结

一条约束只能包含一次信号之间的乘法，但加法次数没有限制。

这一限制可能让人觉得，除了简单算术外，任何有趣计算都无法表达。不过，后续教程会介绍许多巧妙的设计模式来绕过这一限制。

掌握这些设计模式后，就可以组合它们来建模复杂得多的算法。
