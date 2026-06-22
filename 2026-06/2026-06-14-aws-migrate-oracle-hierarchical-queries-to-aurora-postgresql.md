---
title: "Migrate Oracle hierarchical queries to Amazon Aurora PostgreSQL"
source: "https://aws.amazon.com/blogs/database/migrate-oracle-hierarchical-queries-to-amazon-aurora-postgresql/"
saved: "2026-06-14 15:47:45 CST"
tags: [aws, aurora-postgresql, oracle, hierarchical-query, recursive-cte, migration]
---

# Migrate Oracle hierarchical queries to Amazon Aurora PostgreSQL

## AWS Database Blog

# Migrate Oracle hierarchical queries to Amazon Aurora PostgreSQL

We have seen a number of organizations are migrating their database workloads from commercial database engines to the [Amazon Aurora](https://aws.amazon.com/rds/aurora/) database environment. These organizations have reduced their overall efforts on common database administration tasks, data center maintenance, and have moved away from proprietary database features and commercial licenses.

AWS provides the [AWS Schema Conversion Tool](https://aws.amazon.com/dms/schema-conversion-tool/) (AWS SCT) which simplifies schema conversion for heterogeneous database migration. AWS SCT reports objects and SQL requiring [manual efforts](https://aws.amazon.com/blogs/database/how-to-solve-some-common-challenges-faced-while-migrating-from-oracle-to-postgresql/) for conversion. [Hierarchical queries](https://docs.oracle.com/cd/B19306_01/server.102/b14200/queries003.htm) conversion requires additional efforts to convert, validate, and test in comparison to objects automatically converted by AWS SCT.

In this post, we show you how to migrate different hierarchical queries from Oracle to [Amazon Aurora PostgreSQL-Compatible Edition](https://aws.amazon.com/rds/aurora/postgresql-features/) using [recursive queries](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE) in [common table expressions](https://www.postgresql.org/docs/14/queries-with.html) (CTE). We also look at a few limitations of the [tablefunc](https://www.postgresql.org/docs/current/tablefunc.html) extension [supported](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraPostgreSQLReleaseNotes/AuroraPostgreSQL.Extensions.html) by Aurora PostgreSQL.

## Hierarchical queries

In Oracle, hierarchical queries are used to query data that has a parent-child relationship where each child can have only one parent, whereas a parent can have multiple children. This is very useful when trying to build reporting queries such as product lineage, manager reports, and a family tree. A hierarchical query displays organized rows in a tree structure, so in order to retrieve the data, it has to be traversed starting from the root. The following diagram illustrates a sample tree structure.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image001.png)

In the above diagram, the first node at the top of the hierarchy, A1, is called the root node, and the rest of the nodes such as B1, B2, and B3 are the children nodes. If a query needs to find the hierarchy of node D1, it will scan the tree from A1 to B3 and then to C2, traversing down to D1.

## Prerequisites

We use the following sample table and data throughout our examples in this post.

| Oracle | PostgreSQL |
| --- | --- |
| create table hier_test( emp_no number, ename varchar2(5), job varchar2(50), manager_no number ); insert into hier_test values(10,'A1','CEO',null); insert into hier_test values(11, 'B1', 'VP HARDWARE', 10); insert into hier_test values(12, 'B2', 'VP ADMIN', 10); insert into hier_test values(13, 'B3', 'VP DEVELOPMENT', 10); insert into hier_test values(14, 'C1', 'DIRECTOR DEVELOPMENT', 13); insert into hier_test values(15, 'C2', 'DIRECTOR DEVELOPMENT', 13); insert into hier_test values(16, 'D1', 'MANAGER DEVELOPMENT', 15); insert into hier_test values(17 ,'E1', 'ENGINEER HARDWARE', 11); insert into hier_test values(18, 'E2', 'ENGINEER HARDWARE', 11); | create table hier_test( emp_no int, ename varchar(5), job varchar(50), manager_no int ); insert into hier_test values(10,'A1','CEO',null); insert into hier_test values(11, 'B1', 'VP HARDWARE', 10); insert into hier_test values(12, 'B2', 'VP ADMIN', 10); insert into hier_test values(13, 'B3', 'VP DEVELOPMENT', 10); insert into hier_test values(14, 'C1', 'DIRECTOR DEVELOPMENT', 13); insert into hier_test values(15, 'C2', 'DIRECTOR DEVELOPMENT', 13); insert into hier_test values(16, 'D1', 'MANAGER DEVELOPMENT', 15); insert into hier_test values(17 ,'E1', 'ENGINEER HARDWARE', 11); insert into hier_test values(18, 'E2', 'ENGINEER HARDWARE', 11); |

Although PostgreSQL doesn’t have functions or predefined keywords to handle the hierarchical queries directly, you can define custom solutions with the help of the [tablefunc](https://www.postgresql.org/docs/current/tablefunc.html) extension and CTEs. `Tablefunc` is useful for hierarchical queries with `connect by` and level keywords, but with CTE we can support various types of keywords such as `LEVEL`, `NOCYCLE`, `SYS_CONNECT_BY_PATH`, `ORDER SIBLINGS BY`, `CONNECT_BY_ISLEAF`, and `CONNECT_BY_ROOT` of hierarchical queries. We are going to discuss and look at these scenarios in details in the following sections.

## tablefunc extension

The `tablefunc` extension has a function called `connectby`, which produces a display of hierarchical data that is stored in a table.

For the `connectby` function to work, the table needs the following:

- A key field that uniquely identifies rows
- A parent-key field that references the parent (if any) of each row

This function can display the sub-tree descending from a row. The primary use case of this function is to display parent-child connections (hierarchy data).

The following code demonstrates our PostgreSQL query:

```sql
SELECT * FROM 
  Connectby(‘hier_test’, ‘emp_no’, ‘manager_no’, ‘10’, 0, ‘->’) AS t(emp_no int, manager_no int, level int, ord text) 
  order by emp_no;
```

The following screenshot shows our output.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image003.png)

The `connectby` function works best when only the parent, child, and level attributes are selected. For example, in the following query, fetching the `ename` attribute results in an error:

```sql
SELECT * FROM 
  Connectby('hier_test', 'emp_no', 'manager_no', '10', 0, '->','ename') AS 
  t(emp_no int, manager_no int, level int, ord text,ename text) 
  order by emp_no;
```

We get the following output.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image005.png)

Additionally, `connectby` cannot be used with hierarchical queries built using keywords like `NOCYCLE`, `CONNECT_BY_ISLEAF`, `SYS_CONNECT_BY_PATH`, `ORDER SIBLINGS BY`, and `CONNECT_BY_ROOT`.

## Recursive queries

We can achieve hierarchical queries using CTE recursive queries. A recursive query is one that is defined by a `UNION ALL` with an initialization `fullselect` that seeds the recursion. The iterative (recursive) `fullselect` contains a direct reference to itself in the `FROM` clause. See the following code:

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

Let’s go through various use cases of Oracle hierarchical queries and achieve similar functionality using recursive SQL in PostgreSQL.

## Scenario 1: Display employee level along with other details

The keyword `Level` describes the depth of node in the hierarchy. The first level is the root of hierarchy.

The following code shows our Oracle query:

```sql
SELECT   emp_no,ename,job,level
FROM hier_test
  CONNECT BY PRIOR emp_no = manager_no
  START WITH manager_no IS NULL
  order by level ;
```

The following screenshot shows our results.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image007-1.png)

In PostgreSQL, the non-recursive part generates the root of the hierarchy (top-down), which is the employee with no manager ( `manager_no is null`) or with a specific manager (`manager_n = 10`). The recursive part generates the hierarchy by joining the main table with the output of the non-recursive query until the join condition (`e.manager_no = c.emp_no`) is true. See the following PostgreSQL query:

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

We get the following output.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image009.png)

## Scenario 2: Hierarchical queries with SYS_CONNECT_BY_PATH

The keyword [SYS_CONNECT_BY_PATH](https://docs.oracle.com/cd/B19306_01/server.102/b14200/functions164.htm#i1038266) returns the path of a column value from root to node, with column values separated by char(delimiter) for each row returned by the CONNECT BY condition.

Use the following Oracle query:

```sql
SELECT  emp_no,ename,job,level,SYS_CONNECT_BY_PATH(ename,';')
FROM hier_test
CONNECT BY PRIOR  emp_no = manager_no
START WITH manager_no is null
order by level ;
```

We get the following results.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image011.png)

In PostgreSQL, we can achieve a functionality similar to `SYS_CONNECT_BY_PATH` by concatenating the parent and child record attributes with a char/delimiter in every iteration. See the following code:

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

We get the following output.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image013.png)

## Scenario 3: Hierarchical queries with NOCYCLE

The `NOCYCLE` parameter instructs Oracle Database to return rows from a query even if a `CONNECT BY LOOP` exists in the data.

If there is data in which the child is the parent and the parent is the child, then the hierarchical query goes into a data loop. The `NOCYCLE` keyword helps us avoid this loop.

To create a cycle in our data, we added a record of another employee as `emp_no` 13 with their manager as `emp_no` 14. With our new record, the sample data looks like the following screenshot.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image015-1.png)

When you run the SQL code from the previous scenario in Oracle without adding `NOCYCLE`, you encounter an error:

```sql
SELECT  emp_no,ename,job,level,SYS_CONNECT_BY_PATH(ename,';')
FROM hier_test
CONNECT BY PRIOR  emp_no = manager_no
START WITH manager_no is null
order by level ;
```

The following screenshot shows our output.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image017.png)

Adding the `NOCYCLE` keyword in the Oracle query gives you expected results. See the following code:

```sql
SELECT  emp_no,ename,job,level,SYS_CONNECT_BY_PATH(ename,';')
FROM hier_test
CONNECT BY NOCYCLE PRIOR  emp_no = manager_no
START WITH manager_no is null
order by level ;
```

We get the following results.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image019.png)

In PostgreSQL, we use two fields, `route` and `cycle`, to achieve functionality similar to `NOCYCLE`*.*`route` is an array of already visited values and `cycle` is a flag that gets set based on if a value is already present in `route` and the condition `cycle = false` filters out cyclic records. See the following code:

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

The following screenshot shows our output.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image021.png)

## Scenario 4: Hierarchical queries with ORDER SIBLINGS BY

The optional [SIBLINGS](https://docs.oracle.com/cd/B19306_01/server.102/b14200/statements_10002.htm#i2171079) keyword specifies an order that first sorts the parent rows, then sorts the child rows of each parent for each level within the hierarchy. See the following Oracle query:

```sql
SELECT emp_no,ename,job,manager_no,level
 from hier_test
 start with manager_no is null
 CONNECT BY nocycle PRIOR  emp_no = manager_no
order siblings by ename ;
```

The following screenshot shows our results.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image023-1.png)

In PostgreSQL, we can achieve a functionality similar to `ORDER BY SIBLINGS` by ordering the CTE output by path. The path is a concatenation of attributes mentioned in the ORDER BY clause in the Oracle query.

In the following PostgreSQL query, the path attribute has `ename` (`emp_no` is optionally included to handle scenarios of different `emp_no` values with the same `ename` under the same manager):

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

We get the following output.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image025.png)

## Scenario 5: Hierarchical queries with CONNECT_BY_ISLEAF

The [CONNECT_BY_ISLEAF](https://docs.oracle.com/cd/B19306_01/server.102/b14200/pseudocolumns001.htm#i1009434) pseudo column returns `1` if the current row is a leaf of the tree defined by the CONNECT BY condition. Otherwise it returns `0`. This information indicates whether a given row can be further expanded to show more of the hierarchy.

Use the following Oracle query:

```sql
SELECT emp_no,ename,job,manager_no,level,SYS_CONNECT_BY_PATH (ename,';') PATH,CONNECT_BY_ISLEAF  ISLEAF
 from hier_test
 start with manager_no is null
 CONNECT BY nocycle PRIOR  emp_no = manager_no
order siblings by job ;
```

We get the following results.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image027.png)

In PostgreSQL, we can achieve a functionality similar to `CONNECT_BY_ISLEAF` by checking whether the child node is a part of the parent nodes returned by the CTE or not. See the following code:

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

The following screenshot shows our output.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image029.png)

## Scenario 6: Hierarchical queries with CONNECT_BY_ROOT

[CONNECT_BY_ROOT](https://docs.oracle.com/cd/B19306_01/server.102/b14200/operators004.htm#i1035022) is a unary operator that is valid only in hierarchical queries. When you qualify a column with this operator, Oracle returns the column value using data from the root row. See the following query:

```sql
SELECT emp_no,ename,job,manager_no,level,SYS_CONNECT_BY_PATH (ename,';') PATH,CONNECT_BY_ROOT ename
 from hier_test
 start with manager_no is null
 CONNECT BY nocycle PRIOR  emp_no = manager_no;
```

We get the following results.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image031.png)

In PostgreSQL, we can achieve functionality similar to `CONNECT_BY_ROOT` by using `substr` or a split function on the path attributed in the CTE output:

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

We get the following output.

![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/DBBLOG-2220-image033.png)

## Considerations

Consider the following when using this solution in your environment:

- The dataset we used for this post is simple in nature and may not reflect the data complexity of your environment
- Not all scenarios or keywords of Oracle’s hierarchical queries are discussed in this post
- Test the solution and queries you’re going to build referencing the sample queries for functional and performance requirements

## Conclusion

In this post, we demonstrated via sample queries how you can migrate Oracle hierarchical queries using keywords `LEVEL`, `NOCYCLE`, `SYS_CONNECT_BY_PATH`, `ORDER SIBLINGS BY`, `CONNECT_BY_ISLEAF`, and `CONNECT_BY_ROOT` to PostgreSQL. We also talked about use cases and shortcomings of the `tablefunc` extension when migrating Oracle hierarchical queries.

Check out [Database Migration—What Do You Need to Know Before You Start?](https://aws.amazon.com/blogs/database/database-migration-what-do-you-need-to-know-before-you-start/) to get started. Also review the recommended best practices, including [the migration process and infrastructure considerations](https://aws.amazon.com/blogs/database/best-practices-for-migrating-an-oracle-database-to-amazon-rds-postgresql-or-amazon-aurora-postgresql-migration-process-and-infrastructure-considerations/), [source database considerations](https://aws.amazon.com/blogs/database/best-practices-for-migrating-an-oracle-database-to-amazon-rds-postgresql-or-amazon-aurora-postgresql-source-database-considerations-for-the-oracle-and-aws-dms-cdc-environment/), and [target database considerations for the PostgreSQL environment](https://aws.amazon.com/blogs/database/best-practices-for-migrating-an-oracle-database-to-amazon-rds-postgresql-or-amazon-aurora-postgresql-target-database-considerations-for-the-postgresql-environment/).

If you have any questions, comments, or other feedback, share your thoughts on the [Amazon Aurora Discussion Forums](https://forums.aws.amazon.com/forum.jspa?forumID=227).

### About the Authors

**![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/Rakesh-Raghav.png)Rakesh Raghav** is a Database Specialist with the AWS Professional Services in India, helping customers have a successful cloud adoption and migration journey. He is passionate about building innovative solutions to accelerate their database journey to cloud.

**![](../../assets/aws-migrate-oracle-hierarchical-queries-to-aurora-postgresql/Anuradha-Chintha.jpg)Anuradha Chintha** is a Lead Consultant with Amazon Web Services. She works with customers to build scalable, highly available, and secure solutions in the AWS Cloud. Her focus area is homogeneous and heterogeneous database migrations.
