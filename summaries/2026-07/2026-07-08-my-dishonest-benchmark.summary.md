---
title: "My Dishonest Benchmark — 中文摘要"
source: "https://drunkdba.medium.com/my-dishonest-benchmark-b52d2e706b59"
saved: "2026-07-08"
author: "Mayur (Do not drink & database)"
summary_of: "My Dishonest Benchmark"
tags: [postgresql, benchmarking, database, performance, dewitt-clause]
---

# My Dishonest Benchmark — 中文摘要

## 一句话结论

这篇文章用一个“故意不诚实”的 PostgreSQL benchmark 演示：性能数字可以通过坏 baseline、改变工作量、改变持久性语义、批处理和换指标单位轻松做出夸张增长；真正重要的不是图表是否向右上角，而是 benchmark 是否保持同一 workload、同一语义、同一成本和可公开批评的方法论。

## 文章主旨

作者先回顾数据库 benchmark 史上的 DeWitt Clause。早期 Wisconsin Benchmark 曾让 Oracle 数据看起来很难看，据 David DeWitt 的说法，Larry Ellison 曾要求威斯康星大学开除他。后来 Oracle 等商业数据库许可证中出现限制公开发布 benchmark 的条款，这类限制被称为 DeWitt Clause。

作者认为，老式 benchmark 战争并不纯洁，厂商当然会调参、挑硬件、挑 workload、让自家系统有利。但至少当时有公开争论。今天更糟的是：许多公开 benchmark 技术上可复现，但方法论上倾斜；而由于法律和职业成本，挑战这些 benchmark 的人变少了。

文章于是用 PostgreSQL 做了一个“诚实地展示不诚实”的实验：从一个故意很差的 baseline 开始，逐步叠加各种优化和“作弊”，让 PostgreSQL 看起来快很多倍。

## DeWitt Clause 与现代 benchmark 问题

作者说，现代数据库 benchmark 常见问题包括：

- 精心调优自家数据库，却让对比数据库接近默认配置。
- 选择完美贴合自家架构的 workload。
- 忽略成本。
- 忽略 tail latency。
- 忽略 write amplification。
- 忽略 crash behavior。
- 忽略运维复杂度。
- 只发布自己赢的那张图。
- 最后称之为“transparent”。

作者赞同 PlanetScale 最近关于 benchmark transparency 的主张：不要阻止别人发布 benchmark，而是要求公开代码、配置、成本、workload，让争论回到技术方法论本身。

## 实验设计

作者构建了一个迷你 TPC-C 风格的 accounting workload，不是标准 TPC-C，也不声称是 audited payment system。每个事务大致做：

- 读取账户。
- 更新账户余额。
- 插入 ledger row。
- 插入 audit row。
- 偶尔删除旧 audit row。
- commit。

表包括：

- `bench_branches`
- `bench_accounts`
- `bench_ledger`
- `bench_audit`

数据量是 100 个 branch、100 万个 account。作者也给了完整 reproduction pack，包含 schema、pgbench workload、reset scripts、cheat variants、结果收集和图表生成代码。

## 最大“作弊”：从坏 baseline 开始

Case 0 是故意坏的 baseline：查询 `bench_audit.created_at` 并 `ORDER BY created_at LIMIT 1`，但没有 `created_at` 相关索引。

结果 PostgreSQL 被迫做 seq scan + sort，Case 0 只有约 558 TPS。

然后作者加上明显缺失的复合索引：

```sql
CREATE INDEX IF NOT EXISTS bench_audit_created_at_audit_id_idx
ON bench_audit(created_at, audit_id);
```

加索引后，honest indexed baseline 达到 23178 TPS，提升约 41 倍。

作者指出：制造英雄式 benchmark 最简单的方法，就是从坏代码开始，然后把它叫作 baseline。

## 各种“作弊”结果

文章列出了一系列 benchmark trick。很多数字是真的，但语义或比较方式已经变了。

| 手法 | 结果 | 相对 indexed baseline | 问题 |
|---|---:|---:|---|
| Case 0 坏 baseline | 558 TPS | — | 缺索引，schema 懒惰，不是 PostgreSQL 慢 |
| Honest indexed baseline | 23178 TPS | 1x | 合理 baseline |
| `synchronous_commit=off` | 25063 TPS | 1.08x | 改变 commit 持久性承诺 |
| `full_page_writes=off` | 22769 TPS | 0.98x | 弱化 torn-page 保护，且没更快 |
| `fsync=off` | 24863 TPS | 1.07x | 取消核心持久性保证，但提升很小 |
| checkpoint tuning | 23048 TPS | 0.99x | 120 秒短测试里影响不大 |
| all durability off | 25349 TPS | 1.09x | 听起来吓人，但对这台 NVMe 提升很小 |
| less work | 44777 TPS | 1.93x | 直接少做事，workload 已改变 |
| UNLOGGED ledger/audit | 37016 TPS | 1.60x | 跳过 WAL，适合可重建数据，不适合 ledger |
| PL/pgSQL stored procedure | 46126 TPS | 1.99x | 减少 client/server round trips，合理但需公平对比 |
| SQL function | 70626 TPS | 3.05x | 最干净的 PostgreSQL 正向优化 |
| prepared protocol | 31250 TPS | 1.35x | 合理优化，但对比方也应同等待遇 |
| SQL function + prepared | 70796 TPS | 3.05x | 主要收益来自 server-side execution |
| batching x8 | 104466 OPS / 13058 TPS | 看 OPS 4.51x | 指标从 TPS 换成 OPS，图更好看 |
| batching x32 | 130092 OPS / 4065 TPS | 看 OPS 5.61x | OPS 变大，TPS 反而变差 |
| 全部叠加 + batch x32 | 363915 OPS | 对 indexed baseline 15.70x，对坏 baseline 649x | 数字真实， framing 是 trick |

## 最关键的几类 benchmark 造假方式

### 1. 坏 baseline

把明显缺索引、明显错误的 schema 当 baseline，然后修掉这个问题，就能制造几十倍提升。

### 2. 改变 durability 语义

`fsync=off`、`synchronous_commit=off`、`full_page_writes=off` 等设置会改变数据库对 crash safety 和 durability 的承诺。如果不说明，就不是同一语义下的公平比较。

有趣的是，这组实验里 durability cheating 并没有带来巨大收益，说明现代硬件和 group commit 下，很多“危险参数”的营销想象大于实际收益。

### 3. 少做工作

移除 SELECT、FK check、audit insert/cleanup 后 TPS 接近翻倍。但这不是优化同一个 workload，而是把 workload 改小了。

### 4. 使用 UNLOGGED 表

UNLOGGED 可以减少 WAL，适合临时或可重建数据，但不适合需要 crash recovery 的 ledger/audit。把它和持久化表直接对比，是语义变化。

### 5. Server-side execution

PL/pgSQL procedure 和 SQL function 能减少网络往返和客户端开销，是真实优化。尤其 SQL function 达到 70626 TPS，约 3.05x，是作者认为最“体面”的 PostgreSQL-positive 结果。

但公平比较时，对手也应获得等价的 server-side execution 路径。

### 6. 换 y 轴单位

batching 是最典型的营销手法。x32 batching 的 OPS 变大，但 TPS 只有 4065，反而比 baseline 低很多。如果图上只画 OPS，就能制造漂亮增长。

作者总结得很准：数字没有造假，framing 才是 trick。

## 诚实结论

作者说，这个 dishonest benchmark 做了很多事：

- 从坏代码开始。
- 修复缺失索引。
- 改变 durability。
- 改变 WAL 行为。
- 改 checkpoint。
- 减少实际工作量。
- 使用 UNLOGGED 表。
- 把 round trips 藏进 PL/pgSQL / SQL function。
- 批处理多个操作。
- 使用 prepared protocol。

图表当然会向右上角走。这正是营销图想要的。

但更有趣的结论不是 benchmark 可以被操纵——严肃的人早就知道。更有趣的是：PostgreSQL 有足够成熟、无聊、生产级的机制，以至于哪怕是“作弊”，也要靠真实 PostgreSQL 特性才能作弊得像样。

## 对读者的启发

看数据库 benchmark 时，不能只看最终数字。至少要问：

- baseline 是否合理？
- 两边是否同等调优？
- workload 是否真的一致？
- durability / crash safety 语义是否一致？
- 统计的是 TPS、OPS、latency 还是某个包装指标？
- 是否有 tail latency？
- 是否包含成本？
- 是否包含 write amplification 和恢复行为？
- 是否公开脚本、配置、数据规模和运行方式？
- 是否每个 case 都从干净状态开始？

真正透明的 benchmark 不是“不许别人质疑”，而是把方法论放出来，让别人能复现、反驳、改进。

## 我的理解

这篇文章最好的地方是，它没有抽象地说“benchmark 会骗人”，而是把骗术拆成一条清晰 ladder。每一步都是真实优化或真实设置，但只要隐藏语义变化、换 baseline、换单位，就能把一个普通结果包装成惊人的性能突破。

对 PostgreSQL 来说，文章反而是正向宣传：它说明 Postgres 的优化手段很多，而且大多是成熟、可解释、生产可用的。但对所有数据库 benchmark 来说，核心教训是：数字本身不够，数字背后的 workload、语义、成本和约束才是 benchmark 的真实内容。
