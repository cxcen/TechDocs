# 内存管理模块

> `core::mem`, `core::ptr`, `core::alloc` 深度解析

## 概述

Rust 的内存管理系统由三个核心模块组成，形成从底层指针操作到高级内存分配的完整层级：

```mermaid
graph TB
    APP["应用代码"]

    APP -->|使用| MEM["core::mem<br/>值和类型语义"]
    APP -->|使用| PTR["core::ptr<br/>指针操作"]
    APP -->|使用| ALLOC["core::alloc<br/>分配接口"]

    MEM -->|依赖| INTERNAL1["类型内存布局<br/>生命周期管理"]
    PTR -->|依赖| INTERNAL2["指针有效性<br/>Provenance"]
    ALLOC -->|依赖| INTERNAL3["分配器契约<br/>布局要求"]

    MEM -.->|配合| PTR
    PTR -.->|使用| ALLOC
    MEM -.->|支持| ALLOC

    INTERNAL1 --> UNSAFE["Unsafe 代码边界"]
    INTERNAL2 --> UNSAFE
    INTERNAL3 --> UNSAFE
    UNSAFE --> HW["硬件内存"]
```

---

## core::mem 模块

### 功能概述

`core::mem` 模块提供处理类型布局、初始化和销毁的基础功能。

### 核心类型

#### MaybeUninit<T> - 未初始化值容器

```mermaid
graph LR
    subgraph "MaybeUninit 生命周期"
        A["uninit()"] --> B["未初始化状态"]
        B --> C["write(value)"]
        C --> D["已初始化状态"]
        D --> E["assume_init()"]
        E --> F["T 类型值"]
    end
```

**核心用途**：安全地处理未初始化内存

| 方法 | 功能 | 安全性 |
|------|------|--------|
| `uninit()` | 创建未初始化容器 | Safe |
| `zeroed()` | 创建零值容器 | Safe |
| `write(val)` | 写入值 | Safe |
| `assume_init()` | 声明已初始化 | Unsafe |
| `assume_init_ref()` | 获取引用 | Unsafe |

**使用场景**：
- 延迟初始化
- 数组初始化
- Out-pointer 模式
- FFI 边界

#### ManuallyDrop<T> - 抑制自动销毁

```mermaid
graph TD
    subgraph "ManuallyDrop 控制流"
        A["ManuallyDrop::new(value)"]
        A --> B{"使用阶段"}
        B --> C["自动使用<br/>（不会 drop）"]
        B --> D["手动控制"]
        D --> E["ManuallyDrop::drop()"]
        D --> F["ManuallyDrop::into_inner()"]
        E --> G["值被销毁"]
        F --> H["值被取出<br/>可正常 drop"]
    end
```

**核心用途**：阻止编译器自动调用析构函数

| 方法 | 功能 | 安全性 |
|------|------|--------|
| `new(value)` | 创建包装 | Safe |
| `into_inner()` | 提取值 | Safe |
| `take()` | 取出值 | Unsafe |
| `drop()` | 手动销毁 | Unsafe |

**注意事项**：
- 零成本抽象，与 T 有完全相同的内存布局
- 不要与包含 `Box` 的类型一起使用（可能导致 UB）

#### Discriminant<T> - 枚举判别值

用于枚举变体的快速比较，透明包装实现 `PartialEq`、`Eq`、`Hash`。

### 关键函数

#### 大小和对齐查询

```rust
// 编译时确定
size_of::<T>() -> usize
align_of::<T>() -> usize

// 运行时查询（支持 DST）
size_of_val(&T) -> usize
align_of_val(&T) -> usize
```

#### 值操作

```mermaid
graph LR
    subgraph "值操作函数"
        SWAP["swap(&mut a, &mut b)<br/>交换两个值"]
        REPLACE["replace(&mut dest, src)<br/>替换并返回旧值"]
        TAKE["take(&mut dest)<br/>用 Default 替换"]
        DROP["drop(value)<br/>显式销毁"]
        FORGET["forget(value)<br/>阻止销毁"]
    end
```

| 函数 | 功能 | 安全性 |
|------|------|--------|
| `swap(x, y)` | 交换两个值 | Safe |
| `replace(dest, src)` | 替换并返回旧值 | Safe |
| `take(dest)` | 用 Default 替换 | Safe |
| `drop(value)` | 显式销毁 | Safe |
| `forget(value)` | 阻止销毁 | Safe |
| `needs_drop::<T>()` | 检查是否需要 drop | Safe |

#### Transmute 操作

```rust
// 危险：按位重新解释类型
transmute::<T, U>(value: T) -> U      // Unsafe
transmute_copy::<T, U>(&src) -> U     // Unsafe
```

**警告**：`transmute` 是最危险的操作之一，仅在必要时使用。

#### offset_of! 宏

```rust
// 获取字段偏移量
offset_of!(Type, field) -> usize
```

---

## core::ptr 模块

### 功能概述

`core::ptr` 处理原始指针的低级操作，提供安全的指针构造和指针语义（Provenance）的正式定义。

### Provenance（来源性）概念

```mermaid
graph TB
    subgraph "指针组成"
        PTR["指针"]
        PTR --> ADDR["地址 (Address)<br/>内存位置 usize"]
        PTR --> PROV["来源性 (Provenance)<br/>访问权限信息"]
    end

    subgraph "Provenance 成分"
        PROV --> SPATIAL["空间范围 (Spatial)<br/>允许访问的地址范围"]
        PROV --> TEMPORAL["时间范围 (Temporal)<br/>允许访问的时间段"]
        PROV --> MUT["可变性 (Mutability)<br/>读/写权限"]
    end
```

#### Strict Provenance vs Exposed Provenance

```mermaid
graph LR
    subgraph "两种 Provenance 模式"
        STRICT["Strict Provenance<br/>━━━━━━━━━━<br/>✓ 推荐使用<br/>✓ 易于验证<br/>✓ 支持 CHERI<br/>✓ 更安全"]

        EXPOSED["Exposed Provenance<br/>━━━━━━━━━━<br/>⚠ C 兼容<br/>⚠ 整数转换<br/>⚠ 不推荐<br/>⚠ 可能受限"]
    end

    STRICT -.->|优先选择| EXPOSED
```

### 核心类型

#### NonNull<T> - 非空指针

```mermaid
graph TD
    subgraph "NonNull<T> 特性"
        A["NonNull<T>"]
        A --> B["非零保证"]
        A --> C["协变性"]
        A --> D["空指针优化"]

        B --> B1["即使未解引用<br/>也保证非零"]
        C --> C1["适合 Box, Rc,<br/>Arc, Vec 等容器"]
        D --> D1["Option&lt;NonNull&lt;T&gt;&gt;<br/>与 *mut T 大小相同"]
    end
```

**关键方法**：

| 方法 | 功能 | 安全性 |
|------|------|--------|
| `new(ptr)` | 创建（返回 Option） | Safe |
| `new_unchecked(ptr)` | 创建（不检查） | Unsafe |
| `dangling()` | 创建悬空指针 | Safe |
| `as_ptr()` | 转换为 *mut T | Safe |
| `as_ref()` | 转换为 &T | Unsafe |
| `as_mut()` | 转换为 &mut T | Unsafe |

### 关键函数

#### 指针创建

```rust
null::<T>() -> *const T
null_mut::<T>() -> *mut T
dangling::<T>() -> *const T

// Strict Provenance
without_provenance::<T>(addr) -> *const T

// Exposed Provenance（不推荐）
with_exposed_provenance::<T>(addr) -> *const T
```

#### 内存读写

```mermaid
graph TB
    subgraph "内存操作"
        READ["读取操作"]
        READ --> R1["read(src)<br/>对齐读取"]
        READ --> R2["read_unaligned(src)<br/>非对齐读取"]
        READ --> R3["read_volatile(src)<br/>volatile 读取"]

        WRITE["写入操作"]
        WRITE --> W1["write(dst, val)<br/>对齐写入"]
        WRITE --> W2["write_unaligned(dst, val)<br/>非对齐写入"]
        WRITE --> W3["write_volatile(dst, val)<br/>volatile 写入"]

        BULK["批量操作"]
        BULK --> B1["copy(src, dst, count)<br/>允许重叠"]
        BULK --> B2["copy_nonoverlapping<br/>不允许重叠"]
        BULK --> B3["write_bytes(dst, val, count)<br/>填充字节"]
    end
```

#### addr_of! 宏

```rust
// 避免创建中间引用
addr_of!(place) -> *const T
addr_of_mut!(place) -> *mut T
```

---

## core::alloc 模块

### 功能概述

定义内存分配、释放、重新调整的接口，建立分配器与分配内存的契约。

### Layout - 内存布局描述

```mermaid
graph LR
    subgraph "Layout 结构"
        L["Layout"]
        L --> SIZE["size: usize<br/>内存大小"]
        L --> ALIGN["align: usize<br/>对齐要求（2的幂）"]
    end

    subgraph "Layout 组合"
        L --> EXT["extend(next)<br/>连接布局"]
        L --> REP["repeat(n)<br/>重复布局"]
        L --> ALN["align_to(align)<br/>增加对齐"]
        L --> PAD["pad_to_align()<br/>添加填充"]
    end
```

**关键方法**：

| 方法 | 功能 | 返回值 |
|------|------|--------|
| `new::<T>()` | 从类型创建 | Layout |
| `for_value(&T)` | 从值创建 | Layout |
| `from_size_align(size, align)` | 手动创建 | Result<Layout, LayoutError> |
| `size()` | 获取大小 | usize |
| `align()` | 获取对齐 | usize |
| `extend(next)` | 连接布局 | Result<(Layout, usize), LayoutError> |
| `repeat(n)` | 重复布局 | Result<(Layout, usize), LayoutError> |

### Allocator Traits

```mermaid
graph TB
    subgraph "分配器 Trait 层级"
        GA["GlobalAlloc (Stable)<br/>━━━━━━━━━━<br/>全局分配器<br/>返回 *mut u8<br/>不支持错误恢复"]

        AL["Allocator (Unstable)<br/>━━━━━━━━━━<br/>通用分配器<br/>返回 NonNull&lt;[u8]&gt;<br/>支持错误处理"]

        GA -.->|更简单| SIMPLE["简单使用场景"]
        AL -.->|更灵活| FLEX["灵活使用场景"]
    end
```

#### GlobalAlloc Trait

```rust
pub unsafe trait GlobalAlloc {
    // 必须实现
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);

    // 有默认实现
    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8;
    unsafe fn realloc(&self, ptr: *mut u8, layout: Layout,
                      new_size: usize) -> *mut u8;
}
```

#### Allocator Trait (Unstable)

```rust
pub unsafe trait Allocator {
    // 必须实现
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError>;
    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout);

    // 有默认实现
    fn allocate_zeroed(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError>;
    unsafe fn grow(&self, ptr: NonNull<u8>, old: Layout, new: Layout)
        -> Result<NonNull<[u8]>, AllocError>;
    unsafe fn shrink(&self, ptr: NonNull<u8>, old: Layout, new: Layout)
        -> Result<NonNull<[u8]>, AllocError>;
}
```

---

## 完整内存操作流程

```mermaid
graph TB
    START["开始内存操作"]

    START --> PHASE1["1. 规划阶段"]
    PHASE1 --> CALC["计算 Layout<br/>size + align"]

    CALC --> PHASE2["2. 分配阶段"]
    PHASE2 --> ALLOC["调用 Allocator::allocate<br/>返回 NonNull&lt;[u8]&gt;"]

    ALLOC --> PHASE3["3. 初始化阶段"]
    PHASE3 --> INIT["使用 MaybeUninit<br/>ptr::write()"]

    INIT --> PHASE4["4. 使用阶段"]
    PHASE4 --> USE["正常操作<br/>取引用、修改值"]

    USE --> PHASE5["5. 销毁阶段"]
    PHASE5 --> DESTROY["选择销毁方式"]

    DESTROY -->|自动| AUTO["默认 drop"]
    DESTROY -->|手动| MANUAL["ManuallyDrop<br/>显式销毁"]

    AUTO --> PHASE6["6. 释放阶段"]
    MANUAL --> PHASE6
    PHASE6 --> DEALLOC["Allocator::deallocate<br/>ptr + layout"]

    DEALLOC --> END["内存归还"]
```

---

## Unsafe 边界划分

```mermaid
graph LR
    subgraph SAFE["Safe Rust"]
        S1["size_of, align_of"]
        S2["drop, swap, replace"]
        S3["ManuallyDrop::new"]
        S4["Layout::new"]
    end

    subgraph UNSAFE["Unsafe Rust"]
        U1["transmute"]
        U2["ptr::read/write"]
        U3["assume_init"]
        U4["allocator 操作"]
    end

    subgraph HW["底层硬件"]
        H1["物理内存"]
        H2["CPU 缓存"]
    end

    SAFE -->|"必须遵守规则"| UNSAFE
    UNSAFE -->|"完全访问"| HW
```

---

## 设计原则总结

### 1. 零成本抽象
- `ManuallyDrop` 和 `MaybeUninit` 无运行时开销
- `NonNull` 与原始指针大小相同

### 2. 安全与 Unsafe 结合
- 提供 Safe API 包裹不安全操作
- 编译时验证大小和对齐信息

### 3. Provenance 形式化
- 明确定义指针的访问权限
- Strict vs Exposed 两种方案

### 4. 分配器设计
- `GlobalAlloc`：简单、全局、不支持错误恢复
- `Allocator`：灵活、本地、支持错误处理
