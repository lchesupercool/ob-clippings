---
title: "5 个容易踩坑的 PostgreSQL 锁行为"
source: "https://dev.to/shinyakato_/5-postgresql-locking-behaviors-that-trip-people-up-4k7n"
author: "Shinya Kato"
published: "2026-05-26T20:00:00Z"
saved: "2026-05-27"
tags:
  - clipping
  - postgresql
  - database
  - locking
translation_of: "/Users/hermesai/Documents/我的远程大脑/LLM Wiki/raw/clippings/2026-05/2026-05-27-5-postgresql-locking-behaviors-that-trip-people-up.md"
---

# 5 个容易踩坑的 PostgreSQL 锁行为

![PostgreSQL 锁行为封面](../../assets/5-postgresql-locking-behaviors-that-trip-people-up/image-01.png)

## 引言

PostgreSQL 使用 MVCC（Multi-Version Concurrency Control，多版本并发控制）来做并发控制：读不会阻塞写，写也不会阻塞读。

它的锁系统有 8 种表级锁模式和 4 种行级锁模式，文档里的冲突表会精确告诉你哪些锁模式之间会发生冲突。

但在实际运行 PostgreSQL 时，锁往往会在你完全没预料到的地方发生冲突。查询会比预期慢得多，最糟糕时甚至会造成服务中断。

本文梳理 5 个这种反直觉的锁行为。

## 环境

- 版本：PostgreSQL 18
- 事务隔离级别：READ COMMITTED（默认值）

## 1. 一旦 `ACCESS EXCLUSIVE` 请求进入队列，后续查询会被链式阻塞

第一个坑：一个本应瞬间完成的 `ALTER TABLE`，可能让整个服务停住。

假设一个 session 正在表 `t` 上运行一个长时间 `SELECT`，另一个 session 执行下面的 `ALTER TABLE`：

**Session 1**

```sql
SELECT pg_sleep(600) FROM t LIMIT 1; -- a long-running SELECT
```

**Session 2**

```sql
ALTER TABLE t ADD COLUMN name text;
```

由于 Session 1 在表 `t` 上持有 `ACCESS SHARE` 锁，Session 2 的 `ALTER TABLE` 需要的 `ACCESS EXCLUSIVE` 锁就必须等待。到这里为止，这还是符合预期的行为。

但 PostgreSQL 的锁等待像 FIFO 队列一样工作。当 `ACCESS EXCLUSIVE` 锁正在等待时，之后任何针对表 `t` 发出的 `SELECT` 都会卡在它后面——即使这个新的 `SELECT` 与 Session 1 当前正在运行的 `SELECT` 完全不冲突。

![后续 SELECT 也会被迫在锁队列里等待的示意图](../../assets/5-postgresql-locking-behaviors-that-trip-people-up/image-02.png)

> 除了长时间运行的 `SELECT`，如果某个 session 在 `BEGIN` 事务里执行了 `SELECT`，然后没有 `COMMIT`/`ROLLBACK` 就把事务晾在那里（所谓 *idle in transaction* session），也会发生同样的事。`ACCESS SHARE` 锁会一直持有到事务结束，所以即使 `SELECT` 本身瞬间执行完，只要忘了关闭事务，也会持续阻塞 `ALTER TABLE`。这通常是应用漏提交，或连接取完结果后仍处于 idle 状态导致的，需要特别小心。

结果就是下面这条链：

1. 一个长时间运行的 `SELECT` 正在执行。
2. 一个 `ALTER TABLE`（或任何会获取 `ACCESS EXCLUSIVE` 锁的语句）因为锁冲突被迫等待。
3. 后续所有 `SELECT` 都排到第 2 步的锁等待后面。

这种链式阻塞会让你的 `SELECT` 等待远超预期，并可能导致故障。

### 缓解措施

- 在会执行获取 `ACCESS EXCLUSIVE` 锁语句的事务中设置 `lock_timeout`，避免锁等待无限拖长。
- 事先通过 `pg_stat_activity` 视图检查是否存在长时间运行的查询。

## 2. 外键约束导致的“隐形”死锁

第二个坑来自外键约束隐式获取的锁。

当你执行 `INSERT` 时，PostgreSQL 会在外键指向的每个父表的被引用行上获取 `FOR KEY SHARE` 锁。这个机制用于防止父表中被引用的行被删除或更新，从而破坏引用方的完整性。

例如，当你向表 `t` 执行 `INSERT` 时，PostgreSQL 会自动在父表 `s` 中被引用的行上获取 `FOR KEY SHARE` 锁。由于 `FOR KEY SHARE` 与 `FOR UPDATE` 冲突，如果两个 session 各自先用 `FOR UPDATE` 锁住 `s` 中不同的行，然后又以相反顺序执行引用对方行的 `INSERT`，就会形成循环等待，导致死锁。

**Session 1**

```sql
BEGIN;
SELECT * FROM s WHERE id=1 FOR UPDATE; -- lock row s.id=1 with FOR UPDATE
```

**Session 2**

```sql
BEGIN;
SELECT * FROM s WHERE id=2 FOR UPDATE; -- lock row s.id=2 with FOR UPDATE
```

**Session 1**

```sql
INSERT INTO t (s_id) VALUES (2); -- wants FOR KEY SHARE on row s.id=2 (waits for Session 2)
```

**Session 2**

```sql
INSERT INTO t (s_id) VALUES (1); -- wants FOR KEY SHARE on row s.id=1 (deadlock detected)
```

这里反直觉的地方是：

- 应用代码里看起来只有 `INSERT`，但它会隐式锁住外键父表里的行。
- 只看应用 SQL 时，这次锁获取完全不会显式出现。

### 缓解措施

- 避免这样的设计：多个 session 先用 `FOR UPDATE` 锁住外键父表中的某一行，再以相反顺序 `INSERT` 另一个引用它的行——这会造成死锁。
- 如果确实需要这么做，要在整个应用中统一锁获取顺序。例如永远按 `s` 表的 `id` 升序锁行，强制一个单向的锁顺序。
- 死锁一定会以错误形式上报，因此实现时要预设重试机制。

## 3. 唯一约束重复检查导致两个 `INSERT` 之间死锁

第三个坑：两个事务即使只做 `INSERT`，也可能因为以相反顺序插入对方已经插入过的值而死锁。

执行如下 SQL：

**Session 1**

```sql
BEGIN;
INSERT INTO t (id) VALUES (1);
```

**Session 2**

```sql
BEGIN;
INSERT INTO t (id) VALUES (2);
```

**Session 1**

```sql
INSERT INTO t (id) VALUES (2); -- waits for Session 2 to commit
```

**Session 2**

```sql
INSERT INTO t (id) VALUES (1); -- deadlock detected
```

原因是，`PRIMARY KEY` 和 `UNIQUE` 约束的重复检查在发现另一个事务正在插入同一个值时，会等待对方事务结束。PostgreSQL 需要这样决定最终结果：“如果对方事务提交，那就是唯一约束冲突；如果对方回滚，那这次插入成功。”

所以上面的例子里：

- Session 1 尝试插入 `id=2` → Session 2 正在插入 `id=2`，所以它等待。
- Session 2 尝试插入 `id=1` → Session 1 正在插入 `id=1`，所以它等待。
- 两边互相等待对方提交 → 检测到死锁。

### 缓解措施

- 避免同一个值可能被多个 session 并发插入的设计。使用 `SERIAL`/`IDENTITY` 这类序列可以从源头避免重复值。
- 如上一节所说，死锁会以错误形式上报，因此要按可能重试来实现。

## 4. 只有防事务 ID 回卷的 autovacuum 不会在冲突时让步

第四个坑关于 autovacuum 的一个例外行为。

当 autovacuum 获取的 `SHARE UPDATE EXCLUSIVE` 锁与其他语句冲突时，通常会被取消的是 autovacuum 自己，因此它不会长时间阻塞其他语句。这也是为什么大家通常认为“让 autovacuum 跑着就行”。

但有一个例外：用于防止事务 ID 回卷的 autovacuum（`pg_stat_activity` 的 `query` 列以 `(to prevent wraparound)` 结尾的那种）即使发生冲突，也不会被自动取消。

实际过程如下：

1. 某张表的 `relfrozenxid` age 超过 `autovacuum_freeze_max_age`。
2. 防回卷 autovacuum 启动，并持有 `SHARE UPDATE EXCLUSIVE`。
3. 另一个进程执行 `ALTER TABLE` 创建分区。
4. `ALTER TABLE` 等待 autovacuum ← 正常情况下 autovacuum 会被取消，但这次不会。
5. 所有 `SELECT` 都在队列里排队 ← 第 1 节描述的链式阻塞发生。

关键点是：一旦这个行为和第一个行为结合，故障会迅速变得致命。

### 缓解措施

- 如果在 `pg_stat_activity` 里看到 `autovacuum: VACUUM ... (to prevent wraparound)`，要意识到这是一个不会被自动取消的 autovacuum，应该等待它完成。
- 预防比事后反应更重要。定期监控 `pg_class` 中的 `age(relfrozenxid)`，了解哪些表正在接近 `autovacuum_freeze_max_age`（默认 2 亿事务）。

```sql
  SELECT relname, age(relfrozenxid)
  FROM pg_class
  WHERE relkind = 'r'
  ORDER BY age(relfrozenxid) DESC;
```

- 如果计划执行 `ALTER TABLE` 这类 DDL，提前检查目标表的 `age(relfrozenxid)`；如果接近阈值，在 DDL 之前手动执行 `VACUUM FREEZE`。

## 5. VACUUM 隐藏的 `ACCESS EXCLUSIVE` 阶段

最后一个坑同样关于 VACUUM 的例外行为。

通常 VACUUM 使用 `SHARE UPDATE EXCLUSIVE`，但最终的 truncate 阶段是例外：它会截断表尾部的空页并把磁盘空间返还给 OS，而这个阶段会获取 `ACCESS EXCLUSIVE` 锁。

和前面的例子一样，如果 VACUUM 正在等待获取这个 `ACCESS EXCLUSIVE` 锁，同时有一个长时间运行的 `SELECT` 正在执行，后续查询都会按照第 1 节描述的链式阻塞模式被卡住。

在流复制 standby 上，情况又不同。当 primary 上获取并释放 `ACCESS EXCLUSIVE` 后，这个操作会通过 WAL 传到 standby。standby 的 WAL replay 进程尝试应用这个操作，但 standby 上一个长时间运行的 `SELECT` 与它冲突，于是 WAL replay 停住。WAL replay 停住期间，已经收到的 WAL 的 apply lag 会持续增长；一旦 lag 超过 `max_standby_streaming_delay`，冲突的 `SELECT` 就会被强制取消，replay 恢复。

在 primary 上，`SELECT` 可以等到自然结束；在 standby 上，`SELECT` 会因为外部因素（WAL replay）被强制取消。这是最大的区别。

### 缓解措施

- PostgreSQL 12 及以后，可以按表设置 `vacuum_truncate = false` 来禁用 VACUUM 的 truncate 阶段。
- PostgreSQL 18 及以后，还新增了服务端全局 `vacuum_truncate` 参数。
- 如果在 standby 上运行长查询，可以调高 `max_standby_streaming_delay`（默认 30 秒），让 WAL replay 允许等待更久；设为 `-1` 表示无限等待。

## 结论

这些案例都是 PostgreSQL 的锁设计为了正确性而自然产生的行为。从规格角度看很自然，但从运维角度看就是陷阱。

尤其是第 1 个链式阻塞行为，一旦和其他几个案例组合，可能迅速升级成严重事故。因此，设置 `lock_timeout` 并持续监控 `pg_stat_activity`，是至少要有的防线。

## 参考资料

- [https://www.postgresql.org/docs/current/explicit-locking.html](https://www.postgresql.org/docs/current/explicit-locking.html)
- [https://www.postgresql.org/message-id/d7df81620708101038k772a2cderb52bb09f5440bd1b@mail.gmail.com](https://www.postgresql.org/message-id/d7df81620708101038k772a2cderb52bb09f5440bd1b@mail.gmail.com)
- [https://leosjoberg.com/blog/lock-propagation-postgres/](https://leosjoberg.com/blog/lock-propagation-postgres/)
- [https://xata.io/blog/migrations-and-exclusive-locks](https://xata.io/blog/migrations-and-exclusive-locks)
