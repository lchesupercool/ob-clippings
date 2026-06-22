---
title: 我所知道的所有智能体工程技巧（2026年6月）
author: Matt Van Horn (@mvanhorn)
source: https://x.com/mvanhorn/status/2061877533885473181?s=20
saved: 2026-06-08 12:53:48
type: x-post
language: zh
tags: [ai-coding, claude-code, agentic-engineering, workflows]
extraction: browser-twitterArticleReadView
---

# 我所知道的所有智能体工程技巧（2026年6月）

![Image](../../assets/mvanhorn-agentic-engineering-hacks-june-2026/image-1.jpg)

三个月前我发布了一篇[《我所知道的所有 Claude Code 技巧》](https://x.com/mvanhorn/status/2035857346602340637)，获得了 91.3 万次浏览。[@kevinrose](https://x.com/@kevinrose) 曾问我用什么 IDE，我的回答是："不用 IDE。只用 plan.md 文件和语音。"

这曾经被称为 vibe coding。大约在去年感恩节前后，模型变得足够好，这个玩具变成了真实的工具，人们现在称之为智能体工程（Agentic Engineering）。这是我能够交付的唯一原因。今年我发布了 [last30days](https://github.com/mvanhorn/last30days-skill)（27K 星）、[Printing Press](https://printingpress.dev/)（4K+ 星）以及 [Agent Cookie](https://agentcookie.dev/)，刚刚发布，并成为了一些最大开源项目的顶级贡献者：[Python](https://github.com/python/cpython)、[Go](https://github.com/golang/go)、[GStack](https://github.com/garrytan/gstack) 和 [Paperclip](https://github.com/paperclipai/paperclip)。我从高中以来就没有发布过任何别人看得上的软件。这些是我的技巧。

### 技巧

YOLO TL;DR 技巧：把整篇文章粘贴给你的智能体，让它制定一个计划来设置其中的所有内容，然后一次一个技巧地执行这个计划。这就是我的全部技术栈，不需要阅读。

## 1. 有想法的瞬间，立即做一个 CE plan.md

仍然是第一条规则。仍然是我学到的最重要的事情。

我一旦有了想法，就是 `/ce-plan` 来生成一个 plan.md。不是"让我想想"，不是"让我开始写代码"。每次都是 `/ce-plan`。它也能接受图片，所以任何你能捕捉到的东西都可以作为起点：

疯狂的产品创意：`/ce-plan`。

GitHub 上的 bug：复制 issue URL，粘贴进去，`/ce-plan`。

终端错误：Cmd+Shift+4 截图，Ctrl+V 粘贴，`/ce-plan fix this`。

截图、错误信息、设计稿、Slack 讨论串：通通丢进去。

当想法还很模糊，我甚至不知道自己想要什么的时候，我先用 `/ce-brainstorm` 和智能体一起理清思路，等想法清晰后再用 `/ce-plan`。

在底层，`/ce-plan` 会并行启动研究智能体。一个阅读你的代码库，找到模式，检查你的编码规范。一个搜索你过去的解决方案以获取经验。如果话题需要，还会有更多智能体去研究外部文档和最佳实践。所有这些都是同时进行的。然后它整合并写出一份结构化的 plan.md：问题是什么、解决方案思路、涉及哪些文件、带有复选框的验收标准、从你自己的代码中提取的应遵循的模式。所有这些都基于你的仓库、你的规范、你的历史。不是泛泛的建议。

`/ce-work` 接受这个计划并构建它。上下文爆了？开启一个新会话，指向这个计划，从你中断的地方继续。计划是能扛过一切的检查点。

传统开发是 80% 编码，20% 规划。这个流程翻转了它。思考放进计划里。执行是机械的。

[Compound Engineering](https://github.com/EveryInc/compound-engineering-plugin)，来自 [@kieranklaassen](https://x.com/@kieranklaassen) 和 [@trevin](https://x.com/@trevin)，是让它成为现实的插件。

我先是成为超级粉丝，然后成为贡献者，现在我是仅次于核心团队的第三大贡献者。我现在的规则是：除非真的是一个一行代码的改动，否则总是先有一个 plan.md。

### 技巧

安装 Compound Engineering：`/plugin marketplace add EveryInc/compound-engineering-plugin`

粘贴截图、bug URL 或错误信息，然后 `/ce-plan`，然后 `/ce-work`。

想法模糊？先用 `/ce-brainstorm`。

## 2. 不要读 plan.md

我总是会制定 plan.md。我几乎从不阅读它。计划是给智能体的，你这个愚蠢的人类。

强制计划的存在能让智能体不偷懒。它迫使它们做研究、承诺一个方案、写下验收标准，然后真正地去达到它们。有计划的编码智能体会交付完成的工作。没有计划的编码智能体会走捷径、提前停止。计划就是那条牵绳。

所以我让它写计划，我扫一眼标题，然后运行 `/ce-work`。如果我有问题，我就在会话中直接问："等等，为什么用这个方案？"或者我要求一个 TLDR。或者，当我不理解的时候，"eli5 this plan。"我得到一段话的版本，点点头，继续。我不会坐在那里读 300 行 markdown。那是智能体的作业，不是我的。

制定计划。信任计划。不要读计划。

### 技巧

别让自己读计划。直接问：TLDR？，eli5 this plan，或者"等等，为什么用这个方案？"

## 3. 用 `/ce-plan` 做最深度的非工程工作，为计划制定计划

人们以为 `/ce-plan` 和 `/ce-work` 是用来写代码的。自三月以来我学到的最重要的事情是，它们不是。我现在做的最深度的知识工作也走同一个循环，而诀窍是让第一个计划成为"计划的计划"。这也不是我把一个写代码的工具强行用来做它不擅长的事：`/ce-plan` 内置了通用规划模式，就是为这类非代码工作设计的。

也不仅仅是商业问题。策略文档、产品规格、竞争分析、董事会汇报，全都走同一个循环。

这是一个真实的例子。我和 Michael Margolis 见了面，他是前 GV 研究合伙人，以靶心客户方法闻名，讨论我在酝酿的一个商业挑战。他让我读他的书，在他的网站上免费提供 PDF。过去的做法会是浏览一下然后就过去了。相反，我打开 Claude Code 大致说了：

"/ce-plan 为接下来的计划制定一个计划。我马上要交给你两样东西：Margolis 的书（PDF），以及我刚刚和他开的两小时会议的 [Granola](https://granola.ai/) 转录，里面有我们讨论的完整上下文。我想要一个深思熟虑的计划，关于我的商业问题、那次对话以及书中的经验如何整合成我能实际使用的东西。现在不要写那个文档。写文档是之后的工作。现在我只想要一个计划，关于你将如何阅读那本书、挖掘那个转录，并产出一份优秀的文档。"

它花了接下来的 45 分钟制定了一个史诗级的计划。

这也是我知道的让 LLM 不偷懒的最佳单一技巧。直接要求交付成果，它会走捷径。要求它先规划如何产出交付成果，然后执行那个计划，它每次都做深度版本。

### 技巧

深度非代码工作：`/ce-plan make a plan for the plan`，把所有的上下文和转录交给它，然后 `/ce-work`。

## 4. 拥抱语音输入

语音转 LLM 和语音转其他任何东西都不一样。转录不需要完美，因为聆听者理解上下文。它能猜测麦克风捕捉错的地方。你可以含糊不清、话说到一半停掉、重新开始说。语音终于能用了，因为接收端的那东西足够聪明，能填补空白。

我的设置：

Mac：[Monologue](https://monologue.to/)（来自 Every）或 [Wispr Flow](https://wisprflow.ai/)。选一个，把语音输入到当前激活的应用里，对 Claude Code 说话。我为办公室买了一支[鹅颈麦克风](https://www.amazon.com/dp/B0BF969RVP)。

手机：跳过 Monologue 和 Wispr Flow，在 iOS 上切换它们太烦人。Apple 内置的语音输入就足够好了，因为你在跟 LLM 说话，不是跟人说话。它可以错一半的字，智能体仍然能理解。偷懒的笔记也没问题。

一个诚实的坦白：我一个人时用语音很棒。在办公室里我挣扎。人们说你可以对着麦克风低语，但我发现自己实际上做不到，因为我不想显得无礼或打扰周围的人。所以共享房间里的办公桌仍然是我整个工作流的薄弱之处。如果你在开放办公室里破解了语音使用，而且没有成为那种讨厌的人，告诉我你是怎么做到的。我真的想要这个建议。

### 技巧

Mac：安装 Monologue 或 Wispr Flow。手机：使用 Apple 语音输入。买一个鹅颈麦克风。

## 5. 在 [cmux](https://cmux.com/) 里开很多很多的标签页

这就是我实际度过一天的方式。四到六个 cmux 标签页，有时候更多，每个是一个独立会话：

一个在写计划。

一个在根据另一个计划构建。

一个在运行 last30days。

一个在修复我刚测试上一个东西时发现的 bug。

当一个窗口里 `/ce-plan` 在启动研究时，我切换到另一个窗口对已写好的计划执行 `/ce-work`。当那个在构建时，第三个窗口里我粘贴了一个新的 bug。等我循环回来，第一个已经完成并在等着我了。

我听到了很多关于 [Orca](https://onorca.dev/) 的好评，针对他们正在做的移动端工作。我也曾经是一个 [Ghostty](https://ghostty.org/) 纯粹主义者，但我在 ghostty 里漏掉了太多通知。

### 技巧

使用 cmux。

保持 4 到 6 个标签页打开，每个里面一个不同的任务。

## 6. 让你的终端默认进入 Claude 或 Codex，而不是 Shell

新标签页应该直接打开 Claude Code，而不是一个 shell。打开一个标签页，你已经在跟智能体对话。没有 cd，没有敲 claude。当新会话的开启成本只有一个按键时，你会开启更多会话。我也不使用文件夹。你的智能体能找到你的项目。

### 技巧

粘贴给你的智能体："让每个新的终端标签页直接打开 Claude Code。在 `~/.config/ghostty/config` 中添加 `command = ~/.local/bin/claude-launcher.sh`，不要动这个文件里其他的任何设置。然后创建 `~/.local/bin/claude-launcher.sh`，它运行 `claude --dangerously-skip-permissions`，并在 Claude 退出时打印一行简短提示，然后进入一个交互式登录 zsh。给脚本 `chmod +x`。这对 Ghostty 和 cmux 都适用，因为 cmux 读的是同一个 Ghostty 配置。"

## 7. 远程控制每一个窗口，给 Claude Code 或 Codex 一个邮箱地址

两个让每个会话都可以从任何地方访问的技巧。

每个新窗口打开时都开启远程控制

把远程控制设置为每个会话自动开启。

现在每个窗口都可以从 Claude 移动端应用访问。在你桌前开始一个会话，走开，用手机接着同一个正在运行的会话继续工作。在排队时，你在操控家里 Mac 上正在运转的东西。

给你的 Claude 一个电子邮件地址

Claude Code 可以通过 [AgentMail](https://agentmail.to/) 拥有一个电子邮件地址。创始人 Adi [@adisingh](https://x.com/@adisingh) 教会了我这个。发送邮件到那个收件箱，一个新会话会开启并开始处理主题和正文中的所有内容，附件可通过路径访问。晚餐时发现一个 bug？用手机发邮件过去，在你回到屏幕之前，一个会话已经在运行了。我把整个东西开源了：[github.com/mvanhorn/agentmail-to-claude-code](https://github.com/mvanhorn/agentmail-to-claude-code)。

三个部分：

一个守护进程，通过 WebSocket 监听一个 AgentMail 收件箱。每封在允许列表中的邮件到来时，它会开启一个新的 Claude 会话，把邮件写到一个提示文件里，然后告诉 Claude 阅读并执行它。

两个终端后端，cmux 或独立 Ghostty，所以它能驱动你本来就会启动的任何终端。

一个发送器。我把它接入了我的 [Hermes](https://hermes-agent.nousresearch.com/) 里的一个 `cc` 命令，所以从我的手机上我运行 `cc <task>`，它就在我的 Mac 上作为一个工作会话落地，不需要 VPN，不需要 SSH。

允许列表就是门禁。只有你控制的地址才能进入，任何 DKIM 或 SPF 验证失败的邮件在会话开启前就被丢弃了。

### 技巧

始终开启远程控制：在 `~/.claude/settings.json` 中添加 `"remoteControlAtStartup": true`。

给 Claude 一个邮件地址。粘贴给你的智能体："使用 github.com/mvanhorn/agentmail-to-claude-code 给 Claude Code 一个电子邮件地址。克隆它，设置一个 AgentMail 收件箱，将我的 API key、收件箱、仅包含我自己地址的允许列表以及我的终端（cmux 或 Ghostty）填入 `cc.env`，然后运行守护进程并将其安装为 launchd 任务。当我向那个收件箱发送邮件时，一台新的 Claude Code 会话应该在这台 Mac 上打开并开始处理主题和正文。"

## 8. 危险地跳过权限，是的我是认真的

Claude Code 每次编辑和命令都要征求许可。有六个会话同时开着，你不可能守着它。两个设置让它变得可接受。人们说 auto 是做这个的"更安全"的方式，但它太慢我受不了。

`skipDangerousModePermissionPrompt: true` 是关键。没有它的话，Claude 每个会话都要让你确认。你也可以按 Shift+Tab 来切换。人们告诉我新的 "auto" 模式在安全更多的情况下能做到大部分效果。也许吧。我说 YOLO。这是我的电脑。如果我搞砸或毁了一切，还有 GitHub。当我为一个朋友的 Claude Code 设置时，AI 还积极地试图劝他不要启用这个。你得对它态度强硬。

另一个设置是一个声音钩子，有六个会话时这个没得商量。

走开，听到声音时回来。有六个会话在跑，声音是让你知道哪一个刚完成了的方式。

### 技巧

粘贴到 `~/.claude/settings.json`：

```json
{ "permissions": { "allow": [ "WebSearch", "WebFetch", "Bash", "Read", "Write", "Edit", "Glob", "Grep", "Task", "TodoWrite" ], "deny": [], "defaultMode": "bypassPermissions" }, "skipDangerousModePermissionPrompt": true }
```

```json
{ "hooks": { "Stop": [ { "hooks": [ { "type": "command", "command": "afplay /System/Library/Sounds/Blow.aiff" } ] } ] } }
```

Codex 也有同样的 YOLO 模式。在 `~/.codex/config.toml`：

```toml
approval_policy = "never"
sandbox_mode = "danger-full-access"
```

或者用 `codex --yolo` 启动一个一次性会话。

## 9. 我如何在不打开 Codex CLI 的情况下通过 Codex 运行大部分代码

我整天都在向 Codex 派发工作，而我几乎从不打开 Codex CLI 来做这件事。Claude 做规划，Codex 做构建，我从不离开我的 Claude 会话。

三种无需离开 Claude 就把工作交给 Codex 的方式：

[Codex IDE 扩展](https://developers.openai.com/codex/ide)：发送任务，应用结果，从不进入 Codex 终端。

`/ce-work --codex`：在 Compound Engineering 循环内直接把构建委托给 Codex。

Printing Press Codex 模式：在打印一个新 CLI 时把 `codex` 放在提示的末尾，它把构建交给 Codex。

我的设置，两个引擎都调到了超高推理：

Codex：reasoning xhigh，fast mode 开启，始终如此。

Claude Code：reasoning xhigh，fast mode 关闭。它的 fast mode 在你的 $200 Max 订阅之上按 token 计费，所以我跳过它。

并排运行两个 $200 订阅，就是一整个第二引擎。我把大型并行构建推给 Codex，让 Claude 专注于规划和品味。有些朋友反过来用，Codex 构建，Claude 审核。

### 技巧

Codex：reasoning xhigh，fast mode 开启。Claude Code：xhigh，fast mode 关闭。

把工作交给 Codex：Codex IDE 扩展、`/ce-work --codex`，或者在 Printing Press 提示的末尾加上 `codex`。

## 10. 规划前先研究：last30days

在我 `/ce-plan` 之前，我通常会先对它运行 `/last30days`。

我在 Vercel 的 agent-browser 和 Playwright 之间做选择。我没有读文档，而是运行了 `/last30days Vercel agent browser vs Playwright`。几分钟内：几十条 Reddit 帖子、X 帖子、YouTube 视频、HN 故事。agent-browser 每次调用使用的上下文少得多，Playwright 仅工具定义就要耗费数千 token。我把整个输出输入 `/ce-plan integrate agent-browser`。这个计划是基于社区当下真正了解的东西，而不是六个月前的训练数据。

last30days 是开源的，现在已经超过 26K 星。它并行搜索 Reddit、X、YouTube、TikTok、Instagram、HN、Polymarket、GitHub 和网页。我选库之前跑它，构建功能之前跑它，见商业伙伴之前跑它，写文章之前跑它。这篇文章里的好几样东西我都跑过它。研究、规划、构建。这才是真正的循环。

### 技巧

安装 last30days。在 `/ce-plan` 之前，运行 `/last30days <topic>`。

确保你安装了 ScrapeCreators key

## 11. Granola 一切，把原始转录放进你的 LLM

我和一个候选人吃了午饭。我们聊了产品、食物和孩子，九十分钟正常对话中交织着一个产品想法。Granola 在运行。之后，我把完整的原始转录粘贴进 Claude Code：`/ce-plan turn this into a product proposal`。

诀窍是原始。我不先总结。我把整个乱糟糟的转录丢进去，关于寿司的跑题和其他所有内容，让 Claude 对照我实际的代码库和我写过的每一个之前的策略计划来做提取。Granola 上下文加上代码库加上之前的计划等于黄金。它一次性产出了一份提案，忽略了餐厅闲聊，我当晚就发出了。这家伙现在全职跟我们一起工作了。

以及自三月以来的升级：Printing Press [Granola CLI](https://github.com/mvanhorn/printing-press-library/tree/main/cli-skills/pp-granola)。它是魔法。我把任何会议当作干净的结构化数据直接拉到会话中，搜遍我所有参加过的每一次会议，找到某个人三周前说的一句话，然后把它管道输送到计划里。不再需要复制粘贴。每次会议的上下文都只需一个命令。

### 技巧

把原始的 Granola 转录丢进 `/ce-plan`，不要先总结。安装 Printing Press Granola CLI。

## 12. 人类信号

这是我花了最长时间才完成的心态转变。当你运行六个智能体时，你的工作不是亲手做那些事。你的工作是成为信号。

智能体提供体量。你提供品味、方向，以及响应和重新定向的循环。你看着返回的结果，你说"第二个选项更接近，但用第一个选项的措辞"，"解决最大的风险"，"这一段太长了"，它们就开始动。这个循环中稀有而有价值的东西是你的判断力，不是你的打字。我越投入到作为人类信号的角色，停止试图同时成为做执行的那双手，我交付的就越多。

成为品味。让它们成为双手。

### 技巧

用你的大脑指挥你的智能体来为世界增添价值。你的大脑仍然有价值。

## 13. [HyperFrames](https://hyperframes.heygen.com/) 做视频，做一切

视频曾经是我不做或外包的东西。现在我制作它的方式和我制作其他所有东西一样：我说，智能体构建，我响应。

HyperFrames 让我能以 HTML 构建视频，所以智能体可以编写它。这个循环和写代码完全一样，输出只是 MP4 而不是 PR。每个视频是一个包含 script.md 的文件夹，逐场景、动态排版、承载每一个节拍的字幕。智能体把那个脚本转化为合成并渲染出来。没有编辑器，没有时间线。

我用这种方式制作的发布预告片：

Granola CLI 演示

Agent Cookie 发布

Agent Cookie 在 HyperFrame 中制作的发布视频

视频的制作成本降到了一段对话，所以任何值得有视频的东西现在都有一个：发布预告片、产品演示、动画说明、带字幕的短片。它们也不光发在 X 上：我会把一个渲染好的演示直接放到 PR 里，就像 [atlas-lean 上的这个](https://github.com/facebookresearch/atlas-lean/pull/2)，这是 Facebook 的 AI 研究项目。

### 技巧

在 HyperFrames 里构建视频：写一个 script.md，让你的智能体把它渲染成 MP4。

把 GIF 上传到 [catbox](https://catbox.moe/)，它们在 GitHub、PR、README 和 issue 里渲染得很漂亮。

## 14. 你的笔记就是你的智能体的知识库

三月的策略文件夹技巧已被泛化了。计划每次变得更好的原因在于 Claude 可以访问我之前写过的每一个计划。复利上下文。所以我让它指向了我的整个大脑。

我让它指向的工具：

[Bear](https://bear.app/)，搭配 Bear CLI。十年的笔记、会议、半成品想法和决策，智能体可以读和写。个人 RAG，只是不叫这个名字。我放进去的越多，每个会话就越聪明。

[Obsidian](https://obsidian.md/)。我不使用它，但人们很喜欢它做这个，并且它的插件生态很深。

[gbrain](https://github.com/garrytan/gbrain)。我跨机器和智能体同步的大脑。

[supermemory](https://supermemory.ai/)。一个很多人信赖的智能体记忆层。正在深入研究中，评价待定。

这个技巧的形态才是重点：选一个带 CLI 或 API 的笔记工具，让你的智能体指向它，让你的知识复利累积。

### 技巧

让你的智能体同时指向：你用来写的笔记工具（Bear、Obsidian）和帮你记住的智能体大脑（gbrain、supermemory）。选那些有 CLI 或 API 的工具，这样智能体能读它们。

## 15. 随处工作——我的 Mac mini

### 技巧

[Mosh](https://mosh.org/)，当你必须 SSH 的时候。它让会话在糟糕的 wifi 和漫游下仍然感觉像本地一样响应灵敏。在普通 SSH 上，Claude Code 非常卡，每次按键都在等待往返。这是远程机器上可用和痛苦之间的区别。

[Tmux](https://github.com/tmux/tmux)，用于飞机上。在 tmux 会话内 SSH 到远程机器，工作在那里运行，而不是在你的笔记本上。飞越大西洋时 wifi 断了二十分钟，你重新连接，attach，它就在你离开时的位置。我在从欧洲飞回来的整趟航班上交付过功能。

Hermes 和 [OpenClaw](https://github.com/openclaw/openclaw)，两者都运行，用于自主远程工作。Hermes 用于自学习生态系统，在重复任务上越来越强；OpenClaw 用于智能体构建技能的广度。我在两者之间切换。如果你早期放弃了 OpenClaw，把它清掉重新开始。

Agent Cookie 用于在你的 Mac mini 和主力 Mac 之间同步 cookies 和 .env 文件。

## 16. [Proof](https://proofeditor.ai/)：用于把计划发给同事

plan.md 对我完美，但对一个不活在终端里的人来说毫无用处。这是最后一个真正的缺口，而 Proof，同样来自 Every，弥补了它。

在 Proof 里打开一个计划像读文档一样阅读，这很好。但它变得必不可少的地方是把计划发给同事。我把一个 plan.md 或规格丢进 Proof，发送链接，一个不用终端的普通人就可以清晰地阅读它，进行行内评论，那些评论会回流到与智能体的循环中。再也不用把 markdown 粘贴到 Slack 里看着它渲染成一团垃圾。这是对整个计划文件工作流的人类在环审核，也是第一次与普通同事分享智能体工作时没有令人不适的感觉。

我在写这篇文章时把它加载到了 Proof 里。它就是被这样审阅的。

而且我是在 cmux 中写了这整篇文章，Proof 审阅就在旁边开着：

cmux 和 Proof 一起工作

### 技巧

分享一个计划：把 .md 丢进 Proof，发送链接，把评论拉回循环中。

## 17. 编写你自己的技能

最大的升级不是使用智能体。而是教它们能够持久生效的技巧。任何我做了超过两次的事情，我都把它变成一个技能：一个我的智能体能永久运行的可复用命令。通过先编写你自己的技能来自动化你的工作流。

你不是从零开始写它们。为我解锁这个的诀窍是，让你的智能体指向一个已经有效的技能，让它复制其形态。字面上就是："看 Compound Engineering 技能，帮我做一个像这样的用于 [我想自动化的任何事]。它读到一个优秀的范例，学习其结构，然后为我搭建我的那个。我靠这种方式构建了一大堆技能。

这现在也成了我开源生活的大部分。如果你看[我的 GitHub](https://github.com/mvanhorn)，里面的工作就是技能和围绕它们的工具。last30days 最开始只是我想要给自己的一个技能，现在已经是超过 26K 星的开源项目。Printing Press 是一整个用于生成智能体原生 CLI 的工厂，并且是我最常使用的个人工具，已经有超过 320 个合并 PR 进入了它。我是 Compound Engineering 本身的顶级贡献者之一。这些没有一个是大计划。每一块都是一个我运行得足够频繁的工作流，以至于值得让智能体永久精通它。

写一次技能。之后的每个会话都更快。这就是 Compound Engineering 的复利部分。

### 技巧

任何你做超过两次的事，做一个技能："看 Compound Engineering 技能，帮我做一个像这样的用于 [X]。"

## 18. 开源：为你热爱的项目做贡献

这个交付我自己项目的循环也在交付其他人的项目。我有数以百计的 PR 被合并到开源项目中，包括 Python、Go、[OpenCV](https://github.com/opencv/opencv)、[Vercel 的 Agent Browser](https://github.com/vercel-labs/agent-browser) 以及 OpenClaw。不是路过的拼写修正，而是在我每天使用的工具上的真正功能。

不知不觉中，我开始登上了贡献者列表的前列：

#3 在 Compound Engineering、[Superpowers](https://github.com/obra/superpowers) 和 [Emdash](https://github.com/emdash-cms/emdash)

#4 在 GStack 和 Paperclip

#6 在 Vercel 的 Agent Browser

#2 在 [Camoufox](https://github.com/jo-inc/camofox-browser)

[@pejmanjohn](https://x.com/@pejmanjohn) 开玩笑说，当他打开一个仓库，在贡献者网格中寻找我的脸已经成为他个人的"Where's Waldo"游戏。

Superpowers 的贡献者

但合并的 PR 不是真正的奖赏。是人。我跳进 Discord，认识维护者，交真正的朋友。这对招聘也非常有帮助，我刚通过这种方式为我的新公司雇了一名工程师。你为热爱的东西做贡献，你认识同样热爱它的人，这一切在复利。

### 技巧

选一个你每天用的工具，找到一个它真正缺失的功能，用同样的 `/ce-plan` + `/ce-work` 循环把它做出来。

活跃在项目的 Discord 里。PR 让你进门；人才是你为什么留下来的原因。

在 X 上增加价值

在 X 上花 $1-3/月订阅你尊敬的人。我花 $1/月订阅 [@garrytan](https://x.com/@garrytan)，当我提交 PR 时我可以给他发送一条 X 帖子，他会收到一个特别通知，显示我是付费用户。我也订阅了 [@jason](https://x.com/@jason) [@teknium](https://x.com/@teknium) [@Teknium](https://x.com/@Teknium)。

## 19. 我目前的笔记本电脑配置

我那台两年的笔记本在我运行的一切之下几乎不顶用了，整天六个 Claude 会话加上 Codex。所以我升级到了 M5 Max，64GB 内存。它是一头猛兽，我很爱它。但它仍然被工作负载打趴：我的全新机器电池续航最短只有一小时。

于是我恐慌性地买了电源。我现在到哪儿都带着一个 [Anker 电池砖](https://www.amazon.com/dp/B0BYP2F3SG)，我还在 Tesla 里放了一个 [Anker 充电器](https://www.amazon.com/dp/B0CZ7BL16W)，这样车在途中给我补电。

### 技巧

永不休眠：`sudo pmset -a disablesleep 1`。随身带一个 Anker 电池砖；在车里放一个[充电器](https://www.amazon.com/dp/B0CZ7BL16W)。

## 20. Printing Press：运行现实生活的 CLI

这些技巧大部分活在终端里。这个是唯一离开终端的。Printing Press 是一组包裹了现实世界服务的 CLI，让智能体可以直接完成差事。它现在已经是一个独立项目，在 [@ppressdev](https://x.com/@ppressdev)，超过 3.7K 星，我和 @trevin 一起构建它。

让它们真正可用的部分是认证，这个昨晚刚发布：Agent Cookie。它把一个 CLI 交给你真实的浏览器会话，这样它就能以你的身份行动，不需要粘贴密码，不需要重新认证。这是把"一个知道这个服务的智能体"变成"一个已经登录进去的智能体"的关键。

一个真实的下午，从头到尾：

Tesla 预热。孩子们十分钟后上车："preheat the car to 72。"[Tesla CLI](https://github.com/mvanhorn/printing-press-library/tree/main/cli-skills/pp-tesla) 触发，我们走出去之前车已经暖好了。

[Instacart](https://github.com/mvanhorn/printing-press-library/tree/main/cli-skills/pp-instacart)。"add Corona to Costco on Instacart。"

[ESPN](https://github.com/mvanhorn/printing-press-library/tree/main/cli-skills/pp-espn) 轮询。一个会话帮我盯着一场比赛，只在比分接近时给我发提醒。我没有刷新任何东西，我收到了唯一那条重要的提醒。

[Alaska Airlines](https://github.com/mvanhorn/printing-press-library/tree/main/cli-skills/pp-alaska-airlines) 为孩子们的旅行。拉取票价和前后日期，检查我们的 Atmos 余额，输入 `/ce-plan`，得到了一个包含最便宜日期和购票提醒的预订策略。从足球场上搞定的。

不是"AI 写我的代码"。智能体工程干的是跑腿、看比赛、暖车、订旅行，而我在做别的事。

### 技巧

从 [printingpress.dev](https://printingpress.dev/) 的库里安装一个现成的 CLI，把差事直接交给你的智能体。

无痛认证：Agent Cookie 把你的真实浏览器会话传给 CLI，这样它就能以你的身份行动。

真正的技巧：打印你自己的。把一件你整天都在做的事、一个你生活其中的 API 或服务，让 Printing Press 生成一个智能体原生的 CLI。你为你自己的工作流构建的那个，就是改变你工作方式的那个。

## 21. 诚实部分：AI 精神病

智能体本应替我们做所有工作。相反，我认识的每个朋友都在比他们一生中任何时候都更努力地工作。

简单的回应是休息一下，摸草。但这不是这里的问题。这是关于成瘾。和智能体一起构建是有史以来最棒的电子游戏，这个循环就是这么好。

我有一些我真正担心的朋友。他们因为能构建任何东西而兴奋到不做任何别的事了。然后他们发布了，却没有用户。这也没关系。我发布过很多没有用户的东西。陷阱不在空发布，而是消失在构建里，失去了你身边的人。

所以要小心。跟你的至亲说话。问问自己是否真的有人想要你做的东西。如果诚实的答案是它只是给你自己用的工具，那也没关系。我构建过的一些最好的东西从来都只是给我自己的。

如果你确实想要观众，这就是 Gary Vaynerchuk 一直在内容上倡导的路径。你从某个地方开始，向虚空发帖，希望一个人注意到你。然后是三个，然后是十个，然后是一百个，你一步步走向数千人。没有人从数千人开始。你做任何东西都是一样的。

### 技巧

休息。摸草。

跟你的至亲说话。

构建人们想要的东西，即使"人们"只是你自己。

## 22. 这篇文章就是这样写成的

这是一个 markdown 文件。在 cmux 里的 Claude Code，我用 Monologue 说的话："改进不用 IDE 的开场白"，"让不要读计划的部分更有味道些"，"加上 Tesla 和 Instacart 的故事"。它重写，我回应，然后它进了 Proof 做审核。last30days 提供了新鲜素材。顺便说一下，这次没有 Zed。我不再用它了。不用 IDE。不打代码。语音、计划、构建。从桌子前、沙发上、车里、足球场上。

这就是我在六月份知道的一切。一个语音应用、一个计划文件插件、几个配置改动、一堆标签页、一台 Mac Mini、两台远程机器，以及一支运行现实生活的 CLI 舰队。

### 技巧

复制这整篇文章，粘贴到你的智能体里，告诉它设置一切它能设置的东西。好的事情会发生在你的智能体工程工作流上。

![Image](../../assets/mvanhorn-agentic-engineering-hacks-june-2026/image-2.jpg)

![Image](../../assets/mvanhorn-agentic-engineering-hacks-june-2026/image-3.jpg)

![Image](../../assets/mvanhorn-agentic-engineering-hacks-june-2026/image-4.png)

![Image](../../assets/mvanhorn-agentic-engineering-hacks-june-2026/image-5.jpg)
