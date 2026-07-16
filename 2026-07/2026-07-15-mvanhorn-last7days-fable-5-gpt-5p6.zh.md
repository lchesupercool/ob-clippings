---
title: "/last7days：Fable 5 和 GPT-5.6，数千个赞同票说明真正有效的方法"
source: "https://x.com/mvanhorn/status/2077510447016890433?s=52"
author: "Matt Van Horn (@mvanhorn)"
published: 2026-07-15
saved: 2026-07-15
tags: [AI, 智能体, GPT-5.6, Claude, 提示词]
---

# /last7days：Fable 5 和 GPT-5.6，数千个赞同票说明真正有效的方法

![](../../assets/2026-07-15-mvanhorn-last7days-fable-5-gpt-5p6/cover.png)

这是第一次，两个最新的前沿模型同时上线，而这个窗口期刚刚过去一周。不妨叫它 /last7days。本周我把 [/last30days](https://x.com/slashlast30days) 对准了 [Claude Fable 5](https://www.anthropic.com/news/claude-fable-5-mythos-5) 和 [GPT-5.6](https://openai.com/index/gpt-5-6/) 二十多次，每次运行聚焦一个角度：effort 档位、过度提示、编排、成本、迁移错误，各种问题都看了一遍。返回的是 Reddit 讨论串、YouTube 解析、TikTok、LinkedIn 帖子、GitHub issue 和 Hacker News 争论，全都来自人们真正同时上手这两个模型后的短短几天。我读完了每一簇内容。下面是十条真正留下来的实践，每条都有真实用户的凭据，以及你今天就能粘贴使用的内容。

这个窗口期确实就这么短。Fable 5 于 6 月 9 日发布，随后进入一次 [为期 19 天的出口管制下线](https://www.anthropic.com/news/redeploying-fable-5)，7 月 1 日重新面向所有人开放。GPT-5.6 于 7 月 9 日公开发布。写下这些时，这是两者重叠的第七天。而在这条窄缝里，两家公司都发布了一份提示词指南，并落在同一句话上：你正在过度提示，停下来。

我每天都用这两个模型，所以开始时我已经有自己的判断。社区给出了更好的判断，而且他们的判断还带着赞同票数。

## 1. 🎯 给它目标。删掉步骤。

新时代最清晰的一句话来自 TikTok，当时两家公司甚至还没发完自己的指南。

> 大多数人使用 Fable 5 的方式完全错了。你过度指示它时，Fable 5 的表现会下降。不需要角色，不需要一步一步来，不需要“展示你的推理”——全都是浪费 token。以目标为中心的提示：你想做什么、相关背景、哪些事情越界、完成是什么样子。然后让它工作。

- thinkwithv, [TikTok](https://www.tiktok.com/@thinkwithv/video/7658032347421347086), 187 个赞

两份官方指南都逐字支持这一点。Anthropic 的 [Fable 5 指南](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5) 写道：“为以往模型发展出来的技巧，对 Claude Fable 5 来说往往过于规定化，并且可能降低输出质量。”OpenAI 的 [GPT-5.6 指南](https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6)：定义结果和完成标准，然后“给模型留下选择高效路径的空间”。你从 GPT-5.5 和 Opus 时代带来的精心编号作战手册，并不是中性的包袱。它正在主动让新模型变笨。

Anthropic 甚至直接给出了替代形状。现在完整的提示词就是这样：

## 2. ✂️ 每条指令只说一次

OpenAI 在自己的指南上跑了评测并公布了数字，很快，一位 X 上的构建者把它做成了本周最有用的截图。

> 更精简的提示词优于详细提示词。内部评测显示，删除冗余指令让分数提升了 10-15%，同时 token 减少 41-66%，成本降低 33-67%。

- @oliviscusAI, [X](https://x.com/oliviscusAI/status/2076004557751308647), 21 个赞

再读一遍：删字让模型更聪明，还能把账单最多砍掉三分之二。指南中的第二条警告，正是大多数生产提示词会失败的地方：“相互冲突的规则比缺少细节更容易制造不稳定。”每一个已经存活六个月的系统提示词，里面都会有两条悄悄互相矛盾的规则，而模型现在会花推理 token 试图同时满足二者。修复办法就是一个下午的工作：删掉任何重复说过的东西，删掉模型已经不再需要的风格说明，保留成功标准和停止条件、任何涉及安全或权限的内容，以及你要求的输出形状。然后读一遍剩下的内容，专门找矛盾。

## 3. 🪆 让前沿模型指挥更便宜的模型

整个窗口期中赞同票最多的实践不是提示词技巧。它是一张组织结构图。

> Anthropic 刚刚基准测试了“Fable 5 负责编排，便宜模型负责执行”：以 46% 的成本达到 96% 的性能。你今天就能在 Claude Code 里跑这个模式。

- [r/ClaudeAI](https://www.reddit.com/r/ClaudeAI/comments/1ur2ml9/anthropic_just_benchmarked_fable_5_orchestrates/), 1,246 个赞同票

[Theo 的 Fable 5 指南](https://www.youtube.com/watch?v=8GRmLR__OGQ)有 117,924 次观看，走的是同样的玩法：Fable 负责设计和审查，他直接告诉它，中间“基本任何事情”都可以交给更便宜的模型处理。TikTok 上流传的截图式手册把它叫作 [顾问模式](https://www.tiktok.com/@estop845/video/7659939531201727774)。Anthropic 的指南解释了为什么它现在有效，而以前只算半有效：Fable 5“在调度和维持并行子智能体方面明显更可靠”。可以直接借走的编排器提示是：

## 4. 🥊 让两个模型互相打

这是我在整轮扫读里最喜欢的发现。r/ClaudeCode 上有一位开发者拒绝选边站，而是把前沿模型之间第一次真正的正面对决做成了固定工作流。

> 我把同一个问题陈述和目标交给 fable 和 sol 5.6 xhigh，让它们分别设计，谁赢了就执行谁的设计，输家则通过在检查点拆解它来挽回面子。而且我还让它们记分。

- SpaceCowboy077, [r/ClaudeCode](https://www.reddit.com/r/ClaudeCode/comments/1uvjlmr/comment/oxbq5wr/), 302 个赞同票

两个前沿模型，每个任务一次设计对决，输家成为对抗性审查者。AI LABS 解析的评论区也出现了同一思路的更平静的生产版本：Codex 处理大部分审查，最后再让 Fable 过一遍，它“经常能抓到 Codex 审查漏掉的问题”。不管哪种方式，原则都成立。你第一次负担得起第二个前沿意见，而拿到最好结果的人正在购买这个意见。

## 5. 💸 按每个任务的美元成本来选模型

基准测试争论大约一天就烧完了。一直被引用的是成本，而最锋利的一句话说的不是旗舰模型，而是中档模型。

> Terra 以四分之一的成本和 fable 打平 💀

- ethotopia, [r/OpenAI](https://www.reddit.com/r/OpenAI/comments/1us7nml/comment/owlnqkv/), 183 个赞同票

证据堆得很快。Sam Altman 说 [GPT-5.6 在智能体式编码上 token 效率提高了 54%](https://www.cnbc.com/2026/07/09/open-ai-sam-altman-chatgpt-5-6-sol.html)。[Varun Mayya 的解析](https://www.instagram.com/reel/DalTEPIBoRu/)有 47,413 次观看，其中 Sol 用三分之一的 token 达到了 Mythos 级输出。篱笆的另一边，Fable 5 的计费是输入每百万 token 10 美元、输出每百万 token 50 美元，这也解释了为什么一位 YouTuber 在看到单条开场消息就花三四十美分后，把视频标题写成了 [别再在 Claude Code 里使用 Fable 5（它正在拖你的后腿）](https://www.youtube.com/watch?v=d9XCX0PcOq0)。认真做事的人已经不再只选一个模型。他们在写路由表：前沿模型用于困难、漫长、模糊的工作；Terra、Luna 或低 effort 的 Fable 用于所有只需要完成的事。

## 6. 🎚️ effort 档位可以调高。但这不意味着你应该调高。

两个模型现在都把它们思考得有多用力暴露成了一个设置，而这个窗口期最反直觉的发现是，把它拉满通常是个错误。

> 在 Fable 5 上，把 [effort] 调高实际上是一个巨大的陷阱。

- AI LABS, ["如何比 90% 的人更好地使用 Claude Fable 5"](https://www.youtube.com/watch?v=GM7-ei-4Xc8), 27,077 次观看

Anthropic 的指南说得很明确：high 是默认值，xhigh 只适合对能力最敏感的工作，而且“Claude Fable 5 上较低的 effort 设置仍然表现良好，并且经常超过以往模型 xhigh 的性能”。OpenAI 对 GPT-5.6 的推理层级也说了同样的事：只有在“测得的评测收益”支持时才高于 high。额外 effort 不只是花钱和花时间。它还会引来镀金、未经请求的重构、没人要的辅助函数。指南自带的对策可以直接粘贴：

## 7. 🗒️ 给它一个可以保留的笔记本

Anthropic 手册中传播最广的解读一开头就给出了这个几乎没人试过的技巧，而它是这份清单里最便宜的复利升级。

> 给 Fable 一个写笔记的地方。一个简单的 Markdown 文件，让它记录每次运行中学到的东西。它做了哪些修正，哪些方法有效，哪些无效。

- raycfu, [Instagram](https://www.instagram.com/reel/Day9pQbP2up/), 17,304 次观看

一个 Markdown 文件可以把金鱼变成同事。你今天做出的每个纠正，都是下个月不用再做的纠正。直接来自指南：

## 8. 🧾 没有工具凭据，一概不要相信

长时运行是两个模型的头条能力，而人们用惨痛方式学到的失败模式，是那种自信汇报进展、实际却从未发生的报告。一位 Codex 用户描述了真正赢得信任是什么样子。

> GPT-5.6 Sol 是第一个能够持续保持上下文、使用工具、检查自己的工作，并且不用人盯着就能完成任务的模型。

- @raul_rcl22, [X](https://x.com/raul_rcl22/status/2077288097075577194), 2 个赞

检查自己的工作是一种你要安装进去的行为。它不会随订阅免费附赠。Anthropic 测试了一条指令，并报告说它“几乎消除了编造状态报告，即使是在专门设计来诱发这类报告的任务上也是如此”。一位 Reddit 老手也从一线给出同样建议：“我总是让它跑一次最终对抗性审查。它总能发现错误”([jtmonkey, r/ChatGPT, 125 个赞同票](https://www.reddit.com/r/ChatGPT/comments/1utxylf/comment/owzf629/))。这条指令是：

## 9. 🤐 永远不要要求 Fable 5 展示它的推理

这是迁移过程里隐藏的坑。根据 Anthropic 的指南，在 Fable 5 上，提示词如果“要求模型把它的内部推理作为响应文本回显、转写或解释”，可能会触发 reasoning_extraction 拒绝类别，而这个拒绝会悄悄把你的请求回退到 Claude Opus 4.8。你继续为前沿模型付费，却在无声无息中不再得到它。

你为旧版 Claude 写下的每一句“解释你的思考”“带我看一遍你的推理”“展示你的工作过程”，现在都是地雷。用 grep 搜索你的技能和系统提示词，找到它们并删除。如果你需要查看推理，API 已经会返回结构化的思考块；读取那些内容，而不是要求模型把自己叙述到拒绝里。

## 10. 🎛️ 设置 verbosity 旋钮，然后别再唠叨

新模型暴露出的最小气的习惯：在每个提示词里都塞进“保持简洁”，而这正是规则 2 告诉你要删除的冗余指令。GPT-5.6 有一个 `text.verbosity` 设置，low、medium 或 high，可以在 API 层一次性固定默认值。Fable 5 对单条可读性指令的遵循效果，好过一叠提醒。社区解释了为什么简洁值得工程化处理，用的是 Claude 窗口期被引用最多的一句话。

> 如果 Sonnet 5 能用三分之一的输出做到几乎和 Opus 4.8 一样好，那我加入。Opus 4.8 说的话比一个狂喝糖水的幼儿还多。

- trevormead, [r/ClaudeAI](https://www.reddit.com/r/ClaudeAI/comments/1ujwggp/comment/our5env/), 575 个赞同票

把旋钮设置一次。说一次。继续前进。

## 🧵 十条背后共同指向什么

把这十条排在一起，它们其实是同一条指令穿了十套衣服：你的杠杆已经从提示词移到了流程里。两家公司在同样的两周内独立收敛到这一点，而在这个行业里，这已经接近事实标准。人们实际正在兑现的收益，来自把正确任务按正确 effort 路由给正确模型，让前沿模型指挥更便宜的模型，强制要求有证据支撑的状态报告，以及删掉过去提示词里一半的话。而在这一切下面，还坐着一个令人谦卑的事实，来自一位有四十年经验的老手在一条病毒式构建视频下的评论：“AI 是简单的部分。真正的工作是架构、调试、测试、安全、UI、集成”([martyman1964, TikTok, 115 个赞](https://www.tiktok.com/@dhaibuilds/video/7660125139555683606))。模型不再是瓶颈。现在变量是你。

## 🛠️ 可复制的实战模式

过去两周给出的行动建议，一遍说完：

- 把一个旧提示词改写成目标、背景、边界和完成定义。删掉步骤。自己比较输出。

- 审计你最长的系统提示词，找重复和矛盾。按 OpenAI 的评测，这项清理能用最高 67% 更低的成本，换来 10 到 15% 的质量提升。

- 在 Claude Code 里运行编排器模式：Fable 负责计划和审查，便宜模型负责执行。96% 的性能，46% 的成本。

- 对任何困难任务，把同一个目标交给 Fable 和 Sol，用 xhigh，让胜者的设计上线，并让输家在检查点审查它。

- 默认把 effort 保持在 high。把日常工作路由给 Terra、Luna 或低 effort。把 xhigh 和 max 留给真正配得上的问题。

- 给模型一个笔记文件，要求任何进度声明之前必须有工具凭据审计，并用 grep 搜索你的 Fable 提示词里的“show your reasoning”，删除每一个命中。

## 📊 所有智能体均已回报

本文汇总自二十多次 [/last30days](https://x.com/slashlast30days) 运行，范围覆盖 Reddit、X、YouTube、TikTok、Instagram、LinkedIn、Hacker News、GitHub 和 Digg，时间覆盖 Fable 5 回归后的两周，以及 GPT-5.6 公开后的六天。最响亮的信号：编排器基准测试，1,246 个赞同票。最锋利的工作流：Fable 对 Sol 的设计对决，302 个赞同票。最清晰的论点：thinkwithv 的目标聚焦提示词，187 个赞。最硬的凭据：OpenAI 自己的 10 到 15% 质量提升，来自 41 到 66% 更少的 token。主要声音：SpaceCowboy077、thinkwithv、raycfu、ethotopia、@oliviscusAI；r/ClaudeAI、r/ClaudeCode、r/OpenAI。

这篇文章是一叠搜索和一个下午阅读的结果。我没有采访任何人。引文是真实的，凭据是真实的，你也可以自己跑同样的查询。可直接使用的内容来自两份官方指南。社区和模型制造者在同一周写出了这本手册；我只是把它整理到一起。
