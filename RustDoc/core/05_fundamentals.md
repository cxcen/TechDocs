# 基础类型模块

> `core::option`, `core::result`, `core::num` 深度解析

## 概述

Option、Result 和数值类型是 Rust 最基础的类型，它们体现了 Rust 的核心理念：用类型系统消除常见错误。

```mermaid
graph TB
    subgraph "基础类型生态"
        OPTION["Option<T><br/>可选值"]
        RESULT["Result<T, E><br/>可失败操作"]
        NUM["num<br/>数值类型"]
    end

    OPTION --> NONE["None<br/>无值"]
    OPTION --> SOME["Some(T)<br/>有值"]

    RESULT --> OK["Ok(T)<br/>成功"]
    RESULT --> ERR["Err(E)<br/>失败"]

    NUM --> INT["整数类型"]
    NUM --> FLOAT["浮点类型"]
    NUM --> WRAPPER["包装类型"]
```

---

## core::option 模块

### Option 定义

```rust
pub enum Option<T> {
    None,
    Some(T),
}
```

**设计理念**：用类型系统替代空指针，强制处理"无值"情况。

### 方法分类

```mermaid
graph TD
    OPTION["Option<T> 方法"]

    OPTION --> INSPECT["检查方法<br/>is_some, is_none"]
    OPTION --> CONVERT["转换方法<br/>as_ref, as_mut"]
    OPTION --> MAP["映射方法<br/>map, map_or"]
    OPTION --> EXTRACT["取值方法<br/>unwrap, expect"]
    OPTION --> LOGIC["逻辑方法<br/>and, or, xor"]
    OPTION --> ITER["迭代方法<br/>iter, iter_mut"]
    OPTION --> VALUE["值操作<br/>take, replace"]
    OPTION --> COMBINE["组合方法<br/>zip, unzip"]
    OPTION --> SPECIAL["特殊方法<br/>ok_or, transpose"]
```

### 检查方法

| 方法 | 签名 | 功能 |
|------|------|------|
| `is_some()` | `&self -> bool` | 检查是否为 Some |
| `is_none()` | `&self -> bool` | 检查是否为 None |
| `is_some_and(f)` | `self, F -> bool` | Some 且满足条件 |
| `is_none_or(f)` | `self, F -> bool` | None 或满足条件 |

### 转换方法

```mermaid
graph LR
    subgraph "引用转换"
        OPT["Option<T>"]
        OPT -->|"as_ref()"| REF["Option<&T>"]
        OPT -->|"as_mut()"| MUTREF["Option<&mut T>"]
        OPT -->|"as_deref()"| DEREF["Option<&Target>"]
    end
```

### 映射方法

```mermaid
graph LR
    subgraph "映射操作"
        A["Option<T>"]

        A -->|"map(f)"| B["Option<U>"]
        A -->|"map_or(default, f)"| C["U"]
        A -->|"map_or_else(df, f)"| D["U"]
        A -->|"and_then(f)"| E["Option<U>"]
    end
```

| 方法 | 功能 | None 时 |
|------|------|---------|
| `map(f)` | 映射 Some 值 | 保持 None |
| `map_or(default, f)` | 映射或返回默认 | 返回 default |
| `map_or_else(df, f)` | 映射或计算默认 | 调用 df() |
| `and_then(f)` | 链式操作（flatmap） | 保持 None |

### 取值方法

```mermaid
graph TD
    subgraph "取值策略"
        OPT["Option<T>"]

        OPT -->|"unwrap()"| PANIC["T 或 panic"]
        OPT -->|"expect(msg)"| PANIC_MSG["T 或 panic + 消息"]
        OPT -->|"unwrap_or(default)"| DEFAULT["T 或 default"]
        OPT -->|"unwrap_or_else(f)"| COMPUTE["T 或 f()"]
        OPT -->|"unwrap_or_default()"| DEF["T 或 Default::default()"]
    end
```

**选择指南**：

```mermaid
graph TD
    START["需要取值"]

    START --> Q1{"确定是 Some?"}
    Q1 -->|"100%确定"| UNWRAP["unwrap()"]
    Q1 -->|"应该是但需验证"| EXPECT["expect(msg)"]
    Q1 -->|"可能是 None"| Q2{"有默认值?"}

    Q2 -->|"常量默认"| UNWRAP_OR["unwrap_or(val)"]
    Q2 -->|"计算默认"| UNWRAP_OR_ELSE["unwrap_or_else(f)"]
    Q2 -->|"使用 Default"| UNWRAP_OR_DEFAULT["unwrap_or_default()"]
```

### 逻辑方法

```mermaid
graph LR
    subgraph "逻辑运算"
        AND["and(other)<br/>━━━━━━<br/>Some + Some → other<br/>否则 → None"]

        OR["or(other)<br/>━━━━━━<br/>Some → self<br/>None → other"]

        XOR["xor(other)<br/>━━━━━━<br/>恰好一个Some → 它<br/>否则 → None"]

        FILTER["filter(p)<br/>━━━━━━<br/>Some且满足 → Some<br/>否则 → None"]
    end
```

### 值操作方法

| 方法 | 功能 |
|------|------|
| `take()` | 取出值，留下 None |
| `replace(val)` | 替换值，返回旧值 |
| `insert(val)` | 插入值，返回可变引用 |
| `get_or_insert(val)` | 获取或插入，返回引用 |
| `get_or_insert_with(f)` | 获取或计算插入 |

### 特殊转换

```mermaid
graph LR
    subgraph "Option ↔ Result 转换"
        OPT["Option<T>"]
        RES["Result<T, E>"]

        OPT -->|"ok_or(e)"| RES
        OPT -->|"ok_or_else(f)"| RES
        RES -->|"ok()"| OPT
    end

    subgraph "transpose"
        OPT_RES["Option<Result<T, E>>"]
        RES_OPT["Result<Option<T>, E>"]

        OPT_RES -->|"transpose()"| RES_OPT
        RES_OPT -->|"transpose()"| OPT_RES
    end
```

---

## core::result 模块

### Result 定义

```rust
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

**设计理念**：用类型系统强制错误处理，消除未检查的异常。

### 方法分类

```mermaid
graph TD
    RESULT["Result<T, E> 方法"]

    RESULT --> CHECK["状态检查<br/>is_ok, is_err"]
    RESULT --> CONVERT["转换方法<br/>ok, err, as_ref"]
    RESULT --> MAP_OP["映射方法<br/>map, map_err"]
    RESULT --> CHAIN["链式方法<br/>and_then, or_else"]
    RESULT --> EXTRACT["取值方法<br/>unwrap, expect"]
    RESULT --> ITER["迭代方法<br/>iter, iter_mut"]
    RESULT --> SPECIAL["特殊方法<br/>transpose, flatten"]
```

### 状态检查

| 方法 | 签名 | 功能 |
|------|------|------|
| `is_ok()` | `&self -> bool` | 检查是否为 Ok |
| `is_err()` | `&self -> bool` | 检查是否为 Err |
| `is_ok_and(f)` | `self, F -> bool` | Ok 且满足条件 |
| `is_err_and(f)` | `self, F -> bool` | Err 且满足条件 |

### 映射方法

```mermaid
graph LR
    subgraph "双向映射"
        RES["Result<T, E>"]

        RES -->|"map(f)"| RES_U["Result<U, E>"]
        RES -->|"map_err(f)"| RES_F["Result<T, F>"]
        RES -->|"map_or(default, f)"| U["U"]
    end
```

### 链式方法

```mermaid
graph LR
    subgraph "链式操作"
        AND["and_then(f)<br/>━━━━━━<br/>Ok → f(t)<br/>Err → Err(e)"]

        OR["or_else(f)<br/>━━━━━━<br/>Ok → Ok(t)<br/>Err → f(e)"]
    end
```

### ? 运算符

```mermaid
graph TD
    subgraph "? 运算符展开"
        Q["result?"]

        Q --> CHECK{"is_ok?"}
        CHECK -->|是| UNWRAP["返回 Ok 内的值"]
        CHECK -->|否| RETURN["提前返回 Err"]
    end
```

**等价展开**：

```rust
// 这个表达式...
let value = result?;

// 等价于...
let value = match result {
    Ok(v) => v,
    Err(e) => return Err(e.into()),
};
```

### 错误处理模式

```mermaid
graph TD
    subgraph "错误处理策略"
        PANIC["panic 策略<br/>━━━━━━<br/>unwrap()<br/>expect(msg)"]

        RECOVER["恢复策略<br/>━━━━━━<br/>unwrap_or()<br/>or_else()"]

        PROPAGATE["传播策略<br/>━━━━━━<br/>? 运算符<br/>map_err()"]

        INSPECT_E["检查策略<br/>━━━━━━<br/>is_ok/is_err<br/>match"]
    end
```

---

## Option 和 Result 转换关系

```mermaid
graph TB
    subgraph "类型转换网络"
        OPT["Option<T>"]
        RES["Result<T, E>"]
        OPT_REF["Option<&T>"]
        RES_REF["Result<&T, &E>"]
        OPT_RES["Option<Result<T,E>>"]
        RES_OPT["Result<Option<T>,E>"]
    end

    OPT -->|"ok_or(e)"| RES
    RES -->|"ok()"| OPT
    RES -->|"err()"| OPT

    OPT -->|"as_ref()"| OPT_REF
    RES -->|"as_ref()"| RES_REF

    OPT_RES <-->|"transpose()"| RES_OPT
```

---

## core::num 模块

### 数值类型概览

```mermaid
graph TB
    subgraph "整数类型"
        SIGNED["有符号<br/>i8, i16, i32, i64, i128, isize"]
        UNSIGNED["无符号<br/>u8, u16, u32, u64, u128, usize"]
    end

    subgraph "浮点类型"
        FLOAT["f32, f64<br/>（f16, f128 实验性）"]
    end

    subgraph "包装类型"
        NONZERO["NonZero<T><br/>非零值"]
        WRAPPING["Wrapping<T><br/>溢出包装"]
        SATURATING["Saturating<T><br/>溢出饱和"]
    end
```

### NonZero 类型

```rust
// NonZero 保证值永不为零
pub struct NonZero<T>(/* 私有 */);

// 类型别名
type NonZeroU8 = NonZero<u8>;
type NonZeroI32 = NonZero<i32>;
// ...
```

**空指针优化**：

```mermaid
graph LR
    subgraph "内存布局优化"
        NORMAL["Option<u32><br/>━━━━━━<br/>8 字节<br/>（4 字节值 + 4 字节判别）"]

        OPTIMIZED["Option<NonZeroU32><br/>━━━━━━<br/>4 字节<br/>（0 表示 None）"]
    end
```

### Wrapping 和 Saturating

```mermaid
graph TD
    subgraph "溢出处理策略"
        NORMAL["普通运算<br/>━━━━━━<br/>Debug: panic<br/>Release: 包装"]

        WRAPPING["Wrapping<T><br/>━━━━━━<br/>始终包装<br/>MAX+1 → MIN"]

        SATURATING["Saturating<T><br/>━━━━━━<br/>停在边界<br/>MAX+1 → MAX"]

        CHECKED["checked_*<br/>━━━━━━<br/>返回 Option<br/>溢出 → None"]

        OVERFLOWING["overflowing_*<br/>━━━━━━<br/>返回 (T, bool)<br/>bool 表示是否溢出"]
    end
```

**示例**：

```rust
let max = u8::MAX;  // 255

// 不同策略的行为
Wrapping(max) + Wrapping(1);     // Wrapping(0)
Saturating(max) + Saturating(1); // Saturating(255)
max.checked_add(1);              // None
max.overflowing_add(1);          // (0, true)
```

### 数值 Trait

```mermaid
graph TB
    subgraph "数值相关 Trait"
        ZERO["ZeroablePrimitive<br/>可零化原语"]
        STEP["Step<br/>步进（用于Range）"]
    end
```

### 解析错误类型

```mermaid
graph LR
    subgraph "错误类型"
        PARSE_INT["ParseIntError<br/>━━━━━━<br/>整数解析失败"]

        PARSE_FLOAT["ParseFloatError<br/>━━━━━━<br/>浮点解析失败"]

        TRY_FROM["TryFromIntError<br/>━━━━━━<br/>类型转换失败"]
    end

    PARSE_INT --> KIND["IntErrorKind"]
    KIND --> EMPTY["Empty"]
    KIND --> INVALID["InvalidDigit"]
    KIND --> POS_OVER["PosOverflow"]
    KIND --> NEG_OVER["NegOverflow"]
```

### 浮点数分类

```mermaid
graph TB
    subgraph "FpCategory 分类"
        FP["FpCategory"]

        FP --> NAN["Nan<br/>非数字"]
        FP --> INF["Infinite<br/>无穷大"]
        FP --> ZERO["Zero<br/>零"]
        FP --> SUB["Subnormal<br/>次正规数"]
        FP --> NORM["Normal<br/>正规数"]
    end
```

---

## 方法链示例

### Option 链

```mermaid
graph LR
    A["Option<String>"]
    A -->|"as_ref()"| B["Option<&String>"]
    B -->|"map(|s| s.len())"| C["Option<usize>"]
    C -->|"filter(|&n| n > 5)"| D["Option<usize>"]
    D -->|"unwrap_or(0)"| E["usize"]
```

### Result 链

```mermaid
graph LR
    A["Result<String, IoError>"]
    A -->|"map(|s| s.trim())"| B["Result<&str, IoError>"]
    B -->|"and_then(|s| s.parse())"| C["Result<i32, ParseError>"]
    C -->|"map_err(|e| MyError::from(e))"| D["Result<i32, MyError>"]
```

---

## 最佳实践

### 1. 避免不必要的 unwrap

```rust
// 不推荐 ✗
let value = option.unwrap();

// 推荐 ✓
let value = option.unwrap_or_default();
// 或
if let Some(value) = option {
    // ...
}
```

### 2. 善用 ? 运算符

```rust
// 不推荐 ✗
fn process() -> Result<i32, Error> {
    let a = step1()?;
    let b = match step2(a) {
        Ok(v) => v,
        Err(e) => return Err(e),
    };
    Ok(b)
}

// 推荐 ✓
fn process() -> Result<i32, Error> {
    let a = step1()?;
    let b = step2(a)?;
    Ok(b)
}
```

### 3. 使用 map 而非 match

```rust
// 不推荐 ✗
let result = match option {
    Some(x) => Some(x * 2),
    None => None,
};

// 推荐 ✓
let result = option.map(|x| x * 2);
```

### 4. 使用 filter_map 简化

```rust
// 不推荐 ✗
iter.map(|x| x.parse().ok())
    .filter(|x| x.is_some())
    .map(|x| x.unwrap())

// 推荐 ✓
iter.filter_map(|x| x.parse().ok())
```

---

## 速查表

### Option 方法

| 方法 | 返回类型 | 功能 |
|------|----------|------|
| `is_some()` | `bool` | 是否有值 |
| `is_none()` | `bool` | 是否无值 |
| `map(f)` | `Option<U>` | 映射值 |
| `and_then(f)` | `Option<U>` | 链式操作 |
| `unwrap()` | `T` | 取值或 panic |
| `unwrap_or(v)` | `T` | 取值或默认 |
| `ok_or(e)` | `Result<T,E>` | 转为 Result |
| `take()` | `Option<T>` | 取出值 |
| `filter(p)` | `Option<T>` | 条件过滤 |

### Result 方法

| 方法 | 返回类型 | 功能 |
|------|----------|------|
| `is_ok()` | `bool` | 是否成功 |
| `is_err()` | `bool` | 是否失败 |
| `map(f)` | `Result<U,E>` | 映射 Ok 值 |
| `map_err(f)` | `Result<T,F>` | 映射 Err 值 |
| `and_then(f)` | `Result<U,E>` | 链式操作 |
| `or_else(f)` | `Result<T,F>` | 错误恢复 |
| `unwrap()` | `T` | 取值或 panic |
| `ok()` | `Option<T>` | 转为 Option |
