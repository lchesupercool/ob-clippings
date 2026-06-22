---
title: "将 Oracle 层次查询迁移到 Amazon Aurora PostgreSQL"
source: "https://aws.amazon.com/blogs/database/migrate-oracle-hierarchical-queries-to-amazon-aurora-postgresql/"
saved: "2026-06-14 15:47:45 CST"
tags: [aws, aurora-postgresql, oracle, hierarchical-query, recursive-cte, migration]
---

# 将 Oracle 层次查询迁移到 Amazon Aurora PostgreSQL

## AWS Database Blog

# 将 Oracle 层次查询迁移到 Amazon Aurora PostgreSQL

我们看到越来越多的组织正在将其数据库工作负载从商业数据库引擎迁移到 [Amazon Aurora](https://aws.amazon.com/rds/aurora/) 数据库环境中。这些组织减少了在常见数据库管理任务、数据中心维护方面的整体工作量，并且摆脱了专有数据库功能和商业许可证的束缚。

AWS 提供了 [AWS Schema Conversion Tool](https://aws.amazon.com/dms/schema-conversion-tool/)（AWS SCT），可简化异构数据库迁移中的模式转换。AWS SCT 会报告需要[手动处理](https://aws.amazon.com/blogs/database/how-to-solve-some-common-challenges-faced-while-migrating-from-oracle-to-postgresql/)才能转换的对象和 SQL。[层次查询](https://docs.oracle.com/cd/B19306_01/server.102/b14200/queries003.htm)的转换需要额外的工作来转换、验证和测试，相较于 AWS SCT 自动转换的对象而言工作量更大。

在本文中，我们将展示如何使用[递归查询](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE)（配合[公用表表达式](https://www.postgresql.org/docs/14/queries-with.html)（CTE））将不同类型的层次查询从 Oracle 迁移到 [Amazon Aurora PostgreSQL 兼容版](https://aws.amazon.com/rds/aurora/postgresql-features/)。我们还将探讨 Aurora PostgreSQL [支持的](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraPostgreSQLReleaseNotes/AuroraPostgreSQL.Extensions.html) [tablefunc](https://www.postgresql.org/docs/current/tablefunc.html) 扩展的一些局限性。

## 层次查询

在 Oracle 中，层次查询用于查询具有父子关系的数据，其中每个子节点只能有一个父节点，而一个父节点可以有多个子节点。这在构建产品谱系、管理者报表和家谱等报表查询时非常有用。层次查询以树状结构显示有组织的行，因此为了检索数据，必须从根节点开始遍历。下图展示了一个示例树结构。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image001.png)

在上图中，层次结构最顶部的第一个节点 A1 称为根节点，其余节点（如 B1、B2 和 B3）是子节点。如果查询需要查找节点 D1 的层次结构，它将从 A1 扫描到 B3，然后到 C2，再向下遍历到 D1。

## 准备工作

我们在本文的示例中全程使用以下示例表和数据。

| Oracle                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | PostgreSQL                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| create table hier_test( emp_no number, ename varchar2(5), job varchar2(50), manager_no number ); insert into hier_test values(10,'A1','CEO',null); insert into hier_test values(11, 'B1', 'VP HARDWARE', 10); insert into hier_test values(12, 'B2', 'VP ADMIN', 10); insert into hier_test values(13, 'B3', 'VP DEVELOPMENT', 10); insert into hier_test values(14, 'C1', 'DIRECTOR DEVELOPMENT', 13); insert into hier_test values(15, 'C2', 'DIRECTOR DEVELOPMENT', 13); insert into hier_test values(16, 'D1', 'MANAGER DEVELOPMENT', 15); insert into hier_test values(17 ,'E1', 'ENGINEER HARDWARE', 11); insert into hier_test values(18, 'E2', 'ENGINEER HARDWARE', 11); | create table hier_test( emp_no int, ename varchar(5), job varchar(50), manager_no int ); insert into hier_test values(10,'A1','CEO',null); insert into hier_test values(11, 'B1', 'VP HARDWARE', 10); insert into hier_test values(12, 'B2', 'VP ADMIN', 10); insert into hier_test values(13, 'B3', 'VP DEVELOPMENT', 10); insert into hier_test values(14, 'C1', 'DIRECTOR DEVELOPMENT', 13); insert into hier_test values(15, 'C2', 'DIRECTOR DEVELOPMENT', 13); insert into hier_test values(16, 'D1', 'MANAGER DEVELOPMENT', 15); insert into hier_test values(17 ,'E1', 'ENGINEER HARDWARE', 11); insert into hier_test values(18, 'E2', 'ENGINEER HARDWARE', 11); |

虽然 PostgreSQL 没有直接处理层次查询的函数或预定义关键字，但你可以借助 [tablefunc](https://www.postgresql.org/docs/current/tablefunc.html) 扩展和 CTE 来定义自定义解决方案。`Tablefunc` 对带有 `connect by` 和 `level` 关键字的层次查询很有用，但通过 CTE，我们可以支持层次查询中的各种关键字，如 `LEVEL`、`NOCYCLE`、`SYS_CONNECT_BY_PATH`、`ORDER SIBLINGS BY`、`CONNECT_BY_ISLEAF` 和 `CONNECT_BY_ROOT`。我们将在以下各节中详细讨论这些场景。

## tablefunc 扩展

`tablefunc` 扩展有一个名为 `connectby` 的函数，它可以生成存储在表中的层次数据的展示。

要使 `connectby` 函数正常工作，表需要具备以下条件：

- 一个唯一标识行的键字段
- 一个引用每一行的父行（如果有的话）的父键字段

此函数可以显示从某一行开始的子树。该函数的主要用例是展示父子连接（层次数据）。

以下代码展示了我们的 PostgreSQL 查询：

```sql
SELECT * FROM 
  Connectby('hier_test', 'emp_no', 'manager_no', '10', 0, '->') AS t(emp_no int, manager_no int, level int, ord text) 
  order by emp_no;
```

以下截图展示了我们的输出。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image003.png)

`connectby` 函数在仅选择父子属性和层级属性时表现最佳。例如，在以下查询中，获取 `ename` 属性会导致错误：

```sql
SELECT * FROM 
  Connectby('hier_test', 'emp_no', 'manager_no', '10', 0, '->','ename') AS 
  t(emp_no int, manager_no int, level int, ord text,ename text) 
  order by emp_no;
```

我们得到如下输出。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image005.png)

此外，`connectby` 不能与使用 `NOCYCLE`、`CONNECT_BY_ISLEAF`、`SYS_CONNECT_BY_PATH`、`ORDER SIBLINGS BY` 和 `CONNECT_BY_ROOT` 等关键字构建的层次查询一起使用。

## 递归查询

我们可以使用 CTE 递归查询实现层次查询。递归查询由一个 `UNION ALL` 定义，其中包含一个初始化 `fullselect`，用于种下递归的种子。迭代（递归）的 `fullselect` 在 `FROM` 子句中包含对自身的直接引用。参见以下代码：

```sql
WITH RECURSIVE <tab_name>(column_list)
AS
(
    -- seed query
    Anchor query  
    UNION ALL
    -- Recursive member that references <tab_name>.
    recursive_query  
)
-- references expression name
SELECT *
FROM <tab_name> ;
```

让我们逐一了解 Oracle 层次查询的各种用例，并使用 PostgreSQL 中的递归 SQL 实现类似的功能。

## 场景 1：显示员工级别及其他详细信息

关键字 `Level` 描述节点在层次结构中的深度。第一层是层次的根。

以下代码展示了我们的 Oracle 查询：

```sql
SELECT   emp_no,ename,job,level
FROM hier_test
  CONNECT BY PRIOR emp_no = manager_no
  START WITH manager_no IS NULL
  order by level ;
```

以下截图展示了我们的结果。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image007-1.png)

在 PostgreSQL 中，非递归部分生成层次结构的根（自顶向下），即没有经理（`manager_no is null`）或具有特定经理（`manager_n = 10`）的员工。递归部分通过将主表与非递归查询的输出进行连接来生成层次结构，直到连接条件（`e.manager_no = c.emp_no`）成立为止。参见以下 PostgreSQL 查询：

```sql
WITH RECURSIVE cte AS (
  SELECT emp_no, ename,job, manager_no, 1 AS level
  FROM   hier_test
  where manager_no is null
  UNION  ALL
  SELECT e.emp_no, e.ename, e.job,e.manager_no, c.level + 1
  FROM   cte c
  JOIN   hier_test e ON e.manager_no = c.emp_no
  )
  SELECT emp_no,ename,job,level
  FROM   cte
  order by level;
```

我们得到如下输出。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image009.png)

## 场景 2：使用 SYS_CONNECT_BY_PATH 的层次查询

关键字 [SYS_CONNECT_BY_PATH](https://docs.oracle.com/cd/B19306_01/server.102/b14200/functions164.htm#i1038266) 返回从根节点到当前节点的列值路径，对于 CONNECT BY 条件返回的每一行，列值由字符（分隔符）分隔。

使用以下 Oracle 查询：

```sql
SELECT  emp_no,ename,job,level,SYS_CONNECT_BY_PATH(ename,';')
FROM hier_test
CONNECT BY PRIOR  emp_no = manager_no
START WITH manager_no is null
order by level ;
```

我们得到如下结果。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image011.png)

在 PostgreSQL 中，我们可以通过在每个迭代中将父子记录属性与字符/分隔符拼接起来，来实现类似于 `SYS_CONNECT_BY_PATH` 的功能。参见以下代码：

```sql
WITH RECURSIVE cte(emp_no, manager_no, ename,job, level, path)
AS (
  SELECT emp_no, manager_no, ename, job,1 AS level,
  ';'||ename AS path
	FROM hier_test
	WHERE manager_no is null
UNION ALL
  SELECT e.emp_no, e.manager_no, e.ename,e.job, c.level + 1 AS level,
  c.path||';'||e.ename AS path
FROM hier_test e, cte c
	WHERE e.manager_no = c.emp_no
)
SELECT emp_no ,ename ,job,level,path
FROM cte 
order by level,emp_no;
```

我们得到如下输出。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image013.png)

## 场景 3：使用 NOCYCLE 的层次查询

`NOCYCLE` 参数指示 Oracle 数据库即使在数据中存在 `CONNECT BY LOOP` 的情况下也返回查询结果行。

如果数据中存在子节点是父节点、父节点是子节点的情况，层次查询就会陷入数据循环。`NOCYCLE` 关键字帮助我们避免这种循环。

为了在我们的数据中创建循环，我们添加了一条记录，其中另一名员工的 `emp_no` 为 13，其经理为 `emp_no` 14。添加新记录后，示例数据如下截图所示。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image015-1.png)

当你在 Oracle 中运行上一场景的 SQL 代码而不添加 `NOCYCLE` 时，会遇到错误：

```sql
SELECT  emp_no,ename,job,level,SYS_CONNECT_BY_PATH(ename,';')
FROM hier_test
CONNECT BY PRIOR  emp_no = manager_no
START WITH manager_no is null
order by level ;
```

以下截图展示了我们的输出。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image017.png)

在 Oracle 查询中添加 `NOCYCLE` 关键字可以得到预期结果。参见以下代码：

```sql
SELECT  emp_no,ename,job,level,SYS_CONNECT_BY_PATH(ename,';')
FROM hier_test
CONNECT BY NOCYCLE PRIOR  emp_no = manager_no
START WITH manager_no is null
order by level ;
```

我们得到如下结果。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image019.png)

在 PostgreSQL 中，我们使用两个字段 `route` 和 `cycle` 来实现类似于 `NOCYCLE` 的功能。`route` 是一个已访问值的数组，`cycle` 是一个标志，根据某个值是否已存在于 `route` 中来设置，条件 `cycle = false` 会过滤掉循环记录。参见以下代码：

```sql
WITH RECURSIVE cte(emp_no, manager_no, ename,job,  level, route,cycle, path)
AS (
  SELECT emp_no, manager_no, ename, job,1 AS level, array[emp_no] AS route,false AS cycle,
  ';'||ename AS path
  FROM hier_test
  WHERE manager_no is null
UNION ALL
  SELECT e.emp_no, e.manager_no, e.ename,e.job, c.level + 1 AS level,c.route || e.emp_no ,e.emp_no = ANY(c.route) as cycle,
  c.path||';'||e.ename AS path
FROM hier_test e, cte c
	WHERE e.manager_no = c.emp_no AND cycle = false
)
SELECT emp_no,ename,job,level,path
FROM cte
WHERE cycle = false ;
```

以下截图展示了我们的输出。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image021.png)

## 场景 4：使用 ORDER SIBLINGS BY 的层次查询

可选的 [SIBLINGS](https://docs.oracle.com/cd/B19306_01/server.102/b14200/statements_10002.htm#i2171079) 关键字指定一种排序方式：首先对父行排序，然后对层次结构每个层级中每个父行的子行进行排序。参见以下 Oracle 查询：

```sql
SELECT emp_no,ename,job,manager_no,level
 from hier_test
 start with manager_no is null
 CONNECT BY nocycle PRIOR  emp_no = manager_no
order siblings by ename ;
```

以下截图展示了我们的结果。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image023-1.png)

在 PostgreSQL 中，我们可以通过按路径对 CTE 输出排序来实现类似于 `ORDER BY SIBLINGS` 的功能。路径是由 Oracle 查询中 ORDER BY 子句提到的属性拼接而成的。

在以下 PostgreSQL 查询中，path 属性包含 `ename`（可选用 `emp_no` 来处理同一经理下有不同 `emp_no` 但相同 `ename` 的情况）：

```sql
WITH RECURSIVE cte(emp_no, manager_no, ename,job,  level, route,cycle, path)
AS (
  SELECT
        emp_no, manager_no, ename, job,1 AS level, array[emp_no] AS route,false AS cycle,
  ';'||ename||emp_no AS path
  FROM hier_test
  WHERE manager_no is null
UNION ALL
  SELECT
	e.emp_no, e.manager_no, e.ename,e.job, c.level + 1 AS level,c.route || e.emp_no ,e.emp_no = ANY(c.route) as cycle,
  c.path||';'||e.ename||e.emp_no  AS path
FROM hier_test e, cte c
	WHERE e.manager_no = c.emp_no AND cycle = false
)
SELECT
    emp_no,ename,job,manager_no,level
FROM cte
WHERE cycle = false
ORDER  BY path,emp_no ;
```

我们得到如下输出。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image025.png)

## 场景 5：使用 CONNECT_BY_ISLEAF 的层次查询

[CONNECT_BY_ISLEAF](https://docs.oracle.com/cd/B19306_01/server.102/b14200/pseudocolumns001.htm#i1009434) 伪列在当前行是由 CONNECT BY 条件定义的树的叶节点时返回 `1`。否则返回 `0`。此信息表明给定行是否可以进一步展开以显示更多层次结构。

使用以下 Oracle 查询：

```sql
SELECT emp_no,ename,job,manager_no,level,SYS_CONNECT_BY_PATH (ename,';') PATH,CONNECT_BY_ISLEAF  ISLEAF
 from hier_test
 start with manager_no is null
 CONNECT BY nocycle PRIOR  emp_no = manager_no
order siblings by job ;
```

我们得到如下结果。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image027.png)

在 PostgreSQL 中，我们可以通过检查子节点是否属于 CTE 返回的父节点集合来实现类似于 `CONNECT_BY_ISLEAF` 的功能。参见以下代码：

```sql
WITH RECURSIVE cte(emp_no, manager_no, ename,job,  level, route,cycle, path)
AS (
  SELECT
        emp_no, manager_no, ename, job,1 AS level, array[emp_no] AS route,false AS cycle, ';'||ename AS path,  ';'||job||emp_no AS ordersnblpath ,emp_no as root_id
  FROM hier_test
  WHERE manager_no is null
UNION ALL
  SELECT
	e.emp_no, e.manager_no, e.ename,e.job, c.level + 1 AS level,c.route || e.emp_no ,e.emp_no = ANY(c.route) as cycle, c.path||';'||e.ename AS path,
  c.ordersnblpath||';'||e.job||e.emp_no AS ordersnblpath, c.root_id
FROM hier_test e, cte c
	WHERE e.manager_no = c.emp_no AND cycle = false
)
SELECT
    emp_no,ename,job,manager_no,level,path
	, not exists (select * from cte p where p.manager_no = e.emp_no and cycle = false) as is_leaf
FROM cte e
WHERE cycle = false
ORDER BY ordersnblpath,emp_no ;
```

以下截图展示了我们的输出。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image029.png)

## 场景 6：使用 CONNECT_BY_ROOT 的层次查询

[CONNECT_BY_ROOT](https://docs.oracle.com/cd/B19306_01/server.102/b14200/operators004.htm#i1035022) 是一个一元运算符，仅在层次查询中有效。当你用此运算符限定一个列时，Oracle 会使用根行的数据返回列值。参见以下查询：

```sql
SELECT emp_no,ename,job,manager_no,level,SYS_CONNECT_BY_PATH (ename,';') PATH,CONNECT_BY_ROOT ename
 from hier_test
 start with manager_no is null
 CONNECT BY nocycle PRIOR  emp_no = manager_no;
```

我们得到如下结果。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image031.png)

在 PostgreSQL 中，我们可以通过对 CTE 输出中的 path 属性使用 `substr` 或 split 函数来实现类似于 `CONNECT_BY_ROOT` 的功能：

```sql
WITH RECURSIVE cte(emp_no, manager_no, ename,job,level, route,cycle, path)
AS (
  SELECT
        emp_no, manager_no, ename, job,1 AS level, array[emp_no] AS route,false AS cycle,
  ';'||ename AS path ,emp_no as root_id
  FROM hier_test
  WHERE manager_no is null
UNION ALL
  SELECT
	e.emp_no, e.manager_no, e.ename,e.job, c.level + 1 AS level,c.route || e.emp_no ,e.emp_no = ANY(c.route) as cycle,
  c.path||';'||e.ename AS path, c.root_id
FROM hier_test e, cte c
	WHERE e.manager_no = c.emp_no AND cycle = false
)
SELECT
    emp_no,ename,job,manager_no,level,path
	   ,SPLIT_PART(path,';',2) root
FROM cte e
WHERE cycle = false
ORDER  BY path, emp_no ; 
```

我们得到如下输出。

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image033.png)

## 注意事项

在你的环境中使用此解决方案时，请考虑以下事项：

- 我们在本文中使用的数据集性质简单，可能无法反映你环境中数据的复杂性
- 本文并未讨论 Oracle 层次查询的所有场景或关键字
- 请参考示例查询对你将要构建的解决方案和查询进行功能和性能需求测试

## 结论

在本文中，我们通过示例查询演示了如何将使用 `LEVEL`、`NOCYCLE`、`SYS_CONNECT_BY_PATH`、`ORDER SIBLINGS BY`、`CONNECT_BY_ISLEAF` 和 `CONNECT_BY_ROOT` 等关键字的 Oracle 层次查询迁移到 PostgreSQL。我们还讨论了在迁移 Oracle 层次查询时 `tablefunc` 扩展的用例和不足之处。

请查阅 [Database Migration—What Do You Need to Know Before You Start?](https://aws.amazon.com/blogs/database/database-migration-what-do-you-need-to-know-before-you-start/) 开始迁移。同时，请参考推荐的最佳实践，包括[迁移流程和基础设施注意事项](https://aws.amazon.com/blogs/database/best-practices-for-migrating-an-oracle-database-to-amazon-rds-postgresql-or-amazon-aurora-postgresql-migration-process-and-infrastructure-considerations/)、[源数据库注意事项](https://aws.amazon.com/blogs/database/best-practices-for-migrating-an-oracle-database-to-amazon-rds-postgresql-or-amazon-aurora-postgresql-source-database-considerations-for-the-oracle-and-aws-dms-cdc-environment/)，以及[PostgreSQL 环境的目标数据库注意事项](https://aws.amazon.com/blogs/database/best-practices-for-migrating-an-oracle-database-to-amazon-rds-postgresql-or-amazon-aurora-postgresql-target-database-considerations-for-the-postgresql-environment/)。

如有任何问题、评论或其他反馈，请在 [Amazon Aurora Discussion Forums](https://forums.aws.amazon.com/forum.jspa?forumID=227) 上分享你的想法。

### 关于作者

**![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/Rakesh-Raghav.png)Rakesh Raghav** 是 AWS Professional Services（印度）的数据库专家，帮助客户成功实现云采用和迁移之旅。他热衷于构建创新解决方案，加速客户的数据库上云之旅。

**![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/Anuradha-Chintha.jpg)Anuradha Chintha** 是 Amazon Web Services 的首席顾问。她与客户合作，在 AWS 云上构建可扩展、高可用且安全的解决方案。她的重点领域是同构和异构数据库迁移。
