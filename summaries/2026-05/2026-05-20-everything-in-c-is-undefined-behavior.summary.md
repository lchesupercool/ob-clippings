---
title: "Everything in C is undefined behavior — 总结"
source: "https://blog.habets.se/2026/05/Everything-in-C-is-undefined-behavior.html"
summary_of: "../2026-05/2026-05-20-everything-in-c-is-undefined-behavior.md"
saved: "2026-05-20 20:47:10 +0800"
tags: [c, cpp, undefined-behavior, programming, security, summary]
---

# Everything in C is undefined behavior — 总结

原文观点很尖锐：到 2026 年，继续在关键代码里裸写 C/C++，却没有 LLM 或其他强力工具持续审查 undefined behavior（UB），已经接近“不负责任”。作者不是反 C/C++ 的外行，而是一个写了约 30 年 C/C++、仍然喜欢 C++ 的工程师。他的核心判断是：不是程序员太菜，而是 C/C++ 的 UB 太多、太细、太反直觉，任何非平凡代码几乎都不可避免。

## 核心论点

- UB 不是“开优化才会被编译器钻空子”。UB 的真实含义是：编译器和硬件可以假设这种情况不会发生，因此很多意图在编译器阶段之间根本没有表达方式。
- C/C++ 的问题不只是不内存安全。大家熟悉的 double-free、use-after-free、越界、未初始化读只是表层；更麻烦的是大量合法外观代码也可能触发 UB。
- 现代软件环境已经和 C/C++ 诞生时代完全不同。1972/1985 年的抽象机器假设，在今天的多架构、多优化、多安全需求环境下越来越不适配。
- “能在 x86 上跑”不能证明代码正确。x86 太宽容，很多错误在 SPARC、Alpha、ARM、RISC-V 或未来架构上可能崩溃，或被编译器换一种指令后突然暴露。

## 文中举的典型 UB

1. **未对齐访问**：`int*` 指针如果没有按 `int` 要求对齐，解引用就是 UB。x86 上可能没事，SPARC 可能 SIGBUS，Alpha 有时由内核模拟，有时崩溃。
2. **创建错误类型指针**：把网络包字节 `uint8_t*` 直接 cast 成 `int*`，问题不一定发生在解引用时，cast 本身就可能违反标准假设。
3. **`isxdigit(char)` 坑**：`isxdigit()` 只保证接受 `unsigned char` 可表示的值或 `EOF`。如果 `char` 是 signed，高位字节会变成负数，可能触发 UB。
4. **float 转 int**：浮点值超出目标整数范围，或是 NaN/Inf，转换为整数就是 UB；简单的“秒转毫秒”也可能出错。
5. **地址 0 与 NULL**：C 标准不保证 null pointer 的机器表示就是全 0；`memset(&ptr, 0, sizeof(ptr))` 也不严格保证得到 null pointer。
6. **可变参数类型不匹配**：`execl(..., NULL)` 需要显式 `(char*)NULL`；`printf` 打印 `uint64_t` 应使用 `PRIu64`，格式不匹配也是 UB。
7. **除零**：分母常来自不可信输入，所以除零不仅是崩溃问题，也是安全问题。

## 额外提醒

作者还提到整数提升规则：即使不是 UB，人类也很难在扫代码时快速正确判断结果。例如 `unsigned char` 运算会先提升为 `int`，导致溢出判断和位移结果都可能和直觉不一致。

## 作者对 LLM 的看法

作者认为 LLM 很适合做 UB 审查：给它任何 C 代码，让它找 UB，通常能找出不少，而且大多是对的。他甚至让 LLM 看 OpenBSD 的 `find`，找到了越界写并提交过补丁。

但 LLM 不能直接替代人类合入代码。UB 修复需要专家确认，因为这类问题太微妙；同时专家又很忙，所以这是“像清洁工活一样琐碎，但又不能交给初级程序员”的尴尬工作。

## 对工程实践的启发

- C/C++ 项目应把 UB 扫描当成持续工程流程，而不是偶尔 code review。
- LLM 可以作为第一轮 UB 猎手，但输出必须由懂标准、懂平台、懂代码语义的人复核。
- 网络协议解析、序列化、浮点转整数、可变参数、ctype、跨平台底层代码，是优先审查区域。
- “在我的机器上没问题”尤其不可信，因为 x86 会掩盖大量问题。
- 如果是关键系统，继续大量写 C/C++ 却没有自动化 UB 审查，风险已经很难辩护。

## 一句话

这篇文章不是在说 C/C++ 不能用，而是在说：C/C++ 的抽象机器和现代工程现实之间裂缝太大，UB 已经多到超出人类常规 review 能力；未来可行路线不是相信高手手写正确 C，而是用 LLM/静态分析/专家复核组合来系统性压低 UB 风险。
