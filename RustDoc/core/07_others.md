# 其他核心模块

> `core::slice`, `core::str`, `core::hash`, `core::io` 深度解析

## 概述

这些模块提供了 Rust 中最常用的数据处理能力：切片操作、字符串处理、哈希计算和基础 I/O。

```mermaid
graph TB
    subgraph "数据处理模块"
        SLICE["slice<br/>连续内存视图"]
        STR["str<br/>UTF-8 字符串"]
        HASH["hash<br/>哈希计算"]
        IO["io<br/>基础 I/O"]
    end

    SLICE --> ITER_S["切片迭代"]
    SLICE --> SEARCH["搜索排序"]
    SLICE --> SPLIT["分割合并"]

    STR --> CHARS["字符操作"]
    STR --> PATTERN["模式匹配"]
    STR --> ENCODE["编码处理"]

    HASH --> HASHER["Hasher trait"]
    HASH --> HASH_T["Hash trait"]
    HASH --> BUILD["BuildHasher"]

    IO --> ERROR["错误类型"]
    IO --> BORROW_BUF["BorrowedBuf"]
```

---

## core::slice 模块

### 切片概述

切片 `[T]` 是一种动态大小类型（DST），表示连续内存中的元素序列。

```mermaid
graph LR
    subgraph "切片内存布局"
        REF["&[T]<br/>━━━━━━<br/>指针 + 长度<br/>（胖指针）"]
        REF --> MEM["连续内存<br/>[T, T, T, ...]"]
    end
```

### 切片方法分类

```mermaid
graph TD
    SLICE["切片方法"]

    SLICE --> ACCESS["访问方法<br/>get, first, last"]
    SLICE --> ITER["迭代方法<br/>iter, chunks, windows"]
    SLICE --> SEARCH["搜索方法<br/>contains, binary_search"]
    SLICE --> MODIFY["修改方法<br/>sort, reverse, swap"]
    SLICE --> SPLIT_M["分割方法<br/>split, splitn"]
    SLICE --> COPY["复制方法<br/>copy_from_slice, clone_from_slice"]
```

### 访问方法

```mermaid
graph LR
    subgraph "安全访问"
        GET["get(index)<br/>-> Option<&T>"]
        GET_MUT["get_mut(index)<br/>-> Option<&mut T>"]
        FIRST["first()<br/>-> Option<&T>"]
        LAST["last()<br/>-> Option<&T>"]
    end

    subgraph "不检查访问"
        GET_UNCHECKED["get_unchecked(index)<br/>-> &T (unsafe)"]
    end
```

| 方法 | 返回类型 | 越界行为 |
|------|----------|----------|
| `slice[i]` | `T` / `&T` | panic |
| `get(i)` | `Option<&T>` | None |
| `get_unchecked(i)` | `&T` | UB (unsafe) |
| `first()` | `Option<&T>` | None |
| `last()` | `Option<&T>` | None |

### 迭代方法

```mermaid
graph TB
    subgraph "迭代器类型"
        ITER["iter()<br/>━━━━━━<br/>元素引用<br/>Item = &T"]

        ITER_MUT["iter_mut()<br/>━━━━━━<br/>可变引用<br/>Item = &mut T"]

        CHUNKS["chunks(n)<br/>━━━━━━<br/>固定大小块<br/>Item = &[T]"]

        WINDOWS["windows(n)<br/>━━━━━━<br/>滑动窗口<br/>Item = &[T]"]

        SPLIT_I["split(pred)<br/>━━━━━━<br/>按谓词分割<br/>Item = &[T]"]
    end
```

#### chunks vs windows

```mermaid
graph LR
    subgraph "chunks(2)"
        A1["[1,2,3,4,5]"]
        A1 --> B1["[1,2]"]
        A1 --> B2["[3,4]"]
        A1 --> B3["[5]"]
    end

    subgraph "windows(2)"
        A2["[1,2,3,4,5]"]
        A2 --> C1["[1,2]"]
        A2 --> C2["[2,3]"]
        A2 --> C3["[3,4]"]
        A2 --> C4["[4,5]"]
    end
```

### 搜索和排序

```mermaid
graph TB
    subgraph "搜索方法"
        CONTAINS["contains(&x)<br/>-> bool"]
        BINARY["binary_search(&x)<br/>-> Result<usize, usize>"]
        POSITION["iter().position(p)<br/>-> Option<usize>"]
    end

    subgraph "排序方法"
        SORT["sort()<br/>稳定排序"]
        SORT_UNSTABLE["sort_unstable()<br/>不稳定排序（更快）"]
        SORT_BY["sort_by(cmp)<br/>自定义比较"]
        SORT_BY_KEY["sort_by_key(f)<br/>按键排序"]
    end
```

**binary_search 返回值**：

```mermaid
graph LR
    subgraph "binary_search 结果"
        FOUND["找到<br/>Ok(index)"]
        NOT_FOUND["未找到<br/>Err(insert_pos)"]
    end
```

### 修改方法

| 方法 | 功能 |
|------|------|
| `swap(i, j)` | 交换两个元素 |
| `reverse()` | 原地反转 |
| `rotate_left(n)` | 左旋转 |
| `rotate_right(n)` | 右旋转 |
| `fill(value)` | 填充值 |
| `fill_with(f)` | 用闭包填充 |

### 分割方法

```mermaid
graph LR
    subgraph "split 系列"
        SPLIT["split(pred)<br/>按谓词分割"]
        SPLITN["splitn(n, pred)<br/>最多 n 份"]
        SPLIT_FIRST["split_first()<br/>-> Option<(&T, &[T])>"]
        SPLIT_LAST["split_last()<br/>-> Option<(&T, &[T])>"]
        SPLIT_AT["split_at(mid)<br/>-> (&[T], &[T])"]
    end
```

---

## core::str 模块

### str 类型

`str` 是 UTF-8 编码的字符串切片，是 DST 类型。

```mermaid
graph LR
    subgraph "str 内存布局"
        REF["&str<br/>━━━━━━<br/>指针 + 字节长度"]
        REF --> BYTES["UTF-8 字节序列"]
    end
```

### 方法分类

```mermaid
graph TD
    STR["str 方法"]

    STR --> CHAR["字符操作<br/>chars, char_indices"]
    STR --> BYTE["字节操作<br/>bytes, as_bytes"]
    STR --> SEARCH_S["搜索方法<br/>contains, find, starts_with"]
    STR --> TRIM["修剪方法<br/>trim, trim_start, trim_end"]
    STR --> SPLIT_S["分割方法<br/>split, lines, split_whitespace"]
    STR --> CASE["大小写<br/>to_lowercase, to_uppercase"]
    STR --> PARSE["解析<br/>parse"]
```

### 字符与字节

```mermaid
graph TB
    subgraph "字符 vs 字节"
        CHARS["chars()<br/>━━━━━━<br/>Unicode 标量值<br/>Item = char"]

        BYTES["bytes()<br/>━━━━━━<br/>原始字节<br/>Item = u8"]

        CHAR_INDICES["char_indices()<br/>━━━━━━<br/>字符及位置<br/>Item = (usize, char)"]
    end
```

**重要区别**：

```rust
let s = "你好";
s.len();        // 6 (字节数)
s.chars().count(); // 2 (字符数)
```

### 搜索方法

```mermaid
graph LR
    subgraph "模式搜索"
        CONTAINS["contains(pat)<br/>-> bool"]
        STARTS["starts_with(pat)<br/>-> bool"]
        ENDS["ends_with(pat)<br/>-> bool"]
        FIND["find(pat)<br/>-> Option<usize>"]
        RFIND["rfind(pat)<br/>-> Option<usize>"]
        MATCHES["matches(pat)<br/>-> Matches"]
    end
```

**Pattern trait 支持的类型**：
- `&str` - 字符串模式
- `char` - 单个字符
- `&[char]` - 字符集合
- `F: FnMut(char) -> bool` - 谓词函数

### 分割方法

```mermaid
graph TB
    subgraph "分割方法"
        SPLIT_P["split(pat)<br/>按模式分割"]
        SPLITN_P["splitn(n, pat)<br/>最多 n 份"]
        LINES["lines()<br/>按行分割"]
        SPLIT_WS["split_whitespace()<br/>按空白分割"]
        SPLIT_ASCII["split_ascii_whitespace()<br/>按 ASCII 空白"]
    end
```

### 修剪方法

| 方法 | 功能 |
|------|------|
| `trim()` | 两端去空白 |
| `trim_start()` | 开头去空白 |
| `trim_end()` | 结尾去空白 |
| `trim_matches(pat)` | 两端去匹配 |
| `strip_prefix(pat)` | 去前缀 |
| `strip_suffix(pat)` | 去后缀 |

### 大小写转换

```mermaid
graph LR
    subgraph "大小写方法"
        TO_LOWER["to_lowercase()<br/>-> String"]
        TO_UPPER["to_uppercase()<br/>-> String"]
        TO_ASCII_LOWER["to_ascii_lowercase()<br/>-> String"]
        MAKE_ASCII_LOWER["make_ascii_lowercase()<br/>(in-place, &mut str)"]
    end
```

**注意**：`to_lowercase()` 和 `to_uppercase()` 返回 `String`，因为某些 Unicode 转换会改变长度。

---

## core::hash 模块

### Hash 和 Hasher

```mermaid
graph TB
    subgraph "哈希系统"
        HASH["Hash trait<br/>━━━━━━<br/>可哈希类型"]

        HASHER["Hasher trait<br/>━━━━━━<br/>哈希算法"]

        BUILD["BuildHasher trait<br/>━━━━━━<br/>Hasher 工厂"]
    end

    HASH -->|"hash(&self, state)"| HASHER
    BUILD -->|"build_hasher()"| HASHER
```

### Hash Trait

```rust
pub trait Hash {
    fn hash<H: Hasher>(&self, state: &mut H);

    fn hash_slice<H: Hasher>(data: &[Self], state: &mut H)
    where Self: Sized { ... }
}
```

**derive 规则**：
- 可以使用 `#[derive(Hash)]`
- 结构体所有字段必须实现 `Hash`
- 枚举判别值 + 变体数据都被哈希

### Hasher Trait

```rust
pub trait Hasher {
    fn finish(&self) -> u64;
    fn write(&mut self, bytes: &[u8]);

    // 便捷方法
    fn write_u8(&mut self, i: u8) { ... }
    fn write_u16(&mut self, i: u16) { ... }
    fn write_u32(&mut self, i: u32) { ... }
    fn write_u64(&mut self, i: u64) { ... }
    fn write_usize(&mut self, i: usize) { ... }
    // ...更多整数类型
}
```

### BuildHasher Trait

```rust
pub trait BuildHasher {
    type Hasher: Hasher;
    fn build_hasher(&self) -> Self::Hasher;
}
```

**用途**：`HashMap` 和 `HashSet` 使用 `BuildHasher` 创建 Hasher 实例。

### 哈希一致性规则

```mermaid
graph TD
    subgraph "一致性要求"
        RULE1["k1 == k2 ⟹<br/>hash(k1) == hash(k2)"]

        RULE2["hash 必须确定性<br/>同一程序运行内"]

        RULE3["Eq 和 Hash 必须一致<br/>通常一起 derive"]
    end
```

---

## core::io 模块

### 概述

`core::io` 提供了 `no_std` 环境下的基础 I/O 类型，主要是错误类型。

### Error 类型

```mermaid
graph TB
    subgraph "I/O 错误"
        ERROR["Error<br/>━━━━━━<br/>I/O 错误类型"]

        ERROR --> KIND["ErrorKind<br/>错误类别"]

        KIND --> NOT_FOUND["NotFound"]
        KIND --> PERM["PermissionDenied"]
        KIND --> CONN["ConnectionRefused"]
        KIND --> WOULD_BLOCK["WouldBlock"]
        KIND --> OTHER["Other"]
    end
```

### BorrowedBuf（实验性）

```rust
pub struct BorrowedBuf<'data> {
    // 零拷贝缓冲区
}
```

**用途**：允许在不分配的情况下进行缓冲 I/O 操作。

---

## 模块交互

```mermaid
graph TB
    subgraph "切片和字符串关系"
        SLICE["[T]"]
        STR["str"]
        BYTES["[u8]"]

        STR -->|"as_bytes()"| BYTES
        BYTES -->|"str::from_utf8()"| STR_RES["Result<&str, Utf8Error>"]

        SLICE -->|"泛型方法"| STR
    end
```

```mermaid
graph LR
    subgraph "哈希和集合"
        HASH_T["Hash trait"]
        EQ_T["Eq trait"]

        HASH_T --> HASHMAP["HashMap<K, V>"]
        EQ_T --> HASHMAP
        HASH_T --> HASHSET["HashSet<T>"]
        EQ_T --> HASHSET
    end
```

---

## 最佳实践

### 切片操作

```rust
// 优先使用 get() 而非索引
if let Some(first) = slice.get(0) {
    // 安全处理
}

// 使用 chunks_exact 当大小固定
for chunk in data.chunks_exact(4) {
    // chunk 总是 4 个元素
}

// 使用 split_at 而非多次切片
let (left, right) = slice.split_at(mid);
```

### 字符串操作

```rust
// 字符迭代而非字节
for c in s.chars() {
    // 处理 Unicode 字符
}

// 使用 split_whitespace 而非 split(' ')
for word in s.split_whitespace() {
    // 处理所有空白类型
}

// 检查前缀/后缀
if s.starts_with("http") {
    // ...
}
```

### 哈希实现

```rust
// 一起 derive Hash 和 Eq
#[derive(Hash, Eq, PartialEq)]
struct Key {
    id: u64,
    name: String,
}

// 或手动实现保持一致性
impl Hash for Key {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
        self.name.hash(state);
    }
}
```

---

## 速查表

### slice 常用方法

| 方法 | 功能 |
|------|------|
| `len()` | 元素数量 |
| `is_empty()` | 是否为空 |
| `get(i)` | 安全访问 |
| `first()/last()` | 首/尾元素 |
| `iter()` | 迭代器 |
| `chunks(n)` | 分块迭代 |
| `windows(n)` | 窗口迭代 |
| `split(pred)` | 分割 |
| `contains(&x)` | 包含检查 |
| `binary_search(&x)` | 二分查找 |
| `sort()` | 排序 |
| `reverse()` | 反转 |

### str 常用方法

| 方法 | 功能 |
|------|------|
| `len()` | 字节长度 |
| `is_empty()` | 是否为空 |
| `chars()` | 字符迭代 |
| `bytes()` | 字节迭代 |
| `contains(pat)` | 包含检查 |
| `starts_with(pat)` | 前缀检查 |
| `find(pat)` | 查找位置 |
| `split(pat)` | 分割 |
| `lines()` | 按行分割 |
| `trim()` | 去空白 |
| `to_lowercase()` | 转小写 |
| `parse()` | 解析 |
