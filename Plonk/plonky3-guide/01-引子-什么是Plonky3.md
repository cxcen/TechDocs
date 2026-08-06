# 第一章 · 引子：什么是 Plonky3

> 本章目标：用最少的前置知识，讲清楚"零知识证明—STARK—PIOP—Plonky3"这条从直觉到工程的概念链条，以及 Plonky3 在整个生态里的定位。读完你能回答：Plonky3 是什么、不是什么、为什么存在、该不该用它。

![Plonky3 概念封面](images/01-cover.png)

---

## 1.1 从一个朴素问题说起

假设你声称："我算了一道非常难的题，答案是 `Y`。"

别人凭什么相信你？最直接的办法是让你把**完整的计算过程**交出来，对方一步步重算一遍。但这有两个麻烦：

1. **计算量太大**——重算和原计算一样贵，验证毫无意义；
2. **过程里可能有隐私**——比如这笔交易、这个账户余额，你不想全盘托出。

**可验证计算（verifiable computation）** 想解决的就是：你交出一段极短的"证据" `π`，对方只看 `π`（和公开输入）就能在**远小于原计算量**的时间里，以**极高概率**确认你说的是真的。这就把"验证"和"计算"在代价上拉开了一个数量级以上的差距。

把上面两点叠加，就得到加密学里两个核心诉求：

| 诉求 | 含义 |
|---|---|
| **简洁性（succinctness）** | 证据很小、验证很快 |
| **零知识（zero-knowledge）** | 证据不泄露计算内部的任何信息 |

满足这两个性质的证明系统，统称为 **zk-SNARK**（zero-knowledge **S**uccinct **N**on-interactive **AR**gument of **K**nowledge）。

> 一句话直觉：zk-SNARK 让"我相信你算对了"这件事，变得比"你自己算一遍"还便宜，而且过程里看不见秘密。

---

## 1.2 SNARK 与 STARK

zk-SNARK 是一大类技术的总称。其中**构造证据的方式**各有不同，最常见的两大流派是：

- **基于椭圆曲线配对/KZG 的 SNARK**（如 Groth16、Plonk、Marlin）：
  - 证据极小、验证极快；
  - 但依赖"可信设置（trusted setup）"，且**抗量子安全性存疑**（椭圆曲线离散对数会被量子计算机攻破）。
- **基于哈希 + 多项式的 STARK**（Scalable Transparent ARgument of Knowledge）：
  - 只用哈希函数和有限域运算，**无需可信设置（transparent）**；
  - **后量子安全**；
  - 证据稍大、验证稍慢，但抗量子且透明。

Plonky3 属于 **STARK 这条路线**，而且是这条路线里目前工程上最被广泛复用的**底层工具包**之一。它的核心原料只有两样：

1. **有限域（finite field）** 上的算术；
2. **多项式（polynomial）** 与它们的承诺。

> 关键词对照：**Transparent**（无 trusted setup）= 只用公开的哈希/域参数；**Post-quantum** = 安全性不依赖会被 Shor 算法攻破的假设。STARK 两者都占。

---

## 1.3 什么是"多项式 IOP"（PIOP）

要理解 Plonky3 的全部设计，必须先理解一个核心抽象：**PIOP（Polynomial Interactive Oracle Proof）**。

把一次计算想象成一张**巨大的表格**（execution trace）：每一行是一个时间步，每一列是一个"寄存器/变量"的取值。于是"计算正确"这件事，被翻译成"这张表格满足一组**代数约束**"。例如：

- 转移约束：第 `i+1` 行的某列 = 第 `i` 行某列的函数（描述状态如何演化）；
- 边界约束：第一行/最后一行必须是某些公开值。

现在把这张表格的每一列都看成一个**多项式在某个域上的取值**（用插值/FFT 把"一列数"变成"一个多项式"）。于是"表格满足约束"就等价于"某些多项式方程在大量点上恒等于零"，进而可以归约为"某个多项式的次数足够低"。

**交互式 Oracle 证明（IOP）** 的玩法是：

1. 证明者把若干多项式"承诺（commit）"出来——只给承诺值，不给多项式本身；
2. 验证者发随机挑战点；
3. 证明者在挑战点处"打开（open）"多项式，证明取值与承诺一致、且多项式确实低次。

把这种交互用 **Fiat–Shamir 变换**压成非交互（用哈希模拟验证者的随机挑战），就得到一条可独立验证的证据。

> **PIOP = "用多项式 + 承诺 + 随机挑战"来构造的证明协议。** Plonky3 的定位就是：**实现 PIOP 所需的全部原语**。

一个 PIOP 通常由三块"乐高"拼成：

```
PIOP ≈ 约束系统(AIR)  +  多项式承诺(PCS)  +  随机性来源(Challenger/Fiat-Shamir)
```

Plonky3 把这三块都做成了**可插拔的、trait 抽象的模块**。这也是它名字里"3"的部分含义——它是 Polygon Zero 在 Plonky2 之后的**全面重构、解耦、模块化**版本。

---

## 1.4 Plonky3 到底是什么

官方 README 一句话定义：

> *"Plonky3 is a toolkit which provides a set of primitives, such as polynomial commitment schemes, for implementing polynomial IOPs (PIOPs)."*
>
> Plonky3 是一个**工具包**，提供实现多项式 IOP 所需的一组原语（如多项式承诺方案）。

几个关键定位，务必牢记，能帮你避免最常见的误解：

- ✅ **它是一组 Rust crate（库）**，不是一个"开箱即用的 zkVM"，也不是一种"电路编程语言"。
- ✅ **它主要服务 STARK**（univariate / multivariate STARK），原则上也能支撑 PLONK 类电路或其他 PIOP。
- ✅ **它极度模块化**：你可以自由组合"**有限域 × 哈希函数 × 承诺方案**"。
- ❌ 它**不自带**某个具体业务电路（如"以太坊状态转移"）；那些是上层 zkVM 的工作。
- ❌ 它**不是** Plonky2 的简单升级，而是一次**重新设计**：更小更快的小域、更清晰的 trait 边界、更现代的 PCS（如 STIR/Whir）。

一句话：**别人在"写证明系统"，Plonky3 在"造造证明系统的零件"。**

---

## 1.5 为什么值得学

- **产业复用度高**：很多 zkVM / 证明后端直接建立在 Plonky3 之上（或 fork 其 crate）。学会它，等于掌握了当代 STARK 工程的"通用底盘"。
- **代码即规范**：Plonky3 没有一篇"总论文"，**仓库本身就是最权威的规范**。它的每个 crate 对应一个清晰的概念，读代码是理解现代 STARK 最短的路径。
- **概念集大成**：FRI、DEEP-FRI、STIR、Whir、LogUp 查找、sumcheck、circle STARK、小域 SIMD……当代 STARK 的几乎所有关键技巧，都能在它的 crate 里找到生产级实现。

---

## 1.6 阅读建议与学习路径

本文档按"**从地基到塔尖**"的顺序组织，建议按章节顺序读：

```mermaid
flowchart LR
    A["1 引子<br/>全景"] --> B["2 架构<br/>crate 地图"]
    B --> C["3 有限域<br/>地基"]
    C --> D["4 多项式/编码/FFT"]
    D --> E["5 AIR<br/>约束系统"]
    E --> F["6 PCS/FRI<br/>承诺"]
    F --> G["7 STARK 全流程"]
    G --> H["8 Sumcheck/Lookup/FS"]
    H --> I["9 工程与性能"]
    I --> J["10 对比与实践"]
```

- **想快速建立直觉**：读 1、2、5、7、10；
- **想深入密码学**：重点啃 3、4、6、8；
- **想做工程接入**：看 2、5、7、9、10。

每章都遵循"**先给直觉与图，再给代码与形式化**"的写法，代码块引用的是 Plonky3 `main` 分支的真实 trait / 函数签名。

---

## 1.7 名词速查表

| 术语 | 全称 / 含义 |
|---|---|
| **PIOP** | Polynomial IOP，多项式交互式 Oracle 证明 |
| **AIR** | Algebraic Intermediate Representation，代数中间表示（约束系统） |
| **PCS** | Polynomial Commitment Scheme，多项式承诺方案 |
| **FRI** | Fast Reed–Solomon IOP of Proximity，快速低度测试 |
| **DEEP-FRI** | FRI 的"域外采样"增强，显著提升可靠性 |
| **LDE** | Low-Degree Extension，低度扩展（Reed–Solomon 编码） |
| **MMCS** | Mixed Matrix Commitment Scheme，混合矩阵（Merkle）承诺 |
| **2-adicity** | 域乘法群中 2-幂子群的大小，决定能做多大规模的 FFT |
| **Fiat–Shamir** | 用哈希把交互式协议压成非交互的变换 |
| **Challenger** | Plonky3 里驱动 Fiat–Shamir transcript 的抽象 |

---

下一章我们用一张"crate 地图"俯瞰整个仓库，看清这些零件是怎么分层摞起来的。
