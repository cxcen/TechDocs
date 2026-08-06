# 公开输入与私有输入

在 Circom 中，公开输入是见证（witness）中会向验证者公开的信号。

例如，假设我们要创建一个 ZK 证明，声明：“我们知道哪个输入经过哈希后得到 0x492c…9254。”为了让这项声明有实际意义，0x492c…9254（目标哈希输出）的值必须公开。否则，从语义上说，我们只是在声明“我们对某个东西做了哈希”，用途就没那么大了。

下面的电路声明：“我将两个数相乘，得到了第三个数：”

```jsx
template Main() {
  signal input a;
  signal input b;
  signal input c;

  a * b === c;
}

component main = Main();
```

下一个电路的声明与之类似，但有一处变化：结果是公开的——“我将两个数相乘，得到了第三个数，而且第三个数的值是公开已知的：”

```jsx
template Main() {
  signal input a;
  signal input b;
  signal input c;

  a * b === c;
}

component main {public [c]} = Main();
```

- 所有输入信号默认都是私有的，除非使用 `component main {public [c]}` 语法明确将其设为公开。只有在主组件中才能定义哪些输入是公开的。
- `[c]` 是要设为公开的信号列表。列表也可以包含更多信号，例如 `[a,c]`，这样还会让 `a` 公开。
- 只有输入信号可以被指定为公开信号，中间信号不可以。

上面的模板会编译为与下面模板相同的秩一约束系统（Rank-1 Constraint System，R1CS）；下面这个版本在主组件中引入了 `output` 关键字：

```jsx
template Main() {
  signal input a;
  signal input b;
  signal output c;

  a * b ==> c;
}

component main = Main();
```

在上面两个模板中，`c` 都是公开的，并且被约束为 `a` 与 `b` 的乘积。因此，它们底层的 R1CS 相同。不过，第二个版本更“方便”，因为我们不必显式提供 `c`。在第一个使用 `component main {public [c]}` 的电路中，如果提供的 `c` 不满足约束，就无法生成见证。而在第二个将 `c` 用作输出的电路中，见证生成器会自动计算 `c` 的正确值，从而省去手动输入。

由于 `c` 完全由 `a` 和 `b` 决定，没有理由再显式提供 `c` 的值，因此应优先采用 `output` 写法。

请注意，输出是公开的。



对于输入，如果要将其中一些设为公开，就意味着存在某个信号，其值并非完全由其他信号值决定。在这种情况下，我们必须使用 `public` 修饰符。例如，如果我们声明：“我将 `a`、`b` 和 `c` 相乘得到 `d`，其中 `a` 和 `d` 公开，而 `b` 和 `c` 私有”，那么电路可以写成：

```jsx
template Main() {
  signal input a; // explicitly public
  signal input b;
  signal input c;
  signal output d; // implicitly public

  signal s <== a * b; // intermediate signal
  d <== c * s;
}

component main{public [a]} = Main();
```

可以这样理解 `output` 信号：

- 对于子组件，`output` 是一个根据其他输入得到赋值的信号，之后可能会由实例化该子组件的组件使用。
- 对于主组件，`output` 是见证中的公开信号，其值应完全由其他输入信号决定。*声明输出信号却不给它赋值可能造成漏洞，因为证明者可以随意为其赋值。后续章节会展示这种攻击的原理。*

尽管名称叫作“输出”，**但主组件并不存在获取这个“输出”的机制——Circom 无法返回任何内容。其他代码库也无法读取“输出”的值。**

**Circom 只会生成 R1CS，并帮助计算 R1CS 的见证。随后，snarkjs 使用 Circom 代码生成一个 ZK 证明，证明该见证满足 R1CS。**

Circom 并没有被“执行”，所以也不会“返回”任何内容。你不是在“运行”Circom，而只是在描述一个抽象电路；这个描述会被转换为两个部分：R1CS 和见证生成器，二者会被分别使用。

可以将主组件的输出信号理解为一个向验证者公开的中间信号。

## 含公开信号的见证布局

Circom 按以下方式排列见证向量：

```
[constant, public signals, private signals]
```

下面以“我将隐藏值 `a`、`b` 与公开值 `c` 相乘，得到公开值 `d`”为例：

```jsx
// assert that a*b === c*d
template Example() {
  signal input a;
  signal input b;
  signal input c;
  signal input d;

  signal s;

  s <== a * b;
  d === s * c;
}

component main {public [c, d]} = Example();
```

请注意，我们本可以将 `d` 声明为输出以少写一些代码，但为了让接下来的演示更清楚，这里没有这样做。

查看见证结构的方法如下：

1. 将上面的文件保存为 `Example.circom`
2. 使用 `circom Example.circom --sym --r1cs --wasm` 编译
3. 创建 `input.json`：`echo '{"a": "3", "b": "4", "c":"2", "d":"24"}' > input.json`
4. `cd example_js`
5. 计算见证：`node generate_witness.js example.wasm ../input.json witness.wtns`
6. 将见证转换为 JSON 并输出其内容：`snarkjs wej witness.wtns && cat witness.json`

应当得到以下结果。请注意，这与我们在 `input.json` 中提供的值一致：

```jsx
[
 "1", // constant
 "2", // c (public signal)
 "24", // d (public signal)
 "3", // a
 "4", // b
 "12" // s
]
```

由此可见，见证的布局始终是：

- 见证中的常量项（始终为 1）
- 公开信号（`c`、`d`）
- 输入信号（`a`、`b`）
- 中间信号（`s`）。

## 小结

- 输入默认是私有的
- 可以使用 `component main {public [in1, in2]} = Main();` 语法将输入设为公开
- 输出是公开信号
- 输出是根据其他输入为用户计算出来的信号
