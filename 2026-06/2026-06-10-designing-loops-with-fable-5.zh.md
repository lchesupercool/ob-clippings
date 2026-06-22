---
title: "用 Fable 5 设计 loops"
author: "Lance Martin"
source: "https://x.com/RLanceMartin/status/2064397389189071163"
canonical: "https://x.com/RLanceMartin/article/2064380553919676416"
published: "2026-06-09T17:21:06.000Z"
saved: "2026-06-10 20:46:08 CST"
tags: [clipping, x-twitter, claude, fable-5, agent-loops, memory]
translation_of: "2026-06-10-designing-loops-with-fable-5.md"
---

# 用 Fable 5 设计 loops

Lance Martin [@RLanceMartin](https://x.com/RLanceMartin)  
发布：2026-06-09

![用 Fable 5 设计 loops](../../assets/designing-loops-with-fable-5/cover.jpg)

像 Claude Fable 5 这样的 Mythos 级模型，已经改变了 Anthropic 内部很多人的工作方式。我想分享两个技巧，帮助大家更好地使用这类模型。

## 自我纠错 loops

最近大家对 loops 很感兴趣。[@bcherny](https://x.com/bcherny) 提到过，“（他的）工作就是写 loops”。让模型在一个 evaluation 上 hillclimb，是提升任务表现的常见配方：Claude Code 里的 `/goal` 和 Claude Managed Agent 里的 Outcomes，都是让你可以把这个通用配方应用到具体任务上的 primitives。

正如我们在 prompting guide 中提到的，Fable 5 很擅长在 loop 中自我纠错。一个设计良好的 goal 或 rubric，会给 Claude 运行所在的环境增加反馈。这让 Claude 可以运行、通过 goal 或 rubric 收集反馈、自我纠错，然后继续推进，直到 goal 或 rubric 被满足。

我分享一个自己用来测试 Fable 的玩具例子：Parameter Golf 是一个开源 ML engineering challenge，目标是在 8xH100 上用少于 10 分钟训练出最好的模型，并让 artifact 小于 16MB。

它有点像 [@karpathy](https://x.com/karpathy) 的 autoresearch 项目：它测试的是 agent 编辑基础训练代码（一个单独的 `train_gpt.py` 文件）、启动训练、轮询日志、读取分数，并决定下一步该运行什么实验的能力。

我在这个挑战上用 Claude Managed Agents（CMA）比较了 Fable 5 和 Opus 4.7。CMA 提供 agent harness，也提供 hosted sandbox，所以很适合用 Fable 5 做长时间运行的任务。在 Parameter Golf 中，我给 CMA 接入了 8xH100 GPUs，作为 self-hosted sandbox。

一个细微但重要的点是：由什么来评判很重要。我们看到模型在对自己的输出做 self-critique 时会遇到问题。Prithvi Rajasekaran 在我们的 engineering blog 里写过这件事。

我们发现，对 Fable 5 来说，verifier sub-agent 往往比 self-critique 表现更好，因为评分是在独立的 context window 中完成的。CMA 里的 Outcomes 会替你生成一个 grader sub-agent 来处理这件事。

每次测试时，我都会提供一个 rubric（一个文件），其中包含九条可检查的 criteria（例如运行 baseline、运行 20 个实验等）。然后，我让 Parameter Golf 最多运行 8 小时。Outcomes grader 会确认所有实验性 criteria 都满足之后，才允许 Claude 停止工作。

Fable 5 对训练流水线的改进幅度大约是 Opus 4.7 的 6 倍。如果我们把实验分成 structural（例如架构改动）和 scalar（例如调整某个常数），Fable 5 会押注于更大的 structural changes，并表现出韧性（例如在一次 quantization regression 之后继续推进，最终取得最大胜利）。

Opus 4.7 的第一个实验带来了一个小收益，但之后几乎所有实验都沿用同一个模板：调整一个 scalar，测量，如果结果为正就保留。

## Memory

Memory 是 Fable 另一个擅长的领域。我们可以把它看作一个跨越多个 session 的 outer loop：Claude 在一个 session 中写入 memory，而这些 memories 可以在未来 session 中被检索出来。

[@pgasawa](https://x.com/pgasawa) 和团队最近发布了 Continual Learning Bench 1.0，所以我想用它来测试 Fable 5 与更早模型的表现。

我在 benchmark 中选了一个任务，比较了 Fable 5、Opus 4.7 和 Sonnet 4.6：这个任务要求 agent 在能够访问 SQL database 的情况下，回答一系列顺序问题。每个问题都是一个独立的 agent session，并且会提供 memory。

为此，我使用了带 memory 的 CMA，它会给每个 agent 提供一个 mounted filesystem，并且这个 filesystem 可以跨 session 共享。

在这个任务中，有效使用 memory 得益于一个渐进流程：fail（答错并记录）、investigate（继续之前先弄清楚为什么错）、verify（把诊断变成已检查的事实）、distill（把验证结果变成通用规则）、consult（读取规则，而不是重新推导）。

Sonnet 4.6 大约停在第 1 步：它的存储是一组失败记录和开放猜测（例如“maybe prc instead of prc_usd?”）。它很少 consult 之前的 notes。要提升表现，需要加入任务特定的 memory instructions。

Opus 4.7 大约停在第 3 步：它会创建一个 schema reference，并标出不确定性（例如“possibly prc in cents? Verify.”），但 verification coverage 很低：只有 7–33% 的问题被覆盖（中位数运行约 17%）。

Fable 5 往往能完成这个流程：在它最强的运行中，verification coverage 最高达到 73%（30 个问题中的 22 个），并且它会把学到的东西 distill 成通用规则，帮助未来任务。

与其直接 prompting 和 steering Fable 5，更好的做法通常是设计 loops，让模型能够响应环境反馈（例如 `/goal` 或 Outcomes）来自我纠错，并管理自己的上下文（例如通过 memory）。

我只分享了自己跑过的几个小规模实验，但值得你自己在有挑战性的任务上测试 Fable 5，并使用 loops 来做 self-correction 或 memory。

要开始，可以查看我们的 docs，或者询问最新版 Claude Code。它可以使用我们内置的 `/claude-api` skill 来告诉你 Fable 5（例如 prompting best practices）、`/goal`、Claude Managed Agents，或其他 API features。

## 文中提到的链接

- [Prompting Claude Fable 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5)
- [Claude Managed Agents](https://www.anthropic.com/engineering/managed-agents)
- [Managed Agents self-hosted sandboxes](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes)
- [Define outcomes](https://platform.claude.com/docs/en/managed-agents/define-outcomes)
- [Managed Agents memory](https://platform.claude.com/docs/en/managed-agents/memory)
- [Parameter Golf](https://github.com/openai/parameter-golf)
- [Harness design for long-running agents](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [karpathy/autoresearch](https://github.com/karpathy/autoresearch)
- [Claude Code /goal](https://code.claude.com/docs/en/goal)
- [Claude prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Related X post](https://x.com/sairahul1/status/2064279904989147577?s=20)
- [anthropics/skills claude-api](https://github.com/anthropics/skills/tree/main/skills/claude-api)
- [Managed Agents overview](https://platform.claude.com/docs/en/managed-agents/overview)

---

来源：[X status](https://x.com/RLanceMartin/status/2064397389189071163)  
原文：[X article](https://x.com/RLanceMartin/article/2064380553919676416)
