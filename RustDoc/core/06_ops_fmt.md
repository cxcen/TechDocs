# 运算符与格式化模块

> `core::ops`, `core::fmt` 深度解析

## 概述

运算符重载和格式化是 Rust 中最常用的 trait 系统应用，它们让自定义类型能够无缝融入语言。

```mermaid
graph TB
    subgraph "ops 模块"
        ARITH["算术运算符<br/>Add, Sub, Mul, Div, Rem, Neg"]
        BIT["位运算符<br/>BitAnd, BitOr, BitXor, Not, Shl, Shr"]
        CMP_OP["比较运算符<br/>通过 cmp 模块"]
        INDEX["索引运算符<br/>Index, IndexMut"]
        DEREF["解引用运算符<br/>Deref, DerefMut"]
        FN["函数调用<br/>Fn, FnMut, FnOnce"]
        RANGE["范围运算符<br/>Range, RangeBounds"]
        ASSIGN["赋值运算符<br/>AddAssign, SubAssign, ..."]
    end

    subgraph "fmt 模块"
        DISPLAY["Display<br/>用户友好输出"]
        DEBUG["Debug<br/>调试输出"]
        FORMATTER["Formatter<br/>格式化控制"]
        WRITE_M["Write trait<br/>写入目标"]
    end
```

---

## core::ops 模块

### 算术运算符 Trait

```mermaid
graph TB
    subgraph "二元算术运算符"
        ADD["Add<br/>a + b"]
        SUB["Sub<br/>a - b"]
        MUL["Mul<br/>a * b"]
        DIV["Div<br/>a / b"]
        REM["Rem<br/>a % b"]
    end

    subgraph "一元算术运算符"
        NEG["Neg<br/>-a"]
    end
```

#### Add Trait

```rust
pub trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

**使用示例**：

```rust
impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}
```

#### 所有算术 Trait

| Trait | 运算符 | 方法 |
|-------|--------|------|
| `Add<Rhs>` | `+` | `add(self, rhs)` |
| `Sub<Rhs>` | `-` | `sub(self, rhs)` |
| `Mul<Rhs>` | `*` | `mul(self, rhs)` |
| `Div<Rhs>` | `/` | `div(self, rhs)` |
| `Rem<Rhs>` | `%` | `rem(self, rhs)` |
| `Neg` | `-` (一元) | `neg(self)` |

### 位运算符 Trait

```mermaid
graph TB
    subgraph "位运算符"
        BITAND["BitAnd<br/>a & b"]
        BITOR["BitOr<br/>a | b"]
        BITXOR["BitXor<br/>a ^ b"]
        NOT["Not<br/>!a"]
        SHL["Shl<br/>a << b"]
        SHR["Shr<br/>a >> b"]
    end
```

| Trait | 运算符 | 方法 |
|-------|--------|------|
| `BitAnd<Rhs>` | `&` | `bitand(self, rhs)` |
| `BitOr<Rhs>` | `\|` | `bitor(self, rhs)` |
| `BitXor<Rhs>` | `^` | `bitxor(self, rhs)` |
| `Not` | `!` | `not(self)` |
| `Shl<Rhs>` | `<<` | `shl(self, rhs)` |
| `Shr<Rhs>` | `>>` | `shr(self, rhs)` |

### 复合赋值运算符

```mermaid
graph LR
    subgraph "复合赋值"
        ADD_ASSIGN["AddAssign<br/>a += b"]
        SUB_ASSIGN["SubAssign<br/>a -= b"]
        MUL_ASSIGN["MulAssign<br/>a *= b"]
        DIV_ASSIGN["DivAssign<br/>a /= b"]
    end
```

```rust
pub trait AddAssign<Rhs = Self> {
    fn add_assign(&mut self, rhs: Rhs);
}
```

**注意**：复合赋值运算符接收 `&mut self`，不返回值。

### 索引运算符

```mermaid
graph LR
    subgraph "索引操作"
        INDEX["Index<Idx><br/>━━━━━━<br/>container[idx]<br/>返回 &T"]

        INDEX_MUT["IndexMut<Idx><br/>━━━━━━<br/>container[idx] = val<br/>返回 &mut T"]
    end

    INDEX -->|"超 trait"| INDEX_MUT
```

```rust
pub trait Index<Idx: ?Sized> {
    type Output: ?Sized;
    fn index(&self, index: Idx) -> &Self::Output;
}

pub trait IndexMut<Idx: ?Sized>: Index<Idx> {
    fn index_mut(&mut self, index: Idx) -> &mut Self::Output;
}
```

**语法糖**：
- `container[index]` 等价于 `*container.index(index)`
- `container[index] = value` 使用 `IndexMut`

### 解引用运算符

```mermaid
graph TB
    subgraph "Deref/DerefMut"
        DEREF["Deref<br/>━━━━━━<br/>*ptr（不可变）<br/>自动解引用"]

        DEREF_MUT["DerefMut<br/>━━━━━━<br/>*ptr（可变）<br/>自动解引用"]
    end

    DEREF -->|"超 trait"| DEREF_MUT
```

```rust
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}

pub trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

#### Deref 强制转换

```mermaid
graph LR
    subgraph "自动解引用链"
        STRING["&String"]
        STRING -->|"Deref"| STR["&str"]

        VEC["&Vec<T>"]
        VEC -->|"Deref"| SLICE["&[T]"]

        BOX["&Box<T>"]
        BOX -->|"Deref"| T["&T"]
    end
```

**规则**：
- `&T` 自动强制转换为 `&U`，当 `T: Deref<Target=U>`
- `&mut T` 自动强制转换为 `&mut U`，当 `T: DerefMut<Target=U>`
- `&mut T` 自动强制转换为 `&U`，当 `T: Deref<Target=U>`

### 闭包 Trait

```mermaid
graph TB
    subgraph "闭包 Trait 层级"
        FNONCE["FnOnce<br/>━━━━━━<br/>获取所有权 (self)<br/>只能调用一次"]

        FNMUT["FnMut<br/>━━━━━━<br/>可变借用 (&mut self)<br/>可调用多次"]

        FN["Fn<br/>━━━━━━<br/>不可变借用 (&self)<br/>可调用多次"]
    end

    FNONCE -->|"超 trait"| FNMUT
    FNMUT -->|"超 trait"| FN
```

```rust
pub trait FnOnce<Args: Tuple> {
    type Output;
    fn call_once(self, args: Args) -> Self::Output;
}

pub trait FnMut<Args: Tuple>: FnOnce<Args> {
    fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait Fn<Args: Tuple>: FnMut<Args> {
    fn call(&self, args: Args) -> Self::Output;
}
```

#### 闭包捕获规则

```mermaid
graph TD
    START["闭包捕获变量"]

    START --> Q1{"只读取?"}
    Q1 -->|是| FN_IMPL["实现 Fn<br/>捕获 &T"]

    Q1 -->|否| Q2{"修改变量?"}
    Q2 -->|是| FNMUT_IMPL["实现 FnMut<br/>捕获 &mut T"]

    Q2 -->|否| Q3{"消耗变量?"}
    Q3 -->|是| FNONCE_IMPL["实现 FnOnce<br/>捕获 T"]
```

#### move 闭包

```rust
let s = String::from("hello");

// 普通闭包：借用 s
let f1 = || println!("{}", s);

// move 闭包：获取 s 的所有权
let f2 = move || println!("{}", s);
// s 不再可用
```

### 范围运算符

```mermaid
graph TB
    subgraph "范围类型"
        RANGE["Range<br/>a..b"]
        RANGE_FROM["RangeFrom<br/>a.."]
        RANGE_TO["RangeTo<br/>..b"]
        RANGE_FULL["RangeFull<br/>.."]
        RANGE_INCL["RangeInclusive<br/>a..=b"]
        RANGE_TO_INCL["RangeToInclusive<br/>..=b"]
    end
```

| 语法 | 类型 | 范围 |
|------|------|------|
| `a..b` | `Range` | [a, b) |
| `a..` | `RangeFrom` | [a, ∞) |
| `..b` | `RangeTo` | [0, b) |
| `..` | `RangeFull` | 全部 |
| `a..=b` | `RangeInclusive` | [a, b] |
| `..=b` | `RangeToInclusive` | [0, b] |

---

## core::fmt 模块

### 格式化 Trait 概览

```mermaid
graph TB
    subgraph "格式化 Trait"
        DISPLAY["Display<br/>━━━━━━<br/>{}<br/>用户友好"]

        DEBUG["Debug<br/>━━━━━━<br/>{:?} {:?#}<br/>调试输出"]

        OCTAL["Octal<br/>━━━━━━<br/>{:o}<br/>八进制"]

        BINARY["Binary<br/>━━━━━━<br/>{:b}<br/>二进制"]

        LOWER_HEX["LowerHex<br/>━━━━━━<br/>{:x}<br/>小写十六进制"]

        UPPER_HEX["UpperHex<br/>━━━━━━<br/>{:X}<br/>大写十六进制"]

        POINTER["Pointer<br/>━━━━━━<br/>{:p}<br/>指针"]

        LOWER_EXP["LowerExp<br/>━━━━━━<br/>{:e}<br/>小写科学计数"]

        UPPER_EXP["UpperExp<br/>━━━━━━<br/>{:E}<br/>大写科学计数"]
    end
```

### Display Trait

```rust
pub trait Display {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}
```

**特性**：
- 格式字符串：`{}`
- 用于用户可读输出
- 不能 derive，必须手动实现
- 实现 `Display` 自动获得 `ToString`

**实现示例**：

```rust
impl Display for Point {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

### Debug Trait

```rust
pub trait Debug {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}
```

**特性**：
- 格式字符串：`{:?}` 或 `{:#?}`（美化）
- 可以使用 `#[derive(Debug)]`
- 用于调试和开发

**手动实现示例**：

```rust
impl Debug for Point {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

### Formatter 辅助方法

```mermaid
graph TB
    subgraph "Formatter 构建器"
        STRUCT["debug_struct(name)<br/>━━━━━━<br/>结构体格式"]

        TUPLE["debug_tuple(name)<br/>━━━━━━<br/>元组格式"]

        LIST["debug_list()<br/>━━━━━━<br/>列表格式"]

        SET["debug_set()<br/>━━━━━━<br/>集合格式"]

        MAP["debug_map()<br/>━━━━━━<br/>映射格式"]
    end
```

**输出示例**：

```rust
// debug_struct
Point { x: 1, y: 2 }

// debug_tuple
Point(1, 2)

// debug_list
[1, 2, 3]

// debug_set
{1, 2, 3}

// debug_map
{"a": 1, "b": 2}
```

### 格式化参数

```mermaid
graph LR
    subgraph "格式说明符"
        WIDTH["宽度<br/>{:5} {:width$}"]
        PRECISION["精度<br/>{:.3} {:.prec$}"]
        ALIGN["对齐<br/>{:<} {:>} {:^}"]
        FILL["填充<br/>{:0>5} {:_^10}"]
        SIGN["符号<br/>{:+}"]
    end
```

**格式字符串语法**：

```
{[参数]:[填充][对齐][符号][#][0][宽度][.精度][类型]}
```

| 组件 | 说明 | 示例 |
|------|------|------|
| 参数 | 位置或名称 | `{0}`, `{name}` |
| 填充 | 填充字符 | `{:0>5}` → `00042` |
| 对齐 | `<` `>` `^` | `{:<5}` → `42   ` |
| 符号 | `+` | `{:+}` → `+42` |
| `#` | 替代形式 | `{:#x}` → `0x2a` |
| `0` | 零填充 | `{:05}` → `00042` |
| 宽度 | 最小宽度 | `{:5}` |
| 精度 | 小数位数 | `{:.2}` |
| 类型 | 格式类型 | `x`, `b`, `o`, `e` |

### Write Trait

```rust
pub trait Write {
    fn write_str(&mut self, s: &str) -> Result;

    fn write_char(&mut self, c: char) -> Result { ... }
    fn write_fmt(&mut self, args: Arguments<'_>) -> Result { ... }
}
```

**用途**：格式化输出的目标 trait。

---

## 运算符实现最佳实践

### 1. 保持一致性

```rust
impl Add for Complex {
    type Output = Complex;
    fn add(self, rhs: Complex) -> Complex { /* ... */ }
}

// 同时实现引用版本
impl Add<&Complex> for Complex { /* ... */ }
impl Add<Complex> for &Complex { /* ... */ }
impl Add<&Complex> for &Complex { /* ... */ }
```

### 2. 使用正确的返回类型

```rust
// 返回新值（推荐）
impl Add for Vector {
    type Output = Vector;  // 返回拥有所有权的值
    fn add(self, rhs: Vector) -> Vector { /* ... */ }
}
```

### 3. Deref 不要滥用

```mermaid
graph TD
    Q["何时实现 Deref?"]

    Q --> YES["✓ 智能指针类型<br/>Box, Rc, Arc"]
    Q --> YES2["✓ 新类型包装<br/>单字段包装"]

    Q --> NO["✗ 类型语义不同<br/>即使接口相似"]
    Q --> NO2["✗ 会导致混淆<br/>方法冲突"]
```

### 4. 闭包参数选择

```mermaid
graph TD
    START["选择闭包 trait"]

    START --> Q1{"需要多次调用?"}
    Q1 -->|否| FNONCE["FnOnce"]
    Q1 -->|是| Q2{"需要修改状态?"}

    Q2 -->|是| FNMUT["FnMut"]
    Q2 -->|否| FN["Fn"]
```

---

## 速查表

### 运算符 Trait

| 运算符 | Trait | 方法 |
|--------|-------|------|
| `+` | `Add` | `add` |
| `-` | `Sub` | `sub` |
| `*` | `Mul` | `mul` |
| `/` | `Div` | `div` |
| `%` | `Rem` | `rem` |
| `-` (一元) | `Neg` | `neg` |
| `!` | `Not` | `not` |
| `&` | `BitAnd` | `bitand` |
| `\|` | `BitOr` | `bitor` |
| `^` | `BitXor` | `bitxor` |
| `<<` | `Shl` | `shl` |
| `>>` | `Shr` | `shr` |
| `[]` | `Index` | `index` |
| `[]=` | `IndexMut` | `index_mut` |
| `*` | `Deref` | `deref` |
| `*=` | `DerefMut` | `deref_mut` |
| `()` | `Fn/FnMut/FnOnce` | `call*` |

### 格式化说明符

| 说明符 | Trait | 示例 |
|--------|-------|------|
| `{}` | `Display` | `42` |
| `{:?}` | `Debug` | `Point { x: 1 }` |
| `{:#?}` | `Debug` (美化) | 多行格式 |
| `{:o}` | `Octal` | `52` |
| `{:b}` | `Binary` | `101010` |
| `{:x}` | `LowerHex` | `2a` |
| `{:X}` | `UpperHex` | `2A` |
| `{:p}` | `Pointer` | `0x7fff...` |
| `{:e}` | `LowerExp` | `4.2e1` |
| `{:E}` | `UpperExp` | `4.2E1` |
