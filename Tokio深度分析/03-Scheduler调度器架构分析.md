# Tokio Scheduler 调度器架构分析

## 目录

1. [调度器概述](#1-调度器概述)
2. [CurrentThread 调度器](#2-currentthread-调度器)
3. [MultiThread 调度器](#3-multithread-调度器)
4. [Work-Stealing 算法](#4-work-stealing-算法)
5. [任务队列系统](#5-任务队列系统)
6. [空闲管理](#6-空闲管理)
7. [LIFO 优化](#7-lifo-优化)

---

## 1. 调度器概述

Tokio 提供两种调度器实现，满足不同场景需求。

### 1.1 调度器类型对比

```mermaid
graph TB
    subgraph Scheduler["调度器类型"]
        CT["CurrentThread<br/>单线程调度器"]
        MT["MultiThread<br/>多线程调度器"]
    end

    subgraph CTFeatures["CurrentThread 特点"]
        CTQ["单一任务队列"]
        CTL["低开销"]
        CTN["支持非 Send Future"]
    end

    subgraph MTFeatures["MultiThread 特点"]
        MTW["Work-Stealing"]
        MTQ["本地队列 + 全局队列"]
        MTS["多核并行"]
        MTL["LIFO 优化"]
    end

    CT --> CTFeatures
    MT --> MTFeatures
```

### 1.2 性能特征对比

| 特性 | CurrentThread | MultiThread |
|------|---------------|-------------|
| 线程数 | 1 | N (可配置) |
| 队列类型 | VecDeque | Local(256) + Global |
| Work-Stealing | 无 | 有 |
| LIFO 优化 | 无 | 有 |
| 延迟 | 低 | 中等 |
| 吞吐量 | 中 | 高 |
| Send 约束 | 无 | 有 |

---

## 2. CurrentThread 调度器

### 2.1 核心数据结构

```rust
// 文件: runtime/scheduler/current_thread/mod.rs
pub(crate) struct CurrentThread {
    /// 核心调度状态 (原子包装)
    core: AtomicCell<Core>,

    /// 跨线程唤醒通知
    notify: Notify,
}

struct Core {
    /// 就绪任务队列
    tasks: VecDeque<Notified>,

    /// 调度计数器
    tick: u32,

    /// I/O 和时间驱动
    driver: Option<Driver>,

    /// 性能指标
    metrics: MetricsBatch,

    /// 全局队列检查间隔
    global_queue_interval: u32,

    /// 未处理 panic 标志
    unhandled_panic: bool,
}
```

### 2.2 架构图

```mermaid
graph TB
    subgraph CurrentThread["CurrentThread 调度器"]
        Core["Core<br/>核心状态"]
        Notify["Notify<br/>唤醒机制"]
        AtomicCell["AtomicCell<br/>原子包装"]
    end

    subgraph CoreContent["Core 内容"]
        Tasks["tasks: VecDeque<br/>就绪队列"]
        Driver["driver: Driver<br/>事件驱动"]
        Tick["tick: u32<br/>调度计数"]
    end

    subgraph Shared["共享状态"]
        Inject["inject: Inject<br/>全局注入队列"]
        Owned["owned: OwnedTasks<br/>任务集合"]
        Woken["woken: AtomicBool<br/>唤醒标志"]
    end

    Core --> CoreContent
    CurrentThread --> Core
    CurrentThread --> Notify
    CurrentThread --> AtomicCell

    Core -.-> Shared
```

### 2.3 执行流程

```mermaid
flowchart TD
    A["block_on(future)"] --> B["获取 Core"]
    B --> C{"获取成功?"}

    C -->|否| D["等待 notify"]
    D --> B

    C -->|是| E["进入 CoreGuard::block_on"]

    E --> F["重置 woken 标志"]
    F --> G["poll(future)"]

    G --> H{结果?}
    H -->|Ready| I["返回结果"]
    H -->|Pending| J["执行本地任务"]

    J --> K{"tick % interval == 0?"}
    K -->|是| L["检查全局队列"]
    K -->|否| M["检查本地队列"]

    L --> N["获取任务"]
    M --> N

    N --> O{有任务?}
    O -->|是| P["poll(task)"]
    P --> J

    O -->|否| Q["driver.park()"]
    Q --> F

    style A fill:#e3f2fd
    style I fill:#c8e6c9
```

### 2.4 任务调度策略

```rust
fn next_task(&mut self, handle: &Handle) -> Option<Notified> {
    if self.tick % self.global_queue_interval == 0 {
        // 定期检查全局注入队列
        handle.next_remote_task()
            .or_else(|| self.next_local_task())
    } else {
        // 优先本地队列
        self.next_local_task()
            .or_else(|| handle.next_remote_task())
    }
}
```

---

## 3. MultiThread 调度器

### 3.1 核心数据结构

```rust
// 文件: runtime/scheduler/multi_thread/worker.rs
pub(super) struct Worker {
    /// 调度器句柄
    handle: Arc<Handle>,

    /// 工作线程编号
    index: usize,

    /// 线程本地核心
    core: AtomicCell<Core>,
}

struct Core {
    /// 运行计数
    tick: u32,

    /// LIFO 优化槽
    lifo_slot: Option<Notified>,

    /// LIFO 是否启用
    lifo_enabled: bool,

    /// 本地运行队列 (256 容量)
    run_queue: queue::Local<Arc<Handle>>,

    /// 是否在搜索任务
    is_searching: bool,

    /// 是否已关闭
    is_shutdown: bool,

    /// 线程停泊器
    park: Option<Parker>,

    /// 统计信息
    stats: Stats,

    /// 全局队列检查间隔
    global_queue_interval: u32,

    /// 快速随机数
    rand: FastRand,
}
```

### 3.2 整体架构图

```mermaid
graph TB
    subgraph MultiThread["MultiThread 调度器"]
        Handle["Handle<br/>共享句柄"]
    end

    subgraph Workers["Worker 线程池"]
        W0["Worker 0"]
        W1["Worker 1"]
        WN["Worker N"]
    end

    subgraph WorkerCore["Worker Core"]
        LIFO["lifo_slot<br/>LIFO 槽"]
        LocalQ["run_queue<br/>本地队列(256)"]
        Stats["stats<br/>统计"]
    end

    subgraph SharedState["共享状态"]
        Remotes["remotes<br/>远程状态"]
        Inject["inject<br/>全局注入队列"]
        Idle["idle<br/>空闲管理"]
        Owned["owned<br/>任务集合"]
    end

    Handle --> SharedState
    Handle --> Workers

    W0 --> WorkerCore
    W1 --> WorkerCore
    WN --> WorkerCore

    W0 <-.->|steal| W1
    W1 <-.->|steal| WN
    W0 <-.->|steal| WN

    LocalQ -->|overflow| Inject
```

### 3.3 Worker 主循环

```mermaid
flowchart TD
    A["Worker 启动"] --> B["获取 Core"]
    B --> C["进入主循环"]

    C --> D{"检查 LIFO 槽"}
    D -->|有任务| E["执行 LIFO 任务"]
    D -->|无| F{"检查本地队列"}

    F -->|有任务| G["执行本地任务"]
    F -->|无| H{"tick % interval?"}

    H -->|需要| I["检查全局队列"]
    H -->|不需要| J["尝试 Work-Stealing"]

    I --> K{有任务?}
    K -->|是| G
    K -->|否| J

    J --> L{窃取成功?}
    L -->|是| G
    L -->|否| M["进入搜索状态"]

    M --> N{"再次检查队列"}
    N -->|有| G
    N -->|无| O["park() 等待"]

    O --> P["被唤醒"]
    P --> C

    E --> C
    G --> C

    style A fill:#e3f2fd
    style O fill:#fff3e0
```

---

## 4. Work-Stealing 算法

### 4.1 算法概述

Work-Stealing 是一种负载均衡策略，空闲的 Worker 可以从繁忙的 Worker 窃取任务。

```mermaid
graph LR
    subgraph Before["窃取前"]
        W0B["Worker 0<br/>空闲"]
        W1B["Worker 1<br/>8个任务"]
        W2B["Worker 2<br/>2个任务"]
    end

    subgraph After["窃取后"]
        W0A["Worker 0<br/>4个任务"]
        W1A["Worker 1<br/>4个任务"]
        W2A["Worker 2<br/>2个任务"]
    end

    W0B -->|"steal 50%"| W1B
    W1B -.-> W1A
    W0B -.-> W0A
```

### 4.2 窃取流程

```mermaid
sequenceDiagram
    participant W0 as Worker 0 (空闲)
    participant W1 as Worker 1 (繁忙)
    participant Global as 全局队列

    W0->>W0: 本地队列为空
    W0->>W0: LIFO 槽为空
    W0->>Global: 检查全局队列
    Global-->>W0: 空

    W0->>W0: transition_to_searching()
    Note over W0: 检查: searching < workers/2

    W0->>W0: 随机选择目标
    W0->>W1: steal_into()

    Note over W1: 锁定 head 指针
    W1->>W1: 计算可窃取数量
    Note over W1: n = (tail - head) / 2

    W1->>W0: 转移 ~50% 任务
    W0->>W0: 继续执行
```

### 4.3 窃取算法实现

```rust
// 文件: runtime/scheduler/multi_thread/queue.rs
pub(crate) fn steal_into(
    &self,
    dst: &mut Local<T>,
    dst_stats: &mut Stats,
) -> Option<task::Notified<T>> {
    // 1. 检查目标队列容量
    let dst_tail = dst.inner.tail.unsync_load();
    if dst_tail.wrapping_sub(steal) > CAPACITY / 2 {
        return None;  // 目标队列已满
    }

    // 2. 原子声称任务
    let n = src_tail.wrapping_sub(src_head_real);
    let n = n - n / 2;  // 窃取 ~50%

    // 3. CAS 更新 head 指针
    match self.0.head.compare_exchange(
        prev_packed,
        pack(src_head_steal, steal_to),
        AcqRel,
        Acquire,
    ) {
        Ok(_) => { /* 成功 */ }
        Err(actual) => { /* 重试 */ }
    }

    // 4. 复制任务到目标队列
    for i in 0..n {
        // 内存复制
    }

    // 5. 完成窃取，更新指针
    Some(ret)
}
```

### 4.4 搜索频率限制

```mermaid
graph TB
    A["Worker 准备搜索"] --> B{"num_searching < workers/2?"}

    B -->|是| C["transition_to_searching()"]
    C --> D["开始窃取"]

    B -->|否| E["直接 park()"]

    D --> F{窃取成功?}
    F -->|是| G["transition_from_searching()"]
    F -->|否| H["继续搜索其他 Worker"]

    H --> I{所有 Worker 检查完?}
    I -->|否| D
    I -->|是| E

    style B fill:#fff3e0
```

---

## 5. 任务队列系统

### 5.1 队列层级结构

```mermaid
graph TB
    subgraph QueueHierarchy["队列层级"]
        LIFO["LIFO Slot<br/>1个任务<br/>最高优先级"]
        Local["Local Queue<br/>256个任务<br/>中优先级"]
        Global["Global Inject Queue<br/>无限容量<br/>低优先级"]
    end

    LIFO --> Local
    Local -->|overflow| Global
    Global -->|pull| Local
```

### 5.2 本地队列结构

```rust
// 文件: runtime/scheduler/multi_thread/queue.rs
pub(crate) struct Local<T: 'static> {
    inner: Arc<Inner<T>>,
}

pub(crate) struct Inner<T: 'static> {
    /// 双指针队列头
    /// MSB: steal_head (正在被窃取的位置)
    /// LSB: real_head (真实的头指针)
    head: AtomicUnsignedLong,

    /// 队列尾 (仅生产者更新)
    tail: AtomicUnsignedShort,

    /// 固定大小缓冲区
    buffer: Box<[UnsafeCell<MaybeUninit<Notified<T>>>; 256]>,
}
```

### 5.3 队列状态编码

```mermaid
graph LR
    subgraph HeadEncoding["Head 编码 (64位)"]
        MSB["高16位<br/>steal_head"]
        LSB["低16位<br/>real_head"]
    end

    subgraph States["状态含义"]
        Same["steal == real<br/>无活跃窃取"]
        Diff["steal != real<br/>正在窃取中"]
    end

    MSB --> States
    LSB --> States
```

### 5.4 溢出处理

```mermaid
flowchart TD
    A["push_back(task)"] --> B{"本地队列满?"}

    B -->|否| C["直接入队"]
    B -->|是| D{"有窃取进行中?"}

    D -->|是| E["推送到全局队列"]
    D -->|否| F["push_overflow()"]

    F --> G["声称 128 个任务"]
    G --> H["移动到全局队列"]
    H --> I["腾出空间"]
    I --> C

    style F fill:#fff3e0
```

---

## 6. 空闲管理

### 6.1 空闲状态编码

```rust
// 文件: runtime/scheduler/multi_thread/idle.rs
pub(super) struct Idle {
    /// 状态编码
    /// 高16位: 未停泊的 worker 数
    /// 低16位: 正在搜索的 worker 数
    state: AtomicUsize,

    /// worker 总数
    num_workers: usize,
}
```

### 6.2 Worker 生命周期

```mermaid
stateDiagram-v2
    [*] --> Running: Worker 启动

    Running --> Searching: 本地队列空
    Searching --> Running: 找到任务
    Searching --> Parked: 搜索失败

    Parked --> Searching: 被唤醒

    Running --> [*]: 关闭

    note right of Running
        处理任务
        num_unparked++
    end note

    note right of Searching
        Work-Stealing
        num_searching++
    end note

    note right of Parked
        等待唤醒
        加入 sleepers
    end note
```

### 6.3 唤醒决策

```mermaid
flowchart TD
    A["notify_should_wakeup()"] --> B{"num_searching == 0?"}

    B -->|否| C["不唤醒<br/>有人在搜索"]
    B -->|是| D{"num_unparked < num_workers?"}

    D -->|否| E["不唤醒<br/>都在运行"]
    D -->|是| F["需要唤醒"]

    F --> G["worker_to_notify()"]
    G --> H["从 sleepers 弹出"]
    H --> I["unpark(worker)"]

    style F fill:#c8e6c9
    style C fill:#ffcdd2
    style E fill:#ffcdd2
```

---

## 7. LIFO 优化

### 7.1 LIFO 槽设计原理

LIFO (Last In First Out) 槽用于优化消息传递场景的缓存局部性。

```mermaid
sequenceDiagram
    participant TaskA as 任务 A
    participant LIFOSlot as LIFO Slot
    participant TaskB as 任务 B
    participant LocalQueue as Local Queue

    TaskA->>TaskA: 执行中
    TaskA->>TaskB: wake() 唤醒 B

    alt LIFO 启用
        TaskB->>LIFOSlot: 放入 LIFO 槽
        Note over LIFOSlot: 热缓存
    else LIFO 禁用/已满
        TaskB->>LocalQueue: 放入本地队列
    end

    TaskA->>TaskA: 完成当前 poll

    Note over LIFOSlot: 下次调度
    LIFOSlot->>TaskB: 立即执行 B
    Note over TaskB: 缓存命中率高
```

### 7.2 LIFO 调度逻辑

```rust
fn schedule_local(&self, core: &mut Core, task: Notified, is_yield: bool) {
    if is_yield || !core.lifo_enabled {
        // yield 或禁用时入队
        core.run_queue.push_back_or_overflow(task, ...);
    } else {
        // 放入 LIFO 槽
        let prev = core.lifo_slot.take();

        if let Some(prev) = prev {
            // 前一个 LIFO 任务入队
            core.run_queue.push_back_or_overflow(prev, ...);
        }

        core.lifo_slot = Some(task);
    }
}
```

### 7.3 防止 Ping-Pong 饥饿

```mermaid
flowchart TD
    A["执行 LIFO 任务"] --> B{"lifo_polls < 3?"}

    B -->|是| C["执行任务"]
    C --> D["lifo_polls++"]
    D --> E{"LIFO 槽还有任务?"}
    E -->|是| A
    E -->|否| F["检查本地队列"]

    B -->|否| G["禁用 LIFO"]
    G --> H["LIFO 任务入队"]
    H --> F

    style G fill:#ffcdd2
```

### 7.4 LIFO 优化效果

```mermaid
graph LR
    subgraph Without["无 LIFO 优化"]
        A1["Task A poll"] --> B1["Task B 入队"]
        B1 --> C1["执行其他任务"]
        C1 --> D1["Task B poll"]
        D1 --> E1["缓存 Miss"]
    end

    subgraph With["有 LIFO 优化"]
        A2["Task A poll"] --> B2["Task B 入 LIFO"]
        B2 --> C2["Task B poll"]
        C2 --> D2["缓存 Hit!"]
    end

    style E1 fill:#ffcdd2
    style D2 fill:#c8e6c9
```

---

## 性能调优建议

### 调度器选择指南

```mermaid
graph TD
    A["选择调度器"] --> B{"需要多核并行?"}

    B -->|否| C{"Future 是 Send?"}
    C -->|否| D["CurrentThread"]
    C -->|是| E{"任务数量多?"}
    E -->|否| D
    E -->|是| F["MultiThread"]

    B -->|是| F

    style D fill:#e3f2fd
    style F fill:#c8e6c9
```

### 关键配置参数

| 参数 | 默认值 | 调优建议 |
|-----|-------|---------|
| `worker_threads` | CPU 核心数 | I/O 密集型可增加 50-100% |
| `global_queue_interval` | 自适应 | 低延迟场景减小 |
| `event_interval` | 61 | 高频 I/O 可减小 |
| `disable_lifo_slot` | false | 严格 FIFO 需求时启用 |

---

## 总结

Tokio 调度器的设计精髓在于：

1. **两种模式**: 单线程简单高效，多线程高吞吐
2. **Work-Stealing**: 自动负载均衡
3. **LIFO 优化**: 改善缓存局部性
4. **分层队列**: 平衡公平性和性能
5. **搜索限制**: 避免过度竞争
