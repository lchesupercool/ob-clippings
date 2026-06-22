---
source_url: https://thebuild.com/blog/managed-postgres-examined-azure-database-for-postgresql-flexible-server/
ingested: 2026-06-03
author: Christophe Pettus (The Build)
series: Managed Postgres, Examined (#5)
previous: Aurora, Cloud SQL, AlloyDB
---

# Managed Postgres, Examined: Azure Database for PostgreSQL Flexible Server

> Azure 的托管 PG 有一个独特的 HA 架构：同步流复制在进程层而非存储层。standby 在提交路径里，这个设计决定了一切。

## 架构

Flexible Server = 社区 PG 进程跑在 Azure VM + Premium SSD。不是 fork，就是原版 PG。

### HA：同步流复制（和其他人都不一样）

**RDS/Aurora/Cloud SQL/AlloyDB** → 复制在存储层（block-level），数据库感知不到。

**Flexible Server** → 复制在 PG 的提交路径里（process-level）。

- 主库的 commit 不返回给客户端，直到 WAL 在**主库和备库都刷盘**
- 等价于 `synchronous_commit = on`（remote_flush），不是 remote_apply
- `synchronous_commit` 和 `synchronous_standby_names` 是微软管的，你不能改

### 两种 HA 模式

| 模式 | 备库位置 | 延迟 | 保护 |
|------|---------|------|------|
| Same-zone | 同 AZ | 亚毫秒 | 节点/机架 |
| Zone-redundant | 跨 AZ | 1-3ms/commit | AZ 级 |

### 故障转移

- 健康检测 30-40s → 恢复+提升 60-120s 内完成
- 连接断开，新连接等 DNS 传播
- 提升后的新主库**没有备库**，需要等 Azure 重建 → **故障转移后是 non-HA 窗口**

## 最大坑：备库在提交路径里

```
每个 commit → 等待备库刷盘确认 → 才返回客户端
```

备库磁盘慢 / 网络抖 / 主机噪声 → **主库 commit 延迟直接上升**，和主库自身状态无关。

### 三个关键诊断要点

1. **爆炸半径随 commit 数量缩放，不随 commit 大小**。4KB 元数据提交和一个 50KB TOAST 提交在等备库这件事上完全一样疼。想降低敏感性 → 减少提交频率（batch），而不是减小行大小。

2. **看 `SyncRep` 等待事件**。出现在 `pg_stat_activity` 就是同步复制卡住了，不是主库本地问题。

3. **取消一个卡住的 COMMIT 不会回滚它**！会收到 `WARNING: canceling wait for synchronous replication due to user request`，事务返回 committed，只是丢了仲裁持久性。应用如果用"超时=失败=重试"逻辑 → 双重写入。

### zone-redundant 的每提交延迟税

每个 commit 多 1-3ms。几千 OLTP tps 的负载能感觉到。这是持久性合约的代价，不是配置错误。

### 你不能自己调持久性权衡

自建的可以 `synchronous_commit = local` 做批量导入再切回来，Flexible Server 不行。所有写操作统一交同步税。

## 其他值得注意的

### 独有优势
- **内置 PgBouncer**：不需要单独的池化层（RDS 需要 RDS Proxy，Cloud SQL/Aurora 没有）
- **版本跟进好**：社区 GA 后几个月就支持，因为跑的是社区 PG 不是 fork
- **直接参数管理**：不通过参数组抽象层，跟改 postgresql.conf 一样
- **Entra ID 集成**：Azure 原生身份认证
- **热缓存故障转移**：备库一直在 apply WAL，缓存是热的（但只反映主库写入的页面，不是读工作集）
- **近零停机维护**：HA 实例先修备库再故障转移，维护窗口压缩到故障转移那几秒

### 常见坑
- **扩展需要两步**：先加到 `azure.extensions` 允许列表，再 `CREATE EXTENSION`。很多人忘了第一步
- **Burstable tier 是生产陷阱**：有 CPU 信用额度，耗尽后性能崩溃；还不支持 HA
- **Azure Monitor 日志有自己的账单**：开查询日志的详细级别会产生单独的 Log Analytics 费用
- **网络模型选错很难改**：VNet 集成 vs Private Link 是创建时决定
- **没有 SUPERUSER**：`azure_pg_admin` 够大部分 DBA 日常用
- **没有 pg_tle 那样的自定义扩展机制**

## 适合/不适合

### ✅ 适合
- Azure 上的通用 OLTP，需要社区 PG 行为完全一致
- 高连接数/短连接负载（内置 PgBouncer）
- Entra ID 为中心的组织
- 需要可审计的同步持久性（"两个独立磁盘上的两个 PG 提交了，才算提交"）

### ❌ 不适合
- 延迟敏感的高 commit 率写入 + 跨 AZ（zone-redundant 的延迟税）
- 需要不在允许列表里的扩展
- 需要 Aurora 的克隆、AlloyDB 的列存引擎等特殊存储架构
- 真正的 scale-to-zero

## 系列下一篇
Azure Cosmos DB for PostgreSQL（Citus 底层，和 Flexible Server 完全不一个定位）
