---
title: "DuckDB Internals: Why is DuckDB Fast? (Part 1)"
source: "https://www.greybeam.ai/blog/duckdb-internals-part-1"
author: "Kyle Cheung"
published: "2026-05-04"
saved: "2026-06-19T23:19:31"
tags: [duckdb, database, internals, clipping]
---

# DuckDB 内部机制：DuckDB 为什么这么快？（第 1 部分）

[DuckDB](https://duckdb.org/?ref=greybeam-blog.ghost.io) 已经从 2019 年 CWI Amsterdam 的一个研究项目，成长为过去十年中采用最广泛的数据库之一。它出现的场景很长：notebooks、ETL pipelines、dashboards、CI test runners、SaaS 产品内的嵌入式分析，甚至还有[一台以 scale factor 100 运行 TPC-H 的 iPhone](https://duckdb.org/2024/12/06/duckdb-tpch-sf100-on-mobile?ref=greybeam-blog.ghost.io)。

![一台放在干冰盒子里、正在运行 TPC-H 的 iPhone。](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-01.png)

一台放在干冰盒子里、正在运行 TPC-H 的 iPhone。([来源](https://duckdb.org/2024/12/06/duckdb-tpch-sf100-on-mobile?ref=greybeam-blog.ghost.io))

一些公司已经开始围绕它构建真正的产品。MotherDuck 正在把 DuckDB 包装成一个云数据仓库。Hex、Omni 和 Evidence 等 BI 与数据应用平台将它用作应用内执行引擎和缓存。Fivetran 的 Managed Data Lake Service 在其 data-lake writer 中使用 DuckDB 进行合并和压缩。Rill 在它之上构建了一个开源 BI 工具。我们在 Greybeam 也使用它，为 BI 和分析工作负载驱动数百万次查询。

## [什么是 DuckDB？#](#what-is-duckdb)

DuckDB 是一个**进程内分析型 SQL 数据库**。*分析型*意味着它针对那类扫描数百万行以进行过滤、聚合和连接的查询做了优化——而不是那类通过主键查找单条记录的查询。*进程内*意味着没有服务器。你不是连接到 DuckDB；你是像加载 NumPy 或 Polars 一样，把它作为库加载到你的程序中。

DuckDB 得到广泛采用，是因为它实在太容易使用了。它以一个小于 20 MB、没有外部依赖的单个二进制文件发布。你可以用 `pip install duckdb`、`brew install duckdb` 安装它，或者把 `libduckdb` 链接到一个 C++ 项目中。它可以像这些文件已经是 SQL 数据库一样，打开任何包含 Parquet、CSV 或 JSON 文件的目录。

DuckDB 也恰好是目前最快的单节点分析引擎之一，经常能与每年花费数百万美元的整套集群相抗衡。

---

这是关于 DuckDB 内部机制三部分深度解析中的第一篇。我们会跟随一条查询，从它进入引擎的那一刻开始，直到结果返回；在每个阶段，我们都会考察让它变快的设计选择。

DuckDB 的速度来自少数几个设计选择：

1. **进程内执行**
2. **带 zonemap 的列式压缩存储**
3. **向量化执行**
4. **Morsel-driven parallelism**
5. **带乐观 MVCC 的快照隔离**
6. **以及更多！**

本文涵盖从你的 SQL 到引擎准备好运行查询这一刻的路径，以及查询将要读取的存储层。读完之后，你会对 DuckDB 的准备工作和存储布局形成清晰的心智模型。查询执行将在第 2 部分介绍，所以一定要订阅！

## [**查询在进程内运行**#](#queries-run-in-process)

你让 DuckDB 指向笔记本电脑上的一个 6 GB Parquet 文件。结果不到一秒就返回了。没有集群，没有设置，没有迁移，没有 `CREATE TABLE`。这是怎么做到的？

```
SELECT
  *
FROM 'orders.parquet';
```

Copy

大多数分析型数据库都是服务器。Snowflake、Postgres、BigQuery、Redshift。你打开一个连接，通过 TCP（一种通过网络发送数据的协议）发送 SQL，然后等待结果返回。在这个过程中，结果中的每条记录都会被*序列化*成一种 wire protocol，通过网络传输，并在另一端被*反序列化*。

### [序列化与反序列化#](#serializing-and-deserializing)

在数据库内部，查询结果以特定内存地址上的有类型值存在。这里是一个 64-bit integer，那里是一个指向字符串的指针。这些地址只存在于那个进程中。为了把结果发送给另一台机器上的客户端，数据库必须把每个值重写成约定好的字节格式（Postgres 有自己的格式，MySQL 有另一种，ODBC 和 JDBC 则是驱动在其上暴露的客户端 API），这样才能通过 TCP socket 推送出去。然后客户端再把这些字节解析回自己的原生类型。每个值可能会被触碰多次，一次用于编码，一次用于解码；在大型结果集上，这项工作往往比查询本身花的时间还长。

![从客户端到服务器的网络图。](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-02.png)

DuckDB 不是服务器。它是一个库。没有 DuckDB daemon，没有端口，没有集群。你把 `libduckdb` 加载到你的程序里，并直接对它调用函数。

2017 年，Mark Raasveldt 和 Hannes Mühleisen 发表了 [*Don't Hold My Data Hostage*](https://duckdb.org/library/dont-hold-my-data-hostage/?ref=greybeam-blog.ghost.io)，这篇论文测量了当你从一个 warehouse 中取出结果集时实际发生了什么。他们发现，客户端协议本身——ODBC、JDBC 以及类似的逐行逐值 API——常常是整个查询中最慢的单一步骤，有时甚至让数据库计算答案所花的时间相形见绌。

两个成本推动了这一点。第一个是原始带宽：典型的 gigabit Ethernet 链路上限大约是 125 MB/s，而大型结果集的传输时间可能比计算时间更长。第二个是每个值的开销。ODBC 和 JDBC 一次返回一行、一个值，这意味着客户端要为每一行中的每个字段单独调用一次函数。对于一个 1 亿行的结果，这就是数亿次函数调用，每次调用都在做自己的小规模内存复制、类型检查和字符串分配。

> ADBC 以列式 Arrow 格式在系统之间传输数据，从而避免 ODBC 和 JDBC 所要求的逐行序列化/反序列化。我们在 [Columnar](https://columnar.tech/?ref=greybeam-blog.ghost.io) 的朋友们正在让这件事变得普遍。

![](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-03.png)

连接到 Snowflake 时 ODBC 和 ADBC 的差异。

DuckDB 通过与客户端生活在同一个进程中，绕开了这两个瓶颈。

当 Python 脚本针对一个 pandas dataframe 运行 `con.sql("SELECT ... FROM my_df")` 时，DuckDB 可以使用一种叫做 [replacement scan](https://duckdb.org/docs/current/clients/c/replacement_scans?ref=greybeam-blog.ghost.io) 的功能。DuckDB 不是先把 dataframe 复制到内部表中，而是把表引用替换成一个在查询运行时从 dataframe 读取的函数。

在最好的情况下，DuckDB 可以读取 Python 进程已经拥有的同一批底层 buffers，因此避免物化第二份完整的数据副本。这就是 zero-copy！如果 NumPy 说“这里有一块包含 100 万个 int64 值的 buffer（连续内存块）”，DuckDB 通常可以直接读取同一个 buffer，因为它理解相同的物理布局。

实践中，这条路径是否真正 zero-copy 取决于 dataframe 的物理布局、列类型、null 表示以及字符串存储。如果类型或布局不匹配，DuckDB 可能会为某些列分配转换后的 buffers。

[Arrow](https://arrow.apache.org/?ref=greybeam-blog.ghost.io) 是这个故事最干净的版本，因为 Arrow 本身就是一种列式、有类型的内存格式，专为系统之间共享数据而设计。这就是为什么将 DuckDB 结果作为 Arrow 返回，或者查询 Arrow-backed data，可以避免传统 API 强加的大部分逐行转换开销。

## [**从 SQL 到逻辑计划**#](#from-sql-to-logical-plan)

一旦你的 SQL 到达 DuckDB，它会经历常规阶段：parse、bind、plan、optimize。

### [解析#](#parsing)

第一步是把 SQL 解析成抽象语法树（AST）。DuckDB 使用 [Postgres parser](https://github.com/duckdb/duckdb/tree/v1.5.0/third_party/libpg_query?ref=greybeam-blog.ghost.io) 的一个 fork，这也是 DuckDB 的方言感觉如此熟悉的部分原因。

AST 是你的查询的树形表示，其中每个节点都是一个语法结构：SELECT 语句、列引用、函数调用、join、literal。解析会把扁平字符串 `SELECT sum(l_quantity) FROM lineitem WHERE l_shipdate > '2024-01-01'` 转换成引擎真正可以推理的结构化对象。

```
Select(
    expressions=[
        Sum(
            this=Column(
                this=Identifier(this=l_quantity, quoted=False)))],
    from_=From(
        this=Table(
            this=Identifier(this=lineitem, quoted=False))),
    where=Where(
        this=GT(
            this=Column(
                this=Identifier(this=l_shipdate, quoted=False)),
        expression=Literal(this='2024-01-01', is_string=True))))
```

Copy

来自 [SQLGlot](https://github.com/tobymao/sqlglot?ref=greybeam-blog.ghost.io) library 的 AST。

树结构使引擎的其余部分能够完成自己的工作。binder 遍历这些节点，把 `l_quantity` 解析为某个特定表中的特定列。optimizer 对子树进行模式匹配，以识别 `WHERE` predicate 可以被下推到 scan 中。physical planner 将函数调用节点映射为可执行的 operators。这些 pass 都无法直接作用于原始 SQL。它们需要遍历、模式匹配并重写一个有类型的结构。

### [绑定#](#binding)

下一步是 binding，它根据 catalog 解析 AST 中的每个名称。`lineitem` 变成一个具有已知 schema 的特定表。`l_quantity` 变成一个具有已知类型的特定列。`sum` 变成一个特定的 aggregate function，其输入类型与该列匹配。类型检查也发生在这里：将 `l_shipdate` 与字符串 `'2024-01-01'` 比较之所以可行，是因为 binder 会把 literal 强制转换为 date。

输出是一个 bound tree，其中每个节点都知道自己引用的是什么，以及自己产生什么类型。未解析列、歧义引用和类型不匹配等错误会在这个阶段浮现。

至此，DuckDB 已经把原始 SQL 文本转换成一棵有类型的树。引擎不再把 `l_quantity` 看作查询中的一个字符串；它看到的是来自某个特定表、具有某个特定类型的特定列。

### [优化器#](#the-optimizer)

在 DuckDB 中，optimizer 由一系列小而专注的 transformations 组成，而且事实上你可以逐个检查和禁用它们。

```
D SELECT * FROM duckdb_optimizers();
┌────────────────────────────┐
│            name            │
│          varchar           │
├────────────────────────────┤
│ expression_rewriter        │
│ filter_pullup              │
│ filter_pushdown            │
│ empty_result_pullup        │
│ cte_filter_pusher          │
│ regex_range                │
│ in_clause                  │
│ join_order                 │
│ deliminator                │
│ unnest_rewriter            │
│ unused_columns             │
│ statistics_propagation     │
│ common_subexpressions      │
│ common_aggregate           │
│ column_lifetime            │
│ limit_pushdown             │
│ row_group_pruner           │
│ top_n                      │
│ top_n_window_elimination   │
│ build_side_probe_side      │
│ compressed_materialization │
│ duplicate_groups           │
│ reorder_filter             │
│ sampling_pushdown          │
│ join_filter_pushdown       │
│ extension                  │
│ materialized_cte           │
│ sum_rewriter               │
│ late_materialization       │
│ cte_inlining               │
│ common_subplan             │
│ join_elimination           │
│ window_self_join           │
└────────────────────────────┘
           33 rows
```

Copy

运行 `SET disabled_optimizers = 'filter_pullup, join_order'` 会关闭特定 pass，这样你就能看到它们原本在做什么。

下面是几个有趣的 optimizers：

#### [Filter pushdown#](#filter-pushdown)

这是一个经典的数据库优化：把 `WHERE` predicates 移到尽可能靠近 scan 的地方，这样你就能尽可能早地剪枝数据。DuckDB 首先把 filters 拉到 plan 顶部，以便它们可以被合并和重组，然后再把它们尽可能向下推。

![展示 filter pushdown 实际效果的图。](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-04.png)

自底向上阅读。Filter pushdown 会在可能时把 filter 移到树中更早的位置。

#### [Subquery unnesting#](#subquery-unnesting)

Correlated subqueries 传统上会迫使数据库对外层的每一行都运行一次内层查询，这很慢。DuckDB 实现了 [*Unnesting Arbitrary Queries*](https://portal.fis.tum.de/en/publications/unnesting-arbitrary-queries/?ref=greybeam-blog.ghost.io) 论文中的技术，把这些查询重写成 joins，速度会快得多。

#### [Dynamic join-filter pushdown#](#dynamic-join-filter-pushdown)

在 hash join 期间（更多关于 [hash joins 的内容在这里](https://www.greybeam.ai/blog/why-your-joins-are-slow?ref=greybeam-blog.ghost.io#how-snowflake-executes-joins)），build side 必须在 probe side 开始之前被完全读取。DuckDB 利用了这种顺序：一旦 build side 进入内存，它就计算其中实际包含的 join key values 的最小值和最大值，然后把这些边界作为 runtime filter 下推回 probe-side scan。如果 build side 最终只包含 100 到 200 之间的值，那么 probe scan 可以使用表的 zonemaps，在读取之前跳过该范围之外的任何 row groups。

当 build side 的 distinct join key values 少于 50 个时，filter 会变成一个 `IN` list，而不是 min-max range；这更精确，也会跳过更多行。

#### [Join order optimization#](#join-order-optimization)

Join order 是 optimizer 做出的最重要决策。joins 运行的顺序决定了每个 intermediate result 有多大。一个连接六张表的查询有 30,240 种可能的树形结构，最好与最坏之间的运行时差距可能达到数量级。选得好需要估计每个候选 join 会产生多少行，这取决于表大小、predicate selectivity，以及此前 join 的顺序。

DuckDB 将查询建模为一张 graph。每张表是一个 node，每个 join predicate 是一条连接其引用表的 edge。optimizer 的工作是选择一个顺序，把这些 nodes 组合成一棵单一的树，其中每次组合都是一个 join。例如，如果我们有一个查询把 `a` 连接到 `b`，`b` 连接到 `c`，并把 `c` 连接到 `d`，graph 可能看起来像这样：

```
a ── b ── c ── d
```

Copy

为了找到最好的树，DuckDB 使用 dynamic programming，例如 [DPhyp](https://db.in.tum.de/~radke/papers/hugejoins.pdf?ref=greybeam-blog.ghost.io) 或 [DPccp](https://db.in.tum.de/~radke/papers/hugejoins.pdf?ref=greybeam-blog.ghost.io)。Dynamic programming 是一个简单想法的花哨名称：如果你已经弄清楚了连接 `{a, b, c}` 的最佳方式，那么在弄清楚连接 `{a, b, c, d}` 的最佳方式时，你可以复用这个答案。你不需要重新探索 `{a, b, c}` 内部的所有排序。它会对每个连通的 pair、然后 triplet、然后 quadruplet 等等都这样做。

还有几十种优化值得探索，而整个 optimization phase 通常会在大约一毫秒内完成。优化之后，DuckDB 得到一个 logical plan。下一步是把这个 plan 翻译成引擎实际可以执行的东西。

---

如果到目前为止你喜欢这篇文章，可以考虑订阅。我们会继续分享更多关于 DuckDB 和许多其他 query engines 内部机制的内容。

### 及时了解 Greybeam 的 newsletter。

关于 query engines、optimization、data engineering 和 post-modern data stack 的最佳内容。由 Greybeam 团队用 ❤️ 精心策划。

Subscribe

No spam. Unsubscribe anytime.

## [物理计划#](#the-physical-plan)

想象 optimizer 把这个 plan 交给引擎，用普通英语写出来是：

> 从磁盘读取 `events`。丢弃 `event_date` 在 2026-01-01 或之前的行。按 `customer_id` 对剩余部分分组，并把 `amount` 加总。按总额降序排序结果。返回前 10 个。

引擎现在必须决定如何实际运行这些步骤，才能良好地使用 CPU 并在多个 cores 间并行化。

### [将逻辑步骤映射到物理算子#](#mapping-logical-steps-to-physical-operators)

optimizer 的输出仍然是 logical plan。它说明每一步需要计算什么，但没有说明应该用哪种算法来做计算。大多数 logical steps 都有几种 physical implementations。

以 join 为例。同一个 logical join 可以被转换成以下任一种：hash join、index join、piecewise merge join、cartesian join。

DuckDB 遍历 logical plan，并根据输入和 predicates 的形状为每个 node 选择一个 physical operator。输出是一个 physical plan——一棵由 executor 知道如何运行的 physical operators 构成的树。

我们会把 vectorized execution 的细节留到第 2 部分，但现在有一个 execution concept 很有用：physical plan 并不是作为一次巨大的树遍历来运行的。DuckDB 会把它拆成 pipelines。

### [Pipelines#](#pipelines)

把 pipeline 想象成一条 assembly line。数据从一端进入，并穿过一串 stations。每个 station 做一件事（丢弃一行、转换一列、在 hash table 中查找一个值），并把结果交给下一个 station。只要每个 station 只用当前行就能决定该做什么，这条线就会持续运转。pipelines 的例子：

- WHERE：它要么让该行通过，要么丢弃它。不需要状态。
- A Projection：它计算新的列值并发出它们。
- Hash join 的 probe side：一旦 hash table 建好，它就用该行的 key 在 hash table 中查找，并发出 joined row；如果没有匹配则什么也不发出。

在 DuckDB 中，像这样由 streaming stations 组成的连通链条称为一个 **pipeline**。Pipelines 很容易并行化，因为每个 CPU core 都可以在自己的输入切片上运行自己的一份 assembly line。

### [Pipeline breakers#](#pipeline-breakers)

有些 operators 不能以这种方式工作。它们需要看到整个输入，才能产生输出。

- `ORDER BY` 在看到每一行之前不能发出单个已排序的行，因为它不知道哪一行应该排第一。
- `GROUP BY` 在统计完某个 grouping 中的每一行之前，不能发出最终 sum。
- Hash join 的 build side 必须先构建 hash table，才能开始查找任何东西。

这些 operators 被称为 **pipeline breakers** 或 **sinks**。它们标记一个 pipeline 的结束和下一个 pipeline 的开始。physical plan 实际上是一系列由 sinks 缝合在一起的 pipelines。

![](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-05.png)

回到我们最初的查询，physical plan 可能看起来像这样：

- Pipeline 1：在 `GROUP BY` sink 处结束：  
  scan `events` → filter `event_date > '2026-01-01'` → 写入 `GROUP BY` 的 hash table
- Pipeline 2：在 `ORDER BY` sink 处结束：  
  从 hash table 中读取 groups → 把它们写入 sorted run
- Pipeline 3：最终的 assembly line：  
  读取 sorted runs → 取前 10 行 → 返回结果

每个 pipeline 内部并行运行。多个 threads 同时运行整条 assembly line，每个 thread 处理自己的输入 morsel。相互依赖的 pipelines 按顺序运行，因为 pipeline 2 不能在 pipeline 1 的 `GROUP BY` 完成写入之前开始读取。

### [Sink 中发生什么#](#what-happens-in-a-sink)

一个 sink 分三阶段运行：sink、combine 和 finalize。

#### [Sink#](#sink)

每个 thread 接收 chunks（DuckDB 的 2048 行 batches），并把它们写入自己的 local state，例如，为 `HASH_GROUP_BY` 写入自己的 hash table，为 `ORDER_BY` 写入自己的 sorted run，为 `UNGROUPED_AGGREGATE` 写入自己的 partial aggregate，为 `HASH_JOIN` 的 build side 写入自己的 hash table。Threads 不共享状态。如果每个 thread 都写入同一个共享 hash table，它们就会在每次 insert 时争夺锁。Local state 让每个 thread 能在没有协调的情况下全速 sink。

#### [Combine#](#combine)

一旦每个 thread 都完成写入自己的 local space，结果就必须合并成一个 global state。对于 `GROUP BY`，这意味着合并所有 thread-local hash tables 中每个 group 的 partial sums 和 counts。DuckDB 对 sink 的设计让 combine step 本身也能跨所有 cores 运行，而不是在最后作为单线程 merge（第 3 部分会介绍）。

#### [Finalize#](#finalize)

合并后的 global state 被读出，作为下一个 pipeline 的输入。对于我们的 `GROUP BY`，那将是一条由 `customer_id, total)` rows 组成的 stream。

### [并行性是局部的#](#parallelism-is-local)

一个 pipeline 通过给每个 thread 分配自己的输入 morsel，在所有 cores 上运行。一个 sink 通过给每个 thread 分配自己的 local state 并并行合并，在所有 cores 上运行。DuckDB 不会试图为整个查询规划 global parallelism；它一次并行化一个 pipeline。这也是 morsel-driven parallelism（第 3 部分介绍）和 vectorized execution（第 2 部分介绍）能够工作的部分原因。

## [存储层#](#the-storage-layer)

DuckDB 令人惊叹的一点是，它可以把大多数文件变成 SQL 数据库；事实上，它经常被用来直接查询 Parquet、CSV、JSON、XLSX 等文件格式。

![不同 DuckDB 数据源的图示。](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-06.png)

DuckDB 可以连接并查询许多数据源。([来源](https://motherduck.com/blog/python-duckdb-vs-dataframe-libraries/?ref=greybeam-blog.ghost.io))

### [DuckDB database#](#duckdb-database)

DuckDB database 是一个单文件，通常是 `.duckdb` 或 `.db`。这受到了 SQLite 的启发。一个文件便于移动、备份和共享。

在文件内部，数据被拆分成固定大小的 blocks。默认 block size 是 256 KB，不过可以配置更小的 block size（低至 16 KB）。headers 包含 magic bytes、storage format versions、database headers 等 metadata。

每个 block 还带有一个 *checksum*，这是根据 block 内容计算出来的一个小值。当 DuckDB 读取一个 block 时，它会重新计算 checksum 并进行比较。如果值不匹配，说明数据以某种方式损坏了，DuckDB 会报错。Checksums 很重要，因为 bits 偶尔会在内存或磁盘上翻转：[一束宇宙射线击中一个 cell](https://www.thegamer.com/how-ionizing-particle-outer-space-helped-super-mario-64-speedrunner-save-time/?ref=greybeam-blog.ghost.io)、firmware bug 丢掉一个 byte、质量不稳定的 cable 破坏一次 write，等等。Cloud data warehouses 可以通过内存中的内置错误校正和跨磁盘冗余来缓解这一点。笔记本电脑或 edge devices 等 consumer hardware 通常保护较少，因此 checksumming 是一个有用的后备措施。

### [Columns, row groups, zone maps#](#columns-row-groups-zone-maps)

在 blocks 内部，columns 彼此分开存储。row store 会把整条 records 连续地保存在磁盘上。例如：

```
[id_1, name_1, age_1]
[id_2, name_2, age_2]
[id_3, name_3, age_3]
```

Copy

这对于 `SELECT * FROM users WHERE id = 42` 这样的查询很快，因为 record 的 bytes 在物理上紧挨在内存中。

Column stores 则把 columns 连续地保存在磁盘上。

```
[id_1, id_2, id_3]
[name_1, name_2, name_3]
[age_1, age_2, age_3]
```

Copy

一个从 300 列表中读取 4 列的查询只需要读取这 4 列。在 row store 上，所有 300 列都会被读取，其中 296 列随后需要被丢弃。这就是为什么组织会把 column stores（Snowflake、BigQuery、ClickHouse 等）用于 analytics：查询往往会选择、分组并聚合少数几列。

![](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-07.svg)

#### [Row Groups#](#row-groups)

每个 column 被拆分成最多 122,880 行的 row groups；在一个 row group 内部，又拆分成通常映射到单个 256 KB block 的 column segments。row group 是并行性的一个单位。一个在 8 个 threads 上运行的查询，作用范围内应该至少有 8 个 row groups，才能让每个 thread 都保持忙碌。

![](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-08.png)

([来源](https://www.pracdata.io/p/duckdb-beyond-the-hype?ref=greybeam-blog.ghost.io))

#### [Zone Maps#](#zone-maps)

每个 row group 还带有一个 *zone map*。zone map 包含一个 row group 中的最小值和最大值，以及 null count。当 scan 带着类似 `WHERE event_date > '2026-01-01'` 的 predicate 运行时，DuckDB 会在读取任何数据之前检查每个 row group 的最大值。那些最大 `event_date` 在 `'2026-01-01'` 或之前的 row groups 会被完全跳过。

> 这是主要云数据仓库也在使用的类似技术，只是名字不同。Snowflake 称之为 micro-partition pruning，BigQuery 称之为 block pruning，ClickHouse 使用 `minmax` data skipping indexes。

Zone map 的效果很大程度上取决于 column ordering。一个按 insert timestamp 排序或天然有序的 column，会让每个 row group 的 min-max span 很窄。一个值在整个表中随机散布的 column，会产生覆盖很宽范围的 spans，zonemap 的效果就会差很多。

![](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-09.png)

Zone maps（[来源](https://duckdb.org/2025/05/14/sorting-for-fast-selective-queries?ref=greybeam-blog.ghost.io)）

### [Parquet#](#parquet)

大多数时候，practitioners 查询的不是 DuckDB tables。他们是让 DuckDB 指向 Parquet files。常见模式有两种：

```
-- Query a parquet file directly
SELECT
  customer_id
  , SUM(amount) as total_amount
FROM read_parquet('/my/nice/files/*.parquet', union_by_name=TRUE)
WHERE
  event_date > '2026-01-01'
GROUP BY ALL;

-- Or load it into a DuckDB table first
CREATE TABLE events AS
SELECT * FROM read_parquet('s3://bucket/events/*.parquet');
```

Copy

为什么查询 Parquet files 会这么快？DuckDB 并没有把数据转换成自己的格式。没有 DuckDB 构建的 zone map，没有 DuckDB-side compression，也没有 `.duckdb` file。然而，有些查询的运行速度大致和原生 DuckDB table 一样快。

Parquet 与 DuckDB 的原生格式有相似的设计原则：

- **Parquet 是列式的。** 每个 column 都存在于一个 row group 内自己的 column chunk 中。DuckDB 只读取查询引用的 column chunks。
- **Parquet 按 row group、按 column 存储 min/max statistics**。DuckDB 使用这些相同的 statistics，方式与使用 zone maps 完全一样。

查询 Parquet 时，DuckDB 会读取 footer 以发现文件的 schema 和 row group statistics。它使用这些 statistics 来判断哪些 row groups 可以满足 query predicates。对于每个保留下来的 row group，它只读取查询需要的 column chunks，对其解压，并把它们送入我们前面描述的 pipeline-and-sink。

当文件远程存储时，DuckDB 不会下载整个文件。它会发出一个 HTTP request，只获取 footer，决定需要哪些 row groups 和 column chunks，然后再发出 requests，只获取那些 bytes。一个能够正确剪枝的 `WHERE` clause 可以显著提升网络传输场景下的性能。

### [CSVs#](#csvs)

CSV 与 Parquet 相反，它不是 self-describing。Parquet 实际上会把 schema、statistics、带压缩数据的 chunked columns 交给 DuckDB。CSV 不会。它们只是文本。DuckDB 需要弄清楚用什么字符分隔 columns、values 是否被 quoted、quotes 如何 escaped、第一行是否包含 column names，以及每个 column 应该是什么 type。DuckDB 用它的 [CSV sniffer](https://duckdb.org/2023/10/27/csv-sniffer?ref=greybeam-blog.ghost.io) 来完成这些。

```
SELECT *
FROM 'events.csv';

-- Alternatively
SELECT *
FROM read_csv('events.csv');
```

Copy

当 DuckDB 读取 CSV 时，它会自动尝试检测三件主要事情：dialect、column types，以及文件是否有 header row。

![DuckDB sniffer 各阶段的图示](../../assets/duckdb-internals-why-is-duckdb-fast-part-1/image-10.png)

DuckDB CSV sniffer phases。([来源](https://duckdb.org/2023/10/27/csv-sniffer?ref=greybeam-blog.ghost.io))

#### [Dialect Detection#](#dialect-detection)

dialect 是文件的 parsing grammar：delimiter、quote character、escape character 和 newline style。DuckDB 会测试候选 dialects，并选择能产生最一致 rows 和最多 columns 的那个。例如，像这样的文件：

```
Company|Category|City|IsSuperCool
DuckDB|OLAP database|Amsterdam, Netherlands|True
Snowflake|data warehouse|Bozeman, MT|True
BigQuery|data warehouse|Mountain View, CA|N/A
Greybeam|multi-engine router|San Francisco, CA|True
```

Copy

应该按 `|` 分割，而不是按 `,`，尽管 city names 中包含逗号。sniffer 能弄清楚这一点，因为 `|` 会产生一致的四列表。

#### [Column Types#](#column-types)

选择 dialect 后，DuckDB 会尝试把每个 column 中采样到的 values 转换为候选 types，以检测 column types。如果某个 value 不能转换为某个候选 type，那么该 type 会从该 column 的候选集合中移除。samples 处理完后，DuckDB 会选择剩余候选 type 中优先级最高的那个。文档中记录的默认候选类型按优先级顺序为 `NULL`、`BOOLEAN`、`TIME`、`DATE`、`TIMESTAMP`、`TIMESTAMPTZ`、`BIGINT`、`DOUBLE` 和 `VARCHAR`。由于每个值都可以表示为 `VARCHAR`，它就是 fallback type。

#### [Headers#](#headers)

Header detection 随后进行。如果第一行看起来与下面的 rows 不同，例如 `Company` 和 `Category` 这样的字符串会被视为 column names。否则，它会生成 `column0`、`column1` 等默认名称。

sniffer 基于 sample 工作，而不是扫描整个文件。默认 sample size 是 20,480 行。你可以增大它，或者设置 `sample_size = -1` 来检查整个文件。

## [执行#](#execution)

显然，在查询真正运行之前已经发生了大量工作。它被解析成 AST，绑定到 schema，经过约 30 个 passes 优化，并编译成 physical plan。甚至 storage layer 也提前做了很多工作。

第 2 部分会从 execution 接着讲。敬请期待！

---

在 [Greybeam](https://www.greybeam.ai/?ref=greybeam-blog.ghost.io)，这也是构建 multi-engine router 对数据未来如此令人兴奋的部分原因。很明显，DuckDB 很快。DuckDB 的优势是真实的。Snowflake 的优势也是。BigQuery 的优势也是。

我们相信，未来 data teams 可以使用为每个查询运行得最快而构建的 query engine。一起加入这段旅程吧。

### 及时了解 Greybeam 的 newsletter。

关于 query engines、optimization、data engineering 和 post-modern data stack 的最佳内容。由 Greybeam 团队用 ❤️ 精心策划。

Subscribe

No spam. Unsubscribe anytime.
