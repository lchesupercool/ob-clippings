---
title: VACUUM at the Page Level
source: https://boringsql.com/posts/vacuum-at-the-page-level/
author: Radim Marek
published: 2026-07-05
saved: '2026-07-06'
tags:
- postgresql
- vacuum
- database
- storage-internals
---

# VACUUM at the Page Level

In [HOT Updates in Postgres](https://boringsql.com/posts/hot-updates/) we covered page pruning clean up HOT chains, an elegant shortcut where PostgreSQL reclaims dead tuple space during ordinary reads. All that without waiting for any background process. But pruning is exactly that: a shortcut. It only works within a single page, and only for HOT-updated tuples. For everything else (cold updates that touch indexed columns, plain DELETEs, index entry cleanup, free space map registration, visibility map maintenance) we need VACUUM.

This article won't repeat what VACUUM does operationally. The [DELETEs are difficult](https://boringsql.com/posts/deletes-are-difficult/) article covers autovacuum tuning, worker allocation, and the operational side of dead tuple cleanup. Here we are going to watch VACUUM work byte by byte. We'll snapshot a page before and after each phase, tracking exactly what changes in the page header, line pointers, tuple headers, free space map, and visibility map. Same tools as always: `pageinspect`, `pg_visibility`, and `pg_freespacemap`.

## Setup

We need a table with enough rows to make the before-and-after comparison meaningful, plus indexes to demonstrate the full VACUUM cycle.

```
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE EXTENSION IF NOT EXISTS pg_visibility;
CREATE EXTENSION IF NOT EXISTS pg_freespacemap;

CREATE TABLE vacuum_demo (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category text NOT NULL,
    payload text
);

INSERT INTO vacuum_demo (category, payload)
SELECT
    'cat_' || (i % 5),
    repeat('x', 100)
FROM generate_series(1, 50) AS i;
```

Fifty rows with a 100-byte payload each. The primary key gives us an index, which matters: VACUUM's behavior changes when indexes are involved. Run VACUUM once upfront so we start from a clean baseline:

```
VACUUM vacuum_demo;
```

## Snapshot before any deletes

Record the baseline state of page 0. First the page header:

```
SELECT lower, upper, special, pagesize
FROM page_header(get_raw_page('vacuum_demo', 0));
```

```
 lower | upper | special | pagesize
-------+-------+---------+----------
   224 |  1392 |    8192 |     8192
(1 row)
```

pd_lower is at 224: that's the 24-byte page header plus 50 line pointers at 4 bytes each (24 + 200 = 224). pd_upper is at 1392, so our tuples occupy bytes 1392 through 8191. Free space is 1392 - 224 = 1168 bytes. Not much room left; those 100-byte payloads add up.

Now the line pointers and tuple headers:

```
SELECT lp, lp_flags, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
LIMIT 10;
```

```
 lp | lp_flags | lp_off | lp_len | t_xmin | t_xmax | t_ctid
----+----------+--------+--------+--------+--------+--------
  1 |        1 |   8056 |    135 |    746 |      0 | (0,1)
  2 |        1 |   7920 |    135 |    746 |      0 | (0,2)
  3 |        1 |   7784 |    135 |    746 |      0 | (0,3)
  4 |        1 |   7648 |    135 |    746 |      0 | (0,4)
  5 |        1 |   7512 |    135 |    746 |      0 | (0,5)
  6 |        1 |   7376 |    135 |    746 |      0 | (0,6)
  7 |        1 |   7240 |    135 |    746 |      0 | (0,7)
  8 |        1 |   7104 |    135 |    746 |      0 | (0,8)
  9 |        1 |   6968 |    135 |    746 |      0 | (0,9)
 10 |        1 |   6832 |    135 |    746 |      0 | (0,10)
(10 rows)
```

`lp_len` is 135, the tuple's actual byte length, but each tuple occupies MAXALIGN'd slot of 136 bytes on the page; notice the `lp_off` values step down by 136. That aligned stride is what the free-space arithmetic below uses.

Every line pointer is LP_NORMAL (lp_flags = 1). Every tuple has `t_xmax = 0`: nobody has touched these rows since they were inserted. Every `t_ctid` points to itself. This is a perfectly clean page.

## Create dead tuples

Now make some rows dead:

```
DELETE FROM vacuum_demo WHERE id % 3 = 0;
```

```
DELETE 16
```

That deletes roughly every third row: IDs 3, 6, 9, 12, and so on. Sixteen rows are now dead. Look at the page before VACUUM runs:

```
SELECT lp, lp_flags, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
LIMIT 10;
```

```
 lp | lp_flags | lp_off | lp_len | t_xmin | t_xmax | t_ctid
----+----------+--------+--------+--------+--------+--------
  1 |        1 |   8056 |    135 |    746 |      0 | (0,1)
  2 |        1 |   7920 |    135 |    746 |      0 | (0,2)
  3 |        1 |   7784 |    135 |    746 |    747 | (0,3)
  4 |        1 |   7648 |    135 |    746 |      0 | (0,4)
  5 |        1 |   7512 |    135 |    746 |      0 | (0,5)
  6 |        1 |   7376 |    135 |    746 |    747 | (0,6)
  7 |        1 |   7240 |    135 |    746 |      0 | (0,7)
  8 |        1 |   7104 |    135 |    746 |      0 | (0,8)
  9 |        1 |   6968 |    135 |    746 |    747 | (0,9)
 10 |        1 |   6832 |    135 |    746 |      0 | (0,10)
(10 rows)
```

Look at rows 3, 6, and 9. Their `t_xmax` is now 747, the transaction ID of the DELETE statement. But everything else is unchanged. lp_flags is still 1 (LP_NORMAL). lp_off and lp_len are the same. The tuples are still physically sitting on the page, consuming space. The page header hasn't changed either:

```
SELECT lower, upper, special, pagesize
FROM page_header(get_raw_page('vacuum_demo', 0));
```

```
 lower | upper | special | pagesize
-------+-------+---------+----------
   224 |  1392 |    8192 |     8192
(1 row)
```

pd_lower and pd_upper are identical to before the DELETE. PostgreSQL marked the rows as dead (by stamping t_xmax) but did not reclaim a single byte. The dead tuples are bloat, and they will stay that way until VACUUM arrives.

## How VACUUM processes a table

Before we run VACUUM and watch the page change, one thing about *how* it runs will explain everything the snapshots are about to show. VACUUM does its work in three passes, and the split between them is the whole reason a deleted tuple's storage disappears at one moment and its line pointer disappears at another. Keep an eye on that gap: it is what the rest of this article makes visible.

### Phase 1: Heap scan - prune, freeze, collect dead TIDs

VACUUM scans every heap page sequentially, and this first pass does far more than look. For each page, it runs page pruning (the same `heap_page_prune_and_freeze` machinery that fires opportunistically during ordinary reads). Pruning is where the bytes actually come back: it removes the storage of dead tuples, defragments the page, and advances pd_upper. It also opportunistically freezes tuples that are old enough.

Prior to PostgreSQL 17, VACUUM stored dead TIDs in a flat array allocated from `maintenance_work_mem`. If the array filled up, VACUUM had to pause, perform index and heap cleanup for the collected batch, then resume scanning. Since PostgreSQL 17, VACUUM uses a radix tree-based TID store that is far more memory-efficient, making `maintenance_work_mem` much less likely to be a bottleneck.

But here is the subtlety that the rest of this article hinges on. Pruning cannot simply mark a deleted tuple's line pointer LP_UNUSED, because indexes still point at it by TID. So for a table with indexes, a dead tuple's line pointer is set to **LP_DEAD**: its storage is gone, but the 4-byte slot stays put, holding the TID's place until the index entries are removed. Those LP_DEAD TIDs are what VACUUM collects into its dead-TID store for the next phase.

### Phase 2: Index cleanup

With the list of dead TIDs in hand, VACUUM scans each index on the table. For every index, it walks through all index entries and removes any that point to a dead TID. This is the expensive part. VACUUM must read every index page, even if only handful of entries need removal.

This is also why index bloat happens. If VACUUM cannot finish this phase, because a long-running transaction is holding back the visibility horizon or the table has many indexes and the dead TID list exceeds memory, index entries pointing to dead tuples accumulate. We covered the index bloat implications in detail in [VACUUM Is a Lie](https://boringsql.com/posts/vacuum-is-lie/).

### Phase 3: Heap cleanup - freeing the line pointers

After the indexes are clean, no index entry references those dead TIDs anymore, so the reserved slots can finally be released. VACUUM revisits each page that had dead tuples and does what it couldn't do in phase 1:

1. Flips each collected LP_DEAD line pointer to LP_UNUSED (0), reclaiming the slot
2. Sets the visibility map bit if all remaining tuples are visible to everyone

A table with **no indexes** skips this split entirely: with no index entries to worry about, phase 1's prune sets dead line pointers straight to LP_UNUSED in a single heap pass, and there is no phase 3. The two-pass dance, LP_DEAD now and LP_UNUSED later, exists precisely because of the indexes.

Notice what is *not* on that list. The tuple data was already removed and the page already defragmented back in phase 1's prune; that is where pd_upper moved. Phase 3 reclaims the line-pointer slots, not the tuple bytes. The distinction is invisible if you only look before and after plain `VACUUM`, so let's make it visible.

Let's run VACUUM in two steps to catch the intermediate state. `VACUUM (INDEX_CLEANUP OFF)` performs phase 1's prune but skips index cleanup, and therefore skips the second heap pass too; it has no choice but to leave the dead line pointers as LP_DEAD.

**Interactive VACUUM Visualizer**

Run DELETE across three heap pages and watch nothing get reclaimed, page-prune to reach `LP_DEAD`, then step VACUUM through heap scan, index cleanup, and heap cleanup that flips the slots to `LP_UNUSED`.

[Open Visualizer](https://boringsql.com/visualizers/vacuum/)

## After the prune: LP_DEAD, and space already back

Page header after pruning:

```
SELECT lower, upper, special, pagesize
FROM page_header(get_raw_page('vacuum_demo', 0));
```

```
 lower | upper | special | pagesize
-------+-------+---------+----------
   224 |  3568 |    8192 |     8192
(1 row)
```

pd_lower is still 224; the line pointer array hasn't shrunk. But pd_upper jumped from 1392 to 3568. That's 2176 bytes of reclaimed space (16 dead tuples at the 136-byte aligned stride = 2176). Free space on the page went from 1168 to 3344 bytes. And we have not touched an index yet: this all happened during pruning, in the first heap pass.

Now the line pointers:

```
SELECT lp, lp_flags, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
LIMIT 10;
```

```
 lp | lp_flags | lp_off | lp_len | t_xmin | t_xmax | t_ctid
----+----------+--------+--------+--------+--------+--------
  1 |        1 |   8056 |    135 |    746 |      0 | (0,1)
  2 |        1 |   7920 |    135 |    746 |      0 | (0,2)
  3 |        3 |      0 |      0 |        |        |
  4 |        1 |   7784 |    135 |    746 |      0 | (0,4)
  5 |        1 |   7648 |    135 |    746 |      0 | (0,5)
  6 |        3 |      0 |      0 |        |        |
  7 |        1 |   7512 |    135 |    746 |      0 | (0,7)
  8 |        1 |   7376 |    135 |    746 |      0 | (0,8)
  9 |        3 |      0 |      0 |        |        |
 10 |        1 |   7240 |    135 |    746 |      0 | (0,10)
(10 rows)
```

Line pointers 3, 6, and 9 are now **lp_flags = 3 (LP_DEAD)**, not LP_UNUSED. Their storage is gone (lp_off and lp_len are 0), but the slots are still there, reserving those TIDs. They can't be freed yet, because the primary key still points at them:

```
SELECT live_items FROM bt_page_stats('vacuum_demo_pkey', 1);
```

```
 live_items
------------
         50
(1 row)
```

Fifty index entries, same as before the delete. The index cleanup we skipped is exactly what removes the sixteen stale ones, and until it runs, those LP_DEAD slots stay stuck.

## After the full VACUUM: LP_UNUSED

Now run an ordinary VACUUM to finish the job:

```
VACUUM vacuum_demo;

SELECT lower, upper, special, pagesize
FROM page_header(get_raw_page('vacuum_demo', 0));
```

```
 lower | upper | special | pagesize
-------+-------+---------+----------
   224 |  3568 |    8192 |     8192
(1 row)
```

pd_upper is unchanged at 3568. That is the point: the second heap pass reclaims no tuple bytes; there are none left, pruning already took them. What it does is free the line-pointer slots, now that the index is clean:

```
SELECT lp, lp_flags, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
LIMIT 10;
```

```
 lp | lp_flags | lp_off | lp_len | t_xmin | t_xmax | t_ctid
----+----------+--------+--------+--------+--------+--------
  1 |        1 |   8056 |    135 |    746 |      0 | (0,1)
  2 |        1 |   7920 |    135 |    746 |      0 | (0,2)
  3 |        0 |      0 |      0 |        |        |
  4 |        1 |   7784 |    135 |    746 |      0 | (0,4)
  5 |        1 |   7648 |    135 |    746 |      0 | (0,5)
  6 |        0 |      0 |      0 |        |        |
  7 |        1 |   7512 |    135 |    746 |      0 | (0,7)
  8 |        1 |   7376 |    135 |    746 |      0 | (0,8)
  9 |        0 |      0 |      0 |        |        |
 10 |        1 |   7240 |    135 |    746 |      0 | (0,10)
(10 rows)
```

Line pointers 3, 6, and 9 have gone from LP_DEAD to **lp_flags = 0 (LP_UNUSED)**. The slots are now empty and available for reuse by the next INSERT that lands on this page. And the index has dropped to 34 entries; the sixteen stale ones are gone:

```
SELECT live_items FROM bt_page_stats('vacuum_demo_pkey', 1);
```

```
 live_items
------------
         34
(1 row)
```

The surviving heap tuples were compacted back in the prune; notice the lp_off values shifted from their pre-VACUUM positions, VACUUM having moved surviving tuples to form a single contiguous block of free space between pd_lower and pd_upper.

**pd_lower didn't change.** Even though line pointers 3, 6, and 9 are now LP_UNUSED, they still occupy slots in the array. The next INSERT will reuse one of these slots rather than appending a new one at the end. PostgreSQL tracks reusable slots via the `PD_HAS_FREE_LINES` flag in the page header.

## The line pointer lifecycle, precisely

Let's be exact about how line pointers transition between states. This matters because different code paths produce different transitions:

**LP_UNUSED (0)**: the slot is empty. Either it was never used, or VACUUM has fully reclaimed it. The next INSERT that targets this page will grab this slot and assign it to a new tuple, transitioning it to LP_NORMAL.

**LP_NORMAL (1)**: the slot points to a tuple on the page. That tuple might be live (t_xmax = 0 or t_xmax aborted), or it might be dead (t_xmax committed and invisible to all). The line pointer itself doesn't encode liveness; that's determined by the tuple header and the visibility rules we covered in [PostgreSQL MVCC, Byte by Byte](https://boringsql.com/posts/postgresql-mvcc-byte-by-byte/).

**LP_REDIRECT (2)**: the slot points not to a tuple but to another line pointer. This is created by HOT pruning: when the head of a HOT chain is pruned, its line pointer becomes a redirect so that indexes (which still reference the original line pointer number) can follow the redirect to find the current version. We saw this in [HOT Updates in Postgres](https://boringsql.com/posts/hot-updates/).

**LP_DEAD (3)**: the slot is known to contain a dead tuple whose storage has been reclaimed, but whose TID may still be referenced by index entries. Index scans that encounter it skip it immediately without a visibility check. On the heap it is set by page pruning, including the prune in VACUUM's own first heap pass. The index side carries the same LP_DEAD hint, and ordinary readers set it: when an index scan follows an entry to a heap tuple that turns out to be dead to everyone, the `kill_prior_tuple` optimization flags that index entry LP_DEAD so later scans skip it without a heap visit, cleanup work that plain SELECTs do long before VACUUM arrives. For a table with indexes, this is the state a plain deleted tuple sits in between pruning and index cleanup, exactly as we watched above.

The transitions look like this:

```
LP_UNUSED (0) ----INSERT----> LP_NORMAL (1)

-- table WITH indexes: two heap passes
LP_NORMAL (1) --prune (1st pass)--> LP_DEAD (3)
LP_DEAD (3) --index cleanup + 2nd pass--> LP_UNUSED (0)

-- table WITHOUT indexes: single heap pass
LP_NORMAL (1) --prune--> LP_UNUSED (0)

-- HOT
LP_NORMAL (1) --HOT prune (chain head)--> LP_REDIRECT (2)
LP_NORMAL (1) --HOT prune (heap-only chain member)--> LP_UNUSED (0)
LP_REDIRECT (2) --VACUUM (when target is also dead)--> LP_UNUSED (0)
```

Whether a deleted tuple passes through LP_DEAD depends on whether the table has indexes. With indexes, pruning can't free the slot (an index entry still references the TID), so it parks the line pointer at LP_DEAD, and only the second heap pass, after index cleanup, turns it into LP_UNUSED. Without indexes there is nothing to reference the TID, so pruning sets it straight to LP_UNUSED in a single pass. Either way the tuple's *storage* is reclaimed during the prune; LP_DEAD is only ever about the fate of the 4-byte slot.

## Free space map

Before VACUUM, PostgreSQL's free space map (FSM) had no idea that our page contained reclaimable space. The dead tuples were invisible to the FSM; they still looked like occupied bytes. After VACUUM, the reclaimed space is registered:

```
SELECT blkno, avail
FROM pg_freespace('vacuum_demo');
```

```
 blkno | avail
-------+-------
     0 |  3328
(1 row)
```

Page 0 now reports 3,328 bytes of available space. This is slightly less than the raw pd_upper - pd_lower value of 3,344 because the FSM tracks free space in coarse categories of roughly 32 bytes and rounds down. But the critical point is that this page is now a candidate for new INSERTs.

Without this FSM update, PostgreSQL would skip past this page when inserting new rows and extend the table file with a fresh page instead. That's how tables grow even when they contain plenty of free space internally; the FSM wasn't updated because VACUUM never ran.

```
-- New inserts reuse the vacuumed space instead of extending the file
INSERT INTO vacuum_demo (category, payload)
SELECT 'cat_new', repeat('y', 100)
FROM generate_series(1, 10) AS i;

SELECT pg_relation_size('vacuum_demo') AS size;
```

```
 size
------
 8192
(1 row)
```

The table is still a single 8 KB page. The ten new rows landed in the space VACUUM freed on page 0; the FSM pointed the inserter straight at it, so no new page was appended. Delete-then-insert without a VACUUM in between would tell a different story: the FSM would still report the page as full, the inserter would skip it, and the file would grow. That is the mechanism behind a table that keeps expanding on disk while sitting half empty.

## Visibility map

The visibility map tracks two bits per heap page. Those ten inserts we just did left fresh, not-yet-all-visible tuples on page 0, so the bit is currently clear. One more VACUUM settles the page, and now it's set:

```
VACUUM vacuum_demo;

SELECT blkno, all_visible, all_frozen
FROM pg_visibility('vacuum_demo');
```

```
 blkno | all_visible | all_frozen
-------+-------------+------------
     0 | t           | f
(1 row)
```

Page 0 is marked **all_visible**. This means every tuple currently on the page is visible to all transactions. The implications are significant:

- **Index-only scans** can return results from the index without fetching this heap page at all. The visibility map confirms that whatever is on this page, it's visible, so the index's copy of the data is guaranteed to be correct.
- **Future VACUUM passes** can potentially skip this page during the heap scan phase, since there's nothing dead to find.

The visibility map is stored as a separate fork file alongside the main heap file. For a table in `base/16384/24601`, the visibility map is `base/16384/24601_vm`. It's tiny: two bits per page means a 1 GB table (131,072 pages) needs only about 32 KB for its visibility map.

The **all_frozen** bit is still false. That flag indicates something stronger: not only are all tuples visible, but all of their xmin values have been frozen: they will survive XID wraparound without any further attention.

If we now delete a row from this page, the all_visible bit gets cleared:

```
DELETE FROM vacuum_demo WHERE id = 1;

SELECT blkno, all_visible, all_frozen
FROM pg_visibility('vacuum_demo');
```

```
 blkno | all_visible | all_frozen
-------+-------------+------------
     0 | f           | f
(1 row)
```

A single dead tuple on the page is enough to invalidate the all_visible flag. PostgreSQL clears this bit eagerly, at the moment of the DELETE, because it must be conservative. Index-only scans rely on this flag being correct.

## Freezing

Let's bring the page to its final resting state. Running `VACUUM FREEZE` forces PostgreSQL to freeze every tuple on the table, regardless of age:

```
VACUUM FREEZE vacuum_demo;
```

Now inspect the tuples:

```
SELECT lp, t_xmin, t_infomask,
    CASE WHEN (t_infomask & 256) > 0 AND (t_infomask & 512) > 0 THEN 'FROZEN'
         WHEN (t_infomask & 256) > 0 THEN 'XMIN_COMMITTED'
         ELSE 'no hint bits' END AS freeze_status
FROM heap_page_items(get_raw_page('vacuum_demo', 0))
WHERE lp_flags = 1
LIMIT 8;
```

```
 lp | t_xmin | t_infomask | freeze_status
----+--------+------------+---------------
  2 |    746 |       2818 | FROZEN
  3 |    748 |       2818 | FROZEN
  4 |    746 |       2818 | FROZEN
  5 |    746 |       2818 | FROZEN
  6 |    748 |       2818 | FROZEN
  7 |    746 |       2818 | FROZEN
  8 |    746 |       2818 | FROZEN
  9 |    748 |       2818 | FROZEN
(8 rows)
```

In older PostgreSQL versions (before 9.4), freezing literally rewrote `t_xmin` to 2 (FrozenTransactionId). Modern PostgreSQL preserves the original xmin value and sets the infomask bits instead. This is better for debugging: you can still see which transaction originally created the row.

Line pointer 1 is gone: that was `id = 1`, which we just deleted, and `VACUUM FREEZE` pruned it away before freezing the rest. Line pointers 3, 6, and 9 now carry `t_xmin = 748`: those are the `cat_new` rows from the free space map section, which reused the very slots the earlier VACUUM had freed. Every surviving tuple is FROZEN. The t_infomask value of 2818 is `0x0B02`, which is `HEAP_XMIN_COMMITTED` (0x0100) plus `HEAP_XMIN_INVALID` (0x0200) plus `HEAP_XMAX_INVALID` (0x0800), plus the `HEAP_HASVARWIDTH` attribute bit (0x0002) that every row here carries because of its `text` columns. As we covered in [PostgreSQL MVCC, Byte by Byte](https://boringsql.com/posts/postgresql-mvcc-byte-by-byte/), the combination of both XMIN_COMMITTED and XMIN_INVALID is the freeze marker. It means "this tuple's xmin is permanently in the past, regardless of what the XID counter does."

We can also verify the table-level freeze horizon:

```
SELECT relfrozenxid FROM pg_class WHERE relname = 'vacuum_demo';
```

```
 relfrozenxid
--------------
          750
(1 row)
```

This tells PostgreSQL: every tuple in `vacuum_demo` has an xmin that's either frozen or newer than XID 750. Autovacuum uses this value to decide when anti-wraparound freezing is needed. Since we just froze everything, this horizon advanced to the current transaction ID.

And the visibility map now shows both flags set:

```
SELECT blkno, all_visible, all_frozen
FROM pg_visibility('vacuum_demo');
```

```
 blkno | all_visible | all_frozen
-------+-------------+------------
     0 | t           | t
(1 row)
```

The **all_frozen** bit is now true. Future VACUUM passes, including anti-wraparound freezing passes, can skip this page entirely. There is nothing to clean and nothing to freeze. For large, mostly-static tables (lookup tables, historical data, archived partitions), this is a substantial performance benefit: VACUUM can skip millions of pages in seconds by checking two bits per page instead of reading every tuple.

## VACUUM FULL vs regular VACUUM

Everything we've seen so far is regular (lazy) VACUUM. There is a persistent myth that it can never shrink a table's file on disk. That's not quite right: at the very end of its run, lazy VACUUM tries to truncate trailing pages that have become completely empty, handing that space back to the operating system. What it *cannot* do is compact free space stranded in the interior of the file: half-empty pages surrounded by full ones. That distinction is the whole story, and both halves are easy to see.

Start with a table that fills 18 pages:

```
CREATE TABLE vt (id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY, category text, payload text);
INSERT INTO vt (category, payload)
SELECT 'cat_' || (i % 5), repeat('x', 100)
FROM generate_series(1, 1000) AS i;

SELECT pg_relation_size('vt') AS full_size;
```

```
 full_size
-----------
    147456
(1 row)
```

147,456 bytes is 18 pages. Now delete everything past the first fifty rows, which all live on page 0, and vacuum:

```
DELETE FROM vt WHERE id > 50;

VACUUM vt;
SELECT pg_relation_size('vt') AS after_regular_vacuum;
```

```
 after_regular_vacuum
----------------------
                 8192
(1 row)
```

Down to a single page. Every page after the first became completely empty, so those seventeen pages sat at the tail of the file with nothing live on them, and VACUUM's truncation phase returned them to the OS. Regular VACUUM *did* shrink the file, because the free space happened to be all at the end.

Now the case it can't help with. Rebuild the table and delete every *other* row, so no page ever fully empties:

```
DROP TABLE vt;
CREATE TABLE vt (id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY, category text, payload text);
INSERT INTO vt (category, payload)
SELECT 'cat_' || (i % 5), repeat('x', 100)
FROM generate_series(1, 1000) AS i;

DELETE FROM vt WHERE id % 2 = 0;

VACUUM vt;
SELECT pg_relation_size('vt') AS after_regular_vacuum;
```

```
 after_regular_vacuum
----------------------
               147456
(1 row)
```

Still 18 pages. Half the rows are gone and VACUUM registered all that free space in the FSM, but every page still holds live tuples, so there are no trailing empty pages to truncate. The file cannot shrink; the free space is stranded in the interior. This is exactly the bloat people mean when they say "I deleted half the table and it's still the same size on disk."

That is what VACUUM FULL is for:

```
VACUUM FULL vt;
SELECT pg_relation_size('vt') AS after_vacuum_full;
```

```
 after_vacuum_full
-------------------
             73728
(1 row)
```

147 KB down to 73 KB, 18 pages to 9. VACUUM FULL rewrote the entire table into a fresh file, packing the 500 surviving rows tightly and deleting the old file. Regular VACUUM had already made that space reusable, but only VACUUM FULL could compact it back out of the file.

VACUUM FULL requires an **AccessExclusiveLock**: it blocks ALL reads and writes on the table for the entire duration. For a large table, that can mean minutes or hours of complete downtime for that table. In production, consider [pg_repack or pg_squeeze](https://boringsql.com/posts/the-bloat-busters-pg_repack-pg_squeeze/) as online alternatives that achieve the same compaction without exclusive locks.

The tradeoff is clear: regular VACUUM is lightweight (ShareUpdateExclusiveLock, doesn't block reads or writes) and makes space reusable, truncating the file only when trailing pages are empty. VACUUM FULL compacts interior free space out of the file but takes an AccessExclusiveLock and blocks everything. For most workloads, regular VACUUM is sufficient: new INSERTs reuse the freed space, and the table reaches a steady-state size. VACUUM FULL is a remediation tool for when bloat has gotten out of hand.

## Putting it all together

Let's trace the complete lifecycle of a tuple from INSERT to VACUUM, with every state change visible:

**INSERT**

- Line pointer allocated as LP_NORMAL (1), pointing to the new tuple
- Tuple stamped with t_xmin = inserting XID, t_xmax = 0
- pd_lower advances by 4 bytes (new line pointer); pd_upper decreases by the tuple size

**DELETE**

- Tuple's t_xmax set to the deleting XID
- Visibility map all_visible bit cleared for the page
- No space reclaimed, no line pointer change, no pd_lower/pd_upper change

**VACUUM phase 1 (heap scan + prune)**

- Each page pruned: dead tuple data removed, page defragmented
- pd_upper increases; free space map updated
- Dead line pointers set to LP_DEAD (indexed table) or LP_UNUSED (no indexes)
- Dead TIDs collected; tuples opportunistically frozen

**VACUUM phase 2 (index cleanup)**

- Index entries pointing to dead TIDs removed from every index
- Heap pages untouched

**VACUUM phase 3 (heap cleanup, indexed tables only)**

- LP_DEAD line pointers flipped to LP_UNUSED (0), slots reusable
- No tuple bytes reclaimed here; pruning already did that in phase 1
- Visibility map all_visible bit set (if all remaining tuples are visible)

**VACUUM FREEZE**

- Tuple infomask bits set to XMIN_COMMITTED + XMIN_INVALID (frozen)
- Visibility map all_frozen bit set
- pg_class.relfrozenxid advanced

The line pointers move through the state machine we traced above (LP_NORMAL to LP_DEAD to LP_UNUSED on an indexed table, straight to LP_UNUSED without one), but the phase trace is where the *why* lives.

Every INSERT creates a line pointer. Every DELETE stamps t_xmax but changes nothing else on the page. And every VACUUM cycle (prune, index cleanup, heap cleanup) converts dead space back into reusable space. The page never grows beyond 8 KB, and VACUUM's job is to make sure as much of those 8,192 bytes as possible are available for live data.
