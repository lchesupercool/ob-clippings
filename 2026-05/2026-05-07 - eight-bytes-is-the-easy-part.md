---
title: "Eight Bytes Is the Easy Part"
source: "https://thebuild.com/blog/2026/05/07/eight-bytes-is-the-easy-part/"
author: "thebuild.com"
published: "2026-05-07"
clipped: "2026-05-08T14:38:18+08:00"
tags:
  - clipping
  - PostgreSQL
---

# Eight Bytes Is the Easy Part

Source: [https://thebuild.com/blog/2026/05/07/eight-bytes-is-the-easy-part/](https://thebuild.com/blog/2026/05/07/eight-bytes-is-the-easy-part/)  
Published: 2026-05-07  
Category: PostgreSQL

![](../../assets/eight-bytes-is-the-easy-part/eight-bytes-is-the-easy-part-01.png)

[PostgreSQL 19 widens `MultiXactOffset` to 64 bits,](https://thebuild.com/blog/2026/05/06/multixact-members-at-64-bits-one-less-wraparound-to-worry-about/) retiring one of the more interesting failure modes the database can produce in production. The members-space wraparound on the multixact SLRU was a genuine outage class — Metronome ate four of them in a single migration last May — and it is gone in 19. Good. Onward.

The obvious next question, asked roughly once a month somewhere on `pgsql-hackers` since approximately the Bush administration, is: when do regular transaction IDs go to 64 bits?

The answer is “not soon,” and the rest of this post is the long version of why.

## The obvious cost

Every heap tuple carries `xmin` and `xmax` in its header, both `TransactionId`, both currently `uint32`. Widening them to 64 bits adds eight bytes to every tuple. On a wide row that’s a rounding error. On a narrow row — a join table, a bookkeeping table, a queue — it is not. A two-column `(int, int)` row is 32 bytes today including alignment padding; add eight bytes of header and the table inflates by 25%, plus the matching cost in every index that points to it.

Existing clusters cannot just opt in: the on-page format itself would have to change.

That’s the part everyone sees. It is not the hard part.

## There Goes `pg_upgrade`

The reason `pg_upgrade` is fast — and the reason most major version upgrades take minutes instead of hours — is that the on-disk heap format almost never changes. `pg_upgrade --link` simply hard-links the existing data files into the new cluster, replays the system catalogs, and you’re done. The actual user data is never read, never written, never copied.

A change to the tuple header breaks that. `pg_upgrade --link` is no longer an option, because the new server cannot read pages produced by the old one. You are now in `pg_upgrade --copy` territory, which means every byte of every relation in the cluster gets read, transformed, and written back. On a small database, this is a long lunch. On a 30TB cluster, it is a multi-day downtime event, and that’s assuming the I/O subsystem cooperates.

The alternatives are worse or weirder. `pg_dump`/`pg_restore` is `--copy` plus a logical bottleneck. Logical replication-based upgrades — stand up a parallel cluster on the new version, replicate into it, switch over — are how large shops will actually do it, but the operational footprint is real: double the hardware for the duration, dual-write windows, careful handling of sequences and large objects, and a cutover plan that nobody enjoys writing.

This is qualitatively different from any PostgreSQL major upgrade in living memory. `pg_upgrade` made the “stay on the supported version” treadmill operationally cheap. A 64-bit-XID upgrade puts that treadmill back to where it was in the 8.x days, when major upgrades meant `pg_dump` and a long weekend.

## And there go the pushbutton upgrades

Every hosted PostgreSQL provider I’m aware of — RDS, Aurora, Cloud SQL, Azure Database for PostgreSQL, Supabase, Neon, the rest — implements its “upgrade to the next major version” button on top of `pg_upgrade --link`. That’s why the button exists at all. It’s also why the documented downtime for a major version upgrade on a 10TB RDS instance is measured in minutes rather than hours.

Break the on-disk format and that button breaks with it. The provider’s options are the same as everyone else’s: full copy, logical replication into a parallel cluster, or a `pg_dump`/`pg_restore` cycle. None of those are pushbutton. All of them require the customer to plan a real maintenance window, often with a parallel instance running for the duration. For a small database, this might mean an hour of downtime where today there’s five minutes. For a large one, it’s days, and the migration becomes a project rather than a checkbox.

This is the part that will determine when 64-bit XIDs actually ship: not the code, but the upgrade story. The community has spent a decade not landing this feature largely because nobody has been willing to tell every PostgreSQL operator on earth that their next major upgrade is a multi-hour outage.

## Snapshots

A snapshot is the runtime expression of MVCC: an `xmin`, an `xmax`, and an array of XIDs that were in flight at snapshot acquisition. The array lives in the procarray, gets copied into every transaction’s local snapshot, gets serialized into prepared transaction state on disk, gets shipped to logical replication subscribers, and gets exchanged between primary and standby for hot-standby visibility.

Doubling the width of every entry doubles the size of every snapshot. Snapshots are taken on every read; the procarray scan is already a known scaling bottleneck on high-connection workloads, and the cache footprint of snapshot acquisition is part of why. So is the on-wire size when standbys consume the snapshot stream, and so is the size of the in-progress XID list in WAL records that carry snapshot information for logical decoding.

This is not a fatal problem. It is a death-by-a-thousand-paper-cuts problem, which is worse, because every benchmark regresses by a small amount and someone has to defend each one.

## The SLRUs

`pg_xact` (formerly `pg_clog`), `pg_subtrans`, and `pg_commit_ts` are all addressed by transaction ID. The on-disk format assumes the XID space fits into a manageable number of 256KB SLRU segments — roughly a gigabyte for the entire 32-bit range of `pg_xact` at two bits per transaction. A 64-bit address space is not larger by a factor of two; it is larger by a factor of four billion.

You don’t have to materialize all of it. You only have to address it. But every SLRU function — segment naming, page lookup, truncation, the entire lifecycle around vacuum freezing — assumes XIDs index into a closed cyclic space. Truncation in particular is interesting: today, `pg_xact` is truncated behind the oldest live XID using the same wraparound arithmetic the rest of the system uses. Lift the ceiling and you’ve changed what truncation even means.

## The cyclic comparison primitive

`TransactionIdPrecedes` is one line of C: it subtracts two `int32` values and checks the sign. This works because 32-bit XIDs wrap, and the comparison is implicitly modulo 2^32. The fact that it works at all is the load-bearing piece of how PostgreSQL avoids needing a global total ordering over every transaction ever recorded.

Move to 64 bits and the comparison becomes a normal less-than. Strictly easier. Except that every callsite assumes the cyclic semantics — the freeze horizon arithmetic, the `pg_xact` truncation logic, the `FrozenTransactionId` sentinel, the `MultiXactIdPrecedes` family (which inherits the same idiom), and a long tail of comparisons in extensions written against the C API. Each one has to be audited. Most are correct under either semantics. Some are not.

## The on-disk format problem, again

A 64-bit XID changes the heap tuple header. It also changes the `HeapTupleHeaderData` C struct, which is part of the public extension ABI. Every extension that introspects a tuple — and there are a lot of them, including some you depend on without realizing it: `postgis`, `citus`, `timescaledb`, `pg_repack`, every FDW that builds heap tuples — has to be recompiled. Most have to be modified. Some have to be substantially rewritten.

Physical replication assumes byte-identical pages between primary and standby. A 64-bit-XID primary cannot replicate to a 32-bit-XID standby, and there is no bridge. You upgrade the entire replication topology in one motion or you don’t upgrade at all. For shops running large global standby fleets, this is operationally painful in a way that “rewrite every page” undersells.

Logical replication has slightly more flexibility — it’s a logical protocol — but the protocol itself carries XIDs, and emitting a 64-bit XID to a 32-bit subscriber is not a defined operation.

## Why `MultiXactOffset` was easier

`MultiXactOffset` is a pointer into the members array of the multixact SLRU. It is not stored in tuple headers. It is not transmitted in snapshots. It is not part of the public C ABI in any way that downstream code reaches into. It lives entirely inside `pg_multixact`, gets read and written by a small number of functions in `multixact.c`, and gets WAL-logged in a handful of record types.

Widening it required changing the SLRU file format and adding a `pg_upgrade` step that rewrites `pg_multixact` (which is, in absolute terms, a small directory — gigabytes, not terabytes). It did not require touching heap pages, snapshots, the procarray, the visibility map, the freeze logic, or any extension. The blast radius was small.

The XID widening blast radius is “everything.”

## What’s actually being worked on

The serious proposals in this space don’t widen the on-page XID. They keep `xmin` and `xmax` 32-bit on disk and add an **epoch** somewhere — either per-page, per-relation, or implicit in the freeze horizon — so that the effective XID is 64-bit while the storage remains 32-bit. This is the “64-bit XID via epoch” approach that has shown up in various forms from Postgres Pro and others over the years. It avoids the heap rewrite. It avoids the ABI break for tuple introspection, since the on-page format is unchanged. It still requires changing the snapshot representation, the SLRU addressing, and the cyclic comparison semantics — but the single most expensive change, the heap format, is sidestepped.

The cost is complexity. Every tuple visibility check has to compose the on-page 32-bit XID with the relevant epoch to get the comparable 64-bit value. Freezing becomes a thing that updates the epoch rather than the XID itself. Anti-wraparound vacuum doesn’t go away; its character just changes. Instead of “freeze before you wrap,” it becomes “freeze before the epoch overflows the per-page room.” Different problem, different operational signature, still a problem.

## What to expect

Do not expect 64-bit XIDs in PostgreSQL 20. The `MultiXactOffset` patch took years of iteration to land, and that one had a tractable scope. The XID work has been in flight, in various forms, for over a decade, and the reason it hasn’t shipped isn’t that nobody has tried. It’s that every attempt has surfaced a new corner of the system that turned out to depend on the 32-bit assumption in a non-obvious way.

The version that eventually lands will almost certainly be the epoch approach, and it will probably arrive piecemeal: the in-memory representation first, then the SLRU layer, then a way to upgrade existing clusters, then the actual XID widening as the last domino. We’ll know it’s close when somebody proposes a `pg_upgrade` flag for it.

In the meantime: monitor `age(datfrozenxid)`, freeze aggressively on hot tables, and be glad the multixact members ceiling is gone.
