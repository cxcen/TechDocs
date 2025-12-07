# Rust 并发编程详解

## 1. 并发模型概览

Rust 的并发安全建立在类型系统之上，通过 Send 和 Sync trait 在编译时保证线程安全。

```mermaid
graph TB
    subgraph "Rust 并发基础"
        OWNERSHIP["所有权系统<br/>数据竞争编译时检测"]
        SEND["Send trait<br/>可跨线程转移"]
        SYNC["Sync trait<br/>可跨线程共享引用"]
    end

    subgraph "并发原语"
        THREAD["std::thread<br/>系统线程"]
        SYNC_PRIM["std::sync<br/>同步原语"]
        ATOMIC["std::sync::atomic<br/>原子操作"]
        MPSC["std::sync::mpsc<br/>消息传递"]
    end

    OWNERSHIP --> THREAD
    SEND --> THREAD
    SYNC --> SYNC_PRIM

    style OWNERSHIP fill:#c8e6c9
    style SEND fill:#bbdefb
    style SYNC fill:#fff9c4
```

---

## 2. 线程 (std::thread)

### 创建与管理

```mermaid
sequenceDiagram
    participant Main as 主线程
    participant Spawn as thread::spawn
    participant Worker as 工作线程

    Main->>Spawn: spawn(|| { ... })
    Spawn->>Worker: 创建新线程
    Spawn-->>Main: JoinHandle<T>

    Worker->>Worker: 执行闭包
    Worker-->>Worker: 返回结果 T

    Main->>Main: handle.join()
    Worker-->>Main: Result<T, Error>
```

### 线程 API

```mermaid
graph TB
    subgraph "创建线程"
        SPAWN["thread::spawn(|| {...})<br/>返回 JoinHandle<T>"]
        BUILDER["thread::Builder::new()<br/>.name(&quot;worker&quot;)<br/>.stack_size(4096)<br/>.spawn(|| {...})"]
    end

    subgraph "JoinHandle 方法"
        JOIN["join() -> Result<T><br/>等待线程完成"]
        THREAD["thread() -> &Thread<br/>获取 Thread 引用"]
        IS_FINISHED["is_finished() -> bool<br/>检查是否完成"]
    end

    subgraph "当前线程"
        CURRENT["thread::current()<br/>获取当前线程"]
        SLEEP["thread::sleep(Duration)<br/>休眠"]
        YIELD_NOW["thread::yield_now()<br/>让出 CPU"]
        PARK["thread::park()<br/>阻塞线程"]
        UNPARK["thread.unpark()<br/>唤醒线程"]
    end

    SPAWN --> JOIN
    BUILDER --> JOIN
    CURRENT --> SLEEP
    CURRENT --> PARK

    style SPAWN fill:#c8e6c9
    style JOIN fill:#bbdefb
    style SLEEP fill:#fff9c4
```

### 数据传递

```mermaid
graph TB
    subgraph "move 闭包"
        MOVE["move || {<br/>    // 获取所有权<br/>    println!(&quot;{}&quot;, data);<br/>}"]
        MOVE_REQ["要求: 捕获的值实现 Send"]
    end

    subgraph "共享数据"
        ARC_DATA["Arc<T><br/>原子引用计数<br/>跨线程共享所有权"]
        MUTEX["Mutex<T><br/>互斥锁<br/>独占访问"]
        COMBINED["Arc<Mutex<T>><br/>共享可变数据"]
    end

    MOVE --> MOVE_REQ
    ARC_DATA --> COMBINED
    MUTEX --> COMBINED

    style MOVE fill:#c8e6c9
    style ARC_DATA fill:#bbdefb
    style COMBINED fill:#e1bee7
```

---

## 3. 互斥锁 Mutex<T>

Mutex 提供互斥访问，同一时间只有一个线程可以访问数据。

```mermaid
stateDiagram-v2
    [*] --> Unlocked: 初始状态

    Unlocked --> Locked: lock() 成功
    Locked --> Unlocked: MutexGuard drop

    Locked --> Blocked: 其他线程 lock()
    Blocked --> Locked: 获得锁

    Locked --> Poisoned: 持有锁的线程 panic
    Poisoned --> Recovered: into_inner() 恢复
```

### Mutex 使用模式

```mermaid
graph TB
    subgraph "获取锁"
        LOCK["mutex.lock()<br/>阻塞等待<br/>返回 LockResult<MutexGuard>"]
        TRY_LOCK["mutex.try_lock()<br/>非阻塞尝试<br/>返回 TryLockResult<MutexGuard>"]
    end

    subgraph "MutexGuard"
        GUARD["MutexGuard<T><br/>• 实现 Deref/DerefMut<br/>• Drop 时自动释放锁<br/>• 作用域结束自动解锁"]
    end

    subgraph "常见模式"
        PATTERN1["let data = mutex.lock().unwrap();<br/>// 使用 data<br/>// 作用域结束，锁释放"]
        PATTERN2["{<br/>    let mut data = mutex.lock().unwrap();<br/>    *data += 1;<br/>} // 显式限定作用域"]
    end

    LOCK --> GUARD
    GUARD --> PATTERN1
    GUARD --> PATTERN2

    style LOCK fill:#c8e6c9
    style GUARD fill:#bbdefb
```

### 死锁预防

```mermaid
graph TB
    subgraph "死锁场景"
        DEADLOCK["线程A: lock(m1) -> lock(m2)<br/>线程B: lock(m2) -> lock(m1)<br/>= 死锁"]
    end

    subgraph "预防策略"
        ORDER["固定顺序获取锁"]
        TRYLOCK["使用 try_lock"]
        SINGLE["单一锁保护相关数据"]
        CHANNEL["使用消息传递替代"]
    end

    DEADLOCK --> ORDER
    DEADLOCK --> TRYLOCK
    DEADLOCK --> SINGLE
    DEADLOCK --> CHANNEL

    style DEADLOCK fill:#ffccbc
    style ORDER fill:#c8e6c9
```

---

## 4. 读写锁 RwLock<T>

RwLock 允许多读单写，适合读多写少的场景。

```mermaid
stateDiagram-v2
    [*] --> Free: 无锁

    Free --> ReadLocked: read() [N个读者]
    ReadLocked --> ReadLocked: read() [增加读者]
    ReadLocked --> Free: 所有 RwLockReadGuard drop

    Free --> WriteLocked: write() [1个写者]
    WriteLocked --> Free: RwLockWriteGuard drop

    ReadLocked --> Waiting: write() 等待
    Waiting --> WriteLocked: 所有读者释放
```

### RwLock vs Mutex

```mermaid
graph TB
    subgraph "RwLock"
        RW_READ["read()<br/>共享读锁<br/>多个同时"]
        RW_WRITE["write()<br/>独占写锁<br/>阻塞所有"]
        RW_PERF["适用场景:<br/>读多写少"]
    end

    subgraph "Mutex"
        M_LOCK["lock()<br/>独占访问<br/>无论读写"]
        M_PERF["适用场景:<br/>读写均衡<br/>简单场景"]
    end

    RW_READ --> RW_PERF
    RW_WRITE --> RW_PERF
    M_LOCK --> M_PERF

    style RW_READ fill:#c8e6c9
    style RW_WRITE fill:#ffccbc
    style M_LOCK fill:#bbdefb
```

---

## 5. 原子类型 (std::sync::atomic)

原子操作提供无锁的线程安全原语。

```mermaid
graph TB
    subgraph "原子类型"
        BOOL["AtomicBool"]
        INT["AtomicI8/I16/I32/I64/Isize"]
        UINT["AtomicU8/U16/U32/U64/Usize"]
        PTR["AtomicPtr<T>"]
    end

    subgraph "常用操作"
        LOAD["load(Ordering)<br/>读取值"]
        STORE["store(val, Ordering)<br/>存储值"]
        SWAP["swap(val, Ordering)<br/>交换并返回旧值"]
        CAS["compare_exchange<br/>比较并交换"]
        FETCH["fetch_add/sub/and/or/xor<br/>原子算术操作"]
    end

    BOOL --> LOAD
    INT --> FETCH
    UINT --> CAS
    PTR --> SWAP

    style BOOL fill:#c8e6c9
    style LOAD fill:#bbdefb
```

### 内存顺序

```mermaid
graph TB
    subgraph "Ordering 选项"
        RELAXED["Relaxed<br/>• 最弱保证<br/>• 只保证原子性<br/>• 性能最好"]
        ACQUIRE["Acquire<br/>• 读操作<br/>• 防止后续操作重排到前面"]
        RELEASE["Release<br/>• 写操作<br/>• 防止之前操作重排到后面"]
        ACQREL["AcqRel<br/>• 读写操作<br/>• Acquire + Release"]
        SEQCST["SeqCst<br/>• 最强保证<br/>• 全局顺序一致<br/>• 性能最差"]
    end

    RELAXED --> ACQUIRE
    ACQUIRE --> ACQREL
    RELEASE --> ACQREL
    ACQREL --> SEQCST

    style RELAXED fill:#c8e6c9
    style SEQCST fill:#e1bee7
```

### 使用示例

```mermaid
sequenceDiagram
    participant T1 as 线程1
    participant Flag as AtomicBool
    participant Data as 数据
    participant T2 as 线程2

    T1->>Data: 写入数据
    T1->>Flag: store(true, Release)
    Note over Flag: Release 保证<br/>数据写入对其他线程可见

    T2->>Flag: load(Acquire)
    Note over Flag: Acquire 保证<br/>看到 Flag=true 则能看到数据
    T2->>Data: 读取数据
```

---

## 6. 通道 (std::sync::mpsc)

MPSC (Multiple Producer, Single Consumer) 通道实现消息传递。

```mermaid
graph LR
    subgraph "生产者 (多个)"
        P1["Producer 1"]
        P2["Producer 2"]
        P3["Producer 3"]
    end

    subgraph "通道"
        CHANNEL["mpsc::channel()<br/>或<br/>mpsc::sync_channel(n)"]
    end

    subgraph "消费者 (单个)"
        C["Consumer"]
    end

    P1 -->|"send()"| CHANNEL
    P2 -->|"send()"| CHANNEL
    P3 -->|"send()"| CHANNEL
    CHANNEL -->|"recv()"| C

    style CHANNEL fill:#c8e6c9
```

### 通道类型

```mermaid
graph TB
    subgraph "异步通道 channel()"
        ASYNC_TX["Sender<T><br/>• 可 clone<br/>• send() 不阻塞<br/>• 无界缓冲"]
        ASYNC_RX["Receiver<T><br/>• 不可 clone<br/>• recv() 可能阻塞"]
    end

    subgraph "同步通道 sync_channel(n)"
        SYNC_TX["SyncSender<T><br/>• 可 clone<br/>• send() 可能阻塞<br/>• 缓冲区满时等待"]
        SYNC_RX["Receiver<T><br/>• 不可 clone<br/>• recv() 可能阻塞"]
    end

    subgraph "接收方法"
        RECV["recv() -> Result<T><br/>阻塞等待"]
        TRY_RECV["try_recv() -> Result<T><br/>非阻塞"]
        RECV_TIMEOUT["recv_timeout(dur)<br/>带超时"]
        ITER["iter() / try_iter()<br/>迭代器"]
    end

    ASYNC_RX --> RECV
    SYNC_RX --> RECV

    style ASYNC_TX fill:#c8e6c9
    style SYNC_TX fill:#bbdefb
    style RECV fill:#fff9c4
```

### 通道状态

```mermaid
stateDiagram-v2
    [*] --> Open: channel() 创建

    Open --> Open: send/recv 正常
    Open --> SenderDropped: 所有 Sender drop
    Open --> ReceiverDropped: Receiver drop

    SenderDropped --> [*]: recv() 返回 Err
    ReceiverDropped --> [*]: send() 返回 Err
```

---

## 7. Arc<T> 原子引用计数

Arc 允许多个所有者跨线程共享数据。

```mermaid
graph TB
    subgraph "Arc 结构"
        ARC1["Arc<T> #1"]
        ARC2["Arc<T> #2"]
        ARC3["Arc<T> #3"]

        INNER["ArcInner<T><br/>├─ strong: AtomicUsize (3)<br/>├─ weak: AtomicUsize<br/>└─ data: T"]
    end

    ARC1 --> INNER
    ARC2 --> INNER
    ARC3 --> INNER

    style INNER fill:#c8e6c9
```

### Arc vs Rc

```mermaid
graph TB
    subgraph "Rc<T>"
        RC_TRAITS["• !Send<br/>• !Sync"]
        RC_PERF["• 非原子计数<br/>• 更快"]
        RC_USE["• 单线程"]
    end

    subgraph "Arc<T>"
        ARC_TRAITS["• Send (若 T: Send)<br/>• Sync (若 T: Sync)"]
        ARC_PERF["• 原子计数<br/>• 略慢"]
        ARC_USE["• 多线程"]
    end

    RC_TRAITS --> RC_PERF
    RC_PERF --> RC_USE

    ARC_TRAITS --> ARC_PERF
    ARC_PERF --> ARC_USE

    style RC_TRAITS fill:#c8e6c9
    style ARC_TRAITS fill:#bbdefb
```

### 常见组合

```mermaid
graph TB
    subgraph "共享不可变数据"
        ARC_T["Arc<T>"]
        ARC_USE1["多线程读取相同配置"]
    end

    subgraph "共享可变数据"
        ARC_MUTEX["Arc<Mutex<T>>"]
        ARC_USE2["多线程修改共享状态"]
    end

    subgraph "读多写少"
        ARC_RWLOCK["Arc<RwLock<T>>"]
        ARC_USE3["缓存、配置热更新"]
    end

    ARC_T --> ARC_USE1
    ARC_MUTEX --> ARC_USE2
    ARC_RWLOCK --> ARC_USE3

    style ARC_T fill:#c8e6c9
    style ARC_MUTEX fill:#bbdefb
    style ARC_RWLOCK fill:#fff9c4
```

---

## 8. 条件变量 Condvar

Condvar 用于线程间等待和通知。

```mermaid
sequenceDiagram
    participant T1 as 等待线程
    participant Lock as Mutex
    participant CV as Condvar
    participant T2 as 通知线程

    T1->>Lock: lock()
    T1->>T1: 检查条件 (不满足)
    T1->>CV: wait(guard)
    Note over T1,CV: 释放锁并休眠

    T2->>Lock: lock()
    T2->>T2: 修改状态
    T2->>CV: notify_one()
    T2->>Lock: unlock (guard drop)

    CV-->>T1: 唤醒
    T1->>Lock: 重新获取锁
    T1->>T1: 检查条件 (满足)
    T1->>T1: 继续执行
```

### Condvar API

```mermaid
graph TB
    subgraph "等待方法"
        WAIT["wait(guard)<br/>无条件等待"]
        WAIT_WHILE["wait_while(guard, |&x| pred)<br/>条件等待循环"]
        WAIT_TIMEOUT["wait_timeout(guard, dur)<br/>带超时等待"]
    end

    subgraph "通知方法"
        NOTIFY_ONE["notify_one()<br/>唤醒一个等待者"]
        NOTIFY_ALL["notify_all()<br/>唤醒所有等待者"]
    end

    WAIT --> NOTIFY_ONE
    WAIT_WHILE --> NOTIFY_ALL

    style WAIT fill:#c8e6c9
    style NOTIFY_ONE fill:#bbdefb
```

---

## 9. 屏障 Barrier

Barrier 让多个线程在某点同步。

```mermaid
sequenceDiagram
    participant T1 as 线程1
    participant T2 as 线程2
    participant T3 as 线程3
    participant B as Barrier(3)

    par 并行执行
        T1->>T1: 工作阶段1
        T2->>T2: 工作阶段1
        T3->>T3: 工作阶段1
    end

    T1->>B: wait()
    Note over T1,B: T1 等待
    T2->>B: wait()
    Note over T2,B: T2 等待
    T3->>B: wait()
    Note over T3,B: 全部到达，释放

    par 并行继续
        T1->>T1: 工作阶段2
        T2->>T2: 工作阶段2
        T3->>T3: 工作阶段2
    end
```

---

## 10. Once 单次初始化

Once 保证代码只执行一次，适用于全局初始化。

```mermaid
graph TB
    subgraph "Once 状态"
        INCOMPLETE["Incomplete<br/>未初始化"]
        RUNNING["Running<br/>正在初始化"]
        COMPLETE["Complete<br/>已完成"]
    end

    INCOMPLETE -->|"call_once()"| RUNNING
    RUNNING -->|完成| COMPLETE
    COMPLETE -->|"再次 call_once()"| COMPLETE

    subgraph "OnceLock<T>"
        ONCELOCK["OnceLock<T><br/>• 类型安全的单次初始化<br/>• get_or_init(|| value)<br/>• get() -> Option<&T>"]
    end

    style COMPLETE fill:#c8e6c9
    style ONCELOCK fill:#bbdefb
```

---

## 11. 并发模式总结

```mermaid
mindmap
    root((并发模式))
        共享状态
            Arc + Mutex
            Arc + RwLock
            Atomic 类型
        消息传递
            mpsc 通道
            crossbeam 通道
        同步原语
            Barrier 屏障
            Condvar 条件变量
            Once 单次初始化
        无锁编程
            Atomic 操作
            CAS 循环
```

### 选择指南

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 简单计数器 | `AtomicUsize` | 无锁，高性能 |
| 共享配置 (只读) | `Arc<T>` | 无需锁 |
| 共享可变状态 | `Arc<Mutex<T>>` | 简单直接 |
| 读多写少 | `Arc<RwLock<T>>` | 允许并发读 |
| 生产者-消费者 | `mpsc::channel` | 解耦，避免锁 |
| 多生产者多消费者 | `crossbeam::channel` | 标准库不支持 |
| 全局单例 | `OnceLock<T>` | 线程安全初始化 |
| 阶段同步 | `Barrier` | 等待所有线程 |

```mermaid
flowchart TD
    START[并发需求] --> Q1{需要共享数据?}

    Q1 -->|否| CHANNEL[使用 Channel<br/>消息传递]
    Q1 -->|是| Q2{需要修改?}

    Q2 -->|否| ARC[Arc<T>]
    Q2 -->|是| Q3{数据简单?}

    Q3 -->|是| ATOMIC[Atomic 类型]
    Q3 -->|否| Q4{读多写少?}

    Q4 -->|是| RWLOCK[Arc<RwLock<T>>]
    Q4 -->|否| MUTEX[Arc<Mutex<T>>]

    style CHANNEL fill:#c8e6c9
    style ARC fill:#bbdefb
    style ATOMIC fill:#fff9c4
    style MUTEX fill:#e1bee7
```
