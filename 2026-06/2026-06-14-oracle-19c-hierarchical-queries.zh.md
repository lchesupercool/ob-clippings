---
title: "Oracle Database 19c SQL 语言参考：层次查询"
source: "https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html#GUID-E3D35EF7-33C3-4D88-81B3-00030C47AE56"
saved: "2026-06-14 15:41:54 CST"
tags: [oracle, sql, hierarchical-query, connect-by, database]
translation_of: "2026-06-14-oracle-19c-hierarchical-queries.md"
---
## 层次查询

==如果表中包含层次化数据，那么可以使用 hierarchical query clause 按层次顺序选择行：==[^1]

hierarchical_query_clause::=

![hierarchical_query_clause.eps 的说明如下](../../assets/oracle-19c-hierarchical-queries/hierarchical_query_clause.gif)

`condition` 可以是 [Conditions](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Conditions.html#GUID-C2E3ED44-16E7-4924-9125-E1693B1022A8) 中描述的任意条件。

==`START` `WITH` 指定层次结构的根行。==

==`CONNECT` `BY` 指定层次结构中父行与子行之间的关系。==

- `NOCYCLE` 参数指示 Oracle Database 即使数据中存在 `CONNECT` `BY` 循环，也要返回查询中的行。将此参数与 `CONNECT_BY_ISCYCLE` 伪列一起使用，可以查看哪些行包含循环。更多信息请参阅 [CONNECT_BY_ISCYCLE Pseudocolumn](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Pseudocolumns.html#GUID-DA181C6B-7B13-41E3-AAF5-5C19963D9D1C)。
- 在层次查询中，`condition` 中的一个表达式必须用 `PRIOR` 运算符限定，以引用父行。例如：
... PRIOR expr = expr
或
... expr = PRIOR expr
如果 `CONNECT` `BY` `condition` 是复合条件，那么只有一个条件需要 `PRIOR` 运算符，不过你可以有多个 `PRIOR` 条件。例如：
CONNECT BY last_name != 'King' AND PRIOR employee_id = manager_id ...
CONNECT BY PRIOR employee_id = manager_id and 
PRIOR account_mgr_id = customer_id ...
`PRIOR` 是一元运算符，优先级与一元 + 和 - 算术运算符相同。它会在层次查询中针对当前行的父行计算紧跟其后的表达式。
`PRIOR` 最常用于通过等号运算符比较列值。（`PRIOR` 关键字可以位于运算符任意一侧。）`PRIOR` 会让 Oracle 使用父行中该列的值。理论上，`CONNECT` `BY` 子句中也可以使用等号（=）以外的运算符。但是，这些其他运算符创建的条件可能导致在可能组合之间出现无限循环。在这种情况下，Oracle 会在运行时检测到循环并返回错误。

`CONNECT` `BY` 条件和 `PRIOR` 表达式都可以采用非相关子查询的形式。但是，`CURRVAL` 和 `NEXTVAL` 不是有效的 `PRIOR` 表达式，因此 `PRIOR` 表达式不能引用序列。

你还可以在 select list 中使用 `CONNECT_BY_ROOT` 运算符限定某一列，进一步细化层次查询。此运算符扩展了层次查询中 `CONNECT` `BY` [`PRIOR`] 条件的功能：它不仅返回直接父行，还会返回层次结构中的所有祖先行。

另请参阅：

关于此运算符的更多信息，请参阅 [CONNECT_BY_ROOT](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Operators.html#GUID-875C8985-4AEF-4DF1-BA23-3CDF5BCBCD8E) 和“[Hierarchical Query Examples](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html#GUID-E3D35EF7-33C3-4D88-81B3-00030C47AE56)”。

==Oracle 按如下方式处理层次查询：==

- 如果存在 join，则先计算 join；无论该 join 是在 `FROM` 子句中指定，还是通过 `WHERE` 子句谓词指定。
- 然后计算 `CONNECT` `BY` 条件。
- 最后计算所有剩余的 `WHERE` 子句谓词。

随后 Oracle 使用这些计算结果，通过以下步骤形成层次结构：

1. Oracle 选择层次结构的根行——即满足 `START` `WITH` 条件的那些行。
2. ==Oracle 选择每个根行的子行==。每个子行都必须相对于其中一个根行满足 `CONNECT` `BY` 条件。
3. Oracle 选择后续各代子行。Oracle 首先选择步骤 [2](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html#GUID-0118DF1D-B9A9-41EB-8556-C6E7D6A5A84E__I2070828) 返回行的子行，然后选择这些子行的子行，依此类推。Oracle 始终通过相对于当前父行计算 `CONNECT` `BY` 条件来选择子行。
4. 如果查询包含不带 join 的 `WHERE` 子句，那么 Oracle 会从层次结构中剔除所有不满足 `WHERE` 子句条件的行。Oracle 会逐行计算此条件，而不是删除不满足条件的某一行的所有子行。
5. Oracle 按 [Figure 9-1](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html#GUID-0118DF1D-B9A9-41EB-8556-C6E7D6A5A84E__I2066595) 所示顺序返回行。在图中，子节点显示在父节点下方。关于层次树的解释，请参阅 [Figure 3-1](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Pseudocolumns.html#GUID-D91FFF59-ECB0-40F0-AB4C-7A9D27EBEEF1__I1009270)。

Figure 9-1 层次查询

![Figure 9-1 的说明如下](../../assets/oracle-19c-hierarchical-queries/sqlrf002.gif)

==为了查找某个父行的子行，Oracle 会针对父行计算 `CONNECT` `BY` 条件中的 `PRIOR` 表达式，并针对表中的每一行计算另一个表达式。使条件为真的那些行就是该父行的子行==。`CONNECT` `BY` 条件还可以包含其他条件，用于进一步过滤查询选择的行。

如果 `CONNECT` `BY` 条件导致层次结构中出现循环，那么 Oracle 会返回错误。如果某一行既是另一行的父行（或祖父行、直接祖先），又是另一行的子行（或孙行、直接后代），就会发生循环。

注意：

在层次查询中，==不要指定 `ORDER` `BY` 或 `GROUP` `BY`，因为它们会覆盖 `CONNECT` `BY` 结果的层次顺序==。如果想对同一父节点下的兄弟行排序==，请使用 `ORDER` `SIBLINGS` `BY` 子句==。参见 [order_by_clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html#GUID-CFA006CA-6FF1-4972-821E-6996142A51C6__I2171079)。

### 层次查询示例

CONNECT BY 示例

以下层次查询使用 `CONNECT` `BY` 子句定义员工与经理之间的关系：

```sql
SELECT employee_id, last_name, manager_id
   FROM employees
   CONNECT BY PRIOR employee_id = manager_id;

EMPLOYEE_ID LAST_NAME                 MANAGER_ID
----------- ------------------------- ----------
        101 Kochhar                          100
        108 Greenberg                        101
        109 Faviet                           108
        110 Chen                             108
        111 Sciarra                          108
        112 Urman                            108
        113 Popp                             108
        200 Whalen                           101
        203 Mavris                           101
        204 Baer                             101
. . .
```

LEVEL 示例

下一个示例与前一个示例类似，但使用 `LEVEL` 伪列显示父行和子行：

```sql
SELECT employee_id, last_name, manager_id, LEVEL
   FROM employees
   CONNECT BY PRIOR employee_id = manager_id;

EMPLOYEE_ID LAST_NAME                 MANAGER_ID      LEVEL
----------- ------------------------- ---------- ----------
        101 Kochhar                          100          1
        108 Greenberg                        101          2
        109 Faviet                           108          3
        110 Chen                             108          3
        111 Sciarra                          108          3
        112 Urman                            108          3
        113 Popp                             108          3
        200 Whalen                           101          2
        203 Mavris                           101          2
        204 Baer                             101          2
        205 Higgins                          101          2
        206 Gietz                            205          3
        102 De Haan                          100          1
...
```

START WITH 示例

下一个示例添加 `START` `WITH` 子句来指定层次结构的根行，并添加使用 `SIBLINGS` 关键字的 `ORDER` `BY` 子句，以在层次结构内部保留排序：

```sql
SELECT last_name, employee_id, manager_id, LEVEL
      FROM employees
      START WITH employee_id = 100
      CONNECT BY PRIOR employee_id = manager_id
      ORDER SIBLINGS BY last_name;

LAST_NAME                 EMPLOYEE_ID MANAGER_ID      LEVEL
------------------------- ----------- ---------- ----------
King                              100                     1
Cambrault                         148        100          2
Bates                             172        148          3
Bloom                             169        148          3
Fox                               170        148          3
Kumar                             173        148          3
Ozer                              168        148          3
Smith                             171        148          3
De Haan                           102        100          2
Hunold                            103        102          3
Austin                            105        103          4
Ernst                             104        103          4
Lorentz                           107        103          4
Pataballa                         106        103          4
Errazuriz                         147        100          2
Ande                              166        147          3
Banda                             167        147          3
...
```

在 `hr.employees` 表中，员工 Steven King 是公司负责人，没有经理。他的下属中包括 John Russell，后者是部门 80 的经理。如果你更新 `employees` 表，把 Russell 设置为 King 的经理，就会在数据中创建一个循环：

```sql
UPDATE employees SET manager_id = 145
   WHERE employee_id = 100;

SELECT last_name "Employee", 
   LEVEL, SYS_CONNECT_BY_PATH(last_name, '/') "Path"
   FROM employees
   WHERE level <= 3 AND department_id = 80
   START WITH last_name = 'King'
   CONNECT BY PRIOR employee_id = manager_id AND LEVEL <= 4;

ERROR:
ORA-01436: CONNECT BY loop in user data
```

`CONNECT` `BY` 条件中的 `NOCYCLE` 参数会让 Oracle 即使存在循环也返回这些行。`CONNECT_BY_ISCYCLE` 伪列会显示哪些行包含循环：

```sql
SELECT last_name "Employee", CONNECT_BY_ISCYCLE "Cycle",
   LEVEL, SYS_CONNECT_BY_PATH(last_name, '/') "Path"
   FROM employees
   WHERE level <= 3 AND department_id = 80
   START WITH last_name = 'King'
   CONNECT BY NOCYCLE PRIOR employee_id = manager_id AND LEVEL <= 4
   ORDER BY "Employee", "Cycle", LEVEL, "Path";

Employee                       Cycle      LEVEL Path
------------------------- ---------- ---------- -------------------------
Abel                               0          3 /King/Zlotkey/Abel
Ande                               0          3 /King/Errazuriz/Ande
Banda                              0          3 /King/Errazuriz/Banda
Bates                              0          3 /King/Cambrault/Bates
Bernstein                          0          3 /King/Russell/Bernstein
Bloom                              0          3 /King/Cambrault/Bloom
Cambrault                          0          2 /King/Cambrault
Cambrault                          0          3 /King/Russell/Cambrault
Doran                              0          3 /King/Partners/Doran
Errazuriz                          0          2 /King/Errazuriz
Fox                                0          3 /King/Cambrault/Fox
...
```

CONNECT_BY_ISLEAF 示例

以下语句展示了如何使用层次查询把某一列中的值转换成逗号分隔列表：

```sql
SELECT LTRIM(SYS_CONNECT_BY_PATH (warehouse_id,','),',') FROM
   (SELECT ROWNUM r, warehouse_id FROM warehouses)
   WHERE CONNECT_BY_ISLEAF = 1
   START WITH r = 1
   CONNECT BY r = PRIOR r + 1
   ORDER BY warehouse_id; 
 
LTRIM(SYS_CONNECT_BY_PATH(WAREHOUSE_ID,','),',')
--------------------------------------------------------------------------------
1,2,3,4,5,6,7,8,9
```

CONNECT_BY_ROOT 示例

以下示例返回部门 110 中每个员工的姓氏、该员工在层次结构上方最高层级的每个经理、经理与员工之间的层级数，以及两者之间的路径：

```sql
SELECT last_name "Employee", CONNECT_BY_ROOT last_name "Manager",
   LEVEL-1 "Pathlen", SYS_CONNECT_BY_PATH(last_name, '/') "Path"
   FROM employees
   WHERE LEVEL > 1 and department_id = 110
   CONNECT BY PRIOR employee_id = manager_id
   ORDER BY "Employee", "Manager", "Pathlen", "Path";

Employee        Manager            Pathlen Path
--------------- --------------- ---------- ------------------------------
Gietz           Higgins                  1 /Higgins/Gietz
Gietz           King                     3 /King/Kochhar/Higgins/Gietz
Gietz           Kochhar                  2 /Kochhar/Higgins/Gietz
Higgins         King                     2 /King/Kochhar/Higgins
Higgins         Kochhar                  1 /Kochhar/Higgins
```

以下示例使用 `GROUP` `BY` 子句返回部门 110 中每个员工以及层次结构中位于该员工上方的所有员工的总薪资：

```sql
SELECT name, SUM(salary) "Total_Salary" FROM (
   SELECT CONNECT_BY_ROOT last_name as name, Salary
      FROM employees
      WHERE department_id = 110
      CONNECT BY PRIOR employee_id = manager_id)
      GROUP BY name
   ORDER BY name, "Total_Salary";

NAME                      Total_Salary
------------------------- ------------
Gietz                             8300
Higgins                          20300
King                             20300
Kochhar                          20300
```

另请参阅：

- [LEVEL Pseudocolumn](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Pseudocolumns.html#GUID-D91FFF59-ECB0-40F0-AB4C-7A9D27EBEEF1) 和 [CONNECT_BY_ISCYCLE Pseudocolumn](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Pseudocolumns.html#GUID-DA181C6B-7B13-41E3-AAF5-5C19963D9D1C)，用于了解这些伪列在层次查询中的工作方式。
- [SYS_CONNECT_BY_PATH](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SYS_CONNECT_BY_PATH.html#GUID-D25A0F86-B559-4090-9164-7A2C84D1E11E)，用于了解如何检索从根节点到节点的列值路径。
- [order_by_clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html#GUID-CFA006CA-6FF1-4972-821E-6996142A51C6__I2171079)，用于了解 `ORDER` `BY` 子句中的 `SIBLINGS` 关键字。
- [subquery_factoring_clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html#GUID-CFA006CA-6FF1-4972-821E-6996142A51C6__I2077142)，它支持 recursive subquery factoring（recursive WITH），并允许查询层次数据。此功能比 `CONNECT` `BY` 更强大，因为它提供深度优先搜索和广度优先搜索，并支持多个递归分支。

[^1]: Connect By 需要表中数据是有层次化的, 在层次数据之上做层次化查询.
