---
title: "Automating away — 中文摘要"
source: "https://replicated.live/blog/away"
saved: "2026-07-08"
author: "Beagle SCM"
summary_of: "Automating away"
tags: [ai-agent, deterministic-tools, software-engineering, harness-engineering, scm]
---

# Automating away — 中文摘要

## 一句话结论

作者认为，LLM 会越来越聪明，但不会自然变得不笨拙；正确做法不是期待模型完全可靠，而是让它把自己经常做、经常错的动作逐步自动化成快速、确定性、可验证的工具和流程，最终让 LLM 把自己“自动化掉”。

## 文章主旨

文章从 Karpathy 的一句话切入：OpenAI 研究者在通过改进 AI “automating themselves away”。作者把这个想法转到软件工程智能体上：他正在用 Anthropic Fable 开发 Beagle SCM，模型很强，能在大量代码中发现细节问题、创建 ticket、修复代码；但它也很笨拙，比如会两次把 `build/` 目录提交进项目。

作者的核心判断是：这种“聪明但笨拙”不是短期 bug，而是 LLM 的性质决定的。LLM 天生有不精确、非确定性、容易越界的问题。随着模型变强，它会更聪明，但不会自动变成形式化、确定性的工程工具。

## 关键论点

### 1. 强模型仍然会犯低级工程错误

作者举了两个例子：

- Fable 能读代码、提 ticket、做修复，但仍会把构建产物目录提交进去。
- 作者明确写了全大写指令：不要手写 parser，永远不要；但 Claude 还是会尝试手工解析。于是作者需要定期让它扫描代码库，找出并删除手工解析逻辑。

这说明 prompt 约束有用，但不够。模型会违反指令，尤其是在复杂任务和长上下文里。

### 2. LLM 不应该单独承担可靠性

作者把 LLM 和 Ragel 这种 parser generator 对比。Ragel 可以瞬间生成 10K 行形式正确的 parser，而且确定性强；Claude 可以写代码，但不具备同样的形式化保证。

所以，对“昂贵、慢、笨拙但聪明”的 LLM，正确架构是：

- 给它快速、强大、确定性的工具。
- 把整个过程放进形式化 workflow。
- 让它在正确时间看到相关信息。
- 让它更容易自我纠错。
- 用确定性工具和确定性流程夹住 LLM 的非确定性。

这就是一种典型 Harness Engineering：LLM 负责判断、组合、探索；工具和流程负责确定性执行与验证。

### 3. 最有趣的是让工具和流程本身可被改造

作者进一步提出：如果工具和流程是 malleable 的，那么系统就能演化。

- 如果 Claude 经常执行某个动作，就把这个动作自动化。
- 如果 Claude 经常在某类事情上失败，就把对应验证步骤自动化。

这不是让 LLM 永远做重复工作，而是让 LLM 把重复动作沉淀成工具，把失败模式沉淀成检查器。

最终结果是：LLM 自动化自己的一部分工作，把不稳定的模型行为替换成简单、可靠、确定性的工具。

## Beagle SCM 的设计

Beagle SCM 允许 LLM 用 JavaScript 给自己写 routine。底层 heavy lifting 由 C 实现，很少改动；上层工具层和工作流层用 JavaScript 写，并像 `node_modules` 一样从文件系统加载。

作者类比说，可以想象一种加强版 git hooks：

- 能 tokenize 几乎任何语言的源码。
- 能检查文件历史和 commit 历史。
- 能交叉检查链接。
- 能访问 git 内部能访问的各种数据。

这些能力构成了 Beagle 的核心：给 LLM 一个可脚本化、可扩展、接近 SCM 内部数据的工具环境，让它把常用动作和检查逻辑变成确定性 routine。

## 和 Harness Engineering 的关系

这篇短文和 Lilian Weng 的 Harness Engineering 观点高度一致，但更工程化、更直接。

它强调：

- LLM 本身负责“聪明”。
- 工具负责“可靠”。
- workflow 负责“顺序和约束”。
- verification 负责“防止笨拙错误”。
- 可脚本化 routine 负责“把经验固化”。

真正的 Agent 系统不是一个更长的 prompt，而是一套能让模型把重复经验转化为工具、把失败经验转化为验证步骤的自我改进环境。

## 对我们的启发

这篇文章对 Hermes / Codex / Claude Code 工作流很直接：不要把所有可靠性都压在模型上。模型一旦反复做某件事，就应该沉淀成脚本、skill、命令或检查器；模型一旦反复犯某类错，就应该沉淀成验证步骤。

例如：

- 经常忘记排除构建产物 → 加 gitignore 检查或 pre-commit 检查。
- 经常误改索引 → 写结构化验证脚本。
- 经常抓网页丢图 → 固化 clipping quality gate。
- 经常翻译漏段 → 做标题数、链接数、图片数、代码块数对齐校验。
- 经常把临时上下文塞满 → 把状态写进文件和日志。

这就是“让 LLM automate itself away”：不是让模型消失，而是让模型逐渐把自己不该重复承担的低层工作交给确定性工具。

## 我的理解

作者提出的是一个很实际的 Agent 工程原则：LLM 越强，越应该被放在确定性系统之间，而不是被当成确定性系统本身。

强模型适合发现模式、提出改进、连接上下文、写初版工具；但一旦某个动作成为 routine，就应该变成脚本。一旦某个错误成为 recurring failure，就应该变成检查器。这样系统的可靠性不是来自“模型终于不犯错了”，而是来自模型不断把自己的不可靠区域外包给工具。

这也是个人 AI 工作流真正能复利的地方：prompt 只是一次性技巧，工具、skills、验证脚本、workflow 才是长期资产。
