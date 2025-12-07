# Rust 标准库架构详解

## 1. 分层架构

Rust 标准库采用严格的分层设计，从底层到高层依次为：

```mermaid
graph TB
    subgraph "第4层: std 标准库"
        STD[std]
        STD_IO[io/fs/net]
        STD_THREAD[thread/sync]
        STD_COLLECTIONS[collections]
        STD_OS[os 特定功能]
    end

    subgraph "第3层: alloc 分配库"
        ALLOC[alloc]
        VEC[Vec/String]
        BOX[Box]
        RC_ARC[Rc/Arc]
        BTREE[BTreeMap/BTreeSet]
    end

    subgraph "第2层: core 核心库"
        CORE[core]
        OPTION_RESULT[Option/Result]
        ITER[Iterator]
        TRAITS[marker/ops/cmp traits]
        PRIMITIVES_OPS[原生类型方法]
    end

    subgraph "第1层: 编译器内置"
        COMPILER[编译器]
        PRIMITIVES[原生类型定义]
        LANG_ITEMS[lang items]
        INTRINSICS[intrinsics]
    end

    STD --> STD_IO
    STD --> STD_THREAD
    STD --> STD_COLLECTIONS
    STD --> STD_OS

    STD_IO --> ALLOC
    STD_THREAD --> ALLOC
    STD_COLLECTIONS --> ALLOC

    ALLOC --> VEC
    ALLOC --> BOX
    ALLOC --> RC_ARC
    ALLOC --> BTREE

    VEC --> CORE
    BOX --> CORE
    RC_ARC --> CORE
    BTREE --> CORE

    CORE --> OPTION_RESULT
    CORE --> ITER
    CORE --> TRAITS
    CORE --> PRIMITIVES_OPS

    OPTION_RESULT --> COMPILER
    ITER --> COMPILER
    TRAITS --> COMPILER
    PRIMITIVES_OPS --> COMPILER

    COMPILER --> PRIMITIVES
    COMPILER --> LANG_ITEMS
    COMPILER --> INTRINSICS

    style STD fill:#fff3e0
    style ALLOC fill:#e3f2fd
    style CORE fill:#e8f5e9
    style COMPILER fill:#fce4ec
```

### 各层职责

| 层级 | 库 | 职责 | 依赖 |
|------|------|------|------|
| **第4层** | `std` | 操作系统接口、I/O、线程、网络 | alloc, core, 操作系统 |
| **第3层** | `alloc` | 堆内存分配、动态容器 | core, 全局分配器 |
| **第2层** | `core` | 无依赖的基础功能 | 编译器内置 |
| **第1层** | 编译器 | 原生类型、内置函数 | 无 |

---

## 2. core 库详解

`core` 是 Rust 的核心库，不依赖任何外部库，可在 `no_std` 环境使用。

```mermaid
graph LR
    subgraph "core 模块结构"
        subgraph "基础类型"
            OPTION[option]
            RESULT[result]
            CELL[cell]
        end

        subgraph "Trait 定义"
            MARKER[marker]
            OPS[ops]
            CMP[cmp]
            CONVERT[convert]
            FMT[fmt]
            HASH[hash]
            ITER_TRAIT[iter]
        end

        subgraph "原生类型扩展"
            NUM[num]
            SLICE[slice]
            STR[str]
            ARRAY[array]
            CHAR[char]
        end

        subgraph "底层操作"
            MEM[mem]
            PTR[ptr]
            INTRINSICS_MOD[intrinsics]
            HINT[hint]
        end
    end

    OPTION --> MEM
    RESULT --> MEM
    CELL --> MEM
    ITER_TRAIT --> OPTION
    CMP --> MARKER
    CONVERT --> MARKER

    style OPTION fill:#c8e6c9
    style RESULT fill:#c8e6c9
    style MARKER fill:#bbdefb
    style MEM fill:#ffccbc
```

### core 核心 Trait

```mermaid
classDiagram
    class Copy {
        <<marker>>
        按位复制
    }

    class Clone {
        +clone(&self) Self
        +clone_from(&mut self, source: &Self)
    }

    class Default {
        +default() Self
    }

    class PartialEq~Rhs~ {
        +eq(&self, other: &Rhs) bool
        +ne(&self, other: &Rhs) bool
    }

    class Eq {
        <<marker>>
        完全等价关系
    }

    class PartialOrd~Rhs~ {
        +partial_cmp(&self, other: &Rhs) Option~Ordering~
        +lt() gt() le() ge()
    }

    class Ord {
        +cmp(&self, other: &Self) Ordering
        +max() min() clamp()
    }

    class Hash {
        +hash~H: Hasher~(&self, state: &mut H)
    }

    class Debug {
        +fmt(&self, f: &mut Formatter) Result
    }

    class Display {
        +fmt(&self, f: &mut Formatter) Result
    }

    Copy --|> Clone : 要求
    Eq --|> PartialEq : 要求
    Ord --|> Eq : 要求
    Ord --|> PartialOrd : 要求
```

---

## 3. alloc 库详解

`alloc` 提供堆内存分配和动态数据结构。

```mermaid
graph TB
    subgraph "alloc 模块"
        subgraph "智能指针"
            BOX["Box&lt;T&gt;<br/>堆分配"]
            RC["Rc&lt;T&gt;<br/>引用计数"]
            ARC["Arc&lt;T&gt;<br/>原子引用计数"]
        end

        subgraph "动态容器"
            VEC["Vec&lt;T&gt;<br/>动态数组"]
            STRING["String<br/>UTF-8 字符串"]
            VECDEQUE["VecDeque&lt;T&gt;<br/>双端队列"]
            LINKEDLIST["LinkedList&lt;T&gt;<br/>链表"]
        end

        subgraph "B树容器"
            BTREEMAP["BTreeMap&lt;K,V&gt;"]
            BTREESET["BTreeSet&lt;T&gt;"]
        end

        subgraph "分配器"
            GLOBAL["GlobalAlloc trait"]
            ALLOCATOR["Allocator trait"]
            LAYOUT["Layout"]
        end
    end

    BOX --> GLOBAL
    VEC --> GLOBAL
    STRING --> VEC
    RC --> BOX
    ARC --> BOX

    style BOX fill:#bbdefb
    style VEC fill:#c8e6c9
    style ARC fill:#fff9c4
```

### 内存分配流程

```mermaid
sequenceDiagram
    participant User as 用户代码
    participant Vec as Vec::new()
    participant Alloc as 全局分配器
    participant OS as 操作系统

    User->>Vec: vec![1, 2, 3]
    Vec->>Vec: 计算 Layout
    Vec->>Alloc: alloc(layout)
    Alloc->>OS: mmap/HeapAlloc
    OS-->>Alloc: 内存地址
    Alloc-->>Vec: *mut u8
    Vec->>Vec: 写入数据
    Vec-->>User: Vec<i32?>;

    Note over User,OS: Drop 时反向释放
```

---

## 4. std 库详解

`std` 在 `core` 和 `alloc` 基础上提供操作系统接口。

```mermaid
graph TB
    subgraph "std 独有模块"
        subgraph "I/O 系统"
            IO[io]
            FS[fs]
            NET[net]
        end

        subgraph "并发"
            THREAD[thread]
            SYNC[sync]
        end

        subgraph "系统接口"
            ENV[env]
            PROCESS[process]
            OS_MOD[os]
        end

        subgraph "时间"
            TIME[time]
        end

        subgraph "额外集合"
            HASHMAP["HashMap&lt;K,V&gt;"]
            HASHSET["HashSet&lt;T&gt;"]
        end
    end

    subgraph "从 alloc 重导出"
        REEXPORT_VEC[vec]
        REEXPORT_STRING[string]
        REEXPORT_BOXED[boxed]
        REEXPORT_RC[rc]
    end

    subgraph "从 core 重导出"
        REEXPORT_OPTION[option]
        REEXPORT_RESULT[result]
        REEXPORT_ITER[iter]
    end

    IO --> FS
    IO --> NET
    THREAD --> SYNC
    HASHMAP --> SYNC

    style IO fill:#fff3e0
    style THREAD fill:#e1bee7
    style HASHMAP fill:#b2dfdb
```

### std 平台抽象层

```mermaid
graph TB
    subgraph "公共 API"
        API[std 公共接口]
    end

    subgraph "sys 内部模块"
        SYS[sys 抽象层]
    end

    subgraph "平台实现"
        UNIX[Unix 实现<br/>Linux/macOS/BSD]
        WINDOWS[Windows 实现]
        WASM[WebAssembly 实现]
    end

    API --> SYS
    SYS --> UNIX
    SYS --> WINDOWS
    SYS --> WASM

    UNIX --> LIBC[libc]
    WINDOWS --> WIN32[Win32 API]
    WASM --> WASI[WASI]

    style API fill:#e8f5e9
    style SYS fill:#fff3e0
    style UNIX fill:#e3f2fd
    style WINDOWS fill:#e3f2fd
    style WASM fill:#e3f2fd
```

---

## 5. Prelude 预导入

Rust 自动导入 `std::prelude` 到每个模块：

```mermaid
mindmap
  root((std::prelude))
    marker traits
      Copy
      Send
      Sized
      Sync
      Unpin
    操作 traits
      Drop
      Fn
      FnMut
      FnOnce
    转换 traits
      AsRef
      AsMut
      From
      Into
    迭代器
      Iterator
      IntoIterator
      ExactSizeIterator
      Extend
    Option/Result
      Option
      Some
      None
      Result
      Ok
      Err
    String 相关
      String
      ToString
      ToOwned
    智能指针
      Box
    宏
      vec!
```

---

## 6. 模块依赖关系

```mermaid
graph LR
    subgraph "错误处理链"
        RESULT[result] --> OPTION[option]
        IO_ERROR[io::Error] --> RESULT
        FMT_ERROR[fmt::Error] --> RESULT
    end

    subgraph "集合依赖"
        HASHMAP[HashMap] --> HASH[hash]
        HASHMAP --> VEC[vec]
        BTREEMAP[BTreeMap] --> ORD[Ord trait]
        VEC --> SLICE[slice]
        VEC --> ITER[iter]
    end

    subgraph "I/O 依赖"
        FILE[fs::File] --> IO[io traits]
        TCP[net::TcpStream] --> IO
        BUFREADER[BufReader] --> IO
    end

    subgraph "同步依赖"
        MUTEX[Mutex] --> ARC[Arc]
        RWLOCK[RwLock] --> ARC
        MPSC[mpsc] --> ARC
    end

    style RESULT fill:#c8e6c9
    style VEC fill:#bbdefb
    style IO fill:#fff9c4
    style MUTEX fill:#e1bee7
```

---

## 7. 编译条件与特性

### 常用 cfg 属性

```rust
// 平台检测
#[cfg(unix)]
#[cfg(windows)]
#[cfg(target_os = "linux")]
#[cfg(target_os = "macos")]
#[cfg(target_arch = "x86_64")]
#[cfg(target_arch = "aarch64")]

// 指针大小
#[cfg(target_pointer_width = "64")]
#[cfg(target_pointer_width = "32")]

// 特性开关
#[cfg(feature = "std")]
#[cfg(not(feature = "std"))]

// 测试/文档
#[cfg(test)]
#[cfg(doc)]
```

### no_std 环境

```mermaid
graph TB
    subgraph "完整 std 环境"
        STD_FULL[std]
        ALLOC_FULL[alloc]
        CORE_FULL[core]
    end

    subgraph "no_std + alloc"
        ALLOC_ONLY[alloc]
        CORE_NS[core]
    end

    subgraph "纯 no_std"
        CORE_ONLY[core]
    end

    STD_FULL --> ALLOC_FULL
    ALLOC_FULL --> CORE_FULL

    ALLOC_ONLY --> CORE_NS

    style STD_FULL fill:#c8e6c9
    style ALLOC_ONLY fill:#fff9c4
    style CORE_ONLY fill:#ffccbc
```

| 环境 | 可用库 | 典型用途 |
|------|--------|----------|
| 完整 std | std + alloc + core | 普通应用程序 |
| no_std + alloc | alloc + core | 嵌入式系统有堆 |
| 纯 no_std | 仅 core | 裸机/bootloader |
