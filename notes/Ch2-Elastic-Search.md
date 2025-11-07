# 🧱 Elasticsearch 的 Write-Ahead Log（Translog）学习笔记

## 🧩 一、DDIA 视角下的存储抽象层次

在《Designing Data-Intensive Applications（DDIA）》中，  
持久化与一致性机制可以分为如下抽象层次：

| 层次 | 关键概念 | 典型组件 |
|------|-----------|-----------|
| 1️⃣ **Commit Log / WAL** | 记录操作意图（append-only），保证崩溃恢复 | Kafka、RocksDB WAL、Postgres WAL、**Elasticsearch Translog** |
| 2️⃣ **In-memory Buffer / MemTable** | 暂存最近写入数据，用于快速查询与批量刷盘 | Lucene RAM Buffer, MemTable |
| 3️⃣ **Segment File / SSTable** | 不可变排序文件，用于合并与压缩 | Lucene Segment, SSTable |
| 4️⃣ **Compaction / Merge** | 周期性合并旧文件以清理过期项 | Lucene MergePolicy, LSM Compaction |
| 5️⃣ **Index Metadata / Checkpoint** | 标记一致性边界，方便恢复 | ES Commit Point, Checkpoint File |

> Elasticsearch 几乎完全符合这套结构，只是在 Lucene 之上又包了一层 **分布式协调与复制逻辑**。

---

## 🧠 二、Elasticsearch 的 WAL：Translog

在 Elasticsearch 中，Write-Ahead Log 的角色由 **Translog（Transaction Log）** 实现。

### 🔹 核心职责

- **持久化写入请求（WAL 功能）**
  - 每当一个 document 被索引（index / update / delete）时：
    - 写入内存 buffer（in-memory index buffer）；
    - 同时将操作记录写入 translog。

- **崩溃恢复**
  - 如果节点宕机，重启时从 translog 读取未 flush 的操作重放。

- **异步落盘优化**
  - 由 `index.translog.durability` 控制：
    - `"request"`（默认） → 每个请求 fsync；
    - `"async"` → 定期批量 fsync；
    - `"flush"` → checkpoint 后清空旧 translog。

---

## ⚙️ 三、Elasticsearch 写入流程（结合 Lucene）

```text
Client Request
    ↓
Primary Shard
    ↓
Index Buffer (in-memory, Lucene RAM buffer)
    ↓
Translog (append-only WAL)
    ↓
Lucene Segment (flush 时生成新 segment)
    ↓
Commit Point (记录一致性快照)
```
### 🔹 步骤分解
| 阶段  | 操作描述                                                    |
| --- | ------------------------------------------------------- |
| 1️⃣ | 写入请求到达 **Primary Shard**，首先进入 **Lucene RAM buffer**     |
| 2️⃣ | 同时生成一条操作日志写入 **Translog 文件（append-only）**               |
| 3️⃣ | Buffer 满或定时执行 **flush**：写出新的 **Lucene Segment 文件（不可变）** |
| 4️⃣ | Flush 成功后生成新的 **Commit Point** 并截断旧的 translog           |
| 5️⃣ | 同步至 Replica Shard，由 Replica 重放同样操作                      |

---

## 💾 四、Translog 的物理结构

每个分片（Shard）都有独立的 translog 目录：
```
/data/nodes/0/indices/<index_uuid>/<shard_id>/translog/
    ├── translog-123456.tlog
    ├── translog-123457.ckp
    └── translog-generation
```
| 文件                    | 作用                          |
| --------------------- | --------------------------- |
| `.tlog`               | 主体日志文件（append-only），包含操作记录  |
| `.ckp`                | Checkpoint 文件，标记已 fsync 的位置 |
| `translog-generation` | 当前 translog 序号              |


**日志 Entry 格式：**
  - Operation type（index / delete / no-op）
  - seq_no（全局顺序号）
  - docID
  - source data（原始 JSON）
  - version

---

## 🔄 五、与 Lucene Segment 的配合机制

Lucene 的数据结构是 **immutable segment**，但 Elasticsearch 追求 **近实时**（NRT）搜索，
因此采用双层机制：

| 阶段                    | 作用                                                                 |
| --------------------- | ------------------------------------------------------------------ |
| **refresh（默认 1s 一次）** | 将内存 buffer 数据转为新的 Lucene Segment（仅内存 / FS cache，不 fsync），实现 NRT 搜索 |
| **flush**             | 将 segment 落盘（fsync）、提交 commit point，并清理旧 translog，确保持久化            |

> 对应 DDIA 的两层持久化语义：
> - **log-first (WAL) → 崩溃一致性保障；**
> - **segment commit → 长期持久化保障。**

---

## 🧬 六、分布式一致性：Primary-Replica 复制与 WAL 传播

Elasticsearch 的分布式复制机制对应 DDIA 第 9 章 “Leader-Based Replication” 模型。

### 🔹 写入主分片（Primary）
1. Primary 写入 Translog
2. 生成 seq_no + primary_term
3. 转发操作到所有 Replica

### 🔹 Replica 重放日志（Replay）
1. Replica 节点写入自己的 translog
2. 执行相同的索引操作
3. 确认成功后返回 ack

### 🔹 Primary 等待所有副本确认
- 当达到 quorum（可配置） 的 ack 数量后，Primary 标记写入成功。
- 因此 Translog 既是本地 WAL，也是分布式复制的同步媒介。

---

### 🧱 七、DDIA 章节映射
| DDIA 章节     | 核心思想                               | Elasticsearch 对应          |
| ----------- | ---------------------------------- | ------------------------- |
| 第3章：存储与索引结构 | 日志结构存储 + SSTable                   | Lucene Segment + Translog |
| 第5章：复制      | 单主复制 + WAL 传播                      | Primary/Replica 写入流程      |
| 第6章：分区      | 分片（Shard） = 独立数据子集                 | 每个 shard 自带 WAL & Lucene  |
| 第7章：事务      | 操作序列号 + Term 确保顺序一致性               | seq_no + primary_term     |
| 第9章：一致性与复制  | Leader-based replication with logs | Translog = Leader log     |
| 第10章：批处理    | Compaction / Merge                 | Lucene MergePolicy        |


---

### 📊 八、系统对比表：PostgreSQL vs Elasticsearch vs DDIA
| 维度                     | PostgreSQL      | Elasticsearch (Lucene) | DDIA 抽象                       |
| ---------------------- | --------------- | ---------------------- | ----------------------------- |
| **WAL 名称**             | Write-Ahead Log | Translog               | Commit Log                    |
| **数据存储结构**             | Page-based Heap | Immutable Segments     | Log-Structured Storage        |
| **索引方式**               | B+Tree          | Inverted Index         | SSTable-like Sorted Structure |
| **Merge / Compaction** | Vacuum          | Segment Merge          | Compaction                    |
| **一致性模型**              | 单机 ACID         | Primary-Replica ACK    | Leader-based Replication      |
| **恢复机制**               | Redo/Undo Log   | Replay Translog        | Log Rebuild                   |


---

### 🔍 九、总结与洞察

Elasticsearch 的 Translog = DDIA 所说的 “Commit Log / Write-Ahead Log”。

它与 Lucene Segment 构成了 “日志 + 快照” 的双层模型：

| 层次                          | 职责            |
| --------------------------- | ------------- |
| 🧾 **Translog (日志层)**       | 提供崩溃可恢复性与复制保障 |
| 📚 **Lucene Segment (快照层)** | 提供高效可搜索性与压缩能力 |


这种架构融合了：
- 高写入吞吐（append-only）
- 快速恢复（log replay）
- 高查询性能（segment-based inverted index）

是典型的 LSM + Log Replication 混合体系，
完美印证了 DDIA 中的存储与复制设计原则。

---

### 📖 参考资料
-《Designing Data-Intensive Applications》，Martin Kleppmann
- Elasticsearch 官方文档: Translog
- Lucene 文档: Segments and Commits
- DDIA 中文版：第3章、第5章、第9章、第10章
- Blog: Deep Dive into Elasticsearch Translog and Commit Process

### 🧭 延伸阅读建议

探索：如何在分布式系统中实现 “WAL + Snapshot” 一致性（对比 Kafka log + checkpoint）

思考：为什么 Elasticsearch 不使用传统的 page-based 存储结构？

对比：RocksDB 的 WAL + MemTable + SSTable 三层架构与 ES 的异同
