# Tokio Time Driver 时间驱动架构分析

## 目录

1. [时间驱动概述](#1-时间驱动概述)
2. [分层时间轮算法](#2-分层时间轮算法)
3. [TimerEntry 状态机](#3-timerentry-状态机)
4. [Sleep 和 Timeout](#4-sleep-和-timeout)
5. [Time Driver 实现](#5-time-driver-实现)
6. [性能优化](#6-性能优化)

---

## 1. 时间驱动概述

Tokio Time Driver 基于分层时间轮 (Hierarchical Timing Wheel) 算法，提供高效的定时器管理。

### 1.1 设计目标

```mermaid
mindmap
    root((Time Driver))
        高性能
            O(1) 插入
            O(1) 取消
            批量处理
        精确性
            毫秒级精度
            年级别范围
        灵活性
            Sleep
            Timeout
            Interval
```

### 1.2 核心组件

```mermaid
graph TB
    subgraph UserAPI["用户 API"]
        Sleep["tokio::time::sleep"]
        Timeout["tokio::time::timeout"]
        Interval["tokio::time::interval"]
    end

    subgraph Internal["内部实现"]
        TimerEntry["TimerEntry<br/>定时器条目"]
        TimeSource["TimeSource<br/>时间转换"]
    end

    subgraph Driver["时间驱动"]
        Wheel["Wheel<br/>时间轮"]
        Level["Level[6]<br/>6层级"]
        Handle["Handle<br/>驱动句柄"]
    end

    UserAPI --> Internal
    Internal --> Driver

    style UserAPI fill:#e3f2fd
    style Internal fill:#fff3e0
    style Driver fill:#e8f5e9
```

---

## 2. 分层时间轮算法

### 2.1 时间轮结构

时间轮采用 6 层设计，每层 64 个槽位：

```mermaid
graph TB
    subgraph Wheel["分层时间轮"]
        L0["Level 0<br/>64 × 1ms = 64ms"]
        L1["Level 1<br/>64 × 64ms ≈ 4s"]
        L2["Level 2<br/>64 × 4s ≈ 4min"]
        L3["Level 3<br/>64 × 4min ≈ 4h"]
        L4["Level 4<br/>64 × 4h ≈ 12d"]
        L5["Level 5<br/>64 × 12d ≈ 2y"]
    end

    L0 --> L1
    L1 --> L2
    L2 --> L3
    L3 --> L4
    L4 --> L5

    style L0 fill:#c8e6c9
    style L1 fill:#dcedc8
    style L2 fill:#f0f4c3
    style L3 fill:#fff9c4
    style L4 fill:#ffecb3
    style L5 fill:#ffe0b2
```

### 2.2 核心数据结构

```rust
// 文件: runtime/time/wheel/mod.rs
pub(crate) struct Wheel {
    /// 当前时间 (毫秒)
    elapsed: u64,

    /// 6 层级
    levels: Box<[Level; NUM_LEVELS]>,

    /// 待触发的定时器
    pending: EntryList,
}

// 文件: runtime/time/wheel/level.rs
pub(crate) struct Level {
    /// 层级号 (0-5)
    level: usize,

    /// 占用位图 (64位，标记哪些槽有定时器)
    occupied: u64,

    /// 64 个槽位
    slot: [EntryList; LEVEL_MULT],
}
```

### 2.3 Level 选择算法

```rust
fn level_for(elapsed: u64, when: u64) -> usize {
    const SLOT_MASK: u64 = (1 << 6) - 1;

    // XOR 找出不同的位
    let mut masked = elapsed ^ when | SLOT_MASK;

    if masked >= MAX_DURATION {
        masked = MAX_DURATION - 1;
    }

    // 计算最高有效位
    let leading_zeros = masked.leading_zeros() as usize;
    let significant = 63 - leading_zeros;

    // 除以 6 得到 Level
    significant / NUM_LEVELS
}
```

### 2.4 Level 选择可视化

```mermaid
flowchart TD
    A["计算 elapsed XOR when"] --> B["找最高有效位"]
    B --> C["significant / 6"]
    C --> D{时间差范围?}

    D -->|"< 64ms"| E["Level 0"]
    D -->|"64ms - 4s"| F["Level 1"]
    D -->|"4s - 4min"| G["Level 2"]
    D -->|"4min - 4h"| H["Level 3"]
    D -->|"4h - 12d"| I["Level 4"]
    D -->|"> 12d"| J["Level 5"]

    style E fill:#c8e6c9
    style J fill:#ffe0b2
```

### 2.5 Occupied 位图优化

```mermaid
graph LR
    subgraph Bitmap["occupied 位图 (u64)"]
        B0["bit 0"]
        B1["bit 1"]
        B2["..."]
        B63["bit 63"]
    end

    subgraph Operations["操作"]
        Find["找下一个非空槽<br/>trailing_zeros()"]
        Set["添加定时器<br/>occupied |= bit"]
        Clear["移除定时器<br/>occupied ^= bit"]
    end

    Bitmap --> Operations
```

```rust
fn next_occupied_slot(&self, now: u64) -> Option<usize> {
    if self.occupied == 0 {
        return None;  // 全空
    }

    let now_slot = (now / slot_range(self.level)) as usize;
    let occupied = self.occupied.rotate_right(now_slot as u32);

    // O(1) 找到下一个非空槽
    let zeros = occupied.trailing_zeros() as usize;
    let slot = (zeros + now_slot) % LEVEL_MULT;

    Some(slot)
}
```

---

## 3. TimerEntry 状态机

### 3.1 StateCell 结构

```rust
// 文件: runtime/time/entry.rs
pub(super) struct StateCell {
    /// 状态: 截止时间或特殊值
    state: AtomicU64,

    /// 触发结果
    result: UnsafeCell<TimerResult>,

    /// 注册的 Waker
    waker: AtomicWaker,
}

// 特殊状态值
const STATE_DEREGISTERED: u64 = u64::MAX;      // 已注销
const STATE_PENDING_FIRE: u64 = u64::MAX - 1;  // 待触发
```

### 3.2 状态转换图

```mermaid
stateDiagram-v2
    [*] --> Registered: 创建定时器<br/>state = deadline

    Registered --> Registered: extend_expiration()<br/>延后截止时间

    Registered --> PendingFire: mark_pending()<br/>定时器到期

    PendingFire --> Deregistered: fire()<br/>触发完成

    Registered --> Deregistered: cancel()<br/>取消

    Deregistered --> [*]: 释放

    note right of Registered
        state = 有效时间戳
        0 ~ (u64::MAX - 2)
    end note

    note right of PendingFire
        state = u64::MAX - 1
        等待 fire() 调用
    end note

    note right of Deregistered
        state = u64::MAX
        已触发或取消
    end note
```

### 3.3 关键方法

```mermaid
flowchart TD
    subgraph mark_pending["mark_pending(not_after)"]
        MP1["加载当前状态"] --> MP2{已触发?}
        MP2 -->|是| MP3["panic!"]
        MP2 -->|否| MP4{state > not_after?}
        MP4 -->|是| MP5["返回 Err(state)<br/>需要重新调度"]
        MP4 -->|否| MP6["CAS -> PENDING_FIRE"]
        MP6 --> MP7["返回 Ok"]
    end

    subgraph fire["fire(result)"]
        F1["检查状态"] --> F2{已 DEREGISTERED?}
        F2 -->|是| F3["返回 None"]
        F2 -->|否| F4["写入 result"]
        F4 --> F5["state = DEREGISTERED"]
        F5 --> F6["取出 Waker"]
        F6 --> F7["返回 Some(waker)"]
    end
```

### 3.4 TimerEntry 结构

```rust
pub(crate) struct TimerEntry {
    /// 运行时句柄
    driver: scheduler::Handle,

    /// 共享内部状态
    inner: Option<TimerShared>,

    /// 截止时刻
    deadline: Instant,

    /// 是否已注册
    registered: bool,
}
```

---

## 4. Sleep 和 Timeout

### 4.1 Sleep 结构

```rust
// 文件: time/sleep.rs
pub struct Sleep {
    inner: Inner,

    /// 底层定时器条目
    #[pin]
    entry: TimerEntry,
}
```

### 4.2 Sleep 执行流程

```mermaid
sequenceDiagram
    participant User as 用户代码
    participant Sleep as Sleep
    participant Entry as TimerEntry
    participant Wheel as 时间轮
    participant Driver as Time Driver

    User->>Sleep: sleep(duration).await
    Sleep->>Entry: 创建 TimerEntry
    Note over Entry: registered = false

    User->>Sleep: poll() 首次
    Sleep->>Entry: reset(deadline, true)
    Entry->>Wheel: 插入定时器

    Wheel->>Wheel: 计算 Level 和 Slot
    Wheel->>Entry: 链入对应槽位

    Sleep-->>User: Poll::Pending

    Note over Driver: park_timeout()

    Driver->>Wheel: poll(now)
    Wheel->>Entry: 到期检查
    Entry->>Entry: mark_pending()
    Entry->>Entry: fire(Ok(()))

    Entry->>Sleep: wake()
    Sleep->>User: 任务重新调度

    User->>Sleep: poll() 再次
    Sleep->>Entry: state.poll()
    Entry-->>Sleep: Poll::Ready
    Sleep-->>User: 返回
```

### 4.3 Timeout 结构

```rust
// 文件: time/timeout.rs
pub struct Timeout<T> {
    /// 被包装的 Future
    #[pin]
    value: T,

    /// 时间限制
    #[pin]
    delay: Sleep,
}
```

### 4.4 Timeout 执行流程

```mermaid
flowchart TD
    A["Timeout::poll()"] --> B["记录 budget"]

    B --> C["poll(value)"]
    C --> D{结果?}

    D -->|Ready| E["返回 Ok(output)"]

    D -->|Pending| F["poll(delay)"]
    F --> G{超时?}

    G -->|是| H["返回 Err(Elapsed)"]
    G -->|否| I["返回 Pending"]

    subgraph BudgetHandling["预算处理"]
        J["如果 value 耗尽 budget"]
        K["用无约束预算 poll delay"]
    end

    F --> BudgetHandling

    style E fill:#c8e6c9
    style H fill:#ffcdd2
```

---

## 5. Time Driver 实现

### 5.1 Driver 结构

```rust
// 文件: runtime/time/mod.rs
pub(crate) struct Driver {
    /// I/O 驱动栈
    park: IoStack,
}

struct Inner {
    /// 受锁保护的状态
    state: Mutex<InnerState>,

    /// 快速关闭检查
    is_shutdown: AtomicBool,
}

struct InnerState {
    /// 下一次唤醒时刻
    next_wake: Option<NonZeroU64>,

    /// 时间轮
    wheel: wheel::Wheel,
}
```

### 5.2 park_internal 流程

```mermaid
flowchart TD
    A["park_internal(limit)"] --> B["获取锁"]

    B --> C["next_expiration_time()"]
    C --> D{有定时器?}

    D -->|是| E["计算等待时间"]
    E --> F["min(到期时间, limit)"]
    F --> G["park_thread_timeout()"]

    D -->|否| H{有 limit?}
    H -->|是| I["park_thread_timeout(limit)"]
    H -->|否| J["park_thread()"]

    G --> K["被唤醒"]
    I --> K
    J --> K

    K --> L["process()"]
    L --> M["处理到期定时器"]
```

### 5.3 process 处理流程

```mermaid
sequenceDiagram
    participant Driver as Driver
    participant Lock as Mutex
    participant Wheel as Wheel
    participant Entry as TimerEntry
    participant WakeList as WakeList

    Driver->>Lock: 获取锁
    Driver->>Wheel: poll(now)

    loop 遍历到期定时器
        Wheel->>Entry: 获取到期条目
        Entry->>Entry: fire(Ok(()))
        Entry-->>WakeList: 返回 Waker

        alt WakeList 满
            Driver->>Lock: 释放锁
            WakeList->>WakeList: wake_all()
            Driver->>Lock: 重新获取锁
        end
    end

    Driver->>Wheel: 计算 next_wake
    Driver->>Lock: 释放锁
    WakeList->>WakeList: wake_all()
```

### 5.4 reregister 流程

```mermaid
flowchart TD
    A["reregister(entry, deadline)"] --> B["获取锁"]

    B --> C{已注册?}
    C -->|是| D["从时间轮移除"]
    C -->|否| E["继续"]

    D --> E
    E --> F["设置新截止时间"]
    F --> G{驱动关闭?}

    G -->|是| H["标记关闭"]
    G -->|否| I["插入时间轮"]

    I --> J{插入结果?}

    J -->|Ok| K{新时间 < next_wake?}
    J -->|Elapsed| L["立即触发"]

    K -->|是| M["unpark() 唤醒驱动"]
    K -->|否| N["完成"]

    L --> O["在锁外调用 wake()"]
```

---

## 6. 性能优化

### 6.1 批量唤醒

```rust
pub(self) fn process_at_time(&self, mut now: u64) {
    let mut waker_list = WakeList::new();  // 栈上缓冲
    let mut lock = self.inner.lock();

    while let Some(entry) = lock.wheel.poll(now) {
        if let Some(waker) = unsafe { entry.fire(Ok(())) } {
            waker_list.push(waker);

            // 缓冲区满时批量唤醒
            if !waker_list.can_push() {
                drop(lock);
                waker_list.wake_all();
                lock = self.inner.lock();
            }
        }
    }

    drop(lock);
    waker_list.wake_all();
}
```

### 6.2 延迟注册

```mermaid
graph LR
    subgraph LazyRegistration["延迟注册"]
        Create["创建 Sleep"]
        FirstPoll["首次 poll"]
        Register["注册到时间轮"]
    end

    Create -->|"registered = false"| FirstPoll
    FirstPoll -->|"reset(deadline, true)"| Register
```

**优势**:
- 避免未使用的定时器占用资源
- 支持 select! 中被丢弃的分支

### 6.3 级联机制

```mermaid
sequenceDiagram
    participant L5 as Level 5
    participant L4 as Level 4
    participant L0 as Level 0
    participant Task as Task

    Note over L5: 100小时后的定时器

    L5->>L5: 插入 Level 5

    Note over L5,Task: 时间推进...

    L5->>L4: 级联到 Level 4
    Note over L4: 约4小时时

    L4->>L0: 级联到 Level 0
    Note over L0: 最后64ms

    L0->>Task: 触发定时器
```

### 6.4 性能特性

| 操作 | 复杂度 | 说明 |
|-----|--------|------|
| 插入 | O(1) | 直接计算槽位 |
| 取消 | O(1) | 链表移除 |
| 查找下一个 | O(1) | 位图 trailing_zeros |
| 处理到期 | O(m) | m = 到期定时器数 |

---

## 时间精度和范围

### 精度

- **分辨率**: 1 毫秒
- **实际精度**: 取决于操作系统
  - Linux: 1-10 ms
  - Windows: 10-15 ms

### 范围

```
MAX_DURATION = (1 << 36) - 1 毫秒
            ≈ 68,719,476,735 毫秒
            ≈ 2.18 年
```

---

## 总结

Tokio Time Driver 的设计精髓：

1. **分层时间轮**: 6 层 × 64 槽，覆盖毫秒到年
2. **位图优化**: O(1) 查找下一个非空槽
3. **延迟注册**: 减少不必要的开销
4. **批量唤醒**: 减少锁竞争
5. **无锁状态**: CAS 实现并发安全
