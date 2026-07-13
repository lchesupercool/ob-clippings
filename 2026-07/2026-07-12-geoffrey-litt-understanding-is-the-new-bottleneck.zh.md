---
title: "理解是新的瓶颈"
source: "https://www.geoffreylitt.com/2026/07/02/understanding-is-the-new-bottleneck"
author: "Geoffrey Litt"
published: "2026-07-02"
saved: "2026-07-12"
tags: [AI, Agent, AI Coding, 人机协作, 认知债务]
---

### 2026 年 7 月

# 理解是新的瓶颈

这是我于 2026 年 7 月在
[AI Engineer](https://www.ai.engineer/)
大会上所作演讲的文字版，也曾以
[推文串的形式分享。](https://x.com/geoffreylitt/status/2072522251300409556)

![标题幻灯片：理解是新的瓶颈。Geoffrey Litt，Notion 的设计工程师。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-01.webp)

**一个大胆观点：我认为，理解我们的 agent 所写的代码依然很重要！**

在这场演讲中，我会解释原因，并展示一些高效理解代码的方法。好了，让我们开始吧。

![一个人被不断增长的一堆由 agent 编写的代码包围的漫画。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-02.webp)

Agent 正在为我们编写越来越多的代码，而我们都知道，跟上它变得越来越难。

但好消息是：理解代码有很多方法！逐行阅读 diff 并不是唯一的方法。

![列出技术的幻灯片：代码讲解文档、测验、微型世界。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-03.webp)

这场演讲的大部分内容都将围绕我发现有助于理解我的 agent 正在构建的系统的技巧：

- 代码讲解文档
- 用于检验我理解程度的测验
- 我可以玩耍以理解系统的微型世界

但首先，我们必须问一个更基础的问题……

## 为什么要理解？

![写着“为什么要理解？”的幻灯片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-04.webp)

**为什么？为什么要理解？**

我们现在难道不该让自己脱离循环，让 agent 自己循环吗？随着 agent 变得更聪明，我们身处细节之中的重要性难道不会降低吗？

我认为，许多人——即使是支持理解的人——对这个问题都有一个稍微不正确的答案！

![幻灯片：为了验证而理解。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-05.webp)

一个可能的答案是：我们理解是*为了验证*。我们检查 agent 的工作，看看它是否正确。

“正确”可以意味着很多事：它是否符合规格、架构是否良好……但从根本上说，这是一个赞成 / 反对的问题。

![关于 agent 越来越擅长验证自身工作的幻灯片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-06.webp)

关键是：agent 正越来越擅长验证自己的工作。这很好！我喜欢我的 agent 不犯错。

但嗯。这会把我们人类置于何处？

![幻灯片：为了参与而理解。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-07.webp)

**这就引出了另一个答案：我们可以为了参与而理解。**

你可以了解 agent 在做什么，以确保自己能够成为创作过程中的积极参与者。以下是这件事为何重要……

![将一个项目描绘为与 agent 进行多次迭代循环的图表。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-08.webp)

它从来不只是一个循环！一个项目是与 agent 进行的许许多多次循环。

而你对系统的理解，是你提出下一个想法来推动其演进的能力的一部分。

你需要在脑中拥有一套丰富的概念，才能创造性且流畅地思考如何推进某件事。如果缺乏这种流畅性，你参与项目的能力就会受到实质性限制。

![Margaret Storey 关于认知债务的引言：相关的人类可能只是失去了对全局的把握。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-09.webp)

顺便说一句，这与[认知债务](https://margaretstorey.com/blog/2026/02/09/cognitive-debt/)的理念密切相关，这一理念由 Margaret Storey 和 [Simon Willison](https://simonwillison.net/) 推广。

它就像技术债务：短期内，你可以在不理解正在发生什么的情况下蒙混过关，但它最终会反噬你。

![提出“我们如何建立理解？”的幻灯片，并指向教育以获得启发。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-10.webp)

好吧，所以理解确实重要。

但这引出了下一个问题：*怎么做*？**当我们与 AI 协作并快速推进时，如何建立这种人类理解？**

事实证明，这并不是人类第一次思考如何传达理解。我认为我们可以从教育中获得启发。我们能否借用教育史上最好的理念，并将它们应用于这个问题？

## 技巧 1：讲解

![列出三种技巧的幻灯片，其中“讲解”被突出显示。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-11.webp)

今天我想分享三种技巧，以展示我们如何尝试做到这一点。

第一种：讲解。什么造就了好的讲解？

![展示原始代码 diff 的幻灯片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-12.webp)

每当 agent 完成一些工作时，那就是进行讲解——产出一个制品——的机会。

最朴素的做法是，我们可以阅读代码 diff：发生变化的原始材料。

![提问“最好的讲解会是什么？”的幻灯片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-13.webp)

但如果我们问：

**什么才是*最好的*讲解？**如果你有一个团队——人类或 AI——真正不遗余力地为你把某件事讲解清楚，那会是什么感受？

![由 /explain-diff skill 生成的代码讲解文档截图。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-14.webp)

这里有一个答案。我做了一个名为 [/explain-diff](https://gist.github.com/geoffreylitt/a29df1b5f9865506e8952488eac3d524) 的 skill，我每天都会用它，许多同事也发现它很有价值。

它会输出经过深思熟虑、结构化的代码讲解，格式可以是 HTML、markdown 或 Notion 文档。Notion 是团队协作和讨论这些讲解的好地方。（声明：我在 Notion 工作，所以我有偏见。）

让我们看看其中一份讲解包含什么，以编辑一款电子游戏视角为例。

![讲解中介绍游戏引擎背景信息的部分。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-15.webp)

第一条原则：**教给我背景信息！**

甚至在我们讨论发生了什么变化之前，先帮我理解原本已经存在的内容。在这个例子中，先教我了解游戏引擎。

![讲解中阐明变更目标并解释等距投影的部分。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-16.webp)

第二条原则：**直觉先于细节。**

在任何代码之前，它先说明目标——“用 2D 绘图技巧让花园显得三维”——并解释相关概念，例如什么是等距投影。

所有这些都建立了我对这项变更本质的直觉。它在让作为人类的我跟上进度，使我能成为理解过程中的平等参与者。

[![

](/images/talks/understanding-bottleneck/poster-17.webp?1783098284)](/images/talks/understanding-bottleneck/video-17.mp4)

你也可以通过**交互式图形**来建立直觉。

这里，我通过在花园中拖动石头、观察它们的坐标如何移动，来理解等距视角。

（这使用了 Notion 刚刚发布的一项新功能：现在你可以在页面中嵌入交互式 HTML。）

![将原始 diff 与按散文结构组织的文学化 diff 对比的幻灯片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-17.webp)

我们终于讲到代码了。但典型的 diff 是一堆按字母顺序编辑的文件，没有任何解释。

我所说的“文学化 diff”以散文形式组织——按合理顺序带领读者浏览变更，包含周围的讲解和嵌入的代码片段。它比原始 diff 审阅得更快。

![咖啡馆里一叠打印出来的代码讲解材料的照片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-18.webp)

所有这些的最终产物是一份精美的讲解材料。我仍会阅读代码 diff，但我总是先读这个。

有时我会把它们打印出来带到咖啡馆——干扰更少。

这真是美妙的讽刺：AI 将一项交互式活动变成了一份我可以专注深入阅读的静态纸质报告 :)

![引用 Andy Matuschak“书不起作用”的幻灯片。Quantum Country 的截图。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-19.webp)

只有一个问题：阅读很辛苦 😅

正如 [Andy Matuschak 所说：“书不起作用”](https://andymatuschak.org/books/)！你很容易欺骗自己，以为自己读过了，实际上却没有记住或理解。

我们如何解决这个问题？我从 Andy 和 Michael Nielsen 关于[在文章中嵌入间隔重复测验](https://quantum.country/)的工作中获得了启发。

[![

](/images/talks/understanding-bottleneck/poster-21.webp?1783098284)](/images/talks/understanding-bottleneck/video-21.mp4)

现在，我也在代码讲解中做类似的事。一份讲解的底部有一个交互式测验——五个关于这项变更的问题——我会尝试回答它们。

我的规则是：在我能通过测验之前，我不会把代码发给别人；审阅别人的代码时我也遵循同样的规则。

![将测验描述为 AI 循环中速度调节器的幻灯片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-20.webp)

**测验是一个速度调节器。**与 AI 协作时，循环很容易以快于人类理解速度的速度运行。

测验是一股平衡力量：我机械地问自己“我真的理解吗？”，以便能够始终作为完整的创意参与者。

![链接至 /explain-diff skill 的二维码。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-21.webp)

好了，这就是 explain-diff。[如果你想要，这是这个 skill](https://gist.github.com/geoffreylitt/a29df1b5f9865506e8952488eac3d524)：有两个变体，分别输出 HTML 或 Notion 页面。

## 技巧 2：微型世界

![介绍微型世界的幻灯片，配有 Seymour Papert 的照片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-22.webp)

下一个想法：微型世界。它受富有远见的教育家 Seymour Papert 启发。

![关于 Papert“生活在数学国度”理念的幻灯片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-23.webp)

Papert 有一个优美的想法，他称之为*生活在数学国度*：如果你想学数学，就住在数学国度——就像如果你想学法语，就去法国生活。我们能否构建一种环境，让孩子们因其好奇心而自然地学习数学？

那么，我们如何将其应用于代码？**我们能否创造你能置身其中、并自然直觉到系统如何工作和如何变化的世界？**

[![

](/images/talks/understanding-bottleneck/poster-27.webp?1783098284)](/images/talks/understanding-bottleneck/video-27.mp4)

去年，我在编写一个 Prolog 解释器，却难以直觉地理解内部发生了什么。

我与一个 agent 合作构建了这个调试器，它让我可以逐步执行我的逻辑语言——在时间中来回拖动，查看栈上的内容以及每一步评估了哪些规则。我甚至可以给自己留下评论（“不错，我们正确应用了那条规则”）。

为*我*制作一个用于调试的工具，与让 agent 去调试，两者之间有很大差别——亲自做，是我在过程中建立理解的方式。

[![

](/images/talks/understanding-bottleneck/poster-28.webp?1783098284)](/images/talks/understanding-bottleneck/video-28.mp4)

另一个例子。我正在将个人网站从一个框架迁移到另一个框架，Claude 写了一个脚本来完成这件事。但它很难审阅：我不熟悉新框架，而我所能说的只有“我猜看起来大致没问题”。

所以我让 Claude 给我做一款电子游戏——一个让我亲自一步步完成迁移的指挥中心，同时观察可见效果和文件树如何演进。它产出了一个 UI，我可以点击按钮一步步运行迁移，同时并排运行我的旧网站和新网站。

在这个指挥中心里，我看着新网站逐步焕发生机。这让我获得了与亲手完成相似的理解——但快得多，因为整个体验都已为我铺陈好。

![写着“agent 可以编写代码来帮助我们理解代码！”的幻灯片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-24.webp)

这里的重点是，agent 可以编写一些代码，来帮助我们人类理解其他代码。

这是一件大事！

## 技巧 3：共享空间

![介绍共享空间的幻灯片：作为一个团队一起理解。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-25.webp)

好了，最后一种技巧：共享空间。到目前为止，这些都关乎独自理解……但**当你在团队中工作时，你需要共同理解。**

![关于共享心智模型能够实现高效沟通的幻灯片。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-26.webp)

当你和另一个人拥有相同的心智模型时，你们就能高效沟通。你们拥有能唤起相同图像的共同词汇，因此可以即兴碰撞、延展想法，并展开富有创造力的对话。没有那些共享结构，这些对话就困难得多。

我对创建共享环境感到非常兴奋，团队可以在那里共同建立那种理解。这也正是 Notion 的核心所在。

![在 Notion 内运行的 Claude 和 Cursor agent 的截图。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-27.webp)

最近，我们在 Notion 推出了大量让人类和 agent 共同工作的功能，这样你的整个团队就能建立共享理解，而不是每个人各自在孤岛中工作。

一个很小的例子：你现在可以在 Notion 中运行 Claude 和 Cursor agent。我现在就是以这种方式进行大量编程。

当这些 agent 在 Notion 中制定技术计划时，它默认处在一个协作页面中，因此我可以与团队一起评论它并立即讨论。一起思考，而不是独自思考！

## 重点始终是增强

![幻灯片：人类理解事物如何运作仍然很重要。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-28.webp)

好了，让我们收尾。今天我们讨论了一些关于理解代码的技巧……但实际上，我认为这是一个大得多的问题。

人类理解事物一般而言如何运作仍然很重要！**不仅是为了验证，更是为了参与。**

而且，毫不意外，这并不是一个新想法。它可追溯到我们计算领域最初的起源……

![Alan Kay 的愿景：孩子们通过玩耍和编辑一个交互式模拟来学习物理。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-29.webp)

50 年前，Alan Kay 设想计算机可以成为一种新媒介，比书籍更适合教人们——尤其是孩子们——如何思考世界。

在这张图中，这些孩子看起来像是在 iPad 上看 YouTube，但他们不是。他们正在玩一个交互式游戏，并在游玩的同时编辑代码，以更好地理解物理。这可是 50 年前！！

![宇航员迷因：等等，计算机的重点是创建新的动态模拟以帮助人们理解复杂概念？一直都是。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-30.webp)

现在，希望你理解[这个迷因](https://x.com/geoffreylitt/status/2071362040346955777)了。

重点始终是*增强*，而不只是自动化。

AI 现在让创建模拟变得如此容易，这太美妙了……让 AI 教我们，是计算所开启过的最伟大的可能性之一。

![结束幻灯片：我们可以更深入地进入循环。取决于我们。](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-31.webp)

这让我对未来非常乐观！

**如果我们构建正确的工具，我们现在就能比以往任何时候都更好地理解世界。**我们不必仅仅让自己脱离循环，也可以*更深入地进入循环*。这取决于我们。

*完*

## 相关阅读

如果你喜欢这场演讲，你可能会喜欢我写的其他关于人类与 AI 协作的文章：

- [够了，AI 副驾驶！我们需要 AI HUD](/2025/07/27/enough-ai-copilots-we-need-ai-huds)
  — “任何认真为 AI 设计的人，都应该考虑比副驾驶形态更直接地扩展人类心智的非副驾驶形态……”
- [AI 生成的工具可以让编程更有趣](/2024/12/22/making-programming-more-fun-with-an-ai-generated-debugger)
  — “相反，我使用 AI 构建了一个定制调试器 UI……它让我亲自编程更有乐趣……”
- [像外科医生一样编码](/2025/10/24/code-like-a-surgeon)
  — “识别并委派次要的繁琐任务，这样你就能专注于真正重要的主要事项。”
