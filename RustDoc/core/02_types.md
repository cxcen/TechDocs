# 类型系统模块

> `core::marker`, `core::any`, `core::convert`, `core::clone`, `core::cmp`, `core::default` 深度解析

## 概述

Rust 的类型系统模块提供了语言核心的类型抽象和标记 trait，是 Rust 类型安全的基石。

```mermaid
graph TB
    subgraph "类型系统架构"
        MARKER["marker<br/>标记 Trait"]
        ANY["any<br/>类型反射"]
        CONVERT["convert<br/>类型转换"]
        CLONE["clone<br/>值克隆"]
        CMP["cmp<br/>比较操作"]
        DEFAULT["default<br/>默认值"]
    end

    MARKER --> SIZED["Sized"]
    MARKER --> COPY["Copy"]
    MARKER --> SEND["Send"]
    MARKER --> SYNC["Sync"]

    ANY --> TYPEID["TypeId"]
    ANY --> DOWNCAST["向下转型"]

    CONVERT --> FROM["From/Into"]
    CONVERT --> TRYFROM["TryFrom/TryInto"]

    CLONE --> DEEP["深拷贝"]
    CMP --> ORD["Ord/PartialOrd"]
    CMP --> EQ["Eq/PartialEq"]

    DEFAULT --> INIT["默认初始化"]
```

---

## core::marker 模块

### 概述

`marker` 模块定义了编译器用来标记类型属性的特殊 trait，它们通常没有方法，仅用于约束泛型。

### 核心标记 Trait

```mermaid
graph TD
    subgraph "标记 Trait 层级"
        SIZED["Sized<br/>━━━━━━<br/>编译时已知大小<br/>默认绑定"]

        COPY["Copy<br/>━━━━━━<br/>按位复制<br/>隐式复制语义"]

        CLONE["Clone<br/>━━━━━━<br/>显式克隆<br/>可能涉及堆分配"]

        SEND["Send<br/>━━━━━━<br/>可跨线程转移所有权"]

        SYNC["Sync<br/>━━━━━━<br/>可跨线程共享引用"]

        UNPIN["Unpin<br/>━━━━━━<br/>可安全移动<br/>大多数类型默认实现"]
    end

    COPY -->|要求| CLONE
    SYNC -.->|"&T: Send"| SEND
```

#### Sized Trait

```rust
pub trait Sized {
    // 无方法，编译器自动实现
}
```

**特性**：
- 编译时已知类型大小
- 所有类型参数默认绑定 `Sized`
- 使用 `?Sized` 放宽约束以支持 DST

```mermaid
graph LR
    subgraph "Sized vs ?Sized"
        SIZED_T["T: Sized<br/>━━━━━━<br/>固定大小类型<br/>可在栈上分配"]

        UNSIZED["T: ?Sized<br/>━━━━━━<br/>可能是 DST<br/>如 [T], str, dyn Trait"]
    end

    SIZED_T -->|放宽| UNSIZED
```

#### Copy Trait

```rust
pub trait Copy: Clone {
    // 无方法，标记 trait
}
```

**特性**：
- 按位复制，不转移所有权
- 必须实现 `Clone`
- 编译器在赋值/传参时隐式复制

**可 Copy 的类型**：
- 所有基本数值类型
- `bool`, `char`
- 元组（元素都是 Copy）
- 数组（元素是 Copy）
- 共享引用 `&T`

**不可 Copy 的类型**：
- `String`, `Vec<T>`
- `Box<T>`
- 可变引用 `&mut T`

#### Send 和 Sync Trait

```mermaid
graph TB
    subgraph "线程安全标记"
        SEND["Send<br/>━━━━━━<br/>T 可以安全地<br/>转移到另一个线程"]

        SYNC["Sync<br/>━━━━━━<br/>&T 可以安全地<br/>在多线程间共享"]

        RELATION["关系<br/>━━━━━━<br/>T: Sync ⟺ &T: Send"]
    end

    SEND --- RELATION
    SYNC --- RELATION
```

**实现规则**：
- 大多数类型自动实现 `Send` 和 `Sync`
- `Rc<T>` 不是 `Send`（引用计数非原子）
- `RefCell<T>` 不是 `Sync`（借用检查非线程安全）
- 原始指针不是 `Send` 也不是 `Sync`

#### PhantomData<T>

```rust
pub struct PhantomData<T: ?Sized>;
```

**用途**：
- 标记类型参数的"所有权"
- 用于裸指针包装器
- 影响协变性/逆变性

```mermaid
graph LR
    subgraph "PhantomData 使用场景"
        PD["PhantomData<T>"]

        PD --> OWN["所有权标记<br/>结构体逻辑上拥有 T"]
        PD --> LIFE["生命周期标记<br/>与某个生命周期关联"]
        PD --> VAR["方差控制<br/>控制泛型参数的方差"]
    end
```

---

## core::any 模块

### 概述

`any` 模块提供运行时类型反射能力，允许在运行时检查和转换类型。

### TypeId

```rust
pub struct TypeId {
    // 内部实现细节
}

impl TypeId {
    pub fn of<T: 'static + ?Sized>() -> TypeId;
}
```

**特性**：
- 每个类型有唯一的 `TypeId`
- 仅适用于 `'static` 类型
- 支持 `PartialEq`、`Eq`、`Hash`

### Any Trait

```rust
pub trait Any: 'static {
    fn type_id(&self) -> TypeId;
}
```

**向下转型方法**：

```mermaid
graph TD
    subgraph "Any 向下转型"
        ANY["&dyn Any"]

        ANY --> IS["is::<T>()<br/>检查类型"]
        ANY --> DOWN_REF["downcast_ref::<T>()<br/>转为 &T"]
        ANY --> DOWN_MUT["downcast_mut::<T>()<br/>转为 &mut T"]

        IS --> BOOL["bool"]
        DOWN_REF --> OPT_REF["Option<&T>"]
        DOWN_MUT --> OPT_MUT["Option<&mut T>"]
    end
```

---

## core::convert 模块

### 概述

`convert` 模块提供类型之间的转换 trait。

### From 和 Into

```mermaid
graph LR
    subgraph "From/Into 关系"
        FROM["From<T><br/>━━━━━━<br/>fn from(T) -> Self"]
        INTO["Into<U><br/>━━━━━━<br/>fn into(self) -> U"]

        FROM -->|"自动派生"| INTO

        NOTE["实现 From<T> for U<br/>自动获得 Into<U> for T"]
    end
```

```rust
pub trait From<T>: Sized {
    fn from(value: T) -> Self;
}

pub trait Into<T>: Sized {
    fn into(self) -> T;
}
```

**最佳实践**：
- 优先实现 `From`
- 使用 `Into` 作为泛型约束

### TryFrom 和 TryInto

```mermaid
graph LR
    subgraph "可失败转换"
        TRYFROM["TryFrom<T><br/>━━━━━━<br/>fn try_from(T)<br/>-> Result<Self, Error>"]

        TRYINTO["TryInto<U><br/>━━━━━━<br/>fn try_into(self)<br/>-> Result<U, Error>"]

        TRYFROM -->|"自动派生"| TRYINTO
    end
```

```rust
pub trait TryFrom<T>: Sized {
    type Error;
    fn try_from(value: T) -> Result<Self, Self::Error>;
}
```

### AsRef 和 AsMut

```mermaid
graph TB
    subgraph "引用转换"
        ASREF["AsRef<T><br/>━━━━━━<br/>fn as_ref(&self) -> &T<br/>廉价引用转换"]

        ASMUT["AsMut<T><br/>━━━━━━<br/>fn as_mut(&mut self) -> &mut T<br/>廉价可变引用转换"]

        EX1["String::as_ref() -> &str"]
        EX2["Vec<T>::as_ref() -> &[T]"]
    end

    ASREF --> EX1
    ASREF --> EX2
```

### 转换 Trait 选择指南

```mermaid
graph TD
    START["需要类型转换"]

    START --> Q1{"是否消耗原值？"}
    Q1 -->|是| Q2{"可能失败？"}
    Q1 -->|否| Q3{"需要可变？"}

    Q2 -->|是| TRYFROM["TryFrom/TryInto"]
    Q2 -->|否| FROM["From/Into"]

    Q3 -->|是| ASMUT["AsMut"]
    Q3 -->|否| ASREF["AsRef"]
```

---

## core::clone 模块

### Clone Trait

```rust
pub trait Clone: Sized {
    fn clone(&self) -> Self;

    fn clone_from(&mut self, source: &Self) {
        *self = source.clone();
    }
}
```

**Clone vs Copy**：

| 特性 | Copy | Clone |
|------|------|-------|
| 实现方式 | 按位复制 | 可自定义 |
| 调用方式 | 隐式 | 显式 `.clone()` |
| 性能 | 恒定 O(1) | 可能 O(n) |
| 堆分配 | 不允许 | 可能涉及 |
| derive | `#[derive(Copy)]` | `#[derive(Clone)]` |

---

## core::cmp 模块

### 比较 Trait 层级

```mermaid
graph TB
    subgraph "比较 Trait 层级"
        PARTIALEQ["PartialEq<br/>━━━━━━<br/>部分等价关系<br/>eq, ne"]

        EQ["Eq<br/>━━━━━━<br/>完全等价关系<br/>反身性保证"]

        PARTIALORD["PartialOrd<br/>━━━━━━<br/>部分序关系<br/>partial_cmp, lt, le, gt, ge"]

        ORD["Ord<br/>━━━━━━<br/>全序关系<br/>cmp, max, min, clamp"]

        PARTIALEQ -->|超 trait| EQ
        PARTIALEQ -->|超 trait| PARTIALORD
        EQ -->|超 trait| ORD
        PARTIALORD -->|超 trait| ORD
    end
```

### PartialEq vs Eq

```mermaid
graph LR
    subgraph "等价关系"
        PARTIAL["PartialEq<br/>━━━━━━<br/>对称性: a == b ⟹ b == a<br/>传递性: a == b ∧ b == c ⟹ a == c"]

        FULL["Eq<br/>━━━━━━<br/>反身性: a == a<br/>（额外保证）"]

        EX["f32, f64 只实现 PartialEq<br/>因为 NaN != NaN"]
    end

    PARTIAL --> FULL
    PARTIAL --> EX
```

### PartialOrd vs Ord

```rust
pub trait PartialOrd<Rhs = Self>: PartialEq<Rhs> {
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;
}

pub trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, other: &Self) -> Ordering;
}
```

**Ordering 枚举**：

```mermaid
graph LR
    subgraph "Ordering"
        LESS["Less<br/>小于"]
        EQUAL["Equal<br/>等于"]
        GREATER["Greater<br/>大于"]
    end
```

---

## core::default 模块

### Default Trait

```rust
pub trait Default: Sized {
    fn default() -> Self;
}
```

**常见默认值**：

| 类型 | 默认值 |
|------|--------|
| 数值类型 | `0` |
| `bool` | `false` |
| `char` | `'\0'` |
| `String` | `""` |
| `Vec<T>` | `vec![]` |
| `Option<T>` | `None` |

**使用场景**：

```mermaid
graph TB
    subgraph "Default 使用场景"
        DEFAULT["Default::default()"]

        DEFAULT --> STRUCT["结构体部分初始化<br/>..Default::default()"]
        DEFAULT --> OPTION["Option::unwrap_or_default()"]
        DEFAULT --> MEM["mem::take()"]
        DEFAULT --> GENERIC["泛型约束<br/>T: Default"]
    end
```

---

## Trait 实现关系总览

```mermaid
graph TB
    subgraph "标记 Trait"
        SIZED["Sized"]
        COPY["Copy"]
        SEND["Send"]
        SYNC["Sync"]
        UNPIN["Unpin"]
    end

    subgraph "值操作 Trait"
        CLONE["Clone"]
        DEFAULT["Default"]
    end

    subgraph "比较 Trait"
        PARTIALEQ["PartialEq"]
        EQ["Eq"]
        PARTIALORD["PartialOrd"]
        ORD["Ord"]
    end

    subgraph "转换 Trait"
        FROM["From"]
        INTO["Into"]
        TRYFROM["TryFrom"]
        TRYINTO["TryInto"]
        ASREF["AsRef"]
        ASMUT["AsMut"]
    end

    COPY --> CLONE
    EQ --> PARTIALEQ
    ORD --> EQ
    ORD --> PARTIALORD
    PARTIALORD --> PARTIALEQ
    FROM --> INTO
    TRYFROM --> TRYINTO
```

---

## 设计原则

### 1. 标记 Trait 的力量
- 编译时类型检查
- 零运行时开销
- 防止数据竞争（Send/Sync）

### 2. 转换的人体工程学
- 实现 `From` 自动获得 `Into`
- `?` 运算符配合 `TryFrom`
- 链式转换

### 3. 比较的灵活性
- 支持部分序（浮点数）
- 支持跨类型比较
- derive 宏简化实现

### 4. 默认值的便利性
- 结构体更新语法
- Option 的安全解包
- 泛型代码的灵活性
