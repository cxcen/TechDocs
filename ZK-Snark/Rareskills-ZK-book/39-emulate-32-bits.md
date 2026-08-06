# ZK 中的 32 位运算模拟

ZK 中的默认数据类型是域元素，所有算术运算都对一个大素数取模。然而，大多数“现实”计算会根据虚拟机或执行环境，使用 32 位、64 位或 256 位数。

## 为什么需要 32 位模拟？

许多密码学哈希函数以 32 位机器字（word）为单位进行运算，因为历史上许多 CPU 的默认字长就是 32 位，后来才增加到 64 位。EVM 使用 256 位，以便轻松容纳 keccak256 哈希。

如果要用 ZK 证明传统哈希函数或某个不使用有限域的虚拟机（大多数虚拟机都不使用）执行正确，就需要用域元素为传统数据类型“建模”。因此，在 Circom 中，我们用一个域元素（信号）保存不超过 32 位数容量的数，尽管这个信号本身可以保存远大于 32 位的值。

### 32 位机器字与有限域元素

32 位机器字和有限域元素之间的关键区别，是发生溢出的位置。在 Circom 或任何使用 bn128 曲线的语言中，溢出发生在 $p$，其中 $p$ = `21888242871839275222246405745257275088548364400416034343698204186575808495617`。在 32 位机器中，溢出发生在 `4294967296`；一般而言，它发生在 $2^n$，其中 $n$ 是虚拟机的位数。

可以把 32 位虚拟机理解为所有算术运算都对 $2^{32}$ 取模。普通虚拟机默认会在这个数处溢出。然而，在有限域中进行模运算时，计算模 $2^{32}$ 会增加相当多约束（稍后会看到），幸好有一个实用的数学技巧能够高效完成。

下面两个函数在计算模 $2^{32}$ 时等价：

```solidity
contract DemoMod32 {
  function mod32(uint256 x) public pure returns (uint256) {
    return x % (2**32);
  }

  function mod32e(uint256 x) public pure returns (uint256) {
    // only keep the 32 least significant bits
    return uint256(uint32(x));
  }
}
```

只保留最低有效的 32 个比特，就能计算 $\mod 2^{32}$。附录给出了这一点的形式化验证。在对含有 32 位数的信号执行任何算术运算之前，必须先完全确定该信号保存的数确实能放进 32 位。

## 32 位范围检查

如果创建一个使用 32 位机器字模拟计算的 ZK 电路，就需要确保任何信号保存的值都不会大于或等于 $2^{32}$。一种直观做法是像下面这样使用 `LessThan` 模板：

```jsx
signal safe;
safe <== LessThan(252)([x, 2**32]);
safe === 1;
```

但这个电路创建的约束多于实际需要。

更高效的做法是利用二进制表示。核心思路是用 32 个比特编码一个数：如果该数能放进 32 位，电路就正常执行；如果放不下，约束便无法满足。因此，下面的电路可确保 `in` 小于或等于 $2^{32}-1$。

```jsx
include "circomlib/comparators.circom";

// 8 bit range check
template RangeCheck() {
  signal input in;
  component n2b = Num2Bits(32);
  n2b.in <== in;
}

component main = RangeCheck();

// if in = 2**32 - 1, it will accept
// if in = 2**32 it will reject
```

无须约束 `Num2Bits` 的输出；这与 `LessThan` 不同，因为它在内部已经约束 `out` 必须为零或一，也通过 `lc1 === in` 约束二进制表示必须等于输入。下面的 `Num2Bits` 模板展示了这一点：

```jsx
template Num2Bits(n) {
    signal input in;
    signal output out[n];
    var lc1=0;

    var e2=1;
    for (var i = 0; i<n; i++) {
        out[i] <-- (in >> i) & 1;
        out[i] * (out[i] -1 ) === 0; // CONSTRAINT HAPPENS HERE
        lc1 += out[i] * e2;
        e2 = e2+e2;
    }

    lc1 === in;
}

```

## 32 位加法

假设要把表示 32 位数的两个域元素 `x` 和 `y` 相加。

32 位加法的朴素实现，是先把域元素转换为 32 个比特，再构建一个“加法电路”，逐位相加并处理进位。但这样创建的电路会大于实际需要。

可以改为执行以下步骤：

1. 使用上述策略对 `x` 和 `y` 进行范围检查
2. 将 `x` 和 `y` 作为域元素相加，即 `z <== x + y`
3. 将 `z` 转换为一个 33 位数
4. 把这个 33 位数的最低有效 32 位转换成域元素。

可以将其表示为下图：

![展示 32 位加法的电路图](https://r2media.rareskills.io/Bit32Emulation/circuit-diagram.png)

`x + y` 溢出后最多是一个 33 位数。请注意，`x` 和 `y` 能保存的最大值都是 $2^{32}-1$。将这个值与自身相加可得

$$
\begin{align*}
&(2^{32}-1)+(2^{32}-1)\\
&=2\cdot(2^{32}-1)\\
&=2^{33}-2
\end{align*}
$$

最终结果需要 33 位才能容纳。（请回想，33 位能够容纳的最大数是 $2^{33} - 1$，所以上面的数是 33 位能够容纳的第二大数。）因此，在移除第 33 位之前，只需使用 33 位保存总和。

下面是使用 Circom 模拟并验证 32 位加法的代码：

```jsx
include "circomlib/comparators.circom";
include "circomlib/bitify.circom";

template Add32(n) {
  signal input x;
  signal input y;
  signal output out;

  // range check x and y
  component rCheckX = Num2Bits(32);
  component rCheckY = Num2Bits(32);
  rCheckX.in <== x;
  rCheckY.in <== y;

  // convert the sum to 33 bits
  component n2b33 = Num2Bits(33);
  n2b33.in <== x + y;

  // convert the least significant 32 bits
  // to the final result
  component b2n = Bits2Num(32);
  for (var i = 0; i < 32; i++) {
    b2n.in[i] <== n2b33.out[i];
  }

  b2n.out ==> out;
}
```

## 32 位乘法

32 位乘法的逻辑与 32 位加法极其相似，区别只在于：最终只保留 32 位之前，需要允许 32 位乘法暂时占用最多 64 位：

$$
\begin{align*}
&=(2^{32}-1)(2^{32}-1)\\
&=2^{64}-2^{32}-2^{32}+2\\
&=2^{64}-2^{33}+2
\end{align*}
$$

最终结果需要 64 位才能容纳。

*该电路的实现留作读者练习。*

## 32 位除法与取模

整数除法是 ZK 中最容易出现问题的操作之一，因为要正确约束它，比前面的加法和乘法示例困难得多。下面是现实中几个欠约束除法的例子：

- [https://code4rena.com/reports/2023-10-zksync#h-01-missing-range-constraint-on-remainder-check-in-div-opcode-implementation](https://code4rena.com/reports/2023-10-zksync#h-01-missing-range-constraint-on-remainder-check-in-div-opcode-implementation)
- [https://github.com/succinctlabs/sp1/issues/746](https://github.com/succinctlabs/sp1/issues/746)

整数除法中，被除数、除数、商和余数之间的关系是：

$$
\text{numerator}=\text{denominator}\times\text{quotient}+\text{remainder}
$$

然而，仅靠这个约束不足以保证除法执行正确。

例如，假设我们要证明自己计算了 12 / 7 = 1。电路中的值为

$$
12 = 7 \times 1 +5
$$

但下面的见证同样满足约束：

$$
12 = 7 \times 0 + 12
$$

可以添加“余数严格小于除数”的约束来防止这种情况。

此外，还应注意以下潜在漏洞：

- 对于 254 位有限域中的 32 位运算（Circom 的默认设置），这不是问题，但必须确保计算 $\text{denominator}\times\text{quotient}$ 不会使底层有限域溢出。
- 更一般地说，也不能让计算 $\text{denominator}\times\text{quotient}+\text{remainder}$ 溢出。如果将 `denominator` 和 `quotient` 范围检查为 32 位，那么乘积 $\text{denominator}\times\text{quotient}$ 最多占 64 位；如果再将 `remainder` 范围检查为 32 位，则 $\text{denominator}\times\text{quotient}+\text{remainder}$ 最多需要 65 位。因此，32 位 VM 字长对于 Circom 默认有限域不是问题，但使用 128 位等其他 VM 字长时可能发生溢出。
- 除以零的行为可能因所考虑的 ZKVM 而异。例如，EVM 遇到除以零时不会 panic，而是返回零；在 RISC-V 架构中，除以零会返回所有比特均为 1 的字。

如果只使用加法和乘法，直接计算整数除法并不现实（高效算法——例如乘法的 [Karatsuba 方法](https://en.wikipedia.org/wiki/Karatsuba_algorithm)或[高效整数除法](https://en.wikipedia.org/wiki/Division_algorithm#Integer_division_(unsigned)_with_remainder)——会使用 for 循环或递归，而它们很难映射成加法和乘法），因此最好在约束之外计算整数除法结果。

在 Circom 中，`/` 运算符表示模除法（乘以乘法逆元），`\` 运算符表示整数除法。下面的代码展示了如何证明商和余数计算正确。我们同时计算余数，因为在证明整数除法正确时，余数几乎可以免费得到。

```jsx
include "circomlib/comparators.circom";
include "circomlib/bitify.circom";

template DivMod(wordSize) {
  // a wordSize over this could overflow 252
  assert(wordSize < 125);

  signal input numerator;
  signal input denominator;

  signal output quotient;
  signal output remainder;

  quotient <-- numerator \ denominator;
  remainder <-- numerator % denominator;

  // quotient and remainder still need
  // to be range checked because the
  // prover can supply any value

  // range check all the signals
  component n2bN = Num2Bits(wordSize);
  component n2bD = Num2Bits(wordSize);
  component n2bQ = Num2Bits(wordSize);
  component n2bR = Num2Bits(wordSize);
  n2bN.in <== numerator;
  n2bD.in <== denominator;
  n2bQ.in <== quotient;
  n2bR.in <== remainder;

  // core constraint
  numerator === quotient * denominator  + remainder;

  // remainder must be less than the denominator
  signal remLtDen;

  // depending on the application, we might be able
  // to use fewer than 252 bits
  remLtDen <== LessThan(wordSize)([remainder, denominator]);
  remLtDen === 1;

  // denominator is not zero
  signal isZero;
  isZero <== IsZero()(denominator);
  isZero === 0;
}

component main = DivMod(32);
```

## 32 位移位

假设要模拟下面的代码：

```solidity
uint32 x;
uint32 s;
x << s;
```

左移 `s` 位等价于乘以 $2^s$，其中 $s$ 是移位量；右移 s 位等价于除以 $2^s$。上一章已经看到，幂运算会创建一组相当大的约束。因此，通常更高效的做法是预先计算从 2 的 0 次幂到字长减 1 次幂的所有值。对于 32 位数的左移，我们会预先计算直到 31（字长 32 减 1）的所有 2 的幂：1、2、4、8、……、$2^{31}$，再使用前面讨论的条件选择技术，令 `x` 乘以适当的值。如果移位量大于或等于 32，就乘以零。

*实现留作读者练习。*

## 32 位 AND、NOT、OR、XOR 和 NOT

[circomlib gates 库](https://github.com/iden3/circomlib/blob/master/circuits/gates.circom)已经实现了这些电路，而且代码含义很直观，因此建议读者直接阅读。下面给出模板，说明如何模拟以下操作：

### 按位 AND

```solidity
uint32 a;
uint32 b;
a & b;
```

### 按位 NOT

```solidity
uint32 x;
~x; // flip all the bits
```

下面是计算并约束 `a` 和 `b` 按位 AND 的代码。

```jsx
include "circomlib/gates.circom";
include "circomlib/bitify.circom";

template BitwiseAnd32() {
  signal input a;
  signal input b;
  signal output out;

  // range check
  component n2ba = Num2Bits(32);
  component n2bb = Num2Bits(32);
  n2ba.in <== a;
  n2bb.in <== b;

  component b2n = Bits2Num(32);
  component Ands[32];
  for (var i = 0; i < 32; i++) {
    Ands[i] = AND();
    Ands[i].a <== n2ba.out[i];
    Ands[i].b <== n2bb.out[i];
    Ands[i].out ==> b2n.in[i];
  }

  b2n.out ==> out;
}

component main = BitwiseAnd32();
```

*其余 NOT、OR 和 XOR 操作留作读者练习。*

## ZK EVM 如何处理 256 位数

Circom 默认有限域无法容纳 256 位数。与 64 位计算机模拟 EVM 的方式类似，EVM 中的每个机器字必须用一组更小的机器字来建模。

例如，可以用四个 64 位机器字为一个 256 位数建模。执行加法时，把低位机器字的进位传递到下一个更高位机器字。如果最高位机器字溢出，只需丢弃溢出部分。

## 附录 A：等价性证明

我们使用 Certora prover 演示下面两个函数等价：

```solidity
contract DemoMod32 {
    function mod32(uint256 x) public pure returns (uint256) {
        return x % (2**32);
    }

    function mod32e(uint256 x) public pure returns (uint256) {
        // only keep the 32 least significant bits
        return uint256(uint32(x));
    }
}
```

下面是 Certora Verification Language 规则：

```jsx
methods {
  function mod32(uint256) external returns (uint256) envfree;
  function mod32e(uint256) external returns (uint256) envfree;
}

rule test_Mod32AndMod32E_Equivalence() {
  uint256 x;
  assert mod32(x) == mod32e(x);
}
```

下面是 Certora 报告：

[https://prover.certora.com/output/541734/6cd3303cb5f5441e8773adb5c79787d7?anonymousKey=0b945ee1440cd67efed3efba3162b0f924e8cf8f](https://prover.certora.com/output/541734/6cd3303cb5f5441e8773adb5c79787d7?anonymousKey=0b945ee1440cd67efed3efba3162b0f924e8cf8f)
