---
title: "Understanding is the new bottleneck"
source: "https://www.geoffreylitt.com/2026/07/02/understanding-is-the-new-bottleneck"
author: "Geoffrey Litt"
published: "2026-07-02"
saved: "2026-07-12"
tags: [AI, Agent, AI Coding, 人机协作, 认知债务]
---

### July 2026

# Understanding is the new bottleneck

This is a written version of a talk I gave at the
[AI Engineer](https://www.ai.engineer/)
conference in July 2026, also shared as
[a tweet thread.](https://x.com/geoffreylitt/status/2072522251300409556)

![Title slide: Understanding is the new bottleneck. Geoffrey Litt, Design Engineer at Notion.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-01.webp)

**Hot take: I think it's still important to understand the code that our agents write!**

In this talk I'll explain why that's the case, and show some ideas for how to efficiently understand code. Alright, let's dive in.

![Cartoon of a person surrounded by a growing pile of agent-written code.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-02.webp)

Agents are writing more and more code for us, and we all know it's getting harder to keep up.

But the good news is: there are many ways to understand code! Reading diffs line by line is not the only way.

![Slide listing techniques: code explainer docs, quizzes, micro-worlds.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-03.webp)

Most of this talk will be about techniques I have found helpful to understand systems my agents are building:

- Code explainer docs
- Quizzes to check my understanding
- Micro-worlds that I can play with to understand the system

But first we have to ask a more basic question…

## Why understand?

![Slide reading: Why understand?](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-04.webp)

**Why? Why understand?**

Aren't we supposed to be taking ourselves out of the loop now, and letting the agents loop themselves? As the agents get smarter, doesn't it become less important for us to be in the details?

I think many people — even those who are pro-understanding — have a slightly incorrect answer to this question!

![Slide: understand to verify.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-05.webp)

One possible answer: we understand *to verify*. We check the agent's work, we see if it's correct.

Correct can mean many things: does it match the spec, is it well architected… but it's fundamentally a thumbs-up / thumbs-down question.

![Slide about agents getting better at verifying their own work.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-06.webp)

Here's the thing: the agents are getting better and better at verifying their own work. And this is good! I like it when my agent doesn't make mistakes.

But hmm. Where does that leave us humans?

![Slide: understand to participate.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-07.webp)

**That's where another answer comes in: we can understand to participate.**

You can learn what the agent is doing to make sure you can be an active participant in the creative process. Here's why this matters…

![Diagram of a project as many iterative loops with an agent.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-08.webp)

It's never just one loop! A project is many, many loops with the agent.

And the understanding you have of the system is part of your ability to come up with the next idea to evolve it.

You need a rich set of concepts in your mind to think creatively and fluently about how to move something forward. If you're lacking that fluency, your ability to participate in the project is meaningfully limited.

![Quote from Margaret Storey on cognitive debt: the humans involved may have simply lost the plot.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-09.webp)

By the way, this relates closely to the idea of [cognitive debt](https://margaretstorey.com/blog/2026/02/09/cognitive-debt/), popularized by Margaret Storey and [Simon Willison](https://simonwillison.net/).

It's like tech debt: you can get away with not understanding what's going on in the short term, but it'll bite you eventually.

![Slide asking: how do we build understanding? Pointing to education for inspiration.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-10.webp)

OK, so fine, understanding matters.

But this raises the next question: *how*? **How do we build this human understanding when we're working with AI and moving fast?**

Well, turns out this is not the first time anyone has ever thought about how to communicate understanding. I think we can look to education as an inspiration. Can we steal the best ideas ever invented for education and apply them to this problem?

## Technique 1: Explanations

![Slide listing three techniques, with 'explanations' highlighted.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-11.webp)

Today I want to share three techniques that show how we can attempt this.

First: explanations. What makes a good explanation?

![Slide showing a raw code diff.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-12.webp)

Whenever an agent finishes some work, it's an opportunity for an explanation — an artifact.

Most naively, we can read a code diff: the raw material that changed.

![Slide asking: what would the best explanation be?](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-13.webp)

But what if we ask:

**What would the *best* explanation be?** If you had a team — human or AI — that really sweat the details of explaining something well to you, how would that feel?

![Screenshot of a code explainer doc produced by the /explain-diff skill.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-14.webp)

Here's one answer. I made a skill called [/explain-diff](https://gist.github.com/geoffreylitt/a29df1b5f9865506e8952488eac3d524), which I use every day and many coworkers have found valuable.

It outputs thoughtfully structured code explainers as HTML, markdown, or Notion docs. Notion is a good place for collaborating on and discussing these explainers as a team. (Disclaimer: I work at Notion so I'm biased.)

Let's see what's in one of these explainers, using an example of editing the perspective of a video game.

![Explainer section teaching background info about the game engine.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-15.webp)

First principle: **teach me background info!**

Before we even get to what changed, help me understand what was already there. In this case, teach me about the game engine.

![Explainer section stating the goal of the change and explaining isometric projection.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-16.webp)

Second principle: **intuition before details.**

Before any code, it states the goal — “make the garden feel three-dimensional with 2D drawing tricks” — and explains related concepts, like what isometric projection is.

All of this builds my intuition for the essence of the change. It's catching me up as the human so I can be an equal participant in understanding.

[![

](/images/talks/understanding-bottleneck/poster-17.webp?1783098284)](/images/talks/understanding-bottleneck/video-17.mp4)

You can also build intuition with **interactive figures**.

Here I'm understanding the isometric perspective by dragging rocks around the garden and watching their coordinates move.

(This is using a new feature Notion just shipped: you can now embed interactive HTML inside pages.)

![Slide contrasting a raw diff with a literate diff structured as prose.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-17.webp)

We finally get to the code. But a typical diff is a pile of files edited in alphabetical order with no explanation.

A “literate diff” as I call it is structured as prose — walking through the changes in a sensible order, with surrounding explanation and embedded code snippets. Faster to review than a raw diff.

![Photo of a printed code explainer packet at a café.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-18.webp)

The end result of all of this is a nice explainer packet. I still read the code diff but I always read this first.

Sometimes I'll print these out and take them to the café — less distracting.

It's beautifully ironic: AI turns an interactive activity into a static paper report I can focus on deeply :)

![Slide quoting Andy Matuschak: books don't work. Screenshot of Quantum Country.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-19.webp)

There's only one problem: reading is hard work 😅

As [Andy Matuschak says: “books don't work”](https://andymatuschak.org/books/)! It's too easy to fool yourself into thinking you did the reading when you really didn't retain or understand.

How do we fix this? I took inspiration from Andy and Michael Nielsen's work on [embedding spaced repetition quizzes in essays](https://quantum.country/).

[![

](/images/talks/understanding-bottleneck/poster-21.webp?1783098284)](/images/talks/understanding-bottleneck/video-21.mp4)

I do something similar with my code explainers now. At the bottom of an explainer there's an interactive quiz — five questions about the change — and I try to answer them.

My rule: I won't send code to others until I can pass the quiz, and I do the same when reviewing others' code.

![Slide describing the quiz as a speed regulator on the AI loop.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-20.webp)

**A quiz is a speed regulator.** Working with AI, it's easy for the loop to run faster than the speed of human understanding.

The quiz is a counterbalancing force: I mechanically ask “do I actually understand?” so that I can remain a full creative participant.

![QR code linking to the /explain-diff skill.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-21.webp)

OK, so that's explain-diff. [Here's the skill if you want it](https://gist.github.com/geoffreylitt/a29df1b5f9865506e8952488eac3d524): two variants that output either HTML or a Notion page.

## Technique 2: Micro-worlds

![Slide introducing micro-worlds, with a photo of Seymour Papert.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-22.webp)

Next idea: micro-worlds. This one's inspired by the visionary educator Seymour Papert.

![Slide about Papert's idea of living in Mathland.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-23.webp)

Papert had this beautiful idea he called *living in Mathland*: if you want to learn math, live in Mathland — just like if you want to learn French, you go live in France. Could we build an environment where children learn math naturally, as a consequence of their curiosity?

So how do we apply that to code? **Can we make worlds you inhabit and naturally intuit how the system works and how it's changing?**

[![

](/images/talks/understanding-bottleneck/poster-27.webp?1783098284)](/images/talks/understanding-bottleneck/video-27.mp4)

Last year I was coding a Prolog interpreter and struggling to intuit what was happening inside.

I worked with an agent to build this debugger, which let me step through the execution of my logic language — scrub through time, see what's on the stack and which rules are evaluated at each step. I could even leave comments for myself (“nice, we correctly applied that rule”).

There's a big difference between making a tool *for me* to debug and letting the agent debug — doing it myself is how I develop understanding along the way.

[![

](/images/talks/understanding-bottleneck/poster-28.webp?1783098284)](/images/talks/understanding-bottleneck/video-28.mp4)

Another example. I was migrating my personal website from one framework to another, and Claude wrote a script that did it. But it was very hard to review: I wasn't familiar with the new framework, and all I could say was “I guess that looks about right.”

So I asked Claude to make me a video game — a command center where I do the port myself, step by step, watching the visible effects and the file tree evolve. It produced a UI where I click buttons to run the port step by step, with my old site and new site running side by side.

In this command center I watched the new site come to life incrementally. That left me with a similar understanding to doing it by hand — but much faster, because the whole experience was laid out for me.

![Slide reading: agents can write code to help us understand code!](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-24.webp)

The point here is that agents can write bits of code that help us humans understand other code.

This is a big deal!

## Technique 3: Shared spaces

![Slide introducing shared spaces: understanding together as a team.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-25.webp)

Alright, last technique: shared spaces. So far this has all been about understanding solo… but **when you're working on a team, you need to understand together.**

![Slide about shared mental models enabling efficient communication.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-26.webp)

When you and someone else hold the same mental model, you can communicate efficiently. You have a shared vocabulary that evokes the same images, so you can jam and riff and have creative conversations. Without those shared structures, those conversations are much harder.

I'm really excited about creating shared environments where teams build that understanding together. It's kinda what Notion is all about too.

![Screenshot of Claude and Cursor agents running inside Notion.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-27.webp)

Recently in Notion we've been shipping tons of new features for humans and agents to work together, so your whole team develops a shared understanding instead of each working in a silo.

One tiny example: you can now run Claude and Cursor agents in Notion. I do a lot of my coding that way now.

And when those agents make a technical plan in Notion, it's in a collaborative page by default, so I can comment on it with my team and discuss immediately. Thinking together, not alone!

## The point was always to augment

![Slide: it's still important for humans to understand how things work.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-28.webp)

Alright, let's wrap up. Today we've covered some techniques that were about understanding code… but actually I think this is a much bigger issue.

It's still important for humans to understand how things work *in general*! **Not just to verify, but to participate.**

And surprise surprise, this is not a new idea. It harkens back to the very origins of our field of computing…

![Alan Kay's vision: kids learning physics by playing and editing an interactive simulation.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-29.webp)

50 years ago Alan Kay envisioned that computers could be a new medium, better than the book, for teaching people — especially kids — how to think about the world.

In this picture, it might look like these kids are watching YouTube on an iPad, but they're not. They're playing an interactive game and editing the code as they play it to get a better understanding of physics. This was 50 years ago!!

![Astronaut meme: wait, the point of computers is to create new dynamic simulations to help people understand complex concepts? Always has been.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-30.webp)

And now hopefully you understand [this meme](https://x.com/geoffreylitt/status/2071362040346955777).

The point was always to *augment*, not just automate.

It's beautiful that AI now makes creating simulations so accessible… Having AI teach us is one of the greatest possibilities computing has ever opened up.

![Closing slide: we can get deeper in the loop. It's up to us.](../../assets/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck/2026-07-12-geoffrey-litt-understanding-is-the-new-bottleneck-31.webp)

This makes me very optimistic about the future!

**If we build the right tools, we can now understand the world better than we ever could before.** We don't have to merely take ourselves out of the loop, we can get *deeper in the loop* too. It's up to us.

*FIN*

## Related reads

If you enjoyed this talk, you might like these other posts I've written about human-AI collaboration:

- [Enough AI copilots! We need AI HUDs](/2025/07/27/enough-ai-copilots-we-need-ai-huds)
  — "anyone serious about designing for AI should consider non-copilot form factors that more directly extend the human mind..."
- [AI-generated tools can make programming more fun](/2024/12/22/making-programming-more-fun-with-an-ai-generated-debugger)
  — "Instead, I used AI to build a custom debugger UI… which made it more fun for me to do the coding myself..."
- [Code like a surgeon](/2025/10/24/code-like-a-surgeon)
  — "identify and delegate the secondary grunt work tasks, so you can focus on the main thing that matters."
