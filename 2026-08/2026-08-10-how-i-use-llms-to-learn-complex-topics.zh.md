---
title: "我如何使用 LLM 学习复杂主题"
source: "https://laurentiugabriel.github.io/blog/articles/how-i-use-llms-to-learn/"
author: "Laurentiu Raducu"
published: 2026-08-09
saved: 2026-08-10
translation_of: "2026-08-10-how-i-use-llms-to-learn-complex-topics.md"
tags: [llm, learning, simulation, visualization]
---

# 我如何使用 LLM 学习复杂主题

*LLM 被用于各种事情，学习新事物是其中最主要的使用场景之一。*

我认识的许多工程师会将生成式 AI 用于多种用途，例如构建 PoC、内部工具或仪表盘，甚至学习新东西。我个人觉得 LLM 解释事物的风格很难读下去。它实在过于简单，而且根据所用 emoji 的数量，有时也有点烦人。

![如何制造芯片——用类似 RollerCoasterTycoon 的模拟来解释](../../assets/2026-08-10-how-i-use-llms-to-learn-complex-topics/ChipTycoon.png)

在分析可能拖慢数据中心建设的新 AI 瓶颈时，我意识到自己对芯片生产还有许多不了解的地方。浏览网页时，我问自己：如果有一款游戏，能带你走完在晶圆厂制造芯片的全过程，会怎么样？以这种方式学习肯定更容易记住，因为你可以把概念映射到游戏中的对象上。于是我决定试一试，结果确实非常不错。

## 流程

我不会只是让 AI 解释某个主题，而是采用下面的流程：

- 在计划模式中（使用 CC 或 OpenCode），我让模型为主题 X 构建基础知识。
- 我让它审查上一步所构建知识库的准确性。
- 接着，我让它以低多边形、类似 RollerCoaster Tycoon 的动画形式构建该主题的模拟。我也会加入一些 UX 要求，例如页面需要在大屏幕和小屏幕上都可见，并提供控件，让我能随时暂停流程等。
- 然后我把它推送到一个新仓库，并为其启用 GitHub Pages。

## 结果

最终你会得到一个漂亮的动画，它百分之百准确，而且没有幻觉。对我来说，这种方法比阅读从 Google 找到的没完没了的材料，或努力消化语言模型吐出的一串项目符号列表，要有效得多。

我专门用这种方式学习了芯片制造，并将成果发布在这个网站上：[ChipTycoon](https://laurentiugabriel.github.io/ChipTycoon/)。你可以跟随一辆小车，从收集沙子开始，一直到芯片最终制造完成并被运送到数据中心。

在视觉上，你可以跟随小车，也能看到它如何发生变化。由于采用低多边形风格，某些细节可能会缺失，但它仍然能很好地展示产品经过制造流程所需的许多步骤后，是如何逐步变化的。

## 如何进一步改进

假设低多边形设计需要太多想象力，你很难真正看清那堆石英砂离开熔炉后发生了什么。为了把它转化成更逼真的表现形式，你可以使用我的[将图片转换为 3D 对象的 skill](https://github.com/LaurentiuGabriel/unreal-game-assets-creation-skill)，并把生成的对象映射到模拟中。这样就能得到更准确的设计。

你还可以在模拟中加入挑战。尝试回答有关芯片制造流程前一步骤的问题，会极大帮助你保留这些知识。也可以加入直观的谜题，帮助你学得更好。

看看我创建的其他页面：

- [火箭发动机是如何制造的](https://laurentiugabriel.github.io/rocket-engine/)
- [LLM 如何工作](https://laurentiugabriel.github.io/token-town/)
- [F1 发动机是如何制造的](https://laurentiugabriel.github.io/engineworks/)
- [EUV 设备是如何制造的](https://laurentiugabriel.github.io/euv-lithography/)
