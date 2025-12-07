# Rust 集合类型详解

## 1. 集合类型总览

```mermaid
graph TB
    subgraph "std::collections"
        subgraph "序列容器"
            VEC["Vec<T><br/>动态数组"]
            VECDEQUE["VecDeque<T><br/>双端队列"]
            LINKEDLIST["LinkedList<T><br/>双向链表"]
        end

        subgraph "映射容器"
            HASHMAP["HashMap<K,V><br/>哈希映射"]
            BTREEMAP["BTreeMap<K,V><br/>B树映射"]
        end

        subgraph "集合容器"
            HASHSET["HashSet<T><br/>哈希集合"]
            BTREESET["BTreeSet<T><br/>B树集合"]
        end

        subgraph "特殊容器"
            BINARYHEAP["BinaryHeap<T><br/>二叉堆"]
        end
    end

    VEC -->|Deref| SLICE["[T] 切片"]
    STRING -->|Deref| STR["str"]

    style VEC fill:#c8e6c9
    style HASHMAP fill:#bbdefb
    style HASHSET fill:#fff9c4
    style BINARYHEAP fill:#e1bee7
```

### 选择指南

```mermaid
flowchart TD
    START[选择集合类型] --> Q1{需要什么操作?}

    Q1 -->|顺序访问| SEQ{需要两端操作?}
    Q1 -->|键值查找| MAP{需要顺序遍历?}
    Q1 -->|唯一性检查| SET{需要顺序遍历?}
    Q1 -->|优先级队列| HEAP[BinaryHeap]

    SEQ -->|否| VEC[Vec]
    SEQ -->|是| VECDEQUE[VecDeque]

    MAP -->|否| HASHMAP[HashMap]
    MAP -->|是| BTREEMAP[BTreeMap]

    SET -->|否| HASHSET[HashSet]
    SET -->|是| BTREESET[BTreeSet]

    style VEC fill:#c8e6c9
    style HASHMAP fill:#bbdefb
    style BTREEMAP fill:#fff9c4
```

---

## 2. Vec<T> 动态数组

Vec 是最常用的集合类型，提供连续内存的动态数组。

### 内存布局

```mermaid
graph TB
    subgraph "Vec<T> 结构"
        STACK["栈上 (24 字节)<br/>├─ ptr: *mut T<br/>├─ len: usize<br/>└─ capacity: usize"]

        HEAP["堆上<br/>┌─────────────────────┐<br/>│ T │ T │ T │ _ │ _ │<br/>└─────────────────────┘<br/>  0   1   2   3   4<br/>      len=3  cap=5"]
    end

    STACK -->|ptr 指向| HEAP

    style STACK fill:#c8e6c9
    style HEAP fill:#bbdefb
```

### 容量增长策略

```mermaid
sequenceDiagram
    participant User as 用户代码
    participant Vec as Vec
    participant Alloc as 分配器

    User->>Vec: push(item)
    Vec->>Vec: 检查 len < capacity?

    alt 有空间
        Vec->>Vec: 写入并增加 len
    else 需要扩容
        Vec->>Vec: 计算新容量 (通常 2x)
        Vec->>Alloc: 分配新内存
        Alloc-->>Vec: 新地址
        Vec->>Vec: 复制旧数据
        Vec->>Alloc: 释放旧内存
        Vec->>Vec: 写入新元素
    end

    Vec-->>User: 完成
```

### 常用方法

```mermaid
graph TB
    subgraph "创建"
        NEW["Vec::new()<br/>空向量"]
        WITH_CAP["Vec::with_capacity(n)<br/>预分配"]
        VEC_MACRO["vec![1, 2, 3]<br/>宏创建"]
        FROM["Vec::from([1,2,3])<br/>从数组"]
    end

    subgraph "添加/删除"
        PUSH["push(item)<br/>尾部添加"]
        POP["pop() -> Option<T><br/>尾部移除"]
        INSERT["insert(i, item)<br/>指定位置插入"]
        REMOVE["remove(i) -> T<br/>指定位置删除"]
        CLEAR["clear()<br/>清空"]
    end

    subgraph "访问"
        INDEX["v[i]<br/>索引访问"]
        GET["get(i) -> Option<&T><br/>安全访问"]
        FIRST["first()/last()<br/>首尾元素"]
        ITER["iter()/iter_mut()<br/>迭代器"]
    end

    subgraph "容量"
        LEN["len()<br/>当前长度"]
        CAP["capacity()<br/>当前容量"]
        RESERVE["reserve(n)<br/>预留空间"]
        SHRINK["shrink_to_fit()<br/>收缩容量"]
    end

    style NEW fill:#c8e6c9
    style PUSH fill:#bbdefb
    style INDEX fill:#fff9c4
    style LEN fill:#e1bee7
```

### 性能特性

| 操作 | 平均复杂度 | 最坏复杂度 | 说明 |
|------|-----------|-----------|------|
| push | O(1)* | O(n) | 摊还 O(1)，扩容时 O(n) |
| pop | O(1) | O(1) | |
| insert | O(n) | O(n) | 需要移动元素 |
| remove | O(n) | O(n) | 需要移动元素 |
| get/index | O(1) | O(1) | |
| contains | O(n) | O(n) | 线性搜索 |

---

## 3. String 字符串

String 是 UTF-8 编码的动态字符串，本质是 `Vec<u8>` 的包装。

```mermaid
graph TB
    subgraph "String vs &str"
        STRING["String<br/>• 拥有数据<br/>• 可变<br/>• 堆分配<br/>• 24 字节"]
        STR["&str<br/>• 借用数据<br/>• 不可变<br/>• 切片视图<br/>• 16 字节"]
    end

    STRING -->|Deref| STR
    STRING -->|"as_str()"| STR
    STR -->|"to_string()"| STRING
    STR -->|"to_owned()"| STRING
    STR -->|"String::from()"| STRING

    style STRING fill:#c8e6c9
    style STR fill:#bbdefb
```

### 字符串索引

```mermaid
graph TB
    subgraph "UTF-8 编码"
        BYTES["字节视图<br/>H  e  l  l  o  ,     世     界<br/>48 65 6c 6c 6f 2c 20 e4b896 e7958c"]
        CHARS["字符视图<br/>H e l l o ,   世 界<br/>0 1 2 3 4 5 6 7  8"]
    end

    subgraph "访问方式"
        NO_INDEX["❌ s[0] 不支持直接索引"]
        SLICE["✓ &s[0..5] 字节切片"]
        CHARS_ITER["✓ s.chars() 字符迭代"]
        BYTES_ITER["✓ s.bytes() 字节迭代"]
    end

    BYTES --> SLICE
    CHARS --> CHARS_ITER

    style NO_INDEX fill:#ffccbc
    style SLICE fill:#c8e6c9
    style CHARS_ITER fill:#c8e6c9
```

---

## 4. HashMap<K, V>

HashMap 提供 O(1) 平均复杂度的键值存储。

### 内部结构

```mermaid
graph TB
    subgraph "HashMap 结构"
        CTRL["控制字节数组<br/>用于快速查找"]
        BUCKETS["桶数组<br/>存储 (K, V) 对"]
    end

    subgraph "哈希过程"
        KEY["键 K"] --> HASH["hash(K)"]
        HASH --> INDEX["计算桶索引"]
        INDEX --> PROBE["线性探测"]
        PROBE --> FOUND["找到/插入"]
    end

    CTRL --> PROBE
    BUCKETS --> FOUND

    style CTRL fill:#c8e6c9
    style BUCKETS fill:#bbdefb
```

### 使用要求

```mermaid
graph TD
    subgraph "K 的要求"
        HASH_TRAIT["实现 Hash trait"]
        EQ_TRAIT["实现 Eq trait"]
        CONSISTENT["Hash 和 Eq 一致性:<br/>a == b 则 hash(a) == hash(b)"]
    end

    subgraph "常见键类型"
        GOOD["✓ String, &str<br/>✓ 整数类型<br/>✓ PathBuf<br/>✓ 元组 (若元素满足)"]
        BAD["✗ f32, f64 (NaN 问题)<br/>✗ 自定义类型 (需 derive)"]
    end

    HASH_TRAIT --> CONSISTENT
    EQ_TRAIT --> CONSISTENT
    CONSISTENT --> GOOD
    CONSISTENT --> BAD

    style GOOD fill:#c8e6c9
    style BAD fill:#ffccbc
```

### Entry API

```mermaid
flowchart TD
    START["map.entry(key)"] --> CHECK{键存在?}

    CHECK -->|是| OCCUPIED["OccupiedEntry<br/>• get(): &V<br/>• get_mut(): &mut V<br/>• insert(v): V<br/>• remove(): V"]

    CHECK -->|否| VACANT["VacantEntry<br/>• insert(v): &mut V"]

    subgraph "常用模式"
        OR_INSERT["entry(k).or_insert(default)"]
        OR_INSERT_WITH["entry(k).or_insert_with(|| compute())"]
        OR_DEFAULT["entry(k).or_default()"]
        AND_MODIFY["entry(k).and_modify(|v| *v += 1).or_insert(1)"]
    end

    VACANT --> OR_INSERT
    OCCUPIED --> AND_MODIFY

    style OCCUPIED fill:#c8e6c9
    style VACANT fill:#bbdefb
```

### 性能特性

| 操作 | 平均复杂度 | 最坏复杂度 |
|------|-----------|-----------|
| get | O(1) | O(n) |
| insert | O(1) | O(n) |
| remove | O(1) | O(n) |
| contains_key | O(1) | O(n) |
| 遍历 | O(capacity) | O(capacity) |

---

## 5. BTreeMap<K, V>

BTreeMap 使用 B 树实现有序映射。

```mermaid
graph TB
    subgraph "B树 vs 哈希"
        BTREE["BTreeMap<br/>• 有序存储<br/>• O(log n) 操作<br/>• 范围查询高效<br/>• K: Ord"]
        HASH["HashMap<br/>• 无序存储<br/>• O(1) 操作<br/>• 单键查询高效<br/>• K: Hash + Eq"]
    end

    subgraph "BTreeMap 优势"
        RANGE["范围查询<br/>range(start..end)"]
        ORDER["有序遍历<br/>iter() 按键排序"]
        MINMAX["首尾访问<br/>first_key_value()<br/>last_key_value()"]
    end

    BTREE --> RANGE
    BTREE --> ORDER
    BTREE --> MINMAX

    style BTREE fill:#c8e6c9
    style HASH fill:#bbdefb
```

### B树结构

```mermaid
graph TB
    subgraph "B树节点"
        ROOT["根节点<br/>[30, 60]"]
        LEFT["左子树<br/>[10, 20]"]
        MID["中子树<br/>[40, 50]"]
        RIGHT["右子树<br/>[70, 80, 90]"]
    end

    ROOT --> LEFT
    ROOT --> MID
    ROOT --> RIGHT

    style ROOT fill:#e1bee7
    style LEFT fill:#c8e6c9
    style MID fill:#c8e6c9
    style RIGHT fill:#c8e6c9
```

---

## 6. HashSet<T> 和 BTreeSet<T>

集合类型用于存储唯一值。

```mermaid
graph TB
    subgraph "HashSet<T>"
        HASHSET_IMPL["内部: HashMap<T, ()>"]
        HASHSET_REQ["要求: T: Hash + Eq"]
        HASHSET_PERF["性能: O(1)"]
    end

    subgraph "BTreeSet<T>"
        BTREESET_IMPL["内部: BTreeMap<T, ()>"]
        BTREESET_REQ["要求: T: Ord"]
        BTREESET_PERF["性能: O(log n)"]
    end

    subgraph "集合操作"
        UNION["并集: a.union(&b)"]
        INTER["交集: a.intersection(&b)"]
        DIFF["差集: a.difference(&b)"]
        SYM["对称差: a.symmetric_difference(&b)"]
    end

    HASHSET_IMPL --> UNION
    BTREESET_IMPL --> UNION

    style HASHSET_IMPL fill:#c8e6c9
    style BTREESET_IMPL fill:#bbdefb
```

### 集合运算示意

```mermaid
graph LR
    subgraph "集合 A"
        A1((1))
        A2((2))
        A3((3))
    end

    subgraph "集合 B"
        B2((2))
        B3((3))
        B4((4))
    end

    subgraph "运算结果"
        UNION_R["并集: {1,2,3,4}"]
        INTER_R["交集: {2,3}"]
        DIFF_R["A-B: {1}"]
        SYM_R["对称差: {1,4}"]
    end

    A1 --> UNION_R
    A2 --> INTER_R
    A1 --> DIFF_R
    B4 --> SYM_R

    style UNION_R fill:#c8e6c9
    style INTER_R fill:#bbdefb
    style DIFF_R fill:#fff9c4
    style SYM_R fill:#e1bee7
```

---

## 7. VecDeque<T> 双端队列

```mermaid
graph TB
    subgraph "VecDeque 结构"
        RING["环形缓冲区<br/>┌───────────────────┐<br/>│ _ │ a │ b │ c │ _ │<br/>└───────────────────┘<br/>      ^head   ^tail"]
    end

    subgraph "双端操作"
        FRONT["push_front / pop_front<br/>O(1)"]
        BACK["push_back / pop_back<br/>O(1)"]
    end

    subgraph "vs Vec"
        VEC_GOOD["Vec 优势:<br/>• 内存更紧凑<br/>• 缓存友好"]
        DEQUE_GOOD["VecDeque 优势:<br/>• 双端 O(1)<br/>• 适合队列场景"]
    end

    RING --> FRONT
    RING --> BACK

    style RING fill:#c8e6c9
    style FRONT fill:#bbdefb
    style BACK fill:#bbdefb
```

---

## 8. BinaryHeap<T> 二叉堆

BinaryHeap 是最大堆实现，适用于优先队列。

```mermaid
graph TB
    subgraph "堆结构"
        ROOT["90 (最大)"]
        L1["80"]
        R1["70"]
        L2["60"]
        R2["50"]
        L3["40"]
        R3["30"]

        ROOT --> L1
        ROOT --> R1
        L1 --> L2
        L1 --> R2
        R1 --> L3
        R1 --> R3
    end

    subgraph "主要操作"
        PUSH["push(item)<br/>O(log n)"]
        POP["pop()<br/>O(log n) 获取最大值"]
        PEEK["peek()<br/>O(1) 查看最大值"]
    end

    ROOT --> PEEK

    style ROOT fill:#e1bee7
    style PUSH fill:#c8e6c9
    style POP fill:#bbdefb
```

### 最小堆实现

```rust
use std::cmp::Reverse;
use std::collections::BinaryHeap;

// 使用 Reverse 包装实现最小堆
let mut min_heap = BinaryHeap::new();
min_heap.push(Reverse(5));
min_heap.push(Reverse(1));
min_heap.push(Reverse(3));

assert_eq!(min_heap.pop(), Some(Reverse(1))); // 最小值
```

---

## 9. 集合性能对比

```mermaid
graph TB
    subgraph "时间复杂度对比"
        subgraph "序列"
            VEC_T["Vec<br/>随机访问 O(1)<br/>尾部操作 O(1)<br/>中间操作 O(n)"]
            DEQUE_T["VecDeque<br/>随机访问 O(1)<br/>两端操作 O(1)"]
            LIST_T["LinkedList<br/>随机访问 O(n)<br/>已知位置操作 O(1)"]
        end

        subgraph "映射"
            HASH_T["HashMap<br/>查找 O(1)<br/>插入 O(1)<br/>无序"]
            BTREE_T["BTreeMap<br/>查找 O(log n)<br/>插入 O(log n)<br/>有序"]
        end
    end

    style VEC_T fill:#c8e6c9
    style HASH_T fill:#bbdefb
    style BTREE_T fill:#fff9c4
```

### 选择总结

| 场景 | 推荐类型 | 原因 |
|------|----------|------|
| 动态数组 | `Vec<T>` | 默认选择，性能最优 |
| FIFO 队列 | `VecDeque<T>` | 双端 O(1) |
| 键值存储 | `HashMap<K,V>` | O(1) 查找 |
| 有序键值 | `BTreeMap<K,V>` | 范围查询 |
| 唯一值集合 | `HashSet<T>` | O(1) 查找 |
| 有序集合 | `BTreeSet<T>` | 范围操作 |
| 优先队列 | `BinaryHeap<T>` | O(log n) 取最值 |
| 频繁中间插删 | `LinkedList<T>` | O(1) 已知位置操作 |

---

## 10. 迭代器与集合

```mermaid
graph LR
    subgraph "迭代器转换"
        ITER["iter()<br/>&T"]
        ITER_MUT["iter_mut()<br/>&mut T"]
        INTO_ITER["into_iter()<br/>T (消耗)"]
    end

    subgraph "收集方法"
        COLLECT["collect::<Vec<_>>()"]
        EXTEND["extend(iter)"]
        FROM_ITER["FromIterator::from_iter()"]
    end

    ITER --> COLLECT
    INTO_ITER --> COLLECT

    style ITER fill:#c8e6c9
    style COLLECT fill:#bbdefb
```

### 常用迭代模式

```rust
// 过滤并收集
let evens: Vec<_> = v.iter()
    .filter(|x| *x % 2 == 0)
    .collect();

// 转换类型
let strings: Vec<String> = nums.iter()
    .map(|n| n.to_string())
    .collect();

// 构建 HashMap
let map: HashMap<_, _> = pairs.into_iter().collect();

// 分组 (需要 itertools)
let groups: HashMap<_, Vec<_>> = items.iter()
    .map(|x| (x.category, x))
    .into_group_map();
```
