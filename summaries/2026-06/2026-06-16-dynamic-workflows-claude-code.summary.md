---
source: https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code
saved: 2026-06-22 14:49:55 +0800
summary_of: LLM Wiki/raw/clippings/2026-06/2026-06-16-dynamic-workflows-claude-code.md
title: "A harness for every task: dynamic workflows in Claude Code"
---

# `A harness for every task: dynamic workflows in Claude Code` — 摘要

## 一句话结论
Claude Code 现在能为每个任务**临时写一个自己的 harness**(一个编排子 agent 的 JS 脚本)，把"长时、大规模并行、强结构、对抗性"的任务从单一上下文窗口里解放出来，从结构上对抗 agentic laziness、self-preferential bias、goal drift 三类失败模式。

## 文章主旨
dynamic workflows 是 Claude Code 上周发布的新能力。默认 harness 为编码而建，很多任务也能复用，但研究、安全分析、agent teams、code review 这几类以前得 Anthropic 手搓专用 harness。现在 Opus 4.8 够聪明，能即时为你的用例量身写一个：执行一个 JS 文件，用特殊函数 spawn 并协调子 agent，每个子 agent 有独立上下文和聚焦目标，可单独指定 model 和 worktree 隔离。

## 关键脉络
1. dynamic workflows = Claude 即时写 harness；触发词 `ultracode`，或直接让它"make a workflow"。
2. 实现：跑一个 JS 文件，内置 spawn/协调子 agent 的函数 + 标准 JS(JSON/Math/Array)处理数据。
3. 可为每个 agent 选 model、选是否在独立 worktree 跑(智能档位 + 隔离级别)。
4. 中断可恢复：重开 session 接着跑。
5. 为什么需要：默认 harness 要在同一上下文里既 plan 又 execute，长复杂任务会退化。
6. 三类失败模式：**agentic laziness**(没做完就宣布完成，50 项做 35 项)、**self-preferential bias**(偏袒自己的产出，尤其自评时)、**goal drift**(多轮 + compaction 后丢失原始目标和"别做 X"的约束)。
7. 动态 vs 静态：静态 workflow(Agent SDK / `claude -p`)要覆盖所有边界所以更通用；动态的为你单一用例定制。
8. 六个可组合模式：classify-and-act、fan-out-and-synthesize、adversarial verification、generate-and-filter、tournament、loop-until-done。
9. 用例：迁移重构(Bun 从 Zig 重写到 Rust)、deep research、deep verification、sorting、memory/rule adherence、root-cause investigation、triage at scale、exploration and taste、evals、model routing。
10. 何时不用：贵(token 多)、不是每个任务都要；常规编码先问"真需要更多算力吗"，大多不需要 5 个 reviewer。
11. tips：详细 prompt、可用"quick workflow"、配 `/goal` 和 `/loop`、设 token budget("use 10k tokens")、按 `s` 存到 `~/.claude/workflows` 或打包进 skill(当模板而非逐字脚本)。

## 值得注意的细节
- "comparative judgment is more reliable than absolute scoring"——排序用两两比较(tournament)比让模型打绝对分可靠；1000+ 行一次性排会退化、也塞不进上下文。
- **quarantine 模式**：读不可信公开内容的 agent 禁止高权限操作，高权限动作交给另一批 agent——针对 prompt injection 的结构性防御。
- Bun 从 Zig 重写到 Rust 真的是用 workflows 做的：每个 callsite/失败测试/模块 spin off 一个 worktree 子 agent 修 + 另一个 agent 对抗 review + 合并。
- synthesize 步是 **barrier**：等所有 fan-out agent 完成再合并结构化输出。
- 治 rule adherence：一条规则一个 verifier agent，再加一个"怀疑论者"persona 审规则本身，避免误报。

## 我的理解
这篇的价值不在教语法(几乎没给代码)，而在给你一套**何时该用多 agent 编排、用哪种模式**的心智模型。核心洞察：单上下文窗口有物理极限(laziness/bias/drift)，这些不是更好的 prompt 能根治的，是结构性问题，得靠"拆到多个独立上下文 + 对抗性验证"来治。

但要清醒：文章自己反复强调"贵、不是每个任务都要、best practices 还在发展"。它本质是 Anthropic 在推一个更耗 token 的新能力，叙事里带着"以前我们手搓的 harness 现在让模型自己写"的自我合理化。对普通用户真正能落地的，可能就两三个高价值场景(deep research、对抗验证、大规模 triage/排序)，其余多是启发性的可能性清单。

一条暗线最值得记：它把上一篇 mvanhorn 讲的"human signal""plan-first"这些**人肉技巧，变成了可编程的结构**——对抗验证不再靠你手动开两个 session 对喷，而是写进 workflow 自动跑。

## 对我的参考价值
- **你本 session 工具列表里就有 Workflow + `/deep-research` + `/loop` + `ultracode`**——这篇是它们的官方操作手册，直接对应你手上的能力，不是纸上谈兵。
- 你的 skill 体系可升级：`tr-batch` 批量翻译本质就是 fan-out-and-synthesize；`deepread-qa` 逐条回答可加 adversarial verification(一个 agent 答、一个 agent 挑错)。可挑一个用 workflow 重写试试。
- **立刻能做的小实验**：用"quick workflow"对你某篇技术文档跑 deep verification——一个 agent 列出所有技术声明，每条 spin off 子 agent 去 PG 源码核对。正好配你的 `doc-review` / 面试准备，确保你写的 PG 机制描述无误。
- "self-preferential bias + 对抗验证"直接呼应上一篇"human signal"：把"自己当信号"的判断职责，部分下放给 workflow 里的 refuter agent。
- **sorting 用例可迁移**：给一堆 clippings 按"对面试的价值"排序时，别让我一次性打分(会退化)，用 pairwise tournament。
- **反面教材/克制**：你大多数工作(读文章、写卡片、翻译单篇)是低价值高频任务，别套 workflow，token 不划算——文章自己也说"most coding tasks don't need 5 reviewers"。
- **连接点**：和你已读的 mvanhorn、Codex-maxxing、meta-meta-prompting 同簇，但这篇是**官方的、结构化的**那一极，适合当"Agentic Engineering 工作流"概念页的骨架。
