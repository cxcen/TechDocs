# Redis深入分析 - 第1部分：核心架构与数据结构

## 目录
- [1. Redis简介](#1-redis简介)
- [2. 核心架构设计](#2-核心架构设计)
- [3. 底层数据结构](#3-底层数据结构)
- [4. 对象系统](#4-对象系统)

---

## 1. Redis简介

### 1.1 什么是Redis？

Redis（Remote Dictionary Server）是一个开源的内存数据结构存储系统，可以用作：
- **数据库**：持久化的键值存储
- **缓存**：高速数据访问层
- **消息队列**：发布/订阅、流处理
- **实时分析引擎**：计数器、排行榜、地理位置服务

### 1.2 核心特性

```mermaid
graph TB
    A[Redis核心特性] --> B[内存存储]
    A --> C[丰富数据类型]
    A --> D[持久化机制]
    A --> E[高可用性]
    A --> F[可扩展性]

    B --> B1[亚毫秒级延迟]
    B --> B2[高吞吐量]

    C --> C1[String/List/Set/Hash]
    C --> C2[Sorted Set/Stream]
    C --> C3[JSON/时间序列]
    C --> C4[向量搜索]

    D --> D1[RDB快照]
    D --> D2[AOF日志]

    E --> E1[主从复制]
    E --> E2[哨兵Sentinel]
    E --> E3[集群Cluster]

    F --> F1[模块化扩展]
    F --> F2[Lua脚本]
    F --> F3[Redis函数]

    style A fill:#f9f,stroke:#333,stroke-width:4px
```

### 1.3 应用场景

| 场景 | 使用方式 | Redis数据类型 |
|------|----------|---------------|
| 会话存储 | 用户登录状态、购物车 | String, Hash |
| 缓存层 | 数据库查询结果缓存 | String, Hash |
| 排行榜 | 游戏分数、热门文章 | Sorted Set |
| 计数器 | 点赞数、浏览量 | String (INCR) |
| 实时分析 | 在线用户统计 | HyperLogLog, Bitmap |
| 消息队列 | 任务队列、事件流 | List, Stream |
| 分布式锁 | 资源互斥访问 | String (SET NX EX) |
| 地理位置 | 附近的人、配送范围 | Geo |
| 实时推荐 | 协同过滤、相似度计算 | Set, Sorted Set |

---

## 2. 核心架构设计

### 2.1 整体架构

```mermaid
graph TB
    subgraph "客户端层"
        Client1[客户端1]
        Client2[客户端2]
        Client3[客户端N]
    end

    subgraph "网络层"
        IOThread1[IO线程1]
        IOThread2[IO线程2]
        IOThreadN[IO线程N]
        MainThread[主线程]
    end

    subgraph "命令处理层"
        Parser[命令解析器]
        Lookup[命令查找]
        ACL[ACL权限检查]
        Executor[命令执行器]
    end

    subgraph "数据层"
        DB[(数据库数组)]
        KVStore[KVStore键空间]
        Expires[过期字典]
        SubExpires[子键过期]
    end

    subgraph "持久化层"
        RDB[RDB快照]
        AOF[AOF日志]
    end

    subgraph "复制层"
        Master[主节点]
        Replica1[从节点1]
        Replica2[从节点2]
    end

    Client1 & Client2 & Client3 --> IOThread1 & IOThread2 & IOThreadN
    IOThread1 & IOThread2 & IOThreadN --> MainThread
    MainThread --> Parser
    Parser --> Lookup
    Lookup --> ACL
    ACL --> Executor
    Executor --> DB
    DB --> KVStore
    DB --> Expires
    DB --> SubExpires
    Executor --> RDB
    Executor --> AOF
    Master -.复制.-> Replica1
    Master -.复制.-> Replica2

    style MainThread fill:#ffcccc
    style DB fill:#ccffcc
```

### 2.2 事件驱动模型

Redis采用单线程事件驱动模型处理命令，通过多路复用实现高并发。

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant EventLoop as 事件循环
    participant FileEvent as 文件事件
    participant TimeEvent as 时间事件
    participant Command as 命令处理
    participant DB as 数据库

    Client->>EventLoop: 发送命令
    EventLoop->>FileEvent: 检测可读事件
    FileEvent->>Command: 读取命令
    Command->>Command: 解析命令
    Command->>Command: 查找命令表
    Command->>Command: ACL检查
    Command->>DB: 执行命令
    DB-->>Command: 返回结果
    Command->>FileEvent: 写入回复缓冲区
    FileEvent->>Client: 发送响应

    EventLoop->>TimeEvent: 检查定时任务
    TimeEvent->>TimeEvent: serverCron执行
    Note over TimeEvent: 过期键清理<br/>AOF重写<br/>主从同步<br/>统计信息更新
```

### 2.3 内存布局

```mermaid
graph LR
    subgraph "Redis内存分配"
        A[总内存] --> B[数据内存]
        A --> C[复制缓冲区]
        A --> D[客户端缓冲区]
        A --> E[AOF缓冲区]
        A --> F[内部开销]

        B --> B1[键空间]
        B --> B2[过期字典]
        B --> B3[对象元数据]

        F --> F1[dict结构]
        F --> F2[ziplist/listpack]
        F --> F3[skiplist]
        F --> F4[intset]
    end

    style A fill:#ffcccc
    style B fill:#ccffcc
```

---

## 3. 底层数据结构

### 3.1 简单动态字符串 (SDS)

Redis自己实现的字符串结构，解决C字符串的问题。

```c
// SDS结构定义（简化版）
struct sdshdr {
    uint32_t len;        // 已使用长度
    uint32_t alloc;      // 分配的总长度
    unsigned char flags; // 标志位
    char buf[];          // 实际存储字符串的柔性数组
};
```

**SDS的优势：**

```mermaid
graph TB
    A[SDS优势] --> B[O1复杂度获取长度]
    A --> C[防止缓冲区溢出]
    A --> D[减少内存重分配]
    A --> E[二进制安全]
    A --> F[兼容C字符串]

    D --> D1[空间预分配]
    D --> D2[惰性空间释放]

    style A fill:#f9f
```

### 3.2 字典 (Dict)

Redis的核心数据结构，用于实现数据库键空间、哈希对象等。

```mermaid
graph TB
    subgraph "字典结构"
        Dict[dict字典] --> HT0[ht0哈希表]
        Dict --> HT1[ht1哈希表]
        Dict --> Type[type类型特定函数]
        Dict --> Rehashidx[rehashidx渐进式rehash索引]

        HT0 --> Table0[dictEntry数组]
        Table0 --> Entry1[dictEntry]
        Table0 --> Entry2[dictEntry]
        Table0 --> Entry3[dictEntry]

        Entry1 --> Key1[key键]
        Entry1 --> Val1[value值]
        Entry1 --> Next1[next下一个节点]
    end

    style Dict fill:#ffcccc
    style HT0 fill:#ccffcc
```

**渐进式Rehash过程：**

```mermaid
sequenceDiagram
    participant Client as 客户端操作
    participant Dict as 字典
    participant HT0 as ht[0]旧表
    participant HT1 as ht[1]新表

    Note over Dict: rehashidx = 0<br/>开始rehash

    Client->>Dict: 添加/删除/查找操作
    Dict->>Dict: 执行一步rehash
    Dict->>HT0: 迁移rehashidx位置的桶
    HT0->>HT1: 将键值对移动到新表
    Dict->>Dict: rehashidx++

    Note over Dict: 每次操作迁移一个桶

    Client->>Dict: 继续操作...
    Dict->>Dict: 持续渐进式rehash

    Note over Dict,HT1: rehashidx = -1<br/>rehash完成
    Dict->>Dict: 交换ht[0]和ht[1]
    Dict->>HT0: 释放旧表
```

### 3.3 跳跃表 (Skip List)

有序集合的底层实现之一，支持平均O(log N)的查找性能。

```mermaid
graph LR
    subgraph "跳跃表结构"
        Header[头节点<br/>level:32]

        Header -->|level 3| N3[节点<br/>score:3]
        N3 -->|level 3| N7[节点<br/>score:7]
        N7 -->|level 3| Tail[NULL]

        Header -->|level 2| N3
        N3 -->|level 2| N5[节点<br/>score:5]
        N5 -->|level 2| N7
        N7 -->|level 2| Tail

        Header -->|level 1| N1[节点<br/>score:1]
        N1 -->|level 1| N3
        N3 -->|level 1| N5
        N5 -->|level 1| N7
        N7 -->|level 1| N9[节点<br/>score:9]
        N9 -->|level 1| Tail

        N1 -.backward.-> Header
        N3 -.backward.-> N1
        N5 -.backward.-> N3
        N7 -.backward.-> N5
        N9 -.backward.-> N7
    end

    style Header fill:#ffcccc
    style N5 fill:#ccffcc
```

**跳跃表查找过程（查找score=5）：**

```mermaid
sequenceDiagram
    participant Client as 查找操作
    participant Header as 头节点
    participant Level3 as 第3层
    participant Level2 as 第2层
    participant Level1 as 第1层
    participant Target as 目标节点

    Client->>Header: 查找score=5
    Header->>Level3: 从最高层开始
    Level3->>Level3: 3 < 5，前进
    Level3->>Level3: 7 > 5，下降

    Level3->>Level2: 下降到第2层
    Level2->>Level2: 在节点3
    Level2->>Level2: 5 == 5，找到！

    Level2->>Target: 返回节点
    Target-->>Client: score=5的节点
```

### 3.4 整数集合 (IntSet)

集合的底层实现之一，只包含整数值的集合会使用这个结构。

```c
typedef struct intset {
    uint32_t encoding;  // 编码方式：int16/int32/int64
    uint32_t length;    // 元素数量
    int8_t contents[];  // 实际存储数组（类型由encoding决定）
} intset;
```

**整数集合升级过程：**

```mermaid
graph TB
    A[初始int16_t编码] -->|添加65535| B{判断是否超出范围}
    B -->|是| C[升级到int32_t]
    B -->|否| D[直接插入]

    C --> C1[分配新空间]
    C1 --> C2[将所有元素转换为int32_t]
    C2 --> C3[插入新元素]
    C3 --> C4[更新encoding]

    style A fill:#ccffcc
    style C fill:#ffcccc
```

### 3.5 压缩列表 (Listpack)

Listpack是ziplist的改进版本，用于编码小的列表、哈希、有序集合。

```mermaid
graph LR
    subgraph "Listpack结构"
        Header[Total Bytes<br/>4字节] --> NumElem[元素数量<br/>2字节]
        NumElem --> Entry1[Entry 1]
        Entry1 --> Entry2[Entry 2]
        Entry2 --> EntryN[Entry N]
        EntryN --> EOF[EOF<br/>0xFF]

        subgraph "Entry结构"
            Encoding[编码类型] --> Data[数据]
            Data --> BackLen[反向长度]
        end
    end

    style Header fill:#ffcccc
    style Entry1 fill:#ccffcc
```

---

## 4. 对象系统

### 4.1 RedisObject结构

Redis使用对象来表示数据库中的键和值。

```c
typedef struct redisObject {
    unsigned type:4;        // 类型（string/list/set/zset/hash）
    unsigned encoding:4;    // 编码方式
    unsigned lru:24;        // LRU时间或LFU数据
    int refcount;           // 引用计数
    void *ptr;              // 指向实际数据的指针
} robj;
```

### 4.2 类型与编码

```mermaid
graph TB
    subgraph "Redis对象类型与编码"
        String[String字符串] --> S1[int整数]
        String --> S2[embstr短字符串]
        String --> S3[raw长字符串]

        List[List列表] --> L1[quicklist]
        List --> L2[listpack]

        Set[Set集合] --> Set1[intset整数集合]
        Set --> Set2[hashtable哈希表]
        Set --> Set3[listpack]

        Hash[Hash哈希] --> H1[listpack]
        Hash --> H2[hashtable]
        Hash --> H3[listpackEx带过期]

        ZSet[Sorted Set有序集合] --> Z1[listpack]
        ZSet --> Z2[skiplist+dict]

        Stream[Stream流] --> St1[rax+listpack]
    end

    style String fill:#ffcccc
    style List fill:#ccffcc
    style Set fill:#ffffcc
    style Hash fill:#ffccff
    style ZSet fill:#ccffff
    style Stream fill:#ffcccc
```

### 4.3 编码转换策略

Redis会根据数据大小和元素数量自动进行编码转换以优化内存使用。

```mermaid
stateDiagram-v2
    [*] --> Listpack: 创建小哈希

    Listpack --> Hashtable: 元素数量>512 OR<br/>单个元素>64字节

    note right of Listpack
        内存紧凑
        顺序访问快
    end note

    note right of Hashtable
        随机访问O(1)
        内存占用较大
    end note

    Hashtable --> [*]
```

### 4.4 KVObj（键值对象）

Redis 8.0引入的优化，将键和值嵌入同一个对象中。

```c
// kvobj结构示例（概念性）
typedef struct redisObject kvobj;

// kvobj内存布局示例：
// +----------------+--------------+------------------+------------------+
// | redisObject    | 过期时间     | key字符串        | value指针        |
// | 16字节         | 8字节(可选)  | sds             | 8字节            |
// +----------------+--------------+------------------+------------------+
```

**kvobj的优势：**

```mermaid
graph LR
    A[kvobj优势] --> B[减少内存碎片]
    A --> C[提高缓存局部性]
    A --> D[减少指针跳转]
    A --> E[支持嵌入过期时间]

    B --> B1[键值对在同一内存块]
    C --> C1[CPU缓存命中率提升]
    D --> D1[访问速度更快]
    E --> E1[支持Hash字段过期]

    style A fill:#f9f
```

---

## 小结

第1部分介绍了Redis的核心架构和底层数据结构：

1. **整体架构**：单线程事件驱动模型 + IO多线程
2. **数据结构**：SDS、Dict、SkipList、IntSet、Listpack
3. **对象系统**：统一的对象封装和编码转换机制
4. **优化策略**：根据数据特征自动选择最优编码

下一部分将深入探讨**数据类型实现和命令处理流程**。
