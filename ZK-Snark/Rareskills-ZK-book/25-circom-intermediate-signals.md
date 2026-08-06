# Circom 中的中间信号与子组件

Circom 的首要用途是编译成秩一约束系统（R1CS），其次是填充见证。

对于大多数电路，少数几个信号的值会决定其余信号的值。

例如，在下面的模板中把 `c` 作为输入似乎有些多余，因为它的值完全取决于 `a` 和 `b`：

```jsx
template Mul() {
  signal input a;
  signal input b;
  signal input c;

  c === a * b;
}
```

下面来看一个更有说服力的例子。

## 拆分非二次约束

假设要为 `a * b * c === d` 创建 R1CS。由于 R1CS 的每条约束只允许一次乘法，因此必须创建另一个信号 `s` 和一条额外约束来拆分乘法：

```jsx
template Mul3() {
  signal input a;
  signal input b;
  signal input c;
  signal input d;

  signal input s;

  s === a * b;
  d === s * c;
}
```

每当出现一次以上的乘法就要求再提供一个输入，会非常繁琐；在包含大量乘法的大型电路中尤其如此。此外，上述示例中 `s` 的值完全由 `a` 和 `b` 确定。

## 中间信号与赋值

为了避免手工提供 `s`，Circom 提供 `==>` 和 `<==` 运算符，让 Circom 计算并赋予 `s` 的值。请记住，Circom 的功能之一就是生成见证。因此，不再需要把 `s` 的值作为输入提供。准确地说，`==>` 和 `<==` 表示“赋值并约束”：

```jsx
template Mul3() {
  signal input a;
  signal input b;
  signal input c;
  signal input d;

  // no longer an input
  signal s;

  a * b ==> s;
  s * c === d;
}
```

Circom 允许箭头朝任意方向：`a * b ==> s` 与 `s <== a * b` 含义相同。

上述代码中的 `s` 称为*中间信号*。中间信号使用 `signal` 关键字定义，但不带 `input` 关键字。因此，`signal s` 是中间信号，`signal input a` 则不是。

**上面两个模板生成的底层 R1CS 完全相同。`==>` 只是免去了把 `s` 的值作为输入提供的麻烦。**

假设见证向量 $\mathbf{w}$ 表示为 `[1, a, b, c, d, s]`，底层 R1CS 如下：

$$
\begin{bmatrix}
0 & 1 & 0 & 0 & 0 & 0\\
0 & 0 & 0 & 0 & 0 & 1
\end{bmatrix}\mathbf{w}
\circ
\begin{bmatrix}
0 & 0 & 1 & 0 & 0 & 0\\
0 & 0 & 0 & 1 & 0 & 0
\end{bmatrix}\mathbf{w}=
\begin{bmatrix}
0 & 0 & 0 & 0 & 0 & 1\\
0 & 0 & 0 & 0 & 1 & 0
\end{bmatrix}\mathbf{w}
$$

可以把这个过程理解为：向 Circom 传入见证 `[1, a, b, c, d, _]`，Circom 再根据输入计算出完整见证 `[1, a, b, c, d, s]`。

**对 `s` 的赋值发生在 R1CS 外。R1CS 只检查见证向量 $\mathbf{w}$ 是否满足矩阵等式。R1CS 期望获得完整见证，本身不会计算任何见证值。这种方法在保持 R1CS 结构不变的同时，简化了电路设计并减少手工作业。**

## 信号值不能通过 `<==` 重复赋值

一个信号表示见证向量中的一个确定条目，所以值一旦设置就不能改变。因此，以下代码无法编译：

```jsx
template CannotReassign() {
  signal input a;
  signal input b;

  signal c;

  c <== a * b;

  // not allowed
  // c already set
  c <== a * a;
}
```

## 实际示例：检查数组元素的乘积

电路中的乘法越多，`==>` 运算符就越方便，因为它可以避免提供额外的输入信号。

假设要强制输入信号 `k` 等于数组 `in[n]` 中所有信号的乘积。换言之，需要检查：

$$
\prod_{i=0}^{n - 1}\texttt{in}[i]===k
$$

这会引入大量中间信号。为了保持代码整洁，可以把所有中间信号放入单独的数组：

```jsx
template KProd(n) {
  signal input in[n];
  signal input k;

  // intermediate signal array
  signal s[n];

  s[0] <== in[0];
  for (var i = 1; i < n; i++) {
    s[i] <== s[i - 1] * in[i];
  }

  k === s[n - 1];
}
```

根据上述代码，`s[n - 1]` 保存的值为：

$$
\prod_{i=0}^{n - 1}\texttt{in}[i]
$$

然后即可把它约束为等于 `k`。

## 将 Circom 拆分为多个模板

理解 `<==` 运算符后，就可以理解 Circom 如何利用模板提高代码模块化程度。

与 `Mul3` 示例类似，假设有一个电路接收三个输入，并强制其乘积等于第四个输入。代码再次列出如下：

```jsx
template Mul3() {
  signal input a;
  signal input b;
  signal input c;
  signal input d; // d === a * b * c

  // no longer an input
  signal s;

  a * b ==> s;
  s * c === d;
}
```

但如果要对八个输入执行两次该操作，可能会想把代码分别为输入 (a,b,c,d) 和 (x,y,z,u) 复制粘贴两遍，这样会很难看。

```jsx
template Mul3x2() {
  signal input a;
  signal input b;
  signal input c;
  signal input d; // d === a * b * c

  signal input x;
  signal input y;
  signal input z;
  signal input u; // u === x * y * z

  // ugly code here
}
```

更好的做法是把 `Mul3` 定义为单独的模板：

```jsx
// separate template
template Mul3() {
  signal input a;
  signal input b;
  signal input c;
  signal input d; // d === a * b * c

  // no longer an input
  signal s;

  a * b ==> s;
  s * c === d;
}

// main component
template Mul3x2() {
  signal input a;
  signal input b;
  signal input c;
  signal input d; // d === a * b * c

  signal input x;
  signal input y;
  signal input z;
  signal input u; // u === x * y * z

  component m3_1 = Mul3();
  m3_1.a <== a;
  m3_1.b <== b;
  m3_1.c <== c;
  m3_1.d <== d;

  component m3_2 = Mul3();
  m3_2.a <== x;
  m3_2.b <== y;
  m3_2.c <== z;
  m3_2.d <== u;
}
```

需要注意：

- 使用 `component m3_1 = Mul3();` 语法声明组件，这与声明主组件时使用的语法相同。
- 使用 `<==` 运算符“连接”信号。
- 上述代码与把 `Mul3` 的核心逻辑复制粘贴两次完全等价。

## 从模板传回结果

在某些情况下，如果子组件能够把“结果传回”创建它的组件，会非常方便。

例如，下面的主组件使用子组件 `Square`，把 `out` 赋值并约束为 `in` 的平方。

```jsx
template Square() {
  signal input in;
  signal output out;

  out <== in * in;
}

template Main() {
  signal input a;
  signal input b;
  signal input sumOfSquares;

  component a2 = Square();
  component b2 = Square();

  a2.in <== a;
  b2.in <== b;

  // assert that a^2 + b^2 === sum of Squares
  a2.out + b2.out === sumOfSquares;
}

component main = Main();
```

**在子组件语境中，输出信号是一个期望通过 `<==` 运算符获得赋值的信号，它可以把值传回创建该子组件的组件。**

在 `main` 组件的语境中，输出信号的含义完全不同，后续章节会解释。

## 示例：将二进制转换为数值

[circomlib 库](https://github.com/iden3/circomlib)包含用于各种常见操作的 Circom 模板。其中一项操作是把二进制数组转换成一个信号。前面已经看到，可以用 $b_0+2b_1+4b_2+...+2^{n-1}b_{n-1}=v$ 实现这一转换。下面把它放入独立组件中。这个模板位于 Circom 库的 [bitify.circom](https://github.com/iden3/circomlib/blob/master/circuits/bitify.circom) 文件：

```jsx
template Bits2Num(n) {
  signal input in[n];
  signal output out;

  // lc is short for "linear combination"
  // it serves as an accumulator variable
  var lc1=0;

  var e2 = 1;
  for (var i = 0; i<n; i++) {
    lc1 += in[i] * e2;
    e2 = e2 + e2; // could also be e2 *= 2;
  }

  lc1 ==> out;
}
```

不必从库中复制粘贴代码，可以像其他语言导入文件那样“包含”它：

```jsx
include "circomlib/bitify.circom";

template Main(n) {
  signal input in[n];
  signal input v;

  // instantiate the Bits2Num component
  component b2n = Bits2Num(n);

  // loop over each binary value
  // and assign and constrain it to the
  // b2n input array
  for (var i = 0; i < n; i++) {
    b2n.in[i] <== in[i];
  }

  b2n.out === v;
}

component main = Main(4);

/* INPUT = {"in": [1, 0, 0, 1], "v": 9} */
```

上述组件可以在 zkRepl 中测试。如果在本地运行，需要根据目录配置设置导入路径。通常可以用 yarn 或 npm [安装](https://www.npmjs.com/package/circomlib) circomlib。

## 单行组件示例

除了分别为组件的输入信号赋值，还可以把它们作为参数提供。这称为“[匿名组件](https://docs.circom.io/circom-language/anonymous-components-and-tuples/)”。考虑以下示例：

```jsx
template Mul() {
  signal input in[2];
  signal output out;

  out <== in[0] * in[1];
}

template Example() {

  signal input a;
  signal input b;

  signal output out;

  // one line instantiation
  out <== Mul()([a, b]);
}

component main = Example();
```

## 不应忽略输出信号

输出信号必须参与实例化它的组件中的约束。如果输出信号处于“悬空”状态，在某些情况下，恶意证明者可以为它赋予任意值。[攻击欠约束电路](https://www.rareskills.io/post/underconstrained-circom)一章会进一步讨论。

## 总结

- `<==` 与 `==>` 免去了在 input.json 中显式提供信号值的麻烦。
- 只要一个信号的值由另一个信号的值直接决定，就可以使用 `<==` 或 `==>`。
- `<==` 等价于 `==>`，只是参数顺序相反，效果相同。
- 组件可以实例化其他子组件，并使用 `<==` 或 `==>` 向其输入信号传值。
- 子组件的 `output` 信号应被约束为等于实例化它的组件中的其他信号。
