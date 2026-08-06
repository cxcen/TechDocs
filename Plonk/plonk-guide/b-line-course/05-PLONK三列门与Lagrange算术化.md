# 05. PLONK 三列门与 Lagrange 算术化：把整张表变成一个多项式条件

> **模块**：M5  
> **建议时间**：8–10 小时  
> **前置**：[02. 多项式、插值与消失多项式](02-多项式插值与消失多项式.md)、[04. 关系、算术电路与约束安全](04-关系算术电路与约束安全.md)  
> **本章产出**：从三列表格构造 witness/selector polynomials，并验证 gate numerator 可被 $Z_H$ 整除。

## 1. 本章解决什么问题

M4 的 direct verifier 要逐行检查。现在希望把 $n$ 行压缩成一个陈述：

$$
G(h)=0\quad\forall h\in H.
$$

再由消失多项式得到：

$$
Z_H(X)\mid G(X).
$$

这一步仍未使用承诺，也没有让 verifier 变成常数复杂度；它只完成 **arithmetization**。

```mermaid
flowchart LR
    Table["三列表格"] --> Eval["H 上的列取值"]
    Eval --> Poly["插值得到列多项式"]
    Poly --> Gate["构造 gate numerator G"]
    Gate --> Div["检查 Z_H 整除 G"]
```

## 2. 通用三列门

PLONK 的基础门写成：

$$
q_La+q_Rb+q_Mab+q_Oc+q_C=0.
$$

每一行有三条 witness wire $a,b,c$，以及五个由电路固定的 selector：

| Selector | 控制的项 |
|---|---|
| $q_L$ | 左输入 $a$ |
| $q_R$ | 右输入 $b$ |
| $q_M$ | 乘积 $ab$ |
| $q_O$ | 输出 $c$ |
| $q_C$ | 常数 |

例如：

- 加法 $a+b-c=0$：$(q_L,q_R,q_M,q_O,q_C)=(1,1,0,-1,0)$；
- 乘法 $ab-c=0$：$(0,0,1,-1,0)$；
- 常量平移 $a+3-c=0$：$(1,0,0,-1,3)$；
- no-op：五个 selector 全为 0。

这五个 selector 是 fixed circuit data，不由 prover 自由选择。

## 3. 贯穿例子的四行布局

关系仍是：

$$
y=(x+3)(x+5).
$$

本章把 M4 的最后一行从 padding 改作 public-input 行。这样局部算术和公开声明都能由基础门表达：

| 行 $i$ | $a_i$ | $b_i$ | $c_i$ | 非零 selectors | $PI_i$ | 等式 |
|---:|---|---|---|---|---:|---|
| 0 | $x$ | 0 | $r$ | $q_L=1,q_O=-1,q_C=3$ | 0 | $x+3-r=0$ |
| 1 | $x$ | 0 | $s$ | $q_L=1,q_O=-1,q_C=5$ | 0 | $x+5-s=0$ |
| 2 | $r$ | $s$ | $y$ | $q_M=1,q_O=-1$ | 0 | $rs-y=0$ |
| 3 | $y$ | 0 | 0 | $q_L=1$ | $-y_{pub}$ | $y-y_{pub}=0$ |

$PI$ 是 statement-dependent public-input column；$q_C$ 是 verification key 中固定的 circuit constant。两者不能混为一谈。

还需要跨位置相等：

$$
a_0=a_1,\quad c_0=a_2,\quad c_1=b_2,\quad c_2=a_3.
$$

本章只编码每行 gate。上述 copy constraints 留到 M6–M7；因此“gate quotient 通过”还不等于完整关系通过。

### 3.1 $\mathbb F_{17}$ 的具体 witness

取 $x=2$：

$$
r=5,\qquad s=7,\qquad y=5\cdot7=35\equiv1\pmod {17}.
$$

因此：

$$
\mathbf a=(2,2,5,1),
\quad
\mathbf b=(0,0,7,0),
\quad
\mathbf c=(5,7,1,0).
$$

本章新增的 public row 使 $\mathbf a$ 与 M2 的纯插值练习略有不同；这是布局变化，不是数学矛盾。

## 4. 从列取值到多项式

取：

$$
H=(1,\omega,\omega^2,\ldots,\omega^{n-1}),
\qquad Z_H(X)=X^n-1.
$$

插值 witness columns：

$$
A(\omega^i)=a_i,\quad
B(\omega^i)=b_i,\quad
C(\omega^i)=c_i.
$$

同样插值 fixed selector columns：

$$
Q_L(\omega^i)=q_{L,i},\ldots,Q_C(\omega^i)=q_{C,i},
$$

以及 public-input polynomial：

$$
PI(\omega^i)=PI_i.
$$

所有这些插值多项式的 degree 都小于 $n$。

## 5. Gate Polynomial

定义 gate numerator：

$$
\begin{aligned}
G(X)={}&Q_L(X)A(X)+Q_R(X)B(X)+Q_M(X)A(X)B(X)\\
&+Q_O(X)C(X)+Q_C(X)+PI(X).
\end{aligned}
$$

在 $X=\omega^i$ 处代入，恰好恢复第 $i$ 行：

$$
G(\omega^i)
=q_{L,i}a_i+q_{R,i}b_i+q_{M,i}a_ib_i+q_{O,i}c_i+q_{C,i}+PI_i.
$$

所以：

$$
\text{所有行 gate 成立}
\Longleftrightarrow
G(h)=0\ \forall h\in H.
$$

因为 $H$ 中元素互异，因式定理给出：

$$
G(h)=0\ \forall h\in H
\Longleftrightarrow
Z_H(X)\mid G(X).
$$

这正是“批量约束变成整除性”的核心转换。

## 6. Lagrange 视角为什么自然

设 $L_i$ 是 $H$ 上的 Lagrange basis：

$$
L_i(\omega^j)=\delta_{ij}.
$$

则：

$$
A(X)=\sum_{i=0}^{n-1}a_iL_i(X),
$$

selector 和 $PI$ 也一样。电路本来就是“每行放什么值”，所以 evaluation/Lagrange form 是最直接的表达；只有在做 polynomial multiplication、commitment 或 quotient 时，才需要切换表示。

## 7. Degree 账本

假设所有输入多项式 degree 至多 $n-1$，且暂不加入零知识 mask：

| 项 | Degree 上界 |
|---|---:|
| $Q_LA,Q_RB,Q_OC$ | $2n-2$ |
| $Q_C,PI$ | $n-1$ |
| $Q_MAB$ | $3n-3$ |
| $G$ | $3n-3$ |
| 若 $Z_H\mid G$，则 $G/Z_H$ | $2n-3$ |

Degree 账本决定：

- polynomial multiplication 需要多大的 FFT domain；
- SRS 至少支持多高 degree；
- quotient 要拆成多少 chunks；
- 加入 blinding 后是否仍在预算内。

不要把“degree 小于 $n$ 的列”误认为“由这些列相乘得到的约束多项式 degree 仍小于 $n$”。

## 8. 两种正确性检查

### 8.1 Evaluation-domain 检查

直接计算每个 $h\in H$ 的 $G(h)$，确认全为 0。这适合调试行号与 selector。

### 8.2 Coefficient-form 整除检查

构造 coefficient-form $G$，做 polynomial division：

$$
G(X)=Q_{gate}(X)Z_H(X)+R(X).
$$

合法表格必须满足 $R=0$。

两种检查应该 differential-test：一边失败而另一边通过，通常意味着 domain 顺序、插值或 polynomial multiplication 有 bug。

## 9. 实现骨架

```text
build_layout(public_y, private_x):
    compute r, s, y
    return witness columns and selector columns

interpolate_columns(domain, columns):
    use IFFT for every evaluation column

build_gate_numerator(A, B, C, selectors, PI):
    return QL*A + QR*B + QM*A*B + QO*C + QC + PI

check_gate_divisibility(G, ZH):
    quotient, remainder = divmod(G, ZH)
    return remainder == 0
```

开发期保留每个乘积项，失败时打印：

```text
row i:
  QL*A = ...
  QR*B = ...
  QM*A*B = ...
  QO*C = ...
  QC = ...
  PI = ...
```

这比只看到“quotient remainder 非零”更容易定位布局错误。

## 10. 常见错误与反例

| 错误 | 结果 |
|---|---|
| 把 row 0 的 $q_C=3$ 写成 $-3$ | honest witness 失败 |
| row 2 忘记 $q_O=-1$ | $y$ 未受局部门约束 |
| row 3 的 $PI$ 未吸收 public $y$ | proof 未绑定 statement |
| selector 行序与 witness 行序不同 | 插值后约束错位 |
| 只验证随机一点、尚无 commitment | prover 可临时编造 polynomial |
| 只检查 gate、不检查 copy | 同一逻辑 wire 可取不同值 |
| 在 $H$ 上逐点计算 $G/Z_H$ | 分母恒为 0 |

注意：本章的 coefficient division 是教学 verifier；高效 prover 会在扩展 coset 上构造 combined quotient，见 M8。

## 11. 必做测试

1. direct gate rows 与 $G|_H$ 对拍；
2. 合法 witness 的 remainder 为 0；
3. 修改 $r$、$s$ 或乘法输出，remainder 非 0；
4. 修改一个 selector，remainder 非 0；
5. 修改 public $y$，public row 失败；
6. 改坏 $a_0=a_1$ 但保持每行成立，证明 gate 检查仍可通过；
7. FFT 插值与朴素 Lagrange 插值对拍；
8. 检查计算出的 degree 不超过账本上界。

第 6 项不是协议 bug，而是下一章必须解决的问题。

## 12. 自测

1. Selector 为什么属于 circuit/VK，而不是 witness？
2. $PI$ 与 $Q_C$ 的信任来源有何不同？
3. 从 $G(\omega^i)$ 展开出第 $i$ 行门。
4. 为什么 $G|_H=0$ 等价于 $Z_H\mid G$？
5. $Q_MAB$ 的 degree 上界为什么是 $3n-3$？
6. 为什么 gate divisibility 不能保证 copy constraints？
7. 为什么 quotient 不能直接在 $H$ 上逐点相除？
8. 若加入 custom gate，应该先更新哪张账本？

## 13. 通过标准

- 能从表格手写全部 selector evaluations；
- 能插值得到所有列；
- 能逐项构造 $G$；
- evaluation 检查与整除检查对拍；
- 能展示“gate 全对但 copy 错”的恶意 witness；
- degree 预算由公式推导，而不是照抄常数。

---

上一篇：[04. 关系、算术电路与约束安全](04-关系算术电路与约束安全.md) · 下一篇：[06. Copy Constraints 与置换论证](06-Copy约束与置换论证.md) · [课程目录](README.md)
