# 在 ZK 中为栈数据结构建模

本教程介绍如何在 Circom 中创建一个栈。

先提醒一句——本章很长。

不过，创建栈相关 ZK 证明的策略，是下一章构建简单 ZK 虚拟机（ZKVM）时必不可少的知识。理解 ZKVM 工作原理所需的大部分准备工作，都提前放在了本章。

栈可以把一个数 **push（压入）**到栈顶、从栈顶 **pop（弹出）**一个数，或者**不作任何改变**。

栈本质上是可变的，但 Circom 只允许使用不可变数组。因此，必须用不可变数组为栈建模。每次栈发生变化（通过 pop 或 push），都要创建一个表示新栈状态的新数组。

这看起来可能效率不高，但 ZK 中的信号不可变，所以无法绕开（更高级的 ZK 技术可以优化这种情况，但相关讨论超出了本文范围）。

第一个要求是栈必须有固定的“最大高度”，它就是不可变数组的长度。为了记录栈已经“使用”了多少空间，我们使用一个称为“栈指针”的独立变量。它采用从零开始的索引，指示下一个元素要压入的位置。`sp` 指向数组中一个尚未使用、可压入新值的位置，也就是栈顶上方一个索引。下图展示了一个包含 10、16 和 20 三个元素的栈：

$$
\begin{align*}\texttt{sp}=3\\
\begin{array}{|c|c|c|c|c|c|c|}
\hline
\text{array}&10 & 16 & 20 & \circ &  \space &  \space \\ \hline
\text{index}&0 & 1 &2 &3 &4 &5\\
\hline
\end{array}
\end{align*}
$$

栈指针 `sp` 指向索引 3，该位置为空，并以 $\circ$ 表示。栈会忽略索引 3（当前 `sp`）及更大索引处的任何值。那些位置可以含有非零值，但电路不会将其视为栈的一部分。

假设将元素 `21` 压入栈中。这意味着递增栈指针，并复制之前的所有值。更新后栈的 `sp` = 4。

$$
\begin{align*}\texttt{sp}=4\\
\begin{array}{|c|c|c|c|c|c|}
\hline
10& 16 & 20 &  \circ &  \space \\ \hline
10 & 16 & 20 & 21&  \circ &  \space \\ \hline
\end{array}
\end{align*}
$$

如果从栈中执行 pop，`sp` 会递减，并且不会把值 `21` 复制到下一个栈状态：

$$
\begin{align*}\texttt{sp}=3\\
\begin{array}{|c|c|c|c|c|c|}
\hline
10 & 16 & 20 &  \circ &  \space \\ \hline
10 & 16 & 20 & 21&  \circ &  \space \\ \hline
10 & 16 & 20 & \circ&  \space &  \space \\ \hline
\end{array}
\end{align*}
$$

如果栈没有发生任何变化，我们仍然要经过某种计算步骤，只是把之前的所有值复制到下一个栈状态。

$$
\begin{align*}\texttt{sp}=3\\
\begin{array}{|c|c|c|c|c|c|}
\hline
10 & 16 & 20 &  \circ &  \space \\ \hline
10 & 16 & 20 & 21&  \circ &  \space \\ \hline
10 & 16 & 20 & \circ&  \space &  \space \\ \hline
10 & 16 & 20 & \circ&  \space &  \space \\ \hline
\end{array}
\end{align*}
$$

由于 `sp` 会在每轮迭代中变化，而信号一旦赋值就不能更新，因此每次都要把新值存入新的信号。于是，我们使用一个数组记录每轮迭代中的 `sp` 值：

$$

\begin{array}{|c||c|c|c|c|c|c|}
\hline
\texttt{sp}\downarrow & \\
\hline
3 & 10 & 16 & 20 &  \circ &  \space \\ \hline
4 & 10 & 16 & 20 & 21&  \circ &  \space \\ \hline
3 &10 & 16 & 20 & \circ&  \space &  \space \\ \hline
3 &10 & 16 & 20 & \circ&  \space &  \space \\ \hline
\end{array}
$$

正如栈有最大高度一样，我们能够更新它的次数也有上限，因为用于表示栈的表格（底层是一个二维 Circom 数组）必须具有固定大小。

最大规模取决于具体应用。在区块链场景中，执行 100 万条指令的可能性极低；但为计算密集型应用创建电路时，可能需要分配更大的电路，以容纳潜在的栈规模。

### 栈的约束

无论执行 push、pop 还是不作操作，从 0 到 `sp - 2`（含）之间的元素都需要复制到下一个栈状态，并约束为相等。在下面的例子中，栈指针原本为 4，随后执行 pop 操作，栈指针变为 3。栈索引 0、1、2 处的元素（橙色）被复制。

$$
\begin{array}{|c||c|c|c|c|c|c|}
\hline
\texttt{sp} & 0 & 1 & 2 &3 & 4 & 5\\
\hline
4 & \color{orange}{10} & \color{orange}{16} & \color{orange}{20} & 21&  \circ &  \space \\ \hline
3 &\color{orange}{10} & \color{orange}{16} & \color{orange}{20} & \circ&  \space &  \space \space \\ \hline
\end{array}
$$

如果指令是 pop，还必须要求新栈指针比旧值小 1。

### push 的约束

执行 push 时，直到 `sp - 1` 的所有值都必须复制到新栈中（请注意，栈指针指向空白区域，所以无须复制该位置）。`sp - 1` 处的值必须被约束为压入的值。

$$
\begin{array}{|c||c|c|c|c|c|c|}
\hline
\texttt{sp} & 0 & 1 & 2 &3 & 4 & 5\\
\hline
3 &\color{orange}{10} & \color{orange}{16} & \color{orange}{20} & \circ&  \space &  \space
\space \\ \hline
4 & \color{orange}{10} & \color{orange}{16} & \color{orange}{20} & 24&  \circ &  \space \\ \hline
\end{array}
$$

在上面的例子中，栈指针原为 3，我们复制了索引 0、1、2 处的元素。同时还要约束栈指针递增 1，也就是必须比之前大一。

### pop 的约束

执行 pop 时，直到 `sp - 2` 的所有值都必须复制到新栈中。我们约束栈指针递减一。

$$
\begin{array}{|c||c|c|c|c|c|c|}
\hline
\texttt{sp} & 0 & 1 & 2 &3 & 4 & 5\\
\hline
3 &\color{orange}{10} & \color{orange}{16} & \color{orange}{20} & \circ&  \space &  \space
\space \\ \hline
2 & \color{orange}{10} & \color{orange}{16} & \circ & &  &  \space \\ \hline
\end{array}
$$

### nop（不作操作）的约束

直到 `sp - 1` 的所有值都必须约束为相同值。`sp` 必须约束为与上一轮迭代的值相等。

$$
\begin{array}{|c||c|c|c|c|c|c|}
\hline
\texttt{sp} & 0 & 1 & 2 &3 & 4 & 5\\
\hline
3 &\color{orange}{10} & \color{orange}{16} & \color{orange}{20} & \circ&  \space &  \space
\space \\ \hline
3 & \color{orange}{10} & \color{orange}{16} & \color{orange}{20} & \circ& &  \space \\ \hline
\end{array}
$$

## 根据一组指令更新栈

在每轮指令迭代中，需要知道接下来是压入一个数、弹出栈顶，还是不作操作。我们分别用“操作码”或“指令”PUSH、POP 和 NOP 表示。

例如，假设给定指令 `PUSH 10 POP PUSH 16 PUSH 15 PUSH 4 NOP POP`。可以把这组指令表示成一个数组：

[PUSH, 10, POP, 0, PUSH, 16, PUSH, 15, PUSH, 4, NOP, 0, POP, 0]

不失一般性，可以用数字 1 表示 PUSH、2 表示 POP、0 表示 NOP，于是指令编码如下。

$$
\begin{matrix}
1 & 10 & 2 & 0 & 1 & 16 & 1 & 15 & 1 & 4 & 0 & 0 & 2&  0\\
\texttt{PUSH}&&\texttt{POP}&&\texttt{PUSH}&&\texttt{PUSH}&&\texttt{PUSH}&&\texttt{NOP}&&\texttt{POP}
\end{matrix}
$$

请注意，每条指令后面始终跟着一个常量。对 PUSH 来说，它是要压入的值；对 POP 和 NOP 来说，后面的常量会被忽略。把指令放在索引 0、2、4……处，就能每次跨两个位置遍历指令。换句话说，如果指令可能带参数，也可能不带参数，就要根据是否存在参数改变步长。这种有条件的步长会增加示例复杂度，因此我们把操作码等间隔放置，使每一步始终跨两个位置。

现在生成一个“metaTable”，说明每一执行行会发生哪项操作。如果增加 `is_push`、`is_pop` 和 `is_nop` 三列，用来指示当前激活的指令，就会得到下表。

<video src="https://r2media.rareskills.io/ZkStack/stack.mp4" type="video/mp4" autoplay loop muted controls></video>

最终结果如下；下一节会逐步重建这张表：

$$
\begin{array}{|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & 10 & - & - & - \\\hline0 & \color{orange}{1} & 0 & - & 1 & - & - & - & - \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & 16 & - & - & - \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & 16 & 15 & - & - \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & 16 & 15 & 4 & - \\\hline0 & 0 & \color{orange}{1} & - & 3 & 16 & 15 & 4 & - \\\hline0 & \color{orange}{1} & 0 & - & 3 & 16 & 15 & - & - \\\hline\end{array}
$$

`sp` 表示：如果当前指令是 push，下一个值应当写入哪里。如果指令是 `pop`，那么 `sp - 1` 所指的元素不应被复制。

## 填充表格

要填充从 `is_push` 到 `arg` 的几列，只需在循环中复制指令：如果当前指令是 PUSH，就把 `is_push` 设为 1，其他指令同理。得到的结果如下：

$$
\begin{array}{|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} \\\hline\color{orange}{1} & 0 & 0 & 10 & & & & & \\\hline0 & \color{orange}{1} & 0 & - & & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & & & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & & & & & \\\hline\color{orange}{1} & 0 & 0 & 4 & & & & & \\\hline0 & 0 & \color{orange}{1} & - & & & & & \\\hline0 & \color{orange}{1} & 0 & - & & & & & \\\hline\end{array}
$$

栈指针必须始终从零开始，所以可以直接把它写死：

$$
\begin{array}{|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_apop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} \\\hline\color{orange}{1} & 0 & 0 & 10 & \boxed{0}& & & & \\\hline0 & \color{orange}{1} & 0 & - & & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & & & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & & & & & \\\hline\color{orange}{1} & 0 & 0 & 4 & & & & & \\\hline0 & 0 & \color{orange}{1} & - & & & & & \\\hline0 & \color{orange}{1} & 0 & - & & & & & \\\hline\end{array}
$$

可以根据指令填充栈指针列的其余部分：PUSH 时递增，POP 时递减，NOP 时保持不变。请注意，我们根据当前行更新 `sp` 的*下一行*。因此，在第 0 轮迭代中，按如下方式更新第 1 行：

$$
\begin{array}{|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_apop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} \\\hline\color{orange}{1} & 0 & 0 & 10 & \boxed{0}& & & & \\\hline0 & \color{orange}{1} & 0 & - & \boxed{1}& & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & & & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & & & & & \\\hline\color{orange}{1} & 0 & 0 & 4 & & & & & \\\hline0 & 0 & \color{orange}{1} & - & & & & & \\\hline0 & \color{orange}{1} & 0 & - & & & & & \\\hline\end{array}
$$

也就是说，因为当前的 `is_push` 为 1，当前 `sp` 为 0，所以在下一个单元格写入 0 + 1。随后按如下方式填满 `sp` 列：

$$
\begin{array}{|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & & & & \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & & & & \\\hline0 & 0 & \color{orange}{1} & - & 3 & & & & \\\hline0 & \color{orange}{1} & 0 & - & 3 & & & & \\\hline\end{array}
$$

然后只需根据 `sp` 和 `arg` 两列，在 `is_push` 为 1 时将压入值写入相应单元格。请注意，此步骤只填充 `is_push == 1` 的行：

$$
\begin{array}{|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & 10 & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & 16 & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & & 15 & & \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & & & 4 & \\\hline0 & 0 & \color{orange}{1} & - & 3 & & & & \\\hline0 & \color{orange}{1} & 0 & - & 3 & & & & \\\hline\end{array}
$$

请注意，之前的值还没有复制过来——后面一节会解决这个问题。

### 处理第 0 行

第 0 行比较特殊，因为它不会复制上一行的任何值。为了避免根据当前是否为第 0 行进行分支，我们让表格比实际需要多一行，并从第 1 行开始构建约束。这样，每一行都始终从上一行复制。

在接下来的代码演示中，我们会把第 0 行的约束写死，因为之后再把“是否复制上一行的值”中的第 0 行作为特殊情况处理会很不方便。

## 复制上一行

现在需要确保各行之间正确复制值。一个单元格会把上方的值复制到下一行栈指针减 1 的位置为止。请记住，`sp` 指向栈上方的空位，因此 `sp - 1` 指向栈顶。

### 列与栈的术语

由于我们在表格中“横向”存储栈，因此把栈底称为第 0 列，把它上面的元素（如果存在）称为第 1 列，以此类推。这里所说的“列”，就是把栈底记为零时的栈“索引”。

### 复制约束

1. 如果 `is_push = 1`，必须复制栈中从 `0..sp - 1`（含）的所有元素。`sp` 处的单元格将包含新压入的值。这会复制整个栈。
2. 如果 `is_nop = 1`，必须复制栈中从 `0..sp - 1`（含）的所有元素。`sp` 处的单元格不会写入任何内容。这会复制整个栈。
3. 如果 `is_pop = 1`，必须复制栈中从 `0..sp - 2`（含）的所有元素。请记住，`sp` 指向栈上方的空单元格，因此 `sp - 1` 就是要弹出的值。`sp - 2` 及更低位置的所有元素都必须复制。这会复制除栈顶以外的所有元素。

当栈指针分别为 0 或 1 时，第 2、3 种情况会因下溢而成为边界情况。因此，需要额外的列来指示 `sp` 是否小于 2 或 1，并以不同方式处理复制。具体来说：

- 如果 `sp = 0`，不复制任何内容
- 如果 `sp = 1`，仅当指令为 NOP 或 PUSH 时才复制第 0 列的单元格

我们创建额外的列（`copy0`、`copy1`、`copy2`、`copy3`）作为标志，分别指示（`column0`、`column1`、`column2`、`column3`）中的值是否要复制到下一行。

$$
\begin{array}{|c|c|c|c|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} & \texttt{copy0} & \texttt{copy1} & \texttt{copy2} & \texttt{copy3} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & & & & & 10 & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & & & & & 16 & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & & & & & & 15 & & \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & & & & & & & 4 & \\\hline0 & 0 & \color{orange}{1} & - & 3 & & & & & & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & & & & & \\\hline\end{array}
$$

### 处理初始条件

为第 0 行创建约束时还有一个边界情况：如果尝试复制上一行的值，就会发生数组下溢。因此，需要单独处理这一行——以一种更“写死”的方式填充第一行，再从第 1 行开始迭代创建约束。

### 判断 `copy` 应为零还是一

回顾之前的约束：

1. 如果 `is_push = 1`，必须复制从 `0..sp - 1`（含）的所有值。`sp` 处的单元格将包含新值。
2. 如果 `is_nop = 1`，必须复制从 `0..sp - 1`（含）的所有值。`sp` 处的单元格不会写入任何内容。
3. 如果 `is_pop = 1`，必须复制从 `0..sp - 2`（含）的所有值。请记住，`sp` 指向栈上方的空单元格，因此 `sp - 1` 就是要弹出的值。`sp - 2` 及更低位置的所有元素都必须复制。

可以把它们归纳为条件 A 和 B：

A. 如果 `sp` 大于或等于 1，当前列比 `sp` 低 1 个或更多索引，并且当前指令是 PUSH 或 NOP，就应复制

B. 如果 `sp` 大于或等于 2，当前列比 `sp` 低 2 个或更多索引，并且当前指令是 POP，就应复制

如果当前列不满足上述任一条件，就不复制。其中包括：

- 当前列位于栈指针处或其上方
- 当前列比栈指针低 1，且当前指令是 pop
- 栈指针 = 0

按照以上规则填充表格。在第 0 行，`sp = 0`，所以所有 copy 列都为 0：

$$
\begin{array}{|c|c|c|c|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} & \texttt{copy0} & \texttt{copy1} & \texttt{copy2} & \texttt{copy3} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & 0 & 0 & 0 & 0 & 10 & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & & & & & 16 & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & & & & & & 15 & & \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & & & & & & & 4 & \\\hline0 & 0 & \color{orange}{1} & - & 3 & & & & & & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & & & & & \\\hline\end{array}
$$

在索引为 1 的行中，`sp` 大于或等于 1，但指令是 POP；任何一列都不满足条件 A 或 B，因此不复制：

$$
\begin{array}{|c|c|c|c|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} & \texttt{copy0} & \texttt{copy1} & \texttt{copy2} & \texttt{copy3} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & 0 & 0 & 0 & 0 & 10 & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & 0 & 0 & 0 & 0 & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & & & & & 16 & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & & & & & & 15 & & \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & & & & & & & 4 & \\\hline0 & 0 & \color{orange}{1} & - & 3 & & & & & & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & & & & & \\\hline\end{array}
$$

第 2 行的 `sp` 为 0，所以不复制任何内容：

$$
\begin{array}{|c|c|c|c|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} & \texttt{copy0} & \texttt{copy1} & \texttt{copy2} & \texttt{copy3} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & 0 & 0 & 0 & 0 & 10 & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & 0 & 0 & 0 & 0 & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & 0 & 0 & 0 & 0 & 16 & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & & & & & & 15 & & \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & & & & & & & 4 & \\\hline0 & 0 & \color{orange}{1} & - & 3 & & & & & & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & & & & & \\\hline\end{array}
$$

第 3 行的 `sp` 为 1，指令为 PUSH，因此第 0 列满足条件 A：

“如果 `sp` 大于或等于 1，当前列比 `sp` 低 1 个索引，并且当前指令是 PUSH 或 NOP，就应复制”

于是 `copy0` 被设为 1：

$$
\begin{array}{|c|c|c|c|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} & \texttt{copy0} & \texttt{copy1} & \texttt{copy2} & \texttt{copy3} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & 0 & 0 & 0 & 0 & 10 & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & 0 & 0 & 0 & 0 & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & 0 & 0 & 0 & 0 & 16 & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & \boxed{1} & 0 & 0 & 0 & 16 & 15 & & \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & & & & & & & 4 & \\\hline0 & 0 & \color{orange}{1} & - & 3 & & & & & & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & & & & & \\\hline\end{array}
$$

第 4 行的 `sp` 为 2，指令为 PUSH，所以第 0 列和第 1 列满足条件 A：

$$
\begin{array}{|c|c|c|c|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} & \texttt{copy0} & \texttt{copy1} & \texttt{copy2} & \texttt{copy3} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & 0 & 0 & 0 & 0 & 10 & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & 0 & 0 & 0 & 0 & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & 0 & 0 & 0 & 0 & 16 & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & 1 & 0 & 0 & 0 & 16 & 15 & & \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & \boxed{1} & \boxed{1} & 0 & 0 & 16 & 15 & 4 & \\\hline0 & 0 & \color{orange}{1} & - & 3 & & & & & & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & & & & & & & & \\\hline\end{array}
$$

第 5 行的 `sp` 为 3，指令为 NOP，所以第 0、1、2 列满足条件 A，也就是“如果 `sp` 大于或等于 1，当前列比 `sp` 低 1 个索引，并且当前指令是 PUSH 或 NOP，就应复制”：

$$
\begin{array}{|c|c|c|c|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} & \texttt{copy0} & \texttt{copy1} & \texttt{copy2} & \texttt{copy3} & & & & \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 &0 & 0& 0&0 & 10 & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & 0 & 0 & 0 & 0& & & & \\\hline\color{orange}{1} & 0 & 0 & \texttt{16} & 0 & 0 & 0& 0& 0& 16 & & & \\\hline\color{orange}{1} & 0 & 0 & \texttt{15} & 1 & 1& 0& 0& 0& 16& 15 & & \\\hline\color{orange}{1} & 0 & 0 & \texttt{4} & 2 & 1& 1& 0& 0&16 &15 & 4 & \\\hline0 & 0 & \color{orange}{1} & \texttt{-} & 3 &\boxed{1} &\boxed{1} & \boxed{1}&0 & 16& 15& 4& \\\hline0 & \color{orange}{1} & 0 & \texttt{-} & 1 & & & & & & & & \\\hline\end{array}
$$

第 6 行的 `sp` 为 1，指令为 POP，因此使用条件 B：

“如果 `sp` 大于或等于 2，当前列比 `sp` 低 2 个索引，并且当前指令是 POP，就应复制”

这意味着第 0 列和第 1 列会被复制：

$$
\begin{array}{|c|c|c|c|c|c|c|c|c||c|c|c|c|}\hline\texttt{is\_push} & \texttt{is\_pop} & \texttt{is\_nop} & \texttt{arg} & \texttt{sp} & \texttt{copy0} & \texttt{copy1} & \texttt{copy2} & \texttt{copy3} \\\hline\color{orange}{1} & 0 & 0 & 10 & 0 & 0 & 0 & 0 & 0 & 10 & & & \\\hline0 & \color{orange}{1} & 0 & - & 1 & 0 & 0 & 0 & 0 & & & & \\\hline\color{orange}{1} & 0 & 0 & 16 & 0 & 0 & 0 & 0 & 0 & 16 & & & \\\hline\color{orange}{1} & 0 & 0 & 15 & 1 & 1 & 0 & 0 & 0 & 16 & 15 & & \\\hline\color{orange}{1} & 0 & 0 & 4 & 2 & 1 & 1 & 0 & 0 & 16 & 15 & 4 & \\\hline0 & 0 & \color{orange}{1} & - & 3 & 1 & 1 & 1 & 0 & 16 & 15 & 4 & \\\hline0 & \color{orange}{1} & 0 & - & 1 & \boxed{1}&\boxed{1} & 0& 0& & & & \\\hline\end{array}
$$

### copy 条件的 Circom 实现

可以在 Circom 中创建一个专用组件，用于判断是否应当复制上方的值。

- A. 如果 `sp` 大于或等于 1，当前列比 `sp` 低 1 个索引，并且当前指令是 PUSH 或 NOP，就应复制
- B. 如果 `sp` 大于或等于 2，当前列比 `sp` 低 2 个索引，并且当前指令是 POP，就应复制

这个组件会在循环中使用，以判断某一列 `j` 是否应复制。如果某列需要复制，它就设置 `out = 1`。每一行的每一列都会应用该组件。

```jsx
include "circomlib/comparators.circom";
include "circomlib/gates.circom";

// RETURNS 1 IF ALL THE INPUTS ARE 1
template AND3() {
  signal input in[3];
  signal output out;

  signal temp;
  temp <== in[0] * in[1];
  out <== temp * in[2];
}

// j is the column number
// bits is how many bits we need
// for the LessEqThan component
template ShouldCopy(j, bits) {
  signal input sp;
  signal input is_pop;
  signal input is_push;
  signal input is_nop;

  // out = 1 if should copy
  signal output out;

  // sanity checks
  is_pop + is_push + is_nop === 1;
  is_nop * (1 - is_nop) === 0;
  is_push * (1 - is_push) === 0;
  is_pop * (1 - is_pop) === 0;

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
  spGteTwo <== (1 - spEqOne) * (1 - spEqZero);

  // the current column is 1 or more
  // below the stack pointer
  signal oneBelowSp <== LessEqThan(bits)([j, sp - 1]);

  // the current column is 2 or more
  // below the stack pointer
  signal twoBelowSP <== LessEqThan(bits)([j, sp - 2]);

  // condition A
  component a3A = AND3();
  a3A.in[0] <== spGteOne;
  a3A.in[1] <== oneBelowSp;
  a3A.in[2] <== is_push + is_nop;

  // condition B
  component a3B = AND3();
  a3B.in[0] <== spGteTwo;
  a3B.in[1] <== twoBelowSP;
  a3B.in[2] <== is_pop;

  component or = OR();
  or.a <== a3A.out;
  or.b <== a3B.out;
  out <== or.out;
}
```

可以在循环中使用上述组件，判断上一栈状态的哪些部分应复制到新栈。下面的模板返回一个由 0 和 1 组成的数组，用于指示要复制哪些列。例如，若共有 4 列，而前 2 列需要复制，它就返回 `[1, 1, 0, 0]`：

```jsx
template CopyStack(m) {
  var nBits = 4;
    signal output out[m];
    signal input sp;
    signal input is_pop;
    signal input is_push;
    signal input is_nop;

    component ShouldCopys[m];

    // loop over the columns
    for (var j = 0; j < m; j++) {
        ShouldCopys[j] = ShouldCopy(j, nBits);
        ShouldCopys[j].sp <== sp;
        ShouldCopys[j].is_pop <== is_pop;
        ShouldCopys[j].is_push <== is_push;
        ShouldCopys[j].is_nop <== is_nop;
        out[j] <== ShouldCopys[j].out;
    }
}
```

## 最终的栈实现

下面的代码是栈的最终实现，它把所有组件组合在一起。前面已经展示过 `ShouldCopy` 和 `CopyStack` 组件，因此读者可以直接跳到最后的 `StackBuilder` 组件。前面的组件来自本章之前各节。我们把它们放在一个文件里，方便读者复制并粘贴到 [zkrepl](https://zkrepl.dev) 中测试：

```jsx
include "circomlib/comparators.circom";
include "circomlib/gates.circom";

template AND3() {
  signal input in[3];
  signal output out;

  signal temp;
  temp <== in[0] * in[1];
  out <== temp * in[2];
}

// j is the column number
// bits is how many bits we need
// for the LessEqThan component
template ShouldCopy(j, bits) {
  signal input sp;
  signal input is_pop;
  signal input is_push;
  signal input is_nop;

  // out = 1 if should copy
  signal output out;

  // sanity checks
  is_pop + is_push + is_nop === 1;
  is_nop * (1 - is_nop) === 0;
  is_push * (1 - is_push) === 0;
  is_pop * (1 - is_pop) === 0;

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
  spGteTwo <== (1 - spEqOne) * (1 - spEqZero);

  // the current column is 1 or more
  // below the stack pointer
  signal oneBelowSp <== LessEqThan(bits)([j, sp - 1]);

  // the current column is 2 or more
  // below the stack pointer
  signal twoBelowSP <== LessEqThan(bits)([j, sp - 2]);

  // condition A
  component a3A = AND3();
  a3A.in[0] <== spGteOne;
  a3A.in[1] <== oneBelowSp;
  a3A.in[2] <== is_push + is_nop;

  // condition B
  component a3B = AND3();
  a3B.in[0] <== spGteTwo;
  a3B.in[1] <== twoBelowSP;
  a3B.in[2] <== is_pop;

  component or = OR();
  or.a <== a3A.out;
  or.b <== a3B.out;
  out <== or.out;
}

template CopyStack(m) {
  var nBits = 4;
    signal output out[m];
    signal input sp;
    signal input is_pop;
    signal input is_push;
    signal input is_nop;

    component ShouldCopys[m];
    signal copy[m];

    // loop over the columns
  for (var j = 0; j < m; j++) {
    ShouldCopys[j] = ShouldCopy(j, nBits);
    ShouldCopys[j].sp <== sp;
    ShouldCopys[j].is_pop <== is_pop;
    ShouldCopys[j].is_push <== is_push;
    ShouldCopys[j].is_nop <== is_nop;
    out[j] <== ShouldCopys[j].out;
  }
}

// n is how many instructions we can handle
// since all the instructions might be push,
// our stack needs capacity of up to n
template StackBuilder(n) {
  var NOP = 0;
  var PUSH = 1;
  var POP = 2;

  signal input instr[2 * n];

  // we add one extra row for sp because
  // our algorithm always writes to the
  // next row and we don't want to conditionally
  // check for an array-out-of-bounds
  signal output sp[n + 1];

  signal output stack[n][n];

  var IS_NOP = 0;
  var IS_PUSH = 1;
  var IS_POP = 2;
  var ARG = 3;

  // metaTable is the columns IS_NOP, IS_PUSH, IS_POP, ARG
  signal metaTable[n][4];

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
  // uninitialzed signals. For a particular
  // execution, we only want one possible witness
  // to correspond to a particular execution
  sp[0] <== 0;
  sp[1] <== first_op_is_push;
  metaTable[0][IS_PUSH] <== first_op_is_push;
  metaTable[0][IS_POP] <== 0;
  metaTable[0][IS_NOP] <== 1 - first_op_is_push;
  metaTable[0][ARG] <== instr[1];

  // spBranch is what we add to the previous
  // stack pointer based on the opcode.
  // Could be 1, 0, or -1 depending on the
  // opcode. Since the first opcode
  // cannot be POP, -1 is not an option here.
  var SAME = 0;
  var INC = 1;
  var DEC = 2;
  signal spBranch[n][3];
  spBranch[0][INC] <== first_op_is_push * 1;
  spBranch[0][SAME] <== (1 - first_op_is_push) * 0;
  spBranch[0][DEC] <== 0;

  // populate the first row of the metaTable
  // and the stack pointer
  component EqPush[n];
  component EqNop[n];
  component EqPop[n];

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

    EqPop[i] = IsEqual();
    EqPop[i].in[0] <== instr[2 * i];
    EqPop[i].in[1] <== POP;
    metaTable[i][IS_POP] <== EqPop[i].out;

    // get the instruction argument
    metaTable[i][ARG] <== instr[2 * i + 1];

    // if it is a push, write to the stack
    // if it is a copy, write to the stack
    CopyStack[i] = CopyStack(n);
    CopyStack[i].sp <== sp[i];
    CopyStack[i].is_push <== metaTable[i][IS_PUSH];
    CopyStack[i].is_nop <== metaTable[i][IS_NOP];
    CopyStack[i].is_pop <== metaTable[i][IS_POP];
    for (var j = 0; j < n; j++) {
      previousCellIfShouldCopy[i][j] <== CopyStack[i].out[j] * stack[i - 1][j];

      eqSP[i][j] = IsEqual();
      eqSP[i][j].in[0] <== j;
      eqSP[i][j].in[1] <== sp[i];
      eqSPAndIsPush[i][j] <== eqSP[i][j].out * metaTable[i][IS_PUSH];

      // we will either PUSH or COPY or implicilty assign 0
      stack[i][j] <== eqSPAndIsPush[i][j] * metaTable[i][ARG] + previousCellIfShouldCopy[i][j];
    }

    // write to the next row's stack pointer
    spBranch[i][INC] <== metaTable[i][IS_PUSH] * (sp[i] + 1);
    spBranch[i][SAME] <== metaTable[i][IS_NOP] * (sp[i]);
    spBranch[i][DEC] <== metaTable[i][IS_POP] * (sp[i] - 1);
    sp[i + 1] <== spBranch[i][INC] + spBranch[i][SAME] + spBranch[i][DEC];
  }
}

component main = StackBuilder(3);

/* INPUT = {
  "instr": [1, 16, 1, 20, 1, 22]
} */
```

## 小结

要为随时间变化的数据结构建模，需要为所有可能的状态转移编写约束，再根据标志激活相应的状态转移。这些标志受到约束，必须与特定状态转移的指令相匹配。

栈数据结构的算术化可能看起来令人生畏，但现在我们已经掌握了理解如何构建基于栈的 ZKVM 所需的几乎全部知识，下一章就会介绍它。要创建基于栈的 ZKVM，只需修改本章介绍的指令及其对应约束，使其匹配 ZKVM 的操作码。

一般来说，大多数有意义的计算都可以建模为：从一个初始状态开始，逐步更新，直到得到最终结果。本章展示的栈只是这种模式的一个特例。
