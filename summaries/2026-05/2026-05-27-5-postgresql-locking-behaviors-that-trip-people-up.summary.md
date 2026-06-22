---
title: "总结：5 个容易踩坑的 PostgreSQL 锁行为"
source: "https://dev.to/shinyakato_/5-postgresql-locking-behaviors-that-trip-people-up-4k7n"
saved: "2026-05-27"
summary_of: "/Users/hermesai/Documents/我的远程大脑/LLM Wiki/raw/clippings/2026-05/2026-05-27-5-postgresql-locking-behaviors-that-trip-people-up.md"
tags:
  - summary
  - postgresql
  - database
  - locking
---

# 总结：5 个容易踩坑的 PostgreSQL 锁行为

## 一句话结论

PostgreSQL 的锁机制本身是为了保证正确性，但几个“隐式锁 + FIFO 锁队列 + 特殊 VACUUM/autovacuum 行为”组合起来，很容易把一个看似无害的 DDL、INSERT 或 VACUUM 放大成全库级阻塞；最重要的防线是 `lock_timeout`、`pg_stat_activity` 监控、统一锁顺序、死锁重试，以及对 autovacuum wraparound / VACUUM truncate 阶段的预案。

## 文章主旨

文章讲的是 PostgreSQL 中 5 个反直觉的锁行为。核心不是“PostgreSQL 锁很复杂”这么泛，而是：很多锁并不会直接出现在你写的 SQL 里，或者在通常情况下不会造成问题，但一旦进入等待队列，就会因为 PostgreSQL 锁等待的排队规则，把后续本来不冲突的查询也拖下水。

## 5 个关键坑

1. `ACCESS EXCLUSIVE` 一旦排队，会造成链式阻塞：一个长 `SELECT` 先持有 `ACCESS SHARE`，随后 `ALTER TABLE` 想拿 `ACCESS EXCLUSIVE` 被挡住；之后新来的 `SELECT` 虽然不和旧 `SELECT` 冲突，但会排在等待中的 `ACCESS EXCLUSIVE` 后面，于是整个表查询都像被卡住。`idle in transaction` 也会制造同样问题。

2. 外键会让 `INSERT` 隐式锁父表行：向子表插入时，PostgreSQL 会在父表被引用行上拿 `FOR KEY SHARE`。如果两个事务先分别 `FOR UPDATE` 锁住父表不同行，再交叉插入引用对方行，就会形成死锁。应用 SQL 看起来只是 INSERT，但锁其实发生在父表。

3. 唯一约束检查也能让纯 INSERT 死锁：两个事务分别插入不同主键值，然后再反向插入对方已插入但未提交的值。唯一性检查需要等待对方提交或回滚，于是两边互等，最终死锁。

4. 防事务 ID 回卷的 autovacuum 不会自动让步：普通 autovacuum 遇到冲突通常会被取消，但 `to prevent wraparound` 的 autovacuum 不会。这时如果 DDL 等它，再叠加第 1 点的 FIFO 队列，后续 SELECT 也可能被链式阻塞。

5. VACUUM 有隐藏的 `ACCESS EXCLUSIVE` truncate 阶段：普通 VACUUM 多数时间拿 `SHARE UPDATE EXCLUSIVE`，但最后截断表尾空页、归还空间时会拿 `ACCESS EXCLUSIVE`。在 primary 上可能导致链式阻塞；在 standby 上，这个锁操作通过 WAL replay 传播，长查询会阻塞 replay，超过 `max_standby_streaming_delay` 后查询会被强制取消。

## 操作启发

- 所有可能拿 `ACCESS EXCLUSIVE` 的 DDL 都应该设置 `lock_timeout`，避免排队时间无限扩大。
- DDL 前先看 `pg_stat_activity`，尤其查长查询和 `idle in transaction`。
- 外键父表行锁、唯一约束检查都可能让“纯 INSERT”变成锁冲突来源，所以应用层必须有死锁重试。
- 多事务操作同一组资源时，要统一锁顺序，例如永远按 `id` 升序锁。
- 定期监控 `age(relfrozenxid)`，避免 wraparound autovacuum 在业务高峰期不可取消地介入。
- 对依赖 standby 长查询的系统，要理解 `max_standby_streaming_delay` 的取舍：设短会更快恢复 WAL replay，但更容易取消查询；设长会保护查询，但可能扩大复制延迟。

## 我的理解

这篇文章最有价值的点是把 PostgreSQL 锁问题从“锁模式冲突表”拉回到运维现场：真正危险的不是某个锁本身，而是等待队列里的高优先级/强冲突锁把后续请求串起来。尤其 `ALTER TABLE`、VACUUM truncate、wraparound autovacuum 这些平时容易被视作后台或瞬时操作的动作，在大表、长事务、读副本场景下都可能变成事故触发器。

如果要把它转成团队规范，可以落成三条：DDL 必带 `lock_timeout`；事务必须避免长时间 idle；所有数据库死锁都视为正常可重试错误，而不是异常低概率事件。
