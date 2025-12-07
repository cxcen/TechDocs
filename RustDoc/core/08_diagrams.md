# Rust Core 库架构图集

> 本文档包含 Rust Core 库的各种架构图和关系图

## 1. 整体架构

```mermaid
graph TB
    subgraph "Rust Core 库分层架构"
        direction TB

        L1["━━━━━━ 应用层 ━━━━━━"]
        L1_C["用户代码、应用程序"]

        L2["━━━━━━ 高层抽象 ━━━━━━"]
        L2_C["Iterator, Option, Result, String, Vec"]

        L3["━━━━━━ 核心 Trait ━━━━━━"]
        L3_C["Clone, Copy, Send, Sync, Debug, Display"]

        L4["━━━━━━ 运算符系统 ━━━━━━"]
        L4_C["Add, Deref, Fn, Index, Drop"]

        L5["━━━━━━ 并发原语 ━━━━━━"]
        L5_C["Cell, RefCell, Atomic, Future, Pin"]

        L6["━━━━━━ 内存管理 ━━━━━━"]
        L6_C["mem, ptr, alloc, Layout, NonNull"]

        L7["━━━━━━ 编译器接口 ━━━━━━"]
        L7_C["intrinsics, lang_items"]
    end

    L1_C --> L2_C
    L2_C --> L3_C
    L3_C --> L4_C
    L4_C --> L5_C
    L5_C --> L6_C
    L6_C --> L7_C
```

## 2. 模块依赖关系

```mermaid
graph LR
    subgraph "核心模块依赖"
        MEM["mem"]
        PTR["ptr"]
        ALLOC["alloc"]
        MARKER["marker"]
        OPS["ops"]
        ITER["iter"]
        OPTION["option"]
        RESULT["result"]
        FMT["fmt"]
        CELL["cell"]
        SYNC["sync"]
        FUTURE["future"]
        SLICE["slice"]
        STR["str"]
        HASH["hash"]
    end

    PTR --> MEM
    ALLOC --> PTR
    ALLOC --> MEM

    OPTION --> MARKER
    RESULT --> MARKER
    ITER --> OPTION

    OPS --> MARKER
    FMT --> OPS

    CELL --> MEM
    SYNC --> MEM
    FUTURE --> MARKER
    FUTURE --> OPS

    SLICE --> ITER
    STR --> SLICE
    HASH --> MARKER
```

## 3. 类型安全体系

```mermaid
graph TB
    subgraph "Rust 类型安全金字塔"
        TOP["零成本抽象"]

        LEVEL1["━━━ 编译时检查 ━━━"]
        L1A["所有权系统"]
        L1B["借用检查器"]
        L1C["生命周期"]

        LEVEL2["━━━ 类型标记 ━━━"]
        L2A["Send/Sync<br/>线程安全"]
        L2B["Copy/Clone<br/>复制语义"]
        L2C["Sized<br/>大小已知"]

        LEVEL3["━━━ 运行时检查 ━━━"]
        L3A["RefCell<br/>借用检查"]
        L3B["Rc/Arc<br/>引用计数"]
        L3C["Bounds Check<br/>数组边界"]

        LEVEL4["━━━ Unsafe 边界 ━━━"]
        L4A["原始指针"]
        L4B["FFI 调用"]
        L4C["内联汇编"]
    end

    TOP --> LEVEL1
    LEVEL1 --> L1A
    LEVEL1 --> L1B
    LEVEL1 --> L1C

    L1A --> LEVEL2
    L1B --> LEVEL2
    L1C --> LEVEL2

    LEVEL2 --> L2A
    LEVEL2 --> L2B
    LEVEL2 --> L2C

    L2A --> LEVEL3
    L2B --> LEVEL3
    L2C --> LEVEL3

    LEVEL3 --> L3A
    LEVEL3 --> L3B
    LEVEL3 --> L3C

    L3A --> LEVEL4
    L3B --> LEVEL4
    L3C --> LEVEL4

    LEVEL4 --> L4A
    LEVEL4 --> L4B
    LEVEL4 --> L4C
```

## 4. Trait 层级关系

```mermaid
graph TB
    subgraph "核心 Trait 继承关系"
        SIZED["Sized"]
        COPY["Copy"]
        CLONE["Clone"]
        PARTIALEQ["PartialEq"]
        EQ["Eq"]
        PARTIALORD["PartialOrd"]
        ORD["Ord"]
        HASH["Hash"]
        DEBUG["Debug"]
        DISPLAY["Display"]
        DEFAULT["Default"]
        SEND["Send"]
        SYNC["Sync"]
        UNPIN["Unpin"]

        COPY -->|requires| CLONE

        EQ -->|requires| PARTIALEQ
        PARTIALORD -->|requires| PARTIALEQ
        ORD -->|requires| EQ
        ORD -->|requires| PARTIALORD

        DISPLAY -.->|"ToString"| TO_STRING["ToString"]
    end
```

## 5. 迭代器适配器流水线

```mermaid
graph LR
    subgraph "迭代器处理流程"
        SOURCE["数据源<br/>Vec, Array, Range"]

        SOURCE --> INTO["into_iter()<br/>获取所有权"]
        SOURCE --> ITER["iter()<br/>借用迭代"]
        SOURCE --> ITER_MUT["iter_mut()<br/>可变借用"]

        INTO --> CHAIN["适配器链"]
        ITER --> CHAIN
        ITER_MUT --> CHAIN

        CHAIN --> MAP["map()"]
        MAP --> FILTER["filter()"]
        FILTER --> TAKE["take()"]
        TAKE --> ENUMERATE["enumerate()"]

        ENUMERATE --> COLLECT["collect()<br/>消费者"]
        ENUMERATE --> FOLD["fold()<br/>消费者"]
        ENUMERATE --> SUM["sum()<br/>消费者"]

        COLLECT --> RESULT_C["集合"]
        FOLD --> RESULT_F["聚合值"]
        SUM --> RESULT_S["总和"]
    end
```

## 6. 内存管理流程

```mermaid
graph TB
    subgraph "内存生命周期"
        START["需要内存"]

        START --> LAYOUT["计算 Layout<br/>size + align"]

        LAYOUT --> ALLOC_CHOICE{"分配位置"}

        ALLOC_CHOICE -->|栈| STACK["栈分配<br/>自动管理"]
        ALLOC_CHOICE -->|堆| HEAP["堆分配<br/>Box/Rc/Arc"]

        STACK --> USE_S["使用期"]
        HEAP --> USE_H["使用期"]

        USE_S --> DROP_S["作用域结束<br/>自动 Drop"]
        USE_H --> DROP_H{"Drop 时机"}

        DROP_H -->|Box| DROP_BOX["单一所有者<br/>离开作用域"]
        DROP_H -->|Rc| DROP_RC["最后一个 Rc<br/>引用计数归零"]
        DROP_H -->|Arc| DROP_ARC["最后一个 Arc<br/>原子引用计数归零"]

        DROP_S --> END["内存释放"]
        DROP_BOX --> END
        DROP_RC --> END
        DROP_ARC --> END
    end
```

## 7. 错误处理流程

```mermaid
graph TB
    subgraph "错误处理策略"
        OP["可能失败的操作"]

        OP --> RESULT["Result<T, E>"]

        RESULT --> OK{"结果?"}

        OK -->|Ok| SUCCESS["成功路径<br/>继续执行"]
        OK -->|Err| ERROR["错误路径"]

        ERROR --> STRATEGY{"处理策略"}

        STRATEGY -->|传播| PROPAGATE["? 运算符<br/>向上传递"]
        STRATEGY -->|处理| HANDLE["match/if let<br/>就地处理"]
        STRATEGY -->|恢复| RECOVER["unwrap_or<br/>提供默认值"]
        STRATEGY -->|终止| PANIC["unwrap/expect<br/>panic"]

        PROPAGATE --> CALLER["调用者处理"]
        HANDLE --> CONTINUE["继续执行"]
        RECOVER --> DEFAULT["使用默认值"]
        PANIC --> ABORT["程序终止"]
    end
```

## 8. 并发模型

```mermaid
graph TB
    subgraph "Rust 并发模型"
        DATA["数据"]

        DATA --> OWNERSHIP{"所有权模式"}

        OWNERSHIP -->|独占| MOVE["Move 语义<br/>转移所有权"]
        OWNERSHIP -->|共享不可变| SHARED_IMM["Arc<T><br/>共享不可变"]
        OWNERSHIP -->|共享可变| SHARED_MUT["同步原语"]

        SHARED_MUT --> MUTEX["Mutex<T><br/>互斥锁"]
        SHARED_MUT --> RWLOCK["RwLock<T><br/>读写锁"]
        SHARED_MUT --> ATOMIC["Atomic*<br/>原子类型"]

        MOVE --> THREAD1["线程 1"]
        SHARED_IMM --> THREAD2["多线程共享读"]
        MUTEX --> THREAD3["多线程互斥写"]
        RWLOCK --> THREAD4["多线程读多写少"]
        ATOMIC --> THREAD5["无锁并发"]
    end
```

## 9. 异步执行模型

```mermaid
sequenceDiagram
    participant App as 应用
    participant Runtime as 异步运行时
    participant Task as 任务
    participant IO as I/O 事件

    App->>Runtime: spawn(async_fn())
    Runtime->>Task: 创建 Future

    loop 轮询循环
        Runtime->>Task: poll()
        alt 未就绪
            Task-->>Runtime: Poll::Pending
            Task->>IO: 注册 Waker
            Note over IO: 等待事件
            IO-->>Runtime: Waker::wake()
            Runtime->>Runtime: 重新调度任务
        else 已完成
            Task-->>Runtime: Poll::Ready(result)
            Runtime-->>App: 返回结果
        end
    end
```

## 10. Pin 与自引用

```mermaid
graph TB
    subgraph "Pin 解决自引用问题"
        SELF_REF["自引用结构"]

        SELF_REF --> PROBLEM["问题：移动会破坏内部指针"]

        PROBLEM --> SOLUTION["解决方案：Pin"]

        SOLUTION --> PIN_TYPES["Pin 类型"]

        PIN_TYPES --> PIN_REF["Pin<&mut T><br/>栈上固定"]
        PIN_TYPES --> PIN_BOX["Pin<Box<T>><br/>堆上固定"]

        PIN_REF --> GUARANTEE["保证：值不会被移动"]
        PIN_BOX --> GUARANTEE

        GUARANTEE --> SAFE["自引用安全"]
        SAFE --> ASYNC["async/await 状态机"]
        SAFE --> INTRUSIVE["侵入式数据结构"]
    end
```

## 11. 智能指针对比

```mermaid
graph TB
    subgraph "智能指针选择"
        NEED["需要堆分配"]

        NEED --> Q1{"需要共享?"}

        Q1 -->|否| BOX["Box<T><br/>━━━━━━<br/>单一所有者<br/>最简单"]

        Q1 -->|是| Q2{"需要跨线程?"}

        Q2 -->|否| RC["Rc<T><br/>━━━━━━<br/>引用计数<br/>单线程"]

        Q2 -->|是| ARC["Arc<T><br/>━━━━━━<br/>原子引用计数<br/>多线程"]

        RC --> Q3{"需要可变?"}
        ARC --> Q4{"需要可变?"}

        Q3 -->|是| RC_REFCELL["Rc<RefCell<T>><br/>单线程共享可变"]
        Q4 -->|是| ARC_MUTEX["Arc<Mutex<T>><br/>多线程共享可变"]
    end
```

## 12. 内部可变性选择

```mermaid
graph TD
    subgraph "内部可变性选择指南"
        START["需要内部可变性"]

        START --> Q1{"跨线程?"}

        Q1 -->|是| SYNC["需要 Sync"]
        Q1 -->|否| SINGLE["单线程"]

        SYNC --> Q2{"数据类型?"}
        Q2 -->|简单整数| ATOMIC["Atomic*"]
        Q2 -->|复杂类型| MUTEX["Mutex<T>"]

        SINGLE --> Q3{"需要借用?"}
        Q3 -->|否| CELL["Cell<T><br/>移动语义"]
        Q3 -->|是| REFCELL["RefCell<T><br/>运行时借用检查"]

        SINGLE --> Q4{"只初始化一次?"}
        Q4 -->|是| ONCECELL["OnceCell<T>"]
    end
```

## 13. 生命周期关系

```mermaid
graph LR
    subgraph "生命周期层级"
        STATIC["'static<br/>━━━━━━<br/>程序全程<br/>最长"]

        OUTER["'outer<br/>━━━━━━<br/>外层作用域"]

        INNER["'inner<br/>━━━━━━<br/>内层作用域<br/>最短"]
    end

    STATIC -->|"outlives"| OUTER
    OUTER -->|"outlives"| INNER

    subgraph "生命周期规则"
        RULE1["引用的生命周期<br/>≤ 被引用数据的生命周期"]
        RULE2["借用期间<br/>数据不能被移动或销毁"]
        RULE3["可变借用<br/>独占访问"]
    end
```

## 14. 宏系统概览

```mermaid
graph TB
    subgraph "Rust 宏系统"
        MACRO["宏"]

        MACRO --> DECL["声明式宏<br/>macro_rules!"]
        MACRO --> PROC["过程宏"]

        DECL --> PATTERN["模式匹配<br/>展开代码"]

        PROC --> DERIVE["derive 宏<br/>#[derive(Trait)]"]
        PROC --> ATTR["属性宏<br/>#[attribute]"]
        PROC --> FN_LIKE["函数式宏<br/>name!(...)"]

        DERIVE --> AUTO_IMPL["自动实现 Trait"]
        ATTR --> TRANSFORM["代码变换"]
        FN_LIKE --> GENERATE["代码生成"]
    end
```

## 15. 特征对象与泛型

```mermaid
graph TB
    subgraph "多态实现方式"
        POLY["多态"]

        POLY --> GENERIC["泛型 (编译时)<br/>━━━━━━<br/>零成本抽象<br/>单态化"]

        POLY --> DYN["trait 对象 (运行时)<br/>━━━━━━<br/>动态分派<br/>vtable"]

        GENERIC --> MONO["fn foo<T: Trait>(x: T)<br/>每个 T 生成一份代码"]

        DYN --> VTABLE["fn bar(x: &dyn Trait)<br/>通过 vtable 调用"]

        MONO --> PRO_G["优点：性能最优"]
        MONO --> CON_G["缺点：代码膨胀"]

        VTABLE --> PRO_D["优点：代码精简"]
        VTABLE --> CON_D["缺点：间接调用开销"]
    end
```

---

## 使用说明

本文档中的 Mermaid 图表可以在支持 Mermaid 的 Markdown 查看器中渲染，包括：

- GitHub
- GitLab
- VS Code (with Mermaid extension)
- Typora
- Obsidian
- HackMD

如果图表无法正确渲染，请确保您的查看器支持最新版本的 Mermaid 语法。
