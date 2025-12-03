# Tokio I/O Driver 驱动架构分析

## 目录

1. [I/O Driver 概述](#1-io-driver-概述)
2. [核心数据结构](#2-核心数据结构)
3. [Registration 机制](#3-registration-机制)
4. [ScheduledIo 状态管理](#4-scheduledio-状态管理)
5. [事件处理流程](#5-事件处理流程)
6. [与 Mio 集成](#6-与-mio-集成)
7. [性能优化](#7-性能优化)

---

## 1. I/O Driver 概述

Tokio I/O Driver 基于 Mio 库实现跨平台异步 I/O，采用事件驱动的反应器模式。

### 1.1 架构层次

```mermaid
graph TB
    subgraph Application["应用层"]
        TcpStream["TcpStream"]
        UdpSocket["UdpSocket"]
        UnixStream["UnixStream"]
    end

    subgraph TokioIO["Tokio I/O 层"]
        AsyncFd["AsyncFd<T>"]
        PollEvented["PollEvented<E>"]
        Registration["Registration"]
    end

    subgraph Driver["I/O Driver"]
        IODriver["Driver"]
        Handle["Handle"]
        ScheduledIo["ScheduledIo"]
    end

    subgraph Mio["Mio 库"]
        Poll["mio::Poll"]
        Registry["mio::Registry"]
        Events["mio::Events"]
    end

    subgraph OS["操作系统"]
        Epoll["epoll (Linux)"]
        Kqueue["kqueue (BSD/macOS)"]
        IOCP["IOCP (Windows)"]
    end

    Application --> TokioIO
    TokioIO --> Driver
    Driver --> Mio
    Mio --> OS

    style Application fill:#e3f2fd
    style TokioIO fill:#fff3e0
    style Driver fill:#e8f5e9
    style Mio fill:#fce4ec
    style OS fill:#f5f5f5
```

### 1.2 设计目标

```mermaid
mindmap
    root((I/O Driver))
        跨平台
            Linux epoll
            macOS kqueue
            Windows IOCP
        高性能
            零拷贝
            批量处理
            缓存友好
        安全性
            Tick 验证
            原子操作
            生命周期管理
```

---

## 2. 核心数据结构

### 2.1 Driver 结构

```rust
// 文件: runtime/io/driver.rs
pub(crate) struct Driver {
    /// 是否接收到信号令牌
    signal_ready: bool,

    /// 复用的事件缓冲区
    events: mio::Events,

    /// 操作系统级别的事件轮询器
    poll: mio::Poll,
}
```

### 2.2 Handle 结构

```rust
pub(crate) struct Handle {
    /// 操作系统事件注册中心
    registry: mio::Registry,

    /// 所有注册的追踪
    registrations: RegistrationSet,

    /// 需要同步的状态
    synced: Mutex<registration_set::Synced>,

    /// 唤醒停滞的反应器
    waker: mio::Waker,

    /// 驱动指标
    metrics: IoDriverMetrics,
}
```

### 2.3 结构关系图

```mermaid
classDiagram
    class Driver {
        -signal_ready: bool
        -events: mio::Events
        -poll: mio::Poll
        +park()
        +park_timeout(duration)
        +turn(handle, events)
    }

    class Handle {
        -registry: mio::Registry
        -registrations: RegistrationSet
        -synced: Mutex~Synced~
        -waker: mio::Waker
        +add_source(source, interest)
        +deregister_source(registration)
    }

    class Registration {
        -handle: scheduler::Handle
        -shared: Arc~ScheduledIo~
        +poll_read_ready()
        +poll_write_ready()
        +clear_readiness(event)
    }

    class ScheduledIo {
        -readiness: AtomicUsize
        -waiters: Mutex~Waiters~
        +set_readiness(tick, ready)
        +clear_readiness(event)
        +wake(ready)
    }

    Driver --> Handle
    Handle --> Registration
    Registration --> ScheduledIo
```

---

## 3. Registration 机制

### 3.1 Registration 结构

```rust
// 文件: runtime/io/registration.rs
pub(crate) struct Registration {
    /// 运行时句柄
    handle: scheduler::Handle,

    /// 指向 Driver 中的共享状态
    shared: Arc<ScheduledIo>,
}
```

### 3.2 注册流程

```mermaid
sequenceDiagram
    participant App as 应用代码
    participant Reg as Registration
    participant Handle as IO Handle
    participant Registry as mio::Registry
    participant OS as 操作系统

    App->>Reg: Registration::new(source, interest)
    Reg->>Handle: add_source(source, interest)
    Handle->>Handle: 创建 ScheduledIo

    Handle->>Registry: registry.register(source, token, interest)
    Registry->>OS: epoll_ctl(EPOLL_CTL_ADD, ...)

    OS-->>Registry: 成功
    Registry-->>Handle: 成功
    Handle-->>Reg: Arc<ScheduledIo>
    Reg-->>App: Registration
```

### 3.3 poll_read_ready 流程

```mermaid
flowchart TD
    A["poll_read_ready(cx)"] --> B["poll_ready(Direction::Read)"]

    B --> C["获取当前 readiness"]
    C --> D{包含 READABLE?}

    D -->|是| E["返回 Poll::Ready(event)"]

    D -->|否| F["获取 waiters 锁"]
    F --> G["再次检查 readiness"]

    G --> H{现在 READABLE?}
    H -->|是| I["释放锁"]
    I --> E

    H -->|否| J{"reader 槽位空?"}

    J -->|是| K["存储 Waker 到 reader"]
    J -->|否| L{"是否 will_wake?"}

    L -->|是| M["保持现有 Waker"]
    L -->|否| N["更新 Waker"]

    K --> O["返回 Poll::Pending"]
    M --> O
    N --> O
```

---

## 4. ScheduledIo 状态管理

### 4.1 ScheduledIo 结构

```rust
// 文件: runtime/io/scheduled_io.rs
pub(crate) struct ScheduledIo {
    /// 链表指针 (用于 RegistrationSet)
    linked_list_pointers: UnsafeCell<linked_list::Pointers<Self>>,

    /// 打包的就绪状态
    readiness: AtomicUsize,

    /// 等待者列表
    waiters: Mutex<Waiters>,
}

struct Waiters {
    /// AsyncRead 快速路径
    reader: Option<Waker>,

    /// AsyncWrite 快速路径
    writer: Option<Waker>,

    /// 通用等待者链表
    list: WaitList,
}
```

### 4.2 Readiness 位编码

```mermaid
graph LR
    subgraph ReadinessBits["readiness 位编码 (usize)"]
        Shutdown["bit 0<br/>shutdown"]
        Tick["bit 1-15<br/>driver_tick"]
        Ready["bit 16-31<br/>readiness"]
    end

    subgraph ReadyFlags["就绪标志"]
        READABLE["READABLE<br/>0b0001"]
        WRITABLE["WRITABLE<br/>0b0010"]
        READ_CLOSED["READ_CLOSED<br/>0b0100"]
        WRITE_CLOSED["WRITE_CLOSED<br/>0b1000"]
        PRIORITY["PRIORITY<br/>0b10000"]
        ERROR["ERROR<br/>0b100000"]
    end
```

### 4.3 Tick 验证机制

```mermaid
sequenceDiagram
    participant Driver as Driver
    participant SIO as ScheduledIo
    participant Task as Task

    Note over Driver,Task: 场景: 防止过期事件

    Driver->>SIO: set_readiness(Tick=5, READABLE)
    SIO-->>Task: 任务被唤醒

    Task->>SIO: poll_read_ready()
    SIO-->>Task: ReadyEvent{tick:5, ready:READABLE}

    Task->>Task: try_io(read)
    Note over Task: 操作成功

    Task->>SIO: clear_readiness(tick=5)

    Note over Driver,Task: 同时新事件到达

    Driver->>SIO: set_readiness(Tick=6, READABLE)

    Task->>SIO: clear_readiness(tick=5)
    Note over SIO: Tick 不匹配!<br/>忽略清除操作

    Note over SIO: READABLE 保持设置
```

### 4.4 状态转换

```mermaid
stateDiagram-v2
    [*] --> Unset: 创建

    Unset --> Ready: set_readiness()
    Unset --> WaiterAdded: 添加等待者

    WaiterAdded --> Ready: 事件到达
    WaiterAdded --> WaiterAdded: 更多等待者

    Ready --> Ready: 更多事件
    Ready --> Cleared: clear_readiness()<br/>tick 匹配

    Cleared --> Ready: 新事件
    Cleared --> WaiterAdded: 添加等待者

    Ready --> Shutdown: shutdown()
    WaiterAdded --> Shutdown: shutdown()

    Shutdown --> [*]: 释放
```

---

## 5. 事件处理流程

### 5.1 Driver::turn 主循环

```mermaid
flowchart TD
    A["Driver::turn()"] --> B["mio::Poll::poll()"]

    B --> C["遍历 events"]

    C --> D{事件类型?}

    D -->|普通 I/O| E["处理 I/O 事件"]
    D -->|Waker| F["处理唤醒事件"]
    D -->|Signal| G["设置 signal_ready"]

    E --> H["获取 ScheduledIo"]
    H --> I["Ready::from_mio(event)"]
    I --> J["set_readiness(Tick::Set, ready)"]
    J --> K["wake(ready)"]

    K --> L{更多事件?}
    L -->|是| C
    L -->|否| M["返回"]

    F --> L
    G --> L
```

### 5.2 唤醒流程详解

```mermaid
sequenceDiagram
    participant OS as 操作系统
    participant Poll as mio::Poll
    participant Driver as Driver
    participant SIO as ScheduledIo
    participant Waker as Waker
    participant Task as Task
    participant Scheduler as Scheduler

    OS->>Poll: 数据到达
    Poll->>Driver: event (fd, READABLE)

    Driver->>SIO: set_readiness(READABLE)
    Driver->>SIO: wake(READABLE)

    SIO->>SIO: 获取 waiters 锁
    SIO->>Waker: 收集匹配的 Waker

    loop 批量唤醒
        SIO->>SIO: 缓冲区满?
        alt 满了
            SIO->>SIO: 释放锁
            SIO->>Waker: wake_all()
            Waker->>Scheduler: 重新调度任务
            SIO->>SIO: 重新获取锁
        end
    end

    SIO->>SIO: 释放锁
    SIO->>Waker: wake_all()
    Waker->>Task: 唤醒
```

### 5.3 完整 I/O 操作流程

```mermaid
sequenceDiagram
    participant App as 应用代码
    participant Stream as TcpStream
    participant Reg as Registration
    participant SIO as ScheduledIo
    participant Driver as Driver
    participant OS as 操作系统

    App->>Stream: read(buf).await

    Stream->>Reg: poll_read_ready()
    Reg->>SIO: 检查 readiness

    alt 已就绪
        SIO-->>Reg: Ready
        Reg-->>Stream: ReadyEvent
        Stream->>OS: read(fd, buf)
        OS-->>Stream: n bytes
        Stream-->>App: Ok(n)
    else 未就绪
        SIO->>SIO: 注册 Waker
        SIO-->>Reg: Pending
        Reg-->>Stream: Pending
        Stream-->>App: 挂起

        Note over Driver,OS: 等待事件

        OS->>Driver: epoll 返回
        Driver->>SIO: set_readiness(READABLE)
        Driver->>SIO: wake()
        SIO->>App: 唤醒任务

        App->>Stream: 再次 poll
        Stream->>Reg: poll_read_ready()
        Reg->>SIO: 检查 readiness
        SIO-->>Reg: Ready
        Stream->>OS: read(fd, buf)
        OS-->>Stream: n bytes
        Stream-->>App: Ok(n)
    end
```

---

## 6. 与 Mio 集成

### 6.1 平台抽象

```mermaid
graph TB
    subgraph Mio["Mio 跨平台抽象"]
        Poll["mio::Poll"]
        Registry["mio::Registry"]
        Token["mio::Token"]
        Interest["mio::Interest"]
    end

    subgraph Linux["Linux"]
        Epoll["epoll_create"]
        EpollCtl["epoll_ctl"]
        EpollWait["epoll_wait"]
    end

    subgraph BSD["BSD/macOS"]
        Kqueue["kqueue"]
        Kevent["kevent"]
    end

    subgraph Windows["Windows"]
        IOCP["CreateIoCompletionPort"]
        GQCS["GetQueuedCompletionStatus"]
    end

    Poll --> Epoll
    Poll --> Kqueue
    Poll --> IOCP

    Registry --> EpollCtl
    Registry --> Kevent
    Registry --> IOCP
```

### 6.2 Token 映射

```rust
// Token 直接存储 ScheduledIo 指针
let token = mio::Token(Arc::as_ptr(&scheduled_io) as usize);

// 从 Token 恢复 ScheduledIo
let scheduled_io = unsafe {
    &*(token.0 as *const ScheduledIo)
};
```

### 6.3 Interest 转换

```mermaid
graph LR
    subgraph TokioInterest["Tokio Interest"]
        TRead["READABLE"]
        TWrite["WRITABLE"]
        TPriority["PRIORITY"]
        TError["ERROR"]
    end

    subgraph MioInterest["Mio Interest"]
        MRead["READABLE"]
        MWrite["WRITABLE"]
        MPriority["PRIORITY"]
    end

    TRead --> MRead
    TWrite --> MWrite
    TPriority --> MPriority
```

---

## 7. 性能优化

### 7.1 缓存行对齐

```rust
// 根据架构选择对齐大小
#[cfg_attr(
    any(target_arch = "x86_64", target_arch = "aarch64"),
    repr(align(128))
)]
pub(crate) struct ScheduledIo {
    // ...
}
```

```mermaid
graph LR
    subgraph Alignment["对齐策略"]
        X86_64["x86_64: 128 字节"]
        ARM64["aarch64: 128 字节"]
        ARM["arm: 32 字节"]
        S390X["s390x: 256 字节"]
    end
```

### 7.2 两层唤醒机制

```mermaid
graph TB
    subgraph FastPath["快速路径"]
        Reader["reader: Option<Waker>"]
        Writer["writer: Option<Waker>"]
    end

    subgraph SlowPath["通用路径"]
        WaitList["list: LinkedList<Waiter>"]
    end

    subgraph Usage["使用场景"]
        Single["单任务读/写<br/>使用快速路径"]
        Multi["多任务<br/>使用链表"]
    end

    FastPath --> Single
    SlowPath --> Multi
```

### 7.3 批量唤醒

```rust
pub(super) fn wake(&self, ready: Ready) {
    // 栈上缓冲区，容量 32
    let mut wakers = WakeList::new();
    let mut waiters = self.waiters.lock();

    // 收集 Waker
    loop {
        // ...收集匹配的 Waker...

        // 缓冲区满时释放锁并唤醒
        if !wakers.can_push() {
            drop(waiters);
            wakers.wake_all();
            waiters = self.waiters.lock();
        }
    }

    drop(waiters);
    wakers.wake_all();
}
```

### 7.4 延迟释放机制

```mermaid
graph TB
    subgraph RegistrationSet["RegistrationSet"]
        Active["活跃注册<br/>LinkedList"]
        Pending["待释放<br/>Vec (阈值 16)"]
    end

    subgraph Lifecycle["生命周期"]
        New["新注册"] --> Active
        Active -->|drop| Pending
        Pending -->|"达到阈值"| Release["批量释放"]
    end
```

---

## 性能特性总结

| 特性 | 复杂度 | 说明 |
|-----|--------|------|
| set_readiness | O(1) | 单次原子操作 |
| poll_readiness | O(1) | 检查状态 |
| wake (单任务) | O(1) | 快速路径 |
| wake (多任务) | O(n) | 遍历等待者 |
| Token 查找 | O(1) | 指针直接转换 |

---

## 总结

Tokio I/O Driver 的设计精髓：

1. **跨平台**: 通过 Mio 抽象不同操作系统
2. **高效唤醒**: 两层机制优化单任务场景
3. **安全性**: Tick 验证防止竞态条件
4. **缓存友好**: 对齐和批量处理
5. **零拷贝**: 直接指针作为 Token
