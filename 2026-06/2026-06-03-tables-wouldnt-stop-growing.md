---
source_url: https://stormatics.tech/blogs/the-night-our-tables-wouldnt-stop-growing
ingested: 2026-06-03
author: Semab Tariq (Stormatics)
---

# The Night Our Tables Wouldn't Stop Growing

> 一场 PostgreSQL 逻辑复制的生产事故：statement_timeout 与初始表拷贝的致命互动，导致目标端表膨胀至 400GB+。

## The Setup

客户要将 PostgreSQL 数据库通过**逻辑复制**迁移到新服务器。第一步是 initial table copy — PG 先把源端所有现有行拷贝到目标端，然后才开始流复制。

## What Went Wrong

第二天早上发现：源端 50-90GB 的表，在目标端膨胀到了 **400GB+ 每张**，而且还在继续增长。

## Root Cause: statement_timeout

发布端（publisher）设置了 `statement_timeout = 1min`。初始表拷贝本质上是一个长时间运行的 COPY 操作，每次跑到 1 分钟就被 timeout 杀掉。

更致命的是：PG 逻辑复制的 copy 进程被中断后**不会等人工重启**，它自动重试。

## 恶性循环

```
COPY 开始 → timeout 杀掉 → 已复制的行变成 dead tuples → 重试 → 再次 timeout → 更多 dead tuples → 无限循环
```

Autovacuum 来不及清理，dead tuples 堆积速度远超清理速度，表在目标端无限膨胀。

## The Fix

```sql
ALTER ROLE replication_user SET statement_timeout = 0;
```

只为复制用户取消 timeout，其他用户不受影响。一行 SQL 解决。

## Key Takeaway

两个各自合理的设置 — 合理的 `statement_timeout` 和逻辑复制初始拷贝 — 相遇后可以悄悄地互相破坏。Timeout 杀 COPY，COPY 重启，dead tuples 堆积。如果没人盯着，可以跑一整夜。

修复只需一行 SQL，但前提是你知道该往哪看。
