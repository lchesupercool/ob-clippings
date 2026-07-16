---
title: "/last7days of Fable 5 and GPT-5.6: What Thousands of Upvotes Say Actually Works"
source: "https://x.com/mvanhorn/status/2077510447016890433?s=52"
author: "Matt Van Horn (@mvanhorn)"
published: 2026-07-15
saved: 2026-07-15
tags: [AI, agents, GPT-5.6, Claude, prompting]
---

# /last7days of Fable 5 and GPT-5.6: What Thousands of Upvotes Say Actually Works

![](../../assets/2026-07-15-mvanhorn-last7days-fable-5-gpt-5p6/cover.png)

For the first time, the two newest frontier models are live at the same time, and the window is one week old. Call it the /last7days. I pointed [/last30days](https://x.com/slashlast30days) at [Claude Fable 5](https://www.anthropic.com/news/claude-fable-5-mythos-5) and [GPT-5.6](https://openai.com/index/gpt-5-6/) more than twenty times this week, one run per angle: effort dials, over-prompting, orchestration, cost, migration mistakes, the works. Back came Reddit threads, YouTube breakdowns, TikToks, LinkedIn posts, GitHub issues, and Hacker News fights, all from the handful of days people have actually had their hands on both. I read every cluster. What follows is the ten practices that actually stuck, each with a receipt from a real person and a payload you can paste today.

The window really is that small. Fable 5 shipped June 9, disappeared into a [19-day export-control shutdown](https://www.anthropic.com/news/redeploying-fable-5), and came back for everyone July 1. GPT-5.6 went public July 9. Day seven of the overlap as I write this. And inside that sliver, both labs published a prompting guide that lands on the same sentence: you are over-prompting, stop it.

I use both models daily, so I had opinions walking in. The community had better ones, and theirs came with upvote counts.

## 1. 🎯 Give it the goal. Delete the steps.

The single clearest statement of the new era came from TikTok before either lab finished publishing theirs.

> Most people use Fable 5 completely wrong. Fable 5 degrades when you over-instruct it. No role, no step-by-step, no 'show your reasoning' - all wasted tokens. Goal-focused prompting: what you're trying to do, relevant context, what's out of bounds, what done looks like. Then let it work.

- thinkwithv, [TikTok](https://www.tiktok.com/@thinkwithv/video/7658032347421347086), 187 likes

Both official guides back this word for word. Anthropic's [Fable 5 guide](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5): "Skills developed for prior models are often too prescriptive for Claude Fable 5 and can degrade output quality." OpenAI's [GPT-5.6 guide](https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6): define the outcome and the completion bar, then "leave room for the model to choose an efficient path." Your carefully numbered playbook from the GPT-5.5 and Opus era is not neutral baggage. It is actively making the new models dumber.

Anthropic even hands you the replacement shape. This is the whole prompt now:

## 2. ✂️ State every instruction exactly once

OpenAI ran the evals on its own guidance and published the numbers, which a builder on X promptly turned into the most useful screenshot of the week.

> leaner prompts outperform detailed ones. internal evals showed that removing redundant instructions improved scores by 10-15% while cutting tokens by 41-66% and cost by 33-67%.

- @oliviscusAI, [X](https://x.com/oliviscusAI/status/2076004557751308647), 21 likes

Read that again: deleting words made the model smarter and cut the bill by up to two thirds. The guide's second warning is the one most production prompts fail: "Conflicting rules can create more instability than missing detail." Every system prompt that has been alive for six months carries two rules that quietly disagree, and the model now spends reasoning tokens trying to satisfy both. The fix is an afternoon: cut anything said twice, cut style notes the model no longer needs, keep the success criteria and stopping conditions, anything touching safety or permissions, and the output shape you require. Then read what is left and hunt the contradictions.

## 3. 🪆 Let the frontier model boss cheaper models around

The most upvoted practice of the entire window was not a prompt trick. It was an org chart.

> Anthropic just benchmarked "Fable 5 orchestrates, cheap models execute": 96% of the performance at 46% of the cost. You can run this pattern in Claude Code today.

- [r/ClaudeAI](https://www.reddit.com/r/ClaudeAI/comments/1ur2ml9/anthropic_just_benchmarked_fable_5_orchestrates/), 1,246 upvotes

[Theo's guide to Fable 5](https://www.youtube.com/watch?v=8GRmLR__OGQ), 117,924 views, runs the same play: Fable designs and reviews, and he tells it outright that a cheaper model can handle "basically anything" in between. The screenshot playbook making rounds on TikTok named it [the Advisor Pattern](https://www.tiktok.com/@estop845/video/7659939531201727774). Anthropic's guide explains why it works now when it half-worked before: Fable 5 "is significantly more dependable at dispatching and sustaining parallel subagents." The orchestrator line to steal:

## 4. 🥊 Make the two models fight

My favorite find of the whole sweep. One developer on r/ClaudeCode refuses to pick a side, and instead runs the frontier's first genuine head-to-head as a standing workflow.

> I give both fable and sol 5.6 xhigh problem statement and goal and let them design, whoever wins gets to execute the design and the loser gets to redeem itself by tearing it down at checkpoints. And I also make them keep a score.

- SpaceCowboy077, [r/ClaudeCode](https://www.reddit.com/r/ClaudeCode/comments/1uvjlmr/comment/oxbq5wr/), 302 upvotes

Two frontier models, one design-off per task, and the loser becomes the adversarial reviewer. A calmer production version of the same idea showed up in the comments of the AI LABS breakdown: Codex handles the bulk of review, and a final Fable pass "often catches issues that slip through Codex's review." Either way the principle holds. For the first time you can afford a second frontier opinion, and the people getting the best results are buying one.

## 5. 💸 Pick models by dollars-per-task

The benchmark arguments burned out in about a day. The thing that kept getting quoted was cost, and the sharpest line was about the mid tier, not the flagship.

> Terra tying with fable at 1/4 the cost 💀

- ethotopia, [r/OpenAI](https://www.reddit.com/r/OpenAI/comments/1us7nml/comment/owlnqkv/), 183 upvotes

The receipts stack up fast. Sam Altman says [GPT-5.6 is 54% more token efficient on agentic coding](https://www.cnbc.com/2026/07/09/open-ai-sam-altman-chatgpt-5-6-sol.html). [Varun Mayya's breakdown](https://www.instagram.com/reel/DalTEPIBoRu/), 47,413 views, has Sol matching Mythos-class output on a third of the tokens. On the other side of the fence, Fable 5 bills $10 per million tokens in and $50 out, which is how one YouTuber came to title a video [Stop Using Fable 5 in Claude Code (It's Holding You Back)](https://www.youtube.com/watch?v=d9XCX0PcOq0) after clocking a single opening message at thirty to forty cents. Nobody serious is picking one model anymore. They are writing a routing table: frontier for the hard, long, ambiguous work, and Terra, Luna, or low-effort Fable for everything that just needs doing.

## 6. 🎚️ The effort dial goes up. That doesn't mean you should.

Both models now expose how hard they think as a setting, and the most counterintuitive finding of the window is that maxing it out is usually a mistake.

> on Fable 5, turning [effort] up is actually a huge trap.

- AI LABS, ["How To Use Claude Fable 5 Better Than 90% Of The People"](https://www.youtube.com/watch?v=GM7-ei-4Xc8), 27,077 views

Anthropic's guide is explicit: high is the default, xhigh is for the most capability-sensitive work only, and "lower effort settings on Claude Fable 5 still perform well and often exceed xhigh performance on prior models." OpenAI says the same about GPT-5.6's reasoning levels: go above high only "with measured eval gains." Extra effort does not just cost money and minutes. It invites gold-plating, the unrequested refactor, the helper nobody asked for. The guide's own countermeasure is paste-ready:

## 7. 🗒️ Give it a notebook it can keep

The most-shared walkthrough of Anthropic's playbook opened with the tip almost nobody had tried, and it is the cheapest compounding upgrade on this list.

> Give Fable a place to write notes. A simple markdown file where it records what it'll learn from each run. What corrections it made, what approaches worked and what didn't.

- raycfu, [Instagram](https://www.instagram.com/reel/Day9pQbP2up/), 17,304 views

One markdown file turns a goldfish into a colleague. Every correction you make today is a correction you do not make next month. Straight from the guide:

## 8. 🧾 Trust nothing without a tool receipt

Long runs are the headline capability of both models, and the failure mode people learned the hard way is the confident progress report that never happened. One Codex user described what earning trust actually looks like.

> GPT-5.6 Sol is the first model that consistently holds context, uses tools, checks its work, and finishes the job without babysitting.

- @raul_rcl22, [X](https://x.com/raul_rcl22/status/2077288097075577194), 2 likes

Checking its work is a behavior you install. It does not come free with the subscription. Anthropic tested one instruction and reports it "nearly eliminated fabricated status reports even on tasks designed to elicit them." A Reddit veteran gave the same advice from the trenches: "I always have it run a final adversarial review. It always finds mistakes" ([jtmonkey, r/ChatGPT, 125 upvotes](https://www.reddit.com/r/ChatGPT/comments/1utxylf/comment/owzf629/)). The instruction:

## 9. 🤐 Never ask Fable 5 to show its reasoning

The sleeper gotcha of the migration. Per Anthropic's guide, prompts that "tell the model to echo, transcribe, or explain its internal reasoning as response text can trigger the reasoning_extraction refusal category" on Fable 5, and the refusal quietly falls your request back to Claude Opus 4.8. You keep paying for the frontier model and silently stop getting it.

Every "explain your thinking," "walk me through your reasoning," and "show your work" you wrote for older Claudes is now a landmine. Grep your skills and system prompts for them and delete. If you need to see reasoning, the API already returns structured thinking blocks; read those instead of asking the model to narrate itself into a refusal.

## 10. 🎛️ Set the verbosity knob and stop nagging

The pettiest habit the new models expose: stuffing "be concise" into every prompt, which is exactly the redundant instruction rule 2 told you to delete. GPT-5.6 has a `text.verbosity` setting, low, medium, or high, that fixes the default once at the API. Fable 5 follows a single readability instruction better than a stack of reminders. The community explained why brevity is worth engineering for, in the most quoted line of the Claude window.

> If Sonnet 5 can perform nearly as well as Opus 4.8 with a third of the output, count me in. Opus 4.8 talks more than a toddler mainlining sugar.

- trevormead, [r/ClaudeAI](https://www.reddit.com/r/ClaudeAI/comments/1ujwggp/comment/our5env/), 575 upvotes

Set the knob once. Say it once. Move on.

## 🧵 What ties all ten together

Line the ten up and they are one instruction wearing ten outfits: your leverage moved out of the prompt and into the process. Both labs converged on it independently in the same two weeks, which is as close to ground truth as this industry produces. The wins people are actually banking come from routing the right task to the right model at the right effort, letting the frontier model command cheaper ones, forcing evidence-backed status reports, and deleting half of what your prompts used to say. And underneath it all sits the humbling one, from a forty-year veteran on a viral build video: "the AI is the easy part. The real work is the architecture, debugging, testing, security, UI, integrations" ([martyman1964, TikTok, 115 likes](https://www.tiktok.com/@dhaibuilds/video/7660125139555683606)). The models stopped being the bottleneck. You are the variable now.

## 🛠️ Field-tested patterns to copy

What the last two weeks say to do, in one pass:

- Rewrite one old prompt as goal, context, boundaries, and definition of done. Delete the steps. Compare the output yourself.

- Audit your longest system prompt for duplicates and contradictions. OpenAI's evals price that cleanup at 10 to 15% more quality for up to 67% less cost.

- Run the orchestrator pattern in Claude Code: Fable plans and reviews, cheap models execute. 96% of the performance, 46% of the cost.

- On anything hard, give Fable and Sol the same goal at xhigh, ship the winner's design, and make the loser review it at checkpoints.

- Keep effort at high by default. Route routine work to Terra, Luna, or low effort. Reserve xhigh and max for problems that earn them.

- Give the model a notes file, require tool-receipt audits before any progress claim, and grep your Fable prompts for "show your reasoning" and delete every hit.

## 📊 All agents reported back

Compiled from more than twenty [/last30days](https://x.com/slashlast30days) runs across Reddit, X, YouTube, TikTok, Instagram, LinkedIn, Hacker News, GitHub, and Digg, covering the two weeks Fable 5 has been back and the six days GPT-5.6 has been public. Loudest signal: the orchestrator benchmark, 1,246 upvotes. Sharpest workflow: the Fable-versus-Sol design-off, 302 upvotes. Cleanest thesis: thinkwithv's goal-focused prompting, 187 likes. Hardest receipt: OpenAI's own 10-to-15% quality lift from 41-to-66% fewer tokens. Top voices: SpaceCowboy077, thinkwithv, raycfu, ethotopia, @oliviscusAI; r/ClaudeAI, r/ClaudeCode, r/OpenAI.

This piece is a stack of searches and an afternoon of reading. I did not interview anyone. The quotes are real, the receipts are real, and you can run the same queries yourself. The payloads come straight from the two official guides. The community and the model makers wrote this playbook the same week; I just collated it.
