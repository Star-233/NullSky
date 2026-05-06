
[wal_recover](https://github.com/CTF-Archives/2026-LanqiaoCup-Quals/releases/download/attachments/wal_recover.zip)
## 概述

题目给出两个文件：

- `app.db` — SQLite 数据库主文件（4 KB）
- `app.db-wal` — SQLite 预写日志文件（28 KB）

题目描述称"可疑任务已被彻底删除"，但日志文件仍保留在现场。目标是恢复被删除的内容。

---

## 基础知识

### SQLite 的两种日志模式

SQLite 支持多种日志/事务模式，其中最常见的两种是：

#### 1. 回滚日志模式（Rollback Journal，`journal_mode=DELETE`）

这是 SQLite 的默认模式。事务流程如下：

1. 修改某页前，先将该页的**原始内容**复制到 `.db-journal` 文件
2. 直接修改 `.db` 主文件
3. 事务提交后，删除 `.db-journal`

**优点**：`.db` 文件始终包含最新数据。
**缺点**：每次写操作都直接修改主文件，崩溃恢复需要回放 journal。

#### 2. 预写日志模式（Write-Ahead Log，`journal_mode=WAL`）

WAL 模式改变了写入策略：

1. 修改某页时，**不修改 `.db` 主文件**，而是将修改后的页作为一"帧"（frame）**追加**到 `.db-wal` 文件尾部
2. 读操作同时读取 `.db` 和 `.db-wal`，以 WAL 中的版本为最新
3. 当事务提交时，只需在 WAL 中标记一个 commit 记录，无需落盘修改 `.db`

**优点**：
- 读写不互斥（读不阻塞写，写不阻塞读）
- 写入性能更高（顺序追加 vs 随机写）
- 崩溃恢复更简单

**缺点**：
- WAL 会不断增长（需要定期 checkpoint）
- 读操作需要合并两个文件，略微增加读开销

---

### WAL 文件格式

一个 `.db-wal` 文件的布局如下：

```
+------------------+
| WAL Header (32B) |   ← 魔数、版本、页大小、校验盐值等
+------------------+
| Frame 0          |   ← 24B 帧头 + 4096B 页数据（4KB 页）
+------------------+
| Frame 1          |
+------------------+
| ...              |
+------------------+
| Frame N          |
+------------------+
```

#### WAL 头部（32 字节）

| 偏移 | 大小 | 字段 | 说明 |
|------|------|------|------|
| 0 | 4 | magic | 魔数，如 `0x377f0682` |
| 4 | 4 | version | 版本号，如 `3007000` |
| 8 | 4 | pagesize | 页大小（字节） |
| 12 | 4 | checkpoint | 最近一次 checkpoint 的序号 |
| 16 | 4 | salt-1 | 校验盐值 1 |
| 20 | 4 | salt-2 | 校验盐值 2 |
| 24 | 4 | checksum-1 | 头部校验和 |
| 28 | 4 | checksum-2 | 头部校验和 |

#### 帧头（24 字节）

| 偏移 | 大小 | 字段 | 说明 |
|------|------|------|------|
| 0 | 4 | page number | 此帧包含的是数据库中的第几页（从 1 开始） |
| 4 | 4 | commit size | 如果 >0，表示这是一个 commit 帧，值为此事务涉及的页数 |
| 8 | 4 | salt-1 | 继承自 WAL 头部的盐值 1 |
| 12 | 4 | salt-2 | 继承自 WAL 头部的盐值 2 |
| 16 | 4 | checksum-1 | 帧数据校验和 |
| 20 | 4 | checksum-2 | 帧数据校验和 |

帧头之后紧跟的 `pagesize` 字节即为该页的完整镜像。

---

### 什么是 Checkpoint？

**Checkpoint** 是将 WAL 文件中的变更**回放**到主数据库 `.db` 文件的过程。

#### 触发时机

- 自动触发：WAL 文件大小达到阈值（默认 1000 页）
- 手动触发：执行 `PRAGMA wal_checkpoint;`
- 正常关闭数据库连接时（通常也会 checkpoint）

#### Checkpoint 的过程

1. 遍历 WAL 中的所有帧
2. 将每个帧中的页数据写回 `.db` 文件中对应位置
3. 更新 `.db` 头部的 WAL 相关字段（盐值、序号等）
4. **截断或清空** `.db-wal` 文件（或重置其头部，标记所有帧为已 checkpoint）

#### Checkpoint 的类型

| 模式 | 行为 |
|------|------|
| `PASSIVE` | 尽可能 checkpoint，但不阻塞读写 |
| `FULL` | 阻塞其他操作，保证 checkpoint 完成 |
| `RESTART` | 在 FULL 基础上，确保 WAL 完全清空，后续写入从头开始 |

#### Checkpoint 对取证的影响

**一旦 checkpoint 发生，WAL 文件被清空/截断，所有历史变更版本永久丢失。** 这就是为什么本题中**绝不能直接用 `sqlite3 app.db` 打开数据库**——SQLite 可能在打开时自动触发 checkpoint，销毁证据。

---

## 🧩 题目分析

### 初始状态

```
app.db      → 4096 bytes（1 页）
app.db-wal  → 28872 bytes（7 帧）
```

`.db` 文件几乎全零（除头部 100 字节外），说明所有数据都通过 WAL 模式写入，**从未被 checkpoint 到主文件**。

### WAL 帧解读

对 `.db-wal` 逐帧解析结果如下：

| 帧 | 页码 | commit_sz | 内容 |
|----|------|-----------|------|
| 0 | 1 | 0 | `task_log` 表定义 |
| 1 | 2 | **2** | 首次 commit，扩展页面 |
| 2 | 1 | 0 | `task_log` + `sync_cache` 两张表定义 |
| 3 | 3 | **3** | commit |
| 4 | 3 | **3** | `sync_cache` 数据（3 条记录） |
| **5** | **2** | **3** | `task_log` **原始完整数据（5 条记录）** |
| **6** | **2** | **3** | `task_log` **DELETE 后的数据（2 条记录）** |

### 关键发现

**Frame 5（page 2）** 包含原始的 5 条 `task_log` 记录：

```
2024-04-01 10:01:04  sync-bot  part:01:[ZmxhZ3tkNGZi]
2024-04-01 10:01:08  sync-bot  part:02:[ZTdkNy04YWY4]
2024-04-01 10:01:12  sync-bot  part:03:[LTQ1MjMtOTgz]
2024-04-01 10:01:16  sync-bot  part:04:[Zi01ODA1NTA2]
2024-04-01 10:01:20  sync-bot  part:05:[ZGIyNmV9]
```

**Frame 6（page 2）** 只保留了 DELETE 后的 2 条 ops 日志：

```
2024-04-01 10:03:11  ops  cache cleared
2024-04-01 10:04:22  ops  retry after package update
```

显然，开发者删除了 3 条 `sync-bot` 的日志，以为"彻底删除了"。

---

## 🔑 恢复 Flag

每条被删记录的 `note` 字段包含一段 Base64 编码：

| part | Base64 | 解码结果 |
|------|--------|----------|
| 01 | `ZmxhZ3tkNGZi` | `flag{d4fb` |
| 02 | `ZTdkNy04YWY4` | `e7d7-8af8` |
| 03 | `LTQ1MjMtOTgz` | `-4523-983` |
| 04 | `Zi01ODA1NTA2` | `f-5805506` |
| 05 | `ZGIyNmV9` | `db26e}` |

拼接得到完整 flag：

```
flag{d4fbe7d7-8af8-4523-983f-5805506db26e}
```

---

## 恢复方法总结

### 方法一：strings 直接提取（最简单）

```bash
strings app.db-wal | grep -E '\['
```
从 WAL 中捞出所有 Base64 片段，手动解码。

### 方法二：Python 逐帧解析（推荐）

```python
import struct, base64

with open('app.db-wal', 'rb') as f:
    wal = f.read()

pagesize = 4096
frame_hdr = 24
n_frames = (len(wal) - 32) // (frame_hdr + pagesize)

for i in range(n_frames):
    off = 32 + i * (frame_hdr + pagesize)
    pgno, cmt = struct.unpack('>II', wal[off:off+8])
    body = wal[off+24:off+24+pagesize]
    # 对 body 做 strings 即可
    import re
    for m in re.finditer(rb'[\x20-\x7e]{4,}', body):
        print(f"Frame {i}, page {pgno}: {m.group().decode()}")
```

### 方法三：重放 WAL 到隔离数据库（谨慎）

```bash
cp app.db /tmp/recovered.db
cp app.db-wal /tmp/recovered.db-wal
cd /tmp
# 打开时加 IMMEDIATE 模式防 checkpoint
sqlite3 recovered.db "PRAGMA journal_mode=WAL; SELECT * FROM task_log;"
```

### ⚠️ 禁止操作

```bash
# ❌ 这会触发 auto-checkpoint，销毁 WAL！
sqlite3 app.db "PRAGMA wal_checkpoint;"

# ❌ 这也可能触发 checkpoint
sqlite3 app.db ".tables"
```

---

## 小结

| 概念 | 一句话总结 |
|------|-----------|
| `.db` | SQLite 数据库主文件，保存已 checkpoint 的数据 |
| `.db-wal` | 预写日志，保存尚未回放到 `.db` 的变更 |
| WAL 模式 | 修改先写日志、后落盘的机制，提升并发性能 |
| Checkpoint | 将 WAL 中的变更写回 `.db` 并清空 WAL，**不可逆地销毁历史版本** |
| 取证关键 | 在未 checkpoint 的 WAL 帧中，每页的**所有历史版本**都完整保留 |

题目利用的是 WAL 模式的一个"副作用"：**未 checkpoint 的 WAL 保留了数据库页面的全部历史版本**。开发者以为执行了 DELETE 就彻底删除了数据，但 WAL 帧中仍存有被删行所在页的完整镜像——只要找对帧，对页数据做 `strings`，被删记录就原形毕露了。