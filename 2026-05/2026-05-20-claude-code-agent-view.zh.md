---
title: "使用 agent view 管理多个代理"
source: "https://code.claude.com/docs/en/agent-view"
source_language: "zh-CN"
saved: "2026-05-20 18:37:22 +0800"
tags: [claude-code, agent-view, clipping]
---

> ## Documentation Index
> 
> Fetch the complete documentation index at: [https://code.claude.com/docs/llms.txt](https://code.claude.com/docs/llms.txt)
> 
> Use this file to discover all available pages before exploring further.

Agent view 通过 `claude agents` 打开，是所有后台会话的一个屏幕：什么正在运行、什么需要你的输入、什么已完成。调度新会话，一目了然地查看它们的状态而不是滚动浏览记录，只在需要时才介入。每个后台会话都是一个完整的 Claude Code 对话，在没有终端连接的情况下继续运行，所以你可以随时打开它、回复并离开。
![终端中的 Agent view：标题显示 Claude Code v2.1.140、模型、工作目录和摘要计数。会话分组在'需要输入'、'正在工作'和'已完成'下，底部有调度输入和键盘提示页脚。](../../assets/claude-code-agent-view/image-1.png)

![终端中的 Agent view：标题显示 Claude Code v2.1.140、模型、工作目录和摘要计数。会话分组在'需要输入'、'正在工作'和'已完成'下，底部有调度输入和键盘提示页脚。](../../assets/claude-code-agent-view/image-2.png)

当你有多个独立任务 Claude 可以在不需要你观看每一步的情况下处理时，使用 agent view。调度一个 bug 修复、一个拉取请求审查和一个不稳定测试调查作为三行，在另一个窗口中继续工作，当一行显示它需要你或有结果时检查回来。
当你想在任何代理的会话中更直接地工作时，附加到该行以进入完整对话。
要比较 agent view 与 subagents、agent teams 和 worktrees，请参阅 [并行运行代理](https://code.claude.com/docs/zh-CN/agents)。
Agent view 是研究预览版，需要 Claude Code v2.1.139 或更高版本。使用 `claude --version` 检查你的版本。随着功能的发展，界面和快捷键可能会改变。
本页涵盖：
- [快速开始](#quick-start)：给 Claude 一个在后台处理的任务，检查它，并在需要时介入
- [使用 agent view 监控会话](#monitor-sessions-with-agent-view)，包括状态图标、窥视和回复、附加、组织和快捷键
- [调度新代理](#dispatch-new-agents)，从 agent view、从会话内部或从 shell
- [从 shell 管理会话](#manage-sessions-from-the-shell)
- [后台会话如何被托管](#how-background-sessions-are-hosted)，由监督进程

## ​ 快速开始

本演练涵盖核心 agent view 循环：调度一个任务，观看其行在 Claude 工作时更新，窥视以检查它并回复，以及附加到完整对话。你调度的会话在关闭 agent view 后继续运行，所以你可以离开并稍后回到它。
1[#](#)打开 agent view

从你的 shell，运行：
```bash
claude agents
```

Agent view 打开，底部有一个输入框，当会话启动时表格会填充。随时按 `Esc` 返回你的 shell。你的会话在你离开时继续运行，下次打开 agent view 时会重新出现。2[#](#)调度一个会话

输入描述任务的提示并按 `Enter`。一个新的后台会话在该任务上启动并显示为一行，显示它是否正在工作、等待你或已完成。新会话使用 agent view 标题中显示的模型和在该目录中运行 `claude` 时会获得的相同[权限模式](#permission-mode-model-and-effort)。你在此输入的每个提示都会启动自己的新会话。输入另一个提示并按 `Enter` 会启动第二个会话，与第一个会话并行运行，而不是向其发送后续消息。你可以通过这种方式并行运行多个会话。每个会话独立使用你的订阅配额，所以在一次调度多个会话之前，请查看[限制](#limitations)。3[#](#)窥视和回复

用箭头键选择一行并按 `Space` 打开窥视面板。它显示会话的最近输出，或它正在等待的问题，而不是完整的记录。输入回复并按 `Enter` 发送，无需离开 agent view。4[#](#)附加和分离

在一行上按 `Enter` 或 `→` 在你想要完整对话时附加。会话接管终端，就像你运行了 `claude` 一样。在空提示上按 `←` 分离并返回表格。5[#](#)将现有会话引入

要将你已经打开的会话移入 agent view，在其中运行 `/bg`，或在空提示上按 `←` 以后台会话并在一步中打开 agent view。会话继续运行并显示为一行，与你调度的会话并排。
你可以使用 `claude agents` 作为你的主要入口点而不是 `claude`：从 agent view 调度每个任务，当你想要完整对话时附加，按 `←` 返回表格。

## ​ 使用 agent view 监控会话

运行 `claude agents` 打开 agent view。它接管整个终端并列出按状态分组的每个会话，固定的会话和需要你的会话在顶部。每行显示会话的名称、当前活动和上次更改的时间。
默认情况下，列表显示你启动的每个后台会话，跨越所有项目。在一个存储库中工作的会话和在不同 worktree 中工作的另一个会话都会出现在这里，无论你从哪个目录打开 agent view。要将列表限制到一个项目，请传递 `--cwd`（需要 Claude Code v2.1.141 或更高版本）：

```bash
claude agents --cwd ~/projects/my-app
```

这只显示在该目录下启动的会话。已[移入 worktree](#how-file-edits-are-isolated) 到 `~/projects/my-app/.claude/worktrees/` 下的会话仍然算作属于 `~/projects/my-app`。
你在其他终端中打开的交互式会话不会出现，直到你[后台它们](#from-inside-a-session)。[Subagents](https://code.claude.com/docs/zh-CN/sub-agents) 和 [teammates](https://code.claude.com/docs/zh-CN/agent-teams) 会话生成的不会列为单独的行。

```
Pinned

  ✽ clawd walk cycle          Write assets/sprites/clawd-walk.png           3m

Ready for review

  ∙ jump physics              github.com/example/game/pull/2048          ●  2h

Needs input

  ✻ power-up design           needs input: double jump or wall climb?       1m

Working

  ✽ collision detection       Edit src/physics/CollisionSystem.ts           2m

  ✢ playtest level 3          run 12 · all checkpoints cleared           in 4m

Completed

  ✻ title screen              result: menu, options, and credits done       9m

  ∙ sound effects             result: 14 SFX exported to assets/audio       4h

  … 6 more
```

### ​ 读取会话状态

每行以一个图标开头，其颜色和动画显示会话的状态：
| 状态 | 图标显示为 | 含义 |
| --- | --- | --- |
| 工作中 | 动画 | Claude 正在积极运行工具或生成响应 |
| 需要输入 | 黄色 | Claude 等待你的特定问题或权限决定 |
| 空闲 | 暗淡 | 会话没有任何事情要做，准备好接收你的下一个提示 |
| 已完成 | 绿色 | 任务成功完成 |
| 失败 | 红色 | 任务以错误结束 |
| 已停止 | 灰色 | 会话被 Ctrl+X 或 claude stop 停止 |

另外，图标的形状显示底层进程是否正在运行：
| 形状 | 含义 |
| --- | --- |
| ✻ 或动画 ✽ | 会话进程处于活跃状态并立即回复 |
| ∙ | 进程已退出。你仍然可以窥视、回复或附加，Claude 从中断处重新启动 |
| ✢ | 一个 /loop 会话在迭代之间休眠。该行显示其运行计数和倒计时 |

行右边缘可能出现的 `●` 是[拉取请求状态](#pull-request-status)指示器，不是状态图标的一部分。它前面的数字是会话打开的拉取请求数。
后台会话不需要任何打开的终端来继续工作。一个单独的[监督进程](#the-supervisor-process)运行它们，所以你可以关闭 agent view、关闭你的 shell 或启动一个新的交互式会话，你的调度工作继续进行。
会话状态通过自动更新和监督进程重启在磁盘上持久化。会话在你的机器休眠时也会被保留。它们的进程在唤醒时恢复，监督进程重新连接到它们，而不是将时间间隙视为空闲。关闭仍然会停止运行中的会话；请参阅[关闭后会话显示为失败](#sessions-show-as-failed-after-shutdown)了解如何恢复它们。

### ​ 行摘要

每行中的单行摘要由 [Haiku-class 模型](https://code.claude.com/docs/zh-CN/model-config)生成，所以该行可以告诉你会话正在做什么、需要什么或生成了什么，无需打开记录。当会话正在积极工作时，摘要最多每 15 秒刷新一次，加上每个回合结束时刷新一次。
每次刷新是通过你的正常提供商的一个短 Haiku-class 请求，按与会话本身相同的[数据使用条款](https://code.claude.com/docs/zh-CN/data-usage)计费和处理。

### ​ 拉取请求状态

当会话打开拉取请求时，状态点出现在行的右边缘，在支持超链接的终端中链接到拉取请求。当会话打开了多个拉取请求时，计数出现在点之前，颜色反映最需要关注的那个。
| 点的颜色 | 拉取请求状态 |
| --- | --- |
| 黄色 | 等待检查或审查，或检查失败 |
| 绿色 | 检查通过且没有审查阻止 |
| 紫色 | 已合并 |
| 灰色 | 草稿或已关闭 |

对于大多数任务，这行是你收集结果的地方：当点变绿时审查和合并拉取请求。

### ​ 窥视和回复

在选定的行上按 `Space` 打开窥视面板。它显示会话需要什么、其最近的输出和它打开的任何拉取请求。大多数时候这就足够了，你永远不需要打开完整的记录。
在窥视面板中输入回复并按 `Enter` 将其发送到该会话。当会话提出多选问题时，窥视面板显示选项，你可以按数字键选择一个。对于其他被阻止的会话，按 `Tab` 用建议的回复填充输入，你可以在发送前编辑。用 `!` 前缀回复以发送 Bash 命令。
使用 `↑` 和 `↓` 窥视相邻会话而不关闭面板，或 `→` 附加。

### ​ 附加到会话

在选定的行上按 `Enter` 或 `→` 附加。Agent view 被完整的交互式会话替换，就像你在该目录中运行了 `claude` 一样。当你附加时，Claude 发布一个关于你离开时发生的事情的简短回顾。
附加时，会话的行为像任何其他 Claude Code 会话：每个[命令](https://code.claude.com/docs/zh-CN/commands)、快捷键和功能都有效。
在空提示上按 `←` 分离并返回 agent view。如果对话有焦点且不响应 `←`，按 `Ctrl+Z` 立即分离。
分离永远不会停止后台会话：`←`、`Ctrl+C`、`Ctrl+D`、`Ctrl+Z` 和 `/exit` 都让它运行。要从内部结束会话，运行 `/stop`。
在你调度或后台了会话后，在空提示上按 `←` 从任何 Claude Code 会话工作，不仅仅是你从 agent view 附加的会话。它后台当前会话并打开 agent view，该行被选中，所以你可以在不离开终端的情况下切换会话。该行即使从没有对话历史的新会话也会被创建，所以 `→` 会返回到它。当该行是唯一的行时，agent view 在它下方显示一个入门提示。你可以在 `/config` 中关闭此快捷键（`leftArrowOpensAgents` 设置）。

### ​ 组织列表

Agent view 按状态分组会话，需要输入的会话在顶部，`Ready for review` 和 `Needs input` 在 `Working` 和 `Completed` 上方。这些组名不与上面的[状态](#read-session-state)一一对应：当会话有打开的拉取请求时，它移动到 `Ready for review`，`Completed` 收集已完成、失败和已停止的会话。按 `Ctrl+S` 改为按目录分组。你的选择在运行中保存。
在一个组内：
- 按 `Ctrl+T` 将会话固定到顶部
- 按 `Shift+↑` 或 `Shift+↓` 重新排序会话
- 按 `Ctrl+R` 重命名会话
- 在组标题上按 `Enter` 折叠它

要从列表中删除会话，按 `Ctrl+X` 停止它，在两秒内再按 `Ctrl+X` 删除它。在组标题上按 `Ctrl+X` 在确认后删除该组中的每个会话。
删除会从 agent view 中删除会话。如果 Claude [为会话创建了 worktree](#how-file-edits-are-isolated)，删除会删除该 worktree，包括其中的任何未提交的更改，所以在删除前推送或提交你想保留的工作。你自己创建的 worktree 并在其中启动会话的会被保留。对话记录保留在你的本地机器上，并且仍然可以通过 `claude --resume` 访问。
较旧的已完成会话折叠成 `… N more` 行以保持列表简短。失败和有打开拉取请求的会话始终保持可见。

### ​ 过滤会话

在调度输入中输入以过滤而不是调度：
| 过滤 | 显示 |
| --- | --- |
| a:<name> | 运行命名代理的会话 |
| s:<state> | 给定状态的会话，例如 s:working 。也接受 s:blocked 用于等待你的所有内容 |
| #<number> 或 PR URL | 处理该拉取请求的会话 |

### ​ 快捷键

在 agent view 中按 `?` 查看每个快捷键的上下文。下表总结了它们。
| 快捷键 | 操作 |
| --- | --- |
| ↑ / ↓ | 在行之间移动 |
| Enter | 附加到选定的会话，或如果输入中有文本则调度 |
| Space | 打开或关闭选定会话的窥视面板 |
| Shift+Enter | 调度并立即附加 |
| → | 附加到选定的会话 |
| Alt+1 .. Alt+9 | 附加到当前目录中的第 1–9 个会话 |
| Tab | 在空输入上浏览所有 subagents。否则应用突出显示的建议 |
| Ctrl+S | 在状态和目录之间切换分组 |
| Ctrl+T | 固定或取消固定选定的会话 |
| Ctrl+R | 重命名选定的会话 |
| Ctrl+G | 在你的 $VISUAL 或 $EDITOR 中打开调度提示 |
| Ctrl+X | 停止会话；在两秒内再按一次删除它 |
| Shift+↑ / Shift+↓ | 重新排序选定的会话 |
| Esc | 关闭窥视面板、清除输入或退出 |
| Ctrl+C | 清除输入；按两次退出 |
| ? | 显示所有快捷键 |

## ​ 调度新代理

你可以从 agent view 调度新的后台会话、将现有的交互式会话发送到后台，或直接从 shell 启动一个。

### ​ 从 agent view

在 agent view 底部的输入框中输入提示并按 `Enter` 启动新的后台会话。会话从提示自动命名；稍后可以用 `Ctrl+R` 重命名它。
将图像粘贴到提示中以包含任务的屏幕截图或图表。
前缀或提及提示的部分以控制会话如何启动：
| 输入 | 效果 |
| --- | --- |
| <agent-name> <prompt> | 如果第一个单词匹配自定义 subagent 名称，该 subagent 作为会话的主代理运行，使用其 frontmatter 中的配置 |
| @<agent-name> | 在提示中的任何地方提及自定义 subagent 以作为主代理运行它 |
| @<repo> | 提及你打开 agent view 的目录下的存储库以在那里运行会话 |
| /<skill> | 建议 skills 作为提示调度 |
| #<number> 或拉取请求 URL | 如果会话已在处理该 PR，选择它而不是调度 |
| Shift+Enter | 调度并立即附加到新会话 |

将重复任务打包为 [skill](https://code.claude.com/docs/zh-CN/skills) 让你从 agent view 多次启动相同的工作流而无需重新输入提示。
当相同的 `@name` 同时匹配 subagent 和同级存储库时，subagent 优先。不带 `@` 的首字形式也适用，所以以匹配你的某个 subagent 名称的单词开头的提示会调度该 subagent 而不是将该单词视为纯文本。当你想要明确指定时，使用 `@` 形式，或以不同的单词开头提示以避免匹配。

#### ​ 调度到特定目录

新会话在你打开 agent view 的目录中运行。要针对不同的目录：
- 在该目录中打开 `claude agents`。
- 在保存多个存储库的父目录中打开 `claude agents`，并在提示中用 `@<repo>` 提及一个以在那里运行会话。
- 从 shell，`cd` 进入目录并运行 `claude --bg "<prompt>"`。

当 agent view 按目录分组时，突出显示的行的目录成为调度目标，所以你可以滚动到一个组并在不重新输入路径的情况下调度到它。

### ​ 从会话内部

运行 `/background` 或其别名 `/bg` 将当前对话移动到后台会话。传递提示如 `/bg run the test suite and fix any failures` 以在后台化前先给出一个更多指令。
从交互式会话后台化启动一个新的进程，该进程从保存的对话恢复，所以运行 subagent、[monitors](https://code.claude.com/docs/zh-CN/tools-reference#monitor-tool) 和后台命令不会转移到它。当任何正在运行时，Claude 会要求你在后台化前确认。一旦在后台，会话可以启动新的 subagent、monitor 和后台命令，这些会在后续的分离和重新附加中保持运行。
来自原始启动的配置标志会传递到后台化的会话，所以其 MCP servers、settings 和备用模型保持有效：
- `--mcp-config` 和 `--strict-mcp-config`
- `--settings`
- `--add-dir`
- `--plugin-dir`
- `--fallback-model`
- `--allow-dangerously-skip-permissions`

你在会话期间用 [`/add-dir`](https://code.claude.com/docs/zh-CN/permissions#additional-directories-grant-file-access-not-configuration) 添加的目录也会传递。
传递 `--allow-dangerously-skip-permissions` 会在后台化的会话中保持 `bypassPermissions` 可访问，但它不会授予任何新权限。该模式仍然需要在任何会话使用它之前进行相同的一次性交互式接受，如 [权限模式、模型和工作量](#permission-mode-model-and-effort) 中所述。

### ​ 从你的 shell

传递 `--bg` 启动直接进入后台的会话：

```bash
claude --bg "investigate the flaky SettingsChangeDetector test"
```

要运行特定的 subagent 作为会话的主代理，结合 `--bg` 和 `--agent`：

```bash
claude --agent code-reviewer --bg "address review comments on PR 1234"
```

传递 `--name` 以在 agent view 中设置会话的显示名称而不是自动生成的名称：

```bash
claude --bg --name "flaky-test-fix" "investigate the flaky SettingsChangeDetector test"
```

后台化后，Claude 打印会话的短 ID 和管理它的命令。当你传递 `--name` 时，名称出现在短 ID 之后：

```
backgrounded · 7c5dcf5d · flaky-test-fix

  claude agents             list sessions

  claude attach 7c5dcf5d    open in this terminal

  claude logs 7c5dcf5d      show recent output

  claude stop 7c5dcf5d      stop this session
```

### ​ 文件编辑如何隔离

每个后台会话，无论是从 agent view、`/bg` 还是 `claude --bg` 启动，都在你的工作目录中启动。在编辑文件前，Claude 将会话移动到 `.claude/worktrees/` 下的隔离 [git worktree](https://code.claude.com/docs/zh-CN/worktrees) 中，所以并行会话可以读取相同的检出但每个都写入自己的。
Claude 在以下情况下跳过 worktree：
- 会话已经在链接的 git worktree 内，无论 Claude 是在 `.claude/worktrees/` 下创建的还是你用 `git worktree add` 在其他地方创建的
- 工作目录不是 git 存储库且没有配置 [`WorktreeCreate` hook](https://code.claude.com/docs/zh-CN/hooks#worktreecreate)
- 写入在工作目录外

要为 git worktree 不实用的存储库关闭 worktree 隔离，将 [`worktree.bgIsolation`](https://code.claude.com/docs/zh-CN/settings#worktree-settings) 设置为 `"none"`。后台会话随后直接编辑你的工作副本而不先移动到 worktree。将设置添加到项目的 `.claude/settings.json`：

```
{

  "worktree"
: {

    "bgIsolation"
: 
"none"

  }

}
```

`worktree.bgIsolation` 设置需要 Claude Code v2.1.143 或更高版本。
在 git 存储库外，会话直接写入工作目录且彼此不隔离，所以避免调度编辑相同文件的并行会话。如果你使用不同的版本控制系统，配置一个 [`WorktreeCreate` hook](https://code.claude.com/docs/zh-CN/worktrees#non-git-version-control)，Claude 会以与 git 相同的方式隔离编辑。
在 agent view 中删除会话（`Ctrl+X` 两次）会删除 Claude 为其创建的 worktree，包括任何未提交的更改，所以在删除前合并或推送你想保留的更改。从 shell 用 [`claude rm`](#manage-sessions-from-the-shell) 删除会保持有未提交更改的 worktree 并打印其路径，以便你可以自己清理它。你自己创建的 worktree 并在其中启动会话的，无论哪种方式都会保留在原地。
要找到会话的 worktree 路径，查看会话或附加并检查其工作目录。
要使 subagent 始终在其自己的 worktree 中运行，无论如何启动，在其 frontmatter 中设置 [`isolation: worktree`](https://code.claude.com/docs/zh-CN/sub-agents#supported-frontmatter-fields)。

### ​ 设置模型

agent view 标题中显示的模型名称是调度默认值。你从输入启动的新会话使用此模型，这来自你的用户设置中的 [`model` 设置](https://code.claude.com/docs/zh-CN/settings#available-settings)。通过在 [`/model` 选择器](https://code.claude.com/docs/zh-CN/model-config) 中按 `d` 在模型上设置它，或直接编辑设置。要为整个 agent view 会话覆盖它，在打开 agent view 时传递 `--model`。参见 [权限模式、模型和工作量](#permission-mode-model-and-effort)。
每个后台会话可以在不同的模型上运行。要为一个会话覆盖它：
- 从 shell，用 `claude --bg` 传递 `--model`。
- 附加到运行中的会话并在那里运行 `/model`。如果会话被重新生成，更改会持续。
- 调度一个 [subagent](https://code.claude.com/docs/zh-CN/sub-agents)，其 frontmatter 设置 `model` 字段。

### ​ 权限模式、模型和工作量

后台会话从它运行的目录读取其 [settings](https://code.claude.com/docs/zh-CN/settings)，就像你在那里启动了 `claude` 一样。
[permission mode](https://code.claude.com/docs/zh-CN/permissions) 取决于你如何启动会话。用 `/bg` 或 `←` 后台化现有会话会保持当前权限模式，所以你切换到 `acceptEdits` 或 `auto` 的会话在分离后仍保持该模式。从 agent view 输入调度或从你的 shell 运行 `claude --bg` 使用该目录设置中的 `defaultMode`，或调度的 [subagent 的 frontmatter](https://code.claude.com/docs/zh-CN/sub-agents#supported-frontmatter-fields) 中的 `permissionMode`。
你启动后台会话时的权限模式在监督者稍后 [停止并重新启动](#the-supervisor-process) 会话的进程时持续。你用 `claude --bg --dangerously-skip-permissions` 或 `claude --bg --permission-mode bypassPermissions` 启动的会话在该重新启动后仍保持 `bypassPermissions` 而不是回退到目录的 `defaultMode`。
要为从 agent view 调度的每个会话设置默认值，在打开它时传递 `--permission-mode`、`--model` 或 `--effort` 中的任何一个：

```bash
claude agents --permission-mode plan --model opus --effort high
```

`claude agents` 也接受 `--dangerously-skip-permissions` 作为 `--permission-mode bypassPermissions` 的简写，以及 `--allow-dangerously-skip-permissions` 以在每个调度会话的 `Shift+Tab` 循环中使 `bypassPermissions` 可用而不带权限模式启动。两者都匹配 [顶级 CLI 标志](https://code.claude.com/docs/zh-CN/cli-reference)。
向 `claude agents` 传递 `--permission-mode`、`--model`、`--effort` 或 `--dangerously-skip-permissions` 需要 Claude Code v2.1.142 或更高版本。`claude agents` 上的 `--allow-dangerously-skip-permissions` 需要 v2.1.143 或更高版本。更早的版本会以未知选项错误拒绝这些标志。
活跃的默认值出现在调度输入下方的页脚中。
没有这些标志，会话使用该目录设置中的 `defaultMode` 或调度的 [subagent 的 frontmatter](https://code.claude.com/docs/zh-CN/sub-agents#supported-frontmatter-fields) 中的 `permissionMode`，以及 agent view 标题中显示的模型。
使用 `bypassPermissions` 或 `auto` 被拒绝，直到你通过交互式运行 `claude` 一次接受该模式，因为这些模式让你没有看到的会话无需批准就能行动。无论你是将模式传递给 `claude agents` 还是 `claude --bg --permission-mode`，同样的规则都适用。

### ​ Settings、plugins 和 MCP servers

Agent view 接受与 `claude` 相同的配置标志以加载 settings、plugins、MCP servers 和额外目录。这些标志需要 Claude Code v2.1.142 或更高版本。每个标志适用于 agent view 本身，并传递给你从它调度的每个会话，所以以这种方式加载的 plugin 或 MCP server 在这些会话中也可用。
| 标志 | 效果 |
| --- | --- |
| --settings <file-or-json> | 覆盖 agent view 和调度会话的 settings |
| --add-dir <path> | 授予对额外目录的文件访问权限 |
| --plugin-dir <path> | 从本地目录加载 plugin |
| --mcp-config <file-or-json> | 从配置文件或 JSON 字符串加载 MCP servers |
| --strict-mcp-config | 仅使用来自 --mcp-config 的 MCP servers，忽略其他 MCP 配置 |

对每个值重复 `--add-dir`、`--plugin-dir` 或 `--mcp-config`。空格分隔的形式，如 `--add-dir a b c`，不支持与 `claude agents` 一起使用。
以下示例使用 settings 覆盖和一个额外目录打开 agent view：

```bash
claude agents --settings ./ci-settings.json --add-dir ../shared-lib
```

## ​ 从 shell 管理会话

每个后台会话有一个短 ID，你可以从 shell 使用。当你使用 `claude --bg` 启动会话时会打印该 ID，每个会话的 ID 是其在 `~/.claude/jobs/` 下的目录名。这些命令对于脚本编写或当你不想打开 agent view 时很有用。
| 命令 | 目的 |
| --- | --- |
| claude agents | 打开 agent view |
| claude agents --cwd <path> | 打开 agent view，范围限定为在 <path> 下启动的会话 |
| claude agents --json | 将活跃会话打印为 JSON 数组并退出。每个条目都有 pid 、 cwd 、 kind 和 startedAt ，以及设置时的 sessionId 、 name 和 status 。与 --cwd <path> 结合使用以进行过滤 |
| claude attach <id> | 在此终端附加到会话 |
| claude logs <id> | 打印会话的最近输出 |
| claude stop <id> | 停止会话。也接受 claude kill |
| claude respawn <id> | 重新启动会话（运行中或已停止），保持其对话完整，例如用于获取更新的 Claude Code 二进制文件 |
| claude respawn --all | 重新启动每个运行中的会话，例如一次性将所有会话移至更新的 Claude Code 二进制文件 |
| claude rm <id> | 从列表中删除会话。如果没有未提交的更改，会删除 Claude 为会话创建的 worktree；否则打印 worktree 路径以便你清理。保留你自己创建的 worktree。对话记录保存在你的本地机器上，并且仍然可以通过 claude --resume 访问 |
| claude daemon status | 打印 supervisor 的状态、版本、socket 目录和 worker 数量 |

## ​ 后台会话如何被托管

agent view 中列出的每个会话都被视为后台会话，无论你当前是否连接到它。相比之下，通过直接运行 `claude` 启动的会话与该终端绑定，并在终端关闭时结束，除非你[将其发送到后台](#from-inside-a-session)。

### ​ 监督进程

后台会话由每用户监督进程托管，与你的终端和 agent view 分离。监督进程在你第一次后台会话或打开 agent view 时自动启动，你不直接管理它。
监督进程及其会话使用与你的交互式会话相同的凭证进行身份验证，并且除了模型 API 外不进行额外的网络连接。
每个后台会话是其自己的 Claude Code 进程，由监督进程管理而不是与你的终端绑定。积极工作、等待你的输入或有终端连接的会话保持其进程运行。运行中的后台 shell 命令、子代理、工作流或监视器计为活跃工作，因此长时间运行的进程（如开发服务器）会保持会话活跃。
一旦会话完成并未连接地坐了大约一小时，监督进程停止其进程以释放资源。记录和状态保留在磁盘上，下次你附加、窥视或回复时，监督进程从中断处启动一个新进程。当每个会话都完成且没有终端连接时，监督进程本身退出，下次你需要它时再次启动。
监督进程监视磁盘上安装的 Claude Code 二进制文件，在常规[自动更新程序](https://code.claude.com/docs/zh-CN/setup#auto-updates)替换它后重新启动到新版本。这是本地文件监视，不是网络检查。后台会话是分离的进程，所以它们在重新启动期间继续运行，新的监督进程重新连接到它们。

### ​ 状态存储位置

会话状态存储在你的 Claude Code 配置目录下。如果你设置了 [`CLAUDE_CONFIG_DIR`](https://code.claude.com/docs/zh-CN/env-vars)，监督进程使用该目录而不是 `~/.claude` 并作为单独的实例运行，具有其自己的会话。
| 路径 | 内容 |
| --- | --- |
| ~/.claude/daemon.log | 监督进程日志 |
| ~/.claude/daemon/roster.json | 运行中的后台会话列表，用于在重新启动后重新连接 |
| ~/.claude/jobs/<id>/state.json | 在 agent view 中显示的每会话状态 |

要在不直接读取文件的情况下检查此状态，请运行 `claude daemon status`。它报告监督进程是否可达、其进程 ID 和版本、套接字目录以及有多少后台会话处于活跃状态。`/doctor` 包括相同检查的摘要。在 Windows 上，当守护进程的管道密钥文件被锁定或无法读取时，`claude daemon status` 会显示底层文件错误，而不是报告通用连接失败。

### ​ 关闭 agent view

要完全关闭后台代理和 agent view，将 `disableAgentView` [设置](https://code.claude.com/docs/zh-CN/settings)设为 `true` 或设置 `CLAUDE_CODE_DISABLE_AGENT_VIEW` 环境变量。管理员可以通过[托管设置](https://code.claude.com/docs/zh-CN/permissions#managed-settings)强制执行这个。

## ​ 故障排除

### ​ claude agents 列出子代理而不是打开代理视图

如果 `claude agents` 打印一个计数，然后是你配置的子代理，然后退出，说明代理视图在你的环境中不可用。早期版本不会在每个环境中打开代理视图，包括通过 Bedrock、Vertex AI 或 Foundry 连接时。运行 `claude update` 来安装最新版本。
如果更新后代理视图仍然没有打开，检查它是否已被设置或环境变量[关闭](#turn-off-agent-view)。

### ​ Agent view 打开时没有会话

在你调度你的第一个会话之前，agent view 显示一个简短的入门提示，在会话列表的位置显示示例提示。在底部的输入框中输入提示并按 `Enter` 来调度你的第一个会话。

### ​ 无法打开代理，因为后台任务正在运行

如果按 `←` 来后台当前会话显示 `Cannot open agents — N background task(s) running`，说明会话有进行中的工作，例如子代理、工作流或后台 shell 命令，快捷键不会默默放弃它。运行 `/tasks` 来查看正在运行的内容，然后运行 `/bg` 来确认放弃它们。参见[从会话内部](#from-inside-a-session)了解后台时什么会转移，什么不会转移。

### ​ 提示被拒绝，因为太短

调度输入期望一个任务描述，而不是对话开场白。少于四个字符的提示会被拒绝，并显示 `Too short` 提示，这样随意的按键就不会启动会话。描述你希望会话执行的操作，例如 `investigate the flaky checkout test`。

### ​ 会话在关闭后显示为已失败

关闭或重启你的机器会停止运行中的后台会话，所以当你下次打开 agent view 时，它们显示为已失败。附加、窥视或回复任何已失败的会话，会话从中断处重新启动。
睡眠单独不会导致这种情况。会话在睡眠期间被保留，监督进程在唤醒时重新连接到它们。

### ​ 附加后会话响应缓慢

一旦会话完成并未连接地坐了大约一小时，监督进程停止其进程以释放资源。附加启动一个从中断处的新进程，这需要一点时间。工作或等待你的会话永远不会以这种方式停止。

### ​ .claude/worktrees/ 填满了

在 agent view 中删除会话会删除 Claude 为其创建的 worktree。`claude rm` 保留具有未提交更改的 worktree 并打印其路径。在项目目录中用 `git worktree list` 列出剩余条目，并用 `git worktree remove <path>` 删除每个。参见[清理 worktrees](https://code.claude.com/docs/zh-CN/worktrees#clean-up-worktrees)。

## ​ 限制

Agent view 处于研究预览阶段，存在以下限制：
- **速率限制适用**：后台会话消耗你的订阅使用量，与交互式会话相同，因此并行运行十个代理的配额消耗速度大约是运行一个代理的十倍。
- **会话是本地的**：后台会话在你的机器上运行。它们在机器睡眠时保留，但在机器关闭时停止。
- **Claude 创建的 worktrees 在 agent view 中随会话删除**：在删除在其自己的 worktree 中编辑文件的会话之前，请合并或推送更改。`claude rm` 保留具有未提交更改的 worktree；你自己创建的 worktree 保持原位。

## ​ 相关资源

有关以并行方式运行 Claude 的其他方法，请参阅：
- [在并行中运行代理](https://code.claude.com/docs/zh-CN/agents)：比较 agent view 与 subagents、agent teams 和 worktrees
- [Agent teams](https://code.claude.com/docs/zh-CN/agent-teams)：协调相互发送消息的多个会话
- [Claude Code on the web](https://code.claude.com/docs/zh-CN/claude-code-on-the-web)：在托管的云环境中运行会话而不是本地
