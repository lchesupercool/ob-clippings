---
title: Automating PostgreSQL Index Tuning Using AI
source: https://stormatics.tech/blogs/automating-postgresql-index-tuning-using-ai
saved: 2026-05-29 16:35:00 +0800
author: Warda Bibi
published: 2026-05-28
images: 0
---

## [Automating PostgreSQL Index Tuning Using AI](https://stormatics.tech/blogs/automating-postgresql-index-tuning-using-ai "Automating PostgreSQL Index Tuning Using AI")

If you have a slow query, one of the obvious moves is to add an index. So you look at the WHERE clause, pick a column, run CREATE INDEX, and test again. Sometimes it helps, often it doesn’t. And now you have an index sitting there, not helping reads, but slowing down every write, because INSERT, UPDATE, and DELETE all have to maintain it. And it gets worse as your system grows.

Five queries are manageable. You can reason about column choices, test combinations, and check EXPLAIN output. When you are dealing with fifty queries across a dozen tables,  you are evaluating hundreds of possible column combinations manually, each one potentially breaking something in production if you get the locking wrong. 

## Why index tuning is harder than it looks

Most people know that indexes speed up reads. Fewer people think carefully about what actually happens when they add one.

First, PostgreSQL might not use it. The planner compares strategies and picks the cheapest one. If your query touches a large fraction of the table, a sequential scan might actually be cheaper than an index scan. Creating the index doesn’t change that math, it just adds overhead.

Second, even with CONCURRENTLY, creating an index on a busy table isn’t free. It competes with your workload, can cause replication lag, and sometimes it times out. People don’t plan for this until it happens at 2am.

Third, and this one is subtle, adding the wrong index is sometimes worse than adding nothing. You pay the write overhead with zero read benefit.

The harder part is column ordering. A composite index on (status, customer\_id) and one on (customer\_id, status) are completely different things. The planner’s decision about which one to use depends on your data distribution, which conditions appear in your WHERE clause, and how selective each column is. Getting this wrong by hand is easy. Verifying it without touching production is the real challenge.

## The idea behind automation

When a problem is repetitive, time-consuming, and mostly mechanical, you automate it. That’s what pg\_index\_search does. It takes the trial-and-error part of index tuning and turns it into a structured pipeline.

The tool reads your actual query workload from pg\_stat\_statements, figures out which columns are worth indexing, generates candidate indexes, and then, critically, tests them without writing anything to disk. You get back CREATE INDEX statements with measured cost improvements. You decide what to apply and when.

Here’s how each part works.

### Step 1: Start with what’s actually slow

The tool pulls queries from pg\_stat\_statements with a few filters. 

1. It ignores one-off queries (fewer than 50 calls is not enough data to be meaningful).
2. It skips anything already running in under 5ms (indexes won’t move the needle there).
3. It sorts by total\_exec\_time, not mean latency.

That last part matters more than it sounds. A query averaging 50ms that runs 10,000 times a day is a much bigger problem than a 2-second query that runs once. Optimizing for total impact rather than individual query speed is how you actually move the metric that matters.

Each query gets a weight based on call count times average execution time. That weight flows through the rest of the pipeline, so the algorithm keeps its focus on what will reduce total database load.

### Step 2: Parse the queries to find indexable columns

pg\_stat\_statements stores queries with placeholders like $1, $2. The planner needs real values to estimate selectivity, so the tool replaces those placeholders with representative values pulled from pg\_stats (most common values first, histogram data as fallback).

Then it parses each query into an AST using pglast, which uses PostgreSQL’s own parser under the hood. From there it extracts columns from WHERE clauses, JOINs, ORDER BY, and HAVING. Columns that only appear in the SELECT list are ignored as they don’t help with index selection.

For a query like:

```
SELECT user_id, event_type, created_at  
FROM events  
WHERE tenant_id = $1  
  AND event_type = 'feature_use'  
  AND created_at > now() - interval '30 days'  
ORDER BY created_at;
```

The output is: events: [tenant\_id, event\_type, created\_at]. That’s the input to the next step.

### Step 3: Generate candidate indexes

With the relevant columns identified per table, the tool generates all single-column indexes, all two-column combinations, and all three-column combinations. Existing indexes are filtered out. Very wide columns (avg\_width over 40 bytes) are excluded since indexing them is usually impractical.

For the events example above, you would get candidates like 

```
(tenant_id), (event_type),(created_at)  
(tenant_id, event_type), (tenant_id, created_at), (event_type, created_at)
```

and the three-column composite. One honest limitation here: column ordering within each combination is fixed. The tool generates (tenant\_id, event\_type) but not (event\_type, tenant\_id). For many cases this doesn’t matter. For some, especially mixed equality-and-range queries, it does. That gap is exactly where the LLM optimizer comes in, which we’ll get to.

### Step 4: Test everything safely with HypoPG

This is the part that makes the whole approach production-safe.

Instead of creating real indexes to test them, the tool uses HypoPG, a PostgreSQL extension that creates hypothetical indexes entirely in memory. These fake indexes hook directly into the planner, so when you run EXPLAIN, the planner treats them as if they exist and uses real table statistics to estimate costs.

The loop looks like this: create a hypothetical index, run EXPLAIN, record the cost, and clear the index. For every candidate. Results are cached so the same combination isn’t re-evaluated.

There are no disk writes or table locks and there is definitely no impact on live traffic. You can test any combination of columns on any table in your production database during business hours, and nothing will change except the planner’s internal state for the duration of an EXPLAIN call. The weighted cost formula is:

 **Σ (cost × calls × avg\_time)**

across all queries. So, indexes that benefit high-frequency, high-latency queries score better than those that help rare queries slightly.

### Step 5: Find the best combination with greedy search

Testing indexes individually is straightforward but finding the best combination is the hard part.

If you have 50 candidate indexes, there are 2^50 possible subsets to evaluate. That’s roughly a quadrillion combinations, which is not practical.

The tool uses a greedy approach instead. In the first round, it tries every candidate individually and picks the one with the best objective score. In the second round, it tries adding each remaining candidate on top of the winner. It keeps going until no candidate improves cost by at least 10%, or until it hits a time or storage budget.The objective function is :

**log(query\_cost) + 2 × log(total\_space).**

Using logarithms matters as it prevents the algorithm from chasing large absolute numbers. A drop from 1,000,000 to 500,000 and a drop from 100 to 50 are the same relative improvement, and log() treats them that way. The 2× multiplier on storage means large indexes need to justify themselves with meaningful query improvement to make the cut.

In practice, on a 18-million-row SaaS-style database, this produced an overall cost reduction from 3.6 billion to 304 million, about a 12x improvement, with a total index size of roughly 55 KB. That index size is tiny because indexes only store the indexed columns, not the full rows. The 874 MB events table has a 55 KB index that changes query cost by an order of magnitude.

### The output

The output is ready-to-run SQL. Nothing is applied automatically.

```
SUMMARY  
  Recommendations: 2  
  Base cost:       4175.8  
  New cost:        33.2  
  Improvement:     125.7x  
  Total index size: 1.3 MB
```

RECOMMENDED INDEXES

[1] CREATE INDEX ON orders  
    USING btree (customer\_id)  
    Size: 457 kB  
    Standalone: 99.6x (4175 → 41)  
    Stacked:    99.6x (4175 → 41)

[2] CREATE INDEX ON orders  
    USING btree (status, customer\_id)  
    Size: 800 kB  
    Standalone: 1.5x (4175 → 2727)  
    Stacked:    1.3x (41 → 33)

Two numbers per index tell you what you actually need to know. Standalone improvement is what that index does on its own. Stacked improvement is what it adds once the prior indexes are already in place. Index 2 looks weak on its own, 1.5x, but on top of index 1, it still contributes a 1.3x improvement. Whether that’s worth 800 KB of storage is your call.

The improvement is a multiplier (base\_cost ÷ new\_cost), not a percentage. A 125x improvement means the planner estimates 125 times less total work, not 125% less.

Also worth noting: some queries in the output show 1.0x improvement. The tool didn’t recommend indexes for them and that’s intentional. Over-indexing has real costs, and the scoring function is designed to avoid it.

### Where the greedy algorithm falls short and where the LLM picks up

The greedy DTA approach is fast, predictable, and handles most cases well. But it has structural blind spots.

* Column ordering is one. The algorithm generates combinations so (status, customer\_id) might never be tried if (customer\_id, status) came out of the combination generator first. For queries with a mix of equality and range conditions, the order can meaningfully change which rows the index eliminates first.
* Partial indexes are another. The tool doesn’t generate

```
CREATE INDEX ON orders (customer_id) WHERE status = 'pending'
```

A partial index like that would be smaller and faster for queries that always filter on that status value, but it’s outside what the combination generator can produce.

* Expression indexes fall into the same gap. lower(email) as an index expression isn’t something the algorithm considers.

The LLM optimizer handles these. It takes the query and its EXPLAIN plan, sends them to Claude or GPT with a history of previous attempts, and gets back structured index suggestions that go beyond column combinations. Each suggestion is validated the same way, through HypoPG and EXPLAIN, so nothing is trusted without measurement.

The iterative loop tracks the best 5 attempts and stops after 5 consecutive rounds without improvement. After 3 failed attempts, the prompt explicitly asks for less obvious suggestions, and the temperature goes up to 1.2 to get a more varied output.

The practical workflow: run DTA first across the whole workload. Then, on queries where DTA’s result is underwhelming or where you suspect column ordering matters, run LLM mode on that specific query. Please note that LLMs are complementary in this process, not alternatives.

### How to run it:

```
git clone https://github.com/WardaBibi/pg_index_search  
cd pg_index_search  
uv pip install -e .  
On your database:
```

CREATE EXTENSION hypopg;  
CREATE EXTENSION pg\_stat\_statements;  
ANALYZE;

Then point it at your database:

```
# Automatic workload mode  
uv run python main.py "postgresql://user@host/mydb"
```

# Specific query  
uv run python main.py “postgresql://user@host/mydb” \  
  –queries “SELECT \* FROM orders WHERE customer\_id = 5”

# LLM mode  
export ANTHROPIC\_API\_KEY=”sk-ant-…”  
uv run python main.py “postgresql://user@host/mydb” \  
  –method llm \  
  –queries “SELECT \* FROM orders WHERE customer\_id = 5 AND status = ‘pending'”

Run ANALYZE before using the tool so the planner has current statistics. After applying any recommended index, run EXPLAIN ANALYZE to confirm what actually happens during execution, planner estimates are good but not identical to runtime behavior, especially if your data distribution shifts.

### The thing worth internalizing

Manual index tuning has a hard ceiling. It works when you have a handful of queries and time to reason through each one. It breaks down when you have 50+ queries across multiple tables, when your workload changes week to week, and when you need to validate changes without risking a production incident.

The combination of HypoPG simulation and workload-weighted scoring changes the constraint. You go from “let me guess and check” to “here’s what the planner actually prefers, measured against your real workload.”  You are still the one deciding what to apply and when. The tool just replaces the guessing.

The repo is at [github.com/WardaBibi/pg\_index\_search](http://github.com/WardaBibi/pg_index_search).

