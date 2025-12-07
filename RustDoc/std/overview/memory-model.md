# Rust 内存模型详解

## 1. 所有权系统

Rust 的内存安全保证建立在所有权系统之上，这是 Rust 最独特的特性。

```mermaid
graph TB
    subgraph "所有权三大规则"
        R1["规则1: 每个值有且仅有一个所有者"]
        R2["规则2: 同一时间只能有一个所有者"]
        R3["规则3: 所有者离开作用域时值被丢弃"]
    end

    subgraph "示例"
        CODE1["let s1 = String::from(&quot;hello&quot;);"]
        CODE2["let s2 = s1;  // s1 移动到 s2"]
        CODE3["// s1 不再有效"]
        CODE4["} // s2 离开作用域，内存释放"]
    end

    R1 --> CODE1
    R2 --> CODE2
    R3 --> CODE4

    style R1 fill:#c8e6c9
    style R2 fill:#bbdefb
    style R3 fill:#fff9c4
```

### 所有权转移 (Move)

```mermaid
sequenceDiagram
    participant Stack as 栈
    participant Heap as 堆

    Note over Stack,Heap: let s1 = String::from("hello")
    Stack->>Stack: s1 {ptr, len:5, cap:5}
    Stack->>Heap: 分配 "hello"

    Note over Stack,Heap: let s2 = s1 (移动)
    Stack->>Stack: s2 {ptr, len:5, cap:5}
    Stack--xStack: s1 无效化

    Note over Stack,Heap: } 作用域结束
    Stack->>Heap: s2.drop() 释放内存
```

### Copy vs Move

```mermaid
graph TB
    subgraph "Copy 类型 (栈上复制)"
        COPY_TYPES["整数、浮点、布尔、字符<br/>固定大小数组 [T; N]<br/>元组 (仅当所有元素 Copy)"]
        COPY_EXAMPLE["let x = 5;<br/>let y = x;<br/>// x 和 y 都有效"]
    end

    subgraph "Move 类型 (所有权转移)"
        MOVE_TYPES["String, Vec, Box<br/>HashMap, 文件句柄<br/>任何含有堆分配的类型"]
        MOVE_EXAMPLE["let s1 = String::from(&quot;hi&quot;);<br/>let s2 = s1;<br/>// s1 无效"]
    end

    COPY_TYPES --> COPY_EXAMPLE
    MOVE_TYPES --> MOVE_EXAMPLE

    style COPY_TYPES fill:#c8e6c9
    style MOVE_TYPES fill:#ffccbc
```

---

## 2. 借用系统

借用允许在不转移所有权的情况下访问数据。

```mermaid
graph TD
    subgraph "借用规则"
        RULE1["规则1: 任意时刻，要么有一个可变引用<br/>要么有任意数量的不可变引用"]
        RULE2["规则2: 引用必须总是有效的"]
    end

    subgraph "不可变借用 &T"
        IMM1["允许多个同时存在"]
        IMM2["只能读取，不能修改"]
        IMM3["实现 Copy trait"]
    end

    subgraph "可变借用 &mut T"
        MUT1["同一时间只能有一个"]
        MUT2["可以读取和修改"]
        MUT3["不能与 &T 共存"]
    end

    RULE1 --> IMM1
    RULE1 --> MUT1
    RULE2 --> IMM2
    RULE2 --> MUT2

    style RULE1 fill:#e1bee7
    style RULE2 fill:#e1bee7
    style IMM1 fill:#c8e6c9
    style MUT1 fill:#ffccbc
```

### 借用检查器工作流程

```mermaid
flowchart TD
    START[代码分析开始] --> SCAN[扫描所有引用]
    SCAN --> CHECK1{存在 &mut T?}

    CHECK1 -->|是| CHECK2{同时存在其他引用?}
    CHECK1 -->|否| CHECK3{多个 &T?}

    CHECK2 -->|是| ERROR1[编译错误:<br/>不能同时有可变和不可变引用]
    CHECK2 -->|否| CHECK4{引用存活期间<br/>原值被修改?}

    CHECK3 -->|是| OK1[允许: 多个共享引用]
    CHECK3 -->|否| OK2[允许: 单个引用]

    CHECK4 -->|是| ERROR2[编译错误:<br/>借用期间不能修改]
    CHECK4 -->|否| LIFETIME[生命周期检查]

    LIFETIME --> VALID{引用在值有效期内?}
    VALID -->|是| PASS[编译通过]
    VALID -->|否| ERROR3[编译错误:<br/>悬垂引用]

    style ERROR1 fill:#ffcdd2
    style ERROR2 fill:#ffcdd2
    style ERROR3 fill:#ffcdd2
    style PASS fill:#c8e6c9
```

---

## 3. 生命周期

生命周期确保引用在被使用期间始终有效。

```mermaid
graph TB
    subgraph "生命周期标注"
        SYNTAX["语法: &'a T, &'a mut T"]
        MEANING["含义: 引用在 'a 期间有效"]
    end

    subgraph "生命周期省略规则"
        RULE1["1  每个引用参数获得独立生命周期"]
        RULE2["2  若只有一个输入生命周期，<br/>它被赋给所有输出引用"]
        RULE3["3  若有 &self 或 &mut self，<br/>self 的生命周期赋给所有输出引用"]
    end

    subgraph "示例"
        EX1["fn first(s: &str) -> &str<br/>推断为: fn first<'a>(s: &'a str) -> &'a str"]
        EX2["fn longest<'a>(x: &'a str, y: &'a str) -> &'a str<br/>需要显式标注"]
    end

    SYNTAX --> RULE1
    MEANING --> RULE2
    RULE2 --> EX1
    RULE3 --> EX2

    style SYNTAX fill:#bbdefb
    style RULE1 fill:#fff9c4
    style RULE2 fill:#fff9c4
    style RULE3 fill:#fff9c4
```

### 生命周期层次

```mermaid
graph TD
    subgraph "生命周期关系"
        STATIC["'static<br/>整个程序运行期间"]
        LONG["'a<br/>较长生命周期"]
        SHORT["'b<br/>较短生命周期"]

        STATIC -->|包含| LONG
        LONG -->|包含| SHORT
    end

    subgraph "'static 来源"
        S1["字符串字面量: &'static str"]
        S2["静态变量: static X: i32 = 5"]
        S3["泄漏的内存: Box::leak()"]
    end

    STATIC --> S1
    STATIC --> S2
    STATIC --> S3

    style STATIC fill:#e1bee7
    style LONG fill:#c8e6c9
    style SHORT fill:#fff9c4
```

---

## 4. 内存布局

### 栈与堆

```mermaid
graph LR
    subgraph "栈 Stack"
        direction TB
        STACK_DESC["• 固定大小<br/>• 快速分配/释放<br/>• LIFO 顺序<br/>• 函数调用帧"]
        STACK_TYPES["存储:<br/>原生类型<br/>引用<br/>固定大小数组<br/>元组/结构体"]
    end

    subgraph "堆 Heap"
        direction TB
        HEAP_DESC["• 动态大小<br/>• 较慢分配<br/>• 任意顺序释放<br/>• 通过指针访问"]
        HEAP_TYPES["存储:<br/>String<br/>Vec<br/>Box<br/>HashMap"]
    end

    STACK_DESC --> STACK_TYPES
    HEAP_DESC --> HEAP_TYPES

    style STACK_DESC fill:#c8e6c9
    style HEAP_DESC fill:#bbdefb
```

### 类型内存布局

```mermaid
graph TB
    subgraph "String 内存布局"
        STRING_STACK["栈: String<br/>├─ ptr: *mut u8 (8字节)<br/>├─ len: usize (8字节)<br/>└─ capacity: usize (8字节)<br/>总计: 24字节"]
        STRING_HEAP["堆: [u8]<br/>h e l l o<br/>实际字符数据"]
        STRING_STACK -->|ptr 指向| STRING_HEAP
    end

    subgraph "Vec<T> 内存布局"
        VEC_STACK["栈: Vec<T><br/>├─ ptr: *mut T (8字节)<br/>├─ len: usize (8字节)<br/>└─ capacity: usize (8字节)<br/>总计: 24字节"]
        VEC_HEAP["堆: [T]<br/>元素数组"]
        VEC_STACK -->|ptr 指向| VEC_HEAP
    end

    subgraph "Box<T> 内存布局"
        BOX_STACK["栈: Box<T><br/>└─ ptr: *mut T (8字节)<br/>总计: 8字节"]
        BOX_HEAP["堆: T<br/>实际数据"]
        BOX_STACK -->|ptr 指向| BOX_HEAP
    end

    style STRING_STACK fill:#fff9c4
    style VEC_STACK fill:#c8e6c9
    style BOX_STACK fill:#bbdefb
```

### 切片布局

```mermaid
graph TB
    subgraph "胖指针 (Fat Pointer)"
        SLICE["&[T] / &str<br/>├─ ptr: *const T (8字节)<br/>└─ len: usize (8字节)<br/>总计: 16字节"]

        TRAIT_OBJ["&dyn Trait<br/>├─ data_ptr: *const () (8字节)<br/>└─ vtable_ptr: *const () (8字节)<br/>总计: 16字节"]
    end

    subgraph "示例"
        ARRAY["数组: [1, 2, 3, 4, 5]"]
        SLICE_REF["&arr[1..4]<br/>ptr -> 2<br/>len = 3"]
        ARRAY --> SLICE_REF
    end

    style SLICE fill:#e1bee7
    style TRAIT_OBJ fill:#ffccbc
```

---

## 5. Drop 与析构

```mermaid
sequenceDiagram
    participant Code as 代码
    participant Compiler as 编译器
    participant Drop as Drop trait
    participant Memory as 内存

    Code->>Compiler: 变量离开作用域
    Compiler->>Drop: 调用 drop(&mut self)
    Note over Drop: 执行自定义清理逻辑
    Drop->>Memory: 释放堆内存
    Drop->>Memory: 关闭文件句柄
    Drop->>Memory: 释放系统资源
    Memory-->>Compiler: 完成
```

### Drop 顺序

```mermaid
graph TB
    subgraph "Drop 顺序规则"
        R1["1  变量按声明的逆序 drop"]
        R2["2  结构体字段按声明顺序 drop"]
        R3["3  可以用 std::mem::drop() 提前 drop"]
    end

    subgraph "示例"
        CODE["let a = create_a();<br/>let b = create_b();<br/>let c = create_c();<br/>} // drop 顺序: c, b, a"]
    end

    R1 --> CODE

    style R1 fill:#fff9c4
    style R2 fill:#c8e6c9
    style R3 fill:#bbdefb
```

---

## 6. 内部可变性

当需要在不可变引用下修改数据时，使用内部可变性模式。

```mermaid
graph TB
    subgraph "内部可变性类型"
        CELL["Cell<T><br/>• 值语义<br/>• get/set 操作<br/>• 仅限 Copy 类型"]
        REFCELL["RefCell<T><br/>• 引用语义<br/>• borrow/borrow_mut<br/>• 运行时借用检查"]
        UNSAFECELL["UnsafeCell<T><br/>• 底层原语<br/>• 所有内部可变性的基础<br/>• 需要 unsafe"]
    end

    subgraph "线程安全版本"
        ATOMIC["Atomic*<br/>• AtomicBool<br/>• AtomicI32 等<br/>• 原子操作"]
        MUTEX["Mutex<T><br/>• 互斥锁<br/>• 阻塞等待"]
        RWLOCK["RwLock<T><br/>• 读写锁<br/>• 多读单写"]
    end

    CELL --> UNSAFECELL
    REFCELL --> UNSAFECELL
    ATOMIC --> UNSAFECELL
    MUTEX --> UNSAFECELL
    RWLOCK --> UNSAFECELL

    style CELL fill:#c8e6c9
    style REFCELL fill:#bbdefb
    style ATOMIC fill:#fff9c4
    style MUTEX fill:#e1bee7
```

### RefCell 运行时检查

```mermaid
stateDiagram-v2
    [*] --> Unborrowed: 初始状态

    Unborrowed --> SharedBorrow: borrow()
    Unborrowed --> MutableBorrow: borrow_mut()

    SharedBorrow --> SharedBorrow: 再次 borrow()
    SharedBorrow --> Unborrowed: Ref drop

    MutableBorrow --> Unborrowed: RefMut drop

    SharedBorrow --> Panic: borrow_mut()
    MutableBorrow --> Panic: borrow()
    MutableBorrow --> Panic: borrow_mut()

    note right of Panic: 运行时 panic!
```

---

## 7. 零成本抽象

Rust 的内存模型支持零成本抽象，编译时检查确保运行时无开销。

```mermaid
graph TB
    subgraph "编译时完成"
        BORROW["借用检查"]
        LIFETIME["生命周期检查"]
        MOVE["移动语义分析"]
        MONOMORPH["泛型单态化"]
    end

    subgraph "零运行时开销"
        NO_GC["无垃圾回收"]
        NO_REFCOUNT["无引用计数 (除非显式)"]
        NO_NULLCHECK["无空指针检查"]
        INLINE["迭代器内联"]
    end

    BORROW --> NO_GC
    LIFETIME --> NO_NULLCHECK
    MOVE --> NO_REFCOUNT
    MONOMORPH --> INLINE

    style BORROW fill:#c8e6c9
    style NO_GC fill:#bbdefb
```

---

## 8. 常见内存模式

### 所有权模式总结

```mermaid
mindmap
    root((内存模式))
        所有权转移
            函数参数
            返回值
            赋值
        借用
            共享借用 &T
            独占借用 &mut T
            重借用
        智能指针
            Box 堆分配
            Rc 共享所有权
            Arc 线程安全共享
        内部可变性
            Cell 值复制
            RefCell 运行时借用
            Mutex 线程安全
```

### 常见错误与解决

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| use after move | 所有权已转移 | 使用 clone() 或引用 |
| cannot borrow as mutable | 已存在不可变借用 | 调整借用作用域 |
| lifetime may not live long enough | 引用存活太短 | 添加生命周期标注 |
| data race | 多线程访问 | 使用 Arc + Mutex |

```mermaid
flowchart TD
    ERROR[编译错误] --> TYPE{错误类型?}

    TYPE -->|移动后使用| SOL1["解决方案:<br/>1. 使用 clone()<br/>2. 改用引用<br/>3. 实现 Copy"]

    TYPE -->|借用冲突| SOL2["解决方案:<br/>1. 缩小借用作用域<br/>2. 使用 RefCell<br/>3. 重构代码"]

    TYPE -->|生命周期| SOL3["解决方案:<br/>1. 添加生命周期标注<br/>2. 使用 'static<br/>3. 克隆数据"]

    TYPE -->|线程安全| SOL4["解决方案:<br/>1. Arc + Mutex<br/>2. 消息传递<br/>3. 无锁数据结构"]

    style ERROR fill:#ffcdd2
    style SOL1 fill:#c8e6c9
    style SOL2 fill:#c8e6c9
    style SOL3 fill:#c8e6c9
    style SOL4 fill:#c8e6c9
```
