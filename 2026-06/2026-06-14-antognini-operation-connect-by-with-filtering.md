---
title: "Operation CONNECT BY WITH FILTERING"
source: "https://antognini.ch/2008/06/operation-connect-by-with-filtering/"
saved: "2026-06-14 15:47:45 CST"
tags: [oracle, optimizer, connect-by, hierarchical-query, execution-plan]
---

# Operation CONNECT BY WITH FILTERING

# Operation CONNECT BY WITH FILTERING

![](../../assets/antognini-operation-connect-by-with-filtering/fe3509bf34a739bb7054e47e8723b99c52b00933e3b31aa3d3edbd7c09bb0dd7.png)

It happened to me several times to being asked about the mysterious full table scan in CONNECT BY operations. In this post I would like to share with you some of the information I wrote about it in my [book](https://antognini.ch/top) (pages 233 to 236) .

The operation CONNECT BY WITH FILTERING is used to process hierarchical queries. It is characterized by two child operations. The first one is used to get the root of the hierarchy, and the second one is executed once for each level in the hierarchy.

Here is a sample query and its execution plan. Note that the execution plan was generated on Oracle Database 11g (the reason will be explained later).

```sql
SELECT level, rpad('-',level-1,'-')||ename AS ename, prior ename AS manager
FROM emp
START WITH mgr IS NULL
CONNECT BY PRIOR empno = mgr

---------------------------------------------------------------------
| Id  | Operation                     | Name      | Starts | A-Rows |
---------------------------------------------------------------------
|*  1 |  CONNECT BY WITH FILTERING    |           |      1 |     14 |
|*  2 |   TABLE ACCESS FULL           | EMP       |      1 |      1 |
|   3 |   NESTED LOOPS                |           |      4 |     13 |
|   4 |    CONNECT BY PUMP            |           |      4 |     14 |
|   5 |    TABLE ACCESS BY INDEX ROWID| EMP       |     14 |     13 |
|*  6 |     INDEX RANGE SCAN          | EMP_MGR_I |     14 |     13 |
---------------------------------------------------------------------

   1 - access("MGR"=PRIOR "EMPNO")
   2 - filter("MGR" IS NULL)
   6 - access("MGR"=PRIOR "EMPNO")
```

To help you understand the execution plan with a hierarchical query more easily, it is useful to look at the data returned by the query as well:

```text
     LEVEL ENAME      MANAGER
---------- ---------- ----------
         1 KING
         2 -JONES     KING
         3 --SCOTT    JONES
         4 ---ADAMS   SCOTT
         3 --FORD     JONES
         4 ---SMITH   FORD
         2 -BLAKE     KING
         3 --ALLEN    BLAKE
         3 --WARD     BLAKE
         3 --MARTIN   BLAKE
         3 --TURNER   BLAKE
         3 --JAMES    BLAKE
         2 -CLARK     KING
         3 --MILLER   CLARK
```

The execution plan carries out the operations as follows:

1. Operation 1 has two children (2 and 3), and operation 2 is the first of them in ascending order. Therefore, the execution starts with operation 2.
2. Operation 2 scans the table EMP, applies the filter predicate “MGR” IS NULL, and returns the root of the hierarchy to its parent operation (1).
3. Operation 3 is the second child of operation 1. It is therefore executed for each level of the hierarchy—in this case, four times. The first child, operation 4, is executed, and for each row it returns, the inner loop (composed of operation 5 and its child operation 6) is executed once. Notice, as expected, the match between the column A-Rows of operation 4 with the column Starts of operations 5 and 6.
4. For the first execution, operation 4 gets the root of the hierarchy through the operation CONNECT BY PUMP. In this case, there is a single row (KING) at level 1. With the value in the column mgr, operation 6 does a scan of the index EMP_MGR_I by applying the access predicate “MGR”=PRIOR “EMPNO”, extracts the rowids, and returns them to its parent operation (5). Operation 5 accesses the table EMP with the rowids and returns the rows to its parent operation (3).
5. For the second execution of operation 4, everything works the same as for the first execution. The only difference is that the data from level 2 (JONES, BLAKE, and CLARK) is passed to operation 4 for the processing.
6. For the third execution of operation 4, everything works like in the first one. The only difference is that level 3 data (SCOTT, FORD, ALLEN, WARD, MARTIN, TURNER, JAMES, and MILLER) is passed to operation 4 for the processing.
7. For the fourth and last execution of operation 4, everything works like in the first one. The only difference is that level 4 data (ADAMS and SMITH) is passed to operation 4 for the processing.
8. Operation 3 gets the rows passed from its children and returns them to its parent operation (1).
9. Operation 1 applies the access predicate “MGR”=PRIOR “EMPNO” and sends the 14 rows to the caller.

The execution plan generated on Oracle Database 10g is slightly different. As can be seen, the operation CONNECT BY WITH FILTERING has a third child (operation 8). In this case, it was not executed, however. The value in the column Starts for operation 8 confirms this. Actually, the third child is executed only when the CONNECT BY operation uses temporary space. When that happens, performance might degrade considerably. This problem, which is fixed as of version 10.2.0.4, is known as bug 5065418.

```sql
---------------------------------------------------------------------
| Id  | Operation                     | Name      | Starts | A-Rows |
---------------------------------------------------------------------
|*  1 |  CONNECT BY WITH FILTERING    |           |      1 |     14 |
|*  2 |   TABLE ACCESS FULL           | EMP       |      1 |      1 |
|   3 |   NESTED LOOPS                |           |      4 |     13 |
|   4 |    BUFFER SORT                |           |      4 |     14 |
|   5 |     CONNECT BY PUMP           |           |      4 |     14 |
|   6 |    TABLE ACCESS BY INDEX ROWID| EMP       |     14 |     13 |
|*  7 |     INDEX RANGE SCAN          | EMP_MGR_I |     14 |     13 |
|   8 |   TABLE ACCESS FULL           | EMP       |      0 |      0 |
---------------------------------------------------------------------
```
