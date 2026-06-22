---
title: The NULL in your NOT IN
source: https://boringsql.com/posts/not-in-null/
author: Radim Marek / boringSQL
published: 2026-06-15
saved: 2026-06-15
tags: [postgresql, sql, null, query-planner]
---

# The NULL in your NOT IN

A `NOT IN` query can return the wrong answer without telling you. It is valid SQL, it runs without an error, and it hands back a perfectly well-formed result set that happens to be empty when it should not be. No warning, no hint, nothing in the logs: just zero rows where you expected hundreds, and a database that considers it correct.

Almost always the cause is a single `NULL` sitting somewhere you forgot to look, combined with two keywords you have typed a thousand times: `NOT IN`. None of it is a Postgres bug. This is exactly what the SQL standard mandates, implemented faithfully. That is precisely what makes it so easy to walk into, and why the planner could not safely optimize around it for about twenty-five years. It comes down to one `if` statement in the parser.

## Sample schema

Nothing elaborate. A table of products, one of which has no category assigned yet, and a table of archived categories that happens to contain a `NULL`:

```sql
CREATE TABLE products (id int, category_id int);
INSERT INTO products VALUES (1, 10), (2, 20), (3, NULL), (4, 10);

CREATE TABLE archived (category_id int);
INSERT INTO archived VALUES (20), (NULL);
```

The `NULL` in `archived` is not contrived. The moment a column is nullable (and most are, by default), a `NULL` can find its way into any subquery you point a `NOT IN` at. That is the whole point: this is not an exotic data condition, it is the ordinary one.

## The query that returns nothing

Here is the request you have written a hundred times: give me the products whose category is not archived.

```sql
SELECT id, category_id FROM products
WHERE category_id NOT IN (SELECT category_id FROM archived);
```

You expect products 1 and 4 (category 10, which is not in the archived set). What comes back is:

```sql
 id | category_id
----+-------------
(0 rows)
```

Every row gone. Not a subset, not an off-by-one: all of them. Drop the `NULL` from `archived` and the same query behaves:

```sql
SELECT id, category_id FROM products
WHERE category_id NOT IN (SELECT category_id FROM archived
                          WHERE category_id IS NOT NULL);
```

```sql
 id | category_id
----+-------------
  1 |          10
  4 |          10
(2 rows)
```

To understand why a single `NULL` empties the entire result, we have to stop thinking of `NOT IN` as a single thing and watch the parser take it apart.

## IN is an OR, NOT IN is an AND

`IN` is not a primitive operator. It is shorthand that the parser rewrites into a chain of equality comparisons joined by `OR`:

```sql
x IN (a, b, c)
-- becomes
x = a OR x = b OR x = c
```

`NOT IN` is the logical negation of that, and by De Morgan's law negating an `OR` of equalities gives you an `AND` of inequalities:

```sql
x NOT IN (a, b, c)
-- becomes
x <> a AND x <> b AND x <> c
```

This is not an analogy. It is literally the expression Postgres builds, and you can read it straight off an `EXPLAIN`. The literal-list forms collapse into array operators whose names give the whole game away:

```sql
EXPLAIN (COSTS OFF) SELECT * FROM products WHERE category_id IN (1, 2, 3);
--  Filter: (category_id = ANY ('{1,2,3}'::integer[]))

EXPLAIN (COSTS OFF) SELECT * FROM products WHERE category_id NOT IN (1, 2, 3);
--  Filter: (category_id <> ALL ('{1,2,3}'::integer[]))
```

`IN` is `= ANY`: equal to any element, an `OR`. `NOT IN` is `<> ALL`: different from all elements, an `AND`.

The actual node types matter here, because they are what you end up staring at when you dump a parse tree or read a normalized [`pg_stat_statements`](https://boringsql.com/posts/pg-stat-statements/) entry. A literal list compiles to a single `ScalarArrayOpExpr`: the scalar on the left, the array on the right, and a `useOr` flag that is the entire difference between `= ANY` and `<> ALL`. The subquery forms are a different node altogether, a `SubLink`. Recognising those two names on sight tells you immediately which path the planner is on.

If "`IN` and `= ANY` are the same operator" is news: they compile to the same parse node and the same plan, with the spellings diverging only in plan-cache churn and selectivity estimates. The `NOT IN` case in front of you here is the one corner where the choice is not cosmetic but a matter of correctness.

## Three-valued logic does the rest

SQL does not have two truth values, it has three: true, false, and unknown. Any comparison against `NULL` yields `unknown`, because `NULL` means "no value here" and you cannot ask whether an absent value is different from 20:

```sql
-- not false. unknown (displayed as a blank)
SELECT 10 <> NULL;
```

Now walk the `NOT IN` expansion for product 1 (category 10) against the archived set of 20 and `NULL`:

`true AND unknown` is `unknown`, not `true`. A `WHERE` clause keeps a row only when its predicate evaluates to true. Both `false` and `unknown` cause the row to be discarded. So product 1 is dropped. Run the same arithmetic for product 4 and you land on `unknown` again.

The mechanism in one sentence: the instant a single `NULL` enters the right-hand side, the trailing `AND unknown` term can never be `true`, so the whole `NOT IN` can never be `true`, so every row is discarded, regardless of how many million rows you have or what they contain.

## NULLs on the left side too

Keeping `NULL`s out of the subquery is not enough. The same `unknown` arises from `NULL`s on the left: product 3 (whose `category_id` is `NULL`) evaluates to `unknown AND unknown`, so it is dropped even against a spotless right-hand set. `IN` and `NOT IN` are not complements: a row can fail both tests simultaneously. There is a `NULL`-shaped gap between them that belongs to neither.

## The seam, in the source

All of this reduces to one branch in one function. Open `src/backend/parser/parse_expr.c` and find `transformAExprIn`, the routine that turns both `IN` and `NOT IN` list expressions into something the planner can chew on. The very first thing it decides is whether it is building an `OR` or an `AND`:

```sql
/*
 * If the operator is <>, combine with AND not OR.
 */
if (strcmp(strVal(linitial(a->name)), "<>") == 0)
    useOr = false;
else
    useOr = true;
```

That is the entire fork. `IN` arrives carrying the operator `=` and gets `useOr = true`; `NOT IN` arrives carrying `<>` and gets `useOr = false`. The flag rides all the way down to where the boolean tree is finally assembled, several hundred lines later:

```sql
result = (Node *) makeBoolExpr(useOr ? OR_EXPR : AND_EXPR,
                               list_make2(result, cmp),
                               a->location);
```

`OR_EXPR` for `IN`, `AND_EXPR` for `NOT IN`. There is no special-casing of `NULL` anywhere in this function, and there does not need to be: the three-valued behavior is an emergent property of having chosen `AND`. The parser does the obvious, correct thing, and the `NULL` semantics fall straight out of standard boolean logic. The "bug", if you insist on the word, belongs to the SQL standard, which Postgres implements faithfully.

### A grammar asymmetry: list vs. subquery

A list and a subquery are built differently. The list form is the `<> ALL` chain of inequalities you just saw. The subquery form is not a `<> ALL` at all: it becomes `NOT (foo = ANY (subquery))`. Different shapes, same truth table, a `NULL` in the comparison makes the result `unknown`, and `unknown` loses. That `NOT (... = ANY ...)` shape is the one the planner sees.

## Why the planner won't save you

The correctness problem has a plan-shape twin. Watch what the planner does with three sibling queries on a larger schema of 200,000 `orders` and 1,000 `vip` rows.

First, the positive case, `IN` against a subquery:

```sql
EXPLAIN (COSTS OFF) SELECT * FROM orders
WHERE customer_id IN (SELECT customer_id FROM vip);
```

```sql
 Hash Semi Join
   Hash Cond: (orders.customer_id = vip.customer_id)
   ->  Seq Scan on orders
   ->  Hash
         ->  Seq Scan on vip
```

A clean semi-join. The planner promotes the subquery to a first-class relation, picks a hash join, and is free to reorder it against the rest of the query. Now the natural mirror, orders whose customer is not a VIP, written first with `NOT EXISTS`:

```sql
EXPLAIN (COSTS OFF) SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM vip v WHERE v.customer_id = o.customer_id);
```

```sql
                  QUERY PLAN
----------------------------------------------
 Hash Anti Join
   Hash Cond: (o.customer_id = v.customer_id)
   ->  Seq Scan on orders o
   ->  Hash
         ->  Seq Scan on vip v
```

A hash anti-join, the efficient and symmetric counterpart to the semi-join. Now the same intent expressed with `NOT IN`:

```sql
EXPLAIN (COSTS OFF) SELECT * FROM orders
WHERE customer_id NOT IN (SELECT customer_id FROM vip);
```

```sql
                          QUERY PLAN
---------------------------------------------------------------
 Seq Scan on orders
   Filter: (NOT (ANY (customer_id = (hashed SubPlan 1).col1)))
   SubPlan 1
     ->  Seq Scan on vip
(4 rows)
```

No join at all. The subquery collapses into an opaque `SubPlan` filter bolted onto a sequential scan, the `NOT (... = ANY ...)` shape from the grammar is right there in the filter.

The wording of that filter line is version-specific. PostgreSQL 16 prints the terser `Filter: (NOT (hashed SubPlan 1))`; a later version changed `EXPLAIN` to expose the inner comparison, which is the `NOT (ANY (... = (hashed SubPlan 1).col1))` form shown here on 18.4. The plan underneath is the same opaque subplan in every released version.
The difference between those last two plans is structural, not a tuning detail. A `Hash Anti Join` builds one hash table from the inner relation and streams the outer relation through it exactly once, and when the inner side outgrows `work_mem` it partitions into batches and spills to disk without changing the result. A `SubPlan` gives the planner none of that. Even the `hashed` variant you see here is evaluated in place as a filter on the scan: it cannot be reordered during the global join search, it cannot have the outer query's join clauses pushed into it to drive an index scan, and it has no multi-batch spill logic to fall back on if the hashed set turns out larger than the estimate. The subquery stops being a relation the optimizer can plan and becomes an opaque function the optimizer has to call, once per outer row. This is the same optimization-fence effect as [an uninlined CTE](https://boringsql.com/posts/good-cte-bad-cte/): the planner simply cannot see inside. On small inputs nobody notices; on large ones it is the difference between milliseconds and minutes.

The reason the planner is stuck with that shape is precisely the `NULL` semantics from earlier. An anti-join keeps a row when it finds no match. But `NOT IN` must discard a row when the comparison goes `unknown`, and "unknown" is not the same as "no match". Those two behaviors diverge exactly when a `NULL` is in play, so the planner historically could not prove the rewrite safe and never attempted it.

Declaring the columns `NOT NULL` does not rescue you on released Postgres. I constrained both the outer and inner columns `NOT NULL` on 18.4 and re-ran the query, and the plan was still the opaque `SubPlan` filter. The released planner does not even look. So `NOT IN (subquery)` has been a correctness trap for as long as Postgres has existed, and a planner pessimization ever since anti-joins arrived in 8.4 (2009) and every other negated subquery learned to use them — neither of which the released optimizer rescues you from.

## The fix is landing in PostgreSQL 19

That last paragraph is finally going out of date. This is a performance fix, not a semantic one: it rescues the case that was already correct but badly planned (`NOT NULL` columns stuck behind the opaque `SubPlan` from the previous section) and leaves nullable columns exactly where the SQL standard puts them. Tracing this through a checkout of the development branch, I found a function that exists in no released Postgres: `sublink_testexpr_is_not_nullable`, in `src/backend/optimizer/plan/subselect.c`. It guards a brand-new branch inside `convert_ANY_sublink_to_join`:

```sql
/*
 * Per SQL spec, NOT IN is not ordinarily equivalent to an anti-join, so
 * that by default we have to fail when under_not.  However, if we can
 * prove that neither the outer query's expressions nor the sub-select's
 * output columns can be NULL, and further that the operator itself cannot
 * return NULL for non-null inputs, then the logic is identical and it's
 * safe to convert NOT IN to an anti-join.
 */
if (under_not &&
    (!sublink_testexpr_is_not_nullable(root, sublink) ||
     !query_outputs_are_not_nullable(subselect)))
    return NULL;
```

`git blame` dates it to commit `383eb21ebff`, "Convert NOT IN sublinks to anti-joins when safe", merged in March 2026 by Richard Guo. It lives on `master`, bound for PostgreSQL 19. It appears in no `REL_18` tag, which is exactly why my 18.4 box still produced the `SubPlan` even with `NOT NULL` columns. The commit message states the bargain plainly:

if we can prove that neither side of the comparison can yield NULL values, and further that the operator itself cannot return NULL for non-null inputs, the behavior of NOT IN and anti-join becomes identical.

The proof has to establish three things:

- Both operands provably non-`NULL`. Established from schema `NOT NULL` constraints (via a NOT-NULL-attnums hash table), from outer-join nullability tracking (so a `Var` from the nullable side of an outer join does not qualify), and from qual clauses that force a `Var` non-null.

- The subquery's output columns provably non-`NULL`, the `query_outputs_are_not_nullable` half of the guard.

- A `NULL`-safe operator. The operator must belong to a B-tree or Hash operator family. That is a proxy for "behaves like a normal boolean comparison and won't return `NULL` on non-null inputs", because an operator that did return `NULL` there would break the very index it claims to support.

When all three hold, the `NULL`-handling mismatch evaporates and `NOT IN` is finally allowed to become a `JOIN_ANTI`:

```sql
result->jointype = under_not ? JOIN_ANTI : JOIN_SEMI;
```

It lifts a planner limitation that has stood since anti-joins themselves arrived in 8.4, but only when the planner can prove no `NULL` can reach the comparison. If your column is nullable, you are exactly where you have always been, in PostgreSQL 19 as in 9.x. The semantics never changed; the optimizer merely learned to recognize the cases where `NULL` is provably absent.

## Decision matrix

You wroteInternal shapeNULL on right →NULL on left →Planner (≤ PG 18)
`IN (1,2,3)``= ANY`, an `OR`absorbed, no harmrow dropped (no match)`ScalarArrayOpExpr`
`NOT IN (1,2,3)``<> ALL`, an `AND`all rows droppedrow dropped`ScalarArrayOpExpr`
`IN (subquery)``= ANY` sublinkabsorbedrow droppedSemi Join
`NOT IN (subquery)``NOT (= ANY)` sublinkall rows droppedrow droppedopaque `SubPlan` filter ¹
`NOT EXISTS (...)`anti-join sublinkrow keptrow keptAnti Join

¹ PostgreSQL 19 promotes this to an Anti Join when both the outer expression and the subquery output column are provably `NOT NULL`.

## What to do instead

Don't wait for PostgreSQL 19, and don't lean on it once it ships: it only fires on provably `NOT NULL` columns. Every portable fix comes down to one rule: keep `NULL` away from `NOT IN`. In practice that means changing the query, and there are three moves, in order of preference.

Default to `NOT EXISTS`. Make this the habit and you never hit the trap again. Same anti-join semantics and the same plan as the working `NOT IN` would have wanted, on every version, nullable columns or not:

```sql
SELECT id, category_id FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM archived a WHERE a.category_id = p.category_id
);
```

It keeps product 3, the `NULL` category, because `NOT EXISTS` asks whether a matching archived row exists, not whether `category_id = NULL` is true. That is usually the answer you wanted. If you do want the `NULL`-category rows gone, add `AND p.category_id IS NOT NULL` and you have decided it on purpose.

Filter the `NULL`s out of the subquery when you have to keep the `NOT IN` (a generated query you can only edit inside the parentheses, say):

```sql
SELECT id FROM products
WHERE category_id NOT IN (
    SELECT category_id FROM archived WHERE category_id IS NOT NULL
);
```

This covers the right-hand `NULL` and nothing else: product 3 still disappears, because the left-hand `NULL` is untouched. Use it only when you cannot reach for `NOT EXISTS`.

Use `EXCEPT` for whole-set comparisons. It matches rows with `IS NOT DISTINCT FROM`, so two `NULL`s count as equal and the three-valued trap never fires. But that same rule means a `NULL` in `archived` removes the `NULL` rows from `products`:

```sql
SELECT category_id FROM products
EXCEPT
SELECT category_id FROM archived;
-- returns: {10}
-- product 3 (NULL category) is dropped because archived also has a NULL
```

So `NOT EXISTS` keeps product 3 and `EXCEPT` drops it. Pick the one whose answer matches what you're after.
