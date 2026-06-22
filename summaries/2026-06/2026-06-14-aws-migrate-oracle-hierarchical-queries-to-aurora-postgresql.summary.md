---
source: "https://aws.amazon.com/blogs/database/migrate-oracle-hierarchical-queries-to-amazon-aurora-postgresql/"
saved: "2026-06-14 15:47:45 CST"
summary_of: "2026-06-14-aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql.md"
---

# 将 Oracle 层次查询迁移到 Aurora PostgreSQL — 中文摘要

## 一句话结论

Oracle 层次查询（`CONNECT BY` 系列）可以通过 PostgreSQL 的递归 CTE（`WITH RECURSIVE`）完整替代，覆盖 `LEVEL`、`NOCYCLE`、`SYS_CONNECT_BY_PATH`、`ORDER SIBLINGS BY`、`CONNECT_BY_ISLEAF` 和 `CONNECT_BY_ROOT` 六大关键语义；`tablefunc` 扩展的 `connectby` 函数仅适合最简单的层级展示场景，功能受限。

## 文章主旨

本文是 AWS 官方博客的实操指南，面向正在从 Oracle 迁移到 Aurora PostgreSQL 的数据库团队。核心论点是：虽然 PostgreSQL 没有原生的层次查询语法，但通过递归 CTE + 辅助字段（route 数组、cycle 标志、path 字符串拼接）可以完整复现 Oracle `CONNECT BY` 的全部关键语义。文章通过 6 个递进场景逐一给出了 Oracle 原版 SQL 和 PostgreSQL 等价 SQL 的对照，并指出了 `tablefunc` 扩展的适用边界。

## 关键机制

### 1. 递归 CTE 的通用骨架
```sql
WITH RECURSIVE cte(...) AS (
    -- 锚点成员（anchor）：定位根节点，如 manager_no IS NULL
    SELECT ... FROM hier_test WHERE manager_no IS NULL
    UNION ALL
    -- 递归成员：自引用 cte，逐层 JOIN 子节点
    SELECT ... FROM hier_test e JOIN cte c ON e.manager_no = c.emp_no
)
SELECT * FROM cte;
```

### 2. 各 Oracle 关键字的 PostgreSQL 等价实现

| Oracle 关键字 | PostgreSQL 等价方式 | 核心技术点 |
|---|---|---|
| `LEVEL` | 递归中维护 `level + 1` 计数器列 | 锚点 `level = 1`，递归每次 `+1` |
| `SYS_CONNECT_BY_PATH` | 递归中拼接字符串路径列 `path` | 锚点 `';'||ename`，递归 `c.path\|\|';'\|\|e.ename` |
| `NOCYCLE` | `route` 数组 + `cycle` 布尔标志 | `route = array[emp_no]`，每次追加并检测 `= ANY(c.route)`，`WHERE cycle = false` 剪枝 |
| `ORDER SIBLINGS BY` | 按 path 字符串排序 | path 按排序字段拼接（如 `ename\|\|emp_no`），最终 `ORDER BY path` |
| `CONNECT_BY_ISLEAF` | `NOT EXISTS` 子查询检测有无子节点 | `NOT EXISTS (SELECT * FROM cte p WHERE p.manager_no = e.emp_no AND cycle = false)` |
| `CONNECT_BY_ROOT` | `SPLIT_PART(path, ';', 2)` 或 `substr` 提取根节点 | path 首段即为根节点值 |

### 3. tablefunc 扩展的局限性
- `connectby` 函数只适合取 `parent, child, level` 三列，取额外列会报错
- 不支持 `NOCYCLE`、`CONNECT_BY_ISLEAF`、`ORDER SIBLINGS BY`、`CONNECT_BY_ROOT` 等高级语义
- 结论：仅适用于最简单的层级展示，迁移场景中应以递归 CTE 为主方案

## 迁移映射

| 场景 | Oracle 语法核心 | PostgreSQL 等价核心 |
|---|---|---|
| 基础层级查询 | `CONNECT BY PRIOR emp_no = manager_no START WITH ...` | `WITH RECURSIVE ... JOIN ... ON e.manager_no = c.emp_no` |
| 路径追踪 | `SYS_CONNECT_BY_PATH(ename, ';')` | 字符串拼接 `c.path\|\|';'\|\|e.ename` |
| 循环检测 | `CONNECT BY NOCYCLE PRIOR ...` | `route` 数组 + `cycle` 标志 + `WHERE cycle = false` |
| 兄弟排序 | `ORDER SIBLINGS BY ename` | path 中嵌入排序字段，`ORDER BY path` |
| 叶节点判断 | `CONNECT_BY_ISLEAF` | `NOT EXISTS (子节点检查)` |
| 根节点引用 | `CONNECT_BY_ROOT ename` | `SPLIT_PART(path, ';', 2)` |

## 使用注意

1. **数据复杂度差异**：文章示例数据集非常简单（18 行，单表），实际生产环境中的层次查询可能涉及多表 JOIN、复杂过滤和更大的数据量。需要针对实际数据规模和查询模式进行性能测试。
2. **并非全覆盖**：文章仅讨论了 6 种 Oracle 层次查询关键字，Oracle 还有 `CONNECT_BY_ISCYCLE` 等未涵盖在内，迁移时需逐一排查。
3. **功能与性能双重验证**：即使 SQL 语义等价，执行计划和性能特征可能差异显著。递归 CTE 在大深度或宽树结构下的表现需要专门评估。
4. **AWS SCT 的辅助作用**：AWS Schema Conversion Tool 可自动转换大部分对象，但层次查询会标记为需要手动转换 — 本文提供的模式正是手动转换的参考方案。
5. **路径排序的局限**：`ORDER SIBLINGS BY` 的 PostgreSQL 等价方案（按 path 排序）在排序字段有重复值时可能出现顺序偏差，需要额外处理（如拼接 `emp_no` 保证唯一性）。

## 我的理解

这篇文章是一份非常实用的 Oracle-to-PostgreSQL 迁移速查手册，核心价值在于它提供了一个"对照表"式的工程方案，而非泛泛而谈。6 个场景从简到繁，覆盖了层次查询迁移中几乎一定会遇到的语义需求。

从技术深度来看，关键在于理解递归 CTE 的"锚点 + 递归 + 辅助字段"三板斧。`route` 数组做循环检测、`path` 字符串做路径追踪和排序，这两个技巧是 ORACLE `CONNECT BY` 迁移的核心发明。本质上，Oracle 在引擎内部维护了这些信息，而 PostgreSQL 要求我们在 SQL 层面显式维护。

一个值得注意的点是 `tablefunc` 被明确标记为"不够用"——对于实际迁移项目来说基本可以忽略。这与很多人的第一直觉（"用 `connectby` 不就行了"）相反。

从个人知识库的角度，这篇文章与 PostgreSQL 递归查询、数据库迁移、CTE 高级用法这几个主题强相关，后续如果遇到实际迁移项目可以作为参考方案的起点。特别需要留意的是文中提到的性能验证建议——在实际项目中，我会优先对大深度树（>10 层）和宽树（单节点 >1000 子节点）做 benchmark，这是方案是否可行的关键。
