# Running an AI-native engineering org

> 来源: [Claude Blog](https://claude.com/blog/running-an-ai-native-engineering-org)
> 作者: Fiona Fung (Director of Engineering, Claude Code & Claude Cowork)
> 日期: 2026-06-03
> 场合: Code w/ Claude SF 2026

---

## The processes that quietly stopped working

For years, engineering bandwidth was the expensive part of building applications. Every process around software planning and shipping — first waterfall, then agile — was built around that cost.

On the Claude Code team, writing code, writing tests, and refactoring rarely slows us down anymore. But the bottlenecks didn't go away when agentic coding took away the actual need to type code. **Verification, code review, and security took their place.**

Key questions now: Is this code correct? How is it maintained? How are humans keeping up with code reviews?

## Planning: shift roadmaps to just in time

The old norm was spending a lot more time pre-planning because coding time was expensive. When Fiona joined the team, they wrote a 6-month roadmap — but because of Claude Code, so many things changed that it was out of date by month 3.

New approach: **JIT planning (Just-In-Time)** — like JIT compiling, do just the right amount at the right time. Shifted from design docs to discussions in PRs or prototypes. No lengthy product reviews — prototype, get internal users on it, act on feedback.

## Context gathering: ask Claude, not the author

Old norm: find the person who wrote the code. New norm: go deeper — what do you actually need to know? Ask Claude first, with data and context.

Also, always ask: **"Is there a way to automate it?"** Fiona's example: summarizing customer feedback channels went from a manual coffee ritual to an automated background task.

## Code review: trust but verify

Claude handles style/linting, PR feedback, bug catching, test additions. Humans stay for:
- Legal review / risk tolerance
- Trust boundaries and security-sensitive code
- Product sense and taste

The trust vs. verify balance **keeps changing** as models improve — continuously re-evaluate.

## Team makeup: blurring roles

PMs code, engineers do content and design. Two profiles they hire for:
1. **Creative builders with product sense** — dreamers passionate about shipping
2. **Engineers with deep systems expertise** — for the "trust but verify" parts

What they don't index on: raw throughput. "The more important question is where you still need human expertise."

## Core team principles (non-negotiable)

1. **Relentlessly dogfood**: Every team member uses Claude Code (and Claude Cowork)
2. **Keep the team flat**: Every manager starts as IC first, ships code, understands the engineer experience
3. **Kill processes that no longer work**: Explicit permission to question and remove obsolete processes

Within these, each pod has autonomy on how to use Claude for triage, planning rituals, etc.

## How to know new processes are sticking

Track three numbers:
- **Onboarding ramp time** ↓ — engineers ship real code within first week
- **PR cycle time** ↓ — might reveal where CI/build systems are struggling
- **Claude-assisted commits** ↑ — by default every commit is Claude-assisted

Don't confuse throughput with success. Throughput is one metric; the real metric is measuring the thing you're solving.

## Getting started

> Pick your noisiest workflow. Ask: is it still serving its purpose? If so, can you automate it?

Fiona's story: a weekly review meeting with everyone on laptops, only looking up for status reports. One question — "Why are we having this meeting?" — killed it.
