# Circom：Hello World

## 简介

本章说明 Circom 代码与其编译得到的秩一约束系统（R1CS）之间的关系。

理解 R1CS 对理解 Circom **至关重要**。如果还不熟悉，请先复习[秩一约束系统](https://www.rareskills.io/post/rank-1-constraint-system)。

为了说明 Circom 的工作方式，我们从几个示例开始。

## **示例 1：简单乘法**

假设要创建一个 ZK 证明，判断某人是否知道两个任意数的乘积：`c = a * b`。

换言之，给定某些 `a` 和 `b`，我们希望**验证**用户为 `c` 计算出了正确值。

用伪代码表示，验证过程如下（注意，这*不是* Circom 代码）：

```python
def someVerification(a, b, c):
  res = a * b
  assert res == c, "invalid calculation"
```

因此，R1CS 只有一条约束：

```python
assert c == a * b
```

R1CS 使用结构化矩阵格式表达此类约束。根据 [R1CS 章节](https://www.rareskills.io/post/rank-1-constraint-system)的内容，见证向量 $\mathbf{w}$ 应写成 `[1, a, b, c]`，对应的 R1CS 为：

$$
\begin{bmatrix}
0&1&0&0\\
\end{bmatrix}\mathbf{w}
\circ
\begin{bmatrix}
0&0&1&0\\
\end{bmatrix}\mathbf{w}
=
\begin{bmatrix}
0&0&0&1\\
\end{bmatrix}\mathbf{w}
$$

如果 a = 3、b = 4、c = 12，上面的运算为：

$$
\begin{bmatrix}
0&1&0&0\\
\end{bmatrix}
\begin{bmatrix}
1\\
3\\
4\\
12\\
\end{bmatrix}
\circ
\begin{bmatrix}
0&0&1&0\\
\end{bmatrix}\begin{bmatrix}
1\\
3\\
4\\
12\\
\end{bmatrix}
=
\begin{bmatrix}
0&0&0&1\\
\end{bmatrix}\begin{bmatrix}
1\\
3\\
4\\
12\\
\end{bmatrix}
$$

在 Circom 中，上述约束写成：

```jsx
template SomeCircuit() {
  // inputs
  signal input a;
  signal input b;
  signal input c;

  // constraints
  c === a * b;
}

component main = SomeCircuit();
```

- 给定输入 `a`、`b`、`c`，电路验证 `a * b` 是否确实等于 `c`。
- 电路的作用是验证，而**不是**计算。因此，计算结果 `c` 也是必需输入之一。
- `===` 运算符定义了前面以 R1CS 形式表达的约束。`===` 的行为类似断言，所以提供无效输入时，电路约束不会得到满足。在上述代码中，`c === a * b` 约束 `c` 必须等于 `a` 与 `b` 的乘积。

## zkRepl：Circom 在线 IDE

[zkRepl](https://zkrepl.dev) 是进行快速实验的便捷工具。

可以把输入写在注释中，从而在 zkRepl 中测试上述代码：

![展示 Circom 编译器输出的 zkRepl](https://r2media.rareskills.io/HelloWorldCircom/OneConstraint.png)

***注意：** 使用 zkRepl 时，输入以 JSON 对象的形式写在注释中。要测试代码能否编译以及输入是否满足电路，请按 Shift+Enter。*

“non-linear constraints”的值为 1（见红框），因为底层 R1CS 有一行约束，其中两个信号相乘。这里只有一条 `===`，所以结果符合预期。

### **`template`、`component`、`main`**

- **模板**定义电路的蓝图，类似面向对象编程（OOP）中的类为对象定义结构。
- **组件**是模板的实例，类似对象是类的实例。

```jsx
// create template
template SomeCircuit() {
  // .... stuff
}

// instantiate template
component main = SomeCircuit();
```

必须写 `component main = SomeCircuit()`，因为 Circom 要求用唯一的顶层组件 `main` 定义要编译的电路结构。

### `signal input`

- 输入信号是从组件外部提供的值。（Circom 不会强制要求实际提供某个值；开发者必须自行确保这些值确实传入。缺少输入可能造成安全漏洞，后续章节会进一步讨论。）
- 输入信号不可变，不能修改。
- 信号正是 R1CS 见证向量中的变量。

## Circom 的有限域

Circom 在阶为 `21888242871839275222246405745257275088548364400416034343698204186575808495617` 的[有限域](https://www.rareskills.io/post/finite-fields)中执行算术运算，后文简称该数为 $p$。这是一个 **254 位数**，对应 bn128 椭圆曲线的阶。该曲线应用广泛，EVM 预编译合约提供的正是这条曲线。Circom 原本用于开发以太坊上的 ZK-SNARK 应用，因此让域大小与 bn128 曲线的阶一致是合理的。

Circom 允许通过命令行参数修改默认阶。

以下几点应当很直观：

- `p` 在 `mod p` 下与 `0` 同余；
- `p-1` 是[有限域](https://www.rareskills.io/post/finite-field) `mod p.` 中最大的整数；
- 传入大于 `p-1` 的值会导致溢出。

## 示例 2：BinaryXY

再看一个示例，作为本节的收尾。

考虑一个验证传入值是否为二进制数（即 `0` 或 `1`）的电路。

如果输入变量为 `x` 和 `y`，约束组为：

```jsx
(1):  x * (x - 1) === 0
(2):  y * (y - 1) === 0
```

*回顾一下：根据定义，R1CS 的每条约束最多只能包含一次变量之间的乘法。*

***x*(x-1) === 0 用于检查 x 是否为二进制位***

- *该多项式表达式只有两个根。*
- *即 x = 0 或 x = 1。*

**用 Circom 表示**

```jsx
template IsBinary() {

  signal input x;
  signal input y;

  x * (x - 1) === 0;
  y * (y - 1) === 0;
}

component main = IsBinary();
```

### **另一种表达方式：使用数组**

在 Circom 中，可以把输入声明为彼此独立的信号，也可以声明一个包含全部输入的数组。更常见的做法是把所有输入组合到名为 `in` 的信号***数组***中，而不是分别提供 `x` 和 `y`。

按照这一惯例，前面的电路可写成如下形式。数组索引从零开始，这与通常预期一致：

```jsx
template IsBinary() {

  // array of 2 input signals
  signal input in[2];

  in[0] * (in[0] - 1) === 0;
  in[1] * (in[1] - 1) === 0;
}

// instantiate template
component main = IsBinary();
```

## 只接受满足约束的见证

只有输入确实满足电路约束时，Circom 才能生成证明。在下面这个从上一段直接复制的电路中，我们提供 `[0, 2]` 作为输入，而该电路只允许数组元素取 {0,1}。

对于 0，`0 * (0 - 1) === 0` 成立；但 `2 * (2-1) === 2` 违反约束，如下图红框所示。

![zkRepl 显示 Circom 约束未满足](https://r2media.rareskills.io/HelloWorldCircom/IsBinaryViolation.png)

# 在命令行中使用 Circom

本节介绍常用的 Circom 命令。假定读者已经安装 Circom 及其依赖项。

新建一个目录，在其中创建 `somecircuit.circom` 文件并写入以下代码：

```jsx
pragma circom 2.1.8;

template SomeCircuit() {
  // inputs
  signal input a;
  signal input b;
  signal input c;

  // constraints
  c  === a * b;
}

component main = SomeCircuit();
```

## 1. 编译电路

在终端中执行以下命令进行编译：

```bash
circom somecircuit.circom --r1cs --sym --wasm
```

- `--r1cs` 标志表示输出 R1CS 文件；`--sym` 为变量提供便于阅读的名称（更多信息参见 [sym 文档](https://docs.circom.io/circom-language/formats/sym/)）；`--wasm` 用于生成 WASM 代码，以便根据输入 JSON 填充 R1CS 见证，后文会展示这一过程。
- 根据需要，将 `somecircuit.circom` 替换为待编译电路的名称。

预期输出如下：

![Circom 命令行输出](https://r2media.rareskills.io/HelloWorldCircom/CircomSomeCircuitCMDLine.png)

- 可以看到，non-linear constraints 为 1，对应 `a * b === c`。
- Wires 表示 R1CS 的列数。本例包含一列常量和 `a`、`b`、`c` 三个信号。

编译器会创建：

- `somecircuit.r1cs` 文件
- `somecircuit.sym` 文件
- `somecircuit_js` 目录

### **.r1cs 文件**

- 该文件以二进制格式存储电路的 R1CS 约束系统。
- 可以配合不同工具栈构造证明/验证命题，例如 snarkjs、libsnark。

R1CS 文件在某种意义上类似二进制文件：运行 `cat <file>` 只会看到乱码。

运行 `snarkjs r1cs print somecircuit.r1cs`，会得到以下便于阅读的输出：

```
[INFO]  snarkJS: [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.a ] * [ main.b ] - [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.c ] = 0
```

Circom 在有限域中执行算术运算，因此 `21888242871839275222246405745257275088548364400416034343698204186575808495616` 实际上表示 `-1`。不过在 R1CS 文件中，约束运算符写作 `=`，而不是 `==` 或 `===`。

可以通过计算 `-1 mod p`（Python 中为 `-1 % p`）来确认这一点，其中 `p` 是 Circom 有限域的阶。如果把 `snarkjs r1cs print somecircuit.r1cs` 输出的大数转换成负数，会得到：

```solidity
[-1 * main.a] * [main.b] - [-1 * main.c] = 0
```

下面把这个表达式转换成更熟悉的 `a * b === c`，代数推导如下：

```
[-1 * main.a] * [main.b] - [-1 * main.c] = 0

    [-main.a] * [main.b] - [-main.c] = 0 // distribute -1

     [main.a] * [main.b] + [-main.c] = 0 // multiply both sides by -1

     [main.a] * [main.b] = [main.c] // move -main.c to the other side
```

再次注意，该结果与约束 `a * b === c` 一致；该约束定义在 `somecircuit.circom` 中。

### .sym 文件

`somecircuit.sym` 是编译过程中生成的**符号文件**。它很重要，因为：

- 它把便于阅读的变量名映射到这些变量在 R1CS 中的对应位置，供调试使用。
- 它有助于以更易理解的格式打印约束系统，从而方便验证和调试电路。

### somecircuit_js 目录

`somecircuit_js` 目录包含用于生成见证的构件：

- `somecircuit.wasm`
- `generate_witness.js`
- `witness_calculator.js`

下一节将使用 `generate_witness.js`；另外两个文件是 `generate_witness.js` 的辅助文件。

提供电路输入值后，这些构件会计算所需的中间值，并创建可用于生成 ZK 证明的见证。

## 2. 计算见证

要生成见证，必须提供电路的公开输入值。具体做法是创建 `inputs.json` 文件，并把它放在 `somecircuit_js` 目录中。

假设要为输入 `a=1`、`b=2`、`c=2` 创建见证，JSON 文件如下：

```json
{"a": "1","b": "2","c": "2"}
```

Circom 要求使用字符串而不是数字，因为 JavaScript 无法准确处理大于 $2^{53}$ 的整数（[来源](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER)）。

在 `somecircuit_js` 目录中运行：

```bash
node generate_witness.js **somecircuit.wasm** inputs.json witness.wtns
```

输出是计算得到的见证文件 `witness.wtns`。

<aside>
💡

如果传入的值不满足约束 `a*b === c`，例如 `a=1`、`b=2`、`c=3`，`witness_calculator.js` 会抛出错误。

</aside>

### **检查计算得到的见证：witness.wtns**

运行 `cat witness.wtns` 时，输出是一串乱码。

![在终端中 cat 见证文件](https://r2media.rareskills.io/HelloWorldCircom/BinaryFile.png)

这是因为 `witness.wtns` 是采用 snarkjs 所接受格式的二进制文件。

要得到便于阅读的形式，可以运行 `snarkjs wtns export json witness.wtns` 将其导出为 JSON，再用 `cat witness.json` 查看：

![在终端中 cat 见证 JSON](https://r2media.rareskills.io/HelloWorldCircom/Witness.png)

- 第一个 `1` 是见证的常量部分，其值始终为 `1`。这里有 `a = 1`、`b = 2`、`c = 2`，因为输入 JSON 是 `{"a": "1","b": "2","c": "2"}`。
- snarkjs 读取 `witness.wtns` 文件并输出 `witness.json`。
- 计算所得见证符合 R1CS 见证向量布局：`[1, a, b, c]` = `[1, 1, 2, 2]`。

## 示例：isbinary.circom

下面完整演示一个不那么简单的例子：`isbinary.circom`。其约束形式对读者而言应该很熟悉，请回顾示例 2。

```javascript
template IsBinary() {

  // array of 2 input signals
  signal input in[2];

  in[0] * (in[0] - 1) === 0;
  in[1] * (in[1] - 1) === 0;
}

// instantiate template
component main = IsBinary();
```

### **编译电路**

- **`circom isbinary.circom --r1cs --sym --wasm`**
- 对终端输出进行合理性检查：`non-linear constraints: 2`

![在终端中检查 R1CS 约束数量](https://r2media.rareskills.io/HelloWorldCircom/NonLinearConstraints.png)

结果符合预期，因为电路包含两条断言，每条都涉及信号相乘。

接下来检查 R1CS 文件。命令 `snarkjs r1cs print isbinary.r1cs` 会产生以下输出：

```
[INFO]  snarkJS: [ 218882428718392752222464057452572750885483644004160343436982041865758084956161 +main.in[0] ] * [ main.in[0] ] - [  ] = 0
[INFO]  snarkJS: [ 218882428718392752222464057452572750885483644004160343436982041865758084956161 +main.in[1] ] * [ main.in[1] ] - [  ] = 0
```

注意，这个大数与前面强调的 **-1 mod p** 系数略有不同（即 `21888242871839275222246405745257275088548364400416034343698204186575808495616`）。

可以看到，末尾多了一个数字 `1`：

- `21888242871839275222246405745257275088548364400416034343698204186575808495616`
- `21888242871839275222246405745257275088548364400416034343698204186575808495616(1)`

**末尾出现 `1`，是因为 snarkjs 的输出格式存在缺陷。它想表达 -1 * 1，但两者之间没有空格。**

下面通过代数变换，把 snarkjs 的输出还原为原始约束：

```bash
(in[0] - 1) * in[0] === 0
(in[1] - 1) * in[0] === 0
```

推导如下：

```
// original circom output
[ 218882428718392752222464057452572750885483644004160343436982041865758084956161 +main.in[0] ] * [ main.in[0] ] - [  ] = 0
[ 218882428718392752222464057452572750885483644004160343436982041865758084956161 +main.in[1] ] * [ main.in[1] ] - [  ] = 0

// remove empty terms
[ (21888242871839275222246405745257275088548364400416034343698204186575808495616)1 +main.in[0] ] * [ main.in[0] ] = 0
[ (21888242871839275222246405745257275088548364400416034343698204186575808495616)1 +main.in[1] ] * [ main.in[1] ] = 0

// rewrite p - 1 as -1
[ (-1)1 +main.in[0] ] * [ main.in[0] ] = 0
[ (-1)1 +main.in[1] ] * [ main.in[1] ] = 0

// simplify
[ main.in[0] - 1] * [ main.in[0] ] = 0
[ main.in[1] - 1] * [ main.in[1] ] = 0
```

### **生成见证**

- 创建 `inputs.json` 文件，并将其放在 `./isbinary_js` 目录中。
- 这里选择传入 `in[0] = 1`、`in[1] = 0`。
- `inputs.json` 的内容如下：

```javascript
{"in": ["1","0"]}
```

- 生成 `witness.wtns`：运行 `node generate_witness.js isbinary.wasm inputs.json witness.wtns`（在 `isbinary_js` 目录中）。
- `witness.wtns` 创建完成后，将其导出为 JSON 以便检查：
`snarkjs wtns export json witness.wtns`
- 执行 `cat witness.json` 后，会得到以下输出：

```javascript
[
 "1",  // 1
 "1",  // in[0]
 "0"   // in[1]
]
```

- 计算得到的信号符合 R1CS 见证向量布局 `[1, in[0], in[1]]`，各值也与之对应。

## 生成 ZK 证明

R1CS 创建完成后，读者可以按照 [Circom 文档中的步骤生成 ZK 证明](https://docs.circom.io/getting-started/proving-circuits/)，以及配套的智能合约验证器。

## 练习题

完成 ZK Puzzles 仓库中的以下题目，检查自己对本章内容的理解。每道题都要求补全缺失逻辑，只需运行单元测试即可检查答案。

- [https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/BinaryXY/BinaryXY.circom](https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/BinaryXY/BinaryXY.circom)
- [https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/MultiplyNoOut/MultiplyNoOut.circom](https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/MultiplyNoOut/MultiplyNoOut.circom)
- [https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/FourBitBinary/FourBitBinary.circom](https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/FourBitBinary/FourBitBinary.circom)
