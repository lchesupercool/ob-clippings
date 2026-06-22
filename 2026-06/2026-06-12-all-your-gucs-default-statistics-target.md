---
title: "All Your GUCs in a Row: default_statistics_target"
source: "https://thebuild.com/blog/all-your-gucs-in-a-row-defaultstatisticstarget/"
saved: "2026-06-12 14:21:52 CST"
published: "2026-06-11T18:00:00-07:00"
---

# All Your GUCs in a Row: default_statistics_target

![Image 1](../../assets/all-your-gucs-default-statistics-target/fcfb48bb733548e58ca55137b06be995.png)

`default_statistics_target` is one of the most-recommended and least-explained parameters in PostgreSQL. Tuning guides say “raise it to 500 for data warehouses” with the confidence of scripture and rarely a word about what the number _is_. The default is `100`, the range is 1 to 10,000, the context is `user`, and what the number actually controls is the size of three things `ANALYZE` builds for every column. Let’s look at the things.

## What the target buys you

When `ANALYZE` examines a column, it produces (among a few scalars like null fraction and estimated distinct count) two structures the planner leans on constantly, both stored in `pg_statistic` and visible through `pg_stats`. The **most-common-values list** (`most_common_vals`, with `most_common_freqs`) records up to _target_ values and their frequencies — this is how the planner estimates equality predicates like `WHERE state = 'TX'`. The **histogram** (`histogram_bounds`) divides the remaining values into up to _target_ equal-population buckets — this is how it estimates range predicates like `WHERE id BETWEEN 5000 AND 10000`. The statistics target is, literally, the maximum number of entries in each. At 100, the planner can know your hundred most common values exactly and slice the rest of the distribution into a hundred buckets.

The target also sets the sample size, and the formula is pleasingly specific: `ANALYZE` samples `300 × target` rows — 30,000 at the default. The 300 isn’t arbitrary; the comment in `analyze.c` traces it to a 1998 paper on random sampling for histogram construction, which derives a bound the source rounds from 305 down to 300. So the number you set is doing double duty: it sizes the statistics arrays _and_ scales how much of the table gets read to build them.

That double duty is the entire tuning tradeoff. Raise the target and `ANALYZE` reads more rows and takes longer — on every autovacuum-triggered analyze, on every table, forever. The planner also pays a little more at plan time, because estimating a predicate means scanning those arrays, and a 10,000-entry MCV list is scanned at every query that touches the column. Statistics aren’t free at either end.

## When 100 isn’t enough

The failure mode has a specific shape: a column whose distribution has more “common” values than the MCV list can hold. Picture a `customer_id` on a billion-row events table where the top ten thousand customers follow a power law. With a target of 100, the planner knows the top hundred exactly; customer number 4,000 — still wildly more frequent than average — falls off the list and gets estimated from the leftover average, which can be off by orders of magnitude. The symptom is the classic one: `EXPLAIN ANALYZE` shows estimated rows and actual rows diverging by 100× on a predicate against a skewed column, and the bad row estimate cascades into a bad join order or a nested loop that should have been a hash join.

The fix is almost never the global knob. It’s per-column:

```
ALTER TABLE events ALTER COLUMN customer_id SET STATISTICS 1000;
ANALYZE events;
```

That raises the target for the one column whose distribution needs it, costs extra sampling only on that table, and leaves your hundreds of well-behaved boolean and timestamp columns at the cheap default. (Remember the `ANALYZE` — the new target does nothing until statistics are rebuilt.) There’s a less-known variant worth having in your kit: expression indexes get statistics too, and `ALTER INDEX ... ALTER COLUMN expr SET STATISTICS n` tunes the target for an indexed expression that has no real column to alter — Alexander Korotkov documented turning a 1,140% estimation error into 23% that way.

Raising the _global_ default makes sense in one situation: an analytics or warehouse workload where most queries are large scans and joins over skewed data, plan quality is everything, and a slower `ANALYZE` is a rounding error against hour-long queries. There, 500 or 1,000 globally is defensible and common. For OLTP, the global default of 100 is genuinely fine, and the columns that need more are few enough to name individually.

Two closing cautions. First, don’t reach for 10,000 reflexively — `pg_statistic` bloats, every `ANALYZE` slows, and per-query planning pays the array-scan tax on every estimate; Crunchy Data’s phrasing is right that maxing the target “would make ANALYZE and vacuum slow and expensive.” Second, know when the target can’t help: if your misestimates come from _correlated_ columns — city implies country, zip implies state — no amount of per-column histogram resolution fixes the planner’s column-independence assumption. That’s what `CREATE STATISTICS` (extended statistics) is for, and it’s a different tool than the one this post is about. The statistics target makes each column’s solo portrait sharper; it cannot teach the planner that two columns are related. Raise the target when a skewed single column is being mis-estimated, reach for extended statistics when the _combination_ is the problem, and leave the global default alone until your workload looks like a warehouse.

---

Source: https://thebuild.com/blog/all-your-gucs-in-a-row-defaultstatisticstarget/
Saved: 2026-06-12 14:21:52 CST
