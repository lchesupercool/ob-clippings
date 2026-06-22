---
title: "Shann³ 在 X 上：\"如何成为 Hermes Agent 操作员\" / X"
source: "https://x.com/shannholmberg/status/2055335043904492011?s=52"
saved: "2026-05-18 20:43:41"
author: "Shann³ (@shannholmberg)"
tags: [x-twitter, hermes-agent, clipping]
---

# Shann³ 在 X 上："如何成为 Hermes Agent 操作员" / X

学习如何操作并掌握 Hermes Agent。设置 agent control room 模板，配置 specialist agents，并从一个 agent 成长为运行在一台 VPS 上的完整营销公司。

大多数 AI 工具回答问题。Hermes agent 端到端运行你的工作流。

它会导航你的浏览器、执行终端命令、安排 cron jobs、监控你的收件箱、起草工作内容，并把结果发布到你所在的任何地方：telegram、discord、slack、你此刻正在参与的 email thread。

由

构建，而且它是开源的，拥有 150,000 个 github stars。目前在 OpenRouter 的全球 token 使用量中排名 #1。

这是过去几周我围绕其构建整个营销运营的框架，而你即将阅读的这篇文章，就是如果我今天从零开始，我会如何设置它。

![图片](../../assets/how-to-become-a-hermes-agent-operator/01.jpg)

*   什么是 hermes agent，以及为什么营销人员（不只是开发者）应该关心它

*   架构的读者友好版本：大脑、人格、技能集，以及它们如何都存在于一个文件夹里

*   我个人正在 hermes 上运行的用例，以及我已经发布的关于它们的四篇帖子

*   四部分心智模型（你、control room、agents、可选 task bus）以及四个设置层级，从“你笔记本上的一个 agent”到“你可以从手机控制、运行在 VPS 上的全自动 agent 团队”

*   我用来把一个营销工作流从混乱想法推进到自主部署的 prototype → production 方法论

*   我希望自己第一天就拥有的资源：docs、community atlas、值得关注的人、正在发生的 meetups

*   坦诚的取舍，以及它仍然会出问题的地方

我在这篇文章里不卖你任何东西。hermes 是开源的，Nous Portal 有免费层级，而且大多数社区生态也是免费的。fork、修改、让它成为你的东西。

短版：一个运行越久能力越强的 autonomous agent。

长版：hermes 是 Nous Research 构建的一个框架，它把模型变成一个持久化 operator。它有自己的 memory，可以跨 session 存活。它在工作时会编写自己的 skills。它自带 123 个已经内置的 skills（github workflows、obsidian、google workspace、linear、notion、typefully、perplexity、deep research，外加 100+ 更多）。它可以存在于你放置它的任何地方：你的笔记本、docker container、VPS、serverless runtime。你可以通过 20+ 个界面与它对话：telegram、discord、slack、email、voice mode，或者仅仅是你的 terminal。

Shann³

@shannholmberg

Hermes Agent 改变了我的工作方式。它是你现在可以设置的最高杠杆 agent framework。它的不同之处：> 它根据复杂度和成本把任务路由到合适的模型 > 随时间学习你的语气和偏好 > 处理 context switching 而不会

![图片](../../assets/how-to-become-a-hermes-agent-operator/02.jpg)

引用

Shann³

@shannholmberg

5月10日

![图片](../../assets/how-to-become-a-hermes-agent-operator/03.jpg)

1:51

Nate Herk 刚刚写了互联网上最详尽的 Hermes Agent 设置指南。以下是你在构建自己的 agent 之前需要知道的 12 个经验  x.com/19216745449994…

如果你用过 claude code 或 openclaw，hermes 形态相同，但理念不同。

> hermes 是 rails。有主见的默认设置，batteries included，第一天只需最少设置就能高效，agent 会替你做更多思考。

> openclaw 是 linux。原语、保证、显式控制，agent 严格执行你告诉它的事，仅此而已。

两者都有效。我运行 hermes，是因为它捆绑的默认设置会复利增长。每个我用 hermes 开始的项目，agent 在我写一行配置之前就已经知道如何做 100+ 件事。这个起步优势对我来说值得。我也注意到 hermes 远没有同样程度的 gateway 断连或出 bug 问题。

证据就在 Nous Research 刚刚达到的数字里：

*   OpenRouter 全球 token 使用量 #1（在该平台的所有模型和框架中）

*   hermes repo 上有 150,000 个 github stars

*   在 agent 编写自己的第一个 skill 之前，已有 123 个 bundled skills

*   gateway 中有 70+ 个内置 tools，外加通过一个订阅访问 300+ 个 models

*   6 个部署目标：local、docker、ssh、daytona、singularity、modal

*   20+ 个 messaging surfaces：telegram、discord、slack、email、voice

如果你是 AI marketer，却还没有开始运行 hermes，那么你每周都在把复利能力留在桌面上。

每个 hermes agent 都有三样东西。

一个大脑。memory 位于 ~/.hermes/memories/。两个文件 MEMORY.md 和 USER.md 会在 session 开始时注入。你的 voice rubric、brand notes、customer language、上周的 corrections，所有这些都会在第一个 prompt 之前加载。sessions 存储在 sqlite 中，跨 sessions 的 recall 支持全文搜索。

一个人格。soul.md 是 vibe 所在的地方。简洁。讽刺。直白。正式。快速或深思熟虑。你可以启动六个 agents，给每个不同的 soul，底层同一个 brain。一个是带着 closer 能量的 outbound rep。另一个是喜欢长句的 researcher。另一个是让一切保持简短的 assistant。

![图片](../../assets/how-to-become-a-hermes-agent-operator/04.jpg)

开箱即用的 123 个 skills：github PRs、obsidian、google workspace、linear、notion、typefully、perplexity、deep research、browser control、web scraping、vision、voice、scheduling。还有闭环学习：当 agent 工作时，它会一路编写新的 skills。你的专属 skills library 会在这 123 个之上增长，而你不必亲自编写其中任何一个。

然后是 agent 能连接什么。

*   tool gateway：一个订阅，300+ 个 models，外加内置 web scraping 和 browser automation

*   MCP integration：任何支持 Model Context Protocol 的外部服务都会变成你的 agent 可用的 tool

*   20+ 个 messaging surfaces：telegram、discord、slack、email、voice，加上 CLI 本身

![图片](../../assets/how-to-become-a-hermes-agent-operator/05.jpg)

以及 agent 可以住在哪里。

*   你的笔记本（local）

*   一个 docker container（隔离、可移植，也是我运行自己的方式）

*   VPS 上的一个 ssh session（所以即使你的笔记本合上了它也会运行）

*   daytona、singularity、modal（如果你不想管理基础设施，可以用 serverless）

闭环学习是它不同于智能聊天机器人的地方。agent 会观察自己工作，随着它学习你工作的形状而编写新的 skills，定期精炼它的 memory，并使用全文搜索和 LLM 总结的混合方式跨 sessions 召回过去的 context。你下周不必重新教它。

> 我告诉 hermes 新手的规则是：第一天不要试图编写你自己的 skills。运行真实工作，让 agent 观察，并让 harness 编写 skills。通过工作，你会比通过写 prompts 更快构建自定义 skill library。

我是 AI marketer，不是 coder。我在 hermes 上运行的大部分是 marketing infrastructure，偶尔也有 internal tool。以下是真实列表：

*   一个 personal assistant，处理 business 和 private，住在 telegram，每天早上标记四封值得阅读的 emails，安排我的 reminders，总结我错过的 meetings

*   一个 marketing workflow prototyping bench，我在这里用真实工作测试新 flows（lead magnet、ad creative review、content sprint），运行 2-3 次后再提升它们

*   specialized marketing agents：SEO、outbound / BD、design review、content writing，每个都有自己的 soul 和 scope

*   一个 company brain，监控 slack、chats、emails、transcripts、voice memos，并让所有这些都可查询。当我问“我们上个月关于 pricing 对那个客户说了什么”时，我 3 秒得到答案，而不是挖 30 分钟

*   一个 SEO agent，在一个 docker container 中运行从 keyword seed 到 published article 的完整 pipeline，21 个步骤，中间无人参与直到 final review

*   一个 content distribution agent，接收一篇 long form（比如这篇文章），并用各平台特定 hooks 把它拆分发布到 LinkedIn、X、Threads

*   一个 orchestrator agent，它自己不产出工作，只是根据我的请求把 requests 路由到合适的 specialist

我发布的 blueprint 对它的总结：

Shann³

@shannholmberg

我的 Hermes Agent 公司的 org chart：四层，全都是一台 vps 上隔离的 docker containers：1. company brain - vision、brand、customers、products。其他每一层继承的 context 2. orchestrator hermes agent - 读取 company brain，选择正确的 department，

![图片](../../assets/how-to-become-a-hermes-agent-operator/06.jpg)

引用

Shann³

@shannholmberg

5月12日

![图片](../../assets/how-to-become-a-hermes-agent-operator/07.jpg)

Hermes Agent 改变了我的工作方式。它是你现在可以设置的最高杠杆 agent framework。它的不同之处：> 它根据复杂度和成本把任务路由到合适的模型 > 随时间学习你的语气和偏好 > 处理 context switching 而不会  x.com/14559993134531…

特别值得放大来看的是 SEO agent，因为它是我已经公开发布的那个，也是最清晰映射到本文其余架构的那个。五层，全都在一个 docker container 内，从 keyword seed 到 published article 共 21 个步骤。

这 21 个步骤在 terminal 中看起来像这样：

markdown

```
[research + ideate]
  01 keyword seed
  02 serp snapshot
  03 competitor extraction
  04 intent + format analysis
  05 content + visual gap
  06 internal validation
  07 external validation

[production]
  08 angle + positioning brief
  09 visual strategy brief
  10 outline
  11 draft
  12 image gen
  13 flowchart gen
  14 visual qa
  15 article qa

[distribution]
  16 publish prep
  17 schema
  18 internal linking
  19 syndication
  20 analytics setup
  21 monitoring
```

这条 pipeline 之上的 layers：

1.   最上层的 company brain：vision、brand、audience、products。每个 agent 都从这里读取

2.   orchestrator hermes agent：接收 topic 或 keyword seed，并将其路由到 seo agent

3.   seo brain：ranking playbook、voice rules、content formats、visual style guide、每种 format 的 success criteria。所有 seo-specific context 都在这里

4.   SEO agent 内部的三个 sub-agents，每个处理一个 phase：

5.   research + ideate：keyword seed、serp snapshot、competitor extraction、intent and format analysis、content and visual gap、internal and external validation

6.   production：angle and positioning brief、visual strategy brief、outline、draft、image gen、flowchart gen、visual and article qa

7.   distribution：publish prep、schema、internal linking、syndication、analytics、monitoring

8.   一个 docker container 容纳所有三个 sub-agents。它们共享 env、memory 和 tools。sub-profiles 会按 phase 切换 context。一个 process、一个 filesystem、一组 credentials。

为什么是一个 container 而不是三个：seo 工作是顺序的。research 供给 brief，brief 供给 production，production 供给 distribution。每一步都需要记住上游已经决定的内容。拆成三个 containers 意味着要跨边界传递 state，这会变得昂贵并打断链条。

公司里的其他每个 specialized agent 都运行在同一个模板上。clone SEO agent template，替换 brain（seo brain → outbound brain，或 → design brain，或 → support brain），你就能为任何 function 获得一个同样五层形状的新 agent。

Shann³

@shannholmberg

我的 hermes seo agent 在 org chart 中如何工作：它运行从 keyword seed 到 published article 的完整 pipeline，21 个步骤，全部在一个 docker container 内。结构：LAYER 1: company brain shared context: vision、brand、audience、products。每个 agent 都从这里读取

![图片](../../assets/how-to-become-a-hermes-agent-operator/08.jpg)

引用

Shann³

@shannholmberg

5月13日

![图片](../../assets/how-to-become-a-hermes-agent-operator/09.jpg)

我的 Hermes Agent 公司的 org chart：四层，全都是一台 vps 上隔离的 docker containers：1. company brain - vision、brand、customers、products。其他每一层继承的 context 2. orchestrator hermes agent - 读取 company brain，选择正确的 department，  x.com/14559993134531…

> layers 不是装饰。它们是 agent 在工作变得 specialized 时不丢失 context 的原因。company brain 保持稳定，而 worker 迭代。brain layers 让 worker 可替换。

我最近还在我们位于 Lisbon 的

HQ 举办了一场 Nous Research 的 Hermes Agent evening。

来自 Nous 的人主持了 Q&A，来自 noticed .so 的 Simao 展示了一个带 autoresearch 的 agent harness，而我讲解了我们如何在 Espressio 使用 hermes 做 growth。

Shann³

@shannholmberg

我们将在明天于 Espressio HQ 举办

一场 Hermes Agent evening

由 Talent Protocol 协作组织。以下是当晚的 agenda：> 我会先讲如何使用 Hermes Agent 做 growth，我们正在 shipping 的内容

![图片](../../assets/how-to-become-a-hermes-agent-operator/10.jpg)

![图片](../../assets/how-to-become-a-hermes-agent-operator/11.jpg)

如果你在 Lisbon，并且想参加下一场，我会在排期后发布。

在 levels 之前，先讲 mental model。

这个 setup 有四个部分：

*   你是 operator。你可以直接访问系统的每个部分。

*   agent control room 是侧边 control plane。它不是你用来聊天的 agent。它是 /root/vps-agents 下的一个文件夹，用来记录和治理整个 fleet。你打开它、编辑它、检查它，或者在管理系统时让 claude、codex 或 hermes 使用它。

*   hermes agents 是 workers。有些是 specialists（seo、dev、cmo、ops）。其中一个可以可选地作为 orchestrator。

*   agent task bus 是一个可选的 handoff desk，位于 orchestrator 和 specialists 之间。只有当你已经让 orchestrator 参与时才需要它。

整体看起来像这样：

markdown

```
┌───────┐
                                  │  YOU  │   the operator
                                  └───┬───┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
   control path                orchestrated path                direct path
        │                             │                             │
        ▼                             ▼                             ▼
 ┌────────────────────┐    ┌────────────────────┐    ┌────────────────────┐
 │ AGENT CONTROL ROOM │    │ HERMES             │    │ SPECIALIST AGENT   │
 │ /root/vps-agents   │    │ ORCHESTRATOR       │    │                    │
 │                    │    │ (optional door)    │    │ seo · dev · cmo ·  │
 │ docs · rules ·     │    └─────────┬──────────┘    │ ops · life         │
 │ runbooks · env-map │              │ delegates     │                    │
 │ · registry         │              ▼               │ talk to it         │
 │                    │    ┌────────────────────┐    │ directly,          │
 │ side control plane │    │ AGENT TASK BUS     │    │ no routing         │
 │ no raw secrets     │    │ /srv/agent-bus     │    │                    │
 │                    │    └─────────┬──────────┘    │                    │
 └────────────────────┘              │               │                    │
                                     │ routes        │                    │
                                     └───────────────▶                    │
                                                     │                    │
                                                     └────────────────────┘

 the agent control room governs every agent in this diagram. it is the
 single source of truth, and the place you go to manage the fleet, not
 the place you go to run work through it.
```

storage split 比人们想象的更重要：

markdown

```
/root/vps-agents          → control room: docs, rules, runbooks, architecture
                            no raw secrets, ever

/srv/<agent-name>/data    → live runtime: secrets, memory, skills, sessions, crons
                            this is where each hermes agent lives
```

control room 包含这些问题的答案：有哪些 agents，它们做什么，使用哪些 ports，引用哪些 credentials，每个 agent 可以和不可以做什么，以及如何 restart、debug 或 rebuild 它们中的任何一个。live runtime 包含实际运行中的内容。

> control room 是定义系统的大脑。live runtime 是运行系统的身体。你可以从大脑重建身体。你不能从身体重建大脑。

control room 内部：

markdown

```
/root/vps-agents/
  README.md
  CLAUDE.md
  agents/
    <agent-name>/
      inventory.md
      docker.md
      env-map.md
      runbook.md
      backup.md
  shared/
    security.md
    commands.md
  api-keys-sop.md
  orchestrator-and-fleet-skills.md
```

以及每个 agent 在 /srv/<agent-name>/data/ 的 runtime 内部：

markdown

```
.env
config.yaml
SOUL.md
memories/
skills/
cron/
sessions/
logs/
state.db
```

markdown

```
control path:
   you ──────► agent control room
              (add agents, rotate keys, update docs, debug setup)

direct path:
   you ──────► hermes-seo-espressio
              (talk to a specialist directly, fastest)

orchestrated path:
   you ──► hermes-orchestrator ──► task bus ──► specialists ──► you
              (one front door, routes and synthesizes multi-agent work)
```

*   control path 是 meta layer。用于添加 agents、审查 docs、检查 ports、轮换 keys、调试 setup。

*   direct path 最快。用于你已经知道哪个 agent 负责这项工作时。

*   orchestrated path 是 synthesizer。用于你想要一个 front door 来跨多个 specialists 路由并组合工作时。

你有一个 hermes agent。就是这样。control room 仍然可以存在（推荐），但它只记录那一个 agent。

```
you → one hermes agent

control room → documents that one agent
```

最适合：initial setup、你的 personal hermes、root install documentation、simple docker migration。

一个 agent，已经被使用过，带有你调校过的人格和已经开始积累的 memory。用你想要的 voice 填写 SOUL.md，用关于你业务的稳定事实填写 MEMORY.md，用关于你的稳定事实填写 USER.md。把它连接到 telegram 或 discord，让它住在你所在的地方。开始在真实任务上使用它。让它接触你的 tools。让它在过程中编写自己的 skills。

MEMORY.md 保存稳定事实（你的业务是什么、你的 customers 是谁、你的 products 做什么）。USER.md 保存关于你的稳定事实（timezone、working hours、recurring projects、preferred output formats）。当你在真实对话中纠正 agent 时，两者都会每周被精炼。

你有多个 specialized agents，但你仍然直接与每一个对话。还没有 orchestrator。

markdown

```
you → hermes-life
you → hermes-seo-espressio
you → hermes-dev
you → hermes-cmo
```

control room 记录它们全部。

最适合：清晰的 role separation、测试哪些 agents 有用、避免过早 orchestration、让 credentials 按 agent 限定 scope。

> 这里要避免的陷阱，是在证明 specialists 有用之前就伸手去拿 orchestrator。先启动两三个，直接运行它们，只有当你发现自己想要一个 front door 时再添加 orchestrator。

何时启动新 agent，何时继续用已有的：

markdown

```
needs its own credentials → new agent

needs its own long-term memory → new agent

ongoing repeated work that is a separate role → new agent

otherwise stay with what you have
```

坏模式：一个 mega-agent，把每个 credential 和每个 memory layer 都混在一起。你会失去隔离，失去干净撤销访问的能力，而且 agent 会困惑于该使用哪种 voice。

你把 hermes-orchestrator 添加为 front door。你仍然可以直接与 specialists 对话，但 orchestrator 可以路由工作并综合结果。

![图片](../../assets/how-to-become-a-hermes-agent-operator/12.jpg)

orchestrator 会读取 control room，以知道有哪些 agents、每个做什么、task queues 在哪里、什么需要 approval、哪些 actions 被禁止、docs 和 runbooks 在哪里。它不需要问你这些，它会读取。

最适合：cross-functional work、delegation、summary and synthesis、multi-agent workflows 的一个 main interface。

> orchestrator 是你的 setup 从一组 agents 变成一个 team 的时刻。也是 control room 体现价值的时刻，因为 orchestrator 的好坏取决于它读取的 docs。

从我的笔记本或手机快速检查 fleet 的样子：

markdown

```
$ ssh hermes
welcome to hermes-vps-1.
last login: thu may 15 09:14:22

hermes-vps-1 ~ $ cd vps-agents
hermes-vps-1 ~/vps-agents $ docker ps --format \
    "table {{.Names}}\t{{.Status}}\t{{.Image}}"

NAMES                       STATUS         IMAGE
hermes-orchestrator         up 14 hours    hermes-runtime
hermes-seo-espressio        up 8 hours     hermes-runtime
hermes-cmo                  up 8 hours     hermes-runtime
hermes-outbound             up 4 hours     hermes-runtime
hermes-life                 up 12 hours    hermes-runtime

hermes-vps-1 ~/vps-agents $ cat agents/hermes-seo-espressio/runbook.md
# runbook: hermes-seo-espressio
restart:   docker compose restart hermes-seo-espressio
logs:      docker logs -f hermes-seo-espressio
shell:     docker exec -it hermes-seo-espressio bash
...
```

Shann³

@shannholmberg

我的整个 Hermes Agent setup 都由 VPS 上的一个文件夹控制。我可以在 10 秒内从笔记本或手机管理它，为每个项目启动隔离 agents，并且永不丢失 context。完整 setup 如下：> bash command "ssh hermes" 自动连接到 VPS > session

![图片](../../assets/how-to-become-a-hermes-agent-operator/13.jpg)

引用

Shann³

@shannholmberg

5月12日

![图片](../../assets/how-to-become-a-hermes-agent-operator/07.jpg)

Hermes Agent 改变了我的工作方式。它是你现在可以设置的最高杠杆 agent framework。它的不同之处：> 它根据复杂度和成本把任务路由到合适的模型 > 随时间学习你的语气和偏好 > 处理 context switching 而不会  x.com/14559993134531…

与 level 3 形态相同，但有 recurring workflows 和更强的 automation。weekly seo reports 通过 cron 运行。server health checks 每天触发。backup verification 不用你请求就会运行。cross-agent business workflows 按计划启动。

最适合：weekly seo reports、content operations、server health checks、backup verification、cross-agent business workflows。

> level 4 就是在你的 terminal 里的 marketing department。它不需要你来开启一天。它自己上班、提交 reports、检查自身，并且只在需要品味判断的 decisions 上 ping 你。

![图片](../../assets/how-to-become-a-hermes-agent-operator/14.png)

随着你提升 levels，有一个原则要记在脑子里。

control room 用于 config、docs、runbooks 和 governance。它记录哪些 agents 存在、它们做什么、在哪里运行、引用哪些 credentials、每个 agent 可以和不可以做什么。它是 fleet 的 admin panel，包括 orchestrator。它不是你用来做工作的地方。

对于工作，你直接与 agents 对话。要么是 specialist（当你知道哪个 agent 拥有这项 job），要么是 orchestrator（当你想要一个 front door 来跨 specialists 路由）。

现在你理解了 architecture。下面是如何构建它。

我发布了一个 public template，它包含上面描述的精确结构，加上你的 agent 为你设置它所需的 skills。

它位于

。

![图片](../../assets/how-to-become-a-hermes-agent-operator/15.jpg)

你可以手动 clone 它，但重点是你不必这样做。如果你的笔记本上有 claude code 或 codex，在你交出 Hetzner API key 之后，agents 会完成大部分工作。

automated flow：

markdown

```
you  ──►  generate a Hetzner API key
          (5 min: sign up, generate a token, drop it in your .env)
              │
              ▼
agent ──►  create-vps skill
          spins up a Hetzner box, generates an SSH key,
          writes the alias to ~/.ssh/config so `ssh hermes` works
              │
              ▼
agent ──►  setup-control-room skill
          installs Node, Docker, Claude Code, Codex CLI,
          Hermes Agent, then clones the repo to the VPS
          at /root/agent-control-room
              │
              ▼
you  ──►  finish interactive auth on the VPS
          (claude /login, codex, hermes)
              │
              ▼
agent ──►  agent-control-room skill
          registers your first hermes agent in the docs,
          fills in the runbook, sets up the env-map
              │
              ▼
          you are at level 1 with a documented agent
```

十到十五分钟内，你会拥有：

*   一台全新的 Hetzner VPS，并安装好了正确的 tooling

*   control room 被 clone 到 VPS 上的 /root/agent-control-room

*   bundled skills 被链接到 VPS 上的 ~/.claude/skills

*   一个 hermes agent 已注册，runbook 已填写，env-map 已写入

*   你的笔记本上有一个 SSH alias，所以 ssh hermes 可以立即连接

大多数 workflows 一开始都不是 production 级别的。它们一开始很混乱。一个运行 SEO research、drafts an article、在 Typefully 中安排发布并发布到 LinkedIn 的 flow，并不会在你脑中完整成形。你是通过运行它来发现它的。

hermes 就是这种 prototyping environment。以下是我把任何新的 marketing workflow 从 idea 推进到 autonomous deployment 的四步路径：

1.   在 hermes 中 prototype。打开你的 main hermes agent，描述你想发生什么，并让它尝试。第一次运行它大部分都会错。没关系。

2.   用真实工作运行 2-3 次，每次纠正 drift。harness 会观察每次 correction，并在学习这个形状时开始编写 skill。到第三次运行时，agent 不需要 coaching 就能完成你想要的大部分内容。

3.   在 dedicated workspace 中 fine-tune。把 workflow 拉到一个单独的 Claude Code workspace（或者如果你愿意，一个新的 hermes agent），收紧 prompts，锁定 routing，添加 error handling，决定什么应该跑在 cron 上，什么应该被触发。

4.   按 schedule 部署到 VPS。一旦它在一周真实运行中不需要你 babysitting 也能存活，就把它推到 VPS 上自己的 docker container，设置 cron，然后离开。

我是在烧掉几个周末、试图从零编写 production-ready agents 之后学会这个模式的。你不能从零编写 production agent。你必须培养一个。hermes 让培养这个部分变快。

![图片](../../assets/how-to-become-a-hermes-agent-operator/16.png)

1.   在 hermes 中 prototype

2.   在 dedicated workspace 中 fine-tune

3.   在 VPS 上 autonomous deploy

hermes 给你 framework。底层 model 由你选择。通过 tool gateway，你可以用一个订阅路由到 300+ models，按 agent 或按 task 切换。

我个人今天运行的：

*   claude opus 4.7 用于 creative work：copywriting、voice、hook generation、content drafting，以及任何 taste 和 writing quality 很重要的东西

*   codex (gpt 5.5) 用于 structured work：coding、planning、multi-step workflows、browser automation、scraping，以及任何 steps 需要严密且 output 可预测的东西

我两个都运行。opus 写作。codex 构建和规划。hermes 让 routing 变得容易，你把每个 agent 指向适合其工作的 model。

如果你只能运行一个，答案取决于你的 fleet 在做什么类型的工作。heavy on content and copy？从 claude opus 4.7 开始。heavy on infrastructure、automation 和 engineering workflows？从 codex 开始。你之后总能通过同一个 tool gateway 添加第二个 model。

我不会假装 hermes 是完美的。三个真实取舍。

1. bundled defaults 也是

它自带强默认设置，定义 memory 如何工作、skills 如何被编写、agent 如何使用 tools。这就是整个 pitch。但这也意味着，如果你想要的是对每一步都有显式控制的 primitives，hermes 会显得沉重。openclaw 更适合那种品味。选择与你理念匹配的工具。

2. level 3 和 4 有真实的学习曲线。docker、VPS、SSH、control room 文件夹结构、orchestrator skills，这些都不是“install and go”。如果你还没有每天在 level 1 运行 hermes，就不应该跳到 level 3。

3. model 仍然

它是一个让好模型变伟大的 framework。它不会把小模型变成 strategist。对重要的工作使用你能负担的最强 models（你的 orchestrator、你的 strategy agent、你的 brain）。对不重要的工作降到更便宜的 models（research scraping、draft generation、batch processing）。

> 这一切都不是魔法。这是一个会回报你的 framework，因为 memory 会持久化、skills 会累积、agents 会保持 scoped。把它应用到尺寸不合适的 model 上，你会得到一个困惑的 team。把它应用到合适的 model 上，你会得到一个 team。

如果你今天开始，以下是我会按顺序阅读的内容。

*   official docs：

。先从 install guide 开始，然后阅读 skills page，这样你就理解开箱自带什么
*   control room template（我的 repo）：

。上面描述的精确结构，已经准备好 clone。control-room-first template，用于从一个 VPS agent 到 specialist teams 和 orchestrated workflows 管理 hermes agents。fork 它，让它成为你的东西
*   ：community-curated map，收录 100+ 个基于 hermes 构建的 open source tools、plugins、workspaces 和 integrations。按 domain 分类（memory providers、workspaces、skill registries、deployment、orchestration）。还包括 Hermes Handbook，一份 beginner-friendly walkthrough。每周更新，免费 newsletter
*   在 X 上：Nous Research founder。几乎每天发布 hermes updates。codex runtime integration、Nous Portal 上的 DeepSeek V4 Flash free tier、pretext skills，都是最先通过他的 feed 出现
*   在 X 上：official account，官方 feature announcements
*   meetups：现在已经有线下 hermes meetups（Lisbon、Ventura、更多城市）。如果附近有一场，很值得去。你在 90 分钟边聊中学到的东西，比读一周更多

![图片](../../assets/how-to-become-a-hermes-agent-operator/17.jpg)

希望你从中获得了一些价值，感谢你读完整篇。

-- Shann

---

Source: [Shann³ on X](https://x.com/shannholmberg/status/2055335043904492011?s=52)
Saved: 2026-05-18 20:43:41
