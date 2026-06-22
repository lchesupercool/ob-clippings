# A harness for every task: dynamic workflows in Claude Code

> 来源: [@trq212 on X](https://x.com/trq212/status/2061907337154367865)
> 作者: Thariq Shihipar & Sid Bidasaria (Anthropic, Claude Code team)
> 日期: 2026-06-03

---

## 概述

Claude Code 发布了 **dynamic workflows**：Claude 可以动态编写自己的 harness (JavaScript 文件)，为特定任务量身定制执行框架。Workflows 可以分享和复用。

注意：best practices 仍在发展中，dynamic workflows 通常消耗更多 tokens。

## 示例 Prompts

- "This test fails maybe 1 in 50 runs. Set up a workflow to reproduce it, form theories and adversarially test them in worktrees /goal don't stop until one theory works."
- "Using a workflow, go through my last 50 sessions and mine them for corrections I keep making and turn the recurring ones into CLAUDE.md rules"
- "Use a workflow to dig through #incidents in Slack for the past six months and find recurring root causes where nobody has filed a ticket."
- "Take my business plan and run a workflow where different agents tear it apart from an investor's, a customer's, and a competitor's perspective."
- "Here's a folder of 80 resumes, use a workflow to rank them for the backend role and double-check the top ten."
- "I need a name for this CLI tool. Use a workflow to brainstorm a bunch of options and run a tournament to pick the top 3."
- "Go through my blog post draft and using a workflow verify every technical claim against the codebase, I don't want to ship anything wrong."

## 为什么需要 Dynamic Workflows

默认 Claude Code harness 在单个 context window 中同时规划和执行。长任务/大规模并行/对抗性任务中会出现三种失败模式：

1. **Agentic laziness** — Claude 在部分完成后就宣布任务结束（如 security review 只处理了 50 项中的 20 项）
2. **Self-preferential bias** — Claude 倾向偏好自己的结果，特别是被要求验证或评判自己的输出时
3. **Context drift** — 多轮对话后逐渐偏离原始目标，compaction 导致边缘约束丢失

Workflows 通过编排多个独立的 Claude（各自拥有独立的 context window 和专注目标）来解决这些问题。

## Dynamic vs Static Workflows

Static workflows (Agent SDK / `claude -p`) 需要考虑所有边缘情况，因此比较通用。Dynamic workflows 中 Claude 智能地为你的具体用例生成定制 harness。

触发方式：直接让 Claude 创建 workflow，或使用关键词 **"ultracode"**。

## 常见 Workflow 模式

| 模式 | 描述 |
|---|---|
| **Classify-and-act** | 分类 agent 判断任务类型 → 路由到不同 agent |
| **Fan-out-and-synthesize** | 拆分任务 → 并行 agent → 屏障合成结果 |
| **Adversarial verification** | 每个 agent 输出由另一个 agent 按 rubric 对抗性验证 |
| **Generate-and-filter** | 生成大量想法 → rubric 过滤/验证 → 去重 → 返回高质量结果 |
| **Tournament** | N 个 agent 竞争，两两对比判定，选出胜者 |
| **Loop-until-condition** | 循环生成 agent 直到停止条件满足 |

## 使用场景

### Migrations & Refactors
Zig → Rust 重写案例：分解为一系列步骤（callsites、failing tests、modules），每个 fix 在独立 worktree 中由 subagent 完成 → 对抗性 review → merge。

### Deep Research
/deep-research skill：fan-out web searches → 抓取源 → 对抗性验证声明 → 合成带引用的报告。不限于 web search，也可用于 Slack 上下文、代码库深度探索等。

### Deep Verification
验证报告中的每个事实声明：一个 agent 识别所有声明 → subagent 逐一检查 → verification agent 检查 source agent 质量。

### Tournament Ranking
排序大量条目（如按严重程度排序 support tickets）：两两比较 agent pipeline（比较性判断比绝对评分更可靠），或 bucket-rank 并行后合并。

### Memory & Rule Adherence
Claude 容易遗漏的规则 → 创建 workflow，每个规则一个 verifier agent。反向：挖掘近期 session 中的修正 → 聚类 → 对抗性验证 → 提炼回 CLAUDE.md。

### Root-Cause Investigation
从不相交的证据（logs/files/data）生成多个独立假设 → panel of verifiers and refuters。适用于代码、销售分析、数据工程、post-mortem。

### Triaging at Scale
分类 → 去重（对比已有记录） → 执行（fix 或 escalate）。关键模式：**quarantine** — 读取不可信内容的 agent 被禁止高权限操作。

### Exploration & Taste
探索多种方案 → review agent 按 rubric 评估 → tournament 排序。也可用于评估和迭代 skill。

### Model & Intelligence Routing
Classifier agent 根据任务调研选择最佳模型（如 auth 模块的文件数决定用 Sonnet 还是 Opus）。

## 何时不用

- 常规 coding 任务通常不需要（不需要 5 reviewer panel）
- 会增加 token 消耗
- 问自己：真的需要更多 compute 吗？

## 使用技巧

- 详细 prompt + 具体技术描述效果最好
- 不只是大任务，也可以用 "quick workflow" 做快速对抗性审查
- 结合 `/goal` 和 `/loop`：可重复的 workflow 定时运行
- 可设置 token budget：如 "use 10k tokens"
- 保存：按 `s` 在 workflow 菜单保存，存入 `~/.claude/workflows` 或通过 skill 分发
