---
title: "Fable 实战指南：找出你的未知项"
source: "https://x.com/trq212/status/2073100352921215386"
canonical: "https://x.com/trq212/article/2073100352921215386"
author: "Thariq (@trq212)"
published: "2026-07-03T17:43:35.000Z"
saved: "2026-07-05 20:15:17 +0800"
tags: [ai-coding, claude, agentic-coding, x-article]
---

# Fable 实战指南：找出你的未知项

Thariq [@trq212](https://x.com/trq212)

![文章封面图](../../assets/a-field-guide-to-fable-finding-your-unknowns/cover.jpg)

使用 Claude Fable 5 时，它不断让我重新学到一个老道理：地图不是领土。
地图，也就是对待完成工作的表示，是我的 prompts、skills 和 context，是我给 Claude 的东西。领土才是工作真正发生的地方：代码库、真实世界，以及其中实际存在的约束。

地图和领土之间的差异，就是我所说的 unknowns。当 Claude 遇到一个 unknown 时，它必须根据自己对我想要什么的最佳猜测来做决定。要完成的工作越多，Claude 可能遇到的 unknowns 就越多。
Fable 是第一个让我觉得工作质量受限于我澄清 unknowns 能力的模型。
重要的是，提前规划并不总是足够。你可能会在实现深处才发现 unknowns，或者你的 unknowns 会让你意识到：你其实应该用完全不同的方式解决问题。
我发现，和 Fable 一起工作，是一个在实现前、实现中、实现后不断发现自己 unknowns 的迭代过程。
我在这里做了一些用于发现 unknowns 的示例 artifacts，但请先读完，建立什么时候使用它们的直觉。

## 认识你的 unknowns

什么是你的 unknowns？当我带着一个问题来找 Claude 时，我通常会把它拆成 4 类：
- **Known Knowns：**本质上就是我 prompt 里写出来的东西。我告诉 agent 我想要什么？
- **Known Unknowns：**有哪些东西我还没想清楚，但我知道自己还没想清楚？
- **Unknown Knowns：**有哪些东西明显到我永远不会写下来，但只要看到我就能认出来？
- **Unknown Unknowns：**有哪些东西我完全没考虑过？有哪些知识是我不知道自己不知道的？我知道一件事可以好到什么程度吗？

最好的 agentic coders 通常 unknowns 相对较少。看 Boris 或 Jarred 这样的人写 prompt，我很明显能感觉到，他们非常细致地知道自己想要什么。他们既和代码库高度同步，也和模型行为高度同步。
但他们也会假设 unknowns 的存在。在很多意义上，减少并规划你的 unknowns，就是 agentic coding 的核心技能。幸运的是，这是一项可以通过和 Claude 一起工作而提升的技能。

## 帮 Claude 帮你

指挥 Claude 是一种微妙的平衡。如果你太具体，Claude 会照着你的指令走，即使这时转向可能更合适。如果你太模糊，Claude 往往会基于行业最佳实践来做选择和假设，而这些最佳实践未必适合你的任务。
当你没有处理自己的 unknowns 时，两边都会失败。你不知道前方什么时候布满障碍，也不知道什么时候道路其实很清晰、但你仍然希望 Claude 能够偏航调整。
Claude 可以帮助你更快发现 unknowns。它能非常快地搜索你的代码库和互联网，而且它对大多数主题的了解都比你多。它也能更快地从失败中迭代。
这个过程中最重要的部分，是给 Claude 你的起点上下文。比如，告诉它你当前思考到哪一步；说明你对这个问题和代码库的经验；让它像一个思考伙伴一样和你协作。
我之前写过关于和 Claude 一起使用 HTML 的文章；在几乎所有这些场景里，HTML artifact 都是可视化和表示内容的最佳方式。
在这篇文章里，我会详细说明一些我用来挖掘 unknowns 的模式。我不是每次都会用所有技巧，但这是一组很有用的技巧库。

## 实现前

### Blind Spot Pass

开始工作时，最有用的事情之一，是理解你的盲点。比如，如果你要在代码库的一个新区域写功能，或者用 Claude 帮你做一个不熟悉的工作，比如迭代设计，你很可能会有很多 unknown unknowns。
你可能不知道该问什么问题，不知道“好”应该长什么样，不知道过去做过哪些历史工作，也不知道要避开哪些坑。
要做到这一点，你可以让 Claude 帮你找出 unknown unknowns，并向你解释。我喜欢直接使用 “blindspot pass” 和 “unknown unknowns” 这两个词。通常也很重要的是，给它关于你是谁、你知道什么的上下文。

### 示例 Prompts

> “I'm working on adding a new auth provider but I know nothing about the auth modules in this codebase. Can you do a blindspot pass to help me figure out my relevant unknown unknowns and help me prompt you better.”
> “I don’t know what color grading is but I need to grade this video. Can you teach me to understand my unknown unknowns about color grading, so that I can prompt better?”

### Brainstorms and prototypes

当我在一个有大量 unknown knowns 的领域工作时，也就是那些标准只有看到后我才知道怎么定义的地方，我喜欢让 Claude 和我一起 brainstorm 和 prototype。
在 prototyping 的早期识别并说清楚 unknown knowns 非常有价值，因为如果在 implementation 阶段才发现它们，代价可能会相对较高。一个 feature 或 spec 的小改动，可能导致代码里截然不同的实现，而且你的 agent 可能更难回滚之前的修改。
比如，你可能只是想看看把一个按钮加到 frame 里是什么效果，而不想为了它接一个 backend route，或在 frontend 里维护额外 state。
视觉设计对我来说很难表达，但我看到之后就知道自己想要什么。在这些情况下，我会要求它为一个 artifact 给出几种设计方向。
我也几乎会在每次 coding session 开始时先做一轮 exploration 或 brainstorming。这帮助我从定义项目 scope 的意图开始。Claude 经常能找到我会错过的高价值路径，有时也会只见树木不见森林。Brainstorming 可以防止我把 scope 设得太窄或太宽。

示例 prompts：
"I want a dashboard for this data but I have no visual taste and don't know what's possible. Make me an HTML page with 4 wildly different design directions so I can react to them.”
“Before wiring anything up, make a single HTML file mocking the new editor toolbar with fake data. I want to react to the layout before you touch the treal app."
"Here's my rough problem: users churn after onboarding. Search the codebase and brainstorm 10 places we could intervene, from cheapest to most ambitious. I'll tell you which ones resonate."

Interviews

当我已经做了足够的 brainstorming 后，我很可能仍然有 unknowns。
这种情况下，我会让 Claude 针对任何 unknowns 或 ambiguities 来采访我。让 Claude 采访你时，尽量给它关于问题的上下文，以引导它的问题。下面是一些例子。

示例 prompts：
"Interview me one question at a time about anything ambiguous, prioritize questions where my answer would change the architecture."

References

有时你没法详细描述自己想要什么。比如，你可能没有相应的语言，或者它太复杂，要描述清楚会花很长时间。
这种情况下，最好的答案是 reference。你可以提供图、文档或图片，但绝对最好的 reference 是 source code。
如果你有一个 library 用某种方式实现了你想要的东西，或者有一个你很喜欢的 design component，就把 Fable 指向那个 folder，并告诉它要看什么，即使它是用另一种语言写的。
这也是 Claude Design 的工作方式。你不需要手动给它一个文件（当然也可以）。你可以把它指向某个你喜欢的网站模块，它会读取底层代码，而不只是截图。这能提供关于 markup、structure 以及组件实际构建方式的丰富细节。

示例 prompts：
This Rust crate in vendor/rate-limiter implements the exact backoff behavior I want. Read it and reimplement the same semantics in our TypeScript API client.

Implementation Plans

当我觉得已经准备好实现时，我通常会让 Claude 给我做一个 implementation plan 供我 review，重点关注最可能发生变化的部分，比如 review data models、type interfaces 或 UX flows。这能让 Claude 暴露出一些我实际上可能需要调整的东西。

### 示例 Prompts

Write an implementation plan in HTML, but lead with the decisions I'm most likely to tweak with: data model changes, new type interfaces, and anything user-facing. Bury the mechanical refactoring at the bottom, I trust you on that part."

## 实现中

Implementation notes

当我对 plan 满意后，我会新开一个 session，并把相关 artifacts 传给 prompt。比如，我可能会传入一个 spec file 和一个 prototype，然后让 agent 去实现它。
但事实是，无论你规划得多充分，总会有 unknown unknowns 潜伏着。agent 在工作过程中可能会因为发现代码里的某个 edge case，而不得不采取不同路线。
我会要求 Claude Code 维护一个临时的 `implementation-notes.md`（或 `.html`）文件，在里面记录它做出的决策，这样我们可以从下一次尝试中学习。

示例 prompts：
"Keep an implementation-notes.md file. If you hit an edge case that forces you to deviate from the plan, pick the conservative option, log it under 'Deviations', and keep going."

Post implementation

Pitches and explainers

发布某个东西时，最重要的部分之一是获得 buy-in 和 approvals。在最终文档里构建 pitch 和 explainer artifacts 可以帮助：
加速理解，因为 reviewers 一开始也带着和你一样的 unknowns。
加速 approvals，因为专家希望看到你已经考虑到了他们会预判到的 unknowns 和常见 failure points。

示例 prompts：
"Package the prototype, the spec, and the implementation notes into a single doc I can drop in Slack to get buy-in. Lead with the demo GIF."

Quizzes

经过一次长时间工作 session 后，Claude 可能完成的事情比我意识到的多得多。读 code diffs 只能让我浅层理解发生了什么，因为很多行为会取决于已有代码路径。
在 Claude 给我大量上下文之后，让它围绕这次 change 来 quiz 我，可以帮助我理解发生了什么。我只有在完美通过 quiz 后才会 merge。

示例 prompts：
> “I want to make sure I understand everything that's happened in this change. Give me a HTML report on the changes for me to read and understand with context, intuition, what was done, etc. and a quiz at the bottom on the changes that I must pass.”

How this comes together: launching Fable

Fable 的 launch video 完全由 Claude Code 剪辑。这对我来说是一个新领域，我绝不是专家。
所以我从自己已知的东西开始。我知道 Claude 可以用代码剪辑视频并转录它们，但我不确定它是否足够准确。于是我让 Claude 向我解释 Whisper 这样的 transcription 是如何工作的，以及我是否能用 ffmpeg 准确剪掉 ums 或较长停顿。
我希望 Claude 创建一个和我说话词句同步的 UI，但不确定它是否能做到，所以我让 Claude 使用 Remotion 和 transcription 创建一个 prototype video，看看是否可行。
最后，视频本身看起来有点灰暗。我知道这是 color grading 的结果，但我并不真正知道 color grading 是什么。我的第一次尝试是让 Claude 做几个 variation 让我选择，但我意识到，对于 color grading 来说，我不知道什么才叫“好”。所以我转而让 Claude 教我 color grading，帮助我发现自己的 unknowns。
你可以在这里观看更深入的解释。

Matching the Map and Territory

模型越好，用正确方法能完成的事情就越多。当一个 long-horizon task 返回了错误结果，很可能说明你需要花更多时间定义自己的 unknowns，或创建一个允许 Claude 在 unknowns 中即兴调整的 implementation plan。
每一个 explainer、brainstorm、interview、prototype 和 reference，都是一种低成本方式，能在修复成本变高之前发现你原本不知道的东西。
所以，下一个项目开始时，先让 Claude 帮你找出你的 unknowns。

## 提到的链接

- https://x.com/trq212/status/2064826394589442448/video/1
- https://www.google.com/url?q=http://implementation-notes.md&sa=D&source=editors&ust=1783101769359369&usg=AOvVaw1Iqvg51JpzkrkRtHHIjyOL
- https://www.google.com/url?q=https://www.linkedin.com/in/jarred-sumner-a8772425&sa=D&source=editors&ust=1783101769343738&usg=AOvVaw1jFeuVIbBffAC5464Tk_TD
- https://www.google.com/url?q=https://x.com/ClaudeDevs/status/2064399512664526853&sa=D&source=editors&ust=1783101769363678&usg=AOvVaw1MyZd5YMjjShztWHzo8N9u
- https://www.google.com/url?q=http://implementation-notes.md&sa=D&source=editors&ust=1783101769359896&usg=AOvVaw1wFqbnqbAuO_GYnGk8_1bh
- https://x.com/trq212/status/2052809885763747935
- https://thariqs.github.io/html-effectiveness/unknowns/
- https://www.google.com/url?q=https://www.linkedin.com/in/bcherny&sa=D&source=editors&ust=1783101769343560&usg=AOvVaw0NSN4RLOEaJ_k7bIWfat2t

---

Source: [A Field Guide to Fable: Finding Your Unknowns](https://x.com/trq212/status/2073100352921215386)
Canonical: [https://x.com/trq212/article/2073100352921215386](https://x.com/trq212/article/2073100352921215386)
Saved: 2026-07-05 20:15:17 +0800
