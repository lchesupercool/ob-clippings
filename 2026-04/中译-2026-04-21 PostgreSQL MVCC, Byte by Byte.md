---
---

# PostgreSQL MVCC，逐字节解读

原文链接: https://boringsql.com/posts/postgresql-mvcc-byte-by-byte/

你在一个 psql session 里运行 SELECT * FROM orders，看到 5000 万行。与此同时，另一位同事在另一个 session 中执行同样的查询，却看到 49,999,999 行。你们俩都没错，谁也没有读到过期的数据。你们读的是同样的 8KB heap 页，磁盘上同样的字节。

这就是 PostgreSQL 的 MVCC（Multi-Version Concurrency Control，多版本并发控制）所承诺的东西，也正是"读不阻塞写、写不阻塞读"的根源。它同时也是存储引擎里最被误解的部分之一。很多人只知道"一行有多个版本"，然后就止步于此了。

答案就藏在每一条 tuple 的那 8 个字节里。


## xmin 与 xmax：唯一重要的两个 XID

如果你读过 Inside the 8KB Page，你会知道每条 tuple 都以一个 23 字节的 header 开头。这个 header 的前 8 个字节是两个 32 位的事务 ID：t_xmin（插入这个版本的事务）和 t_xmax（删除或更新了它的事务；如果仍然存活，则为 0）。

这就是存储层面上 MVCC 的核心。PostgreSQL 并不会单独维护一张"当前版本"的表，也不会把某行标记为"最新"。每条 tuple 自己携带两个字段的时间戳，当你的查询读到某一页时，PostgreSQL 必须一条一条 tuple 地判定你的事务是否有权看到它。

一个最小的演示：

```sql
CREATE TABLE mvcc_demo (id int, val text);
INSERT INTO mvcc_demo VALUES (1, 'alpha'), (2, 'beta');
```

用 pageinspect 窥视原始页：

```sql
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('mvcc_demo', 0));
```

```sql
 lp | t_xmin | t_xmax | t_ctid
----+--------+--------+--------
  1 |    100 |      0 | (0,1)
  2 |    100 |      0 | (0,2)
(2 rows)
```

两条 tuple。两者的 t_xmin 都标记为 100（执行 INSERT 的那个事务），t_xmax 都是 0（没有谁删除过它们）。在这个时刻，数据库上的每个 session 都会看到这些行，因为每个人的 snapshot 都一致认为事务 100 已经 commit。

现在开两个并发 session。Session A 发起一个 UPDATE 但不 commit：

```sql
-- session A
BEGIN;
UPDATE mvcc_demo SET val = 'alpha-new' WHERE id = 1;
-- do not commit yet
```

再次查看这一页：

```sql
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('mvcc_demo', 0));
```

```sql
 lp | t_xmin | t_xmax | t_ctid
----+--------+--------+--------
  1 |    100 |    101 | (0,3)
  2 |    100 |      0 | (0,2)
  3 |    101 |      0 | (0,3)
(3 rows)
```

一个 UPDATE，三条 tuple。id=1 的旧版本还在，处于行指针 1 上，被打上了 t_xmax = 101；新版本位于行指针 3，t_xmin = 101。

Session A 还没 commit。事务 101 仍在 in-flight。此刻 Session B 如果执行 SELECT * FROM mvcc_demo，仍然看到原来的 alpha，而不是 alpha-new。三条 tuple 都实实在在地躺在页上，但 Session B 的 snapshot 判定 XID 101 还在 in-flight，于是忽略它所做的一切。可见性判断是即时发生的，每次触及一条 tuple 都会判定一次。

这就是 MVCC 反直觉的地方：磁盘上的字节并不会因为是谁在问而变化。真正变化的是 planner 读取这些字节时所应用的那个可见性裁决。


## Snapshot

pg_current_snapshot() 是查看当前 session 所持有内容的最清晰方式。

```sql
SELECT pg_current_snapshot();
```

```sql
 pg_current_snapshot
---------------------
 101:103:101
(1 row)
```

它是 xmin:xmax:xip_list 的形式，也就是完整的 snapshot：

- xmin：可能仍在 in-flight 的最小 XID。比它更小的 XID 都已经有结论了（要么 commit，要么 abort）。你可以直接信任它们的 t_xmin/t_xmax 标记，不必再做进一步检查。
- xmax：第一个尚未分配的 XID。任何大于等于这个值的 XID 都还不存在。打上这种标记的 tuple 必须被忽略。
- xip_list：位于 xmin 和 xmax 之间、仍在运行中的那些 XID。这些就是 "in-flight" 事务，它们的写入对你必须不可见。

PostgreSQL 会逐条 tuple 应用这套判定。如果你的 snapshot 认为 t_xmin 已经 abort 或仍在 in-flight，那么这条 tuple 对你而言就不存在，PostgreSQL 会跳过它。如果 t_xmin 已 commit，那就由 t_xmax 定夺：为零表示该 tuple 存活；如果 t_xmax 是一个已 commit 的事务，说明有人删除了它，你看不到；如果 t_xmax 是 in-flight 或 abort 状态，说明这次删除尚未进入你的 snapshot。

同一页，同样的字节。不同 session 持有不同的 snapshot，于是对同一条 tuple 得到不同的结果。

> Interactive MVCC Visualizer
> 
> 针对同一个 heap 页驱动两个并发 session。观察 xmin、xmax 标记的变化，在 READ COMMITTED 和 REPEATABLE READ 之间切换，逐条 tuple 追踪可见性规则，并在 dead 版本堆积时运行 VACUUM。
> 
> Open Visualizer

[Open Visualizer](https://boringsql.com/visualizers/mvcc/)


## READ COMMITTED 与 REPEATABLE READ

PostgreSQL 最常用的两种隔离级别的区别，归结到最后就是一个问题：snapshot 是什么时候拍下来的？

READ COMMITTED（默认）会在每条语句开始时拍一张新的 snapshot。如果另一个 session 在你第一条 SELECT 和第二条 SELECT 之间 commit 了，你的第二条 SELECT 就会看到这次变更。世界在你的事务之下一条语句一条语句地向前走。

REPEATABLE READ 在事务开始时拍一张 snapshot，并把它复用在之后的每一条语句上。从你这个事务的视角看，世界是被冻结的。其他 session 可以 commit 一千次变更，你的查询仍然返回 BEGIN 时刻可见的内容。

两种情况下页面上的字节完全一样。唯一的区别是你的事务带在身上的是哪张 snapshot。

```sql
-- session A, READ COMMITTED (default)
BEGIN;
SELECT val FROM mvcc_demo WHERE id = 1;  -- 'alpha'

-- session B, in another terminal:
UPDATE mvcc_demo SET val = 'alpha-new' WHERE id = 1;
-- (auto-commits)

-- back in session A:
SELECT val FROM mvcc_demo WHERE id = 1;  -- 'alpha-new' new statement, new snapshot
COMMIT;
```

现在用 REPEATABLE READ 再来一遍：

```sql
-- session A, REPEATABLE READ
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT val FROM mvcc_demo WHERE id = 1;  -- 'alpha-new'

-- session B:
UPDATE mvcc_demo SET val = 'alpha-newer' WHERE id = 1;
-- (auto-commits)

-- Back in session A:
SELECT val FROM mvcc_demo WHERE id = 1;  -- still 'alpha-new'  same snapshot as BEGIN
COMMIT;
```

Visualizer 把这一点直接展示出来：每个 session 上都有一个隔离级别选择器。REPEATABLE READ 下，snapshot 在 BEGIN 时拍下并一直保留。READ COMMITTED 下，每次运行 SELECT 都会刷新一次。看看每条 tuple 上的可见性徽章是如何随之翻转的。


## 每一次 UPDATE 都会留下一条 dead tuple

PostgreSQL 中每一次 UPDATE 都会产生一个新的 tuple 版本。旧版本不会消失，而是被盖上 t_xmax，继续留在页面上占用空间，直到 VACUUM 过来把它回收。

在一张 update 频繁的繁忙表上，dead tuple 累积的速度可能比 VACUUM 清理的速度更快。这就是 "bloat"（膨胀），也是团队误以为 Postgres 需要重新调优的最常见原因之一。MVCC 的那份契约（"永远不阻塞、始终提供一致视图"）是用磁盘空间付的账。

你可以用 pgstattuple 看到 dead tuple 累积起来：

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- After lots of updates
SELECT table_len, tuple_count, dead_tuple_count, dead_tuple_percent
FROM pgstattuple('mvcc_demo');
```

```sql
 table_len | tuple_count | dead_tuple_count | dead_tuple_percent
-----------+-------------+------------------+--------------------
      8192 |           2 |                3 |              42.15
(1 row)
```

三条 dead tuple，两条 live tuple，页面 42% 被浪费。这 42% 会一直被浪费，直到 VACUUM 运行，或者直到下一次触及这一页的查询触发页级 pruning 并注意到那些 dead space。


## xmin horizon

VACUUM 只有在没有任何运行中的事务还可能需要看到这条 dead tuple 时，才能把它回收。如果 Session B 在五分钟前开启了一个 REPEATABLE READ 事务并且一直挂着不动，它的 snapshot 仍然把 id=1 的更新前版本视为 live 的那一版。VACUUM 如果动它，就会破坏这个 session。

因此 VACUUM 会找到系统中最早仍然活跃的事务，并拒绝清理任何更新于它的内容。一个长时间运行的 REPEATABLE READ 事务（比如一条跑一小时的分析查询）实际上会把这一小时内产生的每一个 tuple 版本全部钉死。表继续膨胀，autovacuum 启动，发现没有一条可以清理的 tuple，然后默默回家。

长事务问题不是 MVCC 的 bug，而是 MVCC 按设计运作的结果。"读不阻塞"的代价，就是读可能阻塞清理。如果你曾在一台出状况的生产库上查 pg_stat_activity，发现有个 14 小时之久的 idle in transaction，你就知道这种画面了。

Visualizer 把这件事展示得很清楚：在 Session B 里开一个 REPEATABLE READ 事务，让 Session A 执行一堆 UPDATE 并 COMMIT，然后点 VACUUM。回收数里不会包含 Session B 仍然能看到的那些 tuple 版本。


## Hint bits：为什么 SELECT 也会把页面弄脏

一次页面在新写入之后，第一条触及它的 SELECT 可能会导致这一页被回写到磁盘。不是因为这条 SELECT 改动了任何数据，而是因为它设置了 hint bits。

当 PostgreSQL 遇到一条 t_xmin = 101 的 tuple 并想知道 101 是否 commit 时，它并不会凭空知道答案，必须去 pg_xact（旧名 pg_clog，即 commit log）里查 101 的状态。一旦拿到结果，它就把这个结果缓存到 tuple 的 t_infomask 位上（HEAP_XMIN_COMMITTED 或 HEAP_XMIN_INVALID）。之后的读取者就完全跳过 pg_xact 查询了。

设置这些位属于一次写操作。页面被弄脏。最终会被 flush 出去。你这条无辜的 SELECT 就这么触发了 I/O。

这就是为什么在一张冷表上跑 EXPLAIN (ANALYZE, BUFFERS) 时，哪怕执行计划里只有读操作，你仍可能看到 dirtied buffers 不为零。也是为什么"批量导入后第一次查询"总有那种神秘慢的第一跑：你在为成千上万条新写入页设置 hint bits 付一次性的代价。关于这些计数器如何出现，可以参考 Understanding EXPLAIN Buffers。


## 一段话说清 MVCC 契约

每条 tuple 携带 t_xmin 和 t_xmax。每个事务携带一张 snapshot，即 (xmin, xmax, xip_list)。可见性是对二者的两段式查询比较。UPDATE 和 DELETE 不会原地改字节，它们只是在旧版本上盖 t_xmax，再追加一个新版本。VACUUM 负责清理 dead 版本，但只清那些没有任何 live 事务还可能需要的部分。长事务会阻塞 VACUUM。每一条 SELECT 在第一次看到新数据时都可能把页面弄脏，因为它会把 commit 状态缓存到 hint bits 里。

每条 tuple 8 字节的 XID，加上每个事务三个数字的 snapshot，再加上一个可见性函数。整套机制就这么多，然而它的后果渗透进 PostgreSQL 运维的每一个角落，从 bloat 监控到复制到 autovacuum 调优。

想要完整的字节级导览（hint-bit 编码、visibility map、freezing、XID wraparound），这一系列存储主题的文章会细讲这些。如果你从没亲眼看过 MVCC 是怎么发生的，那 visualizer 是最快建立直觉的方式。让两个 session 互相对打，切一下隔离级别，然后再回到这篇文章。
