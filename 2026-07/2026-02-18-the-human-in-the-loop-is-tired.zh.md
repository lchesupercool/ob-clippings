---
title: 人在回路中，已经累了
description: >-
  关于奖励函数、多巴胺，以及当代码开始自行编写时真实的感受
source: 'https://pydantic.dev/articles/the-human-in-the-loop-is-tired'
saved: '2026-07-17'
date: '2026-02-18'
authors:
  - Laura Summers
categories:
  - Pydantic AI
  - Company
canonical: 'https://pydantic.dev/articles/the-human-in-the-loop-is-tired'
---

> [《人在回路中，已经累了》](https://pydantic.dev/articles/the-human-in-the-loop-is-tired)的 Markdown 版本——规范 HTML 页面。
>
> 作者：[Laura Summers](https://pydantic.dev/authors/laura-summers.md) · 2026-02-18 · Pydantic AI、Company
>
> 相关文章：[Logfire 加入 Stripe Projects](https://pydantic.dev/articles/logfire-stripe-projects.md) · [一个 key 进，不让 key 出](https://pydantic.dev/articles/logfire-ai-gateway.md)
>
> 所有文章：[/articles.md](https://pydantic.dev/articles.md) · 站点索引：[/llms.txt](https://pydantic.dev/llms.txt)

---

又一篇关于 LLM 的观点文章。我知道。请耐心听我说完。

我想尝试为一件我认为大多数开发者此刻都在经历、却还没有时间理清的事找到合适的表述。**用 LLM 编程确实有用，也确实令人失去稳定感。这两件事同时成立。如果我们假装后者没有发生，我们都会精疲力竭。**

在 [Pydantic](https://pydantic.dev)，我们构建供开发者用来[验证数据](https://pydantic.dev/docs/validation/latest/get-started/)、[构建 AI agents](https://pydantic.dev/pydantic-ai)，并在生产环境中[观测其系统运行状况](https://pydantic.dev/logfire)的工具。说得再直白一点，我们做的就是让 LLM 驱动的软件更可靠。而我们*自己*也正经历一段很奇怪的时期。

这不是一篇讨论 AI 是否会取代程序员的文章。它既不是末日论，也不是炒作文章。这是一份来自身处其中之人的坦诚记录：此刻作为开发者是什么感受，以及一些可能真正有所帮助的想法。

## 把手伸进织物

二十出头刚学编程时，我记得自己有一种很鲜明的感觉：编程让我能够把手伸进宇宙的织物，并按自己的意志塑形。当然，那是在我还没遇到太多编译错误之前。但那种触及某个深层、根本的抽象层，能够仅凭逻辑就*造出东西*的感觉，一直留在我心里。

我不是计算机科学毕业生。我是设计师，也是程序员——前者受过正规训练，后者则是自学。我通过痛苦的实践经验，而非学术教育，接触到软件工程的那些形式化方法。如果说有什么不同，那就是在理解它们之后，我会*更*认真地对待这些原则。你若是用艰难的方式赢得了自己对架构和代码质量的看法，它们感受到的就不再像教科书规则，而更像伤疤组织。

那种原始的创造感？正是 2010 年代低代码和无代码工具反复许诺、却从未真正兑现的东西。我年纪大到还记得用 Dreamweaver 做网页，也记得看 Adobe 鼓吹零代码设计工具——它们在底层生成的是彻头彻尾的意大利面条式代码。它总是*差一点*就到位，刚好足以让人瞥见一个近在咫尺的未来（只要你够聪明，能抓住它）。

如果你对当前这一波 AI 工具抱持怀疑，我理解。我们以前就被这样承诺过。但这一次，承诺与现实的距离终于实实在在地缩小到了有意义的程度。而这恰恰让它如此令人不安。

## “代码自己写出来”究竟是什么感受

![](../../assets/the-human-in-the-loop-is-tired/hero.png)

是的，代码（某种程度上）会自己写出来；但负责审查、引导和纠偏的人，感觉反而更糟，而不是更好。

最近我和同事 [Douwe](https://github.com/DouweM) 聊过。他维护 Pydantic AI 框架，也是我认识的、最认真思考如何将 LLM 融入开源工作流的人之一。他说自己每天早晨醒来都会看到三十个 PR——每一个都是某人的 AI 在夜里拉出来的——而他必须对每一个迅速作出判断。把审查本身委托给 AI 的诱惑极大。但正如他说的：*“那样一来，我还在这里做什么？”*。

坦白说，过去几个月里，有些日子我花了将近整整两天，为 LLM 写一份让它执行的计划：痴迷地澄清、细化、再细化，最后它仍会做出某些莫名其妙的蠢事。把一个 React hook 移植到 Storybook 的 story 文件里。读取了错误的计划。编造根本不存在的组件。这些并非能力错误，而是连贯性错误。模型足够聪明，能产出看似合理的代码，却不总是足够聪明，能在一项复杂改动中维持连贯的意图。

这带来一种奇特的新疲惫：*监督*的疲惫。你必须在脑中持守意图，而机器生成海量、大多正确的输出，却仍需要你的眼睛、判断和品味。Douwe 说得很好：过去他会因为和真实的人在开源项目中协作完成一个酷功能而获得多巴胺——帮助某人成为更好的手艺人。现在，他说，*“我写的所有东西都进了某个 AI 黑洞。另一头没有任何人真的学到什么。”* 这种损失是真实的，值得被说出来。

## 强度陷阱

[Simon Willison](https://simonwillison.net/2026/Feb/9/ai-intensifies-work/) 最近重点提到一项 [Berkeley Haas](https://hbr.org/2026/02/ai-doesnt-reduce-work-it-intensifies-it) 研究，研究描述了 AI 的使用如何提升工作的*强度*。那种持续的拉扯：“一天结束时再来一个 prompt，再加一个功能就能让它完美。”我对此感同身受。最近我有一次一直 prompt 到接近凌晨两点，因为我觉得自己离把计划做对已经*非常接近*了。至少当时我是这么想的。

<img src="../../assets/the-human-in-the-loop-is-tired/plan.jpg" alt="这都是计划的一部分" width="300px" style="margin:0 auto;" />

另一位 Pydantic 同事 [Marcelo](https://github.com/kludex) 被问到 Claude Code session 卡死时说：*“直接开 5 个 Claude session。你根本不会注意到，因为你正忙着给其他 session 反馈。”* 他是在开玩笑。我想是吧。但这捕捉到了当下某种真实的东西。并行性令人兴奋，也有点野蛮。你能*启动*的事情数量大幅增加；你能有思考地完成的事情数量却丝毫没变，因为那一部分仍需要唯一无法并行的资源：你的大脑。

我给我认为正在发生的事起个名字：**人类奖励函数问题**。在机器学习中，奖励函数告诉 agent 什么是*好*。亲手写代码从来都不轻松，但它充满了小奖励：在脑中解决一个问题，理解一段棘手的逻辑，看着代码编译通过，感受掌控力。LLM 辅助编程自动化了大量产生那些多巴胺刺激的工作，并以审查和监督的认知负荷取而代之。令人满足的部分缩小了，令人筋疲力尽的部分膨胀了，而没有新的奖励来填补这个缺口。

如果你觉得工作同时更高产*又*更不令人满足，不是你坏掉了，是反馈回路坏掉了。我认为我们需要开始把这当作一个独立的工程问题，而不是个人失败。

坦白地说，它也很孤独。与 LLM 一起编程是一种强烈的独处活动。

你和机器来回交锋、打磨、prompt、审查。原本你会转向同事提问、借人“橡皮鸭调试”一个问题、分享某个东西终于豁然贯通的小胜利的自然时刻，悄悄被另一个 prompt 取代了。一个没有强大既有协作文化的团队，会因此更容易把人分开：恰恰在你最需要确认其他人类也觉得这很难的时候，沟通却被冷却了。

而且它以一种会加剧孤立的方式令人上瘾。有时你得到绝妙的结果，有时得到垃圾，而你始终不太知道会是哪一个。教科书般的[斯金纳箱](https://en.wikipedia.org/wiki/Operant_conditioning_chamber)。你确实可能很难退一步，记起自己有权利只是……写代码。但在 LLM 辅助和手工工作之间切换会令人突兀、不适：这是两种非常不同的思考模式；允许自己切换，需要一种成熟和自信。

## 断点

这个时刻让我想起响应式设计引发的恐惧与焦虑。当时我是一名设计师和前端开发者，和其他人一样关注着 [Ethan Marcotte](https://alistapart.com/article/responsive-web-design/) 以及 [Zeldman](https://zeldman.com/2024/12/05/of-books-and-conferences-past/) / [A Book Apart](https://ethanmarcotte.com/books/responsive-web-design/) 那个圈子。我记得当有人告诉我们：大家都已掌握的固定宽度布局基本结束了，那是多么令人不安。

给年轻开发者补充一点背景：大约 2009 年，网站从固定的、像素完美的杂志式布局，转向流动的响应式布局，确实曾形成一个文化时刻。设计师*讨厌*它。对于身份认同完全建立在精确布局和完美网格之上的人来说，失去控制是关乎存在的。你是说用户可能以*任意*宽度、在*任意*设备上看到我的设计？我精心打造的布局会……*流动？*

<img src="../../assets/the-human-in-the-loop-is-tired/responsive.gif" alt="响应式设计动画" width="200px" />

> 图片设计：[Jyotika Sofia Lindqvist](https://www.behance.net/jyotikasofia)

抵触非常强烈，而且可以理解。人们在一个正在被根本性颠覆的范式中建立了真才实学。那些在转型中成功的设计师重新定义了自己的技能。对比例的眼光仍然重要，对层次的理解仍然重要。手艺没有死去，而是演化了。不那么相关的，是对像素级控制的执念；更相关的，是理解系统、适应性，以及为不确定性而设计。

我不想过度渲染这个类比。响应式设计的演变以年为单位，而当前的转变以月为单位。响应式转型期间，代理公司失去了客户，设计师失去了工作，但它没有带来同样的存在性恐惧。赌注在实质上不同，节奏也确实令人筋疲力尽，其程度是响应式转型从未有过的。不过，我认为底层模式仍然成立：手艺在演化而非消亡，核心技能不是变得不重要，而是变得更重要。

与 LLM 一起做代码工作感觉像是类似的拐点。技能没有消失，而是在转移。你没有亲手写每一行代码，并不会让你变得不那么像工程师。但你仍然需要知道什么是好的——可以说比以往任何时候都更需要——因为你现在是海量输出的质量闸门。

## 留下来的东西

在一个人人都能产出看起来合理的 UI 和能编译的代码的时代，区分标志变成了：品味、细微之处、成熟的架构主张，以及源于真正专业知识而非模式匹配的逆向判断。

我注意到，在那些我们对代码、决策和权衡理解最深入的领域，我们最能成功地引导 LLM。当我们进入自己技能集合较浅的水域，输出会明显变得更具*印象派*色彩：离生产就绪更远，看起来更合理，实际正确性更低。模型不知道自己不知道什么，所以会自信地填补空白。听起来熟悉吗？这也是一种非常人类的失败模式。

但新技能也正在出现。我开始对复杂计划做我称为 pre-mortem 的事：要求一个全新的 LLM session 假设计划已经灾难性失败，并诊断原因。它能发现我在细节中钻了两天后漏掉的规格缺口。我们的一位工程师做了个工具，从他数千条过去的代码审查评论中提取规则，为 `AGENTS.md` 文件播种，本质上是把多年来隐性的工程判断编码成 LLM 能遵循的指令。这不是专业知识的死亡，而是专业知识被*蒸馏*。

目前找到了立足点的人，似乎有一些共同特质：他们拥有由实践得来的强烈观点；他们能区分仍然适用的原则与那些仅仅是带宽约束下形成的习惯；他们愿意演化工作流，同时不放弃自己的标准。

## 来自回路内部的视角

我不认为当前这一波 AI 代表软件工程作为职业的终结。但我确实认为，它代表着一次严重的收缩，以及对这项工作*是什么*的根本重塑。害怕被淘汰是合理的，害怕技能退化是合理的，害怕自己不够快就会被甩下——尽管往往被夸大——也并非完全没有依据。

但瓶颈从来不是代码。它一直都是人类注意力、工程判断，以及为一个系统持守连贯愿景的能力。我们只是没有注意到，因为写代码*感觉上*才是难的部分。如今它正在被自动化，那些人类能力才显露为真正稀缺的资源。而稀缺资源是有价值的。

所以如果你感到不堪重负、失去稳定感，同时更高产却更不快乐，请知道你并不孤单。正在构建你可能用来应对这个时刻的工具的团队，也同样有这种感受。我们和你一样，正在实时调试自己的奖励函数。

代码在改变。我们如何使用它在改变。它带来的感受也在……持续开发中。

但人类仍在回路中。我们只是累了。而这值得被讨论。

---

*我们正在构建让这一切少些混乱的工具：[Pydantic AI](https://pydantic.dev/docs/ai/overview/) 和 [Logfire](https://logfire.pydantic.dev)。我们也正在[招聘](https://pydantic.dev/jobs)。*

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "BlogPosting",
      "mainEntityOfPage": {
        "@type": "WebPage",
        "@id": "https://pydantic.dev/articles/human-in-the-loop-is-tired"
      },
      "headline": "The Human-in-the-Loop is Tired",
      "description": "On reward functions, dopamine, and what it actually feels like when the code starts writing itself",
      "keywords": "AI, LLMs, developer experience, human-in-the-loop, software engineering, AI-assisted programming",
      "image": {
        "@type": "ImageObject",
        "url": ""
      },
      "author": {
        "@type": "Person",
        "name": "Laura Summers",
        "sameAs": [
          "https://www.linkedin.com/in/summerscope/",
          "https://x.com/summerscope",
          "https://github.com/summerscope"
        ]
      },
      "about": [
        {
          "@type": "Thing",
          "name": "Large language models",
          "sameAs": "https://en.wikipedia.org/wiki/Large_language_model"
        },
        {
          "@type": "Thing",
          "name": "Human-in-the-loop",
          "sameAs": "https://en.wikipedia.org/wiki/Human-in-the-loop"
        },
        {
          "@type": "Thing",
          "name": "Software engineering",
          "sameAs": "https://en.wikipedia.org/wiki/Software_engineering"
        }
      ],
      "publisher": {
        "@type": "Organization",
        "name": "Pydantic",
        "url": "https://pydantic.dev/",
        "logo": {
          "@type": "ImageObject",
          "url": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAMgAAADICAMAAACahl6sAAAAw1BMVEX////nJWTDV3rRf5rnKGb8/PzoKmj5+PjoLmrz8fLXu8Ts6OnqQHfpN3HpOnP6+vrl3N/ThJ3j2t3eRXbaWILdU4DRsbvNoa/gUH/f1Nfp5ObHlKXYXIT18/TJj6LOd5PckKXhxczgztTUrLnZc5TUaYzUnK7qSn7cZYzWvsbiZYnsVIXZxszUZIjkfprMqbXLbozje6DuUHzVoLLZlavaRnfKgZjwc5vHepPfs77TYYbZqbngcZTXd5jbl6bevMXOT3jQsE7kAAAIlElEQVR4nO2da3vaOBCFccFgAobcuCeQknBraNM2abbbJtv9/79qoWkonJHl0cWW+6bebwU7nmM80sxo5JZKHo/H4/F4PB6Px+PxeBzxH6+ags4tGrCiAAAAAElFTkSuQmCC",
          "width": "200",
          "height": "200"
        }
      },
      "datePublished": "2026-02-18",
      "dateModified": "2026-02-18"
    }
  ]
}
</script>
