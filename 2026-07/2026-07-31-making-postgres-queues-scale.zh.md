---
title: "Postgres 队列确实可以扩展"
source: "https://www.dbos.dev/blog/making-postgres-queues-scale"
author: ["Qian Li", "Peter Kraft"]
published: 2026-06-02
saved: 2026-07-31
tags: [postgresql, queues, scalability, databases, DBOS]
translation_of: "2026-07-31-making-postgres-queues-scale.md"
---

# Postgres 队列确实可以扩展

关于基于 Postgres 的队列，传统观点认为它们无法扩展。要处理大型工作负载，你不能使用 Postgres，而是需要 RabbitMQ + Celery 或 Redis + BullMQ 之类的专用队列系统。人们这么说是有原因的：队列对 Postgres 来说确实是一种要求很高的工作负载。在大规模场景下，数千个 worker 同时轮询队列表，造成争用并频繁更新索引。但只要采用正确的优化，Postgres 就能应付。在这篇博客文章中，我们将介绍如何大规模优化基于 Postgres 的队列，并在数千台服务器上实现每秒执行 3 万个工作流。

### 经验 1：重新发现 SKIP LOCKED

要让基于 Postgres 的队列真正可用，首先要解决多个 worker 对同一批工作流执行出队时产生的争用。概括来说，基于 Postgres 的队列是这样工作的：客户端把工作流加入队列表，worker 则取出并处理最早入队的工作流（假设使用 FIFO 队列）。最直接的做法是，每个 worker 运行如下查询，找出最早入队的 N 个工作流，然后将它们出队：

![使用 Postgres 实现队列——SQL 示例](../../assets/making-postgres-queues-scale/01.png)

一旦多个 worker 并发执行这个查询，争用就会出现。每个 worker 都会看到相同的一批最早入队工作流，并试图同时将它们出队。但每个工作流只能由一个 worker 取走，因此大多数 worker 都无法获得新工作，只能再次尝试。在大规模场景下，这种争用会成为系统瓶颈，限制任务的出队速度。

![使用 Postgres 实现队列——架构图](../../assets/making-postgres-queues-scale/02.png)

幸运的是，Postgres 提供了解决这个问题所需的基础能力：锁定子句。下面是一个使用 FOR UPDATE SKIP LOCKED 的查询示例：

![使用 Postgres 实现队列——出队 SQL](../../assets/making-postgres-queues-scale/03.png)

以这种方式选择行会产生两个效果。第一，它会锁定这些行，使其他 worker 无法再次选中它们。第二，它会跳过已经锁定的行，因此选中的不是最早入队的 N 个工作流，而是最早入队且**尚未被其他 worker 锁定的 N 个工作流**。这样一来，许多 worker 就能**在没有争用的情况下**并发获取新工作。第一个 worker 选中并锁定最早的 N 个工作流，第二个 worker 选中并锁定接下来的 N 个，依此类推。

![](../../assets/making-postgres-queues-scale/04.png)

锁定子句使基于 Postgres 的队列成为可能（这也是 SKIP LOCKED 这种老牌 Postgres 技巧总被人重新发现的原因）。没有它，worker 之间的争用会使系统无法扩展到每秒约 100 个工作流以上。有了它，Postgres 可以扩展得更远，但要达到更高规模，还需要更多优化。

### 经验 2：留意事务隔离级别

尽管锁定子句显著提升了性能，我们很快又遇到另一个与争用有关的瓶颈：在大规模场景下，出队操作经常因 Postgres 的“序列化失败”（Serialization Failure）异常而失败，必须重试。当出队速度超过每秒约 1000 个工作流时，大多数出队操作都会遇到序列化失败，从而形成性能瓶颈。

罪魁祸首是 Postgres 的事务隔离级别。出队事务最初运行在 REPEATABLE READ 下，以支持诸如“所有 worker 合计最多并发运行 N 个工作流”这样的全局队列限制。实施这些全局限制，需要所有 worker 共享一个全局一致的队列状态视图；Postgres 中的 REPEATABLE READ 保证事务始终基于事务开始时数据库的固定“快照”运行，并且不会“看到”事务执行期间其他并发事务完成后产生的影响。

问题在于，REPEATABLE READ 在高并发下会变得昂贵。如果多个 worker 并发修改相互重叠的行，Postgres 会中止其中一个事务并报告序列化失败。在大规模场景下，worker 花在重试事务上的时间甚至超过了处理工作流的时间。

关键认识是，规模最大的队列很少使用全局流量控制。在大规模场景下，用户通常更偏好本地限制，例如“每个 worker 最多运行 10 个工作流”；这不需要跨 worker 协调。

因此，我们让隔离级别变成有条件的：

![Postgres 事务隔离级别会影响扩展能力](../../assets/making-postgres-queues-scale/05.png)

启用全局流量控制的队列继续使用 REPEATABLE READ；未启用全局流量控制的队列则使用 READ COMMITTED，从而彻底消除序列化失败，并显著提升吞吐量。

### 经验 3：索引并非免费

结合锁定子句和较低的隔离级别后，即使存在数千个 worker，争用也几乎完全消失了。然而，当执行速度超过每秒约 8000 个工作流时，我们遇到了新的瓶颈：CPU 使用率过高。CPU 开销来自两个看似无关的地方：出队查询本身，以及 Postgres autovacuum。我们发现，两者的根因相同：索引效率低下。

为了加速查询，工作流状态表上建有多个二级索引。其中一个索引专门服务于出队查询，对 queue_name 和 status 建索引，使 Postgres 能够快速找出指定队列中所有状态为 ENQUEUED 的工作流：

![使用 Postgres 实现队列——索引](../../assets/making-postgres-queues-scale/06.png)

其他索引主要用于可观测性。例如，针对父工作流 ID 的索引，可以高效查询工作流的层级关系：

![使用 Postgres 实现队列——索引示例](../../assets/making-postgres-queues-scale/07.png)

在大规模场景下，这些索引效率不高。

出队索引有助于找出所有已入队的工作流，但不会以任何特定顺序返回它们。因此，Postgres 执行出队查询时，必须按时间戳对返回的工作流进行排序，才能找到最早入队的工作流，这会增加查询的 CPU 开销。

与此同时，维护大量索引的代价也很高。工作流状态每次更新（入队、出队、完成）都需要更新每一个索引。此外，索引更新后，Postgres autovacuum 还必须清理过期的索引条目。在高吞吐量下，索引维护和 vacuum 会消耗数据库 CPU 的相当大一部分。

解决办法是让索引更具选择性。

首先，我们更新了主要的出队索引，使它不仅能返回特定队列中所有已入队的工作流，还能按优先级和时间戳对它们排序。此外，我们把它改成了部分索引，仅在工作流状态为 ENQUEUED 时维护该索引。

![使用 Postgres 实现队列——让索引更具选择性](../../assets/making-postgres-queues-scale/08.png)

这会从两方面提升性能。第一，出队查询不再需要执行昂贵的排序步骤。第二，工作流出队时，Postgres 可以直接删除其索引条目，而不必在工作流余下的生命周期中继续维护它，从而降低维护和 autovacuum 成本。

我们对大多数可观测性索引也采用了同样的原则。例如，只有当工作流确实存在父工作流时，才维护父工作流 ID 索引：

![使用 Postgres 实现队列——可观测性查询 SQL 示例](../../assets/making-postgres-queues-scale/09.png)

这些优化结合起来，大幅降低了 CPU 使用率，使队列能够扩展到[每秒超过 3 万个工作流，或每月 800 亿个工作流](https://www.dbos.dev/blog/benchmarking-workflow-execution-scalability-on-postgres)。

### 了解更多

如果你喜欢构建可扩展、可靠的系统，我们很乐意与你交流。在 DBOS，我们的目标是让基于 Postgres 的持久化执行尽可能简单且高性能。欢迎查看：

- 快速开始：[https://docs.dbos.dev/quickstart](https://docs.dbos.dev/quickstart)
- GitHub：[https://github.com/dbos-inc](https://github.com/dbos-inc)
- Discord 社区：[https://discord.gg/eMUHrvbu67](https://discord.gg/eMUHrvbu67)
