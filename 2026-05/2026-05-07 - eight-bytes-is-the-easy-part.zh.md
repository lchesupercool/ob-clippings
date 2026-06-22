---
title: "八个字节只是简单的部分"
source: "https://thebuild.com/blog/2026/05/07/eight-bytes-is-the-easy-part/"
author: "thebuild.com"
published: "2026-05-07"
clipped: "2026-05-08T14:38:18+08:00"
tags:
  - clipping
  - PostgreSQL
---

# 八个字节只是简单的部分

Source: [https://thebuild.com/blog/2026/05/07/eight-bytes-is-the-easy-part/](https://thebuild.com/blog/2026/05/07/eight-bytes-is-the-easy-part/)  
Published: 2026-05-07  
Category: PostgreSQL

![](../../assets/eight-bytes-is-the-easy-part/eight-bytes-is-the-easy-part-01.png)

[PostgreSQL 19 将 `MultiXactOffset` 扩展到 64 位，](https://thebuild.com/blog/2026/05/06/multixact-members-at-64-bits-one-less-wraparound-to-worry-about/)从而淘汰了数据库在生产中可能产生的较有意思的一类故障模式。multixact SLRU 上的 members-space 回卷曾是一类真实的宕机问题——去年五月 Metronome 在一次迁移中就撞上了四次——而它在 19 中消失了。很好。继续前进。

下一个显而易见的问题，大约从布什政府时期开始，就差不多每个月都会在 `pgsql-hackers` 的某个地方被问一次：普通事务 ID 什么时候会变成 64 位？

答案是“不会很快”，而本文剩下的部分就是解释原因的长版本。

## 显而易见的成本

每个 heap tuple 的头部都携带 `xmin` 和 `xmax`，二者都是 `TransactionId`，目前也都是 `uint32`。把它们扩展到 64 位会给每个 tuple 增加八个字节。对于宽行来说，这是四舍五入误差。对于窄行——连接表、记账表、队列表——则不是。一个两列的 `(int, int)` 行今天包括对齐填充在内是 32 字节；增加八个字节的头部后，表会膨胀 25%，并且每个指向它的索引也会付出相应成本。

现有集群不能只是选择加入：页面上的格式本身将不得不改变。

这是每个人都看得到的部分。它并不是困难的部分。

## `pg_upgrade` 就这样没了

`pg_upgrade` 之所以快——也是大多数大版本升级只需要几分钟而不是几小时的原因——在于磁盘上的 heap 格式几乎从不改变。`pg_upgrade --link` 只是把现有数据文件硬链接到新集群中，重放系统目录，然后就完成了。实际的用户数据从未被读取、从未被写入、从未被复制。

对 tuple 头部的更改会打破这一点。`pg_upgrade --link` 不再是一个选项，因为新服务器无法读取旧服务器生成的页面。你现在进入了 `pg_upgrade --copy` 的领域，这意味着集群中每个关系的每一个字节都会被读取、转换并写回。在小型数据库上，这是一顿漫长午餐的时间。在 30TB 集群上，这是一次多日停机事件，而且这还是假设 I/O 子系统配合的情况下。

替代方案更糟，或者更古怪。`pg_dump`/`pg_restore` 是 `--copy` 再加上一个逻辑瓶颈。基于逻辑复制的升级——在新版本上搭建一个并行集群，复制进去，然后切换——是大型机构实际会采用的方式，但运维足迹是真实存在的：在此期间需要双倍硬件、双写窗口、仔细处理序列和大对象，以及一个没人喜欢编写的切换计划。

这在性质上不同于仍在记忆中的任何一次 PostgreSQL 大版本升级。`pg_upgrade` 让“保持在受支持版本上”这台跑步机在运维上变得便宜。一次 64-bit-XID 升级会把这台跑步机带回 8.x 时代，那时大版本升级意味着 `pg_dump` 和一个漫长的周末。

## 一键升级也随之没了

我所知道的每一家托管 PostgreSQL 提供商——RDS、Aurora、Cloud SQL、Azure Database for PostgreSQL、Supabase、Neon，以及其他所有——都是在 `pg_upgrade --link` 之上实现其“升级到下一个大版本”按钮的。这就是为什么这个按钮能够存在。这也是为什么一台 10TB RDS 实例的大版本升级所记录的停机时间是以分钟而不是小时计量。

打破磁盘格式，这个按钮也会随之失效。提供商的选项和所有人一样：完整复制、逻辑复制到并行集群，或者一次 `pg_dump`/`pg_restore` 循环。这些都不是一键操作。它们都要求客户规划一个真正的维护窗口，通常还需要在整个过程中运行一个并行实例。对于小型数据库，这可能意味着一小时停机，而今天只需要五分钟。对于大型数据库，这意味着数天，并且迁移会变成一个项目，而不是一个复选框。

这才是将决定 64-bit XID 何时真正发布的部分：不是代码，而是升级故事。社区花了十年都没有让这个特性落地，很大程度上是因为没有人愿意告诉地球上的每一个 PostgreSQL 运维者：他们下一次大版本升级会是一次持续数小时的宕机。

## 快照

快照是 MVCC 在运行时的表达：一个 `xmin`、一个 `xmax`，以及一组在获取快照时仍在进行中的 XID。这个数组存在于 procarray 中，会被复制到每个事务的本地快照中，会被序列化到磁盘上的 prepared transaction 状态中，会被发送给逻辑复制订阅者，并会在主库和备库之间交换，用于 hot-standby 可见性。

将每个条目的宽度翻倍，会使每个快照的大小翻倍。每次读取都会获取快照；procarray 扫描在高连接工作负载上已经是一个众所周知的扩展性瓶颈，而快照获取的缓存足迹正是原因之一。备库消费快照流时的线上传输大小也是如此，携带逻辑解码快照信息的 WAL 记录中 in-progress XID 列表的大小也是如此。

这不是一个致命问题。它是一个千刀万剐式的问题，而这更糟，因为每个基准测试都会小幅退化，并且总得有人为每一个退化辩护。

## SLRU

`pg_xact`（以前叫 `pg_clog`）、`pg_subtrans` 和 `pg_commit_ts` 都通过事务 ID 寻址。磁盘格式假定 XID 空间适合放进一个数量可控的 256KB SLRU segment 集合中——对于整个 32 位范围的 `pg_xact`，按每个事务两位计算，大约是一 GB。64 位地址空间并不是大两倍；它是大四十亿倍。

你不必把它全部物化出来。你只需要能够对它寻址。但每一个 SLRU 函数——segment 命名、页面查找、截断，以及围绕 vacuum freezing 的整个生命周期——都假定 XID 索引到一个封闭的循环空间中。截断尤其有意思：今天，`pg_xact` 会在最老的 live XID 之后被截断，使用的是系统其他部分同样使用的回卷算术。移除上限之后，你就改变了截断本身的含义。

## 循环比较原语

`TransactionIdPrecedes` 是一行 C 代码：它把两个 `int32` 值相减并检查符号。这之所以有效，是因为 32 位 XID 会回卷，而比较隐式地是模 2^32 的。它居然能够成立这一事实，是 PostgreSQL 避免需要对曾经记录过的每一个事务建立全局总序的承重部件。

迁移到 64 位后，比较变成普通的小于比较。严格来说更简单。只不过每个调用点都假定循环语义——freeze horizon 算术、`pg_xact` 截断逻辑、`FrozenTransactionId` 哨兵、`MultiXactIdPrecedes` 家族（它继承了同样的惯用法），以及针对 C API 编写的扩展中很长一串比较。每一个都必须被审计。大多数在任一语义下都是正确的。有些则不是。

## 再次谈磁盘格式问题

64 位 XID 会改变 heap tuple 头部。它还会改变 `HeapTupleHeaderData` C 结构体，而这是公共扩展 ABI 的一部分。每个会内省 tuple 的扩展——数量很多，包括一些你在没有意识到的情况下依赖的扩展：`postgis`、`citus`、`timescaledb`、`pg_repack`，以及每个构建 heap tuple 的 FDW——都必须重新编译。大多数必须被修改。有些必须被大幅重写。

物理复制假定主库和备库之间的页面逐字节相同。一个 64-bit-XID 主库无法复制到一个 32-bit-XID 备库，也不存在桥接方式。你要么一次性升级整个复制拓扑，要么根本不升级。对于运行大型全球备库集群的机构而言，这在运维上会很痛苦，而“重写每个页面”这种说法还低估了它。

逻辑复制稍微更灵活一些——它是一个逻辑协议——但协议本身携带 XID，并且向 32 位订阅者发出 64 位 XID 并不是一个已定义的操作。

## 为什么 `MultiXactOffset` 更容易

`MultiXactOffset` 是指向 multixact SLRU 的 members 数组的指针。它不存储在 tuple 头部中。它不在快照中传输。它并不是公共 C ABI 的一部分，以至于下游代码会伸手进去访问。它完全生活在 `pg_multixact` 内部，由 `multixact.c` 中少数函数读写，并在少数几类记录中写入 WAL。

扩展它需要改变 SLRU 文件格式，并添加一个重写 `pg_multixact` 的 `pg_upgrade` 步骤（从绝对规模上看，这是一个小目录——GB 级，而不是 TB 级）。它不需要触碰 heap 页面、快照、procarray、visibility map、freeze 逻辑，或任何扩展。爆炸半径很小。

XID 扩展的爆炸半径则是“所有东西”。

## 实际正在做的工作

这个领域中的严肃提案并不会扩展页面上的 XID。它们在磁盘上保持 `xmin` 和 `xmax` 为 32 位，并在某个地方添加一个 **epoch**——可能是每页、每关系，或隐含在 freeze horizon 中——从而让有效 XID 是 64 位，而存储仍保持 32 位。这就是多年来 Postgres Pro 等以各种形式提出过的“通过 epoch 实现 64 位 XID”方案。它避免了 heap 重写。它避免了 tuple 内省的 ABI 破坏，因为页面上的格式未变。它仍然需要改变快照表示、SLRU 寻址和循环比较语义——但最昂贵的单项改变，也就是 heap 格式，被绕开了。

代价是复杂性。每一次 tuple 可见性检查都必须把页面上的 32 位 XID 与相关 epoch 组合起来，得到可比较的 64 位值。Freezing 变成了更新 epoch，而不是更新 XID 本身。Anti-wraparound vacuum 并不会消失；它的性质只是改变了。它不再是“在回卷之前 freeze”，而变成“在 epoch 溢出每页空间之前 freeze”。不同的问题，不同的运维特征，但仍然是一个问题。

## 可以期待什么

不要期待 PostgreSQL 20 中会有 64 位 XID。`MultiXactOffset` 补丁花了多年迭代才落地，而那个补丁的范围是可处理的。XID 工作以各种形式已经推进了十多年，它尚未发布的原因并不是没人尝试过。而是每一次尝试都会暴露出系统中一个新的角落，结果发现它以一种并不显而易见的方式依赖 32 位假设。

最终落地的版本几乎肯定会是 epoch 方案，并且它可能会分阶段到来：先是内存表示，然后是 SLRU 层，然后是升级现有集群的方法，最后才是作为最后一张多米诺骨牌的真正 XID 扩展。当有人为它提出一个 `pg_upgrade` 标志时，我们就会知道它快到了。

与此同时：监控 `age(datfrozenxid)`，在热点表上积极 freeze，并为 multixact members 上限已经消失而庆幸。
