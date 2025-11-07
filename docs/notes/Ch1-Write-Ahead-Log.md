# Write-Ahead Log（WAL）学习笔记

**Write-Ahead Log（WAL）** 是一种**保证数据一致性与持久性**的日志机制，常用于数据库系统（如 PostgreSQL、SQLite、MySQL InnoDB）和分布式系统（如 HDFS、Kafka）。其核心思想是：

> **所有修改必须先写入日志，再写入实际数据文件。**

---

### 🧠 原理概述

在数据库中，更新数据通常分为两步：

1. 修改内存（buffer）中的数据页；
2. 将修改写回磁盘上的数据文件。

但如果系统在写入磁盘之前崩溃，就会丢失修改。
因此 WAL 的做法是：

1. **先把修改操作（事务日志）写入顺序日志文件（WAL log）**；
2. **再异步将数据页刷新到数据文件**。

系统重启时，读取 WAL 日志即可重放未持久化的修改，保证数据不丢失。

---

### ⚙️ 执行流程（以数据库事务为例）

| 阶段  | 操作                       |
| --- | ------------------------ |
| 1️⃣ | 事务开始，更新内存中的数据页           |
| 2️⃣ | 将更新操作的记录（redo log）写入 WAL |
| 3️⃣ | WAL 落盘（fsync）成功后，事务可以提交  |
| 4️⃣ | 后台进程异步将脏页写回数据文件          |
| 5️⃣ | 系统崩溃时，恢复过程根据 WAL 重放或回滚事务 |

---

### 💾 WAL 的结构（典型示意）

```
|----WAL 文件----|
| Txn Start | Update X=10 | Update Y=20 | Commit | ...
```

每个操作都记录了：

* 事务 ID
* 修改前值（undo 信息，可选）
* 修改后值（redo 信息）
* 时间戳或 LSN（Log Sequence Number）

---

### 🧩 优点

1. **提高性能**：
   顺序写日志比随机写数据页快，尤其对磁盘友好。
2. **保证原子性与持久性（ACID 的 A/D）**：
   只要日志写成功，即使宕机也能恢复。
3. **支持增量恢复与复制**：
   例如 PostgreSQL 的 streaming replication 就是通过传输 WAL 来做主从同步。

---

### 🧨 缺点

* 增加磁盘空间占用（日志可能很大）；
* 需要定期 checkpoint，将日志内容刷入主数据文件；
* 写放大：一次修改需要写两次（日志 + 数据）。

---

### 🏗️ 举例：PostgreSQL 的 WAL

* WAL 文件默认位于 `pg_wal/` 目录；
* 每个 WAL 段通常 16MB；
* 检查点（Checkpoint）会将缓冲区脏页写入磁盘，并标记之前的日志可回收；
* 在主从复制中，WAL 被实时发送到从库执行重放。

---

### 🔍 类比理解

可以把 WAL 想象成**黑匣子（flight recorder）**：

* 每次操作前，系统先记录“我打算做什么”；
* 即使飞机（系统）坠毁，黑匣子（WAL）仍能帮助“重建现场”。

---

## 🧠 延伸思考

- 为什么顺序写性能优于随机写？  
- Checkpoint 的频率如何平衡恢复时间与磁盘 I/O？  
- 分布式系统中（Kafka、HDFS）的 WAL 有何不同？

---

📚 **参考资料**
- PostgreSQL 官方文档：[https://www.postgresql.org/docs/current/wal-intro.html](https://www.postgresql.org/docs/current/wal-intro.html)
- SQLite Write-Ahead Logging: [https://www.sqlite.org/wal.html](https://www.sqlite.org/wal.html)
- “Designing Data-Intensive Applications”, Chapter 3: Storage and Retrieval

---
