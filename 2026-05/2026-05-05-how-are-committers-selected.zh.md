---
title: "Committers 是如何被选出的？"
source: "https://vondra.me/posts/how-are-committers-selected/"
author: "Tomas Vondra"
published: "2026-05-05"
clipped: "2026-05-06"
tags:
  - postgres
  - committer
  - core-team
  - development
  - process
  - community
---

# Committers 是如何被选出的？

在最近几次会议上，我有机会描述 Postgres 用来选择新 committers/maintainers 的流程。通常听众是使用 Postgres 的用户和开发者，但在某些情况下，即使是经验丰富的 Postgres 贡献者也不清楚这个流程。

[官方文档](https://www.postgresql.org/developer/committers/)相当简短，并没有解释各种重要细节。让我解释一下我所理解的非正式流程、谁负责什么，等等。

这篇文章并不是要给你如何成为 committer 的建议，那是一个主观得多的问题。也许会在未来某篇文章里写，还不确定。

官方文档是这样描述新增 committers 的流程的：

> New Committers and Removing Committers
>
> 新的 committers 大约每年在现有 committers 讨论并投票后加入。PostgreSQL 的贡献者会基于以下宽松标准被选为 committers：
>
> - 对项目有数年的实质性贡献
> - 多次且持续的代码贡献
> - 负责维护代码库的一个或多个区域
> - 有审查 patches 并帮助其他贡献者处理其 patches 的记录
> - 高质量代码贡献，只需要很少的修订或更正即可 commit
> - 展现出对 patch 接受流程和标准的理解
>
> 一般来说，新的 committers 会在 3 月到 4 月期间选出，并在 Hackers 邮件列表上宣布。
>
> 已经不活跃且数年来未对 PostgreSQL 项目做出重大贡献的 committers 会被移除 committer 身份。同样，对此的审查流程大约每年进行一次。
>

所有这些大体上都是正确的。但它没有深入说明实际是怎么做的、谁参与选择。这并非巧合——这个流程相当非正式，以至于它其实并没有真正写在任何地方，而且会随时间演变。

## 谁来做？

最初，选择工作由 [core team](https://www.postgresql.org/developer/core/) 管理，这是一个相对较小的群体，受社区委托监督项目的各个方面。core team 有点像指导委员会或董事会，负责管理项目资源等。

core team 会讨论谁可能是合适的候选人。如果达成一致，就会向那个人提供 commit bit。如果那个人同意，就会发布公告，并完成各种步骤（添加 SSH keys，等等）。

在某个时候，其中大部分工作被委托给了所有 committers，这是一个大得多的群体（目前约 30 人）。我认为这一变化既有实际原因，也有“意识形态”原因。

实际原因是，首要问题是“候选人是否贡献了好的代码？”，而 committers 看起来比 core team 更接近这一点。至少原则上如此，因为 core team 的大多数成员本身也是 committers。

意识形态原因是，这是把各种 core team 职责委托给其他群体的更广泛努力的一部分。不只是为了节省 core team 的时间，也是为了把事务委托给更广泛、更多样化的合格人员群体。更多民主与精英治理。

当前流程大致如下：

- 在 2 月到 3 月的某个时候，有人会在私有 committer 邮件列表上发起一个 thread，请其他人推荐候选人。
- 人们会推荐他们认为已经准备好成为 committer，或者经过一些指导后明年可能准备好的候选人。
- 随后展开讨论。人们同意或不同意提议，解释原因，分享他们与候选人协作的经验，等等。
- 最后会形成某种总体共识：谁已经准备好成为 committer，哪些候选人可能需要指导才能在明年准备好，等等。
- 名单会传给 core team，core team 的一名成员会联系被选中的候选人并提供 commit bit。如果候选人同意，就会发布公告，并授予新的 committer commit access。
- 这个流程通常在 5 月中旬结束。它过去与 PgCon 这个“主要”会议绑定，但我认为现在已经不是这样了。

core team 可以在任一方向推翻这份名单，也就是说，拒绝某些候选人，或者添加某人。不过我不认为这种情况曾经发生过。core team 的大多数成员本身也是 committers，所以他们可以从那个身份表达反对意见。

“成为 committer”在实践中意味着什么？

你的 SSH key 会被添加到[官方 git repository](https://wiki.postgresql.org/wiki/Committing_with_Git)，这样你就可以向其 push changes。你会被加入一个私有邮件列表，用来讨论组织事务。不过，技术事务仍然在 [pgsql-hackers](https://www.postgresql.org/list/pgsql-hackers/) 上公开讨论。Committers 也可以选择请求访问几个内部工具。差不多就是这些。

Committers 并没有普通贡献者无法使用的特殊特权或“秘密 backchannels”。

那些差一点没通过的候选人怎么办？会努力给予他们足够的反馈和帮助，让他们能够专注于这些方面，并希望在下一年成功。

## 标准是什么？

正如官方文档所说，候选人应满足的标准有些宽松。不过技术标准应该相当容易理解且合理。

维护者身份需要足够的专业能力。committer 预期是一个长期贡献者，拥有多个非平凡 patches。这些 patches 需要足够接近“ready for commit”，只需最少的调整。

committer 需要理解一般的社区流程。需要相当多的“soft skills”。我们的开发往往由共识驱动，因此协作能力和处理分歧的能力至关重要。

Committers 也被期望帮助其他贡献者处理他们的 patches——通过做 reviews，但不只是这样。不仅是为了获得这些功能，也是为了培养下一代贡献者以及（最终的）maintainers。

## 结论

这就是目前选择新 committers 的流程。我只是描述这个流程，并不是声称它完美，或声称我们不能在其他各种方面做得更好。

关于如何帮助更多贡献者达到标准，目前有非常活跃的持续讨论。[hacking workshops](https://rhaas.blogspot.com/2026/03/hacking-workshop-for-aprilmay-2026.html) 和 [mentoring program](https://rhaas.blogspot.com/2025/03/mentoring-applications-and-hacking.html) 都是这项努力的一部分。我未来可能会写写这些挑战，以及候选人通常因为什么而被拒绝。

希望这让 committer 选择流程更清楚了一些。但如果你仍有问题，欢迎给我发 e-mail。

---

Source: [How are committers selected?](https://vondra.me/posts/how-are-committers-selected/)
