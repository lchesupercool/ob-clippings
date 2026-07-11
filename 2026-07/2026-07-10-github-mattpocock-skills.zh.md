---
title: 真正工程师使用的 Skills
source: https://github.com/mattpocock/skills
author: Matt Pocock
saved: '2026-07-10'
repo: mattpocock/skills
description: Skills for Real Engineers. Straight from my .claude directory.
stars: 163301
forks: 14057
default_branch: main
pushed_at: '2026-07-09T13:57:49Z'
tags:
- clipping
- github
- skills
- agent
- software-engineering
- ai-coding
---

> [!info] Repository
> URL: https://github.com/mattpocock/skills
> Description: Skills for Real Engineers. Straight from my .claude directory.
> Stars: 163301 · Forks: 14057 · Language: Shell
> Latest push: 2026-07-09T13:57:49Z

[![Skills](../../assets/github-mattpocock-skills/skill-repo-light_2x.png)](https://www.aihero.dev/s/skills-newsletter)

# 真正工程师使用的 Skills

[![skills.sh](../../assets/github-mattpocock-skills/skills-sh-badge.svg)](https://skills.sh/mattpocock/skills)

这些是我每天用来做真实工程工作的 agent skills——不是 vibe coding。

开发真实应用很难。GSD、BMAD 和 Spec-Kit 这类方法试图通过接管流程来帮忙。但在这么做的同时，它们拿走了你的控制权，也让流程里的 bug 变得难以解决。

这些 skills 被设计得小、容易适配、可组合。它们适用于任何模型。它们基于几十年的工程经验。你可以随意折腾它们，把它们变成你自己的东西。祝你玩得开心。

如果你想跟进这些 skills 的变化，以及我创建的任何新 skills，可以加入我的 newsletter，和大约 60,000 名其他开发者一起订阅：

[订阅 Newsletter](https://www.aihero.dev/s/skills-newsletter)

## 快速开始（30 秒设置）

1. 运行 skills.sh 安装器：

```bash
npx skills@latest add mattpocock/skills
```

2. 选择你想要的 skills，以及你想把它们安装到哪些 coding agents 上。**确保你选择 `/setup-matt-pocock-skills`**。

3. 在你的 agent 中运行 `/setup-matt-pocock-skills`。它会：
   - 询问你想使用哪种 issue tracker（GitHub、Linear 或本地文件）
   - 询问你在 triage ticket 时会应用哪些 labels（`/triage` 会使用 labels）
   - 询问你希望把我们创建的文档保存在哪里

4. 搞定——你可以开始了。

## 为什么这些 Skills 存在

我构建这些 skills，是为了解决我在 Claude Code、Codex 和其他 coding agents 上看到的常见失败模式。

### #1：Agent 没有做我想要的事

> “没有人确切知道自己想要什么”
>
> David Thomas & Andrew Hunt，[The Pragmatic Programmer](https://www.amazon.co.uk/Pragmatic-Programmer-Anniversary-Journey-Mastery/dp/B0833F1T3V)

**问题**。软件开发中最常见的失败模式是 misalignment。你以为开发者知道你想要什么。然后你看到他们构建出来的东西——才意识到他们完全没理解你。

在 AI 时代也是一样。你和 agent 之间存在沟通鸿沟。修复方法是一场 **grilling session**——让 agent 围绕你正在构建的东西向你提出详细问题。

**修复方法**是使用：

- [`/grill-me`](https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md) - 用于非代码用途
- [`/grill-with-docs`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md) - 和 [`/grill-me`](https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md) 类似，但增加了更多好东西（见下文）

这些是我最受欢迎的 skills。它们帮助你在开始之前和 agent 对齐，并深入思考你正在做的变更。每次你想做变更时都使用它们。

### #2：Agent 太啰嗦了

> 借助 ubiquitous language，开发者之间的对话和代码表达都源自同一个 domain model。
>
> Eric Evans，[Domain-Driven-Design](https://www.amazon.co.uk/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)

**问题**：在项目开始时，开发者和他们为之构建软件的人（领域专家）通常说着不同的语言。

我在自己的 agents 上也感受到了同样的张力。Agents 通常被扔进一个项目里，被要求边做边弄清术语。所以本来 1 个词能表达的东西，它们会用 20 个词。

**修复方法**是共享语言。它是一份帮助 agents 解码项目中术语的文档。

<details>
<summary>
示例
</summary>

这里有一个来自我的 `course-video-manager` repo 的 [`CONTEXT.md`](https://github.com/mattpocock/course-video-manager/blob/076a5a7a182db0fe1e62971dd7a68bcadf010f1c/CONTEXT.md) 示例。哪一个更容易读？

- **之前**：“当课程某个 section 里的 lesson 被变成 ‘real’（即在文件系统中获得一个位置）时会出现问题”
- **之后**：“materialization cascade 有问题”

这种简洁性会在一个又一个 session 中持续带来收益。

</details>

这被内置在 [`/grill-with-docs`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md) 里。它是一场 grilling session，但也会帮助你和 AI 构建共享语言，并在 ADR 中记录难以解释的决策。

很难解释这有多强大。它可能是这个 repo 中最酷的一项技术。试试看，你就知道了。

> [!TIP]
> 共享语言除了减少啰嗦之外，还有很多其他好处：
>
> - **变量、函数和文件会使用共享语言来一致命名**
> - 因此，**agent 更容易浏览 codebase**
> - Agent 也会**在思考上花更少 token**，因为它可以访问一种更简洁的语言

### #3：代码跑不起来

> “始终采取小而审慎的步骤。反馈速度就是你的速度上限。永远不要接下过大的任务。”
>
> David Thomas & Andrew Hunt，[The Pragmatic Programmer](https://www.amazon.co.uk/Pragmatic-Programmer-Anniversary-Journey-Mastery/dp/B0833F1T3V)

**问题**：假设你和 agent 已经对要构建什么达成一致。当 agent _仍然_ 产出垃圾时会发生什么？

这时该看看你的反馈循环了。如果 agent 没有关于它生成的代码实际如何运行的反馈，它就会盲飞。

**修复方法**：你需要通常那一组反馈循环：静态类型、浏览器访问和自动化测试。

对于自动化测试，red-green-refactor 循环至关重要。也就是 agent 先写一个失败测试，然后修复这个测试。这能给 agent 提供稳定水平的反馈，从而产出好得多的代码。

我构建了一个 **[`/tdd`](https://github.com/mattpocock/skills/blob/main/skills/engineering/tdd/SKILL.md) skill**，你可以把它放进任何项目。它鼓励 red-green-refactor，并给 agent 很多关于好测试和坏测试的指导。

对于调试，我还构建了一个 **[`/diagnosing-bugs`](https://github.com/mattpocock/skills/blob/main/skills/engineering/diagnosing-bugs/SKILL.md)** skill，把最佳调试实践包装进一个简单循环。

### #4：我们构建出了一团泥球

> “每天都要投资于系统设计。”
>
> Kent Beck，[Extreme Programming Explained](https://www.amazon.co.uk/Extreme-Programming-Explained-Embrace-Change/dp/0321278658)

> “最好的模块是深的。它们允许通过一个简单接口访问大量功能。”
>
> John Ousterhout，[A Philosophy Of Software Design](https://www.amazon.co.uk/Philosophy-Software-Design-2nd/dp/173210221X)

**问题**：大多数用 agents 构建的应用都复杂且难以修改。因为 agents 可以极大加速编码，它们也会加速软件熵。Codebase 会以前所未有的速度变复杂。

**修复方法**是一种全新的 AI-powered development 方法：关心代码设计。

这被内置到了这些 skills 的每一层：

- [`/to-spec`](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-spec/SKILL.md) 会在创建 spec 之前，询问你正在触碰哪些模块

更关键的是，[`/improve-codebase-architecture`](https://github.com/mattpocock/skills/blob/main/skills/engineering/improve-codebase-architecture/SKILL.md) 会帮助你拯救已经变成一团泥球的 codebase。我建议每隔几天就在你的 codebase 上运行一次。

### 总结

软件工程基本功比以往任何时候都更重要。这些 skills 是我把这些基本功浓缩成可重复实践的最佳努力，用来帮助你交付职业生涯中最好的应用。祝你玩得开心。

## Reference

这些 skills 按一个轴来划分——谁可以调用它们。**User-invoked** skills 只有在你输入它们时才可达（例如 `/grill-me`）；它们的任务是编排。**Model-invoked** skills 可以由你调用，或者在任务适合时由 agent 自动触达；它们保存可复用的纪律。一个 user-invoked skill 可以调用 model-invoked skills，但绝不能调用另一个 user-invoked skill。

### Engineering

我每天用于代码工作的 skills。

**User-invoked**

- **[ask-matt](https://github.com/mattpocock/skills/blob/main/skills/engineering/ask-matt/SKILL.md)** — 询问哪种 skill 或 flow 适合你的情况。它是这个 repo 中 user-invoked skills 的路由器。
- **[grill-with-docs](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md)** — 一场 grilling session，同时也会构建你项目的 domain model，打磨术语，并内联更新 `CONTEXT.md` 和 ADR。
- **[triage](https://github.com/mattpocock/skills/blob/main/skills/engineering/triage/SKILL.md)** — 让 issues 通过一组 triage 角色组成的状态机流转。
- **[improve-codebase-architecture](https://github.com/mattpocock/skills/blob/main/skills/engineering/improve-codebase-architecture/SKILL.md)** — 扫描 codebase 中可以加深模块的机会，把它们呈现为可视化 HTML 报告，然后围绕你选择的那一项进行 grilling。
- **[setup-matt-pocock-skills](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/SKILL.md)** — 为 engineering skills 配置这个 repo（issue tracker、triage labels、domain doc 布局）。在使用其他 engineering skills 前，每个 repo 运行一次。
- **[to-spec](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-spec/SKILL.md)** — 把当前对话转换成 spec，并发布到 issue tracker。没有访谈——只是综合你们已经讨论过的内容。
- **[to-tickets](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-tickets/SKILL.md)** — 把任何 plan、spec 或 conversation 拆成一组 tracer-bullet tickets，每个 ticket 都声明其阻塞边——可以写成 local file 中的文本，或作为真实 tracker 上的原生 blocking links。
- **[implement](https://github.com/mattpocock/skills/blob/main/skills/engineering/implement/SKILL.md)** — 构建由 spec 或一组 tickets 描述的工作，在预先同意的 seam 上驱动 `/tdd`，并在提交前用 `/code-review` 收尾。
- **[wayfinder](https://github.com/mattpocock/skills/blob/main/skills/engineering/wayfinder/SKILL.md)** — 把一大块超过单个 agent session 容量的工作规划成 issue tracker 上的共享 investigation tickets 地图——逐个解决它们，直到通向目的地的路变清晰。

**Model-invoked**

- **[prototype](https://github.com/mattpocock/skills/blob/main/skills/engineering/prototype/SKILL.md)** — 构建一个一次性 prototype 来回答设计问题：状态/逻辑问题用可运行终端 app，或者在同一路由中可切换的多个极端不同 UI 变体。
- **[diagnosing-bugs](https://github.com/mattpocock/skills/blob/main/skills/engineering/diagnosing-bugs/SKILL.md)** — 面向困难 bug 和性能回归的纪律化诊断循环：reproduce → minimise → hypothesise → instrument → fix → regression-test。
- **[research](https://github.com/mattpocock/skills/blob/main/skills/engineering/research/SKILL.md)** — 针对高可信一手来源调查一个问题，并把发现保存为 repo 中带引用的 Markdown 文件，以 background agent 运行。
- **[tdd](https://github.com/mattpocock/skills/blob/main/skills/engineering/tdd/SKILL.md)** — 使用 red-green-refactor 循环的测试驱动开发。一次构建一个 vertical slice 来实现功能或修复 bug。
- **[domain-modeling](https://github.com/mattpocock/skills/blob/main/skills/engineering/domain-modeling/SKILL.md)** — 主动构建并打磨项目的 domain model——用 glossary 挑战术语，用边界场景做 stress-test，并内联更新 `CONTEXT.md` 和 ADR。
- **[codebase-design](https://github.com/mattpocock/skills/blob/main/skills/engineering/codebase-design/SKILL.md)** — 用于设计 deep modules 的共享纪律和词汇：把大量行为放在小接口背后，放置在干净 seam 上，并可通过该接口测试。
- **[code-review](https://github.com/mattpocock/skills/blob/main/skills/engineering/code-review/SKILL.md)** — 对固定点之后 diff 做双轴 review：**Standards**（是否遵循 repo 的编码标准，以及 Fowler smell baseline）和 **Spec**（是否忠实实现源 issue/PRD），作为并行 sub-agents 运行，让两者互不污染。

### Productivity

通用工作流工具，不专用于代码。

**User-invoked**

- **[grill-me](https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md)** — 围绕一个 plan 或 design 对用户进行 relentless interview，直到决策树的每个分支都被解决。
- **[handoff](https://github.com/mattpocock/skills/blob/main/skills/productivity/handoff/SKILL.md)** — 把当前对话压缩成 handoff document，让另一个 agent 能继续工作。
- **[teach](https://github.com/mattpocock/skills/blob/main/skills/productivity/teach/SKILL.md)** — 使用当前目录作为有状态教学 workspace，跨多个 sessions 教用户一个新 skill 或概念。
- **[writing-great-skills](https://github.com/mattpocock/skills/blob/main/skills/productivity/writing-great-skills/SKILL.md)** — 编写和编辑好 skills 的参考：让 skill 可预测的词汇和原则。

**Model-invoked**

- **[grilling](https://github.com/mattpocock/skills/blob/main/skills/productivity/grilling/SKILL.md)** — 围绕一个 plan 或 design 对用户进行 relentless interview，直到决策树的每个分支都被解决。它是 `grill-me` 和 `grill-with-docs` 背后的可复用循环。
