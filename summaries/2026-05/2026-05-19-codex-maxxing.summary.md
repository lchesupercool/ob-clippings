---
title: Codex-maxxing 摘要
source: https://jxnl.co/writing/2026/05/10/codex-maxxing/
saved: 2026-05-19 14:57:03 +0800
summary_of: ../2026-05/2026-05-19-codex-maxxing.md
---

# Codex-maxxing 摘要

作者的核心观点：Codex 的价值不只是“写代码更快”，而是开始变成一个长期工作可以驻留、持续推进、被审阅和被恢复的工作环境。真正的变化来自一套 operating loop：持久线程、语音输入、steering、共享记忆、电脑/浏览器操作、远程控制、Heartbeats、Goals，以及 side panel。

## 关键要点

1. 持久线程让工作流有连续性

作者为重要 workstream 保留置顶线程，例如 Chief of Staff、Agents SDK、OpenAI CLI、Codex for OSS、Twitter 监控等。这些线程通过 compaction 长期延续，积累历史、偏好和决策。缺点是成本可能更高，但对重要工作流来说，连续性值得。

2. 语音输入让 agent 获得更真实的上下文

语音的优势不只是快，而是能输入“未经整理的思考”。很多模糊、不完整但有价值的上下文，打字时会被省略，说出来则很自然。作者还会用通话录音或 Granola transcript 作为写作和规划材料。

3. Steering 把“一问一答”变成持续引导

Steering 允许在 agent 执行工具调用后继续注入方向。比如审阅网页时，可以连续补充“这里小一点”“文案不对”“完成后开 PR”“preview 完成后发 Slack”。工作单元不再是一个 prompt 对应一个答案，而是一个可持续塑形的小循环。

4. 共享记忆必须落到可检查的 artifact

作者强调，长期线程不能只依赖聊天历史。真正有用的记忆应该被序列化到文件里，例如 Obsidian vault：记录 people、projects、open loops、daily notes、项目状态和决策。vault 放在 GitHub repo 中，让 diff 成为记忆审阅界面。这样即使线程死亡、压缩失败或继续使用太贵，知识仍然存在。

5. Codex 的 Memory 是 recall layer，不是完整记忆系统

Codex 的 Personalization Memories 适合稳定偏好、重复 workflow、项目约定和已知坑，但不能替代显式 vault 或 check-in 的 instructions。Chronicle 的方向很有意思：用最近屏幕上下文辅助构建 memory，但仍有权限、速率限制、prompt injection、本地未加密记忆文件等风险。

6. Computer / Browser Use 让 agent 能真正触达工作对象

作者区分三类工具：

- `$browser`：用于本地 web surface 检查和标注
- `@chrome`：用于已登录浏览器状态和多 tab
- `@computer`：用于只能通过 GUI 完成的工作

再配合 Slack、Gmail、Calendar connectors，agent 能接触很多“代码之前”的工作现场。Skills 则把重复流程打包复用。

7. 远程控制让长循环可移动

Codex 可以在工作机器上继续运行，用户从手机回来查看结果、回答问题、批准下一步或改变方向。重点是任务不再因为人离开桌面而暂停。

8. Heartbeats 让线程具备周期性自动化

Heartbeats 是 thread-local automation。线程可以定期检查 Slack、Gmail、PR comments、Google Docs comments 等，并在条件满足后继续推进。例子包括：每 30 分钟检查未回复消息并草拟回复；每 15 分钟监控 Slack 反馈并重新渲染动画；等待 Amazon 客服上线并争取退款。

9. Goals 需要真实终点和验证机制

作者认为好的 Goals 不是“实现这个 Markdown 计划”，而是有明确成功标准的长期任务。例如把 Python Rich library 迁移到 Rust，并要求通过原项目完整 unit tests。没有 verification 的雄心只是愿望。

10. Side panel 是 Codex 从 chat app 变成 work surface 的关键

Side panel 不只是 preview。它让用户直接检查、标注和操作 artifacts：Markdown、spreadsheet、CSV、PDF、slides、HTML、Storybook、Remotion Studio、Slidev、Streamlit 等。作者尤其看好单文件 `index.html`，因为它比 Markdown 更像一个小应用，也比 Vite app 更持久、轻量。

## 对我的启发

这篇文章最值得借鉴的是：把 AI agent 从“临时帮手”改造成“长期工作进程”。关键不是多写 prompt，而是给 agent 建立：

- 持久线程：保留上下文
- 外部记忆：Obsidian/Git 文件化，可 diff
- 工具权限：能触达真实工作表面
- 周期任务：自动等待反馈和继续推进
- 验证机制：用 tests / PR / artifact review 定义完成
- 可视界面：side panel 让人和 agent 对同一对象协作

一句话总结：Codex-maxxing 不是把 Codex 当代码生成器用到极致，而是把它当成一个能记忆、行动、等待、恢复、审阅 artifact 的长期工作操作系统。