---
title: "How are committers selected?"
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

# How are committers selected?

At a couple recent conferences, I got to describe the process Postgres uses to select new committers/maintainers. Usually to users and developers using Postgres, but in some cases it was unclear even to experienced Postgres contributors.

The [official docs](https://www.postgresql.org/developer/committers/) are rather brief, and don’t explain various important details. Let me explain how I understand the informal process, who’s responsible for what etc.

This post is not meant to give you advice on how to become a committer, that’s a far more subjective question. Perhaps in some future post, not sure yet.

The official docs describe the process of adding committers like this:

> New Committers and Removing Committers
>
> New committers are added approximately annually after discussions and vote among the existing committers. Contributors to PostgreSQL are selected to be committers based on the following loose criteria:
>
> - several years of substantial contributions to the project
> - multiple and continuing code contributions
> - responsibility for maintenance of one or more areas of the codebase
> - track record of reviewing and helping other contributors with their patches
> - high quality code contributions which require very little revision or correction for commit
> - demonstrated understanding of the process and criteria for patch acceptance

> Generally, new committers are selected in March to April and announced on the Hackers mailing list.
>
> Committers who have become inactive and have not contributed significantly to the PostgreSQL project in several years are removed as committers. Again, the review process for this is approximately annual.
>

All of this is generally correct. But it does not go into details about how it’s actually done, who participates in the selection. That’s not a coincidence - the process is quite informal, so much that it’s even really written down anywhere and evolves over time.

## Who does it?

Initially, the selection was managed by the [core team](https://www.postgresql.org/developer/core/), a relatively small group of people entrusted by the community to oversee various aspects of the project. The core team is something like a steering committee or a board of directors, responsible for managing project resources, etc.

The core team discussed who might be a good candidate. If there was an agreement, the commit bit was offered to that person. And if the person agreed, an announcement was made, and the various steps were done (adding SSH keys, …).

At some point, most of this was delegated to all committers, a much larger group of people (currently ~30). I believe this changed for practical and “ideological” reasons.

Practical, because the primary question is “Does the candidate contribute good code?” and committers seem closer to this than the core team. At least in principle, because most of the core team are committers too.

Ideological, because it’s a part of a wider effort to delegate various core team responsibilities to other groups. Not just to save time for the core team, but also to delegate stuff to a wider and more diverse group of qualified people. More democracy & meritocracy.

The current process works roughly like this:

- Sometime in February - March, someone starts a thread on a private committer mailing list, asking others to suggest candidates.
- People suggest candidates they think are ready to be a committer, or might be ready next year with some guidance.
- A discussion follows. People agree or disagree with the proposal, explain why, share their experiences from collaborating with the candidate, etc.
- In the end there’s some sort of general agreement on who is ready to become a committer, which candidates may need guidance to be ready next year, etc.
- The list is passed to the core team, one of the core team members reaches out to the selected candidates and offers the commit bit. If the candidate agrees, an announcement is made and the new committer is given commit access.
- This process usually wraps mid-May. It used to be tied to PgCon, as the “main” conference, but I believe it’s no longer the case.

The core team can overrule the list in either direction, i.e. reject some of the candidates, or add someone. I don’t think that ever happened, though. Most of the core team are committers too, so they could voice their objections from that position.

What does “becoming a committer” mean in practice?

You get your SSH key added to the [official git repository](https://wiki.postgresql.org/wiki/Committing_with_Git), so that you can push changes to it. You gets added to a private mailing list, used to discuss organizational stuff. Technical stuff is still discussed in the open on [pgsql-hackers](https://www.postgresql.org/list/pgsql-hackers/), though. Committers also have an option to request access to a couple internal tools. That’s about it.

Committers don’t have special privileges or “secret backchannels” unavailable to regular contributors.

What about the candidates that didn’t quite make it? There is an effort to give them enough feedback and help so that they can focus on that, and hopefully succeed the following year.

## What are the criteria?

As the official docs say, the criteria a candidate should meet are somewhat loose. The technical criteria should be pretty understandable and reasonable, though.

Maintainership requires sufficient expertise. A committer is expected to be a long-term contributor with multiple non-trivial patches. The patches need to be sufficiently close to “ready for commit” with only minimal tweaks necessary.

A committer needs to understand the general community processes. A fair amount of “soft skills” required. Our development tends to be consensus-driven, so the ability to collaborate and deal with disagreement is crucial.

Committers are also expected to help other contributors with their patches - by doing reviews, but not just that. Not only to get the features, but to grow the next generation of contributors and (eventually) maintainers.

## Conclusion

So that’s how the process of picking new committers currently works. It’s merely a description of the process, I’m not claiming it’s perfect or that we couldn’t do better in various other ways.

There’s a very active ongoing discussion on how to help more contributors to meet the criteria. The [hacking workshops](https://rhaas.blogspot.com/2026/03/hacking-workshop-for-aprilmay-2026.html) and [mentoring program](https://rhaas.blogspot.com/2025/03/mentoring-applications-and-hacking.html) are part of this effort. I might write about these challenges in the future, along with the stuff candidates tend to get rejected for.

Hopefully, this made the committer selection a bit clearer. But if you still have questions, feel free to shoot me an e-mail.

---

Source: [How are committers selected?](https://vondra.me/posts/how-are-committers-selected/)
