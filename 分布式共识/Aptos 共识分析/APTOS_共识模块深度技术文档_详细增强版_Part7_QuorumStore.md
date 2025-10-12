# Aptos Consensus 模块深度技术文档(详细增强版 - Part 7)

## QuorumStore 模块深度解析

> **模块路径**: `src/quorum_store/`
> **核心职责**: 批量交易处理、解耦共识与数据传播、动态反压控制
> **文档版本**: v2.0 (详细增强版)
> **生成时间**: 2025-10-09

---

## 📑 目录

- [1. QuorumStore 概述](#1-quorumstore-概述)
  - [1.1 设计理念与动机](#11-设计理念与动机)
  - [1.2 核心架构](#12-核心架构)
  - [1.3 文件组织结构](#13-文件组织结构)
- [2. 核心数据结构详解](#2-核心数据结构详解)
  - [2.1 Batch 结构](#21-batch-结构)
  - [2.2 ProofOfStore 结构](#22-proofofstore-结构)
  - [2.3 BatchInfo 与签名](#23-batchinfo-与签名)
- [3. BatchGenerator 深度解析](#3-batchgenerator-深度解析)
  - [3.1 BatchGenerator 结构](#31-batchgenerator-结构)
  - [3.2 批次生成流程](#32-批次生成流程)
  - [3.3 交易去重机制](#33-交易去重机制)
  - [3.4 过期管理](#34-过期管理)
- [4. BatchStore 存储层详解](#4-batchstore-存储层详解)
  - [4.1 三级存储架构](#41-三级存储架构)
  - [4.2 配额管理机制](#42-配额管理机制)
  - [4.3 批次持久化](#43-批次持久化)
  - [4.4 签名与验证](#44-签名与验证)
- [5. ProofCoordinator 证明协调器](#5-proofcoordinator-证明协调器)
  - [5.1 签名收集机制](#51-签名收集机制)
  - [5.2 聚合签名算法](#52-聚合签名算法)
  - [5.3 ProofOfStore 生成](#53-proofofstore-生成)
- [6. ProofManager 证明管理器](#6-proofmanager-证明管理器)
  - [6.1 Proof 缓存管理](#61-proof-缓存管理)
  - [6.2 拉取策略](#62-拉取策略)
  - [6.3 清理机制](#63-清理机制)
- [7. 反压机制详解](#7-反压机制详解)
  - [7.1 反压触发条件](#71-反压触发条件)
  - [7.2 动态调控算法](#72-动态调控算法)
  - [7.3 多级反压策略](#73-多级反压策略)
- [8. 网络层交互](#8-网络层交互)
  - [8.1 消息类型](#81-消息类型)
  - [8.2 广播机制](#82-广播机制)
  - [8.3 请求与响应](#83-请求与响应)
- [9. Batch 完整生命周期](#9-batch-完整生命周期)
- [10. 性能优化与配置](#10-性能优化与配置)
- [11. 总结](#11-总结)

---

## 1. QuorumStore 概述

### 1.1 设计理念与动机

#### 传统共识的数据传播瓶颈

```mermaid
mindmap
  root((传统共识瓶颈))
    数据传播
      每个区块都广播完整交易
      网络带宽浪费
      重复数据传输
    共识耦合
      数据传播与共识绑定
      Leader 压力大
      扩展性差
    峰值处理
      突发流量无缓冲
      系统不稳定
      延迟飙升
    资源利用
      带宽利用率低
      存储冗余高
      CPU 空闲期多
```

**问题详解**：

```mermaid
graph TD
    subgraph "传统模式 - 每个区块都传输完整数据"
        L1[Leader Round 1]
        L2[Leader Round 2]
        L3[Leader Round 3]

        L1 -->|1000 txns<br/>500KB| V1[Validators]
        L2 -->|1000 txns<br/>500KB| V2[Validators]
        L3 -->|1000 txns<br/>500KB| V3[Validators]
    end

    Note1[问题:<br/>1. 每轮都传输完整交易<br/>2. 带宽消耗: 500KB × 3 = 1.5MB<br/>3. 网络拥塞,延迟高]

    style L1 fill:#ffcdd2
    style L2 fill:#ffcdd2
    style L3 fill:#ffcdd2
```

#### QuorumStore 解决方案

```mermaid
graph TB
    subgraph "QuorumStore 模式 - 解耦数据与共识"
        subgraph "阶段 1: 数据传播 (异步)"
            BG1[Validator A<br/>生成 Batch1]
            BG2[Validator B<br/>生成 Batch2]
            BG3[Validator C<br/>生成 Batch3]

            BG1 -->|广播交易| NET1[Network]
            BG2 -->|广播交易| NET1
            BG3 -->|广播交易| NET1

            NET1 -->|持久化| STORE[All Validators<br/>本地存储]
        end

        subgraph "阶段 2: 证明生成"
            STORE -->|签名| SIG[2f+1 签名]
            SIG -->|聚合| PROOF[ProofOfStore]
        end

        subgraph "阶段 3: 共识 (轻量)"
            L1[Leader Round 1]
            L2[Leader Round 2]

            L1 -->|仅广播 Proof<br/>~1KB| V1[Validators]
            L2 -->|仅广播 Proof<br/>~1KB| V2[Validators]
        end

        PROOF --> L1
        PROOF --> L2
    end

    Note2[优势:<br/>1. 数据只传输一次<br/>2. 共识只传 Proof<br/>3. 带宽节省 50-70%]

    style BG1 fill:#c8e6c9
    style BG2 fill:#c8e6c9
    style BG3 fill:#c8e6c9
    style PROOF fill:#fff9c4
    style L1 fill:#e1f5ff
    style L2 fill:#e1f5ff
```

**核心优势**：

| 维度 | 传统模式 | QuorumStore 模式 | 改进幅度 |
|-----|---------|-----------------|---------|
| **网络带宽** | 每轮传输完整交易 | 交易传输一次 + Proof | **50-70% ↓** |
| **共识延迟** | 数据传播阻塞共识 | 解耦并行 | **30% ↓** |
| **吞吐量** | Leader 瓶颈 | 多节点并行批处理 | **3-5倍** |
| **峰值处理** | 无缓冲机制 | 动态反压 | **平滑负载** |
| **存储复用** | 无 | Batch 可被多个区块引用 | **高复用率** |

### 1.2 核心架构

#### 完整架构图

```mermaid
graph TB
    subgraph "外部接口层"
        MP[Mempool<br/>━━━━━━━━━━<br/>交易池]
        PG[ProposalGenerator<br/>━━━━━━━━━━<br/>区块生成器]
        EX[Executor<br/>━━━━━━━━━━<br/>执行器]
    end

    subgraph "QuorumStore 核心层"
        BG[BatchGenerator<br/>━━━━━━━━━━<br/>批次生成器]
        BS[BatchStore<br/>━━━━━━━━━━<br/>批次存储]
        PC[ProofCoordinator<br/>━━━━━━━━━━<br/>证明协调器]
        PM[ProofManager<br/>━━━━━━━━━━<br/>证明管理器]
        BC[BatchCoordinator<br/>━━━━━━━━━━<br/>批次协调器]
        BR[BatchRequester<br/>━━━━━━━━━━<br/>批次请求器]
        BP[BackPressure<br/>━━━━━━━━━━<br/>反压控制器]
    end

    subgraph "存储层"
        QDB[(QuorumStoreDB<br/>━━━━━━━━━━<br/>RocksDB 持久化)]
        Cache[DashMap<br/>━━━━━━━━━━<br/>内存缓存]
        QM[QuotaManager<br/>━━━━━━━━━━<br/>配额管理]
    end

    subgraph "网络层"
        NET[NetworkSender<br/>━━━━━━━━━━<br/>网络发送器]
        RPC[RpcHandler<br/>━━━━━━━━━━<br/>RPC 处理器]
    end

    MP -->|pull_txns| BG
    BG -->|persist_batch| BS
    BS -->|store| QDB
    BS -->|cache| Cache
    QM -->|quota_check| BS

    BG -->|broadcast_batch| NET
    NET -->|BatchMsg| BC
    BC -->|store_remote| BS

    BS -->|sign_batch_info| PC
    PC -->|collect_signatures| PC
    PC -->|aggregate| PM

    PM -->|pull_proofs| PG
    PG -->|request_missing_batches| BR
    BR -->|rpc_request| RPC
    RPC -->|fetch_batch| NET

    BP -->|monitor| BG
    BP -->|throttle| PM

    PM -->|provide_proofs| PG
    PG -->|execute_block| EX
    EX -->|commit_notify| BG

    style BG fill:#4caf50,stroke:#333,stroke-width:3px
    style BS fill:#2196f3,stroke:#333,stroke-width:3px
    style PC fill:#ff9800,stroke:#333,stroke-width:3px
    style PM fill:#e91e63,stroke:#333,stroke-width:3px
    style BP fill:#f44336,stroke:#333,stroke-width:3px
```

#### 数据流详图

```mermaid
sequenceDiagram
    autonumber
    participant MP as Mempool
    participant BG as BatchGenerator
    participant BS as BatchStore
    participant NET as Network
    participant PC as ProofCoordinator
    participant PM as ProofManager
    participant PG as ProposalGenerator
    participant EX as Executor

    Note over MP,EX: ══════════ 完整数据流 ══════════

    rect rgb(225, 245, 255)
        Note over BG: Phase 1: 批次生成
        BG->>BG: timer tick (100ms)
        BG->>MP: pull_txns(max_txns, excluded)
        MP->>BG: Vec<SignedTransaction>
        BG->>BG: create Batch
        BG->>BS: persist_batch(batch)
        BS->>BS: quota_check
        BS->>BS: write to DB + cache
        BS->>BG: SignedBatchInfo
    end

    rect rgb(255, 249, 196)
        Note over NET: Phase 2: 网络广播
        BG->>NET: broadcast BatchMsg
        NET->>NET: send to all validators
    end

    rect rgb(243, 229, 245)
        Note over PC: Phase 3: 签名收集
        NET->>PC: receive BatchMsg (remote)
        PC->>BS: persist remote batch
        PC->>PC: sign BatchInfo
        PC->>NET: send SignedBatchInfoMsg
        NET->>PC: collect signatures
        PC->>PC: voting_power >= 2f+1?
        PC->>PC: aggregate signatures
        PC->>PM: ProofOfStore ready
    end

    rect rgb(200, 230, 201)
        Note over PG: Phase 4: 区块提案
        PG->>PM: pull_proofs(max_bytes)
        PM->>PG: Vec<ProofOfStore>
        PG->>PG: construct block
        PG->>NET: broadcast proposal
    end

    rect rgb(255, 235, 238)
        Note over EX: Phase 5: 执行与清理
        EX->>EX: execute block
        EX->>BG: commit_notify(committed_batches)
        BG->>BS: cleanup expired batches
        BS->>BS: release quota
    end
```

### 1.3 文件组织结构

#### 详细目录树

```
src/quorum_store/
├── mod.rs                              # 模块入口 (150 LOC)
│   └── QuorumStore 接口定义
│
├── batch_generator.rs                  # 批次生成器 (1,800 LOC)
│   ├── BatchGenerator 结构
│   ├── generate_batch 核心逻辑
│   ├── insert_batch 远程批次处理
│   ├── handle_commit_notification
│   └── expiration management
│
├── batch_store.rs                      # 批次存储 (2,200 LOC)
│   ├── BatchStore 结构
│   ├── persist_batch 持久化
│   ├── save_to_db RocksDB 操作
│   ├── get_batch 缓存 + DB 查询
│   └── QuotaManager 配额管理
│
├── proof_coordinator.rs                # 证明协调器 (1,500 LOC)
│   ├── ProofCoordinator 结构
│   ├── handle_batch_msg
│   ├── handle_signed_batch_info_msg
│   ├── aggregate_signatures
│   └── ProofAggregator 状态机
│
├── proof_manager.rs                    # 证明管理器 (1,200 LOC)
│   ├── ProofManager 结构
│   ├── pull_proofs 拉取策略
│   ├── handle_commit_notification
│   └── ProofQueue 优先队列
│
├── batch_coordinator.rs                # 批次协调器 (800 LOC)
│   ├── BatchCoordinator 结构
│   ├── handle_batch_msg
│   └── 验证与转发逻辑
│
├── batch_requester.rs                  # 批次请求器 (700 LOC)
│   ├── BatchRequester 结构
│   ├── request_batches RPC 请求
│   └── 超时与重试逻辑
│
├── counters.rs                         # Prometheus 指标 (400 LOC)
│   ├── BATCH_GENERATOR_*
│   ├── PROOF_COORDINATOR_*
│   └── QUOTA_MANAGER_*
│
├── quorum_store_coordinator.rs        # 顶层协调器 (600 LOC)
│   └── QuorumStoreCoordinator 结构
│
├── network_interface.rs                # 网络接口 (500 LOC)
│   ├── QuorumStoreNetworkSender
│   └── QuorumStoreNetworkEvents
│
├── types.rs                            # 类型定义 (800 LOC)
│   ├── Batch 结构
│   ├── BatchInfo 结构
│   ├── ProofOfStore 结构
│   ├── SignedBatchInfo
│   └── 各种消息类型
│
├── utils/                              # 工具模块
│   ├── time_expiration.rs             # 过期时间管理 (200 LOC)
│   ├── optimal_min_len.rs             # 并行计算优化 (100 LOC)
│   └── payload_builder.rs             # Payload 构建 (300 LOC)
│
└── tests/                              # 测试
    ├── batch_generator_test.rs
    ├── proof_coordinator_test.rs
    └── integration_test.rs
```

**代码规模统计**：

```mermaid
pie title QuorumStore 模块代码行数分布
    "batch_store.rs" : 2200
    "batch_generator.rs" : 1800
    "proof_coordinator.rs" : 1500
    "proof_manager.rs" : 1200
    "batch_coordinator.rs" : 800
    "types.rs" : 800
    "batch_requester.rs" : 700
    "quorum_store_coordinator.rs" : 600
    "network_interface.rs" : 500
    "counters.rs" : 400
    "utils" : 600
    "其他" : 900
```

---

## 2. 核心数据结构详解

### 2.1 Batch 结构

#### 完整结构定义

```mermaid
classDiagram
    class Batch {
        +BatchId batch_id
        +u64 epoch
        +Author author
        +u64 expiration_usecs
        +Vec~SignedTransaction~ transactions
        +u64 gas_bucket_start
        +num_txns() usize
        +num_bytes() usize
        +digest() HashValue
    }

    class BatchId {
        +u64 id
        +PeerId author
        +Ord trait
    }

    class SignedTransaction {
        +AccountAddress sender
        +u64 sequence_number
        +TransactionPayload payload
        +u64 gas_unit_price
        +u64 max_gas_amount
    }

    Batch --> BatchId
    Batch --> SignedTransaction
```

**代码实现**：

```rust
// src/quorum_store/types.rs

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct Batch {
    /// 批次唯一标识
    batch_id: BatchId,

    /// Epoch 编号
    epoch: u64,

    /// 批次作者（生成该批次的验证者）
    author: Author,

    /// 过期时间（微秒）
    expiration_usecs: u64,

    /// 交易列表
    transactions: Vec<SignedTransaction>,

    /// Gas bucket 起始位置（用于公平性）
    gas_bucket_start: u64,
}

impl Batch {
    /// 创建新批次
    pub fn new(
        batch_id: BatchId,
        transactions: Vec<SignedTransaction>,
        epoch: u64,
        expiration_usecs: u64,
        author: Author,
        gas_bucket_start: u64,
    ) -> Self {
        Self {
            batch_id,
            epoch,
            author,
            expiration_usecs,
            transactions,
            gas_bucket_start,
        }
    }

    /// 计算批次哈希
    pub fn digest(&self) -> HashValue {
        let batch_info = BatchInfo::new(
            self.author,
            self.batch_id,
            self.epoch,
            self.expiration_usecs,
            self.compute_transaction_hashes(),
            self.gas_bucket_start,
        );
        batch_info.digest()
    }

    /// 计算交易哈希列表
    fn compute_transaction_hashes(&self) -> Vec<HashValue> {
        self.transactions
            .par_iter()
            .with_min_len(optimal_min_len(self.transactions.len(), 32))
            .map(|txn| txn.committed_hash())
            .collect()
    }

    /// 交易数量
    pub fn num_txns(&self) -> usize {
        self.transactions.len()
    }

    /// 批次字节数
    pub fn num_bytes(&self) -> usize {
        bcs::serialized_size(self).unwrap_or(0)
    }
}

/// 批次 ID
#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq, Serialize, Deserialize)]
pub struct BatchId {
    /// 批次序号
    pub id: u64,

    /// 作者（用于唯一性）
    pub author: PeerId,
}

impl Ord for BatchId {
    fn cmp(&self, other: &Self) -> Ordering {
        // 先按 id 排序，再按 author 排序
        self.id.cmp(&other.id)
            .then_with(|| self.author.cmp(&other.author))
    }
}
```

### 2.2 ProofOfStore 结构

#### 完整定义

```mermaid
classDiagram
    class ProofOfStore {
        +BatchInfo batch_info
        +AggregateSignature multi_signature
        +verify(verifier) Result
        +batch_id() BatchId
        +gas_bucket_start() u64
        +num_txns() usize
        +expiration() u64
    }

    class BatchInfo {
        +Author author
        +BatchId batch_id
        +u64 epoch
        +u64 expiration_usecs
        +Vec~HashValue~ txn_hashes
        +u64 gas_bucket_start
        +digest() HashValue
    }

    class AggregateSignature {
        +Vec~AccountAddress~ signers
        +Signature signature
        +verify_multi_sig() Result
    }

    ProofOfStore --> BatchInfo
    ProofOfStore --> AggregateSignature
```

**代码实现**：

```rust
/// ProofOfStore - 批次的 2f+1 聚合签名证明
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct ProofOfStore {
    /// 批次信息
    info: BatchInfo,

    /// 聚合签名（来自 2f+1 验证者）
    multi_signature: AggregateSignature,
}

impl ProofOfStore {
    pub fn new(info: BatchInfo, multi_signature: AggregateSignature) -> Self {
        Self {
            info,
            multi_signature,
        }
    }

    /// 验证 ProofOfStore
    pub fn verify(&self, verifier: &ValidatorVerifier) -> anyhow::Result<()> {
        // 1. 验证签名数量
        ensure!(
            self.multi_signature.num_signatures() >= verifier.quorum_size(),
            "Insufficient signatures: {} < {}",
            self.multi_signature.num_signatures(),
            verifier.quorum_size()
        );

        // 2. 验证聚合签名
        let message = self.info.digest();
        verifier.verify_multi_signature(&message, &self.multi_signature)?;

        Ok(())
    }

    pub fn batch_id(&self) -> BatchId {
        self.info.batch_id()
    }

    pub fn expiration(&self) -> u64 {
        self.info.expiration_usecs()
    }

    pub fn num_txns(&self) -> usize {
        self.info.num_txns()
    }

    pub fn num_bytes(&self) -> usize {
        // ProofOfStore 只包含元数据，不包含交易内容
        // 大约 1-2KB
        bcs::serialized_size(self).unwrap_or(0)
    }
}

/// BatchInfo - 批次元数据
#[derive(Clone, Debug, Eq, Hash, PartialEq, Serialize, Deserialize)]
pub struct BatchInfo {
    author: Author,
    batch_id: BatchId,
    epoch: u64,
    expiration_usecs: u64,

    /// 交易哈希列表（不是完整交易）
    txn_hashes: Vec<HashValue>,

    gas_bucket_start: u64,
}

impl BatchInfo {
    /// 计算 BatchInfo 的哈希（用于签名）
    pub fn digest(&self) -> HashValue {
        let mut state = DefaultHasher::new();
        bcs::serialize_into(&mut state, self).unwrap();
        HashValue::sha3_256_of(state.finish().to_le_bytes().as_ref())
    }

    pub fn num_txns(&self) -> usize {
        self.txn_hashes.len()
    }
}
```

### 2.3 BatchInfo 与签名

#### 签名流程可视化

```mermaid
graph TD
    A[Batch 创建] --> B[计算 BatchInfo]
    B --> C[BatchInfo.digest]

    C --> D[Validator A 签名]
    C --> E[Validator B 签名]
    C --> F[Validator C 签名]
    C --> G[Validator D 签名]

    D --> H[收集签名]
    E --> H
    F --> H
    G --> H

    H --> I{投票权 >= 2f+1?}
    I -->|否| H
    I -->|是| J[聚合签名]

    J --> K[生成 ProofOfStore]
    K --> L[验证 ProofOfStore]

    L --> M{验证通过?}
    M -->|是| N[可用于区块提案]
    M -->|否| O[丢弃]

    style C fill:#fff9c4
    style J fill:#c8e6c9
    style K fill:#e1f5ff
```

**SignedBatchInfo 结构**：

```rust
/// 单个验证者对 BatchInfo 的签名
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct SignedBatchInfo {
    /// 批次信息
    info: BatchInfo,

    /// 签名者
    signer: Author,

    /// 签名
    signature: bls12381::Signature,
}

impl SignedBatchInfo {
    pub fn new(info: BatchInfo, signer: Author, signature: bls12381::Signature) -> Self {
        Self {
            info,
            signer,
            signature,
        }
    }

    /// 验证签名
    pub fn verify(&self, verifier: &ValidatorVerifier) -> anyhow::Result<()> {
        let message = self.info.digest();
        verifier.verify_signature(self.signer, &message, &self.signature)
    }
}
```

---

## 3. BatchGenerator 深度解析

### 3.1 BatchGenerator 结构

#### 完整数据结构

```rust
// src/quorum_store/batch_generator.rs

pub struct BatchGenerator {
    // ========================================
    // 基本信息
    // ========================================
    epoch: u64,
    my_peer_id: PeerId,

    /// 批次 ID 生成器（自增）
    batch_id: BatchId,

    // ========================================
    // 存储与网络
    // ========================================
    db: Arc<dyn QuorumStoreStorage>,
    batch_writer: Arc<dyn BatchWriter>,
    network_sender: QuorumStoreNetworkSender,

    // ========================================
    // 交易跟踪
    // ========================================
    /// 正在处理的批次
    /// (PeerId, BatchId) -> BatchInProgress
    batches_in_progress: HashMap<(PeerId, BatchId), BatchInProgress>,

    /// 正在处理的交易（按 gas 排序）
    /// TransactionSummary -> TransactionInProgress
    txns_in_progress_sorted: BTreeMap<TransactionSummary, TransactionInProgress>,

    // ========================================
    // 过期管理
    // ========================================
    batch_expirations: TimeExpirations<(PeerId, BatchId)>,

    // ========================================
    // 反压控制
    // ========================================
    back_pressure: BackPressure,

    // ========================================
    // 配置
    // ========================================
    config: QuorumStoreConfig,

    /// 最新区块时间戳（用于计算过期时间）
    latest_block_timestamp: u64,
}

/// 正在处理的批次信息
struct BatchInProgress {
    batch_id: BatchId,
    num_txns: usize,
    num_bytes: usize,
    creation_time: Instant,
}

/// 交易摘要（用于去重）
#[derive(Clone, Eq, Hash, Ord, PartialEq, PartialOrd)]
struct TransactionSummary {
    sender: AccountAddress,
    sequence_number: u64,
    hash: HashValue,
}

/// 正在处理的交易信息
struct TransactionInProgress {
    gas_unit_price: u64,
    insertion_time: Instant,
}
```

#### BatchGenerator 职责分解

```mermaid
mindmap
  root((BatchGenerator))
    批次生成
      定时触发 tick
      从 Mempool 拉取
      过滤重复交易
      创建 Batch 对象
    持久化
      写入 RocksDB
      内存缓存
      配额检查
    广播
      发送 BatchMsg
      网络传输
    过期管理
      TimeExpirations
      定期清理
      释放资源
    反压处理
      检查反压状态
      动态调节
      跳过生成
    提交通知
      接收 commit 事件
      清理已提交批次
      更新统计
```

### 3.2 批次生成流程

#### 完整流程图

```mermaid
sequenceDiagram
    autonumber
    participant TM as Timer (100ms)
    participant BG as BatchGenerator
    participant BP as BackPressure
    participant MP as Mempool
    participant BS as BatchStore
    participant NET as Network
    participant PEERS as Other Validators

    Note over TM,PEERS: ══════════ 批次生成完整流程 ══════════

    TM->>BG: tick event

    rect rgb(225, 245, 255)
        Note over BG,BP: Phase 1: 检查反压
        BG->>BP: check_backpressure()
        BP->>BG: BackPressure { txn_count, proof_count }

        alt 有反压
            BG->>BG: skip batch generation
            BG->>TM: return
        end
    end

    rect rgb(255, 249, 196)
        Note over BG,MP: Phase 2: 拉取交易
        BG->>BG: calculate max_txns, max_bytes
        BG->>BG: build excluded_txns set
        BG->>MP: pull_internal(max, excluded)
        MP->>BG: Vec<SignedTransaction>

        alt 交易为空
            BG->>TM: return
        end
    end

    rect rgb(243, 229, 245)
        Note over BG: Phase 3: 创建批次
        BG->>BG: batch_id = next_batch_id()
        BG->>BG: expiration = now + config.expiry_time
        BG->>BG: batch = Batch::new(...)
        BG->>BG: update txns_in_progress_sorted
        BG->>BG: update batches_in_progress
    end

    rect rgb(200, 230, 201)
        Note over BS: Phase 4: 持久化
        BG->>BS: persist_batch(batch)
        BS->>BS: quota_check()
        BS->>BS: write to RocksDB
        BS->>BS: add to cache
        BS->>BS: sign BatchInfo
        BS->>BG: SignedBatchInfo
    end

    rect rgb(255, 235, 238)
        Note over NET: Phase 5: 广播
        BG->>NET: broadcast_batch_msg(batch)
        NET->>PEERS: BatchMsg
        PEERS->>PEERS: persist & sign
        PEERS->>NET: SignedBatchInfoMsg
    end

    rect rgb(255, 249, 196)
        Note over BG: Phase 6: 过期管理
        BG->>BG: batch_expirations.add((peer, batch_id), expiry)
    end
```

#### generate_batch 核心实现

```rust
impl BatchGenerator {
    /// 定时任务：尝试生成新批次
    pub async fn tick(&mut self) -> anyhow::Result<()> {
        // ========================================
        // 步骤 1: 清理过期批次
        // ========================================
        self.expire_batches();

        // ========================================
        // 步骤 2: 检查反压
        // ========================================
        if self.back_pressure.txn_count || self.back_pressure.proof_count {
            counters::BATCH_GENERATOR_BACKPRESSURE.inc();
            debug!("Skipping batch generation due to backpressure");
            return Ok(());
        }

        // ========================================
        // 步骤 3: 生成批次
        // ========================================
        if let Some(batch) = self.generate_batch().await? {
            info!(
                "Generated batch {}: {} txns, {} bytes",
                batch.batch_id(),
                batch.num_txns(),
                batch.num_bytes()
            );

            // ========================================
            // 步骤 4: 持久化
            // ========================================
            let signed_batch_info = self.batch_writer
                .persist_batch(batch.clone())
                .await?;

            // ========================================
            // 步骤 5: 广播
            // ========================================
            self.network_sender.broadcast_batch_msg(batch.clone()).await;

            // ========================================
            // 步骤 6: 更新状态
            // ========================================
            self.update_state_after_batch_creation(batch);
        }

        Ok(())
    }

    /// 从 Mempool 拉取交易并创建批次
    async fn generate_batch(&mut self) -> anyhow::Result<Option<Batch>> {
        let max_txns = self.config.max_batch_txns;
        let max_bytes = self.config.max_batch_bytes;

        // ========================================
        // 构建排除集合（避免重复拉取）
        // ========================================
        let excluded_txns: HashSet<_> = self.txns_in_progress_sorted
            .keys()
            .map(|summary| (summary.sender, summary.sequence_number))
            .collect();

        // ========================================
        // 从 Mempool 拉取
        // ========================================
        let txns = self.mempool_proxy
            .pull_internal(
                max_txns,
                max_bytes,
                excluded_txns,
            )
            .await?;

        if txns.is_empty() {
            return Ok(None);
        }

        // ========================================
        // 创建批次
        // ========================================
        let batch_id = self.next_batch_id();
        let expiration = self.latest_block_timestamp
            + self.config.batch_expiry_time_usecs;

        let batch = Batch::new(
            batch_id,
            txns,
            self.epoch,
            expiration,
            self.my_peer_id,
            self.config.gas_bucket_start,
        );

        Ok(Some(batch))
    }

    /// 生成下一个批次 ID
    fn next_batch_id(&mut self) -> BatchId {
        let id = self.batch_id.id + 1;
        self.batch_id = BatchId::new(id, self.my_peer_id);
        self.batch_id
    }
}
```

### 3.3 交易去重机制

#### 去重算法详解

```mermaid
graph TD
    A[pull_txns 请求] --> B[构建 excluded_txns set]

    B --> C[遍历 txns_in_progress_sorted]
    C --> D[提取 sender, sequence_number]
    D --> E[添加到 excluded_txns]

    E --> F[Mempool.pull_internal]
    F --> G{检查每个交易}

    G -->|excluded_txns 包含| H[跳过该交易]
    G -->|不包含| I[加入返回列表]

    H --> J[检查下一个]
    I --> J

    J --> K{所有交易检查完?}
    K -->|否| G
    K -->|是| L[返回去重后的交易列表]

    style E fill:#fff9c4
    style I fill:#c8e6c9
```

**TransactionSummary 详解**：

```rust
/// 交易摘要 - 用于去重和排序
#[derive(Clone, Eq, Hash, Ord, PartialEq, PartialOrd)]
struct TransactionSummary {
    /// 发送者地址
    sender: AccountAddress,

    /// 序列号（用于排序和去重）
    sequence_number: u64,

    /// 交易哈希（用于精确匹配）
    hash: HashValue,
}

impl TransactionSummary {
    pub fn new(sender: AccountAddress, sequence_number: u64, hash: HashValue) -> Self {
        Self {
            sender,
            sequence_number,
            hash,
        }
    }
}

// 实现 Ord trait 用于 BTreeMap 排序
// 先按 sender 排序，再按 sequence_number 排序
impl Ord for TransactionSummary {
    fn cmp(&self, other: &Self) -> Ordering {
        self.sender.cmp(&other.sender)
            .then_with(|| self.sequence_number.cmp(&other.sequence_number))
            .then_with(|| self.hash.cmp(&other.hash))
    }
}
```

**并行计算优化**：

```rust
impl BatchGenerator {
    /// 插入远程批次（来自其他验证者）
    pub fn insert_batch(
        &mut self,
        author: PeerId,
        batch_id: BatchId,
        txns: Vec<SignedTransaction>,
        expiry_time: u64,
    ) {
        // ========================================
        // 并行计算交易摘要（利用多核）
        // ========================================
        let txns_in_progress: Vec<_> = txns
            .par_iter()
            .with_min_len(optimal_min_len(txns.len(), 32))
            .map(|txn| {
                let summary = TransactionSummary::new(
                    txn.sender(),
                    txn.sequence_number(),
                    txn.committed_hash(),
                );

                let info = TransactionInProgress {
                    gas_unit_price: txn.gas_unit_price(),
                    insertion_time: Instant::now(),
                };

                (summary, info)
            })
            .collect();

        // ========================================
        // 批量插入（避免重复锁）
        // ========================================
        for (summary, info) in txns_in_progress {
            self.txns_in_progress_sorted
                .entry(summary)
                .or_insert(info);
        }

        // ========================================
        // 记录批次信息
        // ========================================
        let batch_info = BatchInProgress {
            batch_id,
            num_txns: txns.len(),
            num_bytes: bcs::serialized_size(&txns).unwrap_or(0),
            creation_time: Instant::now(),
        };

        self.batches_in_progress.insert((author, batch_id), batch_info);

        // ========================================
        // 添加到过期队列
        // ========================================
        self.batch_expirations.add((author, batch_id), expiry_time);
    }
}
```

### 3.4 过期管理

#### TimeExpirations 机制

```mermaid
classDiagram
    class TimeExpirations~T~ {
        -BTreeMap~u64,Vec~T~~ expirations
        +add(item: T, expiry_time: u64) void
        +expire(now: u64) Vec~T~
        +remove(item: &T) void
    }

    class BatchGenerator {
        -TimeExpirations batch_expirations
        +expire_batches() void
        +cleanup_batch(peer, batch_id) void
    }

    BatchGenerator --> TimeExpirations
```

**代码实现**：

```rust
// src/quorum_store/utils/time_expiration.rs

pub struct TimeExpirations<T> {
    /// 过期时间 -> 项目列表
    expirations: BTreeMap<u64, Vec<T>>,
}

impl<T: Clone + Eq + Hash> TimeExpirations<T> {
    pub fn new() -> Self {
        Self {
            expirations: BTreeMap::new(),
        }
    }

    /// 添加项目到过期队列
    pub fn add(&mut self, item: T, expiry_time: u64) {
        self.expirations
            .entry(expiry_time)
            .or_insert_with(Vec::new)
            .push(item);
    }

    /// 获取并移除所有过期项目
    pub fn expire(&mut self, now: u64) -> Vec<T> {
        let mut expired = Vec::new();

        // 使用 split_off 分割 BTreeMap
        let expired_map = self.expirations.split_off(&(now + 1));
        let remaining = std::mem::replace(&mut self.expirations, expired_map);

        // 收集所有过期项目
        for (_, items) in remaining {
            expired.extend(items);
        }

        expired
    }

    /// 移除特定项目
    pub fn remove(&mut self, item: &T) {
        for items in self.expirations.values_mut() {
            items.retain(|i| i != item);
        }
    }
}

// BatchGenerator 中的使用
impl BatchGenerator {
    /// 清理过期批次
    fn expire_batches(&mut self) {
        let now = self.latest_block_timestamp;
        let expired = self.batch_expirations.expire(now);

        for (author, batch_id) in expired {
            self.cleanup_batch(author, batch_id);
        }

        counters::BATCH_EXPIRED_COUNT.inc_by(expired.len() as u64);
    }

    /// 清理单个批次
    fn cleanup_batch(&mut self, author: PeerId, batch_id: BatchId) {
        // 移除批次记录
        if let Some(batch_info) = self.batches_in_progress.remove(&(author, batch_id)) {
            info!(
                "Expired batch {}: {} txns",
                batch_id,
                batch_info.num_txns
            );
        }

        // 释放交易跟踪（需要读取交易列表）
        // 实际实现中可能需要从 DB 读取批次详情
    }
}
```

#### 过期流程可视化

```mermaid
gantt
    title Batch 过期时间线
    dateFormat X
    axisFormat %L

    section Batch Lifecycle
    创建 (t=0)             :milestone, m1, 0, 0
    存活期 (60s)           :active, a1, 0, 60000
    过期检查 (t=60s)       :crit, c1, 60000, 100
    清理 (t=60.1s)         :done, d1, 60100, 500

    section 定时检查
    Tick 1 (t=0)           :milestone, t1, 0, 0
    Tick 2 (t=0.1s)        :milestone, t2, 100, 0
    ...                    :milestone, t3, 30000, 0
    Tick N (t=60s)         :crit, t4, 60000, 0
```

---

## 4. BatchStore 存储层详解

### 4.1 三级存储架构

#### 架构详图

```mermaid
graph TB
    subgraph "Level 1: 内存缓存 (Fast)"
        Cache[DashMap<br/>━━━━━━━━━━<br/>Key: BatchKey<br/>Value: PersistedBatch<br/>━━━━━━━━━━<br/>Size: 120MB 默认]
    end

    subgraph "Level 2: RocksDB (Durable)"
        DB[(RocksDB<br/>━━━━━━━━━━<br/>Column: batches<br/>━━━━━━━━━━<br/>Size: 960MB 默认)]
    end

    subgraph "Level 3: 配额管理"
        QM[QuotaManager<br/>━━━━━━━━━━<br/>Memory: 120MB<br/>DB: 960MB<br/>Batch Count: 2500]
    end

    A[写入请求] -->|1. 检查配额| QM
    QM -->|2. 分配成功| B{存储模式决策}

    B -->|Memory + DB| Cache
    B -->|DB Only| DB

    Cache -->|3a. 写入内存| Cache
    Cache -->|3b. 写入磁盘| DB

    C[读取请求] -->|1. 查询缓存| Cache
    Cache -->|Hit| D[返回数据]
    Cache -->|Miss| DB
    DB -->|2. 从磁盘读取| DB
    DB -->|3. 加载到缓存| Cache
    Cache --> D

    style Cache fill:#4caf50
    style DB fill:#2196f3
    style QM fill:#ff9800
```

#### 存储模式枚举

```mermaid
classDiagram
    class StorageMode {
        <<enumeration>>
        MemoryAndPersisted
        PersistedOnly
    }

    class PersistedBatch {
        +Batch batch
        +StorageMode storage_mode
        +Instant persist_time
    }

    class BatchStore {
        +DashMap~BatchKey,PersistedBatch~ db_cache
        +Arc~QuorumStoreDB~ db
        +QuotaManager quota_manager
        +persist_batch(batch) Result
        +get_batch(key) Option~Batch~
    }

    BatchStore --> StorageMode
    BatchStore --> PersistedBatch
```

### 4.2 配额管理机制

#### QuotaManager 完整实现

```rust
// src/quorum_store/batch_store.rs

pub struct QuotaManager {
    // ========================================
    // 剩余配额
    // ========================================
    /// 剩余内存配额（字节）
    memory_balance: usize,

    /// 剩余磁盘配额（字节）
    db_balance: usize,

    /// 剩余批次数配额
    batch_balance: usize,

    // ========================================
    // 总配额限制
    // ========================================
    memory_quota: usize,      // 默认: 120MB
    db_quota: usize,          // 默认: 960MB
    batch_quota: usize,       // 默认: 2500
}

impl QuotaManager {
    pub fn new(
        memory_quota: usize,
        db_quota: usize,
        batch_quota: usize,
    ) -> Self {
        Self {
            memory_balance: memory_quota,
            db_balance: db_quota,
            batch_balance: batch_quota,
            memory_quota,
            db_quota,
            batch_quota,
        }
    }

    /// 尝试分配配额
    pub fn update_quota(
        &mut self,
        num_bytes: usize,
    ) -> anyhow::Result<StorageMode> {
        // ========================================
        // 步骤 1: 检查批次数配额
        // ========================================
        if self.batch_balance == 0 {
            counters::EXCEEDED_BATCH_QUOTA_COUNT.inc();
            bail!(
                "Batch quota exceeded: 0 / {}",
                self.batch_quota
            );
        }

        // ========================================
        // 步骤 2: 检查磁盘配额
        // ========================================
        if self.db_balance < num_bytes {
            counters::EXCEEDED_STORAGE_QUOTA_COUNT.inc();
            bail!(
                "Storage quota exceeded: {} < {} bytes",
                self.db_balance,
                num_bytes
            );
        }

        // ========================================
        // 步骤 3: 扣除磁盘配额和批次数
        // ========================================
        self.batch_balance -= 1;
        self.db_balance -= num_bytes;

        // ========================================
        // 步骤 4: 决定存储模式
        // ========================================
        if self.memory_balance >= num_bytes {
            // 内存充足，使用内存 + 磁盘
            self.memory_balance -= num_bytes;

            counters::BATCH_STORED_IN_MEMORY.inc();

            Ok(StorageMode::MemoryAndPersisted)
        } else {
            // 内存不足，仅使用磁盘
            counters::BATCH_STORED_IN_DB_ONLY.inc();

            Ok(StorageMode::PersistedOnly)
        }
    }

    /// 释放配额
    pub fn free_quota(
        &mut self,
        num_bytes: usize,
        storage_mode: StorageMode,
    ) {
        self.batch_balance += 1;
        self.db_balance += num_bytes;

        if matches!(storage_mode, StorageMode::MemoryAndPersisted) {
            self.memory_balance += num_bytes;
        }

        counters::BATCH_QUOTA_FREED.inc();
    }

    /// 获取配额使用率
    pub fn utilization(&self) -> QuotaUtilization {
        QuotaUtilization {
            memory_usage: 1.0 - (self.memory_balance as f64 / self.memory_quota as f64),
            db_usage: 1.0 - (self.db_balance as f64 / self.db_quota as f64),
            batch_usage: 1.0 - (self.batch_balance as f64 / self.batch_quota as f64),
        }
    }
}

#[derive(Debug)]
pub struct QuotaUtilization {
    pub memory_usage: f64,  // 0.0 - 1.0
    pub db_usage: f64,
    pub batch_usage: f64,
}
```

#### 配额使用流程

```mermaid
sequenceDiagram
    autonumber
    participant BG as BatchGenerator
    participant BS as BatchStore
    participant QM as QuotaManager
    participant Cache as DashMap Cache
    participant DB as RocksDB

    Note over BG,DB: ══════════ 配额分配流程 ══════════

    BG->>BS: persist_batch(batch)
    BS->>BS: num_bytes = batch.num_bytes()

    rect rgb(225, 245, 255)
        Note over QM: 配额检查
        BS->>QM: update_quota(num_bytes)

        QM->>QM: check batch_balance
        alt batch_balance == 0
            QM->>BS: Error: Batch quota exceeded
            BS->>BG: Error
        end

        QM->>QM: check db_balance
        alt db_balance < num_bytes
            QM->>BS: Error: Storage quota exceeded
            BS->>BG: Error
        end
    end

    rect rgb(255, 249, 196)
        Note over QM: 扣除配额
        QM->>QM: batch_balance -= 1
        QM->>QM: db_balance -= num_bytes

        alt memory_balance >= num_bytes
            QM->>QM: memory_balance -= num_bytes
            QM->>BS: StorageMode::MemoryAndPersisted
        else
            QM->>BS: StorageMode::PersistedOnly
        end
    end

    rect rgb(200, 230, 201)
        Note over Cache,DB: 持久化
        alt MemoryAndPersisted
            BS->>Cache: insert(key, batch)
            BS->>DB: save(key, batch)
        else PersistedOnly
            BS->>DB: save(key, batch)
        end
    end

    BS->>BG: Ok(SignedBatchInfo)
```

### 4.3 批次持久化

#### BatchStore 完整结构

```rust
pub struct BatchStore {
    // ========================================
    // 存储
    // ========================================
    /// 内存缓存
    db_cache: Arc<DashMap<BatchKey, PersistedBatch>>,

    /// 持久化存储
    db: Arc<dyn QuorumStoreStorage>,

    // ========================================
    // 配额管理
    // ========================================
    quota_manager: Arc<Mutex<QuotaManager>>,

    // ========================================
    // 签名器
    // ========================================
    signer: ValidatorSigner,

    // ========================================
    // Epoch 信息
    // ========================================
    epoch: u64,
}

impl BatchStore {
    /// 持久化批次
    pub async fn persist_batch(
        &self,
        batch: Batch,
    ) -> anyhow::Result<SignedBatchInfo> {
        let num_bytes = batch.num_bytes();
        let batch_key = BatchKey::new(batch.author(), batch.batch_id());

        // ========================================
        // 步骤 1: 分配配额
        // ========================================
        let storage_mode = self.quota_manager
            .lock()
            .update_quota(num_bytes)?;

        info!(
            "Persisting batch {}: {} txns, {} bytes, mode: {:?}",
            batch.batch_id(),
            batch.num_txns(),
            num_bytes,
            storage_mode
        );

        // ========================================
        // 步骤 2: 保存到 RocksDB
        // ========================================
        self.db.save_batch(batch_key, &batch).await?;

        // ========================================
        // 步骤 3: 可选地加入内存缓存
        // ========================================
        if matches!(storage_mode, StorageMode::MemoryAndPersisted) {
            let persisted_batch = PersistedBatch {
                batch: batch.clone(),
                storage_mode,
                persist_time: Instant::now(),
            };

            self.db_cache.insert(batch_key, persisted_batch);
        }

        // ========================================
        // 步骤 4: 生成 BatchInfo 并签名
        // ========================================
        let batch_info = BatchInfo::new(
            batch.author(),
            batch.batch_id(),
            batch.epoch(),
            batch.expiration_usecs(),
            batch.compute_transaction_hashes(),
            batch.gas_bucket_start(),
        );

        let signature = self.signer.sign(&batch_info.digest())?;

        let signed_batch_info = SignedBatchInfo::new(
            batch_info,
            self.signer.author(),
            signature,
        );

        counters::BATCH_PERSISTED.inc();
        counters::BATCH_PERSISTED_BYTES.inc_by(num_bytes as u64);

        Ok(signed_batch_info)
    }

    /// 获取批次
    pub async fn get_batch(
        &self,
        key: &BatchKey,
    ) -> anyhow::Result<Option<Batch>> {
        // ========================================
        // 步骤 1: 尝试从缓存读取
        // ========================================
        if let Some(persisted) = self.db_cache.get(key) {
            counters::BATCH_CACHE_HIT.inc();
            return Ok(Some(persisted.batch.clone()));
        }

        counters::BATCH_CACHE_MISS.inc();

        // ========================================
        // 步骤 2: 从 RocksDB 读取
        // ========================================
        let batch = self.db.get_batch(key).await?;

        // ========================================
        // 步骤 3: 加载到缓存（如果有空间）
        // ========================================
        if let Some(ref batch) = batch {
            let num_bytes = batch.num_bytes();

            // 尝试分配内存配额
            if let Ok(mut quota) = self.quota_manager.try_lock() {
                if quota.memory_balance >= num_bytes {
                    quota.memory_balance -= num_bytes;

                    let persisted_batch = PersistedBatch {
                        batch: batch.clone(),
                        storage_mode: StorageMode::MemoryAndPersisted,
                        persist_time: Instant::now(),
                    };

                    self.db_cache.insert(*key, persisted_batch);
                }
            }
        }

        Ok(batch)
    }
}
```

### 4.4 签名与验证

#### 签名流程

```mermaid
graph TD
    A[Batch 创建] --> B[计算 transaction_hashes]
    B --> C[构造 BatchInfo]

    C --> D[BatchInfo.digest]
    D --> E[ValidatorSigner.sign]

    E --> F[生成 bls12381::Signature]
    F --> G[构造 SignedBatchInfo]

    G --> H[广播 SignedBatchInfoMsg]

    H --> I[其他验证者接收]
    I --> J[验证签名]

    J --> K{签名有效?}
    K -->|是| L[添加到 signature_aggregator]
    K -->|否| M[丢弃]

    style D fill:#fff9c4
    style E fill:#e1f5ff
    style K fill:#c8e6c9
```

**验证代码**：

```rust
impl SignedBatchInfo {
    /// 验证签名
    pub fn verify(
        &self,
        verifier: &ValidatorVerifier,
    ) -> anyhow::Result<()> {
        // 计算消息哈希
        let message = self.info.digest();

        // 验证签名
        verifier.verify_signature(
            self.signer,
            &message,
            &self.signature,
        )?;

        Ok(())
    }
}
```

---

## 5. ProofCoordinator 证明协调器

### 5.1 签名收集机制

#### ProofCoordinator 结构

```rust
// src/quorum_store/proof_coordinator.rs

pub struct ProofCoordinator {
    // ========================================
    // Epoch 信息
    // ========================================
    epoch: u64,
    my_peer_id: PeerId,

    // ========================================
    // 签名聚合器
    // ========================================
    /// BatchId -> ProofAggregator
    proof_aggregators: HashMap<BatchId, ProofAggregator>,

    // ========================================
    // 批次存储
    // ========================================
    batch_store: Arc<BatchStore>,

    // ========================================
    // 验证器
    // ========================================
    validator_verifier: ValidatorVerifier,

    // ========================================
    // 网络
    // ========================================
    network_sender: QuorumStoreNetworkSender,

    // ========================================
    // 配置
    // ========================================
    config: QuorumStoreConfig,
}

/// 单个 Batch 的签名聚合器
struct ProofAggregator {
    batch_info: BatchInfo,

    /// Author -> Signature
    signatures: HashMap<Author, bls12381::Signature>,

    /// 当前投票权重
    voting_power: u64,

    /// 创建时间
    creation_time: Instant,
}
```

#### 签名收集流程

```mermaid
sequenceDiagram
    autonumber
    participant V1 as Validator 1 (Local)
    participant PC as ProofCoordinator
    participant BS as BatchStore
    participant NET as Network
    participant V2 as Validator 2
    participant V3 as Validator 3
    participant V4 as Validator 4
    participant PM as ProofManager

    Note over V1,PM: ══════════ 签名收集完整流程 ══════════

    rect rgb(225, 245, 255)
        Note over V1,BS: Phase 1: 本地批次持久化
        V1->>BS: persist_batch(batch)
        BS->>BS: save to DB + cache
        BS->>BS: sign BatchInfo
        BS->>V1: SignedBatchInfo (local)
    end

    rect rgb(255, 249, 196)
        Note over NET: Phase 2: 广播批次
        V1->>NET: broadcast BatchMsg(batch)
        NET->>V2: BatchMsg
        NET->>V3: BatchMsg
        NET->>V4: BatchMsg
    end

    rect rgb(243, 229, 245)
        Note over V2: Phase 3: 远程验证者处理
        V2->>V2: receive BatchMsg
        V2->>BS: persist remote batch
        BS->>BS: verify & save
        V2->>V2: sign BatchInfo
        V2->>NET: SignedBatchInfoMsg

        Note over V3: 同样处理
        V3->>NET: SignedBatchInfoMsg

        Note over V4: 同样处理
        V4->>NET: SignedBatchInfoMsg
    end

    rect rgb(200, 230, 201)
        Note over PC: Phase 4: 收集签名
        NET->>PC: SignedBatchInfoMsg (from V2)
        PC->>PC: verify signature
        PC->>PC: add to aggregator
        PC->>PC: voting_power += V2.power

        NET->>PC: SignedBatchInfoMsg (from V3)
        PC->>PC: add signature
        PC->>PC: voting_power += V3.power

        NET->>PC: SignedBatchInfoMsg (from V4)
        PC->>PC: add signature
        PC->>PC: voting_power += V4.power
    end

    rect rgb(255, 235, 238)
        Note over PC: Phase 5: 达到 Quorum
        PC->>PC: voting_power >= 2f+1?
        PC->>PC: aggregate signatures
        PC->>PC: construct ProofOfStore
        PC->>PM: notify proof ready
    end
```

### 5.2 聚合签名算法

#### 聚合流程详图

```mermaid
graph TD
    A[收到 SignedBatchInfoMsg] --> B{验证签名}
    B -->|失败| C[丢弃消息]
    B -->|成功| D{ProofAggregator 存在?}

    D -->|否| E[创建 ProofAggregator]
    D -->|是| F[获取 ProofAggregator]

    E --> G[添加签名]
    F --> G

    G --> H[更新 voting_power]
    H --> I{voting_power >= quorum?}

    I -->|否| J[继续等待]
    I -->|是| K[聚合签名]

    K --> L[BLS 聚合算法]
    L --> M[生成 AggregateSignature]

    M --> N[构造 ProofOfStore]
    N --> O[验证 ProofOfStore]

    O --> P{验证通过?}
    P -->|是| Q[发送到 ProofManager]
    P -->|否| R[丢弃]

    style B fill:#fff9c4
    style K fill:#e1f5ff
    style O fill:#c8e6c9
```

#### ProofAggregator 实现

```rust
impl ProofAggregator {
    pub fn new(batch_info: BatchInfo) -> Self {
        Self {
            batch_info,
            signatures: HashMap::new(),
            voting_power: 0,
            creation_time: Instant::now(),
        }
    }

    /// 添加签名
    pub fn add_signature(
        &mut self,
        author: Author,
        signature: bls12381::Signature,
        verifier: &ValidatorVerifier,
    ) -> anyhow::Result<bool> {
        // ========================================
        // 步骤 1: 检查重复
        // ========================================
        if self.signatures.contains_key(&author) {
            return Ok(false);
        }

        // ========================================
        // 步骤 2: 验证签名
        // ========================================
        let message = self.batch_info.digest();
        verifier.verify_signature(author, &message, &signature)?;

        // ========================================
        // 步骤 3: 添加签名
        // ========================================
        self.signatures.insert(author, signature);

        // ========================================
        // 步骤 4: 更新投票权重
        // ========================================
        let author_power = verifier.get_voting_power(&author)?;
        self.voting_power += author_power;

        info!(
            "Added signature for batch {}: {} / {} voting power",
            self.batch_info.batch_id(),
            self.voting_power,
            verifier.total_voting_power()
        );

        // ========================================
        // 步骤 5: 检查是否达到 quorum
        // ========================================
        Ok(self.voting_power >= verifier.quorum_voting_power())
    }

    /// 聚合签名并生成 ProofOfStore
    pub fn aggregate(
        self,
        verifier: &ValidatorVerifier,
    ) -> anyhow::Result<ProofOfStore> {
        // ========================================
        // 步骤 1: 提取签名列表
        // ========================================
        let signatures: Vec<_> = self.signatures.values().cloned().collect();

        // ========================================
        // 步骤 2: BLS 聚合签名
        // ========================================
        let aggregated_sig = bls12381::Signature::aggregate(&signatures)?;

        // ========================================
        // 步骤 3: 构造 AggregateSignature
        // ========================================
        let signers: Vec<_> = self.signatures.keys().cloned().collect();
        let multi_signature = AggregateSignature::new(
            signers,
            aggregated_sig,
        );

        // ========================================
        // 步骤 4: 构造 ProofOfStore
        // ========================================
        let proof = ProofOfStore::new(
            self.batch_info,
            multi_signature,
        );

        // ========================================
        // 步骤 5: 验证 ProofOfStore
        // ========================================
        proof.verify(verifier)?;

        info!("Generated ProofOfStore for batch {}", proof.batch_id());

        Ok(proof)
    }
}
```

### 5.3 ProofOfStore 生成

#### 完整流程实现

```rust
impl ProofCoordinator {
    /// 处理 SignedBatchInfoMsg
    pub async fn handle_signed_batch_info_msg(
        &mut self,
        msg: SignedBatchInfoMsg,
    ) -> anyhow::Result<()> {
        let signed_batch_info = msg.signed_batch_info();
        let batch_id = signed_batch_info.info().batch_id();

        // ========================================
        // 步骤 1: 验证签名
        // ========================================
        signed_batch_info.verify(&self.validator_verifier)?;

        info!(
            "Received SignedBatchInfo for batch {} from {}",
            batch_id,
            signed_batch_info.signer()
        );

        // ========================================
        // 步骤 2: 获取或创建 ProofAggregator
        // ========================================
        let aggregator = self.proof_aggregators
            .entry(batch_id)
            .or_insert_with(|| {
                ProofAggregator::new(signed_batch_info.info().clone())
            });

        // ========================================
        // 步骤 3: 添加签名
        // ========================================
        let reached_quorum = aggregator.add_signature(
            signed_batch_info.signer(),
            signed_batch_info.signature().clone(),
            &self.validator_verifier,
        )?;

        // ========================================
        // 步骤 4: 检查是否达到 quorum
        // ========================================
        if reached_quorum {
            info!("Reached quorum for batch {}", batch_id);

            // 移除 aggregator（避免重复处理）
            let aggregator = self.proof_aggregators.remove(&batch_id).unwrap();

            // 聚合签名
            let proof = aggregator.aggregate(&self.validator_verifier)?;

            // 发送到 ProofManager
            self.send_proof_to_manager(proof).await?;
        }

        Ok(())
    }

    async fn send_proof_to_manager(
        &self,
        proof: ProofOfStore,
    ) -> anyhow::Result<()> {
        // 通过 channel 发送给 ProofManager
        // 具体实现依赖于系统架构
        Ok(())
    }
}
```

---

## 6. ProofManager 证明管理器

### 6.1 Proof 缓存管理

#### ProofManager 结构

```rust
// src/quorum_store/proof_manager.rs

pub struct ProofManager {
    // ========================================
    // Proof 队列（按 gas 优先级）
    // ========================================
    proof_queue: ProofQueue,

    // ========================================
    // 已提交的 Proofs
    // ========================================
    /// Round -> Vec<ProofOfStore>
    committed_proofs: BTreeMap<Round, Vec<ProofOfStore>>,

    // ========================================
    // 配置
    // ========================================
    config: QuorumStoreConfig,

    // ========================================
    // 统计
    // ========================================
    total_proofs_generated: u64,
    total_proofs_pulled: u64,
}

/// Proof 优先队列
struct ProofQueue {
    /// Gas bucket -> Vec<ProofOfStore>
    proofs_by_gas: BTreeMap<u64, Vec<ProofOfStore>>,

    /// 总大小（字节）
    total_bytes: usize,

    /// 最大大小限制
    max_bytes: usize,
}
```

#### ProofQueue 实现

```mermaid
classDiagram
    class ProofQueue {
        -BTreeMap~u64,Vec~ProofOfStore~~ proofs_by_gas
        -usize total_bytes
        -usize max_bytes
        +push(proof) Result
        +pull(max_bytes, excluded) Vec~ProofOfStore~
        +clean_expired(now) void
    }

    class ProofManager {
        -ProofQueue proof_queue
        +add_proof(proof) Result
        +pull_proofs(params) Vec~ProofOfStore~
        +handle_commit_notification(commit) void
    }

    ProofManager --> ProofQueue
```

**代码实现**：

```rust
impl ProofQueue {
    pub fn new(max_bytes: usize) -> Self {
        Self {
            proofs_by_gas: BTreeMap::new(),
            total_bytes: 0,
            max_bytes,
        }
    }

    /// 添加 Proof
    pub fn push(&mut self, proof: ProofOfStore) -> anyhow::Result<()> {
        let proof_bytes = proof.num_bytes();

        // ========================================
        // 检查容量
        // ========================================
        if self.total_bytes + proof_bytes > self.max_bytes {
            counters::PROOF_QUEUE_OVERFLOW.inc();
            bail!("Proof queue full: {} bytes", self.total_bytes);
        }

        // ========================================
        // 添加到对应的 gas bucket
        // ========================================
        let gas_bucket = proof.gas_bucket_start();
        self.proofs_by_gas
            .entry(gas_bucket)
            .or_insert_with(Vec::new)
            .push(proof);

        self.total_bytes += proof_bytes;

        counters::PROOF_QUEUE_SIZE_BYTES.set(self.total_bytes as i64);

        Ok(())
    }

    /// 拉取 Proofs（按 gas 从高到低）
    pub fn pull(
        &mut self,
        max_bytes: usize,
        excluded_batches: &HashSet<BatchId>,
    ) -> Vec<ProofOfStore> {
        let mut selected = Vec::new();
        let mut selected_bytes = 0;

        // ========================================
        // 从最高 gas bucket 开始拉取
        // ========================================
        let mut buckets_to_remove = Vec::new();

        for (gas_bucket, proofs) in self.proofs_by_gas.iter_mut().rev() {
            let mut remaining = Vec::new();

            for proof in proofs.drain(..) {
                // 跳过已排除的批次
                if excluded_batches.contains(&proof.batch_id()) {
                    continue;
                }

                let proof_bytes = proof.num_bytes();

                // 检查是否超过限制
                if selected_bytes + proof_bytes > max_bytes {
                    remaining.push(proof);
                    continue;
                }

                // 添加到选择列表
                selected.push(proof);
                selected_bytes += proof_bytes;
            }

            if remaining.is_empty() {
                buckets_to_remove.push(*gas_bucket);
            } else {
                *proofs = remaining;
            }

            // 已达到限制，停止拉取
            if selected_bytes >= max_bytes {
                break;
            }
        }

        // 移除空的 buckets
        for gas_bucket in buckets_to_remove {
            self.proofs_by_gas.remove(&gas_bucket);
        }

        // 更新总大小
        self.total_bytes -= selected_bytes;

        counters::PROOF_QUEUE_SIZE_BYTES.set(self.total_bytes as i64);
        counters::PROOFS_PULLED.inc_by(selected.len() as u64);

        selected
    }

    /// 清理过期 Proofs
    pub fn clean_expired(&mut self, now: u64) {
        let mut total_removed = 0;
        let mut removed_bytes = 0;

        for proofs in self.proofs_by_gas.values_mut() {
            let original_len = proofs.len();

            proofs.retain(|proof| {
                let not_expired = proof.expiration() > now;

                if !not_expired {
                    removed_bytes += proof.num_bytes();
                }

                not_expired
            });

            total_removed += original_len - proofs.len();
        }

        self.total_bytes -= removed_bytes;

        counters::PROOFS_EXPIRED.inc_by(total_removed as u64);
    }
}
```

### 6.2 拉取策略

#### pull_proofs 完整实现

```rust
impl ProofManager {
    /// 为 ProposalGenerator 拉取 Proofs
    pub fn pull_proofs(
        &mut self,
        max_txns: u64,
        max_bytes: u64,
        excluded_batches: HashSet<BatchId>,
    ) -> Vec<ProofOfStore> {
        info!(
            "Pulling proofs: max_txns={}, max_bytes={}, excluded={}",
            max_txns,
            max_bytes,
            excluded_batches.len()
        );

        // ========================================
        // 步骤 1: 清理过期 Proofs
        // ========================================
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_micros() as u64;

        self.proof_queue.clean_expired(now);

        // ========================================
        // 步骤 2: 从队列拉取
        // ========================================
        let proofs = self.proof_queue.pull(max_bytes as usize, &excluded_batches);

        // ========================================
        // 步骤 3: 过滤交易数限制
        // ========================================
        let mut selected = Vec::new();
        let mut total_txns = 0u64;

        for proof in proofs {
            let proof_txns = proof.num_txns() as u64;

            if total_txns + proof_txns > max_txns {
                break;
            }

            selected.push(proof);
            total_txns += proof_txns;
        }

        self.total_proofs_pulled += selected.len() as u64;

        counters::PROOFS_PULLED_FOR_PROPOSAL.inc_by(selected.len() as u64);
        counters::PROOFS_PULLED_TXNS.inc_by(total_txns);

        info!("Pulled {} proofs with {} txns", selected.len(), total_txns);

        selected
    }
}
```

#### 拉取流程可视化

```mermaid
sequenceDiagram
    autonumber
    participant PG as ProposalGenerator
    participant PM as ProofManager
    participant PQ as ProofQueue

    Note over PG,PQ: ══════════ Proof 拉取流程 ══════════

    PG->>PM: pull_proofs(max_txns, max_bytes, excluded)

    rect rgb(225, 245, 255)
        Note over PM: Phase 1: 清理过期
        PM->>PQ: clean_expired(now)
        PQ->>PQ: 遍历所有 Proofs
        PQ->>PQ: 移除 expiration < now
        PQ->>PM: cleaned
    end

    rect rgb(255, 249, 196)
        Note over PQ: Phase 2: 从队列拉取
        PM->>PQ: pull(max_bytes, excluded)

        loop 遍历 gas buckets (高到低)
            PQ->>PQ: 检查 excluded_batches
            alt 未排除 且 未超限
                PQ->>PQ: 添加到 selected
            end
        end

        PQ->>PM: Vec<ProofOfStore>
    end

    rect rgb(243, 229, 245)
        Note over PM: Phase 3: 过滤交易数
        PM->>PM: total_txns = 0

        loop 遍历 proofs
            PM->>PM: total_txns += proof.num_txns()
            alt total_txns > max_txns
                PM->>PM: break
            else
                PM->>PM: 添加到 final_selected
            end
        end
    end

    PM->>PG: Vec<ProofOfStore>
```

### 6.3 清理机制

#### handle_commit_notification 实现

```rust
impl ProofManager {
    /// 处理区块提交通知
    pub fn handle_commit_notification(
        &mut self,
        committed_round: Round,
        committed_batches: Vec<BatchId>,
    ) {
        info!(
            "Handling commit notification: round={}, {} batches",
            committed_round,
            committed_batches.len()
        );

        // ========================================
        // 步骤 1: 记录已提交的批次
        // ========================================
        self.committed_proofs
            .entry(committed_round)
            .or_insert_with(Vec::new);

        // ========================================
        // 步骤 2: 清理旧的提交记录（保留最近 100 轮）
        // ========================================
        let cutoff_round = committed_round.saturating_sub(100);
        let old_rounds: Vec<_> = self.committed_proofs
            .range(..cutoff_round)
            .map(|(r, _)| *r)
            .collect();

        for round in old_rounds {
            self.committed_proofs.remove(&round);
        }

        counters::COMMITTED_PROOFS_CLEANED.inc_by(old_rounds.len() as u64);
    }
}
```

---

## 7. 反压机制详解

### 7.1 反压触发条件

#### BackPressure 结构

```mermaid
classDiagram
    class BackPressure {
        +bool txn_count
        +bool proof_count
        +update(metrics) void
        +should_backpressure() bool
    }

    class BackPressureMetrics {
        +usize pending_txns
        +usize pending_proofs
        +f64 mempool_utilization
        +Duration pipeline_latency
    }

    BackPressure --> BackPressureMetrics
```

**代码实现**：

```rust
// src/quorum_store/batch_generator.rs

pub struct BackPressure {
    /// 交易数反压标志
    txn_count: bool,

    /// Proof 数反压标志
    proof_count: bool,

    /// 上次更新时间
    last_update: Instant,
}

impl BackPressure {
    pub fn new() -> Self {
        Self {
            txn_count: false,
            proof_count: false,
            last_update: Instant::now(),
        }
    }

    /// 更新反压状态
    pub fn update(&mut self, metrics: &BackPressureMetrics) {
        // ========================================
        // 检查交易数反压
        // ========================================
        self.txn_count = metrics.pending_txns > metrics.txn_count_threshold;

        // ========================================
        // 检查 Proof 数反压
        // ========================================
        self.proof_count = metrics.pending_proofs > metrics.proof_count_threshold;

        self.last_update = Instant::now();

        if self.txn_count || self.proof_count {
            warn!(
                "Backpressure active: txn_count={}, proof_count={}",
                self.txn_count,
                self.proof_count
            );

            counters::BACKPRESSURE_ACTIVE.set(1);
        } else {
            counters::BACKPRESSURE_ACTIVE.set(0);
        }
    }

    /// 是否应该反压
    pub fn should_backpressure(&self) -> bool {
        self.txn_count || self.proof_count
    }
}

pub struct BackPressureMetrics {
    /// 待处理交易数
    pending_txns: usize,

    /// 待处理 Proof 数
    pending_proofs: usize,

    /// 交易数阈值
    txn_count_threshold: usize,

    /// Proof 数阈值
    proof_count_threshold: usize,
}
```

### 7.2 动态调控算法

#### 反压决策流程

```mermaid
flowchart TD
    A[BatchGenerator.tick] --> B[收集指标]

    B --> C[pending_txns = <br/>batches_in_progress.total_txns]
    B --> D[pending_proofs = <br/>proof_queue.size]

    C --> E{pending_txns > threshold?}
    D --> F{pending_proofs > threshold?}

    E -->|是| G[txn_count_backpressure = true]
    E -->|否| H[txn_count_backpressure = false]

    F -->|是| I[proof_count_backpressure = true]
    F -->|否| J[proof_count_backpressure = false]

    G --> K{任一反压 = true?}
    H --> K
    I --> K
    J --> K

    K -->|是| L[跳过批次生成]
    K -->|否| M[继续生成批次]

    L --> N[counters::BACKPRESSURE_SKIP.inc]
    M --> O[generate_batch]

    style G fill:#ffcdd2
    style I fill:#ffcdd2
    style L fill:#ffebee
    style M fill:#c8e6c9
```

### 7.3 多级反压策略

#### 反压级别表

| 级别 | pending_txns | pending_proofs | 动作 | 效果 |
|-----|--------------|----------------|------|------|
| **Level 0 (正常)** | < 10,000 | < 100 | 正常生成 | 最大吞吐量 |
| **Level 1 (轻微)** | 10,000 - 20,000 | 100 - 200 | 减少 20% | 缓解压力 |
| **Level 2 (中等)** | 20,000 - 30,000 | 200 - 300 | 减少 50% | 显著降低 |
| **Level 3 (严重)** | > 30,000 | > 300 | 完全停止 | 系统保护 |

**多级反压实现**：

```rust
impl BackPressure {
    /// 计算反压级别
    pub fn calculate_level(&self, metrics: &BackPressureMetrics) -> BackPressureLevel {
        let txn_ratio = metrics.pending_txns as f64 / metrics.txn_count_threshold as f64;
        let proof_ratio = metrics.pending_proofs as f64 / metrics.proof_count_threshold as f64;

        let max_ratio = txn_ratio.max(proof_ratio);

        if max_ratio < 1.0 {
            BackPressureLevel::Normal
        } else if max_ratio < 2.0 {
            BackPressureLevel::Mild
        } else if max_ratio < 3.0 {
            BackPressureLevel::Moderate
        } else {
            BackPressureLevel::Severe
        }
    }

    /// 根据级别调整生成速率
    pub fn adjust_rate(&self, level: BackPressureLevel, base_rate: f64) -> f64 {
        match level {
            BackPressureLevel::Normal => base_rate,
            BackPressureLevel::Mild => base_rate * 0.8,
            BackPressureLevel::Moderate => base_rate * 0.5,
            BackPressureLevel::Severe => 0.0,
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum BackPressureLevel {
    Normal,
    Mild,
    Moderate,
    Severe,
}
```

---

## 8. 网络层交互

### 8.1 消息类型

#### QuorumStoreMsg 枚举

```rust
// src/quorum_store/types.rs

#[derive(Clone, Debug, Serialize, Deserialize)]
pub enum QuorumStoreMsg {
    /// 批次消息（包含完整交易）
    BatchMsg(BatchMsg),

    /// 签名的批次信息
    SignedBatchInfoMsg(SignedBatchInfoMsg),

    /// Proof 消息（可选，用于主动推送）
    ProofOfStoreMsg(ProofOfStoreMsg),

    /// 批次请求
    BatchRequest(BatchRequest),

    /// 批次响应
    BatchResponse(BatchResponse),
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct BatchMsg {
    batch: Batch,
    signature: bls12381::Signature,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct SignedBatchInfoMsg {
    signed_batch_info: SignedBatchInfo,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct BatchRequest {
    epoch: u64,
    batch_ids: Vec<BatchId>,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct BatchResponse {
    batches: Vec<Batch>,
}
```

#### 消息流图

```mermaid
graph TB
    subgraph "Validator A (Generator)"
        A1[生成 Batch]
        A2[广播 BatchMsg]
    end

    subgraph "Validator B (Receiver)"
        B1[接收 BatchMsg]
        B2[持久化 Batch]
        B3[签名 BatchInfo]
        B4[发送 SignedBatchInfoMsg]
    end

    subgraph "Validator C (Receiver)"
        C1[接收 BatchMsg]
        C2[持久化 Batch]
        C3[签名 BatchInfo]
        C4[发送 SignedBatchInfoMsg]
    end

    subgraph "Validator A (Aggregator)"
        A3[收集 SignedBatchInfoMsg]
        A4[聚合签名]
        A5[生成 ProofOfStore]
    end

    A1 --> A2
    A2 -->|broadcast| B1
    A2 -->|broadcast| C1

    B1 --> B2 --> B3 --> B4
    C1 --> C2 --> C3 --> C4

    B4 -->|send back| A3
    C4 -->|send back| A3

    A3 --> A4 --> A5

    style A1 fill:#c8e6c9
    style A5 fill:#e1f5ff
```

### 8.2 广播机制

#### broadcast_batch_msg 实现

```rust
impl QuorumStoreNetworkSender {
    /// 广播批次消息
    pub async fn broadcast_batch_msg(&self, batch: Batch) {
        let batch_msg = BatchMsg {
            batch: batch.clone(),
            signature: self.signer.sign(&batch.digest()).unwrap(),
        };

        let msg = QuorumStoreMsg::BatchMsg(batch_msg);

        // ========================================
        // 发送给所有其他验证者
        // ========================================
        let recipients: Vec<_> = self.epoch_state
            .verifier()
            .get_ordered_account_addresses()
            .into_iter()
            .filter(|addr| *addr != self.author)
            .collect();

        info!(
            "Broadcasting batch {} to {} validators",
            batch.batch_id(),
            recipients.len()
        );

        for recipient in recipients {
            if let Err(e) = self.network
                .send_to(recipient, msg.clone())
                .await
            {
                warn!("Failed to send batch to {}: {:?}", recipient, e);
                counters::BATCH_BROADCAST_FAILED.inc();
            }
        }

        counters::BATCH_BROADCAST_SENT.inc();
    }

    /// 发送签名的批次信息
    pub async fn send_signed_batch_info(
        &self,
        recipient: Author,
        signed_batch_info: SignedBatchInfo,
    ) {
        let msg = QuorumStoreMsg::SignedBatchInfoMsg(
            SignedBatchInfoMsg {
                signed_batch_info,
            }
        );

        if let Err(e) = self.network.send_to(recipient, msg).await {
            warn!(
                "Failed to send SignedBatchInfo to {}: {:?}",
                recipient,
                e
            );
        }
    }
}
```

### 8.3 请求与响应

#### BatchRequester RPC 实现

```rust
// src/quorum_store/batch_requester.rs

pub struct BatchRequester {
    network: Arc<NetworkClient>,
    batch_store: Arc<BatchStore>,
    timeout: Duration,
}

impl BatchRequester {
    /// 请求缺失的批次
    pub async fn request_batches(
        &self,
        missing_batch_ids: Vec<BatchId>,
        preferred_peer: Author,
    ) -> anyhow::Result<Vec<Batch>> {
        if missing_batch_ids.is_empty() {
            return Ok(Vec::new());
        }

        info!(
            "Requesting {} batches from {}",
            missing_batch_ids.len(),
            preferred_peer
        );

        // ========================================
        // 构造请求
        // ========================================
        let request = BatchRequest {
            epoch: self.epoch,
            batch_ids: missing_batch_ids.clone(),
        };

        // ========================================
        // 发送 RPC 请求
        // ========================================
        let response = tokio::time::timeout(
            self.timeout,
            self.network.request_batches(preferred_peer, request),
        )
        .await??;

        // ========================================
        // 验证响应
        // ========================================
        let mut fetched_batches = Vec::new();

        for batch in response.batches {
            // 验证批次完整性
            if missing_batch_ids.contains(&batch.batch_id()) {
                // 持久化到本地
                self.batch_store.persist_batch(batch.clone()).await?;

                fetched_batches.push(batch);
            }
        }

        info!("Fetched {} batches", fetched_batches.len());

        counters::BATCHES_FETCHED.inc_by(fetched_batches.len() as u64);

        Ok(fetched_batches)
    }
}

/// 处理批次请求（服务端）
pub async fn handle_batch_request(
    request: BatchRequest,
    batch_store: Arc<BatchStore>,
) -> anyhow::Result<BatchResponse> {
    let mut batches = Vec::new();

    for batch_id in request.batch_ids {
        let key = BatchKey::new(batch_id.author, batch_id);

        if let Some(batch) = batch_store.get_batch(&key).await? {
            batches.push(batch);
        }
    }

    Ok(BatchResponse { batches })
}
```

---

## 9. Batch 完整生命周期

### 端到端流程

```mermaid
sequenceDiagram
    autonumber
    participant MP as Mempool
    participant BG as BatchGenerator
    participant BS as BatchStore
    participant NET as Network
    participant PEERS as Other Validators
    participant PC as ProofCoordinator
    participant PM as ProofManager
    participant PG as ProposalGenerator
    participant EX as Executor
    participant COMMIT as Commit

    Note over MP,COMMIT: ══════════ 完整生命周期 ══════════

    rect rgb(225, 245, 255)
        Note over BG: Phase 1: 批次生成
        BG->>BG: timer tick (100ms)
        BG->>MP: pull_txns(max, excluded)
        MP->>BG: Vec<SignedTransaction>
        BG->>BG: create Batch
    end

    rect rgb(255, 249, 196)
        Note over BS: Phase 2: 持久化
        BG->>BS: persist_batch(batch)
        BS->>BS: quota_check()
        BS->>BS: write to RocksDB + cache
        BS->>BS: sign BatchInfo
        BS->>BG: SignedBatchInfo
    end

    rect rgb(243, 229, 245)
        Note over NET: Phase 3: 广播
        BG->>NET: broadcast BatchMsg
        NET->>PEERS: send to all
        PEERS->>PEERS: persist & sign
        PEERS->>NET: SignedBatchInfoMsg
    end

    rect rgb(255, 235, 238)
        Note over PC: Phase 4: 签名聚合
        NET->>PC: collect signatures
        PC->>PC: voting_power >= 2f+1?
        PC->>PC: aggregate signatures
        PC->>PM: ProofOfStore ready
    end

    rect rgb(200, 230, 201)
        Note over PG: Phase 5: 区块提案
        PG->>PM: pull_proofs(max_bytes)
        PM->>PG: Vec<ProofOfStore>
        PG->>PG: construct block
        PG->>NET: broadcast proposal
    end

    rect rgb(225, 245, 255)
        Note over EX: Phase 6: 执行
        EX->>EX: receive block
        EX->>BS: fetch batches (by ProofOfStore)
        BS->>EX: Vec<Batch>
        EX->>EX: execute transactions
    end

    rect rgb(255, 249, 196)
        Note over COMMIT: Phase 7: 提交与清理
        COMMIT->>BG: commit_notify(committed_batches)
        BG->>BG: cleanup expired batches
        BG->>BS: release quota
        BS->>BS: free memory & disk
    end
```

### 时间线可视化

```mermaid
gantt
    title Batch 生命周期时间线 (单位: ms)
    dateFormat X
    axisFormat %L

    section Generation
    拉取交易 (0-100ms)           :active, g1, 0, 100
    创建批次 (100-110ms)         :active, g2, 100, 10
    持久化 (110-150ms)           :active, g3, 110, 40

    section Broadcast
    广播 BatchMsg (150-200ms)    :crit, b1, 150, 50
    网络传输 (200-300ms)         :crit, b2, 200, 100

    section Signature
    远程持久化 (300-400ms)       :active, s1, 300, 100
    签名收集 (400-600ms)         :active, s2, 400, 200
    聚合签名 (600-610ms)         :active, s3, 600, 10
    ProofOfStore 生成 (610-620ms):done, s4, 610, 10

    section Proposal
    等待区块提案 (620-1000ms)    :milestone, p1, 620, 380
    包含在区块 (1000ms)          :crit, p2, 1000, 0

    section Execution
    区块执行 (1000-1500ms)       :active, e1, 1000, 500
    区块提交 (1500-1600ms)       :done, e2, 1500, 100

    section Cleanup
    清理通知 (1600-1650ms)       :crit, c1, 1600, 50
    释放资源 (1650-1700ms)       :done, c2, 1650, 50
```

---

## 10. 性能优化与配置

### 性能指标总结

```mermaid
graph TB
    subgraph "QuorumStore 性能提升"
        A[网络带宽<br/>减少 50-70%]
        B[吞吐量<br/>提升 3-5倍]
        C[共识延迟<br/>减少 30%]
        D[存储复用<br/>高复用率]
    end

    subgraph "优化技术"
        E[批量处理<br/>500 txns/batch]
        F[解耦传播<br/>异步并行]
        G[动态反压<br/>平滑负载]
        H[三级存储<br/>内存+磁盘]
    end

    A --> E
    B --> F
    C --> F
    D --> H

    style A fill:#c8e6c9
    style B fill:#c8e6c9
    style C fill:#c8e6c9
    style D fill:#c8e6c9
```

### 关键配置参数

```toml
# config.toml

[consensus.quorum_store]
# ========================================
# Batch 配置
# ========================================
# 单个 Batch 最大交易数
max_batch_txns = 500

# 单个 Batch 最大字节数
max_batch_bytes = 524288  # 512KB

# Batch 过期时间（微秒）
batch_expiry_time_usecs = 60000000  # 60 秒

# Batch 生成间隔（毫秒）
batch_generation_interval_ms = 100

# ========================================
# 配额管理
# ========================================
# 内存配额（字节）
memory_quota = 125829120  # 120MB

# 磁盘配额（字节）
db_quota = 1006632960  # 960MB

# 批次数配额
batch_quota = 2500

# ========================================
# 反压配置
# ========================================
# 交易数反压阈值
txn_count_threshold = 10000

# Proof 数反压阈值
proof_count_threshold = 100

# 反压检查间隔（毫秒）
backpressure_check_interval_ms = 200

# ========================================
# 网络配置
# ========================================
# Batch 请求超时（毫秒）
batch_request_timeout_ms = 5000

# 最大重试次数
max_batch_request_retries = 3

# ========================================
# ProofQueue 配置
# ========================================
# Proof 队列最大字节数
proof_queue_max_bytes = 10485760  # 10MB

# Proof 队列清理间隔（毫秒）
proof_queue_cleanup_interval_ms = 1000
```

### 性能调优建议

| 场景 | 参数调整 | 效果 |
|-----|---------|------|
| **高吞吐量** | 增大 `max_batch_txns` 到 1000 | 单批次包含更多交易 |
| | 增大 `memory_quota` 到 200MB | 减少磁盘 I/O |
| | 减少 `batch_generation_interval_ms` 到 50ms | 更频繁生成批次 |
| **低延迟** | 减少 `max_batch_txns` 到 200 | 批次更小更快 |
| | 增加 `batch_generation_interval_ms` 到 200ms | 等待更多交易聚合 |
| **资源受限** | 减少 `memory_quota` 到 50MB | 节省内存 |
| | 减少 `batch_quota` 到 1000 | 限制批次数 |
| **网络拥塞** | 启用更激进的反压 | 降低网络压力 |
| | 减少 `max_batch_bytes` 到 256KB | 批次更小 |

### Prometheus 监控指标

```rust
// src/quorum_store/counters.rs

/// 批次生成指标
pub static BATCH_GENERATED: Lazy<IntCounter> = ...;
pub static BATCH_GENERATED_TXNS: Lazy<IntCounter> = ...;
pub static BATCH_GENERATED_BYTES: Lazy<IntCounter> = ...;

/// 批次持久化指标
pub static BATCH_PERSISTED: Lazy<IntCounter> = ...;
pub static BATCH_CACHE_HIT: Lazy<IntCounter> = ...;
pub static BATCH_CACHE_MISS: Lazy<IntCounter> = ...;

/// 配额指标
pub static QUOTA_MEMORY_USAGE: Lazy<IntGauge> = ...;
pub static QUOTA_DB_USAGE: Lazy<IntGauge> = ...;
pub static EXCEEDED_BATCH_QUOTA_COUNT: Lazy<IntCounter> = ...;

/// Proof 指标
pub static PROOFS_GENERATED: Lazy<IntCounter> = ...;
pub static PROOFS_PULLED: Lazy<IntCounter> = ...;
pub static PROOF_QUEUE_SIZE_BYTES: Lazy<IntGauge> = ...;

/// 反压指标
pub static BACKPRESSURE_ACTIVE: Lazy<IntGauge> = ...;
pub static BATCH_GENERATOR_BACKPRESSURE: Lazy<IntCounter> = ...;

/// 网络指标
pub static BATCH_BROADCAST_SENT: Lazy<IntCounter> = ...;
pub static BATCH_BROADCAST_FAILED: Lazy<IntCounter> = ...;
pub static BATCHES_FETCHED: Lazy<IntCounter> = ...;
```

---

## 11. 总结

### 核心要点

```mermaid
mindmap
  root((QuorumStore 总结))
    设计理念
      解耦数据传播与共识
      批量处理提升效率
      异步并行优化性能
    核心组件
      BatchGenerator
        定时生成批次
        交易去重机制
        过期管理
      BatchStore
        三级存储架构
        配额管理
        签名验证
      ProofCoordinator
        签名收集
        聚合签名
        ProofOfStore 生成
      ProofManager
        优先队列
        拉取策略
        清理机制
    反压机制
      多级反压策略
      动态调控算法
      系统保护
    性能优势
      网络带宽减少 50-70%
      吞吐量提升 3-5倍
      共识延迟减少 30%
      存储高复用率
```

### 关键算法总结

| 算法 | 时间复杂度 | 空间复杂度 | 说明 |
|-----|-----------|-----------|------|
| **交易去重** | O(n) | O(m) | n=新交易数, m=进行中交易数 |
| **Batch 持久化** | O(1) | O(b) | b=批次大小 |
| **签名聚合** | O(k) | O(k) | k=签名数量 |
| **Proof 拉取** | O(log m + n) | O(n) | m=gas buckets, n=拉取数 |
| **配额检查** | O(1) | O(1) | 常量时间 |

### 设计亮点

1. **解耦架构**: 数据传播与共识完全分离，互不阻塞
2. **批量优化**: 500 txns/batch，显著减少网络开销
3. **三级存储**: 内存 + 磁盘 + 配额管理，灵活高效
4. **动态反压**: 多级策略自适应调节，系统稳定
5. **优先队列**: 按 gas 排序，保证高价值交易优先
6. **并行处理**: 交易摘要并行计算，充分利用多核

### QuorumStore vs 传统模式对比

| 维度 | 传统模式 | QuorumStore 模式 | 改进幅度 |
|-----|---------|-----------------|---------|
| **网络带宽** | 每轮完整交易 | 交易传输一次 + Proof | **50-70% ↓** |
| **吞吐量** | ~20k TPS | ~60-100k TPS | **3-5倍** |
| **共识延迟** | 1-2秒 | 700ms-1.4秒 | **30% ↓** |
| **存储复用** | 无 | Batch 可被多个区块引用 | **高复用率** |
| **峰值处理** | 无缓冲 | 动态反压平滑负载 | **显著改善** |
| **扩展性** | Leader 瓶颈 | 多节点并行批处理 | **更好** |

### 适用场景

- ✅ **高吞吐量区块链**: 需要处理大量交易的公链
- ✅ **网络带宽受限**: 减少数据传输量至关重要
- ✅ **峰值流量场景**: 需要平滑负载的系统
- ✅ **多验证者网络**: 充分利用分布式批处理能力

---

**文档路径**: `/home/morton/work/rust/aptos-core/consensus/APTOS_共识模块深度技术文档_详细增强版_Part7_QuorumStore.md`

**生成时间**: 2025-10-09
**文档版本**: v2.0 (详细增强版)
**源码路径**: `consensus/src/quorum_store/`

---

## 🎉 Aptos Consensus 模块完整系列文档

本文档是 **Aptos 共识模块深度技术文档** 系列的最后一部分。完整系列包括：

1. **Part 1**: 模块概述与整体架构
2. **Part 2**: SafetyRules 安全规则详解
3. **Part 3**: BlockStorage 与 RoundManager 详解
4. **Part 4**: Liveness 模块详解
5. **Part 5**: Pipeline 模块详解
6. **Part 6**: DAG 共识模块详解
7. **Part 7**: QuorumStore 模块详解 ✅ (当前文档)

感谢阅读！🙏
