---
title: "所有 GUC 排成一排：default_statistics_target"
source: "https://thebuild.com/blog/all-your-gucs-in-a-row-defaultstatisticstarget/"
saved: "2026-06-12 14:21:52 CST"
published: "2026-06-11T18:00:00-07:00"
---

# 所有 GUC 排成一排：default_statistics_target

![图片 1](../../assets/all-your-gucs-default-statistics-target/fcfb48bb733548e58ca55137b06be995.png)

`default_statistics_target` 是 PostgreSQL 中最常被推荐、但解释最少的参数之一。调优指南会像引用经文一样自信地说“数据仓库把它提高到 500”，却很少说明这个数字到底_是什么_。默认值是 `100`，取值范围是 1 到 10,000，上下文是 `user`，而这个数字真正控制的是 `ANALYZE` 为每一列构建的三类东西的大小。我们来看看这些东西。

## 这个 target 给你买来了什么

当 `ANALYZE` 检查一列时，它会生成一些标量信息，比如 null 比例和估算的 distinct count；除此之外，还会生成两个 planner 经常依赖的结构。它们都存储在 `pg_statistic` 中，并可通过 `pg_stats` 查看。**最常见值列表**（`most_common_vals`，以及 `most_common_freqs`）记录最多 _target_ 个值及其频率——planner 就靠它估算类似 `WHERE state = 'TX'` 这样的等值谓词。**直方图**（`histogram_bounds`）把剩余值划分成最多 _target_ 个等人口桶——planner 就靠它估算类似 `WHERE id BETWEEN 5000 AND 10000` 这样的范围谓词。statistics target 字面上就是每个结构中的最大条目数。在 100 时，planner 可以准确知道最常见的 100 个值，并把其余分布切成 100 个桶。

这个 target 还会设置采样大小，而且公式非常明确：`ANALYZE` 会采样 `300 × target` 行——默认值下是 30,000 行。这个 300 不是随便来的；`analyze.c` 里的注释把它追溯到一篇 1998 年关于随机采样构建直方图的论文，论文推导出的界限值是 305，源码把它向下取整为 300。所以你设置的这个数字承担双重职责：它既决定统计数组的大小，_也_决定为了构建这些统计信息需要读取表中多少数据。

这种双重职责就是整个调优权衡。提高 target，`ANALYZE` 就会读取更多行、耗时更长——每一次 autovacuum 触发的 analyze、每一张表、永远如此。planner 在规划阶段也会多付出一点成本，因为估算一个谓词意味着扫描这些数组，而一个 10,000 项的 MCV 列表会在每个触及该列的查询中被扫描。统计信息在两端都不是免费的。

## 什么时候 100 不够

失败模式有一个明确形状：某一列的分布中，“常见”值数量超过了 MCV 列表能容纳的数量。想象一张十亿行事件表上的 `customer_id`，前一万个客户遵循幂律分布。target 为 100 时，planner 能准确知道前 100 个客户；第 4,000 个客户虽然仍然比平均值高得多，却已经掉出了列表，于是会根据剩余值的平均频率估算，误差可能达到几个数量级。症状是经典的：`EXPLAIN ANALYZE` 显示针对倾斜列的谓词，其估算行数和实际行数相差 100 倍；错误的行数估计随后级联成错误的 join order，或者让本该使用 hash join 的地方用了 nested loop。

修复方法几乎从来不是全局开关，而是按列设置：

```
ALTER TABLE events ALTER COLUMN customer_id SET STATISTICS 1000;
ANALYZE events;
```

这只会提高那个确实需要更高分布分辨率的列的 target，只在那张表上带来额外采样成本，并让你那几百个表现良好的 boolean 和 timestamp 列继续使用便宜的默认值。（记得执行 `ANALYZE`——新 target 在统计信息重建之前不会生效。）还有一个不太知名但值得掌握的变体：表达式索引也有统计信息，`ALTER INDEX ... ALTER COLUMN expr SET STATISTICS n` 可以为一个没有真实列可改的索引表达式调整 target——Alexander Korotkov 记录过用这种方法把 1,140% 的估算误差降到 23%。

提高_全局_默认值只在一种情况下有意义：分析型或数据仓库工作负载，其中大多数查询都是对倾斜数据做大扫描和 join，计划质量压倒一切，而更慢的 `ANALYZE` 相比动辄数小时的查询只是舍入误差。在这种情况下，全局设为 500 或 1,000 是可以辩护且常见的。对于 OLTP，全局默认值 100 确实没问题；真正需要更高 target 的列少到可以逐个点名。

最后有两个提醒。第一，不要条件反射式地设到 10,000——`pg_statistic` 会膨胀，每次 `ANALYZE` 都会变慢，每次查询规划都会在每次估算时付出数组扫描成本；Crunchy Data 的说法是对的，把 target 拉满“会让 ANALYZE 和 vacuum 变慢且昂贵”。第二，要知道什么时候 target 帮不上忙：如果误估来自_相关_列——city 暗示 country，zip 暗示 state——再高的单列直方图分辨率也无法修复 planner 的列独立性假设。这是 `CREATE STATISTICS`（扩展统计信息）要解决的问题，它和本文讨论的工具不同。statistics target 会让每一列的单独画像更清晰；它不能教会 planner 两列之间存在关系。当一个倾斜的单列被误估时，提高 target；当问题来自_组合_时，用扩展统计信息；除非你的工作负载看起来像数据仓库，否则让全局默认值保持不变。

---

来源：https://thebuild.com/blog/all-your-gucs-in-a-row-defaultstatisticstarget/
保存时间：2026-06-12 14:21:52 CST
