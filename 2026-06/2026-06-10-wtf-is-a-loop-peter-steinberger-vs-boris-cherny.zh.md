---
title: "WTF 是 Loop？Peter Steinberger vs. Boris Cherny"
author: "Matt Van Horn"
source: "https://x.com/mvanhorn/status/2063865685558903149?s=20"
canonical: "https://www.linkedin.com/pulse/wtf-loop-peter-steinberger-vs-boris-cherny-matt-van-horn-cpslc"
published: "2026-06-08"
saved: "2026-06-10 16:03:08 CST"
tags: [clipping, x-twitter, linkedin, ai-coding, agents]
translation_of: "2026-06-10-wtf-is-a-loop-peter-steinberger-vs-boris-cherny.md"
---

# WTF 是 Loop？Peter Steinberger vs. Boris Cherny

Matt Van Horn  
发布：2026-06-08

![WTF 是 Loop？Peter Steinberger vs. Boris Cherny](../../assets/wtf-is-a-loop-peter-steinberger-vs-boris-cherny/cover.jpg)

本周 AI coding 圈被反复提起最多的一句话只有六个词，但几乎没人能把它定义清楚。有一条推文让整个时间线都被它牵着走，所以我用 /last30days 跑了一遍大家争论的这个词。答案是真的，它有一条五年的谱系，而且关键结论是：现在昂贵的部分不再是模型，而是 loop。

## 让整个时间线着迷的那条推文

本周有一条推文让整个 AI coding 时间线都陷入沉迷。Peter Steinberger 在 6 月 7 日发了它，浏览量超过 220 万，回复区随即变成了一场关于它到底是什么意思的混战。

> “你不应该再提示 coding agents 了。你应该设计能提示 agents 的 loops。”——@steipete，2026 年 6 月 7 日

最有代表性的回复问出了唯一重要的问题：这在实践中到底长什么样？后来成为整场讨论氛围答案的，是 Matthew Berman 的那句回复。

> “除了他和 Boris，没人知道。”——@MatthewBerman，2026 年 6 月 7 日

这才是真正的故事。不是说 loops 就是未来，而是一个六词短语获得了两百万浏览量，同时那些转发它的人还在争论它到底是什么意思。我没有翻白眼，因为我自己每天晚上都会运行一个 loop，在我睡觉时给大约三十个开源仓库提交 pull requests。AI coding 圈最响亮的想法，恰恰是大多数复述它的人解释不清的东西。

## Loop 到底是什么

Boris Cherny 在 2024 年 9 月把 Claude Code 作为一个 side project 创建出来。据称现在 GitHub 上接近 4% 的公开 commits 都与它有关。6 月 2 日，在 WorkOS 的 Acquired Unplugged 活动上，他给出了你能找到的最清楚的 loop 定义。

> “我已经不再提示 Claude 了。我有一些正在运行的 loops。它们负责提示 Claude，并判断该做什么。我的工作是写 loops。”——Boris Cherny，2026 年 6 月 2 日

Loop 是一个小程序：它替你提示 agent，读取 agent 产出的结果，判断任务是否完成；如果没完成，就再次提示它。

你不再是那个在 loop 里面不断输入 prompts 的东西。你变成了 loop 的作者，而模型变成了一个 subroutine。一年前，Boris 还在用 autocomplete 手写代码。后来，他并行运行五到十个 Claude sessions。现在，他完全不再亲自 prompt，几百个 agents 会读取他的 GitHub、Slack 和 Twitter，并决定下一步该构建什么。工作没有消失。它只是上升了一个抽象层级：从写代码，变成写那个会写代码的东西。

## 谱系：从 ReAct 到 orchestration

回复区之所以一团糟，是因为 loop 这个词至少藏着五种不同的东西。第一阶段是 2022 年 ReAct 论文里的学术 while-loop。第二阶段是 2023 年的 AutoGPT，它拿到一个目标后，就因为无限空转而出名。第三阶段是 Geoffrey Huntley 在 2025 年 7 月发布的 ralph loop：一个 bash one-liner，会把同一个 prompt 文件一遍又一遍地 pipe 给 agent，每次迭代都重置上下文。第四阶段是产品化：2026 年春天，Codex 和 Claude Code 都发布了 /goal，它会运行 ralph loop，直到一个小型 validator model 确认任务完成。

第五阶段才是 Boris 和 Steinberger 真正指的东西，而且它确实是新的。Loop 变成了工作单元，而不是任务本身。Loops 开始监督其他 loops，并发执行，并按 schedule 运行。调度取代了人类的启动动作。Durability 也变成了显式能力：有 git-backed state 和 crash recovery。Ralph 假设你的终端会一直开着。2026 年版本则假设终端不会一直开着。

## 它只是戴了顶帽子的 cron job

语料里最好的怀疑派说法只有四个词。

> “Cronjobs 正在被搞笑地重新包装。”——X 回复，2026 年 6 月

这句话说对了一半。调度层确实是 cron。但 cron 从来没有中间那部分：一个模型会查看当前状态，决定下一步做什么，执行它，检查是否成功，并决定是否继续。做决定的是 agent，而不是硬编码分支。把这些叠起来，让一个 loop 监督其他 loops，给它们持久化的共享状态，你就拥有了 cron 无法表达的东西。

Loops 就是 cron 加上一个位于 body 内部的 decision-maker。

## 真正构建出来时它长什么样

入门只需要一行。Claude Code 发布了 /loop，而 Boris 自己的例子就是标准起点。

> /loop babysit all my PRs. Auto-fix build issues, and when comments come in, use a worktree agent to fix them.

几天后，Boris 发布了五条让 Opus 自主运行数小时甚至数天的技巧：使用 auto mode 处理权限；使用 dynamic workflows 编排数百或数千个 agents；使用 /goal 或 /loop 持续运行直到完成；在 cloud 中使用 Claude Code，这样你就可以合上笔记本；并确保 Claude 有办法端到端地自我验证工作。

一个 loop 是否值得信任，取决于它检查自己工作的能力。

更深的玩法是 Steve Yegge 的 Gas Town：二十到三十个 Claude Code instances 由一个 Mayor agent 协调，patrol agents 持续运行 loops，状态存储在 git 里，这样即使崩溃工作也能存活下来。但增长最快的子主题不是 orchestration，而是 verification。Dan Kornas 正在发布 roborev，它会在后台审查每个 commit，并趁上下文还新鲜时把发现反馈给 agent。Loop 本身不是魔法。它内部的反馈才是。

## 剧情反转：loop 现在成了昂贵的部分

研究到这里，从哲学问题变成了财务问题。最尖锐的祛魅来自一位一线工程师。

> “我今年发布的每个 ai agent 都是一个 for-loop、一次 llm call，以及包在 json parsing 外面的 try/catch。唯一 agentic 的地方，是月底的 anthropic 账单。”——rohit，2026 年 6 月

这张账单不是玩笑。Uber 在四个月内烧完了年度 AI 预算后，把工程师使用 Claude Code 和 Cursor 的额度限制在每人每工具每月 1,500 美元。一旦模型几乎免费地写代码，成本就转移到了运行它的 loop 上。这就是为什么 2026 年所有严肃文章都会收敛到同样三个硬性停止条件：最大迭代次数、无进展检测，以及 token 或美元预算上限。你写 loops，而你大部分工作是在确保它们会停下来。

## 重点不是 loops，而是 skills

看了一周之后，我自己的看法是这样的。Loop 是 plumbing。真正的资产是它调用的 skill。如果你做某件事超过一次，就把它变成一个自动化 skill；如果你做了某件很难的事，事后也把它变成 skill，这样下次就是免费的。一个里面没有可复用 skills 的 loop，只是围着陌生人套了一个 while-true。一个调用一组锋利、经过测试、命名清楚的 skills 的 loop，才是一个会复利的系统。

所以，“WTF is a loop” 的答案不是一个关于 prompt engineering 已死的热观点。不要再做 loop 里的那个东西。把 loop 写一次，给它值得调用的 skills 和能自检的反馈，给它设置上限让它会停，然后让它在 cron 上运行，而你去决定下一步该构建什么。好消息是，从这个月开始，入门只需要一个 slash command。

## 研究中的关键模式

Loop 是 cron 加上一个位于 body 内部的 decision-maker：每一 tick 中，选择下一步动作的是模型，而不是硬编码分支。

它的谱系是真实存在的：2022 年的 ReAct，2023 年的 AutoGPT，2025 年的 ralph，2026 年春天的 /goal，以及现在的 orchestration loops。

Loop 的质量取决于它的反馈。持续 review 和 validation gates 才让它值得信任。

昂贵资源已经从 tokens 转移到了 loop management。要限制迭代次数，检测无进展，并设置美元预算。

Loop 内部可复用的单元是 skill，而不是 prompt。

本文基于 2026-06-07 的 /last30days 运行结果整理。我运行能在睡觉时提交开源 PRs 的 loops，并且会在后台运行 /last30days research 来写它们。

---

来源：[X status](https://x.com/mvanhorn/status/2063865685558903149?s=20)  
原文：[LinkedIn article](https://www.linkedin.com/pulse/wtf-loop-peter-steinberger-vs-boris-cherny-matt-van-horn-cpslc)
