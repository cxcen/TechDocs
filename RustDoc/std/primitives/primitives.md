# Rust 原生类型详解

## 1. 类型总览

```mermaid
graph TB
    subgraph "原生类型 (Primitive Types)"
        subgraph "标量类型"
            INT["整数<br/>i8 i16 i32 i64 i128 isize<br/>u8 u16 u32 u64 u128 usize"]
            FLOAT["浮点<br/>f32 f64"]
            BOOL["布尔<br/>bool"]
            CHAR["字符<br/>char"]
        end

        subgraph "复合类型"
            TUPLE["元组<br/>(T1, T2, ...)"]
            ARRAY["数组<br/>[T; N]"]
            SLICE["切片<br/>[T]"]
            STR["字符串切片<br/>str"]
        end

        subgraph "指针类型"
            REF["引用<br/>&T, &mut T"]
            RAW["裸指针<br/>*const T, *mut T"]
            FN_PTR["函数指针<br/>fn(T) -> U"]
        end

        subgraph "特殊类型"
            UNIT["单元<br/>()"]
            NEVER["Never<br/>!"]
        end
    end

    style INT fill:#c8e6c9
    style TUPLE fill:#bbdefb
    style REF fill:#fff9c4
    style UNIT fill:#e1bee7
```

---

## 2. 整数类型

```mermaid
graph TB
    subgraph "有符号整数"
        I8["i8<br/>-128 ~ 127"]
        I16["i16<br/>-32768 ~ 32767"]
        I32["i32 (默认)<br/>-2³¹ ~ 2³¹-1"]
        I64["i64<br/>-2⁶³ ~ 2⁶³-1"]
        I128["i128<br/>-2¹²⁷ ~ 2¹²⁷-1"]
        ISIZE["isize<br/>指针大小"]
    end

    subgraph "无符号整数"
        U8["u8<br/>0 ~ 255"]
        U16["u16<br/>0 ~ 65535"]
        U32["u32<br/>0 ~ 2³²-1"]
        U64["u64<br/>0 ~ 2⁶⁴-1"]
        U128["u128<br/>0 ~ 2¹²⁸-1"]
        USIZE["usize<br/>指针大小"]
    end

    style I32 fill:#c8e6c9
    style USIZE fill:#bbdefb
```

### 字面量语法

```mermaid
graph TB
    subgraph "整数字面量"
        DEC["十进制: 98_222"]
        HEX["十六进制: 0xff"]
        OCT["八进制: 0o77"]
        BIN["二进制: 0b1111_0000"]
        BYTE["字节 (u8): b'A'"]
    end

    subgraph "类型后缀"
        SUFFIX["42i32, 42u8, 42usize"]
        UNDERSCORE["1_000_000"]
    end

    DEC --> SUFFIX
    HEX --> SUFFIX

    style DEC fill:#c8e6c9
    style SUFFIX fill:#bbdefb
```

### 整数方法

```mermaid
graph TB
    subgraph "算术运算"
        CHECKED["checked_add/sub/mul/div<br/>溢出返回 None"]
        SATURATING["saturating_add/sub<br/>饱和运算"]
        WRAPPING["wrapping_add/sub<br/>环绕运算"]
        OVERFLOWING["overflowing_add/sub<br/>返回 (结果, 是否溢出)"]
    end

    subgraph "位操作"
        COUNT["count_ones/zeros<br/>位计数"]
        LEADING["leading_zeros<br/>前导零"]
        TRAILING["trailing_zeros<br/>尾部零"]
        ROTATE["rotate_left/right<br/>循环移位"]
        REVERSE["reverse_bits<br/>位反转"]
    end

    subgraph "转换"
        TO_BE["to_be/le_bytes<br/>字节序转换"]
        FROM_BE["from_be/le_bytes<br/>从字节构建"]
        ABS["abs/signum<br/>绝对值/符号"]
        POW["pow/ilog2<br/>幂运算"]
    end

    style CHECKED fill:#c8e6c9
    style COUNT fill:#bbdefb
    style TO_BE fill:#fff9c4
```

---

## 3. 浮点类型

```mermaid
graph TB
    subgraph "浮点类型"
        F32["f32 (单精度)<br/>• 32 位<br/>• ~7 位有效数字<br/>• IEEE 754"]
        F64["f64 (双精度, 默认)<br/>• 64 位<br/>• ~15 位有效数字<br/>• IEEE 754"]
    end

    subgraph "特殊值"
        INF["INFINITY / NEG_INFINITY"]
        NAN["NAN (非数字)"]
        MIN_MAX["MIN / MAX"]
        EPSILON["EPSILON"]
    end

    F32 --> INF
    F64 --> INF
    F32 --> NAN
    F64 --> NAN

    style F64 fill:#c8e6c9
    style NAN fill:#ffccbc
```

### 浮点方法

```mermaid
graph TB
    subgraph "数学运算"
        SQRT["sqrt() 平方根"]
        POWI["powi(n) / powf(x) 幂"]
        LOG["ln() / log2() / log10()"]
        EXP["exp() / exp2()"]
        TRIG["sin() / cos() / tan()"]
    end

    subgraph "取整"
        FLOOR["floor() 向下"]
        CEIL["ceil() 向上"]
        ROUND["round() 四舍五入"]
        TRUNC["trunc() 截断"]
        FRACT["fract() 小数部分"]
    end

    subgraph "比较"
        IS_NAN["is_nan()"]
        IS_INFINITE["is_infinite()"]
        IS_FINITE["is_finite()"]
        IS_NORMAL["is_normal()"]
        TOTAL_CMP["total_cmp() 全序比较"]
    end

    style SQRT fill:#c8e6c9
    style FLOOR fill:#bbdefb
    style IS_NAN fill:#fff9c4
```

---

## 4. bool 和 char

```mermaid
graph TB
    subgraph "bool"
        BOOL_SIZE["大小: 1 字节"]
        BOOL_VAL["值: true / false"]
        BOOL_OPS["运算: && || !"]
    end

    subgraph "char"
        CHAR_SIZE["大小: 4 字节"]
        CHAR_VAL["值: Unicode 标量值<br/>'a' '中' '😀'"]
        CHAR_RANGE["范围: U+0000 ~ U+D7FF<br/>U+E000 ~ U+10FFFF"]
    end

    style BOOL_SIZE fill:#c8e6c9
    style CHAR_SIZE fill:#bbdefb
```

### char 方法

```mermaid
graph TB
    subgraph "判断"
        IS_ALPHA["is_alphabetic()"]
        IS_NUM["is_numeric()"]
        IS_ALNUM["is_alphanumeric()"]
        IS_WS["is_whitespace()"]
        IS_ASCII["is_ascii()"]
    end

    subgraph "转换"
        TO_UPPER["to_uppercase()"]
        TO_LOWER["to_lowercase()"]
        TO_ASCII["to_ascii_uppercase()"]
        TO_DIGIT["to_digit(radix)"]
    end

    subgraph "编码"
        LEN_UTF8["len_utf8() -> 1~4"]
        ENCODE_UTF8["encode_utf8(&mut [u8])"]
        FROM_U32["char::from_u32(n)"]
    end

    style IS_ALPHA fill:#c8e6c9
    style TO_UPPER fill:#bbdefb
    style LEN_UTF8 fill:#fff9c4
```

---

## 5. 元组 (T1, T2, ...)

```mermaid
graph TB
    subgraph "元组特性"
        FIXED["固定长度"]
        HETEROGENEOUS["异构类型"]
        STACK["栈分配"]
        INDEX["索引访问 .0 .1 .2"]
    end

    subgraph "特殊元组"
        UNIT["单元类型 ()<br/>• 大小 0<br/>• 函数默认返回"]
        PAIR["二元组 (A, B)<br/>• 常用于返回多值"]
        LARGE["最多 12 元素<br/>实现标准 trait"]
    end

    subgraph "解构"
        PATTERN["let (a, b, c) = tuple;"]
        IGNORE["let (a, _, c) = tuple;"]
        NESTED["let ((a, b), c) = nested;"]
    end

    FIXED --> UNIT
    HETEROGENEOUS --> PAIR
    INDEX --> PATTERN

    style UNIT fill:#c8e6c9
    style PATTERN fill:#bbdefb
```

---

## 6. 数组 [T; N]

```mermaid
graph TB
    subgraph "数组特性"
        FIXED["固定大小 N"]
        HOMOGENEOUS["同类型元素"]
        STACK["栈分配"]
        CONTIGUOUS["连续内存"]
    end

    subgraph "创建"
        LITERAL["[1, 2, 3, 4, 5]"]
        REPEAT["[0; 100]  // 100 个 0"]
        FROM_FN["std::array::from_fn"]
    end

    subgraph "方法"
        LEN["len() 长度"]
        GET["get(i) / get_mut(i)"]
        ITER["iter() / iter_mut()"]
        MAP["map(f) -> [U; N]"]
        EACH_REF["each_ref() -> [&T; N]"]
    end

    FIXED --> LITERAL
    STACK --> REPEAT

    style FIXED fill:#c8e6c9
    style LITERAL fill:#bbdefb
```

---

## 7. 切片 [T]

切片是动态大小类型 (DST)，只能通过引用使用。

```mermaid
graph TB
    subgraph "切片引用"
        SHARED["&[T] 共享切片<br/>• ptr + len<br/>• 16 字节"]
        MUT["&mut [T] 可变切片<br/>• 独占访问"]
    end

    subgraph "创建"
        FROM_ARRAY["&arr[..]"]
        FROM_VEC["&vec[..]"]
        RANGE["&arr[1..4]<br/>&arr[..3]<br/>&arr[2..]"]
    end

    subgraph "方法"
        LEN["len() / is_empty()"]
        GET["get(i) / first() / last()"]
        SPLIT["split_at() / split()"]
        CHUNKS["chunks() / windows()"]
        SORT["sort() / sort_by()"]
        BINARY["binary_search()"]
        CONTAINS["contains()"]
    end

    SHARED --> FROM_ARRAY
    MUT --> FROM_VEC

    style SHARED fill:#c8e6c9
    style FROM_ARRAY fill:#bbdefb
    style SORT fill:#fff9c4
```

### 切片内存布局

```mermaid
graph TB
    subgraph "数组 [i32; 5]"
        ARR["[10, 20, 30, 40, 50]"]
    end

    subgraph "切片 &arr[1..4]"
        SLICE_PTR["ptr → 20"]
        SLICE_LEN["len = 3"]
    end

    ARR --> SLICE_PTR

    style ARR fill:#c8e6c9
    style SLICE_PTR fill:#bbdefb
```

---

## 8. 字符串切片 str

```mermaid
graph TB
    subgraph "str 特性"
        UTF8["UTF-8 编码"]
        DST["动态大小类型"]
        ALWAYS_REF["总是通过 &str 使用"]
    end

    subgraph "&str 来源"
        LITERAL["字符串字面量<br/>&'static str"]
        FROM_STRING["&String"]
        SLICE_STR["&s[start..end]<br/>必须在 char 边界"]
    end

    subgraph "常用方法"
        LEN["len() 字节长度"]
        CHARS["chars() 字符迭代"]
        BYTES["bytes() 字节迭代"]
        CONTAINS["contains()"]
        STARTS["starts_with() / ends_with()"]
        FIND["find() / rfind()"]
        SPLIT["split() / lines()"]
        TRIM["trim() / trim_start/end()"]
        REPLACE["replace()"]
        TO_OWNED["to_string() / to_owned()"]
    end

    UTF8 --> LITERAL
    DST --> FROM_STRING

    style UTF8 fill:#c8e6c9
    style LITERAL fill:#bbdefb
    style CHARS fill:#fff9c4
```

### 字符串索引

```mermaid
graph TB
    subgraph "为什么不能索引"
        UTF8["UTF-8 变长编码"]
        REASON["s[i] 复杂度应为 O(1)<br/>但 UTF-8 需要 O(n)"]
    end

    subgraph "替代方案"
        CHARS["s.chars().nth(i)"]
        BYTES["s.bytes().nth(i) 或 s.as_bytes()[i]"]
        SLICE["&s[start..end] (字节范围)"]
        CHAR_INDICES["s.char_indices()"]
    end

    UTF8 --> REASON
    REASON --> CHARS

    style REASON fill:#ffccbc
    style CHARS fill:#c8e6c9
```

---

## 9. 引用类型

```mermaid
graph TB
    subgraph "共享引用 &T"
        SHARED_PROP["• 不可变<br/>• 可多个同时存在<br/>• 实现 Copy"]
    end

    subgraph "独占引用 &mut T"
        MUT_PROP["• 可变<br/>• 同时只能有一个<br/>• 不实现 Copy"]
    end

    subgraph "生命周期"
        LIFETIME["&'a T<br/>引用在 'a 期间有效"]
        ELISION["生命周期省略规则"]
    end

    SHARED_PROP --> LIFETIME
    MUT_PROP --> LIFETIME

    style SHARED_PROP fill:#c8e6c9
    style MUT_PROP fill:#bbdefb
```

---

## 10. 裸指针

```mermaid
graph TB
    subgraph "裸指针类型"
        CONST["*const T<br/>不可变裸指针"]
        MUT["*mut T<br/>可变裸指针"]
    end

    subgraph "特性"
        NO_SAFETY["不保证安全"]
        NULLABLE["可以为空"]
        NO_LIFETIME["无生命周期跟踪"]
        FFI["用于 FFI"]
    end

    subgraph "操作 (unsafe)"
        DEREF["*ptr 解引用"]
        OFFSET["ptr.offset(n)"]
        READ["ptr.read() / .read_unaligned()"]
        WRITE["ptr.write() / .write_unaligned()"]
    end

    CONST --> NO_SAFETY
    MUT --> NO_SAFETY
    NO_SAFETY --> DEREF

    style CONST fill:#fff9c4
    style NO_SAFETY fill:#ffccbc
```

---

## 11. 函数指针

```mermaid
graph TB
    subgraph "函数指针类型"
        FN["fn(T) -> U"]
        FN_UNSAFE["unsafe fn(T) -> U"]
        FN_EXTERN["extern &quot;C&quot; fn(T) -> U"]
    end

    subgraph "与闭包区别"
        FN_PTR["函数指针<br/>• 不捕获环境<br/>• 固定大小 (8 字节)<br/>• 实现 Copy"]
        CLOSURE["闭包<br/>• 可捕获环境<br/>• 大小不定<br/>• 实现 Fn/FnMut/FnOnce"]
    end

    FN --> FN_PTR
    FN --> CLOSURE

    style FN fill:#c8e6c9
    style FN_PTR fill:#bbdefb
```

---

## 12. Never 类型 !

```mermaid
graph TB
    subgraph "Never 类型"
        NEVER["!"]
        MEANING["表示永不返回"]
        BOTTOM["是所有类型的子类型"]
    end

    subgraph "产生 ! 的表达式"
        PANIC["panic!()"]
        LOOP["loop { } (无 break)"]
        RETURN["return"]
        BREAK["break / continue"]
        DIVERGE["发散函数"]
    end

    subgraph "用途"
        MATCH["match 分支类型统一"]
        RESULT["Result<T, !>"]
    end

    NEVER --> PANIC
    NEVER --> LOOP
    BOTTOM --> MATCH

    style NEVER fill:#e1bee7
    style PANIC fill:#ffccbc
```

---

## 13. 类型大小总结

| 类型 | 大小 (64位) | 对齐 |
|------|------------|------|
| `bool` | 1 | 1 |
| `i8` / `u8` | 1 | 1 |
| `i16` / `u16` | 2 | 2 |
| `i32` / `u32` / `f32` | 4 | 4 |
| `i64` / `u64` / `f64` | 8 | 8 |
| `i128` / `u128` | 16 | 16 |
| `char` | 4 | 4 |
| `usize` / `isize` | 8 | 8 |
| `&T` / `*const T` | 8 | 8 |
| `&[T]` / `&str` | 16 | 8 |
| `()` | 0 | 1 |
| `[T; N]` | N × size(T) | align(T) |
| `(T1, T2)` | 带填充对齐 | max(align) |

```mermaid
graph TB
    subgraph "类型大小可视化"
        UNIT["() - 0 字节"]
        BOOL["bool - 1 字节"]
        I32["i32 - 4 字节"]
        USIZE["usize - 8 字节"]
        SLICE_REF["&[T] - 16 字节"]
    end

    UNIT --> BOOL
    BOOL --> I32
    I32 --> USIZE
    USIZE --> SLICE_REF

    style UNIT fill:#e1bee7
    style I32 fill:#c8e6c9
    style SLICE_REF fill:#bbdefb
```
