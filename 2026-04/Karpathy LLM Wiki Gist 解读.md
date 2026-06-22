---
date: 2026-04-05
source: 4ccdfd13-f6bb-4774-9c47-78165be7e4cd.jsonl
---

> Karpathy有没有给出对应的 git

---

找到了。Karpathy 发了一个 **GitHub Gist**，不是完整 repo：

有。Karpathy 发了一个 **GitHub Gist**：

**https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f**

2788 stars，532 forks。不是一个完整 repo，而是一个名为 `llm-wiki.md` 的 idea 文件，设计成直接复制给你的 LLM Agent（Claude Code / Codex 等），让它帮你搭建知识库。

# 核心架构（三层）

| 层 | 说明 |
|---|---|
| **Raw Sources** | 不可变的原始文档（文章、论文、图片），LLM 只读不改 |
| **Wiki** | LLM 拥有和维护的 markdown 目录，包含摘要、实体页、概念页、对比、综述 |
| **Schema** | CLAUDE.md / AGENTS.md，告诉 LLM wiki 的结构约定和工作流 |

# 四个操作

1. **Ingest** — 放入新源文件，LLM 读取 → 写摘要 → 更新索引 → 更新相关实体/概念页（一个源可能触及 10-15 个页面）
2. **Query** — 对 wiki 提问，LLM 搜索相关页面合成答案。好的答案可以**回填到 wiki** 形成复利
3. **Lint** — 健康检查：矛盾检测、过时声明、孤立页面、缺失交叉引用、数据缺口
4. **索引 + 日志** — `index.md`（内容目录）+ `log.md`（时间线记录）

# 社区衍生工具

评论区出了不少实现：
- [Astro-Han/karpathy-llm-wiki](https://github.com/Astro-Han/karpathy-llm-wiki) — 一键安装的 Claude Code skill
- [pedronauck/skills/karpathy-kb](https://github.com/pedronauck/skills/tree/main/skills/karpathy-kb) — skill 实现
- [sage-wiki](https://github.com/xoai) — Go 单二进制实现，含 MCP server
- [fakechris/obsidian_vault_pipeline](https://github.com/fakechris/obsidian_vault_pipeline) — AutoPilot 自动维护

这个模式跟你的 Obsidian vault 天然契合。要保存到 vault 或者试着搭一个吗？
