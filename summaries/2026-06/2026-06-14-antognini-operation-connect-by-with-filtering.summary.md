---
source: "https://antognini.ch/2008/06/operation-connect-by-with-filtering/"
saved: "2026-06-14 15:47:45 CST"
summary_of: "../../2026-06/2026-06-14-antognini-operation-connect-by-with-filtering.md"
---

# Antognini：Operation CONNECT BY WITH FILTERING 摘要

## 一句话结论

Oracle 的 `CONNECT BY WITH FILTERING` 执行层次查询时，先通过一个子操作找根节点，再按层级反复执行另一个子操作扩展子节点；执行计划里看似“神秘”的全表扫描通常是用于定位根节点，而不是每一层递归都做全表扫描。

## 文章主旨

文章解释 Oracle 层次查询 `CONNECT BY` 的执行计划如何阅读，特别是 `CONNECT BY WITH FILTERING` 操作下各个子节点的角色。Antognini 用经典 `EMP` 表的上下级关系示例说明：

- `START WITH mgr IS NULL` 用来找根节点 `KING`；
- `CONNECT BY PRIOR empno = mgr` 用来逐层查找下属；
- 执行计划中的 `TABLE ACCESS FULL EMP` 对应根节点查找；
- 后续层级扩展通过 `CONNECT BY PUMP`、`NESTED LOOPS`、索引 `EMP_MGR_I` 以及按 rowid 回表完成；
- `Starts` 与 `A-Rows` 能帮助判断每个操作被执行的次数以及实际返回的行数。

## 执行计划机制

在 Oracle Database 11g 的示例计划中，`CONNECT BY WITH FILTERING` 有两个子操作：

1. 第一个子操作是 `TABLE ACCESS FULL EMP`：
   - 执行一次；
   - 应用过滤条件 `MGR IS NULL`；
   - 返回层次结构根节点 `KING`。
2. 第二个子操作是 `NESTED LOOPS`：
   - 按层级执行，本例共执行四次；
   - 通过 `CONNECT BY PUMP` 接收上一层节点；
   - 对上一层每个节点，用 `INDEX RANGE SCAN EMP_MGR_I` 查找 `mgr = prior empno` 的子节点；
   - 再通过 `TABLE ACCESS BY INDEX ROWID` 取出完整行。

层级推进过程可以理解为：

- 第 1 层：根节点 `KING`；
- 第 2 层：`JONES`、`BLAKE`、`CLARK`；
- 第 3 层：`SCOTT`、`FORD`、`ALLEN`、`WARD`、`MARTIN`、`TURNER`、`JAMES`、`MILLER`；
- 第 4 层：`ADAMS`、`SMITH`。

最终，`CONNECT BY WITH FILTERING` 应用访问谓词 `MGR = PRIOR EMPNO`，向调用方返回 14 行。

## 关键观察

- `CONNECT BY WITH FILTERING` 不是普通单层 join，而是一个专门处理层次查询的执行操作。
- 它的第一个子操作负责确定根节点，因此即使后续层级访问使用索引，计划中仍可能出现 `TABLE ACCESS FULL`。
- `CONNECT BY PUMP` 可以看作把当前层或上一层的行“泵入”下一轮层级扩展的机制。
- `NESTED LOOPS` 的执行次数与层级深度相关；本例有四层，所以 `Starts = 4`。
- 操作 4 的 `A-Rows` 与操作 5、6 的 `Starts` 匹配，说明对 `CONNECT BY PUMP` 输出的每一行都会触发一次索引查找与回表。
- 在 Oracle 10g 的计划中，`CONNECT BY WITH FILTERING` 可能出现第三个子操作；示例中的第三个子操作 `TABLE ACCESS FULL EMP` 没有执行，因为 `Starts = 0`。
- 这个第三个子操作只有在 `CONNECT BY` 使用临时空间时才会执行；一旦执行，性能可能显著下降。
- 相关性能问题在 10.2.0.4 起修复，Antognini 指出它对应 bug 5065418。

## 对 CONNECT BY 优化的启发

- 阅读 `CONNECT BY` 执行计划时，不应只看到 `TABLE ACCESS FULL` 就判断整条层次查询低效；要先确认该全表扫描是否只是用于 `START WITH` 根节点定位。
- 对递归扩展阶段，更关键的是 `CONNECT BY` 条件列是否有合适索引。例如本例中 `mgr` 上的 `EMP_MGR_I` 使每个父节点查找子节点时可以走 `INDEX RANGE SCAN`。
- `Starts` 是诊断层次查询性能的重要指标：它揭示某个节点被反复调用了多少次，尤其能帮助识别递归层级扩展中的重复访问成本。
- 如果在旧版本 Oracle 上看到 `CONNECT BY WITH FILTERING` 的第三个子操作实际执行，并伴随临时空间使用，应警惕 bug 或执行计划退化导致的性能问题。
- 对深层或高扇出层次结构，成本可能来自“每个父节点一次索引探查/回表”的累积，而不一定来自根节点查找的全表扫描。

## 我的理解

这篇文章的价值在于把 `CONNECT BY WITH FILTERING` 拆成了两个阶段来看：先找根，再逐层扩展。执行计划中的缩进、`Starts`、`A-Rows` 和谓词信息合在一起，才能还原真实执行流程。所谓“神秘全表扫描”并不神秘，它对应 `START WITH mgr IS NULL`，只是优化器为了找根节点选择了全表扫描。真正需要关注的是递归扩展阶段是否可索引化、层级深度和节点数量会导致多少次循环访问，以及旧版本中是否触发了临时空间相关的第三子操作。
