---
title: 'pg_stat_statements: everything it tells you | boringSQL'
source: https://boringsql.com/posts/pg-stat-statements/
author: Radim Marek
published: ''
saved: 2026-06-16 14:35 +0800
tags:
- postgres
- pg_stat_statements
- monitoring
- performance
---

# pg_stat_statements: everything it tells you | boringSQL

Source: [https://boringsql.com/posts/pg-stat-statements/](https://boringsql.com/posts/pg-stat-statements/)  
Author: Radim Marek  
Published:   
Saved: 2026-06-16 14:35 +0800

If not first, `pg_stat_statements` is one of the most used extensions in the PostgreSQL ecosystem. It ships in **contrib** and costs almost nothing to use. Most of us turn to it to answer the question: *what is the database actually doing?* It's genuinely useful. You can use it to get a snapshot of what happened in a given timeframe, and make a faster decision about what to fix.

Coming from other database engines, you might reach for it expecting something a bit more, a query store. The built-in feature that keeps normalized queries and their plan history. Except `pg_stat_statements` is not this. This article is going to deep dive into what the extension really provides.

Because once you lean on it, you might start noticing the rough edges. The gap comes from what "a query store" might come with and what it actually tells you.

The very same query might show up in many separate rows. Discrepancies between what your monitoring says and what `mean_exec_time` shows. Missing queries. Numbers that changed overnight.

None of that is a bug. It all follows from what `pg_stat_statements` really is: a fixed-size hash table of running counters, kept in shared memory, keyed by a hash of your parse tree. It counts; it does not record. Hold that one idea in your head and every surprising thing on the list above stops being surprising.

Everything below is reproducible. Paste this into a scratch database:

```
CREATE TABLE customers (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL
);

CREATE TABLE orders (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id integer NOT NULL REFERENCES customers(id),
    amount numeric(10,2) NOT NULL,
    status text NOT NULL DEFAULT 'pending',
    created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO customers (name)
SELECT 'Customer ' || i FROM generate_series(1, 2000) AS i;

INSERT INTO orders (customer_id, amount, status, created_at)
SELECT (random() * 1999 + 1)::int,
       (random() * 500 + 5)::numeric(10,2),
       (ARRAY['pending','shipped','delivered','cancelled'])[floor(random()*4+1)::int],
       '2024-01-01'::date + (random() * 700)::int
FROM generate_series(1, 100000);

ANALYZE;
```

And turn the extension on:

```
-- in postgresql.conf config
-- shared_preload_libraries = 'pg_stat_statements'  (requires restart)
CREATE EXTENSION pg_stat_statements;
```

## What it actually stores

We've just turned it on, so start with what it puts in front of you.

```
SELECT * FROM pg_stat_statements;
```

You get one row per distinct query *shape* and a set of counters behind it: `calls`, `total_exec_time`, `rows`, `shared_blks_hit`, `shared_blks_read`, and a lot more.

While there is a `query` column, you might think it keeps your SQL. But it does not. If you look carefully query text is normalized, with the literals stripped out (`WHERE id = 42` is stored as `WHERE id = $1`), and there is exactly one row per shape no matter how many times it ran. The counters only add up. A statement runs, the counters for its shape tick up, and that individual execution, its values, its timing, its plan, is gone. Each row is an **entry**: a running total per shape, not a record of executions. Most of the limitations in this article follow from that.

First, where the data lives. The counters sit in shared memory allocated at startup, and as we said, it's a fixed hash map. The number of tracked statements is capped (`pg_stat_statements.max`, default **5000**). The query text lives in a file on disk, `$PGDATA/pg_stat_tmp/pgss_query_texts.stat`, which the view joins against the in-memory counters when you read it.

Each entry in that map is keyed by `queryid`, and that's where the trouble starts.

## The queryid

The jumble moved into core in PostgreSQL 14; before that it lived inside `pg_stat_statements` itself. It's gated by the `compute_query_id` setting, whose default `auto` turns it on whenever a module that needs it, such as this one, is loaded.

The `queryid` is computed by Postgres core, not by the extension. When Postgres parses and analyzes a statement, it walks the resulting tree and hashes the parts that define the query's structure into single 64-bit number, leaving the constant values out. The Postgres source calls this step the *query jumble*; `pg_stat_statements` just reads the result and uses it as the key for each row.

Leaving the constants out is the whole trick. `WHERE id = 42` and `WHERE id = 99` jumble to the same number, with the literal shown as a `$n` placeholder, so one entry can stand in for millions of executions that differ only in their values.

```
SELECT * FROM orders WHERE id = 42;
SELECT * FROM orders WHERE id = 99;
SELECT * FROM orders WHERE id = 12345;

SELECT queryid, query, calls
FROM pg_stat_statements
WHERE query LIKE 'SELECT * FROM orders WHERE id%';
```

```
       queryid        |               query                | calls
----------------------+------------------------------------+-------
 -8198406948996196422 | SELECT * FROM orders WHERE id = $1 |     3
```

Three executions, one row. Now look at the queryid: it's negative. It's a signed 64-bit integer, which can easily break a dashboard that stored it in an unsigned column or printed it as a friendly identifier. It's a hash, not a sequence number. Treat it as an opaque token that might be negative and move on.

This is also where the surprises start, because the jumble is more sensitive than "it ignores constants" makes it sound.

### The queryid isn't stable across versions

The manual is blunt about it:

> it is not safe to assume that `queryid` will be stable across major versions of PostgreSQL.

> The hashing process is also sensitive to differences in machine architecture and other facets of the platform.

If you followed the previous section carefully, you'll have spotted that the hash comes from the shape of the parsed tree, not the query text. Upgrade from Postgres 16 to 17, or move to a different CPU architecture, and identical SQL can hash to a different queryid.

Anything that adds up query costs across a fleet by joining on queryid quietly falls apart the moment the fleet isn't uniform. The docs call out replicas specifically:

> logical replication schemes do not promise to keep replicas identical in all relevant details, so `queryid` will not be a useful identifier for accumulating costs across a set of logical replicas.

**Don't treat `queryid` as a durable key.**  
Within one server it's stable across minor versions, and that is the only guarantee. Across a major upgrade, a different CPU architecture, or the replicas in a logical-replication set, the same SQL can hash to a different value. Any dashboard or rollup that joins on `queryid` across machines is one upgrade away from quietly double-counting or dropping rows.

### The queryid tracks the OID, not the name

The jumble uses the **OIDs** of objects a query touches, not their names. Two behaviors fall out of that, both straight from the manual, and both catch people off guard:

> `pg_stat_statements` will consider two apparently-identical queries to be distinct, if they reference for example a function that was dropped and recreated between the executions of the two queries.

> Conversely, if a table is dropped and recreated between the executions of queries, two apparently-identical queries may be considered the same.

Drop and recreate a table, which happens all the time during migrations, dump-and-restore, or a blue-green cutover, and every query against it gets a fresh queryid, because the table now has a fresh OID. Your old stats for that table are stranded under the old id, looking like queries that simply stopped running. The dead query on your dashboard isn't dead; its OID moved.

`search_path` hides the same trap. `SELECT * FROM orders` resolves to `tenant_a.orders` on one connection and `tenant_b.orders` on another, so a schema-per-tenant app scatters one logical query across every tenant's OID. Correct behaviour, but rolling up "all the reads against `orders`" means resolving OIDs yourself.

### Aliases and structure leak in too

> if the alias for a table is different for otherwise-similar queries, these queries will be considered distinct.

That last one, alias sensitivity, is what hurts ORM users most.

## The ORM row explosion

The jumble normalizes constants. It does not normalize *structure*. Two queries you'd read as "the same query" but that differ in the parsed tree, say a different column list, a table alias, an extra `LIMIT`, or a reordered `AND`, are different shapes with different queryids in different rows.

ORMs generate these variations constantly. Here are four queries an ActiveRecord, Hibernate, or Ecto app might emit for what is really one operation, "load a customer by id":

```
SELECT id, name FROM customers WHERE id = $1;             -- queryid A
SELECT id, name, created_at FROM customers WHERE id = $1; -- queryid B  (.select added a column)
SELECT c.id, c.name FROM customers c WHERE c.id = $1;     -- queryid C  (aliased)
SELECT id, name FROM customers WHERE id = $1 LIMIT $2;    -- queryid D  (.first)
```

Four rows. The calls split four ways, and each one looks minor even when the real operation behind them might be your single hottest path. The `mean_exec_time` is per-row, so a slowdown that hits all four equally gets diluted across all four.

ORMs don't stop at four. Every conditional `.where`, every optional `.select`, every eager-load is another shape; one logical query spreads across dozens or hundreds of entries. You can see real production snapshots where a single logical query lands under **hundreds** of distinct queryids, each looking minor on its own.

This is the job of a tool like [qshape](https://github.com/boringsql/qshape), part of the boringSQL stack.

The extension can't fix this, and it isn't supposed to. The jumble runs before anything understands that "these read different columns from the same table for the same reason." Clustering the variants back together means re-parsing each entry's text, canonicalizing the tree (strip aliases, sort `AND` predicates, spot column-set supersets), and re-fingerprinting. `pg_stat_statements` hands you the raw, fragmented shapes; reassembling them into logical queries is a layer it doesn't have.

A downstream re-parse can work because normalization only looks like it threw the information away. The jumble strips one thing, the constant values (`= 42` becomes `= $1`), and leaves all the structure in the `query` text: aliases, column list, the `AND`/`OR` tree, `LIMIT`, join order. That structure is everything a canonicalizer needs to see that `WHERE a = $1 AND b = $2` and `WHERE b = $1 AND a = $2` are the same predicate, or that `c.id` and `id` are the same column. The extension over-normalizes constants and under-normalizes structure.

One thing genuinely is gone, though: anything that was **evicted**. A canonicalizer only sees the rows currently in the view, so if your ORM has flooded the hash table and `dealloc` has dropped the tail, there's nothing left to fingerprint. Keep cardinality under control at the source first, by raising `pg_stat_statements.max`, getting on PG 18, or switching to array binding (more on that next), or you'll just be fingerprinting whichever rows happened to survive, not your real workload.

### The IN-list special case, and its PG 18 fix

There's one structural variation Postgres historically handled badly: lists of constants.

```
SELECT * FROM orders WHERE id IN (1, 2, 3);
SELECT * FROM orders WHERE id IN (1, 2, 3, 4);
SELECT * FROM orders WHERE id IN (1, 2, 3, 4, 5);
```

Before PostgreSQL 18, each of those was its own entry, because the length of the list changed the parsed tree. An app that builds `IN` clauses on the fly, which is most apps, could burn through thousands of entries with a single query shape and evict everything else on the way (more on eviction below).

PostgreSQL 18 finally collapses them. The three statements above, differing only in list length, now land in a single entry:

```
query | SELECT * FROM orders WHERE id IN ($1 /*, ... */)
calls | 3
```

**PostgreSQL 18: `IN`-lists squashed.**  
A constant list that differs only in length now collapses to one entry, shown as `IN ($1 /*, ... */)`. It's unconditional, with no GUC to turn on. The limit: it only fires for inline literals, so a driver that binds `IN ($1, $2, $3)` as parameters still gets a fresh entry for every list length.

That's a real improvement, and a good reason to be on 18. Two things keep it from being the whole answer. Most production clusters are still on 17 or earlier, where the explosion is alive and well. And the parameter-binding drivers it skips, JDBC and friends, are often the worst offenders: they send `IN ($1, $2, $3)` as bind parameters, so the values were never constants in the parsed tree and there is nothing to squash. `IN ($1,$2,$3)` and `IN ($1,$2,$3,$4)` stay separate entries, one new shape per list length.

The squashing briefly sat behind a GUC, `query_id_squash_values`, during PG 18 development. It was [removed before release](https://github.com/postgres/postgres/commit/9fbd53dea5d513a78ca04834101ca1aa73b63e59), so the behaviour is now always on.

If you're on one of those drivers, you're stuck regardless of your Postgres version, but there's a real fix, and it beats the PG 18 squashing because it works everywhere. Stop sending `IN`-lists and send an array. Rewrite this:

```
SELECT * FROM orders WHERE id IN ($1, $2, $3, ... $n); -- n params, n distinct shapes
```

as this:

```
SELECT * FROM orders WHERE id = ANY($1); -- one param, one shape, forever
```

`= ANY(array)` takes a single parameter, one array value, so the parsed tree is identical whether the array holds 3 elements or 3,000. That's one queryid, on every Postgres version and every driver, prepared or not, because the array length is never part of the structure the jumble sees. Most drivers and ORMs support array binding directly (`setArray`, `ARRAY[...]`, pgx's slice binding, ActiveRecord's `where(id: array)`), and the planner uses it for index selection much like an `IN`-list. For an ORM-heavy team it's the highest-leverage way to keep `pg_stat_statements` readable.

## The first-seen text is frozen

Every entry shows one query text, and it's the first one that created the entry. Not the most recent, not the most common, the first. After that the counters keep climbing on every matching execution, but the text on disk never changes, at least until the entry gets evicted and rebuilt.

Most of the time that's fine, because the constants are normalized to `$1` anyway. It stops being fine the moment your queries carry comments, and modern apps carry a lot of them. The [sqlcommenter](https://google.github.io/sqlcommenter/) and marginalia conventions tack structured key-value context onto every query:

```
SELECT id, name FROM customers WHERE id = $1
/*application='checkout',controller='orders',action='show'*/
```

Comments aren't part of the parsed tree, so they don't change the queryid. Every execution still maps to one entry. But the text on that entry is whatever showed up first. That splits comment tags into two kinds, and the split is the whole story:

| Tag class | Examples | What `pg_stat_statements` does with it |
| --- | --- | --- |
| **Static** (same value every call from one place) | `application`, `controller`, `action`, `job`, `framework` | **Survives.** The first-seen value is representative, because every call from that spot carries the same value. You can scrape it from the `query` text and trust it. |
| **Dynamic** (a new value every request) | `traceparent`, `request_id`, `trace_id`, `tenant_id`, `user_id` | **Lost.** The first request's value is frozen into the text forever, and every later per-request value is silently dropped. |

That second row is why `pg_stat_statements` can never be a join target for an APM or tracing tool. You can't ask it "show me the queries from the trace that timed out at 14:32," because it kept exactly one `traceparent`, the first it ever saw, and threw the rest away. A counter is a sum, and a sum has no room to remember which request contributed what. If you need dynamic tags, you need a per-execution capture path running alongside the extension (auto_explain, or a hook-based harvester), not the extension itself.

A `traceparent` in the `pg_stat_statements` text is a warning sign, not a feature: a per-request value reached the entry first and is now frozen there as if it were static context. If you scrape tags from these texts into a catalog, drop the hex-id-shaped ones.

## The averages lie to you

This is the limitation that catches the most people, because the numbers look so trustworthy.

Every counter is cumulative since the last reset. There's no time dimension inside the extension at all. The view is one snapshot of running totals, and that has two consequences that feed each other.

**There's no history built in.** Say you want to know what got slower this week. The obvious move is to read `mean_exec_time`, and that's exactly where it goes wrong. That number is the average over *all* time since the last reset, so on a cluster that's been up for months it blends a cold-cache Monday morning, the nightly batch job, and a quiet Sunday into one figure that describes no real moment. To get a "this week" number you have to do the bookkeeping yourself: snapshot the view on a schedule, keep each snapshot in a table, and subtract two of them. The average over the last hour is just `(total_exec_time_now - total_exec_time_then) / (calls_now - calls_then)`. The view hands you a running total; turning it into a rate is on you.

That subtraction is less safe than it looks. It quietly assumes the "now" counters carry on from the "then" counters, that they only ever climb. Usually true, not always. Eviction and resets, both covered in [part two](/posts/pg-stat-statements-part-2/), break that assumption, and a snapshot diff you'd actually trust has to handle three cases:

- **The entry was evicted and rebuilt.** If a query drops out of the table and later runs again, it comes back with counters at zero. Now `total_exec_time_now` is smaller than `total_exec_time_then`, and the subtraction gives you a negative delta. Don't read that as "the query got faster." Read it as "this window is invalid for this queryid," because the entry's history was cut.
- **A global reset happened between snapshots.** Someone ran `pg_stat_statements_reset()`, or a deploy script did. Every row's "now" is smaller than its "then." This one is easy to catch: compare `pg_stat_statements_info.stats_reset` against the time of your earlier snapshot, and if the reset is newer, the whole window crossed a reset and the entire diff is garbage.
- **The queryid is on only one side.** A query in "then" but missing from "now" was evicted, or its table was dropped and recreated (new OID, new id). A query in "now" but not "then" is new or rebuilt. Join the two snapshots with a full outer join, and treat the rows that don't match on both sides as events, not as zeros to subtract from.

PostgreSQL 17 adds a cleaner per-entry guard: every row carries `stats_since`, the moment the entry was created or recreated. If a row's `stats_since` is newer than your earlier snapshot, its counters reset under you, so discard that window no matter what the subtraction says.

Before PG 17 the global `stats_reset` is your only built-in signal, and per-entry eviction is invisible except through the counters-went-backwards trick. The bare formula is right; the monitoring layer around it is mostly this bookkeeping, and skipping it is how homegrown dashboards end up reporting impossible negative latencies.

**The average hides the spread.** Even inside one window, the extension only keeps four numbers per shape: `min`, `max`, `mean`, and `stddev` of exec time. It builds them on the fly, folding each execution into running aggregates and then dropping the timing, so there's no histogram, no percentiles, no per-call record to go back to. A query that runs in 1ms 99% of the time and 2 seconds the other 1% has a `mean_exec_time` around 21ms, a number that matches none of its real executions and hides the p99 that's paging your on-call. Two-humped latency, a common shape for real performance problems, just disappears into an average and a standard deviation. The extension can tell you a query is slow on average, not that it's sometimes catastrophically slow, which is usually the more dangerous case.

The mean is just `total_exec_time / calls`, but the extension maintains it (and the variance behind `stddev`) with Welford's online algorithm: each execution folds into a running mean and sum-of-squares, then its timing is dropped. An average and a standard deviation are all that survive, never a percentile, because no samples are kept to rank.

If you want percentiles or a latency timeline, this is the wrong tool, and querying it harder won't help. It dropped the spread when it wrote the row.

That's the first half of the story: everything `pg_stat_statements` tells you, and the ways it quietly bends what it reports. One logical query fragmented into hundreds of shapes, the text frozen at first sight, the whole distribution flattened into a single average.

The other half is what it never records at all: the entries it silently evicts, the query text that can vanish in one stroke, the plans it can't see, and the replicas it doesn't know exist. That's [part two](/posts/pg-stat-statements-part-2/), which also returns to the question we opened on: is this the query store Postgres is missing, or just the floor you'd build one on?
