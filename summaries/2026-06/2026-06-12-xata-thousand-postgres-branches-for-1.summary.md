---
source: https://xata.io/blog/a-thousand-postgres-branches-for-1
title: A thousand Postgres branches for $1
saved: 2026-06-12 14:25:21 CST
published: 2026-06-11T16:30:00.000Z
---

# A thousand Postgres branches for $1 — 总结

## 一句话结论

Xata 通过存算分离、ZFS copy-on-write、NVMe-oF 远程块设备和 Postgres warm pools，把 Postgres 分支创建/唤醒从 20+ 秒降到 1–2 秒，并用 scale-to-zero 把大量短生命周期分支的成本压到“1000 个约 1 美元”的量级。

## 文章主旨

Xata 最新版本大幅优化了数据库 provision、branching 和从 scale-to-zero 唤醒的速度。它的目标不是单纯让创建数据库更快，而是让 Postgres branch 成为一种可以随手创建、随手销毁、几乎不需要思考成本的开发原语。

典型场景包括：

- 每个 PR / preview environment 一个独立 Postgres。
- coding agent 每次迭代创建一个数据库分支。
- 交互式 SQL session 为每个用户创建一个数据仓库分支。

这些分支包含父库的 schema 和 data，但因为底层使用 copy-on-write，子分支只按相对父分支的差异计费。

## 成本模型

Xata 最小实例 `xata.micro` 是 1GB RAM，价格为每小时 0.012 美元，全天运行约 9 美元/月。

但 Xata 按分钟计费，并且 inactive branch scale-to-zero 后不收 compute 费用。如果一个 branch 只为了测试唤醒 5 分钟，成本约 0.001 美元；10 个约 1 美分；1000 个约 1 美元。

storage 成本也因为 CoW 被压低。比如父分支是 1TB，子分支看起来也拥有完整 1TB 数据，但实际只对相对父分支的 diff 计费，通常很小。

## 真实案例

文章提到两个客户：

- Enginy：AI sales 工具，每次 CI build 都创建隔离 Postgres branch，几分钟后自动 scale-to-zero，需要时再唤醒。这样 preview environment 可以使用接近生产形态的数据，同时不污染共享数据库。
- Runner：AI 电商 builder，大量 autonomous coding agents 分任务工作并开 PR，每周创建数千个 PostgreSQL branches。每个 agent 都有隔离但真实的数据环境。

这类工作负载如果不开 scale-to-zero，月成本会超过 600 美元；使用 Xata 后不到 30 美元，至少便宜 20 倍。

## 技术实现

Xata 同时改了 storage layer 和 compute layer。

### Storage layer：Xatastor

Xata 的 Xatastor 是一个通过 NVMe-oF 暴露 ZFS volumes 的存储引擎。关键点：

- PostgreSQL 本身不需要改源码，仍然是 vanilla Postgres。
- ZFS 提供 copy-on-write clone，用于快速创建 branch volume。
- NVMe-oF 让 compute 和 storage 分离，volume 可以远程挂载到不同 compute node。
- inactive volume 如果没有活跃网络连接，除了磁盘空间外不消耗计算资源。

这点很关键。如果没有存算分离，父分支和所有子分支必须在同一节点上才能利用 CoW，可扩展性会被单机限制住。Xatastor 让上百/上千个分支可以分散挂载到不同 compute 节点。

### Compute layer：warm pools

原始测量显示：PostgreSQL 本身启动只要约 350ms，真正慢的是 Kubernetes pod provisioning 和 control-plane 开销，总耗时约 25 秒。

他们先优化了：

- startup/readiness probes
- init containers
- gateway polling interval
- volume attachment 相关等待
- 自研 CSI driver 的轮询和 attach 路径

这些把时间降到 3–5 秒。但要到 sub-second/1–2 秒级，必须做架构变化，于是引入 warm pools。

Xata 有两类 warm pool：

- Create pools：用于创建新的空 Postgres cluster，里面是已经初始化并运行好的 PostgreSQL。
- Wake-up pools：用于 scale-to-zero 唤醒和 branch，pod 已经 provision 好，但 Postgres 进程还没启动，等待挂载正确 volume。

## 三个关键流程

### Create flow

创建空数据库最简单：从 create pool 拿一个 ready-to-go cluster，在 control plane 里分配给项目即可。

### Wake-up flow

唤醒 scale-to-zero 的数据库时，从 wake-up pool 取一个已 provision 的 pod，然后把对应 volume 通过 NVMe-oF 热连接到这个 compute node。Xata 的 CSI driver 会绕过 Kubernetes API，直接在 OS 层建立 NVMe-oF 连接。volume 挂载后再放行 Postgres 启动。

Postgres 视角里，这只是一次普通 restart：它不知道自己可能已经睡了几天，也不知道自己换到了另一个 pod / node，只要 volume 内容和关停时一致，它就能正常启动。

### Branching flow

branching 最复杂，但复用了前两个 primitive：

1. control plane 触发 Xatastor 对父 volume 做 CoW clone。
2. 得到 child volume。
3. 从 wake-up pool 取一个 warm cluster。
4. 通过 NVMe-oF 把 child volume 挂载过去。
5. 启动 PostgreSQL。

所以 branch 创建快的本质是：volume clone 快 + compute 预热好 + attach 路径短。

## 对 AI coding 的意义

这篇文章真正有价值的点是：Postgres branch 不再只是 DBA 或云数据库 UI 里的“高级功能”，而会变成 agentic development 的基础设施。

在 AI coding 场景里，agent 很适合频繁试错，但数据库状态一直是难点：共享库会互相污染，本地 mock 又不真实，完整 clone 成本高且慢。Xata 这种方案让每个 agent / PR / session 都可以拿到一个隔离、真实、低成本、几秒可用的生产形态数据分支。

## 我的理解

这篇文章的核心不是“Xata 便宜”，而是它展示了未来开发数据库基础设施的一个方向：数据库环境要像 Git branch 一样轻量。技术路径上，它没有魔改 Postgres，而是把复杂度放在存储和编排层：ZFS CoW 负责数据分支，NVMe-oF 负责远程块设备挂载，warm pools 负责绕过 Kubernetes 冷启动成本，scale-to-zero 负责成本闭环。

对数据库内核/云数据库视角来说，这个设计很有意思：它保留 vanilla Postgres，把 branch 的核心能力下沉到 volume 层；这比在 Postgres 内部做多租户/分支语义更工程化，也更容易兼容生态。代价是平台需要强控制 storage、CSI、control plane、compute lifecycle 和 billing model。对于 AI agent 工作流，这种“真实数据 + 快速隔离 + 低成本”的数据库分支能力会越来越重要。
