# 迭代器模块

> `core::iter` 深度解析

## 概述

迭代器是 Rust 中最强大的抽象之一，提供了惰性求值、链式操作和零成本抽象的数据处理能力。

```mermaid
graph TB
    subgraph "迭代器生态"
        TRAIT["Iterator Trait<br/>核心抽象"]

        TRAIT --> ADAPTERS["适配器<br/>懒惰转换"]
        TRAIT --> CONSUMERS["消费者<br/>触发计算"]
        TRAIT --> EXTENSIONS["扩展 Trait<br/>额外能力"]
    end

    ADAPTERS --> MAP["map, filter, take..."]
    CONSUMERS --> COLLECT["collect, fold, sum..."]
    EXTENSIONS --> DEI["DoubleEndedIterator"]
    EXTENSIONS --> ESI["ExactSizeIterator"]
    EXTENSIONS --> FI["FusedIterator"]
```

---

## Iterator Trait 核心

### 基本定义

```rust
pub trait Iterator {
    type Item;

    // 唯一必须实现的方法
    fn next(&mut self) -> Option<Self::Item>;

    // 提供 75+ 个默认方法...
}
```

### 方法分类总览

```mermaid
graph TD
    ITER["Iterator 方法"]

    ITER --> ADAPT["适配器方法<br/>返回新迭代器"]
    ITER --> CONSUME["消费者方法<br/>产生最终结果"]
    ITER --> QUERY["查询方法<br/>检查状态"]

    ADAPT --> A1["map, filter, take<br/>skip, chain, zip<br/>enumerate, flatten"]

    CONSUME --> C1["collect, fold, sum<br/>product, count<br/>max, min, find"]

    QUERY --> Q1["size_hint<br/>is_sorted<br/>all, any"]
```

---

## 适配器（Adapters）

适配器是**惰性的** - 它们不立即执行，而是返回一个新的迭代器。

### 完整适配器列表

```mermaid
graph TB
    subgraph "转换适配器"
        MAP["map(f)<br/>元素映射"]
        FILTER["filter(p)<br/>条件过滤"]
        FILTERMAP["filter_map(f)<br/>过滤+映射"]
        CLONED["cloned()<br/>Clone 元素"]
        COPIED["copied()<br/>Copy 元素"]
    end

    subgraph "数量控制适配器"
        TAKE["take(n)<br/>取前 n 个"]
        SKIP["skip(n)<br/>跳过前 n 个"]
        TAKEWHILE["take_while(p)<br/>条件取"]
        SKIPWHILE["skip_while(p)<br/>条件跳过"]
        STEPBY["step_by(n)<br/>步进取"]
    end

    subgraph "组合适配器"
        CHAIN["chain(iter)<br/>连接迭代器"]
        ZIP["zip(iter)<br/>配对压缩"]
        ENUMERATE["enumerate()<br/>添加索引"]
        FLATTEN["flatten()<br/>展平嵌套"]
        FLATMAP["flat_map(f)<br/>映射+展平"]
    end

    subgraph "结构适配器"
        REV["rev()<br/>反向迭代"]
        CYCLE["cycle()<br/>无限循环"]
        FUSE["fuse()<br/>融合"]
        PEEKABLE["peekable()<br/>可窥视"]
        INSPECT["inspect(f)<br/>调试观察"]
    end
```

### 常用适配器详解

#### map - 元素映射

```rust
fn map<B, F>(self, f: F) -> Map<Self, F>
where F: FnMut(Self::Item) -> B;
```

```mermaid
graph LR
    A["[1, 2, 3]"] -->|"map(|x| x * 2)"| B["[2, 4, 6]"]
```

#### filter - 条件过滤

```rust
fn filter<P>(self, predicate: P) -> Filter<Self, P>
where P: FnMut(&Self::Item) -> bool;
```

```mermaid
graph LR
    A["[1, 2, 3, 4, 5]"] -->|"filter(|x| x % 2 == 0)"| B["[2, 4]"]
```

#### chain - 连接迭代器

```rust
fn chain<U>(self, other: U) -> Chain<Self, U::IntoIter>
where U: IntoIterator<Item = Self::Item>;
```

```mermaid
graph LR
    A["[1, 2]"] --> CHAIN["chain"]
    B["[3, 4]"] --> CHAIN
    CHAIN --> C["[1, 2, 3, 4]"]
```

#### zip - 配对压缩

```rust
fn zip<U>(self, other: U) -> Zip<Self, U::IntoIter>
where U: IntoIterator;
```

```mermaid
graph LR
    A["[a, b, c]"] --> ZIP["zip"]
    B["[1, 2, 3]"] --> ZIP
    ZIP --> C["[(a,1), (b,2), (c,3)]"]
```

#### enumerate - 添加索引

```rust
fn enumerate(self) -> Enumerate<Self>;
```

```mermaid
graph LR
    A["[a, b, c]"] -->|"enumerate()"| B["[(0,a), (1,b), (2,c)]"]
```

#### flatten - 展平嵌套

```rust
fn flatten(self) -> Flatten<Self>
where Self::Item: IntoIterator;
```

```mermaid
graph LR
    A["[[1,2], [3,4]]"] -->|"flatten()"| B["[1, 2, 3, 4]"]
```

---

## 消费者（Consumers）

消费者是**贪婪的** - 它们立即消耗迭代器并产生结果。

### 消费者分类

```mermaid
graph TD
    CONSUMERS["消费者方法"]

    CONSUMERS --> AGG["聚合操作"]
    AGG --> COUNT["count() -> usize"]
    AGG --> SUM["sum() -> S"]
    AGG --> PRODUCT["product() -> P"]
    AGG --> MAXMIN["max()/min() -> Option"]

    CONSUMERS --> SEARCH["查找操作"]
    SEARCH --> FIND["find(p) -> Option"]
    SEARCH --> POSITION["position(p) -> Option<usize>"]
    SEARCH --> ALL["all(p) -> bool"]
    SEARCH --> ANY["any(p) -> bool"]

    CONSUMERS --> FOLD_OPS["折叠操作"]
    FOLD_OPS --> FOLD["fold(init, f) -> B"]
    FOLD_OPS --> REDUCE["reduce(f) -> Option"]
    FOLD_OPS --> FOREACH["for_each(f)"]

    CONSUMERS --> COLLECT_OPS["收集操作"]
    COLLECT_OPS --> COLLECT["collect() -> B"]
    COLLECT_OPS --> PARTITION["partition(p) -> (B, B)"]
    COLLECT_OPS --> UNZIP["unzip() -> (A, B)"]
```

### 常用消费者详解

#### collect - 收集到集合

```rust
fn collect<B: FromIterator<Self::Item>>(self) -> B;
```

```mermaid
graph LR
    A["Iterator"] -->|"collect::<Vec<_>>()"| B["Vec<T>"]
    A -->|"collect::<HashSet<_>>()"| C["HashSet<T>"]
    A -->|"collect::<String>()"| D["String"]
```

#### fold - 折叠操作

```rust
fn fold<B, F>(self, init: B, f: F) -> B
where F: FnMut(B, Self::Item) -> B;
```

```mermaid
graph LR
    subgraph "fold 执行过程"
        INIT["init: 0"]
        INIT -->|"f(0, 1)"| S1["1"]
        S1 -->|"f(1, 2)"| S2["3"]
        S2 -->|"f(3, 3)"| S3["6"]
    end

    INPUT["[1, 2, 3]"] --> INIT
    S3 --> OUTPUT["结果: 6"]
```

#### find - 查找元素

```rust
fn find<P>(&mut self, predicate: P) -> Option<Self::Item>
where P: FnMut(&Self::Item) -> bool;
```

#### partition - 分区

```rust
fn partition<B, F>(self, f: F) -> (B, B)
where B: Default + Extend<Self::Item>, F: FnMut(&Self::Item) -> bool;
```

```mermaid
graph LR
    A["[1,2,3,4,5]"] -->|"partition(|x| x%2==0)"| B["([2,4], [1,3,5])"]
```

---

## 扩展 Trait

### Trait 层级结构

```mermaid
graph TB
    ITER["Iterator<br/>━━━━━━<br/>核心 trait<br/>next()"]

    DEI["DoubleEndedIterator<br/>━━━━━━<br/>双向迭代<br/>next_back()"]

    ESI["ExactSizeIterator<br/>━━━━━━<br/>精确长度<br/>len()"]

    FI["FusedIterator<br/>━━━━━━<br/>融合保证<br/>None 后永远 None"]

    ITER --> DEI
    ITER --> ESI
    ITER --> FI

    style ITER fill:#e3f2fd
    style DEI fill:#f3e5f5
    style ESI fill:#e8f5e9
    style FI fill:#fff3e0
```

### DoubleEndedIterator

```rust
pub trait DoubleEndedIterator: Iterator {
    fn next_back(&mut self) -> Option<Self::Item>;

    // 提供的方法
    fn nth_back(&mut self, n: usize) -> Option<Self::Item>;
    fn rfold<B, F>(self, init: B, f: F) -> B;
    fn rfind<P>(&mut self, predicate: P) -> Option<Self::Item>;
}
```

```mermaid
graph LR
    subgraph "双向迭代"
        DATA["[a, b, c, d, e]"]
        DATA -->|"next()"| FRONT["→ a, b, c..."]
        DATA -->|"next_back()"| BACK["← e, d, c..."]
    end
```

### ExactSizeIterator

```rust
pub trait ExactSizeIterator: Iterator {
    fn len(&self) -> usize;
    fn is_empty(&self) -> bool;
}
```

**实现者**：
- `Range<T>`
- `slice::Iter`
- `vec::IntoIter`
- `Option::IntoIter`

### FusedIterator

```rust
pub trait FusedIterator: Iterator { }
```

**保证**：一旦返回 `None`，后续调用 `next()` 永远返回 `None`。

```mermaid
graph LR
    subgraph "FusedIterator 行为"
        A["Some(1)"] --> B["Some(2)"]
        B --> C["None"]
        C --> D["None"]
        D --> E["None"]
        E --> F["...永远 None"]
    end
```

---

## 适配器链示例

```mermaid
graph LR
    A["Vec.iter()<br/>Iterator&lt;Item=&T&gt;"]
    A -->|map| B["Map<br/>I, F"]
    B -->|filter| C["Filter<br/>I, P"]
    C -->|take| D["Take<br/>I"]
    D -->|enumerate| E["Enumerate<br/>I"]
    E -->|collect| F["Vec&lt;Item&gt;"]

    style A fill:#bbdefb
    style B fill:#c8e6c9
    style C fill:#fff9c4
    style D fill:#ffccbc
    style E fill:#f8bbd0
    style F fill:#bbdefb
```

**代码示例**：

```rust
let result: Vec<(usize, i32)> = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    .into_iter()
    .map(|x| x * 2)           // 映射
    .filter(|x| x % 3 == 0)   // 过滤
    .take(3)                  // 取前3个
    .enumerate()              // 添加索引
    .collect();               // 收集

// result = [(0, 6), (1, 12), (2, 18)]
```

---

## IntoIterator Trait

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;

    fn into_iter(self) -> Self::IntoIter;
}
```

### 三种迭代方式

```mermaid
graph TB
    subgraph "Vec 的三种迭代方式"
        VEC["Vec<T>"]

        VEC -->|"into_iter()"| OWNED["IntoIter<T><br/>━━━━━━<br/>获得所有权<br/>Item = T"]

        VEC -->|"iter()"| REF["Iter<'_, T><br/>━━━━━━<br/>不可变借用<br/>Item = &T"]

        VEC -->|"iter_mut()"| MUTREF["IterMut<'_, T><br/>━━━━━━<br/>可变借用<br/>Item = &mut T"]
    end
```

### for 循环糖语法

```rust
// 这个 for 循环...
for x in collection {
    // ...
}

// 被展开为...
let mut iter = IntoIterator::into_iter(collection);
loop {
    match iter.next() {
        Some(x) => { /* ... */ }
        None => break,
    }
}
```

---

## 创建迭代器

### 常用创建方法

```mermaid
graph TB
    subgraph "迭代器创建"
        ONCE["once(value)<br/>单元素迭代器"]
        REPEAT["repeat(value)<br/>无限重复"]
        EMPTY["empty()<br/>空迭代器"]
        FROM_FN["from_fn(f)<br/>从闭包创建"]
        SUCCESSORS["successors(first, f)<br/>后继序列"]
        RANGE["0..10<br/>范围迭代器"]
    end
```

```rust
use std::iter;

// 单元素
iter::once(42);           // [42]

// 无限重复
iter::repeat(1);          // [1, 1, 1, ...]

// 从闭包创建
iter::from_fn(|| Some(1)); // [1, 1, 1, ...]

// 后继序列
iter::successors(Some(1), |n| Some(n * 2));  // [1, 2, 4, 8, ...]

// 范围
(0..10);                  // [0, 1, 2, ..., 9]
(0..=10);                 // [0, 1, 2, ..., 10]
```

---

## 性能特性

### 零成本抽象

```mermaid
graph LR
    subgraph "编译优化"
        ITER_CODE["iter().map().filter().collect()"]
        ITER_CODE -->|"编译"| OPT["等效于手写循环"]
        OPT --> ASM["优化后的机器码"]
    end
```

**关键优化**：
- **内联**：小闭包被内联到调用点
- **融合**：多个适配器合并为单次遍历
- **消除**：未使用的中间值被消除

### 惰性求值

```mermaid
graph TD
    subgraph "惰性 vs 贪婪"
        LAZY["适配器（惰性）<br/>━━━━━━<br/>不立即执行<br/>返回新迭代器<br/>可组合"]

        EAGER["消费者（贪婪）<br/>━━━━━━<br/>立即执行<br/>产生最终结果<br/>终止链"]
    end

    LAZY -->|"触发"| EAGER
```

---

## 最佳实践

### 1. 优先使用迭代器

```rust
// 推荐 ✓
let sum: i32 = numbers.iter().sum();

// 不推荐 ✗
let mut sum = 0;
for n in &numbers {
    sum += n;
}
```

### 2. 善用 collect 的类型推断

```rust
// 显式类型
let v: Vec<_> = iter.collect();

// turbofish 语法
let v = iter.collect::<Vec<_>>();

// 让上下文推断
fn process(v: Vec<i32>) { }
process(iter.collect());
```

### 3. 避免不必要的克隆

```rust
// 推荐 ✓
strings.iter().map(|s| s.len())

// 不推荐 ✗
strings.iter().cloned().map(|s| s.len())
```

### 4. 使用合适的适配器

```rust
// 推荐 ✓
iter.filter_map(|x| x.parse().ok())

// 不推荐 ✗
iter.map(|x| x.parse().ok()).filter(|x| x.is_some()).map(|x| x.unwrap())
```

---

## 方法速查表

| 方法 | 类型 | 功能 |
|------|------|------|
| `map(f)` | 适配器 | 映射每个元素 |
| `filter(p)` | 适配器 | 过滤元素 |
| `filter_map(f)` | 适配器 | 过滤+映射 |
| `take(n)` | 适配器 | 取前 n 个 |
| `skip(n)` | 适配器 | 跳过前 n 个 |
| `chain(iter)` | 适配器 | 连接迭代器 |
| `zip(iter)` | 适配器 | 配对压缩 |
| `enumerate()` | 适配器 | 添加索引 |
| `flatten()` | 适配器 | 展平嵌套 |
| `rev()` | 适配器 | 反向 |
| `collect()` | 消费者 | 收集到集合 |
| `fold(init, f)` | 消费者 | 折叠 |
| `sum()` | 消费者 | 求和 |
| `count()` | 消费者 | 计数 |
| `find(p)` | 消费者 | 查找 |
| `any(p)` | 消费者 | 存在判断 |
| `all(p)` | 消费者 | 全部判断 |
