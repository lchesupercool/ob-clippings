---
title: "Oracle Database 19c SQL Language Reference: Hierarchical Queries"
source: "https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html#GUID-E3D35EF7-33C3-4D88-81B3-00030C47AE56"
saved: "2026-06-14 15:41:54 CST"
tags: [oracle, sql, hierarchical-query, connect-by, database]
---

## Hierarchical Queries

If a table contains hierarchical data, then you can select rows in a hierarchical order using the hierarchical query clause:

hierarchical_query_clause::=

![Description of hierarchical_query_clause.eps follows](../../assets/oracle-19c-hierarchical-queries/hierarchical_query_clause.gif)

`condition` can be any condition as described in [Conditions](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Conditions.html#GUID-C2E3ED44-16E7-4924-9125-E1693B1022A8).

`START` `WITH` specifies the root row(s) of the hierarchy.

`CONNECT` `BY` specifies the relationship between parent rows and child rows of the hierarchy.

- The `NOCYCLE` parameter instructs Oracle Database to return rows from a query even if a `CONNECT` `BY` loop exists in the data. Use this parameter along with the `CONNECT_BY_ISCYCLE` pseudocolumn to see which rows contain the loop. Refer to [CONNECT_BY_ISCYCLE Pseudocolumn](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Pseudocolumns.html#GUID-DA181C6B-7B13-41E3-AAF5-5C19963D9D1C) for more information.
- In a hierarchical query, one expression in `condition` must be qualified with the `PRIOR` operator to refer to the parent row. For example,
... PRIOR expr = expr
or
... expr = PRIOR expr
If the `CONNECT` `BY` `condition` is compound, then only one condition requires the `PRIOR` operator, although you can have multiple `PRIOR` conditions. For example: 
CONNECT BY last_name != 'King' AND PRIOR employee_id = manager_id ...
CONNECT BY PRIOR employee_id = manager_id and 
PRIOR account_mgr_id = customer_id ...
`PRIOR` is a unary operator and has the same precedence as the unary + and - arithmetic operators. It evaluates the immediately following expression for the parent row of the current row in a hierarchical query. 
`PRIOR` is most commonly used when comparing column values with the equality operator. (The `PRIOR` keyword can be on either side of the operator.) `PRIOR` causes Oracle to use the value of the parent row in the column. Operators other than the equal sign (=) are theoretically possible in `CONNECT` `BY` clauses. However, the conditions created by these other operators can result in an infinite loop through the possible combinations. In this case Oracle detects the loop at run time and returns an error.

Both the `CONNECT` `BY` condition and the `PRIOR` expression can take the form of an uncorrelated subquery. However, `CURRVAL` and `NEXTVAL` are not valid `PRIOR` expressions, so the `PRIOR` expression cannot refer to a sequence.

You can further refine a hierarchical query by using the `CONNECT_BY_ROOT` operator to qualify a column in the select list. This operator extends the functionality of the `CONNECT` `BY` [`PRIOR`] condition of hierarchical queries by returning not only the immediate parent row but all ancestor rows in the hierarchy.

See Also:

[CONNECT_BY_ROOT](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Operators.html#GUID-875C8985-4AEF-4DF1-BA23-3CDF5BCBCD8E) for more information about this operator and "[Hierarchical Query Examples](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html#GUID-E3D35EF7-33C3-4D88-81B3-00030C47AE56)"

Oracle processes hierarchical queries as follows:

- A join, if present, is evaluated first, whether the join is specified in the `FROM` clause or with `WHERE` clause predicates.
- The `CONNECT` `BY` condition is evaluated.
- Any remaining `WHERE` clause predicates are evaluated.

Oracle then uses the information from these evaluations to form the hierarchy using the following steps:

1. Oracle selects the root row(s) of the hierarchy—those rows that satisfy the `START` `WITH` condition.
2. Oracle selects the child rows of each root row. Each child row must satisfy the condition of the `CONNECT` `BY` condition with respect to one of the root rows.
3. Oracle selects successive generations of child rows. Oracle first selects the children of the rows returned in step [2](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html#GUID-0118DF1D-B9A9-41EB-8556-C6E7D6A5A84E__I2070828), and then the children of those children, and so on. Oracle always selects children by evaluating the `CONNECT` `BY` condition with respect to a current parent row.
4. If the query contains a `WHERE` clause without a join, then Oracle eliminates all rows from the hierarchy that do not satisfy the condition of the `WHERE` clause. Oracle evaluates this condition for each row individually, rather than removing all the children of a row that does not satisfy the condition.
5. Oracle returns the rows in the order shown in [Figure 9-1](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html#GUID-0118DF1D-B9A9-41EB-8556-C6E7D6A5A84E__I2066595). In the diagram, children appear below their parents. For an explanation of hierarchical trees, see [Figure 3-1](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Pseudocolumns.html#GUID-D91FFF59-ECB0-40F0-AB4C-7A9D27EBEEF1__I1009270).

Figure 9-1 Hierarchical Queries

![Description of Figure 9-1 follows](../../assets/oracle-19c-hierarchical-queries/sqlrf002.gif)

To find the children of a parent row, Oracle evaluates the `PRIOR` expression of the `CONNECT` `BY` condition for the parent row and the other expression for each row in the table. Rows for which the condition is true are the children of the parent. The `CONNECT` `BY` condition can contain other conditions to further filter the rows selected by the query.

If the `CONNECT` `BY` condition results in a loop in the hierarchy, then Oracle returns an error. A loop occurs if one row is both the parent (or grandparent or direct ancestor) and a child (or a grandchild or a direct descendent) of another row.

Note:

In a hierarchical query, do not specify either `ORDER` `BY` or `GROUP` `BY`, as they will override the hierarchical order of the `CONNECT` `BY` results. If you want to order rows of siblings of the same parent, then use the `ORDER` `SIBLINGS` `BY` clause. See [order_by_clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html#GUID-CFA006CA-6FF1-4972-821E-6996142A51C6__I2171079).

### Hierarchical Query Examples

CONNECT BY Example

The following hierarchical query uses the `CONNECT` `BY` clause to define the relationship between employees and managers:

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

LEVEL Example

The next example is similar to the preceding example, but uses the `LEVEL` pseudocolumn to show parent and child rows:

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

START WITH Examples

The next example adds a `START` `WITH` clause to specify a root row for the hierarchy and an `ORDER` `BY` clause using the `SIBLINGS` keyword to preserve ordering within the hierarchy:

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

In the `hr.employees` table, the employee Steven King is the head of the company and has no manager. Among his employees is John Russell, who is the manager of department 80. If you update the `employees` table to set Russell as King's manager, you create a loop in the data:

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

The `NOCYCLE` parameter in the `CONNECT` `BY` condition causes Oracle to return the rows in spite of the loop. The `CONNECT_BY_ISCYCLE` pseudocolumn shows you which rows contain the cycle:

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

CONNECT_BY_ISLEAF Example

The following statement shows how you can use a hierarchical query to turn the values in a column into a comma-delimited list:

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

CONNECT_BY_ROOT Examples

The following example returns the last name of each employee in department 110, each manager at the highest level above that employee in the hierarchy, the number of levels between manager and employee, and the path between the two:

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

The following example uses a `GROUP` `BY` clause to return the total salary of each employee in department 110 and all employees above that employee in the hierarchy:

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

See Also:

- [LEVEL Pseudocolumn](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Pseudocolumns.html#GUID-D91FFF59-ECB0-40F0-AB4C-7A9D27EBEEF1) and [CONNECT_BY_ISCYCLE Pseudocolumn](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Query-Pseudocolumns.html#GUID-DA181C6B-7B13-41E3-AAF5-5C19963D9D1C) for a discussion of how these pseudocolumns operate in a hierarchical query
- [SYS_CONNECT_BY_PATH](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SYS_CONNECT_BY_PATH.html#GUID-D25A0F86-B559-4090-9164-7A2C84D1E11E) for information on retrieving the path of column values from root to node
- [order_by_clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html#GUID-CFA006CA-6FF1-4972-821E-6996142A51C6__I2171079) for more information on the `SIBLINGS` keyword of `ORDER` `BY` clauses
- [subquery_factoring_clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html#GUID-CFA006CA-6FF1-4972-821E-6996142A51C6__I2077142), which supports recursive subquery factoring (recursive WITH) and lets you query hierarchical data. This feature is more powerful than `CONNECT` `BY` in that it provides depth-first search and breadth-first search, and supports multiple recursive branches.
