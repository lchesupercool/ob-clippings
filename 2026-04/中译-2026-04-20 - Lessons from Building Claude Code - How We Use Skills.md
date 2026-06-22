---
---

# 构建 Claude Code 的经验：我们如何使用 Skills

- 作者：Thariq (@trq212)
- 来源：https://x.com/trq212/status/2033949937936085378
- 发布时间：2026-03-18 00:53
- 剪藏时间：2026-04-20

## 摘要

- Skill 是文件夹，不只是 markdown 文件。
- 高价值的 skill 类型：API reference、verification、data analysis、automation、scaffolding、review、deployment、runbooks、ops。
- 最有用的章节通常是 gotchas、progressive disclosure、scripts 以及以触发为导向的 description。
- 好的 skill 是从真实失败中演化出来的，如果要广泛分享，就应该做好策展（curation）。

## 正文

![](../../assets/x-clips/2026-04-20-claude-code-skills-01-HDl2jn9a0AAZkyz.jpg)

构建 Claude Code 的经验：我们如何使用 Skills。Skills 已经成为 Claude Code 中最常被使用的扩展点之一。它们灵活、易于制作、也便于分发。但这种灵活性也让人很难搞清楚什么做法最有效。什么类型的 skill 值得做？写好一个 skill 的秘诀是什么？什么时候该把它们分享给别人？我们在 Anthropic 内部大量使用 Claude Code 里的 skill，目前有数百个正在被活跃使用。以下是我们在用 skill 加速开发过程中总结出的经验。

### 什么是 Skills？

如果你刚接触 skill，我建议先去看看我们的文档，或者看一下我们最新发布在 Skilljar 上的 Agent Skills 课程，本文会默认你对 skill 已经有一定了解。我们经常听到的一个常见误解是，skill 只是"一堆 markdown 文件"，但 skill 最有意思的地方恰恰在于它们不只是文本文件。它们是文件夹，里面可以包含 scripts、assets、data 等等，agent 可以发现、探索并操作这些内容。在 Claude Code 里，skill 还有各种各样的配置项，包括注册动态 hook。我们发现，Claude Code 中一些最有意思的 skill，都会创造性地使用这些配置项和文件夹结构。

## Skills 的类型

在把我们所有的 skill 整理分类之后，我们发现它们会聚类到几个反复出现的类别里。最好的 skill 会干净利落地落在某一个类别里，比较混乱的 skill 则会横跨多个类别。这不是一份权威清单，但它是一个不错的思考框架，可以帮你看看你们组织内部还缺哪类 skill。

![](../../assets/x-clips/2026-04-20-claude-code-skills-02-HDlvMmubEAIzF-N.jpg)

### 1. Library 与 API Reference

这类 skill 讲解如何正确使用一个 library、CLI 或 SDK。它既可以针对内部 library，也可以针对 Claude Code 有时处理得不太好的通用 library。这类 skill 通常会包含一个用于参考的代码片段文件夹，以及一份 gotcha 列表，用来提醒 Claude 在写脚本时避免踩坑。示例：

- billing-lib —— 你们内部的 billing library：各种 edge case、footgun 等等。
- internal-platform-cli —— 你们内部 CLI wrapper 的每一个子命令，以及什么时候该使用它们的示例。
- frontend-design —— 让 Claude 更好地理解和使用你们的 design system。
### 2. 产品验证（Product Verification）

这类 skill 描述如何测试或验证你的代码是否正常工作。它们通常会搭配某个外部工具一起使用，比如 playwright、tmux 等等，用来执行验证。验证类 skill 对于确保 Claude 的产出正确性极其有用。花一个工程师一周时间，专门把你们的验证类 skill 打磨得非常好，是非常值得的。可以考虑一些技巧，比如让 Claude 把它的输出录成一段视频，这样你就能准确看到它都测了什么；或者在每一步都对状态做程序化的断言。这些通常通过在 skill 里包含各种脚本来实现。示例：

- signup-flow-driver —— 在 headless browser 里走一遍 signup → email verify → onboarding 的流程，并在每一步都带有用于断言状态的 hook。
- checkout-verifier —— 用 Stripe 的测试卡驱动 checkout 的 UI，验证 invoice 最终确实处于正确的状态。
- tmux-cli-driver —— 用于交互式 CLI 测试，适用于被验证对象需要一个 TTY 的场景。
### 3. 数据抓取与分析（Data Fetching & Analysis）

这类 skill 连接到你的数据和监控栈。它们可能会包含用于带凭据抓取数据的 library、特定的 dashboard id 等等，以及常见 workflow 或获取数据方式的说明。示例：

- funnel-query —— "我要 join 哪些事件才能看到 signup → activation → paid 这条漏斗"，以及那张真正存有规范 user_id 的表。
- cohort-compare —— 比较两个 cohort 的留存或转化，把具有统计显著性的差异标记出来，并链接到对应的 segment 定义。
- grafana —— datasource 的 UID、集群名字、从问题到 dashboard 的查表映射。
### 4. 业务流程与团队自动化（Business Process & Team Automation）

这类 skill 把重复性的 workflow 自动化成一条命令。它们的指令通常相当简单，但可能会对其它 skill 或 MCP 有比较复杂的依赖。对这类 skill 来说，把之前的执行结果保存在日志文件里，可以帮助模型保持一致性，并回顾过去这个 workflow 的执行情况。示例：

- standup-post —— 汇总你的 ticket 追踪系统、GitHub 活动以及之前的 Slack 消息，生成一份格式化的站会内容，只输出增量。
- create-<ticket-system>-ticket —— 强制执行 schema（合法的枚举值、必填字段），加上创建后的 workflow（通知 reviewer、在 Slack 里发链接）。
- weekly-recap —— 把已 merge 的 PR、已关闭的 ticket 以及部署记录汇总成一篇格式化的周报。
### 5. 代码脚手架与模板（Code Scaffolding & Templates）

这类 skill 为 codebase 中某个特定功能生成框架样板代码。你可能会把这类 skill 和一些可以组合的脚本搭配使用。当你的脚手架里带有无法完全用代码覆盖的自然语言要求时，它们特别有用。示例：

- new-<framework>-workflow —— 用你们的注解生成一个新的 service/workflow/handler。
- new-migration —— 你们的 migration 文件模板，加上常见的 gotcha。
- create-app —— 生成一个新的内部 app，auth、logging 和部署配置都预先接好。
### 6. 代码质量与 Review（Code Quality & Review）

这类 skill 在你们组织内部强制执行代码质量标准，并辅助 review 代码。它们可以包含确定性的脚本或工具，以获得最大程度的健壮性。你可能会希望把这类 skill 作为 hook 的一部分自动运行，或者放进 GitHub Action 里。

- adversarial-review —— 派生一个"视角全新"的 subagent 来挑刺，随后实施修复，并一直迭代到发现的问题只剩下吹毛求疵的小细节为止。
- code-style —— 强制执行代码风格，尤其是那些 Claude 默认做得不太好的风格。
- testing-practices —— 关于怎么写测试、该测什么的指南。
### 7. CI/CD 与部署（CI/CD & Deployment）

这类 skill 帮你在 codebase 内部拉代码、推代码以及部署代码。它们可能会引用其它 skill 来收集数据。示例：

- babysit-pr —— 盯着一个 PR → 重试 flaky CI → 解决 merge 冲突 → 开启自动合并（auto-merge）。
- deploy-<service> —— 构建 → 冒烟测试 → 渐进式流量灰度并对比错误率 → 出现回归时自动回滚。
- cherry-pick-prod —— 用独立的 worktree 做 cherry-pick → 解决冲突 → 用模板生成 PR。
### 8. Runbooks

这类 skill 接收一个症状（比如一条 Slack thread、一次告警或一段错误签名），走一遍多工具的调查流程，并产出一份结构化的报告。示例：

- <service>-debugging —— 为你们流量最高的服务建立"症状 → 工具 → 查询模式"的映射。
- oncall-runner —— 拉取告警 → 检查那些常见的嫌疑对象 → 把调查结果格式化输出。
- log-correlator —— 给定一个 request ID，从每一个可能经手过它的系统里拉出匹配的日志。
### 9. 基础设施运维（Infrastructure Operations）

这类 skill 执行日常维护和运维流程 —— 其中有些涉及破坏性操作，非常需要护栏（guardrails）来保护。它们让工程师更容易在关键操作中遵循最佳实践。示例：

- <resource>-orphans —— 找出被遗弃的 pod/volume → 发到 Slack → 经过一段"冷却期" → 用户确认后 → 做级联清理。
- dependency-management —— 你们组织内部的依赖审批 workflow。
- cost-investigation —— "为什么我们的存储 / 出网流量账单突然飙升"，带上具体的 bucket 和查询模式。
## 制作 Skills 的小建议

![](../../assets/x-clips/2026-04-20-claude-code-skills-03-HDoKg58bEAAL1bw.jpg)

一旦你决定了要做哪个 skill，该怎么写呢？以下是我们总结出来的一些最佳实践、小建议和小技巧。我们最近也发布了 Skill Creator，让在 Claude Code 里创建 skill 变得更容易。

### 不要说那些显而易见的东西

Claude Code 对你的 codebase 已经了解很多，Claude 对写代码本身也了解很多，包括许多默认的偏好。如果你要发布的 skill 主要是知识型的，那就试着把重点放在那些能把 Claude 推出它惯常思维方式的信息上。frontend-design 这个 skill 就是一个很好的例子 —— 它是 Anthropic 的一位工程师打造的，通过和客户反复迭代来改善 Claude 的设计审美，避免那些老套的模式，比如 Inter 字体和紫色渐变。

### 建一个 Gotchas 章节

![](../../assets/x-clips/2026-04-20-claude-code-skills-04-HDlwEG1bEAUdmcV.jpg)

任何一个 skill 里信噪比最高的内容就是 Gotchas 章节。这类章节应该是从 Claude 在使用你的 skill 时常见的失败点中沉淀下来的。理想情况下，你应该随着时间的推移持续更新你的 skill，把这些 gotcha 都记录进去。

### 善用文件系统与渐进式披露（Progressive Disclosure）

![](../../assets/x-clips/2026-04-20-claude-code-skills-05-HDlwhSjbEAIJSc9.jpg)

就像我们前面说过的，skill 是一个文件夹，不只是一个 markdown 文件。你应该把整个文件系统看作是一种 context engineering 和渐进式披露的形式。告诉 Claude 你的 skill 里有哪些文件，它就会在合适的时候去读它们。渐进式披露最简单的形式就是指向其它 markdown 文件让 Claude 去用。举个例子，你可能会把详细的函数签名和使用示例拆到 references/api.md 里。再举一个例子：如果你最终的产出是一个 markdown 文件，你可以在 assets/ 里放一个模板文件，让它直接拷贝过来用。你可以有若干文件夹，分别放 reference、script、example 等等，它们能让 Claude 更高效地工作。

### 避免把 Claude 卡死在一条轨道上（Avoid Railroading）

Claude 通常会努力遵守你的指令，而且因为 skill 具有高度可复用性，你需要小心别把指令写得太具体。给 Claude 需要的信息，但也要给它根据具体情况做调整的余地。举个例子：

![](../../assets/x-clips/2026-04-20-claude-code-skills-06-HDlwurvbEAM5ZNu.jpg)

### 把"初始设置"这一步想清楚

![](../../assets/x-clips/2026-04-20-claude-code-skills-07-HDlw1mYbEAY-Bul.jpg)

有些 skill 可能需要用用户提供的上下文来做初始设置。举个例子，如果你在做一个把站会发到 Slack 的 skill，你可能会希望 Claude 问一下要发到哪个 Slack channel。一个不错的做法是把这些初始设置信息存到 skill 目录下的一个 config.json 文件里，就像上面的例子那样。如果 config 还没被设置，agent 就可以去询问用户相关信息。如果你希望 agent 以结构化的方式呈现多选题，你可以在指令里让 Claude 使用 AskUserQuestion 这个 tool。

### Description 字段是写给模型看的

当 Claude Code 启动一个 session 时，它会把所有可用的 skill 连同它们的 description 列成一份清单。Claude 会扫描这份清单来判断"这个请求有没有对应的 skill？"这意味着 description 字段不是摘要 —— 它是一份关于"在什么情况下应该触发这个 PR"的描述。

![](../../assets/x-clips/2026-04-20-claude-code-skills-08-HDlw5ULbEAQOqtJ.jpg)

### 记忆与数据存储

![](../../assets/x-clips/2026-04-20-claude-code-skills-09-HDoImh1bEAU-mMI.jpg)

有些 skill 可以通过在内部存储数据的方式带有某种形式的记忆。你可以把数据存到任何东西里，简单到一个只追加（append-only）的文本日志文件或 JSON 文件，复杂到一个 SQLite 数据库都行。举个例子，一个 standup-post 的 skill 可能会维护一个 standups.log，里面记录它写过的每一条站会内容，这样下次运行时，Claude 读一下自己的历史，就能告诉你从昨天到今天发生了哪些变化。存放在 skill 目录里的数据，在你升级 skill 时可能会被删掉，所以你应该把它存在一个稳定的文件夹里；截至今天，我们为每个 plugin 提供了 `${ CLAUDE_PLUGIN_DATA }` 作为稳定的数据存储目录。

### 存放脚本与生成代码

你能给 Claude 最强大的工具之一就是代码。把 scripts 和 library 交给 Claude，可以让它把每一轮（turn）的算力花在"组合、决定下一步做什么"上，而不是去重复造样板代码。举个例子，在你的数据科学 skill 里，你可能有一个从 event 数据源抓取数据的函数 library。为了让 Claude 做复杂的分析，你可以给它一组这样的 helper 函数：

![](../../assets/x-clips/2026-04-20-claude-code-skills-10-HDlxbtkbkAAOse7.jpg)

然后 Claude 就可以在运行时即时生成脚本，把这些功能组合起来，做更高级的分析，比如回应"周二发生了什么？"这类 prompt。

![](../../assets/x-clips/2026-04-20-claude-code-skills-11-HDlxfEIb0AA2E7l.jpg)

### 按需 Hook（On Demand Hooks）

skill 里可以包含一些 hook，这些 hook 只有在该 skill 被调用时才会激活，并在当前 session 的整个生命周期里持续有效。对那些你不想一直开着、但在某些场景下又非常有用的"强意见"式 hook，这种做法特别合适。举个例子：

- /careful —— 通过 Bash 上的 PreToolUse matcher 拦截 rm -rf、DROP TABLE、force-push、kubectl delete。你只在知道自己正在动生产的时候才想开它 —— 一直开着会把人逼疯。
- /freeze —— 拦截任何不在指定目录下的 Edit/Write。有用。
- 调试的时候："我想加点 log，但总是一不小心就把不相关的地方『修』掉……"
## 分发 Skills（Distributing Skills）

skill 最大的好处之一就是你可以把它们分享给团队其他人。分享 skill 给别人通常有两种方式：

- 把你的 skill 提交到你的 repo 里（放在 ./.claude/skills 下）。
- 做成一个 plugin，并搭建一个 Claude Code Plugin marketplace，让用户可以在上面上传和安装 plugin（详见这里的文档）。
对于只在少量 repo 里工作的小团队，直接把 skill check 到 repo 里效果很好。但是每一个被 check 进去的 skill 也会给模型的 context 增加一点点负担。当规模扩大时，一个内部的 plugin marketplace 可以让你分发 skill，并让团队自己决定要安装哪些。

### 管理一个 Marketplace

你怎么决定哪些 skill 可以进 marketplace？大家又是怎么提交 skill 的？我们没有一个集中式的团队来做决定；相反，我们尝试让最有用的 skill 自然地浮现出来。如果你有一个希望别人试用的 skill，你可以把它上传到 GitHub 上的一个 sandbox 文件夹，然后在 Slack 或其它论坛上把链接发给大家。一旦某个 skill 积累了足够的势头（这由 skill 的 owner 自己判断），他们就可以提一个 PR 把它移到 marketplace 里。这里要提醒一句，创建糟糕或冗余的 skill 是相当容易的，所以在发布前确保有某种策展机制非常重要。

### 组合 Skills

你可能会希望有一些相互依赖的 skill。举个例子，你可能有一个用来上传文件的 file upload skill，和一个生成 CSV 并把它上传的 CSV generation skill。这种依赖管理目前还没有原生地内建到 marketplace 或 skill 里，但你可以直接按名字引用其它 skill，只要它们已经安装了，模型就会去调用它们。

### 度量 Skills

为了了解一个 skill 的表现如何，我们使用一个 PreToolUse hook，让我们可以在公司内部记录 skill 的使用情况（示例代码见这里）。这样我们就能发现哪些 skill 很受欢迎，或者哪些 skill 相比我们的预期触发得过少。

## 结语

对 agent 来说，skill 是极其强大、灵活的工具，但现在还处于早期，我们都还在摸索怎么用好它们。与其把本文当成一份权威指南，不如把它看成我们见到行得通的一堆实用小贴士的大杂烩。理解 skill 最好的方式就是动手开始、做实验、看看什么对你有用。我们大多数的 skill 一开始都只是几行字加一条 gotcha，后来之所以越来越好，是因为每当 Claude 踩到新的 edge case，大家就会不断往里面补充内容。希望本文对你有帮助，如果你有任何问题，告诉我一声。
