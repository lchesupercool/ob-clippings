---
title: "Oracle 层次查询（Hierarchical Queries）摘要"
source: "https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html#GUID-E3D35EF7-33C3-4D88-81B3-00030C47AE56"
saved: "2026-06-14 15:41:54 CST"
summary_of: "../2026-06/2026-06-14-oracle-19c-hierarchical-queries.md"
tags: [oracle, sql, hierarchical-query, connect-by, database]
---

## 一句话结论

Oracle 的层次查询是通过 `START WITH ... CONNECT BY PRIOR ...` 在普通表上按父子关系生成树形结果的语法；它在递归 CTE 普及前是 Oracle 查询组织结构、目录树、员工上下级等层次数据的核心机制。

## 文章主旨

这篇 Oracle 19c SQL 参考文档解释了 hierarchical query clause 的语法和执行顺序：

- `START WITH` 定义层次结构的根行。
- `CONNECT BY` 定义父行与子行之间的关系。
- `PRIOR` 用于在 `CONNECT BY` 条件中引用当前行的父行。
- `NOCYCLE` 允许即使数据中存在循环也返回结果，并可配合 `CONNECT_BY_ISCYCLE` 标记循环行。
- `LEVEL`、`CONNECT_BY_ISLEAF`、`CONNECT_BY_ROOT`、`SYS_CONNECT_BY_PATH` 等伪列/函数可用来表达层级、叶子节点、根节点和路径。

## 关键机制

Oracle 处理层次查询时，顺序很重要：先计算 join，再计算 `CONNECT BY` 条件，最后计算剩余 `WHERE` 谓词。随后它先找根节点，再逐层找子节点。`WHERE` 过滤是逐行发生的：如果某一行不满足 `WHERE`，Oracle 删除该行，但不会自动删除它的全部子孙。

`PRIOR` 是核心运算符。`CONNECT BY PRIOR employee_id = manager_id` 表示：父行的 `employee_id` 等于子行的 `manager_id`。因此它能从员工表中沿着“经理 → 下属”方向展开组织结构。`PRIOR` 可以放在等号任意一侧，不同位置决定遍历方向。

## 使用注意

文档特别提醒：层次查询中不要直接使用普通 `ORDER BY` 或 `GROUP BY` 来排序结果，因为它们会破坏 `CONNECT BY` 生成的层次顺序。如果只想对同一父节点下的兄弟节点排序，应使用 `ORDER SIBLINGS BY`。

如果 `CONNECT BY` 条件造成循环，例如某个员工既是上级又在自己的下级链条中，Oracle 默认会报 `ORA-01436: CONNECT BY loop in user data`。使用 `NOCYCLE` 后可以继续返回结果，并通过 `CONNECT_BY_ISCYCLE` 找出包含循环的行。

## 示例覆盖

文档示例包括：

1. 用 `CONNECT BY PRIOR employee_id = manager_id` 生成员工-经理层次。
2. 用 `LEVEL` 显示每行在树中的层级。
3. 用 `START WITH employee_id = 100` 指定根节点，并用 `ORDER SIBLINGS BY last_name` 保持兄弟节点排序。
4. 演示循环数据导致的 `ORA-01436`，以及 `NOCYCLE` 和 `CONNECT_BY_ISCYCLE` 的解决方式。
5. 用 `CONNECT_BY_ISLEAF` 和 `SYS_CONNECT_BY_PATH` 把列值串成逗号分隔列表。
6. 用 `CONNECT_BY_ROOT` 找出每个节点对应的根节点/最高层经理，并结合 `GROUP BY` 聚合层次路径上的薪资。

## 我的理解

这篇文档的价值在于把 Oracle 老派层次查询的执行模型讲清楚。对 PostgreSQL 背景的人来说，可以把它类比为 recursive CTE，但 Oracle 的 `CONNECT BY` 更像是一套专门为树遍历设计的 DSL：语法短、内置层级伪列丰富，但表达复杂递归分支时不如 recursive `WITH` 灵活。文档最后也明确指出，recursive subquery factoring 更强大，因为它支持深度优先/广度优先搜索和多个递归分支。
