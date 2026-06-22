---
title: 为什么 Postgres 没有 remote_receive —— 以及我尝试实现它之后发生了什么
source: https://www.robins.in/2026/06/why-postgres-doesnt-have-remotereceive.html
author: Robins Tharakan
published: June 15, 2026
saved: 2026-06-16 14:31 +0800
tags:
- postgres
- replication
- benchmarking
- database
translation_of: 2026-06-16-why-postgres-doesnt-have-remotereceive.md
---

# 为什么 Postgres 没有 remote_receive —— 以及我尝试实现它之后发生了什么

Source: [https://www.robins.in/2026/06/why-postgres-doesnt-have-remotereceive.html](https://www.robins.in/2026/06/why-postgres-doesnt-have-remotereceive.html)  
Author: Robins Tharakan  
Published: June 15, 2026  
Saved: 2026-06-16 14:31 +0800

在分布式数据库环境里，如何在持久性和性能之间取得平衡，一直是一场拉锯战。PostgreSQL 的 `synchronous_commit` 参数正处在这个问题的核心位置：它给管理员一个旋钮，用来精确选择 `COMMIT` 在什么时候向客户端返回成功。

`remote_receive` 这个想法来自一个简单的问题：如果跳过 standby 上的磁盘写入，是否能带来可测量的、真实世界里的性能收益？如果只等待 WAL 字节到达 standby 的内存，而不是等待它进入 `remote_write` 所要求的位置，我们能否得到有意义的提升？我决定实现并 benchmark 这个特性来找出答案。

接下来发生的是一段穿过网络延迟、OS page cache、CPU 调度器抖动以及 benchmark 噪声的旅程。下面是实现、测试、初始异常现象以及最终结果的拆解。

---

## 1. 这个特性：什么是 `remote_receive`？

在这个分支之前，PostgreSQL 提供了四种主要的同步提交模式：

- **`off`**：完全异步。（最快，最不安全）
- **`local`**：等待 primary 上的本地磁盘 flush。
- **`remote_write`**：等待 standby 将 WAL 写入它的 OS buffer cache（`pwrite`）。
- **`remote_apply`**：等待 standby 完整 replay WAL。（最慢，最安全）

`remote_receive` 正好位于 `local` 和 `remote_write` 之间。在这个模式下，primary 保证 WAL 字节已经物理到达 standby 的 `walreceiver` 进程 buffer。它*不会*等待 standby 调用 `pwrite()`。

**假设：**通过完全绕过 standby 的磁盘 I/O，`remote_receive` 应该比 `remote_write` 提供更低延迟和更高吞吐，尤其是在 replica 硬件磁盘较慢时。

### 实现细节

为了构建这个功能，我必须同时修改 standby 和 primary：

1. **Standby 状态更新：**我修改了 standby 发送给 primary 的 34 字节 wire message，增加了一个新的 8 字节 `receivePtr`（形成 42 字节消息，并保持向后兼容）。
2. **提前回复：**我修改了 `walreceiver.c`，让它在内存中收到 WAL chunk 后*立即*发送 reply message，发生在 `XLogWalRcvWrite()` 调用执行 `pg_pwrite` 之前。
3. **Primary 等待逻辑：**我更新了 `syncrep.c` 和 `walsender.c`，用于跟踪 `SYNC_REP_WAIT_RECEIVE` wait queue；一旦 standby 的 `receivePtr` 前进，就释放正在等待的 backend。

---

## 2. 场景 1：SSD 基线（快速 primary，快速 replica）

为了验证代码，我先用千兆 LAN 上的两台快速机器搭建了基线测试。

**服务器：**

- **Primary:** `lenovo`（Intel Core i7-12700 12C/20T，48GB RAM，NVMe SSD）
- **Replica:** `camry`（Intel Core i7-4770 4C/8T，24GB RAM，SATA SSD）

我使用 pgbench（Scale 10，4 clients）在不同模式下各运行 30 秒。

**结果：**

- `remote_write`：3,944 TPS（中位数）
- `remote_receive`：3,946 TPS（中位数）

性能几乎完全相同（差异为 0.06%）。为什么 `remote_receive` 没有领先？

**`pwrite()` 的现实：**在有空闲 RAM 的现代操作系统上，对 buffered file 执行普通 `pwrite()` 并不会立刻写入物理磁盘。它会把数据复制到 OS page cache 中（本质上是一次只需要几微秒的内存拷贝），然后由内核异步 flush dirty pages。

在千兆 LAN 上，网络 round-trip time（RTT）大约在 0.2ms 到 1.0ms 之间；绕过 `pwrite()` 省下来的 5 微秒会被网络延迟完全淹没。这使得 `remote_write` 和 `remote_receive` 在典型条件下表现几乎相同。

这个基本事实解释了为什么 PostgreSQL core 开发者历史上会质疑 memory-only receive mode 的投入产出比（RoI）。因为普通 `pwrite()` 写入 OS page cache 已经是 RAM 速度的操作（只需要几微秒），所以在正常条件下，`remote_write` 实际上已经和任何只等待网络接收的模式一样快，但它还有一个重要的持久性优势：只要 OS 仍在运行，它可以在 standby 上的 PostgreSQL 进程崩溃后存活。绕过它会降低持久性，却不会在典型条件下提供任何真实世界的性能收益。在 pgsql-hackers 讨论中——例如引入独立 `write_lag` 和 `flush_lag` 跟踪的 [Measuring Replay Lag](https://www.postgresql.org/message-id/flat/CAEepm%3D2%3DjCZ35%2BmhRYE1yN%2BF97p5BYo%2BV%2BKx71Wu2xKaa3dQDg%40mail.gmail.com#234c830ffef92c855be4b7348d20ab54) 线程——可以清楚看到，网络往返时间主导了复制 pipeline。要找到 receive-only 模式的可测量收益，我需要一个 replica，在那里磁盘 I/O 严重到足以成为瓶颈，造成 page cache 压力并拖慢 `pwrite()` 调用本身。

---

## 3. 场景 2：HDD 挑战（非对称硬件）

下一次测试中，我把快速的 `camry` replica 换成了一台弱得多的机器。

**服务器：**

- **Primary:** `lenovo`（i7-12700，SSD）
- **Replica:** `mac`（Intel Core i5-2520M 2C/4T，16GB RAM，5400RPM HDD）

由于使用机械硬盘和较慢的双核处理器，我原本预期 `remote_receive` 会超过 `remote_write`。我对每种模式运行了三次 30 秒测试，但初始结果出乎意料：

**第一次（异常）结果：**

- `remote_write`：203.6 TPS（中位数）
- `remote_receive`：179.0 TPS（中位数）

`remote_write` 比 `remote_receive` 快了约 20 TPS。跟踪 `walreceiver` 和 `walsender` 循环排除了代码 bug；瓶颈最终来自四个因素：

1. **OS cache 幻觉：**即使在慢 HDD 上，`pwrite()` 仍然是写入 RAM。机械硬盘的极端延迟只会在 `fsync` 期间或 page cache 被填满时体现出来，这意味着 receive-only 的优势仍然很小。
2. **CPU/调度器抖动：**通过在写磁盘前发送“提前回复”，`remote_receive` 会产生两倍数量的 TCP reply packet（一次用于 receive，稍后一次用于 flush）。在 replica 那颗较老的双核 CPU 上，一边处理这些 packet 洪流，一边进行 WAL replay，会导致很高的 context-switching 开销。
3. **流控：**`remote_write` 起到了天然的 flow-control 机制作用。它在回复前等待写入，从而轻微 throttle primary，让 replica 的 CPU 避免进入抖动状态。
4. **统计噪声：**`remote_write` 的运行结果范围从 177 到 211 TPS（19% 的跨度）。3 次、每次 30 秒的测试实在太 noisy，无法得到可靠的中位数。

> **插曲：Raspberry Pi 4 尝试**
> 在最终选择 Mac Mini HDD 之前，我尝试使用 Raspberry Pi 4（`pi4` —— Cortex-A72 4C/4T，4GB RAM，SD card / USB storage）作为慢 replica。然而，Pi 4 并不适合这个 benchmark。主要问题不只是 CPU 打满，而是低功耗 ARM CPU 无法跟上 primary 更快的 WAL 生成速率。这个滞后会级联出次生问题——例如快速累积的 replication lag、TCP buffer queue、进程饥饿——这些因素完全主导了环境，并掩盖了任何 storage-level 的性能差异。

---

## 4. 最终测试：消除噪声

在初始运行中，慢机械硬盘与标准 kernel buffering 结合，产生了巨大的噪声来源：刚刚结束的测试留下的持续后台 I/O 写入，在下一次测试开始时仍然在 flush 到磁盘。虽然我已经有 LSN cross-check（等待 replica 的 `replay_lsn` 追上 primary 当前 WAL LSN），但这只能验证数据库层面的追平，并不能验证物理磁盘队列是否清空。OS cache 中残留的写队列会严重惩罚下一次运行，制造人为方差。为了隔离真正的复制性能并消除噪声，我彻底调整了 benchmark 方法：

1. **交错运行：**我交错执行测试（Write、Receive、Write、Receive……），以平均掉时间相关的后台 OS 任务和热状态。
2. **更长运行和更多迭代：**我对每种模式运行 10 次，每次 60 秒（总计 20 次运行）。
3. **激进 cache flush：**我在每次运行之间，在 primary 和 replica 上都执行 `sync`，随后 sleep 30 秒，以便把 OS page cache flush 到物理磁盘盘片，并确保磁盘队列干净。

**最终结果：**

| Mode | Median TPS | Mean TPS | Median Latency |
| --- | --- | --- | --- |
| `remote_write` | 201.58 | 200.74 | 20.663 ms |
| `remote_receive` | 211.44 | 206.85 | 19.434 ms |

### 结果

在消除噪声并 flush 磁盘队列后，真实行为浮现出来：**`remote_receive` 确实优于 `remote_write`**，中位数上提升约 **10 TPS（约 4.9%）**，均值上提升约 **6 TPS（约 3.0%）**。这个差距很小，并且符合预期：绕过 replica 上的 RAM-buffered `pwrite()` 只能带来微秒级收益，而网络 round-trip time 仍然是主导因素。

## 结论

`remote_receive` 实现成功引入了一种更细粒度的持久性选项：它保证 WAL 在提交前已经跨过网络进入 standby 的内存。

这次实验强调了数据库 benchmark 的一个关键规则：**OS page cache 会掩盖物理磁盘延迟，直到它被填满。**要准确 benchmark 高方差存储，必须使用交错运行、更多迭代，以及在测试之间进行严格的 OS-level cache flush。否则，你测到的是 cache 行为和统计噪声，而不是原始数据库吞吐。

---

***关于开发过程和资源的说明：**虽然我偶尔会写 C 和 C++，但如果没有 AI 帮助，我不可能在 PostgreSQL internals 这个层级完成开发。此外，这个项目使用的所有资源——包括 Claude 订阅、开发机器、笔记本电脑和时间——都完全来自个人，与我的雇主完全无关。*
