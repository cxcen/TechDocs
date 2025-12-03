# Tokio Task 任务系统深度分析

## 目录

1. [任务系统概述](#1-任务系统概述)
2. [任务内存布局](#2-任务内存布局)
3. [任务状态机](#3-任务状态机)
4. [引用计数机制](#4-引用计数机制)
5. [JoinHandle 实现](#5-joinhandle-实现)
6. [Waker 机制](#6-waker-机制)
7. [任务生命周期](#7-任务生命周期)

---

## 1. 任务系统概述

Tokio 任务系统是运行时的核心，负责管理异步任务的创建、执行和销毁。

### 1.1 设计目标

```mermaid
mindmap
    root((任务系统))
        高性能
            零成本抽象
            缓存友好
            最小化分配
        内存安全
            引用计数
            原子操作
            Pin 约束
        灵活性
            类型擦除
            可取消
            可等待
```

### 1.2 核心组件

```mermaid
graph TB
    subgraph TaskComponents["任务组件"]
        Task["Task<br/>任务结构"]
        JoinHandle["JoinHandle<br/>等待句柄"]
        Waker["Waker<br/>唤醒器"]
        Notified["Notified<br/>就绪通知"]
    end

    subgraph Internal["内部结构"]
        Header["Header<br/>状态和元信息"]
        Core["Core<br/>Future 容器"]
        Trailer["Trailer<br/>冷数据"]
    end

    Task --> Header
    Task --> Core
    Task --> Trailer

    JoinHandle -->|持有| Task
    Waker -->|指向| Task
    Notified -->|表示| Task
```

---

## 2. 任务内存布局

### 2.1 Cell 结构

任务使用单次堆分配，包含三个部分：

```rust
// 文件: runtime/task/core.rs
#[repr(C)]
pub(super) struct Cell<T: Future, S: Schedule> {
    /// 热数据 - 缓存行对齐
    pub(super) header: Header,

    /// Future 和调度器
    pub(super) core: Core<T, S>,

    /// 冷数据
    pub(super) trailer: Trailer,
}
```

### 2.2 内存布局图

```mermaid
graph TB
    subgraph Cell["Cell<T, S> - 单次堆分配"]
        subgraph Header["Header (热数据)"]
            State["state: AtomicUsize<br/>状态 + 引用计数"]
            QueueNext["queue_next: Option<NonNull>"]
            Vtable["vtable: &'static Vtable"]
            OwnerId["owner_id: Option<NonZeroU64>"]
        end

        subgraph Core["Core<T, S>"]
            Scheduler["scheduler: S"]
            TaskId["task_id: Id"]
            Stage["stage: CoreStage<T>"]
        end

        subgraph Trailer["Trailer (冷数据)"]
            Owned["owned: Pointers<Header>"]
            WakerCell["waker: Option<Waker>"]
            Hooks["hooks: TaskHarnessScheduleHooks"]
        end
    end

    Header --> Core
    Core --> Trailer

    style Header fill:#ffecb3
    style Core fill:#c8e6c9
    style Trailer fill:#e1f5fe
```

### 2.3 缓存行对齐

根据 CPU 架构选择对齐大小：

```mermaid
graph LR
    subgraph Alignment["缓存行对齐策略"]
        X86_64["x86_64, aarch64, ppc64<br/>128 字节"]
        ARM["arm, mips, sparc<br/>32 字节"]
        X86["x86, riscv, wasm<br/>64 字节"]
        S390X["s390x<br/>256 字节"]
    end
```

### 2.4 Stage 枚举

```rust
pub(super) enum Stage<T: Future> {
    /// Future 正在运行
    Running(T),

    /// Future 已完成，存储结果
    Finished(Result<T::Output, JoinError>),

    /// 结果已被消费
    Consumed,
}
```

```mermaid
stateDiagram-v2
    [*] --> Running: 创建任务

    Running --> Finished: poll() -> Ready
    Running --> Running: poll() -> Pending

    Finished --> Consumed: JoinHandle 取走结果
    Finished --> Consumed: 任务被取消

    Consumed --> [*]: 释放内存
```

---

## 3. 任务状态机

### 3.1 状态字段编码

State 使用单个 `AtomicUsize` 存储状态标志和引用计数：

```
位布局 (64位系统):
┌────────────────────────────────┬────────┬────────┬───────────┬──────────┬──────────┬─────────┐
│        引用计数 (58位)          │CANCELLED│JOIN_WAKER│JOIN_INTEREST│NOTIFIED │ COMPLETE │ RUNNING │
│           bit 6+               │  bit 5  │  bit 4  │   bit 3    │  bit 2  │  bit 1   │  bit 0  │
└────────────────────────────────┴────────┴────────┴───────────┴──────────┴──────────┴─────────┘
```

### 3.2 状态常量

```rust
// 文件: runtime/task/state.rs
const RUNNING:       usize = 0b0000_0001;  // 任务正在 poll
const COMPLETE:      usize = 0b0000_0010;  // 任务已完成
const NOTIFIED:      usize = 0b0000_0100;  // 存在 Notified 对象
const JOIN_INTEREST: usize = 0b0000_1000;  // 存在 JoinHandle
const JOIN_WAKER:    usize = 0b0001_0000;  // 设置了 join waker
const CANCELLED:     usize = 0b0010_0000;  // 任务被取消

const REF_ONE:       usize = 0b0100_0000;  // 单个引用计数单位

// 初始状态: 3个引用 + JOIN_INTEREST + NOTIFIED
const INITIAL_STATE: usize = (REF_ONE * 3) | JOIN_INTEREST | NOTIFIED;
```

### 3.3 状态转换图

```mermaid
stateDiagram-v2
    [*] --> Idle: spawn()<br/>state = INITIAL

    state Idle {
        [*] --> NotifiedIdle: NOTIFIED=1
    }

    state Running {
        [*] --> Polling: RUNNING=1
    }

    state Complete {
        [*] --> OutputStored: COMPLETE=1
    }

    Idle --> Running: transition_to_running()
    Running --> Idle: transition_to_idle()<br/>无新通知
    Running --> Running: 新的 wake()

    Running --> Complete: poll() Ready<br/>transition_to_complete()
    Running --> Complete: CANCELLED<br/>取消处理

    Complete --> [*]: 所有引用释放<br/>dealloc()

    note right of Idle
        RUNNING=0
        可能 NOTIFIED=1
    end note

    note right of Running
        RUNNING=1
        获得 Future 互斥访问
    end note

    note right of Complete
        COMPLETE=1 (永不清除)
        输出可被读取
    end note
```

### 3.4 关键状态转换方法

```mermaid
flowchart TD
    subgraph transition_to_running["transition_to_running()"]
        A1["检查 NOTIFIED=1"] --> A2["设置 RUNNING=1"]
        A2 --> A3["清除 NOTIFIED=0"]
        A3 --> A4{"CANCELLED?"}
        A4 -->|是| A5["返回 Cancelled"]
        A4 -->|否| A6["返回 Success"]
    end

    subgraph transition_to_idle["transition_to_idle()"]
        B1["检查 CANCELLED"] --> B2{"被取消?"}
        B2 -->|是| B3["返回 Cancelled"]
        B2 -->|否| B4["清除 RUNNING=0"]
        B4 --> B5{"新的 NOTIFIED?"}
        B5 -->|是| B6["ref++, 返回 OkNotified"]
        B5 -->|否| B7["ref--, 返回 Ok/OkDealloc"]
    end

    subgraph transition_to_complete["transition_to_complete()"]
        C1["XOR (RUNNING | COMPLETE)"]
        C1 --> C2["RUNNING: 1->0"]
        C2 --> C3["COMPLETE: 0->1"]
    end
```

---

## 4. 引用计数机制

### 4.1 引用计数来源

```mermaid
graph TB
    subgraph RefSources["引用计数来源"]
        OwnedTask["OwnedTask<br/>在 OwnedTasks 中"]
        JoinHandle["JoinHandle<br/>用户持有"]
        Waker["Waker<br/>可有多个"]
        Notified["Notified<br/>在调度队列中"]
    end

    subgraph InitialRefs["初始引用 (3个)"]
        Ref1["1: OwnedTasks"]
        Ref2["2: Notified (调度器)"]
        Ref3["3: JoinHandle"]
    end

    OwnedTask --> Ref1
    JoinHandle --> Ref3
    Notified --> Ref2
    Waker -.->|动态| RefSources
```

### 4.2 引用计数生命周期

```mermaid
sequenceDiagram
    participant spawn as spawn()
    participant task as Task
    participant scheduler as Scheduler
    participant user as User Code

    spawn->>task: 创建, ref=3
    Note over task: OwnedTask + Notified + JoinHandle

    scheduler->>task: poll(), 消耗 Notified
    task->>task: ref=2 (idle时)

    user->>task: wake()
    task->>task: 创建新 Notified, ref=3

    scheduler->>task: poll() -> Ready
    task->>task: complete(), ref=2

    user->>task: JoinHandle.await
    task->>user: 返回结果
    user->>task: drop JoinHandle, ref=1

    scheduler->>task: release(), ref=0
    task->>task: dealloc()
```

### 4.3 引用计数操作

```rust
// 增加引用 - Relaxed 顺序
fn ref_inc(&self) {
    self.val.fetch_add(REF_ONE, Ordering::Relaxed);
}

// 减少引用 - AcqRel 顺序确保可见性
fn ref_dec(&self) -> bool {
    let prev = self.val.fetch_sub(REF_ONE, Ordering::AcqRel);
    prev.ref_count() == 1  // 是否变为 0
}

// 双倍减少 (快速路径优化)
fn ref_dec_twice(&self) -> bool {
    let prev = self.val.fetch_sub(2 * REF_ONE, Ordering::AcqRel);
    prev.ref_count() == 2
}
```

---

## 5. JoinHandle 实现

### 5.1 JoinHandle 结构

```rust
// 文件: runtime/task/join.rs
pub struct JoinHandle<T> {
    raw: RawTask,
    _p: PhantomData<T>,
}
```

### 5.2 JoinHandle 功能

```mermaid
graph LR
    subgraph JoinHandle["JoinHandle<T>"]
        Await["await<br/>等待任务完成"]
        Abort["abort()<br/>取消任务"]
        IsFinished["is_finished()<br/>检查完成"]
        Id["id()<br/>获取任务ID"]
    end

    subgraph Internal["内部操作"]
        PollOutput["try_read_output()"]
        SetWaker["set_join_waker()"]
        DropHandle["drop_join_handle()"]
    end

    Await --> PollOutput
    Await --> SetWaker
    Abort --> Internal
```

### 5.3 Future 实现

```mermaid
flowchart TD
    A["JoinHandle::poll()"] --> B["try_read_output()"]

    B --> C{任务完成?}

    C -->|是| D["读取输出"]
    D --> E["返回 Poll::Ready(result)"]

    C -->|否| F{"JOIN_WAKER 已设置?"}

    F -->|是| G["检查 will_wake()"]
    G --> H{相同 Waker?}
    H -->|是| I["返回 Poll::Pending"]
    H -->|否| J["更新 Waker"]

    F -->|否| K["设置 JOIN_WAKER"]

    J --> I
    K --> I
```

### 5.4 JOIN_WAKER 访问协议

```mermaid
stateDiagram-v2
    [*] --> JW_Unset: JOIN_WAKER=0

    JW_Unset --> JW_Setting: JoinHandle.poll()
    JW_Setting --> JW_Set: set_join_waker()

    JW_Set --> JW_Set: 更新 Waker

    JW_Set --> JW_Unsetting: COMPLETE=1
    JW_Unsetting --> JW_Runtime: unset_waker()

    JW_Runtime --> [*]: cleanup

    note right of JW_Unset
        JoinHandle 独占 waker 访问
    end note

    note right of JW_Set
        两端都可读
        JoinHandle 可修改
    end note

    note right of JW_Runtime
        Runtime 独占 (完成后)
    end note
```

### 5.5 Drop 实现

```mermaid
flowchart TD
    A["JoinHandle::drop()"] --> B["drop_join_handle_fast()"]

    B --> C{CAS 成功?}

    C -->|是| D["快速路径完成"]

    C -->|否| E["drop_join_handle_slow()"]
    E --> F["transition_to_join_handle_dropped()"]
    F --> G{"需要清理?"}

    G -->|drop_output| H["drop_future_or_output()"]
    G -->|drop_waker| I["set_waker(None)"]

    H --> J["drop_reference()"]
    I --> J
    J --> K{"ref == 0?"}

    K -->|是| L["dealloc()"]
    K -->|否| M["完成"]

    style D fill:#c8e6c9
    style E fill:#fff3e0
```

---

## 6. Waker 机制

### 6.1 Waker 创建

```rust
// 文件: runtime/task/waker.rs
pub(super) fn waker_ref<S>(header: &Header) -> WakerRef<'_>
where
    S: Schedule,
{
    let raw = RawWaker::new(
        header as *const _ as *const (),
        &RawWakerVTable::new(
            clone_waker::<S>,
            wake_by_val::<S>,
            wake_by_ref::<S>,
            drop_waker::<S>,
        ),
    );

    WakerRef::new(unsafe { Waker::from_raw(raw) })
}
```

### 6.2 Waker VTable

```mermaid
graph TB
    subgraph VTable["Waker VTable"]
        Clone["clone<br/>增加引用计数"]
        WakeVal["wake_by_val<br/>消耗 Waker 唤醒"]
        WakeRef["wake_by_ref<br/>借用 Waker 唤醒"]
        Drop["drop<br/>减少引用计数"]
    end

    subgraph Implementation["实现"]
        CloneImpl["ref_inc()"]
        WakeImpl["schedule_task()"]
        DropImpl["ref_dec()"]
    end

    Clone --> CloneImpl
    WakeVal --> WakeImpl
    WakeRef --> WakeImpl
    Drop --> DropImpl
```

### 6.3 唤醒流程

```mermaid
sequenceDiagram
    participant Future as Future
    participant Waker as Waker
    participant Task as Task
    participant Scheduler as Scheduler

    Future->>Waker: waker.wake()
    Waker->>Task: transition_to_notified()

    alt 任务空闲
        Task->>Task: 设置 NOTIFIED=1
        Task->>Task: 创建 Notified
        Task->>Scheduler: schedule(Notified)
    else 任务运行中
        Task->>Task: 设置 NOTIFIED=1
        Note over Task: 下次 idle 时处理
    end
```

---

## 7. 任务生命周期

### 7.1 完整生命周期图

```mermaid
graph TB
    subgraph Spawn["创建阶段"]
        S1["tokio::spawn(future)"]
        S2["分配 Cell<T,S>"]
        S3["初始化状态"]
        S4["返回 JoinHandle"]
    end

    subgraph Schedule["调度阶段"]
        SC1["加入 OwnedTasks"]
        SC2["推送 Notified"]
        SC3["Worker 获取任务"]
    end

    subgraph Execute["执行阶段"]
        E1["transition_to_running()"]
        E2["创建 waker_ref"]
        E3["poll_future()"]
        E4{"结果?"}
        E5["transition_to_idle()"]
        E6["transition_to_complete()"]
    end

    subgraph Complete["完成阶段"]
        C1["存储输出"]
        C2["唤醒 JoinHandle"]
        C3["release()"]
        C4["dealloc()"]
    end

    S1 --> S2 --> S3 --> S4
    S4 --> SC1 --> SC2 --> SC3
    SC3 --> E1 --> E2 --> E3 --> E4

    E4 -->|Pending| E5
    E5 -->|NOTIFIED| SC3
    E5 -->|无通知| SC3

    E4 -->|Ready| E6
    E6 --> C1 --> C2 --> C3 --> C4

    style S1 fill:#e3f2fd
    style C4 fill:#ffcdd2
```

### 7.2 Harness::poll() 详细流程

```mermaid
flowchart TD
    A["Harness::poll()"] --> B["poll_inner()"]

    B --> C["transition_to_running()"]
    C --> D{转换结果?}

    D -->|Success| E["创建 waker_ref"]
    D -->|Cancelled| F["cancel_task()"]
    D -->|Failed| G["返回"]

    E --> H["poll_future()"]
    H --> I["catch_unwind 包装"]

    I --> J{poll 结果?}

    J -->|Ready(output)| K["存储输出"]
    J -->|Pending| L["transition_to_idle()"]
    J -->|Panic| M["存储 panic 信息"]

    K --> N["complete()"]
    M --> N

    L --> O{转换结果?}
    O -->|OkNotified| P["重新调度"]
    O -->|Ok| Q["返回"]
    O -->|OkDealloc| R["dealloc()"]
    O -->|Cancelled| F

    N --> S["wake_join()"]
    S --> T["release()"]
    T --> U["transition_to_terminal()"]
    U --> V{最后引用?}
    V -->|是| R
    V -->|否| Q

    F --> N

    style A fill:#e3f2fd
    style R fill:#ffcdd2
```

### 7.3 取消流程

```mermaid
sequenceDiagram
    participant User as User Code
    participant AbortHandle as AbortHandle
    participant Task as Task
    participant Scheduler as Scheduler

    User->>AbortHandle: abort()
    AbortHandle->>Task: set CANCELLED flag

    alt 任务空闲
        Task->>Task: transition_to_notified_and_cancel()
        Task->>Scheduler: schedule(Notified)
    else 任务运行中
        Note over Task: 等待当前 poll 完成
    end

    Scheduler->>Task: poll()
    Task->>Task: transition_to_running()
    Note over Task: 检测到 CANCELLED

    Task->>Task: cancel_task()
    Task->>Task: drop_future_or_output()
    Task->>Task: 存储 JoinError::cancelled()

    Task->>Task: complete()
    Task->>User: JoinHandle 返回 Err
```

---

## 类型擦除机制

### Vtable 结构

```rust
// 文件: runtime/task/raw.rs
pub(super) struct Vtable {
    /// 轮询任务
    pub(super) poll: unsafe fn(NonNull<Header>),

    /// 调度任务
    pub(super) schedule: unsafe fn(NonNull<Header>),

    /// 释放内存
    pub(super) dealloc: unsafe fn(NonNull<Header>),

    /// 尝试读取输出
    pub(super) try_read_output: unsafe fn(NonNull<Header>, *mut (), &Waker),

    /// 慢速路径 drop JoinHandle
    pub(super) drop_join_handle_slow: unsafe fn(NonNull<Header>),

    /// drop AbortHandle
    pub(super) drop_abort_handle: unsafe fn(NonNull<Header>),

    /// 关闭任务
    pub(super) shutdown: unsafe fn(NonNull<Header>),

    /// 字段偏移量
    pub(super) trailer_offset: usize,
    pub(super) scheduler_offset: usize,
    pub(super) id_offset: usize,
}
```

### 类型擦除流程

```mermaid
graph LR
    subgraph TypedWorld["类型化世界"]
        Cell["Cell<T, S>"]
        Future["T: Future"]
        Scheduler["S: Schedule"]
    end

    subgraph ErasedWorld["擦除后世界"]
        Header["*mut Header"]
        VtablePtr["&'static Vtable"]
        Offsets["字段偏移量"]
    end

    Cell -->|类型擦除| Header
    Future -->|存储| Cell
    Scheduler -->|存储| Cell

    Header -->|通过 Vtable| VtablePtr
    VtablePtr -->|指针算术| Offsets
    Offsets -->|恢复| Cell
```

---

## 性能优化

### 关键优化点

| 优化 | 描述 | 效果 |
|-----|------|------|
| 单次分配 | 整个 Cell 一次分配 | 减少碎片 |
| 缓存对齐 | Header 独立缓存行 | 减少伪共享 |
| 位字段 | 状态+引用计数复用 | 减少原子操作 |
| 快速路径 | JoinHandle drop 优化 | 减少锁竞争 |
| Relaxed 顺序 | 引用计数增加 | 提高性能 |

---

## 总结

Tokio 任务系统的设计精髓：

1. **零成本抽象**: 通过 Vtable 实现类型擦除，无运行时开销
2. **内存效率**: 单次分配、位字段紧凑、缓存对齐
3. **并发安全**: 精细的内存顺序控制
4. **灵活性**: 支持取消、超时、等待
5. **可观测性**: 任务 ID、追踪支持
