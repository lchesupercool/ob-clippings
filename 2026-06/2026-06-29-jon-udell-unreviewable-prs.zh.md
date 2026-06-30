---
title: "“医生，我一看到代理创建不可审查的 PR 就难受。”“那就别这么做。”"
source: "https://blog.jonudell.net/2026/06/28/doctor-it-hurts-when-agents-create-unreviewable-prs-dont-do-that/"
author: "Jon Udell"
published: "2026-06-28T17:39:14+00:00"
saved: "2026-06-29"
tags: [ai-agent, agentic-engineering, code-review, workflow]
translation_of: "2026-06-29-jon-udell-unreviewable-prs.md"
---

# “医生，我一看到代理创建不可审查的 PR 就难受。”“那就别这么做。”

我最近参加了一场演讲，演讲者是一家大型软件公司的工程师，主题是不可审查的 PR。问题是什么？当 agent 提交包含数千行由 LLM 编写的新增、删除、编辑内容的 PR 时，人们无法理解它们。解决方案是什么？往这个问题上扔更多 agent：让 reviewer agent 扫描 coding agent 生成的内容、识别问题并进行分诊。

我并不在工业规模上制作软件，所以我无法评估“吞吐量收益足以证明缺少端到端人类参与是合理的”这一说法。可以说的是：当我使用 [Bram](https://github.com/judell/bram) 来 bootstrap 它自身时，正是这个工具所体现的工作流让我始终充分参与其中。

下面是 Bram 的语言构成。

| 语言 | 代码行数 |
| --- | --- |
| Rust | 24,630 |
| JavaScript | 7,542 |
| XMLUI | 4,149 |
| Python | 3,152 |
| Markdown | 1,419 |
| XS (XMLUI) | 742 |
| 总计 | 42,805 |

Bram 是一个 Tauri 桌面应用，而 Tauri 的原生语言是 Rust，所以 Rust——一种我在这个项目之前从未接触过的语言——占据了主导地位。我到现在还没有亲手写过一行 Rust！但我会在 Claude Code 和 Codex 为我写 Rust 代码的同时阅读这些代码。我理解这些代码的性质和目的；当某些东西闻起来不对劲时，我会提出反对。

Bram 的工作流通过把问题拆成小的、可测试的块，并以有序方式处理它们，帮助我做到这一点。这当然不是什么新想法。在 LLM 时代，我们正在发现新的理由去尊重旧的最佳实践。比如，我们一直说文档是产品的重要组成部分，但过去并不总是把它真正做到位。现在读者既包括人，也包括机器，于是我们会在文档上投入更多精力。那为什么不也邀请 LLM 加入我们传统的敏捷实践呢？

## 丰富的本地上下文

当我们把这些新伙伴请上船时，该如何帮它们定向？聊天会话会构建上下文，但那是 LLM 私有的上下文，并不会与由人和 agent 组成的团队共享。Bram 把这种上下文提升到两类共享空间中：本地 worklist 和 GitHub repository。在本地 worklist 里，你定义一个任务或功能，迭代它的规格说明，执行任务或构建功能，然后迭代结果。这个 worklist item 存在于本地 repo 中；无论它是否被版本控制跟踪，它都为你、Claude Code，也许还有 Codex，提供了共享上下文。如下图所示，在不同 agent 之间切换只需点击一次，因此一个 agent 可以对另一个 agent 写出的计划或实现发表意见。这里我正准备把 Claude 叫进来当救援投手。

[![图 1](../../assets/2026-06-29-jon-udell-unreviewable-prs/tauri-subscribe-churn.png)](https://i0.wp.com/jonudell.info/images/tauri-subscribe-churn.png?ssl=1)

这个系统令人愉快的涌现特性之一，是 agent 会为 worklist item 创建富有表现力的名字。命名出了名地困难。我当然也可以自己想出像 _startup-freeze-tail-fanout-diagnostics_ 这样的名字，但这些名字并不是面向公众的；它们完全够用，也没有理由让我承担创建它们的认知负担。

Bram 会记录 worklist item 的可搜索历史，所以我和我的 agent 都可以引用它们。

[![图 2](../../assets/2026-06-29-jon-udell-unreviewable-prs/fanout-history-search.png)](https://i0.wp.com/jonudell.info/images/fanout-history-search.png?ssl=1)

我们人类的上下文窗口一次大约只能处理五到七件事，所以我会相应修剪 worklist。如果出现其他事情，把 _startup-freeze-tail-fanout-diagnostics_ 的优先级挤下去了，我可以用 Drop 按钮把它从 worklist 中清掉。之后我可以在 History 页面重新找到它，也许通过搜索 _fanout_，然后让当前活跃的 agent 把它复活为一个新的 worklist item。

## ~~人在~~ Agent 在循环中

我不喜欢“human in the loop”（人在循环中）这个短语，因为它把主导权让给了机器。让我们反过来讲这个叙事。这是我们的循环，我们仍然以一贯的方式工作，只是现在招募 agent 加入团队。一个 agent 辅助的流程不必是一个吃进 prompt、吐出功能的黑箱。

这让我想起 Brian Marick 的一个漂亮想法，Ward Cunningham 曾经实现并演示给我看。Brian 称之为 [visible workings](https://blog.jonudell.net/2008/03/04/ward-cunninghams-visible-workings/)（可见的工作机理）。Ward 的实现让 Eclipse Foundation 的一个工作流变得可见。当 UI 展示一个表单时，它会添加一个 Explore 按钮，你可以用它检查驱动这个表单的业务规则。

让我们也这样做 agentic 软件开发。不要把它当作一个我们被排除在外的循环，而要把它当作一个我们邀请 agent 加入的循环。
