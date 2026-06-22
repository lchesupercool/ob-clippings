---
title: NOT IN 里的 NULL
source: https://boringsql.com/posts/not-in-null/
author: Radim Marek / boringSQL
published: 2026-06-15
saved: 2026-06-15
translation_of: 2026-06/2026-06-15-boringsql-not-in-null.md
tags: [postgresql, sql, null, query-planner]
---

# NOT IN 里的 NULL

一个 `NOT IN` 查询可能悄悄返回错误答案。它是合法 SQL，可以正常运行，没有错误，并返回一个格式完全正常的结果集——只是这个结果集为空，而它本不该为空。没有 warning，没有 hint，日志里也什么都没有：你期待几百行，结果是 0 行，而数据库认为这是正确的。

原因几乎总是：某个你忘记检查的位置有一个 `NULL`，再加上你已经敲过无数次的两个关键字：`NOT IN`。这不是 PostgreSQL 的 bug。这正是 SQL 标准要求的行为，Postgres 只是忠实实现了它。也正因为如此，它很容易踩中；也解释了为什么大约二十五年来，planner 都无法安全地绕过这个问题。最终，它可以追溯到 parser 里的一个 `if` 语句。

## 示例 schema

不需要复杂结构。一个产品表，其中有一个产品还没有分配 category；另一个 archived categories 表，里面恰好有一个 `NULL`：

```sql
CREATE TABLE products (id int, category_id int);
INSERT INTO products VALUES (1, 10), (2, 20), (3, NULL), (4, 10);

CREATE TABLE archived (category_id int);
INSERT INTO archived VALUES (20), (NULL);
```

`archived` 里的这个 `NULL` 并不牵强。只要一个列允许为空（大多数列默认如此），`NULL` 就可能进入任何你拿来做 `NOT IN` 的子查询。这正是重点：这不是罕见数据条件，而是普通情况。

## 返回空结果的查询

这个需求你可能写过无数次：给我所有 category 没有被归档的产品。

```sql
SELECT id, category_id FROM products
WHERE category_id NOT IN (SELECT category_id FROM archived);
```

你期待得到产品 1 和 4（category 10，不在 archived 集合里）。实际返回：

```sql
 id | category_id
----+-------------
(0 rows)
```

所有行都没了。不是少了一部分，也不是差一行，而是全部消失。把 `archived` 里的 `NULL` 去掉，同一个查询就正常了：

```sql
SELECT id, category_id FROM products
WHERE category_id NOT IN (SELECT category_id FROM archived
                          WHERE category_id IS NOT NULL);
```

```sql
 id | category_id
----+-------------
  1 |          10
  4 |          10
(2 rows)
```

要理解为什么一个 `NULL` 会清空整个结果，我们必须停止把 `NOT IN` 当成一个单独操作，而要看 parser 如何拆解它。

## IN 是 OR，NOT IN 是 AND

`IN` 不是一个 primitive operator。它是 parser 改写出来的简写：一串 equality comparison，用 `OR` 连接：

```sql
x IN (a, b, c)
-- becomes
x = a OR x = b OR x = c
```

`NOT IN` 是它的逻辑否定。根据德摩根律，否定一串 `OR` equality，会得到一串用 `AND` 连接的不等式：

```sql
x NOT IN (a, b, c)
-- becomes
x <> a AND x <> b AND x <> c
```

这不是类比。这就是 PostgreSQL 实际构建的表达式，你可以直接从 `EXPLAIN` 里读出来。literal-list 形式会折叠成 array operator，名字已经说明了一切：

```sql
EXPLAIN (COSTS OFF) SELECT * FROM products WHERE category_id IN (1, 2, 3);
--  Filter: (category_id = ANY ('{1,2,3}'::integer[]))

EXPLAIN (COSTS OFF) SELECT * FROM products WHERE category_id NOT IN (1, 2, 3);
--  Filter: (category_id <> ALL ('{1,2,3}'::integer[]))
```

`IN` 是 `= ANY`：等于任意一个元素，也就是 `OR`。`NOT IN` 是 `<> ALL`：必须不同于所有元素，也就是 `AND`。

这里的实际节点类型很重要，因为当你 dump parse tree，或者读规范化后的 [`pg_stat_statements`](https://boringsql.com/posts/pg-stat-statements/) 条目时，看到的就是这些东西。literal list 会编译成一个 `ScalarArrayOpExpr`：左边是 scalar，右边是 array，而 `useOr` 这个 flag 就是 `= ANY` 和 `<> ALL` 的全部区别。子查询形式则是另一种节点：`SubLink`。一眼认出这两个名字，就能知道 planner 走的是哪条路径。

如果“`IN` 和 `= ANY` 是同一个 operator”对你来说是新知识：它们会编译成同一个 parse node 和同一个 plan，只是在 plan cache churn 和 selectivity estimates 上拼写不同。本文里的 `NOT IN` 是一个例外：这里的选择不是语法审美问题，而是 correctness 问题。

## 三值逻辑完成剩下的部分

SQL 不是两值逻辑，而是三值逻辑：true、false、unknown。任何和 `NULL` 的比较都会得到 `unknown`，因为 `NULL` 表示“这里没有值”，你不能问一个不存在的值是否不同于 20：

```sql
-- not false. unknown (displayed as a blank)
SELECT 10 <> NULL;
```

现在把产品 1（category 10）代入 archived 集合 `{20, NULL}`，展开 `NOT IN`：

```sql
10 <> 20 AND 10 <> NULL
true     AND unknown
unknown
```

`true AND unknown` 是 `unknown`，不是 `true`。`WHERE` 只保留 predicate 结果为 true 的行。`false` 和 `unknown` 都会导致行被丢弃。所以产品 1 被丢掉。产品 4 同理，也会得到 `unknown`。

一句话总结机制：只要右侧出现一个 `NULL`，尾部那个 `AND unknown` 项就永远不可能为 true，于是整个 `NOT IN` 永远不可能为 true，于是每一行都会被丢弃——无论你有几百万行，也无论行里是什么值。

## 左侧的 NULL 也一样

只把子查询里的 `NULL` 排除还不够。左侧有 `NULL` 也会产生同样的 `unknown`：产品 3 的 `category_id` 是 `NULL`，即使右侧集合完全没有 `NULL`，它也会计算成 `unknown AND unknown`，然后被丢弃。

`IN` 和 `NOT IN` 不是互补关系：同一行可以同时不满足这两个测试。它们之间存在一个 `NULL` 形状的空隙，既不属于 `IN`，也不属于 `NOT IN`。

## 源码里的分叉点

这一切都可以缩减到一个函数里的一个分支。打开 `src/backend/parser/parse_expr.c`，找到 `transformAExprIn`。这个函数负责把 `IN` 和 `NOT IN` list expression 转成 planner 能处理的结构。它首先决定的是：构建 `OR` 还是 `AND`：

```sql
/*
 * If the operator is <>, combine with AND not OR.
 */
if (strcmp(strVal(linitial(a->name)), "<>") == 0)
    useOr = false;
else
    useOr = true;
```

这就是整个分叉。`IN` 带着 operator `=` 进入，于是 `useOr = true`；`NOT IN` 带着 `<>` 进入，于是 `useOr = false`。这个 flag 会一路传到几百行之后最终组装 boolean tree 的位置：

```sql
result = (Node *) makeBoolExpr(useOr ? OR_EXPR : AND_EXPR,
                               list_make2(result, cmp),
                               a->location);
```

`IN` 用 `OR_EXPR`，`NOT IN` 用 `AND_EXPR`。这个函数里没有任何针对 `NULL` 的 special case，也不需要有。三值逻辑行为是选择了 `AND` 之后自然涌现出来的结果。parser 做了显而易见且正确的事情，`NULL` 语义则直接来自标准 boolean logic。如果你非要说这是“bug”，那它属于 SQL 标准；Postgres 只是忠实实现了标准。

### 语法上的不对称：list vs subquery

list 和 subquery 的构建方式不同。list 形式是你刚看到的 `<> ALL` 不等式链。subquery 形式并不是 `<> ALL`：它会变成 `NOT (foo = ANY (subquery))`。形状不同，truth table 相同：比较中出现 `NULL` 会让结果变成 `unknown`，而 `unknown` 会输掉。Planner 看到的正是这个 `NOT (... = ANY ...)` 形状。

## 为什么 planner 救不了你

正确性问题还有一个 plan-shape 层面的孪生问题。看一下 planner 在 200,000 行 `orders` 和 1,000 行 `vip` 的大 schema 上如何处理三个相邻查询。

首先是正向情况：对子查询使用 `IN`：

```sql
EXPLAIN (COSTS OFF) SELECT * FROM orders
WHERE customer_id IN (SELECT customer_id FROM vip);
```

```sql
 Hash Semi Join
   Hash Cond: (orders.customer_id = vip.customer_id)
   ->  Seq Scan on orders
   ->  Hash
         ->  Seq Scan on vip
```

这是干净的 semi join。planner 把子查询提升为一等关系，选择 hash join，并且可以把它和查询中的其他关系自由重排。现在看自然的镜像：找出 customer 不是 VIP 的订单，先用 `NOT EXISTS` 写：

```sql
EXPLAIN (COSTS OFF) SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM vip v WHERE v.customer_id = o.customer_id);
```

```sql
                  QUERY PLAN
----------------------------------------------
 Hash Anti Join
   Hash Cond: (o.customer_id = v.customer_id)
   ->  Seq Scan on orders o
   ->  Hash
         ->  Seq Scan on vip v
```

这是 hash anti join，也就是 semi join 的高效、对称 counterpart。现在把同样意图写成 `NOT IN`：

```sql
EXPLAIN (COSTS OFF) SELECT * FROM orders
WHERE customer_id NOT IN (SELECT customer_id FROM vip);
```

```sql
                          QUERY PLAN
---------------------------------------------------------------
 Seq Scan on orders
   Filter: (NOT (ANY (customer_id = (hashed SubPlan 1).col1)))
   SubPlan 1
     ->  Seq Scan on vip
(4 rows)
```

完全没有 join。子查询塌缩成一个 opaque `SubPlan` filter，被挂在 sequential scan 上。语法里的 `NOT (... = ANY ...)` 形状就直接出现在 filter 里。

这行 filter 的显示方式和版本有关。PostgreSQL 16 会打印更简短的 `Filter: (NOT (hashed SubPlan 1))`；后续版本改了 `EXPLAIN`，暴露出内部 comparison，也就是这里在 18.4 上展示的 `NOT (ANY (... = (hashed SubPlan 1).col1))`。底层 plan 在所有已发布版本里都是同一个 opaque subplan。

最后两个 plan 的区别是结构性的，不是调参细节。`Hash Anti Join` 会从 inner relation 构建一个 hash table，然后把 outer relation 流式扫过一次；如果 inner 超出 `work_mem`，它可以分 batch 并 spill 到磁盘，同时不改变结果。`SubPlan` 不给 planner 这些能力。即使你在这里看到的是 `hashed` variant，它仍然是在 scan 上作为 filter 原地执行：不能在全局 join search 中重排，不能把 outer query 的 join clause 下推进去驱动 index scan，也没有 multi-batch spill 逻辑来应对 hashed set 比估计更大的情况。子查询不再是 optimizer 可以规划的关系，而变成 optimizer 必须调用的 opaque function，每个 outer row 调一次。这和 [没有被 inline 的 CTE](https://boringsql.com/posts/good-cte-bad-cte/) 的 optimization fence 效果一样：planner 根本看不进去。小输入没人注意；大输入时，可能就是毫秒和分钟的区别。

planner 被困在这个形状上的原因，正是前面的 `NULL` 语义。anti join 在找不到 match 时保留一行。但 `NOT IN` 在比较结果为 `unknown` 时必须丢弃一行，而 “unknown” 和 “no match” 不是同一回事。两者恰好在存在 `NULL` 时分歧，所以 planner 历史上无法证明这种 rewrite 安全，因此从不尝试。

在已发布的 PostgreSQL 上，把列声明成 `NOT NULL` 也救不了你。作者在 18.4 上把 outer 和 inner 列都加了 `NOT NULL` 约束后重跑查询，plan 仍然是 opaque `SubPlan` filter。已发布的 planner 甚至不会看这些信息。所以 `NOT IN (subquery)` 从 Postgres 诞生以来一直是 correctness trap；自 8.4（2009）引入 anti join、其他 negated subquery 都学会使用 anti join 以来，它也一直是 planner pessimization。已发布 optimizer 两边都不会救你。

## 修复会在 PostgreSQL 19 落地

上一段终于快过时了。这是性能修复，不是语义修复：它拯救的是本来语义正确但计划很差的情况，也就是 `NOT NULL` 列仍卡在 opaque `SubPlan` 后面的场景；nullable 列仍然保持 SQL 标准规定的行为。作者在 development branch 里追踪到一个 released Postgres 里不存在的函数：`sublink_testexpr_is_not_nullable`，位于 `src/backend/optimizer/plan/subselect.c`。它守在 `convert_ANY_sublink_to_join` 里的一个全新分支前：

```sql
/*
 * Per SQL spec, NOT IN is not ordinarily equivalent to an anti-join, so
 * that by default we have to fail when under_not.  However, if we can
 * prove that neither the outer query's expressions nor the sub-select's
 * output columns can be NULL, and further that the operator itself cannot
 * return NULL for non-null inputs, then the logic is identical and it's
 * safe to convert NOT IN to an anti-join.
 */
if (under_not &&
    (!sublink_testexpr_is_not_nullable(root, sublink) ||
     !query_outputs_are_not_nullable(subselect)))
    return NULL;
```

`git blame` 显示它来自 commit `383eb21ebff`，标题是 “Convert NOT IN sublinks to anti-joins when safe”，由 Richard Guo 在 2026 年 3 月合入。它在 `master` 上，目标是 PostgreSQL 19。它不存在于任何 `REL_18` tag 中，这正解释了为什么作者的 18.4 环境即使列都是 `NOT NULL`，仍然产生 `SubPlan`。commit message 很直白地说明了交易条件：

如果我们可以证明 comparison 两边都不会产生 `NULL`，并且 operator 本身也不会在非 NULL 输入上返回 NULL，那么 `NOT IN` 和 anti join 的行为就完全一致。

这个证明必须建立三件事：

- 两个 operand 都可证明非 `NULL`。来自 schema 的 `NOT NULL` 约束（通过 NOT-NULL-attnums hash table）、outer join nullability tracking（所以 outer join nullable 侧的 `Var` 不算），以及强制某个 `Var` 非空的 qual clause。

- 子查询输出列可证明非 `NULL`，也就是 guard 里的 `query_outputs_are_not_nullable` 部分。

- operator 是 `NULL` 安全的。operator 必须属于 B-tree 或 Hash operator family。这是“行为像普通 boolean comparison，且不会在非 NULL 输入上返回 `NULL`”的代理条件，因为一个会这样返回 `NULL` 的 operator 会破坏它声称支持的 index。

当三者都成立时，`NULL` 处理上的不匹配就消失了，`NOT IN` 终于可以变成 `JOIN_ANTI`：

```sql
result->jointype = under_not ? JOIN_ANTI : JOIN_SEMI;
```

这解除了一个自 anti join 在 8.4 中出现以来就存在的 planner 限制，但只在 planner 能证明没有 `NULL` 会进入 comparison 时才生效。如果你的列 nullable，那么 PostgreSQL 19 和 9.x 一样，你仍然待在老地方。语义没有变；optimizer 只是学会识别那些 `NULL` 可证明不存在的场景。

## 决策矩阵

| 你写的形式 | 内部形状 | 右侧有 NULL | 左侧有 NULL | Planner（≤ PG 18） |
|---|---|---|---|---|
| `IN (1,2,3)` | `= ANY`，一个 `OR` | 被吸收，无害 | 行被丢弃（无匹配） | `ScalarArrayOpExpr` |
| `NOT IN (1,2,3)` | `<> ALL`，一个 `AND` | 所有行被丢弃 | 行被丢弃 | `ScalarArrayOpExpr` |
| `IN (subquery)` | `= ANY` sublink | 被吸收 | 行被丢弃 | Semi Join |
| `NOT IN (subquery)` | `NOT (= ANY)` sublink | 所有行被丢弃 | 行被丢弃 | opaque `SubPlan` filter ¹ |
| `NOT EXISTS (...)` | anti-join sublink | 行保留 | 行保留 | Anti Join |

¹ PostgreSQL 19 会在 outer expression 和 subquery output column 都可证明 `NOT NULL` 时，把它提升为 Anti Join。

## 应该怎么写

不要等 PostgreSQL 19；等它发布后也不要依赖它。它只会在列可证明 `NOT NULL` 时触发。所有可移植修复都归结为一条规则：让 `NULL` 远离 `NOT IN`。实践中就是改写查询，优先顺序如下。

默认使用 `NOT EXISTS`。把它变成习惯，你就不会再踩这个坑。它在所有版本上都有 anti-join 语义，也有原本正常 `NOT IN` 想要得到的 plan，无论列是否 nullable：

```sql
SELECT id, category_id FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM archived a WHERE a.category_id = p.category_id
);
```

它会保留产品 3，也就是 `NULL` category，因为 `NOT EXISTS` 问的是是否存在匹配的 archived row，而不是 `category_id = NULL` 是否为 true。这通常是你真正想要的答案。如果你确实想把 `NULL` category 的行去掉，就显式加上 `AND p.category_id IS NOT NULL`，这样你是在有意做这个决定。

当你必须保留 `NOT IN` 时，过滤掉子查询里的 `NULL`。比如生成查询时你只能编辑括号里的部分：

```sql
SELECT id FROM products
WHERE category_id NOT IN (
    SELECT category_id FROM archived WHERE category_id IS NOT NULL
);
```

这只覆盖右侧 `NULL`，不处理左侧：产品 3 仍会消失，因为左侧 `NULL` 没变。只有在你不能用 `NOT EXISTS` 时才这么做。

对于 whole-set comparison，可以使用 `EXCEPT`。它用 `IS NOT DISTINCT FROM` 匹配行，所以两个 `NULL` 会被视为相等，三值逻辑陷阱不会触发。但同样因为这个规则，如果 `archived` 里有 `NULL`，它会把 `products` 里的 `NULL` 行移除：

```sql
SELECT category_id FROM products
EXCEPT
SELECT category_id FROM archived;
-- returns: {10}
-- product 3 (NULL category) is dropped because archived also has a NULL
```

所以 `NOT EXISTS` 会保留产品 3，而 `EXCEPT` 会丢掉它。选择哪一个，取决于你真正想要的答案。
