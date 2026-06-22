---
title: "WTF Is a Loop? Peter Steinberger vs. Boris Cherny"
author: "Matt Van Horn"
source: "https://x.com/mvanhorn/status/2063865685558903149?s=20"
canonical: "https://www.linkedin.com/pulse/wtf-loop-peter-steinberger-vs-boris-cherny-matt-van-horn-cpslc"
published: "2026-06-08"
saved: "2026-06-10 16:03:08 CST"
tags: [clipping, x-twitter, linkedin, ai-coding, agents]
---

# WTF Is a Loop? Peter Steinberger vs. Boris Cherny

Matt Van Horn  
Published: 2026-06-08

![WTF Is a Loop? Peter Steinberger vs. Boris Cherny](../../assets/wtf-is-a-loop-peter-steinberger-vs-boris-cherny/cover.jpg)

The most repeated sentence in AI coding this week is six words long, and almost nobody saying it can define it. One tweet had the entire timeline in a chokehold, so I ran /last30days on the word everyone was fighting about. The answer is real, it has a five-year lineage, and the punchline is that the loop, not the model, is now the expensive part.

## The tweet that has the timeline in a chokehold

One tweet has had the entire AI-coding timeline obsessed this week. Peter Steinberger posted it on June 7, it cleared 2.2 million views, and the replies turned into a brawl over what it actually meant.

> "You shouldn't be prompting coding agents anymore. You should be designing loops that prompt your agents." - @steipete, June 7, 2026

The most telling reply asked the only question that mattered: what does this look like in practice? The answer that became the whole mood was Matthew Berman's.

> "nobody knows but him and boris." - @MatthewBerman, June 7, 2026

That is the real story. Not that loops are the future, but that a six-word phrase hit two million views while the people boosting it argued about what it meant. I did not roll my eyes, because I run a loop every night that opens pull requests across roughly thirty open-source repos while I sleep. The loudest idea in AI coding is one most people repeating it cannot explain.

## What a loop actually is

Boris Cherny created Claude Code as a side project in September 2024. It now reportedly sits behind close to four percent of all public commits on GitHub. On stage at WorkOS's Acquired Unplugged event on June 2, he gave the cleanest definition of a loop you will find.

> "I don't prompt Claude anymore. I have loops that are running. They're the ones prompting Claude and figuring out what to do. My job is to write loops." - Boris Cherny, June 2, 2026

A loop is a small program that prompts the agent for you, reads what it produced, decides whether it is done, and if not, prompts it again.

You stop being the thing inside the loop typing prompts. You become the author of the loop, and the model becomes a subroutine. A year ago Boris wrote code by hand with autocomplete. Then he ran five to ten Claude sessions in parallel. Now he does not prompt at all, and a couple hundred agents read his GitHub, Slack, and Twitter and decide what to build next. The job did not vanish. It moved up an altitude, from writing the code to writing the thing that writes the code.

## The spectrum: from ReAct to orchestration

The replies were a mess because loop hides at least five different things. Stage one is the academic while-loop from the 2022 ReAct paper. Stage two is AutoGPT in 2023, which gave it a goal and became famous for spinning forever doing nothing. Stage three is the ralph loop, published by Geoffrey Huntley in July 2025: a bash one-liner that pipes the same prompt file into the agent over and over, resetting context every iteration. Stage four productized it: in spring 2026 both Codex and Claude Code shipped /goal, which runs the ralph loop until a small validator model confirms the task is done.

Stage five is what Boris and Steinberger actually mean, and it is genuinely new. The loop became the unit of work, not the task. Loops started supervising other loops, concurrently and on a schedule. Scheduling replaced the human kickoff. And durability became explicit, with git-backed state and crash recovery. Ralph assumed your terminal stayed open. The 2026 version assumes it does not.

## It's just a cron job with a hat on

The best skeptic line in the corpus was four words.

> "Cronjobs have funny re-branding rn." - X reply, June 2026

This is half right. The scheduling layer is cron. What cron never had is the part in the middle: a model that looks at the current state, decides what to do next, does it, checks whether it worked, and decides whether to keep going. The decision is the agent's, not a hardcoded branch. Stack those, let one loop supervise others, give them durable shared state, and you have something cron cannot express.

Loops are cron plus a decision-maker in the body.

## What it looks like when you actually build one

The on-ramp is one line. Claude Code shipped /loop, and Boris's own example is the canonical starter.

> /loop babysit all my PRs. Auto-fix build issues, and when comments come in, use a worktree agent to fix them.

Days later, Boris posted five tips for running Opus autonomously for hours or days: use auto mode for permissions; use dynamic workflows to orchestrate hundreds or thousands of agents; use /goal or /loop to keep going until it's done; use Claude Code in the cloud so you can close your laptop; and make sure Claude has a way to self-verify its work end to end.

A loop is only as trustworthy as its ability to check its own work.

The deep end is Steve Yegge's Gas Town: twenty to thirty Claude Code instances coordinated by a Mayor agent, with patrol agents running continuous loops and state stored in git so work survives a crash. But the fastest-growing sub-theme was not orchestration, it was verification. Dan Kornas is shipping roborev, which reviews every commit in the background and feeds the findings back into the agent while the context is still fresh. The loop is not the magic. The feedback inside it is.

## The plot twist: the loop is now the expensive part

Here is where the research turned from philosophy to a finance problem. The sharpest deflation came from a working engineer.

> "Every ai agent i shipped this year is a for-loop, an llm call, and a try/catch around the json parsing. The only thing agentic about it is the anthropic bill at the end of the month." - rohit, June 2026

That bill is not a joke. Uber capped its engineers at 1,500 dollars per person per tool per month for Claude Code and Cursor after burning its annual AI budget in four months. Once the model writes the code for almost nothing, the cost moves to the loop running it. Which is why every serious 2026 write-up converges on the same three hard stops: a maximum iteration count, no-progress detection, and a token or dollar budget ceiling. You write the loops, and most of your job is making sure they halt.

## It's not loops. It's skills.

Here is my own take after a week of watching this. The loop is plumbing. The asset is the skill it calls. If you do something more than once, turn it into an automated skill, and if you do something hard, turn it into a skill afterward so next time is free. A loop with no reusable skills inside it is just a while-true around a stranger. A loop that calls a library of sharp, tested, named skills is a system that compounds.

So the answer to WTF is a loop is not a hot take about prompt engineering dying. Stop being the thing in the loop. Write the loop once, give it skills worth calling and feedback so it can check itself, cap it so it halts, and let it run on cron while you go decide what to build next. The good news is that, as of this month, the on-ramp is a single slash command.

## Key patterns from the research

A loop is cron plus a decision-maker in the body: the model, not a hardcoded branch, picks the next action each tick.

The lineage is real: ReAct in 2022, AutoGPT in 2023, ralph in 2025, /goal in spring 2026, orchestration loops now.

The loop is only as good as its feedback. Continuous review and validation gates are what make it trustworthy.

The expensive resource shifted from tokens to loop management. Cap iterations, detect no-progress, set a dollar budget.

The reusable unit inside the loop is a skill, not a prompt.

Compiled from /last30days runs on 2026-06-07. I run loops that ship open-source PRs while I sleep, and I write them with /last30days research running in the background.

---

Source: [X status](https://x.com/mvanhorn/status/2063865685558903149?s=20)  
Canonical: [LinkedIn article](https://www.linkedin.com/pulse/wtf-loop-peter-steinberger-vs-boris-cherny-matt-van-horn-cpslc)
