# 并发与异步模块

> `core::sync`, `core::cell`, `core::future`, `core::task`, `core::pin` 深度解析

## 概述

Rust 的并发模型建立在所有权系统之上，通过类型系统在编译时防止数据竞争。

```mermaid
graph TB
    subgraph "并发原语体系"
        CELL["cell<br/>内部可变性"]
        SYNC["sync<br/>线程同步"]
        FUTURE["future<br/>异步抽象"]
        TASK["task<br/>任务调度"]
        PIN["pin<br/>位置锁定"]
    end

    CELL --> UNSAFE_CELL["UnsafeCell"]
    CELL --> CELL_T["Cell<T>"]
    CELL --> REFCELL["RefCell<T>"]

    SYNC --> ATOMIC["atomic 模块"]

    FUTURE --> POLL["Poll 枚举"]
    FUTURE --> FUTURE_TRAIT["Future trait"]

    TASK --> WAKER["Waker"]
    TASK --> CONTEXT["Context"]

    PIN --> PIN_T["Pin<P>"]
    PIN --> UNPIN["Unpin trait"]
```

---

## core::cell 模块 - 内部可变性

### 概念

内部可变性允许在只有不可变引用时修改数据，是 Rust 中"共享 XOR 可变"规则的受控绕过。

```mermaid
graph TB
    subgraph "内部可变性层级"
        UNSAFE["UnsafeCell<T><br/>━━━━━━<br/>最底层原语<br/>所有安全抽象的基础"]

        CELL["Cell<T><br/>━━━━━━<br/>移动语义<br/>适用于 Copy 类型"]

        REFCELL["RefCell<T><br/>━━━━━━<br/>运行时借用检查<br/>适用于复杂类型"]

        ONCECELL["OnceCell<T><br/>━━━━━━<br/>一次性初始化<br/>无运行时检查"]

        LAZYCELL["LazyCell<T, F><br/>━━━━━━<br/>延迟初始化<br/>配对初始化函数"]
    end

    UNSAFE --> CELL
    UNSAFE --> REFCELL
    UNSAFE --> ONCECELL
    ONCECELL --> LAZYCELL
```

### Cell<T> - 移动语义的内部可变性

```rust
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}
```

**特性**：
- 通过**移动**而非借用修改值
- 永远无法获得 `&mut T`
- 对 `Copy` 类型提供 `get()` 方法
- 不是 `Sync`（仅用于单线程）
- 零运行时开销

**核心方法**：

```mermaid
graph LR
    subgraph "Cell<T> 方法"
        SET["set(val)<br/>设置新值"]
        REPLACE["replace(val) -> T<br/>替换并返回旧值"]
        GET["get() -> T<br/>复制值（Copy类型）"]
        TAKE["take() -> T<br/>用 Default 替换"]
    end
```

| 方法 | 功能 | 要求 |
|------|------|------|
| `set(val)` | 设置值，丢弃旧值 | - |
| `replace(val)` | 替换并返回旧值 | - |
| `get()` | 复制并返回值 | `T: Copy` |
| `take()` | 用 Default 替换 | `T: Default` |
| `into_inner()` | 消耗 Cell 返回值 | - |

### RefCell<T> - 动态借用检查

```rust
pub struct RefCell<T: ?Sized> {
    borrow: Cell<BorrowFlag>,
    value: UnsafeCell<T>,
}
```

**特性**：
- 将**编译时借用检查转移到运行时**
- 维护借用计数
- 违反借用规则时 panic
- 运行时开销：借用计数器

**核心方法**：

```mermaid
graph TB
    subgraph "RefCell<T> 借用方法"
        BORROW["borrow()<br/>━━━━━━<br/>返回 Ref<'_, T><br/>不可变借用"]

        BORROW_MUT["borrow_mut()<br/>━━━━━━<br/>返回 RefMut<'_, T><br/>可变借用"]

        TRY_BORROW["try_borrow()<br/>━━━━━━<br/>返回 Result<Ref, BorrowError><br/>不会 panic"]

        TRY_BORROW_MUT["try_borrow_mut()<br/>━━━━━━<br/>返回 Result<RefMut, BorrowMutError><br/>不会 panic"]
    end
```

**借用规则（运行时检查）**：

```mermaid
graph TD
    START["RefCell 状态"]

    START --> UNBORROWED["未借用"]
    UNBORROWED -->|"borrow()"| SHARED["共享借用<br/>（可多次）"]
    UNBORROWED -->|"borrow_mut()"| EXCLUSIVE["独占借用"]

    SHARED -->|"borrow()"| SHARED
    SHARED -->|"borrow_mut()"| PANIC1["PANIC!"]

    EXCLUSIVE -->|"borrow()"| PANIC2["PANIC!"]
    EXCLUSIVE -->|"borrow_mut()"| PANIC3["PANIC!"]

    style PANIC1 fill:#ff6b6b
    style PANIC2 fill:#ff6b6b
    style PANIC3 fill:#ff6b6b
```

### Cell vs RefCell 对比

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| 值访问 | 移动/复制 | 借用 |
| 运行时检查 | 无 | 有（借用计数） |
| 性能 | 极快 | 稍慢 |
| Copy 要求 | 需要（get） | 不需要 |
| panic 可能 | 无 | 借用冲突时 |
| 适用场景 | 简单类型 | 复杂类型 |

### OnceCell<T> 和 LazyCell<T>

```mermaid
graph LR
    subgraph "一次性初始化"
        ONCE["OnceCell<T><br/>━━━━━━<br/>只能设置一次<br/>可从 &self 获得 &T"]

        LAZY["LazyCell<T, F><br/>━━━━━━<br/>配对初始化函数<br/>首次访问时初始化"]
    end

    ONCE -->|"配合初始化函数"| LAZY
```

---

## core::sync::atomic 模块 - 原子操作

### 内存顺序

```mermaid
graph TB
    subgraph "内存顺序强度"
        RELAXED["Relaxed<br/>━━━━━━<br/>最弱<br/>仅保证原子性<br/>无同步"]

        ACQUIRE["Acquire<br/>━━━━━━<br/>读操作<br/>获取语义<br/>防止后续操作重排到前面"]

        RELEASE["Release<br/>━━━━━━<br/>写操作<br/>释放语义<br/>防止前面操作重排到后面"]

        ACQREL["AcqRel<br/>━━━━━━<br/>读-修改-写<br/>同时具有两种语义"]

        SEQCST["SeqCst<br/>━━━━━━<br/>最强<br/>顺序一致<br/>全局同步"]
    end

    RELAXED --> ACQUIRE
    RELAXED --> RELEASE
    ACQUIRE --> ACQREL
    RELEASE --> ACQREL
    ACQREL --> SEQCST
```

**选择指南**：

| 场景 | 推荐顺序 |
|------|----------|
| 简单计数器 | `Relaxed` |
| 锁的获取 | `Acquire` |
| 锁的释放 | `Release` |
| 自旋锁 | `AcqRel` |
| 需要完全同步 | `SeqCst` |

### Acquire-Release 配对

```mermaid
sequenceDiagram
    participant T1 as 线程1
    participant MEM as 内存
    participant T2 as 线程2

    T1->>MEM: 写入数据
    T1->>MEM: store(Release)
    Note over MEM: 同步点

    T2->>MEM: load(Acquire)
    T2->>T2: 读取数据
    Note over T2: 可见线程1写入的数据
```

---

## core::future 模块 - 异步抽象

### Future Trait

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

### Poll 枚举

```mermaid
graph LR
    subgraph "Poll 状态"
        POLL["Poll<T>"]

        POLL --> PENDING["Pending<br/>━━━━━━<br/>未完成<br/>需要再次轮询"]

        POLL --> READY["Ready(T)<br/>━━━━━━<br/>已完成<br/>返回结果"]
    end
```

### 异步执行流程

```mermaid
sequenceDiagram
    participant U as 用户代码
    participant R as 运行时
    participant F as Future
    participant W as Waker

    U->>R: spawn(future)
    R->>F: poll(cx)
    F-->>R: Poll::Pending
    F->>W: 注册 Waker

    Note over W: 等待事件

    W-->>R: wake()
    R->>R: 加入任务队列
    R->>F: poll(cx)
    F-->>R: Poll::Ready(result)
    R->>U: 返回结果
```

### Future 状态机 

```mermaid
stateDiagram-v2
    [*] --> Created: async fn 调用
    Created --> Polled: 首次 poll()
    Polled --> Pending: 未就绪
    Polled --> Ready: 已完成
    Pending --> Woken.Waker.wake()
    Woken --> Polled: 再次 poll()
    Ready --> [*]: 消费结果

    note right of Pending
        存储 Waker
        等待唤醒信号
    end note

    note right of Woken
        运行时调度
        重新加入队列
    end note
```

---

## core::task 模块 - 任务调度

### Waker 和 Context

```mermaid
graph TB
    subgraph "任务调度组件"
        CONTEXT["Context<'_><br/>━━━━━━<br/>poll() 的上下文<br/>包含 Waker 引用"]

        WAKER["Waker<br/>━━━━━━<br/>唤醒句柄<br/>可克隆、Send + Sync"]

        RAWWAKER["RawWaker<br/>━━━━━━<br/>底层实现<br/>数据指针 + vtable"]
    end

    CONTEXT --> WAKER
    WAKER --> RAWWAKER
```

### Waker 机制

```rust
impl Waker {
    // 唤醒关联的任务
    pub fn wake(self);

    // 唤醒但不消耗 Waker
    pub fn wake_by_ref(&self);

    // 创建空操作 Waker
    pub fn noop() -> &'static Waker;
}
```

```mermaid
graph LR
    subgraph "Waker 使用"
        FUTURE["Future"] -->|"保存"| WAKER["Waker"]
        EVENT["I/O 事件"] -->|"触发"| WAKER
        WAKER -->|"wake()"| RUNTIME["运行时"]
        RUNTIME -->|"poll()"| FUTURE
    end
```

---

## core::pin 模块 - 位置锁定

### Pin 的作用

```mermaid
graph TB
    subgraph "Pin 解决的问题"
        PROBLEM["自引用结构问题"]

        PROBLEM --> SELF_REF["结构体包含指向自身的指针"]
        SELF_REF --> MOVE["移动会使指针失效"]
        MOVE --> UB["未定义行为"]

        PIN["Pin<P>"]
        PIN --> GUARANTEE["保证值不会被移动"]
        GUARANTEE --> SAFE["自引用结构安全"]
    end

    UB -.->|"解决"| PIN
```

### Pin<P> 类型

```rust
pub struct Pin<Ptr> {
    pointer: Ptr,
}
```

**Pin 包装的是指针，不是值**：
- `Pin<&mut T>` - 固定的可变引用
- `Pin<Box<T>>` - 固定的堆分配值
- `Pin<Rc<T>>` - 固定的引用计数值

### Unpin Trait

```rust
pub auto trait Unpin { }
```

**语义**：
- 大多数类型自动实现 `Unpin`
- 实现 `Unpin` 的类型可以安全移动
- `Pin<&mut T>` 对 `T: Unpin` 等同于 `&mut T`

```mermaid
graph TD
    Q["类型 T"]

    Q -->|"T: Unpin"| UNPIN["可自由移动<br/>Pin 无约束力"]
    Q -->|"T: !Unpin"| PINNED["必须保持固定<br/>移动是 unsafe"]

    UNPIN --> USE1["Pin::new(&mut t)<br/>零成本"]
    PINNED --> USE2["Box::pin(t)<br/>堆分配"]
    PINNED --> USE3["pin!() 宏<br/>栈固定"]
```

### Pinning 三种方式

```mermaid
graph TD
    START["需要 Pin"]

    START --> Q1{"T: Unpin?"}

    Q1 -->|是| METHOD1["Pin::new(&mut value)<br/>━━━━━━<br/>零成本<br/>行为如同普通引用"]

    Q1 -->|否| Q2{"存储位置?"}

    Q2 -->|堆上| METHOD2["Box::pin(value)<br/>━━━━━━<br/>堆分配<br/>管理生命周期"]

    Q2 -->|栈上| METHOD3["pin!() 宏<br/>━━━━━━<br/>栈固定<br/>无分配"]
```

---

## 并发原语关系图

```mermaid
graph TD
    subgraph "单线程"
        CELL["Cell<T>"]
        REFCELL["RefCell<T>"]
        ONCECELL["OnceCell<T>"]
    end

    subgraph "多线程"
        ATOMIC["Atomic*"]
        MUTEX["Mutex (std)"]
        RWLOCK["RwLock (std)"]
    end

    subgraph "异步"
        FUTURE["Future"]
        PIN["Pin<P>"]
        WAKER["Waker"]
    end

    CELL -.->|"!Sync"| SINGLE["单线程安全"]
    REFCELL -.->|"!Sync"| SINGLE

    ATOMIC -.->|"Send + Sync"| MULTI["多线程安全"]

    FUTURE --> PIN
    FUTURE --> WAKER
```

---

## 设计原则

### 1. 编译时优先

```mermaid
graph LR
    subgraph "检查时机"
        COMPILE["编译时<br/>━━━━━━<br/>Send/Sync<br/>所有权/借用"]

        RUNTIME["运行时<br/>━━━━━━<br/>RefCell 借用<br/>原子操作"]
    end

    COMPILE -->|"尽可能"| RUNTIME
```

### 2. 显式控制

- 内存顺序必须显式指定
- Pin 要求显式声明意图
- 不隐藏潜在的性能陷阱

### 3. 零成本抽象

- Cell 无运行时开销
- Pin 与基础指针大小相同
- 原子操作直接映射到 CPU 指令

### 4. 安全边界清晰

```mermaid
graph LR
    SAFE["Safe API<br/>Cell, RefCell, Future"]
    UNSAFE["Unsafe 基础<br/>UnsafeCell, RawWaker"]

    UNSAFE -->|"包装"| SAFE
```

---

## 选择指南

### 内部可变性选择

```mermaid
graph TD
    START["需要内部可变性"]

    START --> Q1{"需要借用值？"}

    Q1 -->|否| Q2{"T: Copy?"}
    Q1 -->|是| REFCELL["RefCell<T>"]

    Q2 -->|是| CELL["Cell<T>"]
    Q2 -->|否| CELL2["Cell<T><br/>（使用 replace/take）"]

    START --> Q3{"只初始化一次？"}
    Q3 -->|是| ONCECELL["OnceCell<T>"]
```

### 线程同步选择

```mermaid
graph TD
    START["需要线程同步"]

    START --> Q1{"只是计数器？"}
    Q1 -->|是| ATOMIC["Atomic*<br/>（Relaxed 足够）"]

    Q1 -->|否| Q2{"需要复杂操作？"}
    Q2 -->|是| MUTEX["Mutex (std)"]
    Q2 -->|否| ATOMIC2["Atomic*<br/>（合适的内存顺序）"]
```
