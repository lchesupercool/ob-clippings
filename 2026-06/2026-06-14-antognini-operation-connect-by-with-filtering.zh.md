---
title: "操作 CONNECT BY WITH FILTERING"
source: "https://antognini.ch/2008/06/operation-connect-by-with-filtering/"
saved: "2026-06-14 15:47:45 CST"
tags: [oracle, optimizer, connect-by, hierarchical-query, execution-plan]
---

# 操作 CONNECT BY WITH FILTERING

![](../../assets/antognini-operation-connect-by-with-filtering/fe3509bf34a739bb7054e47e8723b99c52b00933e3b31aa3d3edbd7c09bb0dd7.png)

我曾多次被问到 CONNECT BY 操作中那个神秘的全表扫描。在这篇文章中，我想与你分享一些我在我的[书](https://antognini.ch/top)（第 233 到 236 页）中写到的相关信息。

操作 CONNECT BY WITH FILTERING 用于处理层次查询。它的特征是有两个子操作。==第一个子操作用于获取层次结构的根节点，第二个子操作会针对层次结构中的每一层执行一次。==

下面是一个示例查询及其执行计划。请注意，该执行计划是在 Oracle Database 11g 上生成的（原因稍后解释）。

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

为了帮助你更容易理解带有层次查询的执行计划，同时查看查询返回的数据也很有用：

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

执行计划按如下方式执行这些操作：

1. 操作 1 有两个子操作（2 和 3），而操作 2 是按升序排列时的第一个。因此，执行从操作 2 开始。
2. 操作 2 扫描表 EMP，应用过滤谓词 “MGR” IS NULL，并把层次结构的根节点返回给其父操作（1）。
3. 操作 3 是操作 1 的第二个子操作。因此，它会针对层次结构的每一层执行一次——在这个例子中是四次。先执行第一个子操作，即操作 4；对于它返回的每一行，内层循环（由操作 5 及其子操作 6 组成）执行一次。注意，正如预期，操作 4 的 A-Rows 列与操作 5 和 6 的 Starts 列相匹配。
4. 在第一次执行时，操作 4 通过操作 CONNECT BY PUMP 获取层次结构的根节点。在这个例子中，第 1 层只有一行（KING）。使用 mgr 列中的值，操作 6 通过应用访问谓词 “MGR”=PRIOR “EMPNO” 扫描索引 EMP_MGR_I，提取 rowid，并将它们返回给其父操作（5）。操作 5 使用这些 rowid 访问表 EMP，并把行返回给其父操作（3）。
5. 对于操作 4 的第二次执行，一切都与第一次执行相同。唯一的区别是，第 2 层的数据（JONES、BLAKE 和 CLARK）被传递给操作 4 进行处理。
6. 对于操作 4 的第三次执行，一切都像第一次一样。唯一的区别是，第 3 层数据（SCOTT、FORD、ALLEN、WARD、MARTIN、TURNER、JAMES 和 MILLER）被传递给操作 4 进行处理。
7. 对于操作 4 的第四次也是最后一次执行，一切都像第一次一样。唯一的区别是，第 4 层数据（ADAMS 和 SMITH）被传递给操作 4 进行处理。
8. 操作 3 获取从其子操作传递来的行，并将它们返回给其父操作（1）。
9. 操作 1 应用访问谓词 “MGR”=PRIOR “EMPNO”，并将 14 行发送给调用方。

在 Oracle Database 10g 上生成的执行计划略有不同。可以看到，操作 CONNECT BY WITH FILTERING 有第三个子操作（操作 8）。不过在这个例子中，它没有被执行。操作 8 的 Starts 列中的值证实了这一点。实际上，只有当 CONNECT BY 操作使用临时空间时，第三个子操作才会执行。一旦发生这种情况，性能可能会显著下降。这个问题从 10.2.0.4 版本起已修复，被称为 bug 5065418。

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
