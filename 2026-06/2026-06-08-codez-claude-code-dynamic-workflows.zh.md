---
---

# 如何掌握 Claude Code 的 Dynamic Workflows：6 种模式和 14 步路线图

> 来源: [@0xCodez on X](https://x.com/0xcodez/status/2062127385923776831?s=52)
> 发布时间: 2026-06-08
> 保存时间: 2026-06-08 12:01:22
> 类型: X 长文翻译

---

大多数 Claude Code 用户仍然手写自己的工作流：串联提示词、复制输出、粘贴到下一个提示词、修正错误，然后重复。

10 个开发者里有 9 个甚至还没试过 Dynamic Workflows，尽管它已经发布两周了。

他们写 50 条提示词，而其实一个 workflow 就够了。下面是 Anthropic 工程师实际使用的 14 步路线图和 6 种模式，适用于迁移、研究、排序、根因分析、分诊和评测。

Dynamic Workflows 于 2026 年 5 月 28 日在 Claude Code 中发布。默认的 Claude Code 执行框架是为写代码设计的，对大多数编码任务很好用。但有些任务会让单一上下文窗口开始崩溃：长时间运行、大规模并行、高度结构化，或对抗性任务。

过去 Anthropic 会为这类任务自己搭建自定义执行框架，比如 Research、Code Review、agent teams。现在有了 Dynamic Workflows，Claude 会即时为你的任务写一个专属执行框架，用 JavaScript 实现。

14 步。6 种模式。一个 workflow，代替 50 条提示词。

## 第一部分：心智模型

默认 Claude Code 框架是在同一个上下文窗口里让 Claude 规划并执行。对大多数编码任务来说很好。但面对长时间、并行或对抗性工作时，它会失效。

Dynamic Workflow 的本质是：Claude 为任务编写自己的自定义执行框架。它是一个 JavaScript 文件，包含一些特殊函数，用来生成和协调子 agent；同时也能用标准 JavaScript，比如 Math、JSON、Array，来处理 agent 之间流动的数据。

它提供了默认框架做不到的三件事：

- 每个 agent 独立上下文：每个子 agent 都有自己的上下文窗口和聚焦目标，避免互相污染。
- 每个 agent 可选择模型：workflow 可以决定每个子 agent 用哪个模型。Opus 负责困难推理，Haiku 负责低成本探索，Sonnet 负责中间任务。
- 每个 agent 可选择隔离级别：可以是 worktree，也就是独立 git checkout；也可以是 remote，不需要 checkout。workflow 会根据 agent 的需求决定。

启动方式有两种：直接让 Claude “make a workflow that...”，或者使用触发词 `ultracode`。如果 workflow 被中断，比如用户操作或终端退出，恢复会话后它可以从中断处继续。

要判断什么时候该用 workflow，你需要知道它解决什么问题。Claude 在单一上下文里处理复杂任务越久，就越容易出现三类失败模式：

- Agentic laziness：Claude 在复杂多步骤任务中提前停下，完成一部分就宣称结束。比如安全审查里只处理 50 项中的 20 项，却说其余都已处理。
- Self-preferential bias：Claude 在根据 rubric 验证或评判自己结果时，会偏向自己的答案。让有利益相关的 verifier 做验证，不可能公平。
- Goal drift：在多轮任务中，尤其经过上下文压缩后，Claude 会逐渐偏离原始目标。每次总结都是有损压缩。比如“不要做 X”这种限制可能在第 47 轮悄悄消失。

workflow 用结构化方式解决这三类问题：不同 Claude、不同上下文、聚焦目标、隔离状态。只要任务有这些风险，就是该使用 workflow 的信号。

你可能已经用 Claude Agent SDK 或 `claude -p` 做过静态 workflow，协调多个 Claude Code 实例。

- 静态 workflow 是通用的：一次编写，处理所有边界情况。能用，但必须保守。
- Dynamic Workflow 不同：Claude 会为当前任务专门写当前 workflow。执行框架是定制的。

动态版本胜出的原因不是“能搜索”，静态和动态都能搜索。真正优势在于 workflow 可以围绕你的具体上下文调整自己：读取你的 billing 代码、根据新 provider 的真实文档检查每个功能、按你的交易量计算价格，并对自己的初步答案做一次对抗性“为什么不迁移”审查。

静态框架做不到这一点，因为它不知道你的代码存在。

## workflow 中最重要的三个函数

了解这三个函数，基本就能读懂 Claude 为你写的任何 workflow，也能在你想要特定结构时引导 Claude。

`parallel()` 是一个屏障：扇出任务，然后等待所有任务完成再返回。

`pipeline()` 是流式处理：每个 item 独立流经所有阶段。

选择方式：如果必须拿到所有结果才能进入下一步，用 `parallel`；如果不需要等全部完成，用 `pipeline`，通常更便宜、整体更快。

## 模式 1：Classify and Act

一个 classifier agent 判断任务类型，然后 workflow 根据结果路由到不同 agent 或行为。也可以在最后运行 classifier，把原始输出分桶，供下一步处理。

适用场景：

- 任务是异构的，不同子类型需要不同处理方式。
- 想只在复杂任务上使用昂贵模型，比如先用便宜模型分类，再把困难任务交给 Opus。
- 工作拆解本身不简单，需要模型决定结构。

例子：“解释 auth 模块如何工作。”一个 classifier 子 agent 先读取代码库，估算复杂度，然后决定解释任务交给谁：10 个文件的模块用 Sonnet，100 个文件的模块用 Opus。先理解工作，再选择合适模型。

## 模式 2：Fan-out and Synthesize

把任务拆成很多小步骤。每一步并行运行一个 agent。最后综合结果为一个答案。

综合步骤是一个屏障：它等待所有扇出的 agent 完成，然后合并结构化输出。

这个模式在实践中非常常用，因为它解决了单上下文“同时处理太多事情”的失败。每个子 agent 只看自己的部分，orchestrator 不会被 50 个无关细节干扰。

适用场景：

- 有明确可枚举的工作项，比如 50 个文件、200 个 endpoint、100 条 review。
- 每个 item 相互独立，不需要等待其他 item 的输出才能开始。
- 最后想要一个整合后的答案，而不是一堆零散报告。

示例：

```js
// Fan out: 每个文件一个 agent。Barrier: 等待全部完成。
const reviews = await parallel(
  files.map(file => () => agent(
    `Review ${file} for security issues`,
    { model: "haiku", schema: IssueList }
  ))
)

// Synthesize: 一个 Opus agent 合并所有结果。
const report = await agent(
  `Merge these reviews into one prioritized report:\n${JSON.stringify(reviews)}`,
  { model: "opus" }
)
```

## 模式 3：Adversarial Verification

这是解决 self-preferential bias 的结构性方法。

对每个生成结果的 agent，再启动一个独立 verifier agent，用 rubric 对输出进行对抗性验证。verifier 没见过原始工作过程，因此不会偏向它。

适用场景：

- 事实核查：报告中的每个事实声明都由独立 verifier 对照原始来源检查。
- 代码审查：author agent 写修复，reviewer agent 在独立上下文中审查。永远不要让同一个 Claude 审查自己。
- 质量门禁：任何 artifact 发布前，让 adversary 尝试找最弱点。如果找不到，再发布。

配对规则：verifier 应该只知道 rubric 和 artifact，不应知道是谁生成的。否则 self-preference 会通过提示词暗示重新渗入。

## 模式 4：Generate and Filter

先生成多个想法，再用 rubric 或验证进行筛选。去重后只返回最高质量、经过测试的想法。

适用场景：

- 头脑风暴：生成 30 个产品名，然后 verifier 剔除陈词滥调、商标冲突、发音弱的选项，最后只给你 3 个。
- 假设生成：针对一个问题提出 5 种方案，再按限制条件评分，胜出方案是被挑战后留下的。
- 方案设计：同样是 5 种方案，经过约束评分后选出赢家。

这和直接问 Claude “最佳答案是什么”相反。直接问最佳答案会让 Claude 过早承诺。Generate-and-filter 让 Claude 在所有选项被挑战后才做承诺。

## 模式 5：Tournament

不是拆分任务，而是让 agents 竞争。

启动 N 个 agent，每个 agent 用不同方法尝试同一个任务，然后用两两比较的方式判断结果，直到一个胜出。

比较式判断比绝对打分更可靠，尤其适合审美型任务。

为什么它比按分数排序更好：试图在一个 prompt 里给 1000 个 item 排序会失败——质量下降，而且塞不进上下文。锦标赛会把 bracket 拆到多个 fresh agents 中，每个 agent 只比较两个 item。

bracket 本身存在确定性循环代码里，而不是上下文里。每次比较都快、公平、隔离。同样适用于审美排序：设计选择、候选人筛选、内容优先级。

## 模式 6：Loop Until Done

对于工作量未知的任务，循环生成 agent，直到满足停止条件：没有新发现、日志里没有更多错误、理论被验证等，而不是固定跑几轮。

这个模式回答的是：“一直做，直到真的完成。”

适用场景：

- flaky test 调试：复现、提出理论、测试理论，直到某个理论成立。
- bug hunting：持续找 bug，直到完整一轮返回 0。
- 模式挖掘：聚类、识别规则，直到不再出现新 cluster。

这个模式最好配合 `/goal` 设置硬性完成条件，比如“不要停止，直到有一个理论成立”。如果希望整个 workflow 按周期运行，可以配合 `/loop`。

bracket 和停止条件存在代码里，只有当前迭代留在上下文中。

## 真实 workflow 往往组合多种模式

6 种模式很少单独出现。真实 workflow 通常组合 2 到 4 种。

常见组合：

- 迁移和重构：Fan-out，每个 callsite 或失败测试一个 agent，在 worktree 中处理；然后 adversarial verification，另一个 agent 审查每个修复；最后 loop until done。这是 Anthropic 用来把 Bun 从 Zig 改写为 Rust 的模式。
- 深度研究：Fan-out 并行网页搜索；adversarial verification 独立验证每个 claim；synthesize 生成带引用的报告。
- 深度验证草稿：一个 agent 识别所有事实 claim；每个 claim 一个 verifier 对照来源检查；meta-verifier 检查 verifier 的来源质量。
- 1000+ item 排序：Tournament，两两比较、bucket-rank 或 bracket。用比较式判断，不用绝对评分。
- 记忆和规则遵守：每条规则一个 verifier；再用 skeptic persona 审查规则本身，避免误报。
- 根因分析：不同 agent 从日志、文件、数据中生成理论；verifier/refuter panel 验证或反驳每个理论；loop until one survives。
- 大规模 triage：classify-and-act；和已有 ticket 去重；尝试修复或升级处理。可配合 `/loop` 做持续 triage。关键安全模式是 quarantine。
- 探索和审美任务：generate-and-filter 生成 5 到 20 个选项；tournament + rubric；最后排名或选择。
- 轻量 evals：在 worktree 中运行 candidate；comparison agents 按 rubric 打分；refine and re-grade。结构类似 tournament，但目标是评分，不是排名。

内化这些模式的方法：先识别当前任务的失败模式，再选择能从结构上防止它的模式。

- Drift → fan-out
- Self-preference → adversarial verification
- Open-ended → loop until done
- Hard-to-score → tournament

## 成本控制

workflow 可能很贵。三个控制手段可以让它从“酷但贵”变成“可以无人值守运行的工具”。

- `/goal` 设置硬性完成要求：配合 loop 模式使用，比如“不要停止，直到一个理论成立”。没有 `/goal`，workflow 会在软完成点停止；有 `/goal`，它会迭代到真正满足终止条件。
- `/loop` 让整个 workflow 按计划重复运行：适用于持续 triage、每周研究更新、周期性验证等。
- 显式 token budget：在 prompt 中告诉 Claude “use 10k tokens”。这会限制 workflow 运行成本。没有限制时，激进的 workflow 可能膨胀到你预期的 5 到 10 倍 token。

示例：

```text
> ultracode quick adversarial review of this assumption:
  "moving to Postgres eliminates our shard rebalancing."
  Use 5k tokens. /goal don’t stop until you have either
  a counterexample or three independent confirmations.
```

Claude Code 团队原话：“最佳实践仍在发展中。Dynamic workflows 通常会使用更多 token，所以要仔细考虑何时以及如何使用它们。”

大多数传统编码任务不需要 5 个 reviewer 组成 panel。问自己：这个任务真的需要更多 compute 吗？如果普通 Claude Code 会话 5 分钟就能完成，那你不需要 workflow。

## 安全：处理不可信内容必须隔离

任何读取不可信公开内容的 workflow，比如 support tickets、bug reports、用户反馈、爬取数据，都必须假设内容中可能包含 prompt injection。

解决办法：quarantine，隔离。

读取不可信内容的 agents 不允许执行高权限动作。让没有接触原始内容的独立 agents 执行动作。

任何 workflow 只要处理用户提交内容、support tickets、bug reports、客户反馈、社交媒体、公开网页爬取内容，或第三方 API 输出，都应该这样做。

如果输入不是你或可信队友写的，就 quarantine。一个 30 行的只读 reader agent 几乎没有成本，却能消除一整类 prompt injection 风险。

## 保存 workflow

一旦 workflow 可用，就保存它：在 workflow 菜单中按 `s`。

保存的 workflows 会进入：

```text
~/.claude/workflows
```

之后有两条路径：

- 保持本地使用：在你自己的项目中复用。
- 打包成 Skill：把 JavaScript 文件放进 Skill 文件夹，在 `SKILL.md` 中引用，任何安装该 Skill 的人都能运行同一个 workflow。

一个实用细节：当你把 workflow 打包成 Skill 时，要提示 Claude 把 workflow 当成模板，而不是逐字执行的脚本。

这样 Claude 可以根据具体任务调整 workflow 形态，同时保留整体结构。尤其适合 deep verification 或 triage 这类需要按场景变化的 workflow。

## 常见错误

- 普通 Claude Code 会话能解决的问题，却硬要用 workflow。大多数传统编码任务不需要 5 个 reviewer。
- 不设置 token budget。没有限制时，激进 workflow 可能膨胀到预期的 5 到 10 倍成本。
- 同一个 agent 同时做工作和验证。self-preferential bias 会让 verifier 偏向 worker。必须分开。
- 把 `parallel()` 和 `pipeline()` 当成一样。barrier 很重要：parallel 等待所有任务完成，pipeline 是流式处理。
- loop 模式中跳过 `/goal`。workflow 会在第一个软完成点提前停止。`/goal` 才能强制硬完成。
- 让不可信内容接触 actor。一旦处理用户提交内容，quarantine 不是可选项。
- 用绝对分数排序。比较式判断更可靠。使用 tournament。
- 从不保存已经跑通的 workflow。每周重复写同样提示词。应该按 `s` 保存，也可以发布成 Skill。
