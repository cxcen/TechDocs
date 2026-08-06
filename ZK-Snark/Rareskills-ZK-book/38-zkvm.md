# ZKVM 的工作原理

**零知识虚拟机**（Zero-Knowledge Virtual Machine，ZKVM）是一种虚拟机，它可以创建 ZK 证明，以验证自己正确执行了一组机器指令。借助它，我们可以取得一个程序（一组操作码）和一份虚拟机规范（虚拟机如何运行、使用哪些操作码等），并证明生成的输出正确。验证者不必重新运行程序，只需检查生成的 ZK 证明——这样便能实现简洁验证。这种简洁性正是 ZK Layer 2 区块链具备可扩展性的基础。它也让人们无须重新运行整个机器学习算法，就能检查算法是否按声明运行。

与名称给人的印象不同，ZKVM 很少在“将计算保密”的意义上实现“零知识”。它们主要使用 ZK 算法，为程序在某个输入上正确执行生成简洁证明，使验证者能够以指数级减少的工作量复核计算。虽然可以选择是否公开程序输入，但要避免意外的数据泄漏，并让多方就私有状态达成一致，仍然是极具挑战性的工程问题，其中还有尚未解决的难题和扩展限制。

ZKVM 会“计算”操作码中的每一步，再约束该操作码确实执行正确。约束的设计必须允许我们使用任意但有效的操作码序列。

可以把 ZKVM 理解成一系列“状态”转换。“状态转移函数”接收前一个状态和当前要执行的操作码，并生成新状态。ZKVM 实现这个“状态转移函数”，以及为其行为建模的电路约束。请注意，“状态”可以包括“[程序计数器](https://en.wikipedia.org/wiki/Program_counter)”或 VM 正常运行所需的其他记录信息。

本教程将构建一个极其简单、只支持基础算术但可扩展其他操作码的 ZKVM。这里的 VM 只有栈，没有内存或存储。文末会提供进一步学习 ZKVM 的建议。

## 前置知识

本教程假定读者对 EVM（或 Java 虚拟机等其他基于栈的架构）的工作原理有基本了解。我们会大量沿用上一章的栈示例，因此也假定读者掌握该章内容。

## 简单的基于栈的 ZKVM

我们将构建一个简单的基于栈的 ZKVM，其中有一个专门保存计算结果的信号。VM 接收一系列操作码和数字，然后把最终结果输出到一个称为 `out` 的专用信号。

这个 ZKVM 只有以下操作码：

- PUSH（将第一个参数压入栈中）
- ADD（弹出栈顶两个元素，再压入它们的和）
- MUL（弹出栈顶两个元素，再压入它们的积）
- NOP（无操作，不作任何处理）

为简单起见，所有操作码都接收一个参数，但只有 PUSH 会使用该参数，其余指令会忽略它。之所以仍给不使用参数的操作码提供参数，是为了避免根据操作码有条件地检查参数是否存在。

这里没有 STOP 或 RETURN 操作码（稍后会解释替代方案）。VM 接收一个 `steps` 参数，执行 `steps` 条指令后，返回栈底的值。

下面的动画给出了在这种架构中将两个数相加的简单示例：

<video src="https://r2media.rareskills.io/ZKVM/zkvm.mp4" type="video/mp4" autoplay loop muted controls></video>

在 Circom 中，循环不能具有可变长度，而必须始终执行固定次数，因为底层的秩一约束系统（Rank-1 Constraint System，R1CS）本身必须具有固定大小。同理，程序也不能具有可变大小。无论运行哪个程序，都必须含有相同数量的操作码。

如果程序的操作码少于固定数量，只需用 NOP 填充到程序最大长度。为了知道何时“停止执行”，用户必须提供前面提到的 `steps` 参数，用它确定何时返回栈底的值。

关于这个架构，还有几点需要说明：

- VM 与 EVM、Java 虚拟机或（对了解它的人来说）[逆波兰表示法计算器](https://en.wikipedia.org/wiki/Reverse_Polish_notation)一样，基于栈工作。
- 没有跳转指令，因此程序计数器只会递增。
- 所有操作码都接收一个参数，但 ADD、MUL 和 NOP 会忽略传给它们的参数。这样每次都能让程序计数器增加相同的值——不必为 PUSH 增加 2、为 ADD 增加 1，依此类推。计数器始终增加 2。
- 要读取 PUSH 的参数，只需从程序计数器向前“查看”一个索引。
- 加法和乘法使用模运算（Circom 的默认方式）。我们将 Circom 默认有限域的阶用作“字长”，不尝试模拟采用 64 位或 256 位等传统字长的 VM。下一章将讨论如何模拟固定比特宽度的计算。

## 更新栈代码

对上一章的栈代码作几项关键修改，就能构建符合上述规范的 ZKVM。

- 删除不再需要的 POP 操作码。
- 加入 ADD 和 MUL 操作码。

回顾之前复制前一个栈状态的规则：

- A. 如果 `sp` 大于或等于 1，第 `j` 列比 `sp` 低 1 个索引，并且当前指令为 PUSH 或 NOP，就应复制第 `j` 列
- B. 如果 `sp` 大于或等于 2，第 `j` 列比 `sp` 低 2 个索引，并且当前指令为 POP，就应复制第 `j` 列

规则 A 保持不变，但 B 需要更新为：

- B. 如果 `sp` 大于或等于 2，第 `j` 列比 `sp` 低 **3** 个索引，并且当前指令为 ADD 或 MUL，就应复制第 `j` 列

之所以这样修改，是因为之前的 POP 指令不会改变栈中倒数第二个元素，只会移除栈顶元素。而 ADD 实际上会弹栈两次，再压入两数之和；MUL 同样会弹栈两次，再压入两数之积。

之前的栈实现只会向栈指针位置写入新值。新实现则可以在栈指针下方两个索引处写入和或积。例如，下面栈中的 12 会在加法后变成 15，而该位置比栈指针低两个索引：

加法之前：

[12 , 3, sp] (sp = 3)

加法之后：

[15, sp] (sp = 2)

这里，12 位于栈底，`sp` 指向栈上方的空位。

因此，需要一个信号来指示某列是否位于栈指针下方两个元素的位置。

下面的代码大量沿用了上一章的栈，但实现了本章所述的更新。具体来说：

- 我们将 NOP、PUSH 和 POP 替换为 NOP、PUSH、ADD 和 MUL。ADD 与 MUL 使栈指针减一，NOP 让栈指针保持不变，PUSH 则让栈指针加一，并把参数复制到栈顶。

```jsx
pragma circom 2.1.6;

include "circomlib/comparators.circom";
include "circomlib/gates.circom";

template AND3() {
  signal input in[3];
  signal output out;

  signal temp;
  temp <== in[0] * in[1];
  out <== temp * in[2];
}

// i is the column number
// bits is how many bits we need
// for the LessEqThan component
template ShouldCopy(i, bits) {
  signal input sp;
  signal input is_push;
  signal input is_nop;
  signal input is_add;
  signal input is_mul;

  // out = 1 if should copy
  signal output out;

  // sanity checks
  is_add + is_mul + is_push + is_nop === 1;
  is_nop * (1 - is_nop) === 0;
  is_push * (1 - is_push) === 0;
  is_add * (1 - is_add) === 0;
  is_mul * (1 - is_mul) === 0;

  // it's cheaper to compute ≠ 0 than > 0 to avoid
  // converting the number to binary
  signal spEqZero;
  signal spGteOne;
  spEqZero <== IsZero()(sp);
  spGteOne <== 1 - spEqZero;

  // it's cheaper to compute ≠ 0 and ≠ 1 than ≥ 2
  signal spEqOne;
  signal spGteTwo;
  spEqOne <== IsEqual()([sp, 1]);
  spGteTwo <== 1 - spEqOne * spEqZero;

  // the current column is 1 or more
  // below the stack pointer
  signal oneBelowSp <== LessEqThan(bits)([i, sp - 1]);

  // the current column is 3 or more
  // below the stack pointer
  signal threeBelowSP <== LessEqThan(bits)([i, sp - 3]);

  // condition A
  component a3A = AND3();
  a3A.in[0] <== spGteOne;
  a3A.in[1] <== oneBelowSp;
  a3A.in[2] <== is_push + is_nop;

  // condition B
  component a3B = AND3();
  a3B.in[0] <== spGteTwo;
  a3B.in[1] <== threeBelowSP;
  a3B.in[2] <== is_add + is_mul;

  component or = OR();
  or.a <== a3A.out;
  or.b <== a3B.out;
  out <== or.out;
}

template CopyStack(m) {
  var nBits = 4;
    signal output out[m];
    signal input sp;
    signal input is_add;
    signal input is_mul;
    signal input is_push;
    signal input is_nop;

    component ShouldCopys[m];
    signal copy[m];

    // loop over the columns
    for (var i = 0; i < m; i++) {
      ShouldCopys[i] = ShouldCopy(i, nBits);
      ShouldCopys[i].sp <== sp;
      ShouldCopys[i].is_add <== is_add;
      ShouldCopys[i].is_mul <== is_mul;
      ShouldCopys[i].is_push <== is_push;
      ShouldCopys[i].is_nop <== is_nop;
      out[i] <== ShouldCopys[i].out;
    }
}

// n is how many instructions we can handle
// since all the instructions might be push,
// our stack needs capacity of up to n
template ZKVM(n) {
  var NOP = 0;
  var PUSH = 1;
  var ADD = 2;
  var MUL = 3;

  signal input instr[2 * n];

  // we add one extra row for sp because
  // our algorithm always writes to the
  // next row and we don't want to conditionally
  // check for an array-out-of-bounds
  signal output sp[n + 1];

  signal output stack[n][n];

  var IS_NOP = 0;
  var IS_PUSH = 1;
  var IS_ADD = 2;
  var IS_MUL = 3;
  var ARG = 4;
  signal metaTable[n][5];

  // first instruction must be PUSH or NOP
  (instr[0] - PUSH) * (instr[0] - NOP) === 0;

  signal first_op_is_push;
  first_op_is_push <== IsEqual()([instr[0], PUSH]);

  // if the first op is NOP, we are forcing the first
  // value to be zero, but this is where the stack
  // pointer is, so it doesn't matter
  stack[0][0] <== first_op_is_push * instr[1];

  // initialize the rest of the first stack to be zero
  for (var i = 1; i < n; i++) {
    stack[0][i] <== 0;
  }

  // we fill out the 0th elements to avoid
  // uninitialzed signals
  sp[0] <== 0;
  sp[1] <== first_op_is_push;
  metaTable[0][IS_PUSH] <== first_op_is_push;
  metaTable[0][IS_NOP] <== 1 - first_op_is_push;
  metaTable[0][IS_ADD] <== 0;
  metaTable[0][IS_MUL] <== 0;
  metaTable[0][ARG] <== instr[1];

  // spBranch is what we add to the previous stack pointer
  // based on the opcode. Could be 1, 0, or -1 depending on the
  // opcode. Since the first opcode cannot be POP, -1 is not
  // an option here.
  var SAME = 0;
  var INC = 1;
  var DEC = 2;
  signal spBranch[n][3];
  spBranch[0][INC] <== first_op_is_push * 1;
  spBranch[0][SAME] <== (1 - first_op_is_push) * 0;
  spBranch[0][DEC] <== 0;

  // populate the metaTable and the stack pointer
  component EqPush[n];
  component EqNop[n];
  component EqAdd[n];
  component EqMul[n];

  component eqSP[n][n];
  signal eqSPAndIsPush[n][n];
  for (var i = 0; i < n; i++) {
    eqSPAndIsPush[0][i] <== 0;
  }

  // signals and components for copying
  component CopyStack[n];
  signal previousCellIfShouldCopy[n][n];
  for (var i = 0; i < n; i++) {
    previousCellIfShouldCopy[0][i] <== 0;
  }

  component eqSPMinus2[n][n];
  signal eqSPMinus2AndIsAdd[n][n];
  signal eqSPMinus2AndIsMul[n][n];
  for (var i = 0; i < n; i++) {
    eqSPMinus2AndIsAdd[0][i] <== 0;
    eqSPMinus2AndIsMul[0][i] <== 0;
  }

  // (the current column = sp - 2 and is_add) * sum
  signal eqSPMinus2AndIsAddWithValue[n][n];
  signal eqSPMinus2AndIsMulWithValue[n][n];

  signal sum_result[n][n];
  signal mul_result[n][n];
  for (var i = 0; i < n; i++) {
    eqSPMinus2AndIsAddWithValue[0][i] <== 0;
    eqSPMinus2AndIsMulWithValue[0][i] <== 0;
    sum_result[0][i] <== 0;
    mul_result[0][i] <== 0;
  }

  for (var i = 1; i < n; i++) {
    // check which opcode we are executing
    EqPush[i] = IsEqual();
    EqPush[i].in[0] <== instr[2 * i];
    EqPush[i].in[1] <== PUSH;
    metaTable[i][IS_PUSH] <== EqPush[i].out;

    EqNop[i] = IsEqual();
    EqNop[i].in[0] <== instr[2 * i];
    EqNop[i].in[1] <== NOP;
    metaTable[i][IS_NOP] <== EqNop[i].out;

    EqAdd[i] = IsEqual();
    EqAdd[i].in[0] <== instr[2 * i];
    EqAdd[i].in[1] <== ADD;
    metaTable[i][IS_ADD] <== EqAdd[i].out;

    EqMul[i] = IsEqual();
    EqMul[i].in[0] <== instr[2 * i];
    EqMul[i].in[1] <== MUL;
    metaTable[i][IS_MUL] <== EqMul[i].out;

    // carry out the sums and muls
    for (var j = 0; j < n - 1; j++) {
      sum_result[i][j] <== stack[i - 1][j] + stack[i - 1][j + 1];
      mul_result[i][j] <== stack[i - 1][j] * stack[i - 1][j + 1];
    }

    // these values cannot be used in practice because
    // the stack doesn't go that high.
    // However, we still need to initialize
    // them because every column checks
    // if it is sp - 1, even the last 2
    for (var j = n - 1; j < n; j++) {
      sum_result[i][j] <== 0;
      mul_result[i][j] <== 0;
    }

    // get the instruction argument
    metaTable[i][ARG] <== instr[2 * i + 1];

    // if it is a push, write to the stack
    // if it is a copy, write to the stack
    CopyStack[i] = CopyStack(n);
    CopyStack[i].sp <== sp[i];
    CopyStack[i].is_push <== metaTable[i][IS_PUSH];
    CopyStack[i].is_nop <== metaTable[i][IS_NOP];
    CopyStack[i].is_add <== metaTable[i][IS_ADD];
    CopyStack[i].is_mul <== metaTable[i][IS_MUL];
    for (var j = 0; j < n; j++) {
      previousCellIfShouldCopy[i][j] <== CopyStack[i].out[j] * stack[i - 1][j];

      eqSP[i][j] = IsEqual();
      eqSP[i][j].in[0] <== j;
      eqSP[i][j].in[1] <== sp[i];
      eqSPAndIsPush[i][j] <== eqSP[i][j].out * metaTable[i][IS_PUSH];

      // check if the column is two less
      // than the stack pointer
      // if so, we prepare to write the sum or
      // product here
      // if the current instruction is add or mul
      eqSPMinus2[i][j] = IsEqual();
      eqSPMinus2[i][j].in[0] <== j;
      eqSPMinus2[i][j].in[1] <== sp[i] - 2; // underflow doesn't matter

      eqSPMinus2AndIsAdd[i][j] <== eqSPMinus2[i][j].out * metaTable[i][IS_ADD];
      eqSPMinus2AndIsMul[i][j] <== eqSPMinus2[i][j].out * metaTable[i][IS_MUL];

      eqSPMinus2AndIsAddWithValue[i][j] <== eqSPMinus2AndIsAdd[i][j] * sum_result[i][j];
      eqSPMinus2AndIsMulWithValue[i][j] <== eqSPMinus2AndIsMul[i][j] * mul_result[i][j];
      // we will either
      // - PUSH
      // - COPY or implicilty assign 0
      // - ADD
      // - MUL
      stack[i][j] <== eqSPAndIsPush[i][j] * metaTable[i][ARG] + previousCellIfShouldCopy[i][j] + eqSPMinus2AndIsAddWithValue[i][j] + eqSPMinus2AndIsMulWithValue[i][j];
    }

    // write to the next row's stack pointer
    spBranch[i][INC] <== metaTable[i][IS_PUSH] * (sp[i] + 1);
    spBranch[i][SAME] <== metaTable[i][IS_NOP] * (sp[i]);
    spBranch[i][DEC] <== (metaTable[i][IS_ADD] + metaTable[i][IS_MUL]) * (sp[i] - 1);
    sp[i + 1] <== spBranch[i][INC] + spBranch[i][SAME] + spBranch[i][DEC];
  }
}

component main = ZKVM(5);

/* INPUT = {
    "instr": [1,3,1,6,1,2,3,0,3,0]
} */
```

## 如果只使用一个操作码，却为每个操作码都设置约束，效率不会很低吗？

在这个 ZKVM 中，我们会对栈里的每个元素执行加法和乘法，尽管实际上只使用其中一个结果。对于加法或乘法这样很轻量的操作，这不会造成严重影响。但如果存在哈希等重型操作的操作码，就会生成多得多的约束；即使只有栈顶需要进行哈希，也必须为栈中每个元素填充一个哈希电路。所有这些都会导致无谓的计算和高昂的计算开销。

可以使用一个（或两个）Quin Selector 判断栈中的哪些元素要作为操作码的输入，从而提升效率，但这仍意味着每轮栈迭代都需要哈希约束，即使该轮并不会使用它们。

*留给读者的练习：实现两个 Quin Selector，只对栈顶两个元素执行加法或乘法，而不是对整个栈执行。*

这种无谓重复未使用约束的低效问题，是原始 R1CS 的一项严重缺点，因为它不允许有条件地使用约束。

## 提升效率的解决方案

目前有两种能显著提升效率的方法：基于查找表的约束和递归证明。

- 查找表是一种算术化方案，只有实际使用的约束才属于表的一部分，随后 ZK 证明会证明每条指令都使用了表中的正确条目。
- 递归证明会为每条指令单独创建一个 ZK 证明，再使用另一个 ZK 证明验证输入证明有效，从而把这些证明组合起来。（请注意，ZK 中的验证算法本身也可以用算术电路建模。）

这些改进是 ZK Book 后续部分的主题。不过，为 VM 中的有效状态转移建模时所采用的思路，也适用于现代 VM。

## 进一步了解 ZKVM

最早提出的 ZKVM 是 TinyRAM，出自论文 [Snarks for C: verifying Program Executions Succinctly and in Zero Knowledge](https://eprint.iacr.org/2013/507.pdf)。用作者的话说，他们创建了“一台极简的 [RISC](https://en.wikipedia.org/wiki/Reduced_instruction_set_computer) 随机访问机器，采用 [Harvard 架构](https://en.wikipedia.org/wiki/Harvard_architecture)和按字寻址的随机访问内存”。[TinyRAM 规范](https://www.scipr-lab.org/doc/TinyRAM-spec-2.000.pdf)只需要 29 个操作码。

这篇论文开启了 ZKVM 研究，是一篇值得理解的高影响力论文。

大多数现代 ZKVM 基于 RISC-V 架构，但也存在基于 MIPS 架构的 ZKVM。ZK Layer 2 区块链则经常使用自己的定制架构。

Ye Zhang 介绍 Scroll 如何创建 ZKEVM 的[视频](https://www.youtube.com/watch?v=vuQGdbpDWcs)也很适合用来获得高层次理解。
