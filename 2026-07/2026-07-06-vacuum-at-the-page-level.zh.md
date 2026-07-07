---
title: 页级别的 VACUUM
source: https://boringsql.com/posts/vacuum-at-the-page-level/
author: Radim Marek
published: 2026-07-05
saved: '2026-07-06'
tags:
- postgresql
- vacuum
- database
- storage-internals
---

# 页级别的 VACUUM

在 [Postgres 中的 HOT 更新](https://boringsql.com/posts/hot-updates/) 里，我们讲过页面剪枝如何清理 HOT 链：这是一个优雅的捷径，PostgreSQL 可以在普通读取期间回收死元组空间，而不必等待任何后台进程。但剪枝正是这样一个“捷径”。它只在单个页面内生效，而且只适用于 HOT 更新过的元组。对于其他所有情况（触及索引列的 cold update、普通 DELETE、索引项清理、free space map 注册、visibility map 维护），我们都需要 VACUUM。

本文不会重复 VACUUM 在运维层面做什么。[DELETEs are difficult](https://boringsql.com/posts/deletes-are-difficult/) 一文已经讲过 autovacuum 调优、worker 分配，以及死元组清理的运维侧内容。这里我们要逐字节观察 VACUUM 的工作方式。我们会在每个阶段前后对页面做快照，精确跟踪 page header、line pointer、tuple header、free space map 和 visibility map 中到底发生了什么变化。工具仍然一样：`pageinspect`、`pg_visibility` 和 `pg_freespacemap`。

## 设置

我们需要一张有足够多行的表，让前后对比有意义；还需要索引，用来展示完整的 VACUUM 周期。

```
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE EXTENSION IF NOT EXISTS pg_visibility;
CREATE EXTENSION IF NOT EXISTS pg_freespacemap;

CREATE TABLE vacuum_demo (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category text NOT NULL,
    payload text
);

INSERT INTO vacuum_demo (category, payload)
SELECT
    'cat_' || (i % 5),
    repeat('x', 100)
FROM generate_series(1, 50) AS i;
```

五十行，每行带一个 100 字节的 payload。主键给了我们一个索引，这一点很重要：当涉及索引时，VACUUM 的行为会变化。先手动运行一次 VACUUM，让我们从干净的基线开始：

```
VACUUM vacuum_demo;
```

## 删除前的快照

记录 page 0 的基线状态。先看 page header：

```
SELECT lower, upper, special, pagesize
FROM page_header(get_raw_page('vacuum_demo', 0));
```

```
 lower | upper | special | pagesize
-------+-------+---------+----------
   224 |  1392 |    8192 |     8192
(1 row)
```

`pd_lower` 是 224：也就是 24 字节的 page header，加上 50 个 line pointer，每个 4 字节（24 + 200 = 224）。`pd_upper` 是 1392，所以我们的元组占用了从 1392 到 8191 的字节区间。空闲空间是 1392 - 224 = 1168 字节。剩余空间不多；这些 100 字节的 payload 加起来很快。

现在看 line pointer 和 tuple header：

```
SELECT lp, lp_flags, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
LIMIT 10;
```

```
 lp | lp_flags | lp_off | lp_len | t_xmin | t_xmax | t_ctid
----+----------+--------+--------+--------+--------+--------
  1 |        1 |   8056 |    135 |    746 |      0 | (0,1)
  2 |        1 |   7920 |    135 |    746 |      0 | (0,2)
  3 |        1 |   7784 |    135 |    746 |      0 | (0,3)
  4 |        1 |   7648 |    135 |    746 |      0 | (0,4)
  5 |        1 |   7512 |    135 |    746 |      0 | (0,5)
  6 |        1 |   7376 |    135 |    746 |      0 | (0,6)
  7 |        1 |   7240 |    135 |    746 |      0 | (0,7)
  8 |        1 |   7104 |    135 |    746 |      0 | (0,8)
  9 |        1 |   6968 |    135 |    746 |      0 | (0,9)
 10 |        1 |   6832 |    135 |    746 |      0 | (0,10)
(10 rows)
```

`lp_len` 是 135，也就是元组的实际字节长度，但每个元组在页面上占据的是经过 MAXALIGN 对齐后的 136 字节槽位；注意 `lp_off` 的值每次递减 136。下面的空闲空间计算用的就是这个对齐后的步长。

每个 line pointer 都是 LP_NORMAL（`lp_flags = 1`）。每个元组都有 `t_xmax = 0`：自插入以来没有人碰过这些行。每个 `t_ctid` 都指向自己。这是一个完全干净的页面。

## 创建死元组

现在制造一些死行：

```
DELETE FROM vacuum_demo WHERE id % 3 = 0;
```

```
DELETE 16
```

这大约删除了每三行中的一行：ID 3、6、9、12，依此类推。现在有 16 行已经死亡。在 VACUUM 运行前看看页面：

```
SELECT lp, lp_flags, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
LIMIT 10;
```

```
 lp | lp_flags | lp_off | lp_len | t_xmin | t_xmax | t_ctid
----+----------+--------+--------+--------+--------+--------
  1 |        1 |   8056 |    135 |    746 |      0 | (0,1)
  2 |        1 |   7920 |    135 |    746 |      0 | (0,2)
  3 |        1 |   7784 |    135 |    746 |    747 | (0,3)
  4 |        1 |   7648 |    135 |    746 |      0 | (0,4)
  5 |        1 |   7512 |    135 |    746 |      0 | (0,5)
  6 |        1 |   7376 |    135 |    746 |    747 | (0,6)
  7 |        1 |   7240 |    135 |    746 |      0 | (0,7)
  8 |        1 |   7104 |    135 |    746 |      0 | (0,8)
  9 |        1 |   6968 |    135 |    746 |    747 | (0,9)
 10 |        1 |   6832 |    135 |    746 |      0 | (0,10)
(10 rows)
```

看第 3、6、9 行。它们的 `t_xmax` 现在是 747，也就是 DELETE 语句的事务 ID。但其他一切都没变。`lp_flags` 仍然是 1（LP_NORMAL）。`lp_off` 和 `lp_len` 也相同。元组仍然物理地待在页面上，占用空间。page header 也没有变化：

```
SELECT lower, upper, special, pagesize
FROM page_header(get_raw_page('vacuum_demo', 0));
```

```
 lower | upper | special | pagesize
-------+-------+---------+----------
   224 |  1392 |    8192 |     8192
(1 row)
```

`pd_lower` 和 `pd_upper` 与 DELETE 前完全相同。PostgreSQL 把这些行标记为死亡（通过写入 `t_xmax`），但没有回收哪怕一个字节。死元组就是 bloat，并且会一直这样，直到 VACUUM 到来。

## VACUUM 如何处理一张表

在我们运行 VACUUM 并观察页面变化之前，先理解一件关于它“如何运行”的事，这会解释接下来快照中看到的一切。VACUUM 分三轮完成工作，而这三轮之间的分割，正是一个已删除元组的存储空间在某个时刻消失、而它的 line pointer 在另一个时刻消失的全部原因。请留意这中间的间隙：本文后面的内容就是要把它显现出来。

### 阶段 1：Heap scan - prune, freeze, collect dead TIDs

VACUUM 会顺序扫描每个 heap page，而第一轮做的远不只是查看。对于每个页面，它会运行 page pruning（也就是普通读取期间机会性触发的同一套 `heap_page_prune_and_freeze` 机制）。剪枝是真正把字节拿回来的地方：它移除死元组的存储、对页面做碎片整理，并推进 `pd_upper`。它也会机会性地冻结足够老的元组。

在 PostgreSQL 17 之前，VACUUM 会把死 TID 存进一个从 `maintenance_work_mem` 分配出来的扁平数组。如果数组满了，VACUUM 就必须暂停，先对已经收集到的这一批执行索引和 heap 清理，然后再继续扫描。从 PostgreSQL 17 开始，VACUUM 使用基于 radix tree 的 TID store，内存效率高得多，使得 `maintenance_work_mem` 不太容易成为瓶颈。

但本文后面依赖的微妙点在这里。剪枝不能简单地把已删除元组的 line pointer 标记为 LP_UNUSED，因为索引仍然通过 TID 指向它。所以对于带索引的表，一个死元组的 line pointer 会被设为 **LP_DEAD**：它的存储已经没了，但这个 4 字节槽位仍保留在原地，占住这个 TID 的位置，直到索引项被移除。这些 LP_DEAD TID 就是 VACUUM 收集到 dead-TID store 里、供下一阶段使用的内容。

### 阶段 2：Index cleanup

拿到死 TID 列表后，VACUUM 会扫描表上的每个索引。对于每个索引，它遍历所有索引项，并移除任何指向死 TID 的条目。这是昂贵的部分。VACUUM 必须读取每个索引页，即使只有少量条目需要移除。

这也是索引膨胀发生的原因。如果 VACUUM 无法完成这一阶段——可能是因为一个长事务拖住了可见性边界，也可能是因为表上有很多索引且死 TID 列表超过内存——指向死元组的索引项就会积累。我们在 [VACUUM Is a Lie](https://boringsql.com/posts/vacuum-is-lie/) 中详细讨论过索引膨胀的影响。

### 阶段 3：Heap cleanup - freeing the line pointers

索引清干净之后，就不再有索引项引用这些死 TID 了，因此保留的槽位终于可以释放。VACUUM 会重新访问每个曾经有死元组的页面，做它在阶段 1 中不能做的事：

1. 把每个收集到的 LP_DEAD line pointer 翻转为 LP_UNUSED（0），回收槽位
2. 如果所有剩余元组对所有事务可见，则设置 visibility map bit

一张**没有索引**的表会完全跳过这种分裂：由于没有索引项需要担心，阶段 1 的剪枝会在单次 heap pass 中直接把死 line pointer 设为 LP_UNUSED，也就没有阶段 3。LP_DEAD 先出现、LP_UNUSED 后出现的两轮舞蹈，正是因为索引存在。

注意不在上面列表里的东西。元组数据早在阶段 1 的剪枝中就已经被移除，页面也已经碎片整理过；`pd_upper` 就是在那里移动的。阶段 3 回收的是 line-pointer 槽位，而不是元组字节。如果只看普通 `VACUUM` 前后，这个区别是看不见的，所以我们把它显现出来。

我们分两步运行 VACUUM，捕捉中间状态。`VACUUM (INDEX_CLEANUP OFF)` 会执行阶段 1 的剪枝，但跳过索引清理，因此也跳过第二次 heap pass；它别无选择，只能把死 line pointer 留作 LP_DEAD。

**交互式 VACUUM 可视化器**

跨三个 heap page 运行 DELETE，观察什么都没有被回收；再执行 page-prune 到达 `LP_DEAD`；然后逐步让 VACUUM 经过 heap scan、index cleanup 和 heap cleanup，把槽位翻转为 `LP_UNUSED`。

[打开可视化器](https://boringsql.com/visualizers/vacuum/)

## 剪枝之后：LP_DEAD，且空间已经回来

剪枝后的 page header：

```
SELECT lower, upper, special, pagesize
FROM page_header(get_raw_page('vacuum_demo', 0));
```

```
 lower | upper | special | pagesize
-------+-------+---------+----------
   224 |  3568 |    8192 |     8192
(1 row)
```

`pd_lower` 仍然是 224；line pointer 数组没有缩小。但 `pd_upper` 从 1392 跳到了 3568。这意味着回收了 2176 字节空间（16 个死元组，每个按 136 字节对齐步长计算 = 2176）。页面上的空闲空间从 1168 字节变成 3344 字节。而我们还没有碰过索引：这一切都发生在第一次 heap pass 的剪枝期间。

现在看 line pointer：

```
SELECT lp, lp_flags, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
LIMIT 10;
```

```
 lp | lp_flags | lp_off | lp_len | t_xmin | t_xmax | t_ctid
----+----------+--------+--------+--------+--------+--------
  1 |        1 |   8056 |    135 |    746 |      0 | (0,1)
  2 |        1 |   7920 |    135 |    746 |      0 | (0,2)
  3 |        3 |      0 |      0 |        |        |
  4 |        1 |   7784 |    135 |    746 |      0 | (0,4)
  5 |        1 |   7648 |    135 |    746 |      0 | (0,5)
  6 |        3 |      0 |      0 |        |        |
  7 |        1 |   7512 |    135 |    746 |      0 | (0,7)
  8 |        1 |   7376 |    135 |    746 |      0 | (0,8)
  9 |        3 |      0 |      0 |        |        |
 10 |        1 |   7240 |    135 |    746 |      0 | (0,10)
(10 rows)
```

line pointer 3、6、9 现在是 **`lp_flags = 3`（LP_DEAD）**，不是 LP_UNUSED。它们的存储已经没了（`lp_off` 和 `lp_len` 为 0），但槽位仍在那里，保留这些 TID。它们还不能释放，因为主键索引仍然指向它们：

```
SELECT live_items FROM bt_page_stats('vacuum_demo_pkey', 1);
```

```
 live_items
------------
         50
(1 row)
```

索引项仍然是 50 个，和删除前一样。我们跳过的索引清理，正是移除那 16 个过期条目的动作；在它运行之前，这些 LP_DEAD 槽位会一直卡住。

## 完整 VACUUM 之后：LP_UNUSED

现在运行一次普通 VACUUM 来完成工作：

```
VACUUM vacuum_demo;

SELECT lower, upper, special, pagesize
FROM page_header(get_raw_page('vacuum_demo', 0));
```

```
 lower | upper | special | pagesize
-------+-------+---------+----------
   224 |  3568 |    8192 |     8192
(1 row)
```

`pd_upper` 仍然是 3568。这正是重点：第二次 heap pass 不再回收任何元组字节；已经没有元组字节可回收了，剪枝早就拿走了它们。它做的是在索引清干净之后释放 line-pointer 槽位：

```
SELECT lp, lp_flags, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
LIMIT 10;
```

```
 lp | lp_flags | lp_off | lp_len | t_xmin | t_xmax | t_ctid
----+----------+--------+--------+--------+--------+--------
  1 |        1 |   8056 |    135 |    746 |      0 | (0,1)
  2 |        1 |   7920 |    135 |    746 |      0 | (0,2)
  3 |        0 |      0 |      0 |        |        |
  4 |        1 |   7784 |    135 |    746 |      0 | (0,4)
  5 |        1 |   7648 |    135 |    746 |      0 | (0,5)
  6 |        0 |      0 |      0 |        |        |
  7 |        1 |   7512 |    135 |    746 |      0 | (0,7)
  8 |        1 |   7376 |    135 |    746 |      0 | (0,8)
  9 |        0 |      0 |      0 |        |        |
 10 |        1 |   7240 |    135 |    746 |      0 | (0,10)
(10 rows)
```

line pointer 3、6、9 已经从 LP_DEAD 变成 **`lp_flags = 0`（LP_UNUSED）**。这些槽位现在为空，可以被下一次落到这个页面上的 INSERT 复用。索引也降到了 34 个条目；那 16 个过期条目已经没了：

```
SELECT live_items FROM bt_page_stats('vacuum_demo_pkey', 1);
```

```
 live_items
------------
         34
(1 row)
```

存活的 heap 元组早在剪枝时就已经被压实；注意 `lp_off` 值已经从 VACUUM 前的位置移动了，因为 VACUUM 移动了存活元组，在 `pd_lower` 和 `pd_upper` 之间形成了一整块连续空闲空间。

**`pd_lower` 没有变化。** 即使 line pointer 3、6、9 现在已经是 LP_UNUSED，它们仍然占据数组中的槽位。下一次 INSERT 会复用这些槽位之一，而不是在数组末尾追加一个新槽位。PostgreSQL 通过 page header 中的 `PD_HAS_FREE_LINES` 标志跟踪可复用槽位。

## line pointer 的生命周期，精确版

我们精确说明 line pointer 如何在状态之间转换。这很重要，因为不同代码路径会产生不同转换：

**LP_UNUSED (0)**：槽位为空。它要么从未被使用过，要么已经被 VACUUM 完全回收。下一次以这个页面为目标的 INSERT 会抓取这个槽位并分配给新元组，把它转换为 LP_NORMAL。

**LP_NORMAL (1)**：槽位指向页面上的一个元组。该元组可能是活的（`t_xmax = 0` 或 `t_xmax` 已中止），也可能是死的（`t_xmax` 已提交且对所有事务不可见）。line pointer 本身不编码存活性；存活性由 tuple header 和我们在 [PostgreSQL MVCC, Byte by Byte](https://boringsql.com/posts/postgresql-mvcc-byte-by-byte/) 中讲过的可见性规则决定。

**LP_REDIRECT (2)**：槽位不是指向元组，而是指向另一个 line pointer。这由 HOT pruning 创建：当 HOT 链的头部被剪枝时，它的 line pointer 会变成 redirect，使得索引（它们仍然引用原始 line pointer 编号）可以跟随 redirect 找到当前版本。我们在 [Postgres 中的 HOT 更新](https://boringsql.com/posts/hot-updates/) 中见过这个过程。

**LP_DEAD (3)**：槽位已知包含一个死元组，它的存储已经被回收，但它的 TID 可能仍被索引项引用。遇到它的索引扫描会立刻跳过它，而不进行可见性检查。在 heap 上，它由 page pruning 设置，包括 VACUUM 自己第一次 heap pass 中的剪枝。索引侧也携带同样的 LP_DEAD hint，而且普通 reader 会设置它：当一次索引扫描沿着某个条目找到 heap 元组，并发现该元组已经对所有人死亡时，`kill_prior_tuple` 优化会把这个索引项标记为 LP_DEAD，让后续扫描无需访问 heap 就能跳过它；这类清理工作在 VACUUM 到来前很久，就已经由普通 SELECT 完成。对于有索引的表，普通删除后的元组在剪枝和索引清理之间正处于这个状态，就像我们上面观察到的一样。

这些转换如下：

```
LP_UNUSED (0) ----INSERT----> LP_NORMAL (1)

-- table WITH indexes: two heap passes
LP_NORMAL (1) --prune (1st pass)--> LP_DEAD (3)
LP_DEAD (3) --index cleanup + 2nd pass--> LP_UNUSED (0)

-- table WITHOUT indexes: single heap pass
LP_NORMAL (1) --prune--> LP_UNUSED (0)

-- HOT
LP_NORMAL (1) --HOT prune (chain head)--> LP_REDIRECT (2)
LP_NORMAL (1) --HOT prune (heap-only chain member)--> LP_UNUSED (0)
LP_REDIRECT (2) --VACUUM (when target is also dead)--> LP_UNUSED (0)
```

一个已删除元组是否经过 LP_DEAD，取决于表是否有索引。有索引时，剪枝不能释放槽位（索引项仍然引用该 TID），所以它把 line pointer 暂停在 LP_DEAD；只有在索引清理之后，第二次 heap pass 才能把它变成 LP_UNUSED。没有索引时，没有东西会引用该 TID，所以剪枝会在单次 pass 中直接把它设为 LP_UNUSED。不管哪种情况，元组的*存储*都是在剪枝期间被回收的；LP_DEAD 只关乎那个 4 字节槽位的命运。

## Free space map

VACUUM 之前，PostgreSQL 的 free space map（FSM）并不知道我们的页面包含可回收空间。死元组对 FSM 是不可见的；它们看起来仍像被占用的字节。VACUUM 之后，回收的空间被登记进去：

```
SELECT blkno, avail
FROM pg_freespace('vacuum_demo');
```

```
 blkno | avail
-------+-------
     0 |  3328
(1 row)
```

page 0 现在报告有 3,328 字节可用空间。这比原始的 `pd_upper - pd_lower` 值 3,344 略小，因为 FSM 用大约 32 字节的粗粒度类别跟踪空闲空间，并向下取整。但关键点是，这个页面现在成为新 INSERT 的候选位置。

如果没有这次 FSM 更新，PostgreSQL 在插入新行时会跳过这个页面，并用一个新页面扩展表文件。这就是表即使内部有大量空闲空间也会继续增长的原因：FSM 没有被更新，因为 VACUUM 从未运行。

```
-- New inserts reuse the vacuumed space instead of extending the file
INSERT INTO vacuum_demo (category, payload)
SELECT 'cat_new', repeat('y', 100)
FROM generate_series(1, 10) AS i;

SELECT pg_relation_size('vacuum_demo') AS size;
```

```
 size
------
 8192
(1 row)
```

这张表仍然只有一个 8 KB 页面。这 10 条新行落在 VACUUM 在 page 0 上释放出来的空间里；FSM 直接把 inserter 指向了它，所以没有追加新页面。如果 DELETE 之后没有先 VACUUM 就 INSERT，故事会不一样：FSM 仍会报告该页面已满，inserter 会跳过它，文件会增长。这就是一张表在磁盘上持续膨胀、但内部却半空的机制。

## Visibility map

visibility map 为每个 heap page 跟踪两个 bit。我们刚插入的 10 行让 page 0 上出现了新的、尚未 all-visible 的元组，所以当前该 bit 被清除。再运行一次 VACUUM 让页面稳定下来，现在它被设置了：

```
VACUUM vacuum_demo;

SELECT blkno, all_visible, all_frozen
FROM pg_visibility('vacuum_demo');
```

```
 blkno | all_visible | all_frozen
-------+-------------+------------
     0 | t           | f
(1 row)
```

page 0 被标记为 **all_visible**。这意味着页面上当前每个元组都对所有事务可见。其影响很重要：

- **Index-only scan** 可以直接从索引返回结果，而完全不获取这个 heap page。visibility map 确认这个页面上的任何东西都是可见的，因此索引中的数据副本保证是正确的。
- **未来的 VACUUM pass** 可能在 heap scan 阶段跳过这个页面，因为里面没有死元组可找。

visibility map 作为一个单独的 fork 文件，与主 heap 文件并列存储。对于 `base/16384/24601` 中的一张表，visibility map 是 `base/16384/24601_vm`。它非常小：每页两个 bit 意味着一张 1 GB 的表（131,072 个页面）只需要大约 32 KB 的 visibility map。

**all_frozen** bit 仍然是 false。这个标志表示更强的性质：不仅所有元组都可见，而且它们所有的 xmin 值都已经冻结；它们可以在 XID wraparound 中存活，而无需任何进一步关注。

如果我们现在从这个页面删除一行，all_visible bit 会被清除：

```
DELETE FROM vacuum_demo WHERE id = 1;

SELECT blkno, all_visible, all_frozen
FROM pg_visibility('vacuum_demo');
```

```
 blkno | all_visible | all_frozen
-------+-------------+------------
     0 | f           | f
(1 row)
```

页面上只要有一个死元组，就足以让 all_visible 标志失效。PostgreSQL 会在 DELETE 发生的那一刻积极地清除这个 bit，因为它必须保守。Index-only scan 依赖这个标志的正确性。

## Freezing

让页面进入最终静止状态。运行 `VACUUM FREEZE` 会强制 PostgreSQL 冻结表上的每个元组，无论年龄如何：

```
VACUUM FREEZE vacuum_demo;
```

现在检查元组：

```
SELECT lp, t_xmin, t_infomask,
    CASE WHEN (t_infomask & 256) > 0 AND (t_infomask & 512) > 0 THEN 'FROZEN'
         WHEN (t_infomask & 256) > 0 THEN 'XMIN_COMMITTED'
         ELSE 'no hint bits' END AS freeze_status
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
WHERE lp_flags = 1
LIMIT 8;
```

```
 lp | t_xmin | t_infomask | freeze_status
----+--------+------------+---------------
  2 |    746 |       2818 | FROZEN
  3 |    748 |       2818 | FROZEN
  4 |    746 |       2818 | FROZEN
  5 |    746 |       2818 | FROZEN
  6 |    748 |       2818 | FROZEN
  7 |    746 |       2818 | FROZEN
  8 |    746 |       2818 | FROZEN
  9 |    748 |       2818 | FROZEN
(8 rows)
```

在较老的 PostgreSQL 版本（9.4 之前）中，冻结会真的把 `t_xmin` 重写为 2（FrozenTransactionId）。现代 PostgreSQL 会保留原始 xmin 值，并改为设置 infomask bit。这对调试更好：你仍然可以看到是哪一个事务最初创建了这行。

line pointer 1 已经消失了：那是 `id = 1`，也就是我们刚删除的行；`VACUUM FREEZE` 在冻结其余元组前把它剪枝掉了。line pointer 3、6、9 现在带有 `t_xmin = 748`：它们是 free space map 小节中的 `cat_new` 行，正好复用了早先 VACUUM 释放的那些槽位。每个存活元组都是 FROZEN。`t_infomask` 值 2818 是 `0x0B02`，也就是 `HEAP_XMIN_COMMITTED`（0x0100）加上 `HEAP_XMIN_INVALID`（0x0200）加上 `HEAP_XMAX_INVALID`（0x0800），再加上 `HEAP_HASVARWIDTH` 属性 bit（0x0002），这里每行都因为有 `text` 列而携带这个 bit。正如我们在 [PostgreSQL MVCC, Byte by Byte](https://boringsql.com/posts/postgresql-mvcc-byte-by-byte/) 中讲过的，同时拥有 XMIN_COMMITTED 和 XMIN_INVALID 的组合就是 freeze marker。它表示“无论 XID 计数器如何变化，这个元组的 xmin 都永久处于过去”。

我们还可以验证表级别的 freeze horizon：

```
SELECT relfrozenxid FROM pg_class WHERE relname = 'vacuum_demo';
```

```
 relfrozenxid
--------------
          750
(1 row)
```

这告诉 PostgreSQL：`vacuum_demo` 中每个元组的 xmin 要么已经冻结，要么比 XID 750 更新。Autovacuum 用这个值决定何时需要 anti-wraparound freezing。由于我们刚冻结了一切，这个 horizon 前进到了当前事务 ID。

visibility map 现在也显示两个标志都已设置：

```
SELECT blkno, all_visible, all_frozen
FROM pg_visibility('vacuum_demo');
```

```
 blkno | all_visible | all_frozen
-------+-------------+------------
     0 | t           | t
(1 row)
```

**all_frozen** bit 现在为 true。未来的 VACUUM pass，包括 anti-wraparound freezing pass，都可以完全跳过这个页面。这里没有东西要清理，也没有东西要冻结。对于大型且大多静态的表（lookup table、历史数据、归档分区），这是很大的性能收益：VACUUM 只需检查每页两个 bit，就能在几秒内跳过数百万页面，而不是读取每个元组。

## VACUUM FULL 与普通 VACUUM

到目前为止我们看到的都是普通（lazy）VACUUM。有一个持久的误解说它永远不能缩小磁盘上的表文件。这不完全正确：在运行末尾，lazy VACUUM 会尝试截断已经完全变空的尾部页面，把这部分空间交还给操作系统。它*不能*做的是压缩滞留在文件内部的空闲空间：被满页包围的半空页面。这个区别就是完整故事，而且两半都很容易看到。

从一张填满 18 个页面的表开始：

```
CREATE TABLE vt (id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY, category text, payload text);
INSERT INTO vt (category, payload)
SELECT 'cat_' || (i % 5), repeat('x', 100)
FROM generate_series(1, 1000) AS i;

SELECT pg_relation_size('vt') AS full_size;
```

```
 full_size
-----------
    147456
(1 row)
```

147,456 字节就是 18 个页面。现在删除前 50 行之后的所有行（前 50 行都在 page 0 上），然后 vacuum：

```
DELETE FROM vt WHERE id > 50;

VACUUM vt;
SELECT pg_relation_size('vt') AS after_regular_vacuum;
```

```
 after_regular_vacuum
----------------------
                 8192
(1 row)
```

降到了单个页面。第一个页面之后的每个页面都完全变空了，所以这 17 个页面位于文件尾部且没有任何 live tuple，VACUUM 的 truncation 阶段把它们还给了 OS。普通 VACUUM *确实*缩小了文件，因为空闲空间刚好全在末尾。

现在看它无能为力的情况。重建表并删除每隔一行，这样没有任何页面会完全变空：

```
DROP TABLE vt;
CREATE TABLE vt (id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY, category text, payload text);
INSERT INTO vt (category, payload)
SELECT 'cat_' || (i % 5), repeat('x', 100)
FROM generate_series(1, 1000) AS i;

DELETE FROM vt WHERE id % 2 = 0;

VACUUM vt;
SELECT pg_relation_size('vt') AS after_regular_vacuum;
```

```
 after_regular_vacuum
----------------------
               147456
(1 row)
```

仍然是 18 个页面。一半行已经没了，VACUUM 也把所有空闲空间登记进 FSM，但每个页面仍然持有 live tuple，所以没有尾部空页面可以截断。文件无法缩小；空闲空间滞留在文件内部。这正是人们说“我删掉半张表，但磁盘上的大小还是一样”时所指的 bloat。

这就是 VACUUM FULL 的用途：

```
VACUUM FULL vt;
SELECT pg_relation_size('vt') AS after_vacuum_full;
```

```
 after_vacuum_full
-------------------
             73728
(1 row)
```

从 147 KB 降到 73 KB，从 18 个页面降到 9 个。VACUUM FULL 把整张表重写到一个新文件中，紧密打包 500 个存活行，并删除旧文件。普通 VACUUM 已经让那些空间可复用，但只有 VACUUM FULL 能把它从文件里压缩出去。

VACUUM FULL 需要 **AccessExclusiveLock**：在整个持续期间，它会阻塞该表上的所有读写。对于大表，这可能意味着该表数分钟甚至数小时的完全停机。在生产环境中，可以考虑 [pg_repack 或 pg_squeeze](https://boringsql.com/posts/the-bloat-busters-pg_repack-pg_squeeze/) 这类在线替代方案，它们能在没有排他锁的情况下实现同样的压缩效果。

权衡很清楚：普通 VACUUM 很轻量（ShareUpdateExclusiveLock，不阻塞读写），会让空间可复用，并且只在尾部页面为空时截断文件。VACUUM FULL 可以把内部空闲空间压缩出文件，但会获取 AccessExclusiveLock 并阻塞一切。对大多数工作负载来说，普通 VACUUM 已经足够：新的 INSERT 会复用释放出来的空间，表会达到稳定大小。VACUUM FULL 是 bloat 已经失控时的修复工具。

## 汇总

让我们追踪一个元组从 INSERT 到 VACUUM 的完整生命周期，让每个状态变化都可见：

**INSERT**

- line pointer 被分配为 LP_NORMAL（1），指向新元组
- 元组被打上 `t_xmin = inserting XID`、`t_xmax = 0`
- `pd_lower` 前进 4 字节（新的 line pointer）；`pd_upper` 按元组大小下降

**DELETE**

- 元组的 `t_xmax` 被设为删除事务的 XID
- 该页面的 visibility map all_visible bit 被清除
- 没有空间被回收，line pointer 没有变化，`pd_lower`/`pd_upper` 也没有变化

**VACUUM 阶段 1（heap scan + prune）**

- 每个页面被剪枝：死元组数据被移除，页面被碎片整理
- `pd_upper` 增加；free space map 被更新
- 死 line pointer 被设为 LP_DEAD（有索引的表）或 LP_UNUSED（无索引）
- 死 TID 被收集；元组被机会性冻结

**VACUUM 阶段 2（index cleanup）**

- 从每个索引中移除指向死 TID 的索引项
- heap page 不被触碰

**VACUUM 阶段 3（heap cleanup，仅有索引的表）**

- LP_DEAD line pointer 被翻转为 LP_UNUSED（0），槽位可复用
- 此处不回收元组字节；剪枝已经在阶段 1 做完了
- visibility map all_visible bit 被设置（如果所有剩余元组都可见）

**VACUUM FREEZE**

- tuple infomask bit 被设为 XMIN_COMMITTED + XMIN_INVALID（frozen）
- visibility map all_frozen bit 被设置
- `pg_class.relfrozenxid` 前进

line pointer 会沿着我们上面追踪的状态机移动（有索引的表中从 LP_NORMAL 到 LP_DEAD 再到 LP_UNUSED；无索引时直接到 LP_UNUSED），但阶段追踪解释的是*为什么*。

每次 INSERT 都会创建一个 line pointer。每次 DELETE 都会写入 `t_xmax`，但不会改变页面上的其他任何东西。每个 VACUUM 周期（prune、index cleanup、heap cleanup）都会把死空间转回可复用空间。页面永远不会超过 8 KB，而 VACUUM 的职责就是确保这 8,192 字节中尽可能多的空间可用于 live data。
