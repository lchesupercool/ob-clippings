---
title: Why Postgres Doesn't Have remote_receive - And What Happened When I Tried It
source: https://www.robins.in/2026/06/why-postgres-doesnt-have-remotereceive.html
author: Robins Tharakan
published: June 15, 2026
saved: 2026-06-16 14:31 +0800
tags:
- postgres
- replication
- benchmarking
- database
---

# Why Postgres Doesn't Have remote_receive - And What Happened When I Tried It

Source: [https://www.robins.in/2026/06/why-postgres-doesnt-have-remotereceive.html](https://www.robins.in/2026/06/why-postgres-doesnt-have-remotereceive.html)  
Author: Robins Tharakan  
Published: June 15, 2026  
Saved: 2026-06-16 14:31 +0800

In distributed database environments, balancing durability and performance is a constant tug-of-war. PostgreSQL’s `synchronous_commit` parameter sits at the heart of this, giving administrators a dial to choose exactly when a `COMMIT` returns success to the client.

The idea of `remote_receive` was born from a simple question: does skipping the standby's disk write yield a measurable, real-world performance benefit? By waiting only for WAL bytes to reach the standby's memory, could we get a meaningful boost over `remote_write`? I set out to implement and benchmark this feature to find out.

What followed was a journey through network latency, OS page caches, CPU scheduler thrashing, and benchmarking noise. Here is the breakdown of the implementation, the tests, the initial anomalies, and the final results.

---

## 1. The Feature: What is `remote_receive`?

Before this branch, PostgreSQL offered four primary synchronous commit modes:

- **`off`**: Fully asynchronous. (Fastest, least safe)
- **`local`**: Waits for local disk flush on the primary.
- **`remote_write`**: Waits for the standby to write the WAL to its OS buffer cache (`pwrite`).
- **`remote_apply`**: Waits for the standby to fully replay the WAL. (Slowest, most safe)

`remote_receive` sits directly between `local` and `remote_write`. In this mode, the primary guarantees that the WAL bytes have physically arrived at the standby's `walreceiver` process buffer. It does *not* wait for the standby to call `pwrite()`.

**The Hypothesis:** By completely bypassing the standby's disk I/O, `remote_receive` should deliver lower latency and higher throughput than `remote_write`, especially on replica hardware with slow disks.

### Implementation Details

To build this, I had to modify both the standby and the primary:

1. **Standby Status Update:** I modified the 34-byte wire message that the standby sends to the primary, adding a new 8-byte `receivePtr` (creating a 42-byte message, backward compatible).
2. **Early Replies:** I modified `walreceiver.c` to send a reply message *immediately* upon receiving a WAL chunk in memory, before the `XLogWalRcvWrite()` call executes the `pg_pwrite`.
3. **Primary Wait Logic:** I updated `syncrep.c` and `walsender.c` to track the `SYNC_REP_WAIT_RECEIVE` wait queue, releasing waiting backends as soon as the standby's `receivePtr` advanced.

---

## 2. Scenario 1: The SSD Baseline (Fast Primary, Fast Replica)

To validate the code, I first set up a baseline test using two fast machines on a gigabit LAN.

**The Servers:**

- **Primary:** `lenovo` (Intel Core i7-12700 12C/20T, 48GB RAM, NVMe SSD)
- **Replica:** `camry` (Intel Core i7-4770 4C/8T, 24GB RAM, SATA SSD)

I ran pgbench (Scale 10, 4 clients) for 30 seconds across the different modes.

**The Result:**

- `remote_write`: 3,944 TPS (Median)
- `remote_receive`: 3,946 TPS (Median)

The performance was virtually identical (a 0.06% difference). Why didn't `remote_receive` pull ahead?

**The Reality of `pwrite()`:** On modern operating systems with free RAM, a standard `pwrite()` to a buffered file does not write to physical disk immediately. It copies the data into the OS page cache (essentially a memory copy taking mere microseconds), leaving the kernel to flush dirty pages asynchronously.

With network round-trip time (RTT) on a gigabit LAN between 0.2ms and 1.0ms, the 5 microseconds saved by bypassing `pwrite()` is completely dwarfed by network latency. This makes `remote_write` and `remote_receive` perform nearly identically in typical conditions.

This fundamental reality explains why core PostgreSQL developers historically questioned the return on investment (RoI) of a memory-only receive mode. Because a standard `pwrite()` to the OS page cache is already a RAM-speed operation (taking mere microseconds), `remote_write` is practically as fast as any network-receive-only mode under normal conditions, but with a major durability advantage: it survives a PostgreSQL process crash on the standby (as long as the OS remains running). Bypassing it would reduce durability without providing any real-world performance benefit in typical conditions. In pgsql-hackers discussions—such as the thread on [Measuring Replay Lag](https://www.postgresql.org/message-id/flat/CAEepm%3D2%3DjCZ35%2BmhRYE1yN%2BF97p5BYo%2BV%2BKx71Wu2xKaa3dQDg%40mail.gmail.com#234c830ffef92c855be4b7348d20ab54) where distinct `write_lag` and `flush_lag` tracking was introduced—it is clear that the network round-trip time dominates the replication pipeline. To find a measurable benefit for a receive-only mode, I needed a replica where disk I/O was a severe enough bottleneck to cause page cache pressure and slow down the `pwrite()` call itself.

---

## 3. Scenario 2: The HDD Challenge (Asymmetric Hardware)

For the next test, I replaced the fast `camry` replica with a much weaker machine.

**The Servers:**

- **Primary:** `lenovo` (i7-12700, SSD)
- **Replica:** `mac` (Intel Core i5-2520M 2C/4T, 16GB RAM, 5400RPM HDD)

With a mechanical hard drive and a slow dual-core processor, I expected `remote_receive` to outpace `remote_write`. I ran three 30-second runs per mode, but the initial results were unexpected:

**The First (Anomalous) Result:**

- `remote_write`: 203.6 TPS (Median)
- `remote_receive`: 179.0 TPS (Median)

`remote_write` was ~20 TPS faster than `remote_receive`. Tracing the `walreceiver` and `walsender` loops ruled out code bugs; instead, the bottleneck came down to four factors:

1. **The OS Cache Illusion:** Even on a slow HDD, `pwrite()` still writes to RAM. The mechanical disk's extreme latency only hits during `fsync` or when the page cache fills up, meaning the receive-only advantage remained small.
2. **CPU/Scheduler Thrashing:** By sending an "early reply" before writing to disk, `remote_receive` generates twice as many TCP reply packets (one for receive, one later for flush). On the replica's older dual-core CPU, processing this packet flood alongside WAL replay caused high context-switching overhead.
3. **Flow Control:** `remote_write` acted as a natural flow-control mechanism. By waiting to write before replying, it throttled the primary slightly, keeping the replica's CPU out of a thrashing state.
4. **Statistical Noise:** The `remote_write` runs ranged from 177 to 211 TPS (a 19% spread). A 3-run, 30-second test was simply too noisy to yield a reliable median.

> **Aside: The Raspberry Pi 4 Attempt**
> Before settling on the Mac Mini HDD, I attempted to use a Raspberry Pi 4 (`pi4` — Cortex-A72 4C/4T, 4GB RAM, SD card / USB storage) as the slow replica. However, the Pi 4 was a poor fit for this benchmark. The main issue wasn't simply that the CPU maxed out, but that the low-power ARM CPU could not keep pace with the primary's faster rate of WAL generation. This lag cascaded into secondary issues—such as rapidly mounting replication lag, TCP buffer queues, and process starvation—which completely dominated the environment and masked any storage-level performance differences.

---

## 4. The Final Test: Eliminating the Noise

In the initial runs, the slow mechanical disk combined with standard kernel buffering created a massive source of noise: the sustained background I/O writes from a just-finished test were still flushing to disk when the next test began. Although an LSN cross-check was already in place (waiting for the replica's `replay_lsn` to catch up to the primary's current WAL LSN), this only verified database-level catch-up, not physical disk queue clearance. The residual write queue in the OS cache severely penalized the subsequent run, creating artificial variance. To isolate the actual replication performance and eliminate this noise, I overhauled the benchmarking methodology:

1. **Interleaved Runs:** I interleaved executions (Write, Receive, Write, Receive...) to average out temporal background OS tasks and thermal states.
2. **Longer Runs and More Iterations:** I ran 10 iterations per mode at 60 seconds per run (20 runs total).
3. **Aggressive Cache Flushing:** I ran `sync` on both the primary and replica between runs, followed by a 30-second sleep, to flush the OS page cache to physical disk platters and guarantee clean disk queues.

**The Final Results:**

| Mode | Median TPS | Mean TPS | Median Latency |
| --- | --- | --- | --- |
| `remote_write` | 201.58 | 200.74 | 20.663 ms |
| `remote_receive` | 211.44 | 206.85 | 19.434 ms |

### The Results

With the noise eliminated and disk queues flushed, the true behavior emerged: **`remote_receive` outperformed `remote_write`** by **~10 TPS (~4.9% gain)** at the median and **~6 TPS (~3.0% gain)** at the mean. The small gap is expected: bypassing a RAM-buffered `pwrite()` on the replica yields microsecond-level gains, while network round-trip time remains the dominant factor.

## Conclusion

The `remote_receive` implementation successfully introduces a granular durability option that guarantees WAL has crossed the network into the standby's memory before committing.

This exercise highlighted a key rule of database benchmarking: **the OS page cache masks physical disk latency until it fills up.** To accurately benchmark high-variance storage, one must use interleaved runs, higher iteration counts, and rigorous OS-level cache flushes between tests. Otherwise, you are measuring cache behavior and statistical noise rather than raw database throughput.

---

***A Note on the Development Process & Resources:** While I dabble in C and C++ from time to time, development at this level within PostgreSQL internals would not have been possible without AI assistance. Additionally, all resources for this project—including the Claude subscription, development machines, laptop, and time—were entirely personal and completely independent of my employer.*
