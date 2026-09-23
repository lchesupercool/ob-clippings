---
title: "关于 TypeSafe 编码智能体的一些思考"
author: "Diogo Almeida (@CompleteSkeptic)"
source: "https://x.com/completeskeptic/status/2101894250401271876?s=52"
canonical: "https://docs.google.com/document/d/1G61uUB0FifUnmmrPzFQojZ3KpczYKmXGpgEXDJ2l_Zg/edit?tab=t.0"
published: "2026-09-21T04:40:15.000Z"
saved: "2026-09-22 14:12:19 +0800"
tags: [clipping, x-twitter, coding-agents, typesafe, context-management]
---

# 关于 TypeSafe 编码智能体的一些思考

Diogo Almeida (@CompleteSkeptic)

![关于 TypeSafe 编码智能体的一些思考](../../assets/thoughts-on-a-typesafe-coding-agent/cover.png)

## 为什么还要再做一个智能体

为什么还要再做一个智能体？

- 我有几个关键假设：
  - 编码智能体出乎意料地简单
    - 尤其是智能体部分
      - 它们往往就是配有少量工具的 while 循环
    - 非智能体部分似乎几乎没有多少有意义的创新
  - 我们可以复用现有编码智能体中最好的部分
    - 模型当然也包括在内 -> 我们应该能通过 API 使用所有最好的模型
    - 甚至也许还能复用 UX/UI 组件
      - 有大量开源项目可供借鉴
    - （不确定认证有多复杂，例如 MCP 的认证）
  - 第一方智能体的成本优势可能正在缩小
    - （也许）智能体的使用方式似乎正更多地转向按使用量付费的 API 定价
- 有些事情只有原生实现才能做到
  - 也就是说，不能作为现有智能体的插件
  - 我在这里最喜欢问人们的一个问题是：“如果 LLM 没有 KV cache，你会怎样设计编码智能体？”
    - 这显然使我们能够设计一种以 TypeSafe 为中心的方式
    - 但也让我们得以看清当前设计的局限，以及为什么人们凭直觉认为应该奏效的事情实际上并不奏效

## KV cache 的暴政导致了哪些怪事？

### 第 1 件事：路由不起作用。旧计算：

```
- opus: 5 / 25
  - sonnet: 3 / 15
  - assumptions
    - compare:
      1. pure opus
      2. opus -> sonnet -> opus
    - X million context tokens
    - Y million output tokens
    - Z million additional generated tokens in that output (eg. commands, file reading)
  - path 1:
    - cost is
      - 25 * Y (generate w/ opus)
      - 5 * Z (read w/ opus)
      - total: 25 Y + 5 Z
  - path 2:
    - cost is
      - 3 * X (sonnet load context)
      - 15 * Y (generate w/ sonnet)
      - 3 * Z (read w/ sonnet)
      - 5 * (Y + Z) (opus load context)
      - total: 3 X + 20 Y + 8 Z
  - path 2 is more expensive
    - in a long session (large X)
    - if there is 67% more generated tokens than generated output tokens
    - some combination of the 2
  - let's have chatgpt vibe some proportion of X / Y / Z
      - X = 0.65
      - Y = 0.12
      - Z = 0.23
    - (25 * 0.12 + 5 * 0.23) / (3 * 0.65 + 20 * 0.12 + 8 * 0.23)
      - pure opus is 2/3 the cost
```

- TL;DR：先路由到较小的模型，再路由回来，成本可能反而*更高*，因为较大的模型需要重新处理上下文

### 第 2 件事：tool calling 是一种奇怪的权衡

- 工具需要预先指定（在 system message 中）
- 但它们并非总是相关
- 而且你需要随 tool calls 指定参数/所有内容
- 这会造成一些问题：
  - 占用巨量上下文
  - 没那么智能
    - 最佳猜测：模型不太擅长以下两者的某种组合：(1) 高基数；(2) 离策略 tool calling
- （有争议的是，这也许正是 skills 具有很大优势的原因，但它们与工具 / MCP 有很大不同）

### 第 3 件事：compaction 的存在

- 如果你假设智能体未来的所有轮次都想要一个共享状态，那么 compaction 完全合理
- 但话说回来，为什么要作此假设？
- compaction 试图解决压缩问题，而这个问题：
  - 非常困难
  - 并且很可能不如任何查询感知的压缩方法（例如，如果你知道要找什么，压缩起来就容易得多）

### 第 4 件事：subagents 一般般

- 对此我不是特别确定，但令我有点惊讶的是，模型没有进行更多自动并行化
- 我怀疑必须搞清楚状态的问题是原因之一
  - 要传入上下文的哪些部分
  - 要将 subagents 上下文的哪些部分合并回来

### 第 5 件事：重启的存在

- 如果智能体是有状态的，并且该状态最终会损坏，那么这也说得通
- 但按需加载所有相关的旧状态也同样合理

### 第 6 件事：不自带全套功能

- 对于智能体是否应该自带全套功能，存在很大争议
- 示例：<https://x.com/sudoingx/status/2061073944611029393>
- 易用性（以 openclaw 为极端）与高级用户需求（例如 claude code/codex）之间往往存在权衡

## TypeSafe + 编码智能体

我们可以做的事情非常多，所以我会把它们分成几类！

### 基础内容——可集成到任何智能体中的功能

### **权限 / 审批**

- 是否应该运行任意命令
  - 类似于 claude 的 auto-mode
  - 理想情况下，可以通过可编程查询来判断什么允许/不允许
- 可以有更深入的权限控制
  - 例如，在执行文件之前读取其内容（针对 python/bash 等）

### **MCP / Tool calling**

- 我们可以充当 tool calls 的路由器
- 先用高层次的文本描述我们要做什么，然后通过许多 typesafe 调用，找出最佳的 tool call 或排名前 X 的调用

### 高级内容——TypeSafe 原生功能

### **“元注意力”：智能上下文 / 相关性 / 过滤**

- 可以彻底移除上下文是静态的这一观念
- 对于每一个用户查询，我们都可以重新计算：
  - 先前上下文的质量如何/曾经如何
    - 也就是说，复用现有 KV cache 是可以的，而且很自然；我们应该根据复用当前 cache 与从头重新创建 cache 的效果优劣，做出充分知情且考虑成本的决策
  - 如何构建一个包含所有相关内容的新上下文
    - 最简单的形式是，你可以把它想象成每个上下文“块”上的一个 noul
      - Tool call 输入
      - Tool call 输出
      - 内部推理
      - 甚至可能包括与用户的来回交互
    - 未来版本甚至可以为每个“块”设置以下等级：
      - 不显示
      - 显示简短摘要
      - 显示较长摘要
      - 显示全部内容
      - （或类似的设置）

### **路由 + “Sub”-agents**

- 一等的动态上下文随后会让把较简单的任务路由到更便宜/更快的模型变得更容易
  - 当然，所有事情都应该同时考虑成本/智能水平！
- 我猜测，subagents 的一大成本在于确定上下文，而不是处理简单命令（就像用户会输入的命令）；如果这件事更容易、更便宜、更自动化，我们就能用 sub-agents 做更多事情
- 这也很可能让我们可以向用户提供更多调节选项，在“花更多钱以获得更好/更快的结果”和“保守地尽可能少花钱”之间选择

### **从第一性原理出发的 Skills/MCP/Tool calling 版本**

- 我认为应该存在一种介于两者之间的东西：
  - 关于有哪些可用内容的简短片段
    - 因为如果模型连某种操作可能存在都没有大致了解，它又怎么能建议该操作呢
    - 类似于 skills 的描述
      - 但它可能不必包含在 system message 中，而可以在适当的时候动态加载
  - 能够在需要时输出所有可用操作的完整 schema
    - 类似于工具搜索工具
  - 而且这不会污染上下文
    - 前所未有（：
- 如果这能奏效，我们就真能以极低乃至零成本内置各种全套功能！
  - 如果我们能以约等于零的成本加入各种功能，那么无论是联合营销，还是让各种东西开箱即用™，我们都能拥有一种不可思议的超级能力
  - （想象一下内置数百种工具 + 数千份文档）

### 更怪的内容——只是把一些东西扔进来继续琢磨

### **更好的全套功能**

- 人们构思了大量项目，理想情况下是为了帮助智能体，但实际上并没有
  - 示例：<https://github.com/rtk-ai/rtk>
    - 上下文压缩工具
  - 我的猜测：因为 LLM 并不能原生理解它们
- 我们可以围绕这些工具制作第一方 prompts 等内容，确保以正确方式向模型提示它们（可以把它想成某个工具的原生 subagent / skill）
- 上下文的整洁使我们能够加载自定义逻辑，同时不受其污染
- 一般来说，内置当下热门风味项目的全套功能，可能是为我们自己造势的好办法
  - 题外话：一个开源的造势载体或许会是很棒的交付物（某种能持续跟进当下热门事物的东西

### **条件式 system messages / AGENTS.md**

- 我们可以根据不同条件动态加载 AGENTS.md 的不同部分
  - 例如：
    - 是前端吗？加载样式指南
    - 位于这个子目录吗？加载易踩坑/注意事项文件（说实话，每个子目录都应该有一个注意事项文件）
- 这与 skills 有些相似，但根据我的使用经验，skills 往往是“现在做这件事”，而不是“把它放在记忆中的某处”
  - 此外，“把它放在记忆中的某处”这一约束可以不受 compaction/过滤移除的影响（也就是说，如果你加载一个 skill，随后最终触发 compaction，那么该 skill 很可能会被 compaction 掉/总结掉）
- 题外话：我发现自己真的很想把这个用于聊天机器人
  - 我想要摘要吗？这是我希望它采用的 org-mode 格式
  - 我是在要求按我的风格写作吗？这里有一些样本 + 不要做的事情
  - 是代码吗？不要添加一百万条断言，另外这里有一份样式指南

### **结构化 skills**

- 这个想法还不太成熟，但根据我们把 harness 做得多么可编程，skills 可以加入大量行为变化
- 类似于 claude skill hooks：<https://code.claude.com/docs/en/hooks#hooks-in-skills-and-agents>，但更强大
  - 我上次查看时，这些会将 hook 永久添加到 session 中

### **递归语言模型**

- 参见 <https://alexzhang13.github.io/blog/2025/rlm/>
- TL;DR：更多变量/显式状态
- 在某种可能的情形下，把状态视为显式变量会整洁得多

### **更精巧的总结**

- 一个想法是，如果我们能用热力图标出“这份 grep 输出的哪些部分相关”，就可以按我们希望的任意程度动态过滤它
- 看起来会很酷

### **极致的 subagents + 并行化**

- 不确定这对我们而言会如何运作，但（由于上面的部分）同时生成许多任务可能会变得非常容易；然后我们就必须解决各种烦人但有用的问题，比如同步原语，甚至是通信
  - 也就是说，它们可以共享状态，并针对写入冲突使用锁等机制

### **安全感知路由**

- 中国模型要*便宜得多*，而且很可能会攫取流经它们的所有数据（例如 DeepSeek V4 便宜得离谱）
- 如果：
  - Sub-agents/任务对自己触及不同类型文件的可能性有某种认知
  - 我们针对不同类型的文件制定不同策略
- 那么我们就可以把其中一些查询路由到*便宜得多*的模型
- 进一步扩展：除难度/成本外，可能还有许多其他路由理由（例如，不要使用 anthropic 模型进行 LLM 研究；不要使用 openai/anthropic 处理安全敏感事务）

## 附录

- 可以集成 / 改进的工具示例
  - <https://github.com/chopratejas/headroom>
    - 上下文压缩器
    - 备注
      - 可以用一个分类器来判断压缩结果是否包含必要/重要的信息
  - <https://github.com/rtk-ai/rtk>
    - 工具输出压缩器
  - <https://github.com/ast-grep/ast-grep>
    - 替代搜索方案
    - 备注
      - 对于新工具，我们可以为一次性查询加载手册，生成 N 个查询，然后使用我们的模型按相关性过滤这些查询
  - <https://ast-outline.github.io/>
    - 另一种结构化搜索
    - 备注
      - 我想知道我们能否将它用于分层函数调用 -> 使用模型确定要研究哪个子树
  - <https://github.com/microsoft/fastcontext>
    - 用于探索 repo 的 sub-agent
    - 备注
      - 有趣的说法：在我们对 GPT-5.4 轨迹的分析中，读取和搜索占全部工具使用轮次的 56.2%，占主智能体总 token 数的 46.5%
        - 如果这具有普遍性，就可能意味着只需专注于这一点便能获得大量效率提升
      - 同时做到以下两点可能很有价值：
        - 能够智能地路由到 sub-agents
        - 用结构化方法取代 sub-agents 的搜索！
  - <https://github.com/dmtrKovalenko/fff>
    - 抗拼写错误的路径与内容搜索、按频率排序的文件访问、后台 watcher，以及轻量级内存内容索引。在任何会进行多次搜索的长时间运行进程中，它都比 ripgrep 和 fzf 等 CLI 快得多。
- 我想知道是否有办法让类似 /goal 的行为变得更好（例如，通过显式去重）
  - 我不确定它是否真的会重复，但你可以设想，在生成一项事务之前，它可以有一个“subgoal”，并与此前所有 subgoals 进行去重
- SLOP：编码智能体子任务明细
  - ![文档图片 1](../../assets/thoughts-on-a-typesafe-coding-agent/doc-image-1.png)
  - 如果我们想为子任务制定特殊行为，这份列表似乎是一个很好的起点

## 附录 2：后台处理：一种潜在的智能体模式

- 我最近注意到一些相当流行的智能体工作流中似乎存在某种模式：
  - <https://x.com/xpasky/status/2073298979203211628>
    - 构建并行更新的 HTML 页面
    - ![图片](../../assets/thoughts-on-a-typesafe-coding-agent/doc-image-2.jpg)
  - <https://x.com/geoffreylitt/status/2072522251300409556>
    - “理解是新的瓶颈”
    - ![图片](../../assets/thoughts-on-a-typesafe-coding-agent/doc-image-3.jpg)
  - <https://x.com/samhogan/status/2071608749429829858>
    - 在后台创建 evals
    - ![文档图片 4](../../assets/thoughts-on-a-typesafe-coding-agent/doc-image-4.png)
  - <https://x.com/trq212/status/2090884854590382515>
    - ELI5 skill
    - ![文档图片 5](../../assets/thoughts-on-a-typesafe-coding-agent/doc-image-5.png)
- 这些事物有什么共同点？
  - 我最先注意到的是，它们全都在后台运行！
  - 它们通常是正常编码工作流的后台扩展
  - 它们不仅在后台运行，而且往往是当前代码库状态的只读函数
- 指出这一点后，可能已有一些模式适合纳入后台处理的范式
  - 我首先想到的是跨模型审查
    - 例如，让 codex 审查 claude 的工作

### TypeSafe 在哪里发挥作用？

- 我对以 TypeSafe 为中心的编码智能体有一个假设：我们必须非常明确地处理状态（上下文中的一切）
  - 具体来说，通过智能地处理状态，我们可以更高效地进行路由、sub-agents 等操作
- 这与只读后台任务之间的协同效应似乎相当高！
  - 为代码变更寻找相关信息，大概是一项不小的工作
  - 如果这项工作可以在所有后台任务之间共享，运行这些任务的成本就会低很多，并且运行比其他情况下更多的任务也可能变得经济可行
- 我认为，明确区分状态（读取与写入）最终应该会赋予我们超级能力 🙃

## 附录 3：近期帖子

<https://x.com/CompleteSkeptic/status/2100804802339082717>

![文档图片 6](../../assets/thoughts-on-a-typesafe-coding-agent/doc-image-6.png)

---

来源：[X 上的 Diogo Almeida](https://x.com/completeskeptic/status/2101894250401271876?s=52)  
规范来源：[公开 Google 文档](https://docs.google.com/document/d/1G61uUB0FifUnmmrPzFQojZ3KpczYKmXGpgEXDJ2l_Zg/edit?tab=t.0)  
保存时间：2026-09-22 14:12:19 +0800