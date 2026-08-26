#👍👍👍 

了解如何为 AI 编码 agent 优化 `AGENTS.md` 文件。掌握渐进式披露，让指令保持聚焦，并尽可能提升 agent 的表现。

Matt Pocock

你是否曾经担心过自己的 `AGENTS.md` 文件太大？

或许你确实应该担心。糟糕的 `AGENTS.md` 文件会让 agent 困惑，带来难以维护的问题，而且每次请求都会消耗你的 token。

所以，你最好知道该怎样修正它。

## 什么是 AGENTS.md？

`AGENTS.md` 是一个提交到 Git 的 Markdown 文件，用于自定义 AI 编码 agent 在仓库中的行为。它位于对话历史的顶部，紧接在[系统提示词](https://www.aihero.dev/ai-coding-dictionary/system-prompt)之后。

可以把它看作 agent 基础指令与你的实际代码库之间的一层配置。这个文件可以包含两类指导信息：

- **个人范围**：你偏好的 commit 风格和编码模式
- **项目范围**：项目的用途、使用的包管理器以及架构决策

`AGENTS.md` 是一项开放标准，许多工具都支持它，但并非所有工具都支持。

CLAUDE.md

需要注意的是，Claude Code 不使用 `AGENTS.md`，而是使用 `CLAUDE.md`。你可以在二者之间创建符号链接，让所有工具以相同方式工作：

```shellscript
# Create a symlink from AGENTS.md to CLAUDE.md
ln -s AGENTS.md CLAUDE.md
```

## 为什么巨大的 AGENTS.md 文件会带来问题

存在一种自然形成的反馈循环，会让 `AGENTS.md` 文件变得大得危险：

1. agent 做了你不喜欢的事情
2. 你添加一条规则来防止它再次发生
3. 几个月里重复数百次
4. 文件变成一个“泥球”

不同的开发者加入互相冲突的意见。没有人对全文做统一的风格审查。结果呢？文件成了无法维护的混乱内容，实际上还会损害 agent 的表现。

另一个罪魁祸首是自动生成的 `AGENTS.md` 文件。绝不要使用初始化脚本自动生成 `AGENTS.md`。这类脚本会用那些“在大多数场景中有用”、但其实更适合通过渐进式披露提供的内容塞满文件。生成的文件重视面面俱到，而不是有所节制。

### 指令预算

Humanlayer 的 Kyle 在一篇[文章](https://www.humanlayer.dev/blog/writing-a-good-claude-md)中提到了“指令预算”这一概念：

> 前沿推理 LLM ==能够以相当稳定的表现遵循大约 150 到 200 条指令==。小模型能关注的指令少于大模型，非推理模型能关注的指令少于推理模型。

无论是否相关，`AGENTS.md` 文件中的每个 token 都会在**每一次请求**中加载。这就产生了一个严格的预算问题：

| 场景 | 影响 |
| --- | --- |
| 小而聚焦的 `AGENTS.md` | 有更多 token 可用于与具体任务有关的指令 |
| 庞大臃肿的 `AGENTS.md` | 留给实际工作的 token 更少；agent 会感到困惑 |
| 无关的指令 | 浪费 token，加上分散 agent 的注意力，导致表现变差 |

综合来看，这意味着**理想的 `AGENTS.md` 文件应该尽可能小。**

### 过时的文档会污染上下文

大型 `AGENTS.md` 文件的另一个问题是内容容易过时。

文档很快就会过时。对人类开发者来说，过时的文档很烦人，但人通常有足够的固有记忆，会对有问题的文档保持怀疑。对于每次请求都会阅读文档的 AI agent，过时的信息会主动*污染*上下文。

记录文件系统结构时，这尤其危险。文件路径经常变化。如果你的 `AGENTS.md` 写着“身份验证逻辑位于 `src/auth/handlers.ts`”，而该文件后来被重命名或移动，agent 就会信心十足地去错误的位置查找。

与其记录结构，不如描述能力。可以提示相关内容*可能*位于何处，并说明项目的整体形态。让 agent 在规划期间按需生成自己的即时文档。

领域概念，比如“organization”“group”和“workspace”之间的区别，比文件路径更稳定，因此记录它们更安全。但在快速演进、由 AI 辅助开发的代码库中，即使这些概念也可能发生变化。应当有所节制。

## 精简大型 AGENTS.md 文件

要严格筛选写进这里的内容。可以把以下内容视为绝对的最低限度：

- **一句话项目描述**（作用类似基于角色的 prompt）
- **包管理器**（如果不是 npm；也可以使用 `corepack` 发出警告）
- **构建和类型检查命令**（如果不是标准命令）

说真的，就这些。其他所有内容都应该放到别处。

### 一句话项目描述

这一句话为 agent 提供上下文，让它明白自己*为什么*要在这个仓库中工作。它为 agent 做出的每项决策提供依据。

示例：

```markdown
This is a React component library for accessible data visualization.
```

这就是基础。agent 现在理解自己的工作范围了。

### 指定包管理器

如果你在 JavaScript 项目中使用 npm 以外的工具，要明确告诉 agent：

```markdown
This project uses pnpm workspaces.
```

如果没有这条说明，agent 可能会默认使用 `npm`，并生成错误的命令。

Corepack 也很好用。你还可以使用 [`corepack`](https://github.com/nodejs/corepack)，让系统自动处理警告，从而节省宝贵的指令预算。

### 使用渐进式披露

不要把所有内容都塞进 `AGENTS.md`，而应使用**渐进式披露**：只向 agent 提供它当下需要的信息，并在需要时指向其他资源。

agent 很擅长在文档层级中快速导航。它们对上下文的理解足以帮助它们找到所需内容。

#### 将特定语言的规则移到独立文件中

如果你的 `AGENTS.md` 目前写着：

```markdown
Always use const instead of let.
Never use var.
Use interface instead of type when possible.
Use strict null checks.
...
```

应当把这些内容移到独立文件中。在根目录的 `AGENTS.md` 中写：

```markdown
For TypeScript conventions, see docs/TYPESCRIPT.md
```

注意这种克制的写法：没有“always”，也没有用全大写来强制要求，只是以对话式语气给出一处引用。

这样做的好处是：

- 只有在 agent 编写 TypeScript 时才加载 TypeScript 规则
- 其他任务，例如调试 CSS、管理依赖，不会浪费 token
- 文件保持聚焦，并能适应模型变化

#### 嵌套使用渐进式披露

你还可以进一步深入。`docs/TYPESCRIPT.md` 可以引用 `docs/TESTING.md`。可以创建一棵便于发现的资源树：

```txt
docs/
├── TYPESCRIPT.md
│   └── references TESTING.md
├── TESTING.md
│   └── references specific test runners
└── BUILD.md
    └── references esbuild configuration
```

你甚至可以链接到外部资源，例如 Prisma 文档、Next.js 文档等。agent 会高效地在这些层级中导航。

#### 使用 Agent Skills

许多工具都支持“agent skills”，即 agent 可以调用的命令或工作流，用来学习如何完成某项具体工作。它们也是[渐进式披露](https://www.aihero.dev/ai-coding-dictionary/progressive-disclosure)的一种形式：agent 只在需要时加载知识。

我们会在另一篇文章中深入介绍 agent skills。

[

AI Hero · Skill 系统

### 优秀的 AGENTS.md 是第一步

看看我在自己的 AGENTS.md 之上运行了哪些 skills，从而把这个文件转化为实际交付的工作。

查看 skill 集合](https://www.aihero.dev/skills)

## Monorepo 中的 AGENTS.md

你并非只能在根目录放置一个 `AGENTS.md`。你可以在子目录中放置 `AGENTS.md` 文件，它们会**与根目录文件合并**。

这对 monorepo 非常有用：

### 各层级应该放什么

| 层级 | 内容 |
| --- | --- |
| **根目录** | Monorepo 的用途、如何浏览各个 package、共享工具（pnpm workspace） |
| **Package** | Package 的用途、具体技术栈、package 特有的约定 |

根目录的 `AGENTS.md`：

```markdown
This is a monorepo containing web services and CLI tools.
Use pnpm workspaces to manage dependencies.
See each package's AGENTS.md for specific guidelines.
```

Package 级别的 `AGENTS.md`（位于 `packages/api/AGENTS.md`）：

```markdown
This package is a Node.js GraphQL API using Prisma.
Follow docs/API_CONVENTIONS.md for API design patterns.
```

**不要让任何一个层级负载过重。** agent 会在上下文中看到合并后的所有 `AGENTS.md` 文件。每一层都应只关注与该范围有关的内容。

## ==使用这段 Prompt 修复有问题的 AGENTS.md==

如果你开始担心仓库中的 `AGENTS.md` 文件，并希望按照渐进式披露原则重构它，可以尝试把下面这段 prompt 复制粘贴到编码 agent 中：

```txt
I want you to refactor my AGENTS.md file to follow progressive disclosure principles.

Follow these steps:

1. **Find contradictions**: Identify any instructions that conflict with each other. For each contradiction, ask me which version I want to keep.

2. **Identify the essentials**: Extract only what belongs in the root AGENTS.md:
   - One-sentence project description
   - Package manager (if not npm)
   - Non-standard build/typecheck commands
   - Anything truly relevant to every single task

3. **Group the rest**: Organize remaining instructions into logical categories (e.g., TypeScript conventions, testing patterns, API design, Git workflow). For each group, create a separate markdown file.

4. **Create the file structure**: Output:
   - A minimal root AGENTS.md with markdown links to the separate files
   - Each separate file with its relevant instructions
   - A suggested docs/ folder structure

5. **Flag for deletion**: Identify any instructions that are:
   - Redundant (the agent already knows this)
   - Too vague to be actionable
   - Overly obvious (like "write clean code")
```

## 不要构建泥球

准备向 `AGENTS.md` 添加内容时，问问自己它应该放在哪里：

| 位置 | 适用情况 |
| --- | --- |
| 根目录的 `AGENTS.md` | 与仓库中的每一项任务都有关 |
| 独立文件 | 与某个领域有关（TypeScript、测试等） |
| 嵌套文档树 | 可以按层级组织 |

理想的 `AGENTS.md` 小而聚焦，并指向其他位置。它只向 agent 提供足够开始工作的上下文，同时留下通往更详细指导信息的线索。

其他所有内容都通过渐进式披露提供：独立文件、嵌套的 `AGENTS.md` 文件或 skills。

这样既能有效利用指令预算，让 agent 保持专注，也能随着工具和最佳实践的发展，让你的配置继续适用于未来。
