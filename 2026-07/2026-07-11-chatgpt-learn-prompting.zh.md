---
title: 提示词编写（Prompting）
source: https://learn.chatgpt.com/docs/prompting
author: OpenAI
saved: '2026-07-11'
published: '2026-07-11'
tags:
- clipping
- openai
- chatgpt
- 提示词
- codex
- ai-coding
---


# 提示词编写（Prompting）

## 提示概述

提示是您告诉 ChatGPT 您想知道、想做什么或想改变什么的方式。提示可以是问题、指示或目标。您不需要技术语法或严格的公式。用你自己的话开始，查看回复，并使用后续消息来塑造结果。

简短的提示通常就足够了。对于更大或更重要的任务，请包括重要的部分：

- **目标：** ChatGPT 应该做什么？
- **背景：** 哪些信息或来源会有帮助？
- **输出：** 您需要什么格式、长度或详细程度？
- **边界：** 什么必须保持不变？ ChatGPT 在采取行动之前应避免或向您核实哪些内容？

仅使用有帮助的部分。您无需填写每一项或遵循所需的格式。

## 描述你需要的结果

从结果开始，而不是详细的步骤列表。当这些细节改变 ChatGPT 应生成的内容时，请包括受众或格式。

```
Turn these meeting notes into a short update for the project team.
Put the decisions and next steps first.
```

此提示解释了要创建的内容以及谁将阅读它。当流程本身很重要时，描述流程。否则，请给 ChatGPT 留出空间来搜索、比较信息并调整其方法。

## 添加有用的上下文

分享可能改变结果的信息。仅添加重要的来源，并解释 ChatGPT 应从每个来源中获取什么。

- 当您希望 ChatGPT 进行总结、比较、转换或[创建文件以供审阅](/codex/artifacts-viewer) 时，请附加文档、电子表格、演示文稿或 PDF 文件。
- 当任务依赖于视觉上下文时，添加屏幕截图、图表或其他[图像输入](/codex/image-inputs)。指出重要的区域，而不是仅仅依赖图像。
- 当答案取决于当前信息时，请ChatGPT使用[网络搜索](/codex/web-search)，并在需要检查结果时询问来源。
- 当相关聊天或任务应共享文件、源或本地文件夹时，使用[项目](/codex/projects)。

### 使用连接的源

当 ChatGPT 可以访问连接的源时，请指定它应该查找的位置以及应该查找的内容。您不需要描述它应该运行的每个搜索。

```
Use the latest project plan in Drive and relevant decisions and updates from
the project's Slack channel to prepare a status update.
```

连接的源需要匹配的插件，其可用性可能取决于您的计划和工作区设置。

### 使用插件

插件为 ChatGPT 提供可重复使用的指令以及与 Google Drive、Gmail、Slack 和 GitHub 等工具的连接。询问您需要的结果，然后让 ChatGPT 从可用的工具中进行选择。要选择特定插件，请在编辑器中输入`@`。

[了解插件在ChatGPT中查找、安装和使用插件。](/codex/plugins)

### 个性化 ChatGPT

将适用于聊天和任务的首选项放入 **设置 > 个性化** 作为自定义说明。在提示中保留仅与当前任务相关的详细信息。

[查看个性化设置 设置默认个性、自定义说明和其他应用程序首选项。](/codex/app/settings#personalization)

## 设置界限以防止出现实际问题

边界是 ChatGPT 需要的几条指令，以避免创建额外的工作或采取您不希望的操作。当更改错误的细节会导致结果无法使用时，或者当您想在某些内容影响其他人之前对其进行检查时，请添加一项。

- 保持批准日期和预算数字不变。
- 仅使用提供的来源。标记缺失的信息而不是猜测。
- 将建议保持在规定的预算范围内。
- 将消息准备为草稿。不要发送它。

专注于最重要的一两个边界。您无需控制 ChatGPT 执行的每一步。

## 使结果可供使用

告诉 ChatGPT 您打算如何使用结果。这有助于它选择正确的长度、详细程度和组织。

- 将此作为一页摘要，供董事在会议前浏览。把决定和后续步骤放在第一位。
- 将这些笔记变成包含决定、所有者和截止日期的后续电子邮件。
- 创建一个清晰的计划支出与实际支出表格，并突出显示任何超过 10% 的差异。

对于重要工作，请要求 ChatGPT 进行最终检查，例如确认每个操作项目都有所有者和截止日期或标记其无法验证的信息。然后在使用或分享之前亲自查看结果。

## 通过后续消息改进结果

您的第一个提示不必是完美的。查看结果，然后询问您想要的具体更改。

```
Make the opening more direct, keep the evidence, and move the recommendation
above the background section.
```

您可以添加缺失的源、更正方向、请求其他选项或更改详细程度，而无需重新开始。

### 引导和排队

当 Codex 已经开始工作时，您可以发送另一条消息，而无需等待当前运行完成：

- **Steer** 将消息添加到当前运行。用它来改变方向、添加缺失的细节或共享新信息。
- **队列** 保存下一次运行的消息。将其用于后续工作，应等到当前工作完成。

在 ChatGPT 桌面应用程序中，选择 [**设置 > 常规 > 后续行为**](/codex/app/settings#general) 下的默认值。排队的消息显示在编辑器上方，您可以在其中编辑、重新排序、发送或删除它们。该设置还显示了在不更改默认值的情况下对一条消息使用其他行为的快捷方式。

在 Codex CLI 中，当 Codex 正在执行时按 `Enter` 可引导当前轮次，或按 `Tab` 将消息排队到下一轮次。详情请参见[交互快捷键](/codex/developer-commands?surface=cli#cli-interactive-shortcuts)。

## 将各个部分组合在一起

对于使用连接源的项目更新，完整的提示可能如下所示：

```
Prepare a one-page project status update for Monday's leadership meeting. Use
the latest project plan in Drive and relevant decisions and updates from the
project's Slack channel.

Lead with the decisions leadership needs to make and the next steps. Summarize
progress, risks, owners, and due dates. Keep approved dates and budget figures
unchanged. Flag any conflicting or missing information, and don't send or
publish anything.

Before you finish, check that every next step has an owner and due date.
```

此提示涵盖**目标**、**上下文**、**输出**和**边界**，然后要求进行最终检查，但不详细说明每个步骤。

## 使用语音听写

在 ChatGPT 桌面应用程序中，在作曲家可见时按住 `Ctrl`+`M`，然后开始讲话。 ChatGPT 将您的语音转录为作曲家，以便您可以在发送提示之前查看和编辑它。

![编辑器中的语音听写指示器，带有转录提示](../../assets/chatgpt-learn-prompting/voice-dictation-light.webp)

## 聊天提示示例

使用聊天来提出问题、想法、草稿和日常决策。从您想要的结果开始，然后仅在改变答案时添加详细信息。

### 理解一个主题

```
Explain how compound interest works for someone who has never invested.
Use one concrete example and define any financial terms you introduce.
```

### 起草和完善写作

```
Draft a friendly email declining this invitation because I will be traveling.
Keep it under 120 words and leave the door open for a future event.
```

### 比较选项

```
Compare these two phone plans for one person who travels internationally twice
a year. Show the important differences in a table, then recommend one and explain
the tradeoff.
```

### 制定一个切实可行的计划

```
Plan five weekday dinners that take less than 30 minutes. Avoid peanuts, reuse
ingredients across meals, and finish with one consolidated shopping list.
```

## 提示工作

使用聊天功能进行快速提问、简短重写、头脑风暴和轻量级草稿。将“工作”用于利用不同来源或工具、涉及一系列步骤、进行更改或产生更大可交付成果的任务。

对于工作任务，描述您需要的结果，提供源材料，命名受众，并解释您将如何审查工作。要求 ChatGPT 进行规划、收集所需信息、创建文件并在完成之前进行检查。

### 高效使用工作

工作对于耗时或重复的任务或可重复使用的已完成文件很有用。如果使用更多学分的任务可以节省时间、提高质量或帮助您做出重要决定，那么它仍然是值得的。

从一个您可以查看的结果开始：

- 仅包含相关来源并在适当时限制日期范围。
- 定义受众、输出格式和所需长度。
- 将所需的工作与可选的改进或完善分开。
- 当方法很重要时询问计划。在 ChatGPT 发送、发布或更改其他人依赖的信息之前需要您的批准。
- 如果任务开始执行您不再需要的工作，则缩小或停止该任务。

查看第一个结果，完善说明，并在工作流程有效时重用该工作流程。

### 将源材料转化为成品文件

```
Use the attached quarterly reports to create a leadership brief and a six-slide
presentation.

The audience is the executive team. Lead with the three decisions they need to
make, distinguish reported facts from your analysis, cite each number to its
source file, and check that the brief and slides agree before you finish.
```

### 研究决定

```
Research three customer-support platforms for a 50-person company. Compare
pricing, security, integrations, and migration effort using current sources.
Deliver a recommendation memo with links, assumptions, and the questions we
should answer before signing a contract.
```

### 协调发布

```
Create a launch plan for the attached product brief. Include the timeline,
owners, dependencies, risks, announcement draft, customer FAQ, and a checklist
for launch day. Flag any missing decisions before producing the final files.
```

对于重复性工作，首先细化正常任务中的提示。输出可靠后，[安排该任务的工作](/codex/automations#schedule-work-from-a-task)。当每个计划运行应该启动一个新任务时，创建一个独立的计划任务。

## 提示法典

当您希望 ChatGPT 使用代码、代码库或开发人员工具时，请使用 Codex。有用的 Codex 提示会命名您想要的行为，指出相关的代码或复制步骤，保留重要的约束，并说明如何验证更改。

对于多步骤任务，当您希望 Codex 在编辑之前进行调查并提出方法时，请在应用程序编辑器中输入`/plan`。当[目标模式](/codex/long-running-work)可用时，在计划后使用`/goal`设定持久目标。有关当前命令列表，请参阅[应用程序斜杠命令](/codex/reference/slash-commands)。

### 如何阅读这些示例

每个工作流程包括：

- **何时使用**以及哪种 Codex 界面最适合（IDE、CLI 或云）。
- **步骤**以及示例用户提示。
- **上下文注释**：Codex 自动看到的内容与您应该附加的内容。
- **验证**：如何检查输出。

> **注意：** IDE 扩展会自动将您打开的文件作为上下文包含在内。在 CLI 中，明确提及路径，或使用 `/mention` 和 `@` 路径自动完成功能附加文件。

Codex 在限制文件和网络访问的[沙箱](/codex/sandboxing) 内运行本地命令。如果任务需要跨越该边界，Codex 在继续之前会遵循您的批准政策。

### 解释一下代码库

当您加入、继承服务或尝试推理协议、数据模型或请求流时，请使用此选项。

#### IDE 扩展工作流程（本地探索速度最快）

1. 打开最相关的文件。
2. 选择您关心的代码（可选但推荐）。
3. 提示法典：

   ```
   Explain how the request flows through the selected code.

   Include:
   - a short summary of the responsibilities of each module involved
   - what data is validated and where
   - one or two "gotchas" to watch for when changing this
   ```

确认：

- 索要可验证的图表或清单：

```
Summarize the request flow as a numbered list of steps. Then list the files involved.
```

#### CLI 工作流程（当您需要脚本 + shell 命令时很好）

1. 启动交互式会话：

   ```
   codex
   ```
2. 附加文件（可选）并提示：

   ```
   I need to understand the protocol used by this service. Read @foo.ts @schema.ts and explain the schema and request/response flow. Focus on required vs optional fields and backward compatibility rules.
   ```

上下文注释：

- 您可以在编辑器中使用`@`插入工作区中的文件路径，或使用`/mention`附加特定文件。

### 修复一个错误

当您有可以在本地重现的失败行为时，请使用此选项。

#### CLI 工作流程（带有复制和验证的紧密循环）

1. 在存储库根目录启动 Codex：

   ```
   codex
   ```
2. 向 Codex 提供复制配方，以及您怀疑的文件：

   ```
   Bug: Clicking "Save" on the settings screen sometimes shows "Saved" but doesn't persist the change.

   Repro:
   1) Start the app: npm run dev
   2) Go to /settings
   3) Toggle "Enable alerts"
   4) Click Save
   5) Refresh the page: the toggle resets

   Constraints:
   - Do not change the API shape.
   - Keep the fix minimal and add a regression test if feasible.

   Start by reproducing the bug locally, then propose a patch and run checks.
   ```

上下文注释：

- 由您提供：重现步骤和约束（这些比高级描述更重要）。
- 由 Codex 提供：命令输出、发现的调用站点以及它触发的任何堆栈跟踪。

确认：

- Codex 应在修复后重新运行重现步骤。
- 如果您有标准检查管道，请要求它运行它：

```
After the fix, run lint + the smallest relevant test suite. Report the commands and results.
```

#### IDE 扩展工作流程

1. 打开您认为错误所在的文件以及最近的调用者。
2. 提示法典：

   ```
   Find the bug causing "Saved" to show without persisting changes. After proposing the fix, tell me how to verify it in the UI.
   ```

### 写一个测试

当您想要定义要测试的确切范围时，请使用此选项。

#### IDE 扩展工作流程（基于选择）

1. 使用函数打开文件。
2. 选择定义函数的行。从命令面板中选择“添加到 Codex 线程”，将这些行添加到上下文中。
3. 提示法典：

   ```
   Write a unit test for this function. Follow conventions used in other tests.
   ```

上下文注释：

- 由“添加到 Codex 线程”命令提供：选定的行（这是“行号”范围），加上打开的文件。

#### CLI 工作流程（提示中描述的路径+行范围）

1. 启动法典：

   ```
   codex
   ```
2. 提示函数名称：

   ```
   Add a test for the invert_list function in @transform.ts. Cover the happy path plus edge cases.
   ```

### 截图中的原型

当您想要将设计模型、屏幕截图或 UI 参考转变为工作原型时，请使用此选项。

#### CLI 工作流程（图像 + 提示）

1. 将屏幕截图保存在本地（例如`./specs/ui.png`）。
2. 运行法典：

   ```
   codex
   ```
3. 将图像文件拖到终端中以将其附加到提示中。
4. 跟进约束和结构：

   ```
   Create a new dashboard based on this image.

   Constraints:
   - Use react, vite, and tailwind. Write the code in typescript.
   - Match spacing, typography, and layout as closely as possible.

   Outputs:
   - A new route/page that renders the UI
   - Any small components needed
   - README.md with instructions to run it locally
   ```

上下文注释：

- 图像提供了视觉要求，但您仍然需要指定实现约束（框架、路由、组件样式）。
- 包括图像未在文本中显示的行为，例如悬停状态、验证规则或键盘交互。

确认：

- 要求 Codex 运行开发服务器（如果允许）并告诉您确切的查找位置：

```
Start the dev server and tell me the local URL/route to view the prototype.
```

#### IDE 扩展工作流程（图像 + 现有文件）

1. 在 Codex 任务中附加图像（拖放或粘贴）。
2. 提示法典：

   ```
   Create a new settings page. Use the attached screenshot as the target UI.
   Follow design and visual patterns from other files in this project.
   ```

### 通过实时更新迭代 UI

当您希望在 Codex 编辑代码时有一个紧密的“设计 → 调整 → 刷新 → 调整”循环时，请使用此选项。

#### CLI 工作流程（运行 Vite，然后用小提示进行迭代）

1. 启动法典：

   ```
   codex
   ```
2. 在单独的终端窗口中启动开发服务器：

   ```
   npm run dev
   ```
3. 提示Codex进行更改：

   ```
   Propose 2-3 styling improvements for the landing page.
   ```
4. 选择一个方向并通过小的、具体的提示进行迭代：

   ```
   Go with option 2.

   Change only the header:
   - make the typography more editorial
   - increase whitespace
   - ensure it still looks good on mobile
   ```
5. 针对重点请求重复：

   ```
   Next iteration: reduce visual noise.
   Keep the layout, but simplify colors and remove any redundant borders.
   ```

确认：

- 当 Codex 更新代码时，在浏览器中查看更改。
- 提交您喜欢的更改并恢复您不喜欢的更改。
- 如果您恢复或更改编辑，请告诉 Codex，以便它在下一个提示下运行时不会覆盖您的编辑。

### 将重构委托给云

当您想要设计具有本地上下文的方法，然后将长时间的实现委托给可以并行运行的云任务时，请使用此方法。

#### 本地规划 (IDE)

1. 确保您当前的工作已提交或至少已隐藏，以便您可以清楚地比较更改。
2. 要求 Codex 制定重构计划。如果您有可用的 `$plan` 技能，请显式调用它：

   ```
   $plan

   We need to refactor the auth subsystem to:
   - split responsibilities (token parsing vs session loading vs permissions)
   - reduce circular imports
   - improve testability

   Constraints:
   - No user-visible behavior changes
   - Keep public APIs stable
   - Include a step-by-step migration plan
   ```
3. 审查计划并协商变更：

   ```
   Revise the plan to:
   - specify exactly which files move in each milestone
   - include a rollback strategy
   ```

上下文注释：

- 当 Codex 可以在本地扫描当前代码（入口点、模块边界、依赖图提示）时，规划效果最佳。

#### 云委托（IDE → 云）

1. 如果您还没有这样做，请设置 [Codex 云环境](/codex/environments/cloud-environment)。
2. 单击提示编辑器下方的云图标，然后选择您的云环境。
3. 当您输入下一个提示时，Codex 会在云中创建一个新任务，该任务继承现有任务上下文（包括计划和任何本地源更改）。

   ```
   Implement Milestone 1 from the plan.
   ```
4. 检查云差异，如果需要则进行迭代。
5. 直接从云端创建 PR 或在本地拉取更改以进行测试并完成。
6. 迭代计划的其他里程碑。

委托给云的任务在隔离的环境中运行。除非您为环境启用 Internet 访问，否则在代理阶段将关闭。了解更多关于[云上网](/codex/cloud/internet-access)的信息。

### 进行本地代码审查

当您在提交或创建 PR 之前需要第二双眼睛时，请使用此选项。

#### CLI 工作流程（检查您的工作树）

1. 启动法典：

   ```
   codex
   ```
2. 运行审核命令：

   ```
   /review
   ```
3.可选：提供自定义焦点说明：

   ```
   /review Focus on edge cases and security issues
   ```

确认：

- 根据审核反馈应用修复，然后重新运行`/review`以确认您解决了问题。

### 查看 GitHub 拉取请求

当您想要查看反馈而不在本地拉取分支时，请使用此选项。

在使用此功能之前，请在您的存储库上启用 Codex **代码审查**。请参阅[代码审查](/codex/third-party/github)。

#### GitHub 工作流程（评论驱动）

1. 在 GitHub 上打开拉取请求。
2. 发表评论，为 Codex 标记明确的重点领域：

   ```
   @codex review
   ```
3. 可选：提供更明确的说明。

   ```
   @codex review for security vulnerabilities and security concerns
   ```

### 更新文档

当您需要准确、清晰的文档更改时使用此功能。

#### IDE 或 CLI 工作流程（本地编辑 + 本地验证）

1. 识别要更改的文档文件并打开它们 (IDE) 或`@` 提及它们（IDE 或 CLI）。
2. 向法典提示范围和验证要求：

   ```
   Update the "advanced features" documentation to provide authentication troubleshooting guidance. Verify that all links are valid.
   ```
3. 法典起草变更后，审查文档并根据需要进行迭代。

确认：

- 阅读渲染的页面。
