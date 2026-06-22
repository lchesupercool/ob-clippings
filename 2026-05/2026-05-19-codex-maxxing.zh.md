---
title: Codex-maxxing
description: 我如何把 Codex 当作长期运行工作的驻留之处。
date: 2026-05-10
authors:
  - jxnl
categories:
  - Applied AI
  - Personal
comments: true
draft: false
slug: codex-maxxing
tags:
  - Codex
  - AI
  - Workflows
  - Personal Software
source: https://jxnl.co/writing/2026/05/10/codex-maxxing/
source_markdown: https://github.com/jxnl/blog/raw/main/docs/writing/posts/codex-maxxing.md
clipped: 2026-05-19 14:57:03 +0800
---

# Codex-maxxing

在 Codex 之前，我已经大量使用 coding agents。不过大多数时候，我是通过为 coding work 构建的界面来使用它们：制作 diff、修改 repo，以及交付代码。

大约从 11 月开始，我也开始把它们推向知识工作。我用 [Slidev](https://sli.dev/) 做演示，把 agents 更像带语音输入的记笔记者来用，并持续寻找 coding agent 能帮我产出的其他 artifact：一个 `index.html`、一份 PDF、一张电子表格、一套幻灯片。

最新的 Codex app 升级，是我用过的第一样东西，让这种更宽泛的模式感觉像是原生的。Codex 依然非常擅长 coding，但更有意思的转变在于，它给了我的工作一个可以驻留的地方。

真正改变我行为的，是学会给工作一个 operating loop：一个持久线程、共享记忆、能在我的电脑上行动的工具、引导并恢复任务的方式，以及一个我可以审阅 artifact 本身的表面。

<!-- more -->

## 持久线程

首先改变我行为的是 compaction。

!!! note "Compaction"

    Compaction
    : 对长期运行的线程进行压缩，使它无需完整携带每一条旧消息也能继续推进。

现在，我会为每个我关心的重要 workstream 保留一个置顶线程：

- 我的 Chief of Staff 线程
- [Agents SDK](https://openai.com/index/the-next-evolution-of-the-agents-sdk/)
- [OpenAI CLI](https://github.com/openai/openai-cli)
- [面向开源的 Codex](https://openai.com/form/codex-for-oss)
- 一个专门用来监控 Twitter 的线程

这些不是短聊天。它们是我已经 compact 了几个月的超长线程。它们会积累历史、偏好和旧决策，而这些是我不想每次回来时都重新创建的。

!!! tip "Pinned thread shortcuts"

    你可以用 `Command-1` 到 `Command-9` 直接跳转到置顶线程。

这里有一个取舍。长期运行的线程不是免费的。如果你之后再访问它们，对话很可能已经不在缓存里，所以相比一个全新的短线程，你可能会产生更多成本。对于我关心的 workstream，连续性值得这个代价。

## 语音输入

语音输入让更多我真实的思考进入 Codex。

好处不在于速度。而在于 agent 得到的是我未经编辑的思考版本。Codex 内置了语音输入，但我也使用 Wispr Flow，因为系统级听写会改变我能向其他所有东西输入多少上下文。如果我正在规划一项工作，我可能会说：“我觉得 Slack 里有个叫 Ben 的人提到过这件事，我不太记得具体是什么，去找找看。” 这句话打出来太模糊、太烦人，但说出来却完全自然。

同样的事情也适用于 transcripts。如果我想写一篇文章，我可以给某人打电话，录下对话，或者带着手机上的 Granola 当面和他们聊，然后用 transcript 作为起始材料。当模型能够访问我思考中混乱的版本，而不只是打磨过的版本时，很多计划都会变得更好。

## 引导

语音和引导结合起来时会更有用。

!!! note "Steering"

    Steering
    : 在 Codex 已经开始工作时继续添加方向，而不是等当前步骤完成。

Steering 让你可以在一次工具调用之后注入下一条消息。如果我正在审阅一个网站，我可以一边看一边继续说：

- 把这个做小一点
- 这段文案不对
- 这两个东西之间的间距感觉不对
- 这个完成后，开一个 PR
- 等 preview deploy 完成
- 把 preview link 发给 Slack 上需要审阅它的人

我不需要等每一步完成后再决定下一步。我可以在 agent 仍在工作时继续添加意图，然后在队列已经被塑形好之后离开。

之后，Heartbeats 可以在我离开后监控 PR 或 Slack thread。工作的单位不再是“一次 prompt，一个答案”。它变成了一个小型 operating loop。

## 记忆

一旦线程开始持续更久，它们就需要任何单个 repo 之外的共享记忆。

重要的动作不只是保留消息历史。一个长线程可以记住很多东西，==但除非有用的部分被序列化到某个持久位置，否则那份记忆会被困在线程内部。==memory system 的重点，是把线程学到的东西转化为我可以检查、编辑、diff 和复用的 artifact。

我的大多数长期运行线程都从一个 Obsidian vault 开始：

```text
vault/
├── TODO.md
├── people/
├── projects/
├── agent/
└── notes/
```

在顶层，我会保留 `AGENTS.md` 指令，内容大致是：当你进一步了解人、推进项目，或关闭一个 open loop 时，更新 vault 中相关页面。

vault 是 agent 所生活的地方，与任何单个项目分离。Repositories 存放代码。vault 存放围绕我工作的滚动上下文：人、决策、open loops、daily notes、项目状态，以及那些否则会在线程之间丢失的理解碎片。

我也把 vault 作为一个 GitHub repo 保存。这给了我两件事：

1. 它可以在云端工作
2. diffs 成为记忆的审阅表面

当 agent 更新 vault 时，我可以阅读 diff，看看它认为哪些内容重要到值得记住。这个审阅步骤很重要。我不希望 evergreen threads 在对话历史里悄悄积累某种氛围。我希望它们写下发生了什么变化：这个人偏好这个，这个项目在等待那个，这个决策已经做出，这个 loop 已经关闭。

这也是为什么我喜欢以文件作为记忆。文件迫使 agent 把经验压缩成一种能够在线程之外存活的形式。如果线程死掉、compact 得很糟，或者继续依赖它变得太昂贵，有用的知识仍然在那里。

到了那时，置顶线程开始感觉不太像聊天，而更像是不同 worker 在阅读同一本笔记本。

Codex 在 `Settings > Personalization > Memories` 中也有第一方记忆功能。我把它们看作本地 recall layer：对稳定偏好、重复 workflow、项目约定和已知陷阱很有用，但不能替代 check-in 的 instructions 或一个显式 vault。[Chronicle](https://developers.openai.com/codex/memories/chronicle) 在这里尤其有意思，因为它可以使用最近的屏幕上下文来帮助构建 memories。我还没有认真使用过它，而且文档很清楚地说明，它是一个 opt-in research preview，在权限、速率限制、prompt injection 和未加密的本地记忆文件方面存在真实取舍。但从方向上看，它指向的正是我关心的同一件事：工作应该留下结构化记忆，而不只是更长的聊天 transcript。

!!! note "Shared memory"

    Shared memory
    : 保存在单个聊天之外的上下文，例如我的 vault 中可被不同线程复用的 notes。

## 电脑与浏览器使用

一旦一个线程有了记忆，下一个问题就是它能触碰什么。

在我脑中最有用的区分是：

- `$browser` 用于我想检查和标注的本地 web surfaces
- `@chrome` 用于已登录的浏览器状态和多个 tabs
- `@computer` 用于只以 GUI 形式存在的工作

如果我正在迭代一个本地 app，我想要 `$browser`。如果我需要在已登录的浏览器会话中工作，我想要 `@chrome`。如果完成任务的唯一方式是在桌面 app 里点击，我想要 `@computer`。

在我的工作机器上，Twitter 登录在 Safari 里。如果我让 `@computer` 在那里读取 Twitter，它工作时我就失去了 Safari。当我希望 agent 并行使用几个已认证的 tabs，而不接管我正在使用的整个 app 时，`@chrome` 更好。

Connectors 将这种触达扩展到我实际工作的其余部分。我最常使用的是 `$slack`、`$gmail` 和 `$calendar`，因为 Slack threads、inboxes 和 calendars 是很多工作在变成代码之前出现的地方。

Skills 让重复 workflow 可复用。Skill Creator 和 Skill Installer 是很好的起点。Skill Installer 让你可以直接从 composer 添加 OpenAI 推荐的 skills。在 [Codex pets](https://developers.openai.com/codex/app/settings#codex-pets) 发布后，我用它安装了 Hatch Pet skill，但有用的模式是通用的：一旦你做过一次有用的事情，通常就可以把它打包起来，让 Codex 下次无需重新教 workflow 也能再做一次。

## 远程控制

远程控制让这些更长的 loops 感觉可携带。

Codex 可以继续在你的文件、权限和本地设置已经存在的那台机器上工作，而你可以从移动端回来查看、审阅它发现的内容、回答问题、批准下一步，或在不回到桌前的情况下改变方向。[OpenAI 将其描述为一种从任何地方与 Codex 一起工作的方式](https://openai.com/index/work-with-codex-from-anywhere/)。

当 Codex 已经在做某个长期运行的事情，而你想保持 momentum 时，这一点最重要。你可以启动一个任务，走开，然后当它到达决策点时从手机上引导它。

这与置顶线程、语音和 Heartbeats 重要的原因相同。工作不再只是因为我换了地点就必须暂停。一个线程可以继续推进，而我可以保留刚好足够的注意力来解锁下一步。

## Heartbeats

置顶线程很有用，但它们仍然会等你说些什么。Heartbeats 让它们变成周期性的。

!!! note "Heartbeats"

    Heartbeats
    : 线程可以为自己安排的周期性检查，例如监控 Slack 或 pull request 的新活动。

Heartbeat 是一种 thread-local automation。你可以说，“每隔几个小时关注一下这个”，然后线程可以为自己安排计划。一个线程可以有多个 schedules，运行到某个条件满足为止，并随着时间调整自己的 cadence。

### Chief of Staff

我的 Chief of Staff 线程每 30 分钟运行一次：

```text
Every 30 minutes, check Slack and Gmail for unanswered messages that need my attention.
Help me prioritize what matters most.
If someone asks me a question, research the answer as deeply as you can and draft a reply for me, but do not send it.
```

当我回到 Slack 时，回复通常已经在 drafts 里等着了。我仍然决定发送什么，但收集上下文这个昂贵的部分已经完成。

### 监控反馈

同样的模式也适用于 review loops。Heartbeat 可以监控 Google Docs comments、pull request comments 或 Slack replies，并在反馈到来时让工作继续推进。

我最喜欢的例子之一来自一个动画项目。我在 Slack 里发布了一段视频，并要求 Codex 每 15 分钟检查一次 thread 中的反馈，在 comments 到来时重新渲染一个新版本，并在 thread 里 tag reviewer 回复。Slack MCP server 无法上传文件，所以 agent 还是用 `@computer` 按下 Add file 按钮并发布了修订后的 render。

有趣的部分不只是它每 15 分钟检查一次 Slack。这个 loop 跨越了工具边界：Slack 用于反馈，Remotion 用于 render，`@computer` 用于 upload。正是在那时，Heartbeats、connectors 和 computer use 不再像是分开的功能。它们合在一起成为一个无需我坐在那里也能持续运行的 feedback loop。

### 获得退款

最近我有一个包裹被偷了。Amazon 告诉我大约需要 25 分钟才能和真人客服交谈，于是我用 `@computer` 创建了一个线程，并告诉它：

```text
Every 5 minutes, check whether the customer support agent has joined this thread.
If they have, do your best to get me a refund.
Once they reply, switch to checking every minute so you can respond faster.
```

等我洗完澡出来时，退款已经完成了。

我的许多 Heartbeats 也会更新我的 Obsidian vault，把它作为一种显式记忆。

## 目标

我仍在学习如何用好的最新功能是 Goals。

!!! note "Goals"

    Goals
    : Codex 可以持续推进、有真实终点线的更长期任务。

你应该对它们有雄心。一个弱 goal 是“实现这个 Markdown 文件里的计划”。一个强 goal 具有真实的成功标准，agent 可以持续朝它推进。

上周我尝试把 Python [Rich](https://github.com/Textualize/rich) library 迁移到 Rust。因为原项目已经有一套大型 unit test suite，我可以设定这样的 goal：把 Rich 迁移到 Rust，但它必须通过原 library 的所有 unit tests。

这套 test suite 给这次运行提供了一个真实 oracle：直到 Rust port 通过与 Python library 相同的 tests，它才算完成。

这不同于和 AI 进行一场长对话，积累一个 Markdown plan，然后最终说：“实现这个。” 执行的质量只取决于你给它的 goal 和 verification。没有 verification 的雄心只是一个愿望。

## 侧边面板

Codex 中最让我兴奋的部分是 side panel。

很容易把它看作一个发生 previews 的地方。但这低估了它。side panel 是 Codex 不再只是一个 chat app，而开始成为工作发生之处的地方。

对我来说，它承担三项工作：检查 artifacts、操作 web surfaces，以及审阅 changes。在这三者中，我都可以查看并评论 agent 正在作用的同一个对象。

### 检查 artifacts

Markdown、spreadsheets、CSVs、PDFs 和 slides 都可以在那里驻留。

Markdown 可以评论。Spreadsheets 会渲染 formulas 并支持 cell edits，我用它来管理 Codex open-source plans。CSVs 会变成 tables，而不是 raw text。PDFs 会直接渲染，这对 LaTeX 尤其有用。Slides 可以在不离开 app 的情况下创建和审阅。

重要的并不只是 Codex 可以生成 artifacts。而是我可以检查并标注它们，同时不打断这个 loop。

### 操作 web surfaces

app 内浏览器甚至更有意思。agent 可以看到它，通过 `$browser` 用 JavaScript 控制它，而我可以直接在我正在看的东西上留下 annotations。

现在有几个 web surfaces 我一直这样使用：

- `index.html` 用于轻量级静态 artifacts
- Storybook 用于审阅 UI components
- Remotion Studio 用于 programmatic animation
- [Slidev](https://sli.dev/guide/) 用于 presentations
- Streamlit 用于 data apps

最小的版本通常是最好的。你可以让模型制作一个带 JavaScript 和 CSS 的单个 `index.html` 文件，在 side panel 中打开它，并立即开始与它交互。无需 server。我一直在尝试用 Heartbeats 随时间更新一个静态 `index.html`，这样每当我回到一个线程时，已经有一个新鲜的 artifact 在等我。

[Thariq 有一篇非常好的帖子](https://x.com/trq212/status/2052809885763747935)，谈到为什么偏好 HTML 而不是 Markdown 作为输出格式。我认为这种直觉是对的。一旦输出变成一个小应用，而不只是一个文档，关系就会改变。

如果我需要更重的东西，我可以使用 Vite app，但那样我就需要保持一个 server 运行。一个普通的 `index.html` 要持久得多。

在动画工作中，我经常把 Storybook 和 Remotion Studio 并排打开。我可以留下像“让这个 bounce”或“这个应该更大”这样的评论，而 agent 可以检查我正在看的同一个 browser state，包括动画中的当前 frame。

做演示时，我经常使用 Slidev。Codex 可以检查 slides，发现被截断的内容，在 slides 之间切换，并在我审阅时回应 annotations。

我也预计，随着时间推移，这对 Streamlit 和 Jupyter 这样的 tools 会变得更有用。不同的人已经生活在不同的 applications 里。Codex 越来越能在那里与他们会合。

Codex 越是获得可以记住、重新访问、检查和行动的地方，我的工作就越不容易死在 prompts 之间。这就是我关心的变化。不是 agent 能替我写代码，而是我离开之后，更多我的工作仍能继续推进。
