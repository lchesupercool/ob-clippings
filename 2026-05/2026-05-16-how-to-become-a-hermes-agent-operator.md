---
title: "How to Become a Hermes Agent Operator"
source: "https://x.com/shannholmberg/status/2055335043904492011?s=52"
author: "Shann³ (@shannholmberg)"
saved: "2026-05-16 09:46:22 +0800"
type: "x-twitter-longform"
---

# How to Become a Hermes Agent Operator

Author: Shann³ (@shannholmberg)  
Source: [https://x.com/shannholmberg/status/2055335043904492011?s=52](https://x.com/shannholmberg/status/2055335043904492011?s=52)  
Saved: 2026-05-16 09:46:22 +0800

![](../../assets/how-to-become-a-hermes-agent-operator/01.jpg)

How to Become a Hermes Agent Operator

learn how to operate and master Hermes Agent. set up the agent control room template, configure specialist agents, and grow from one agent to a whole marketing company on one VPS.

most AI tools answer questions. Hermes agent runs your workflows end-to-end.

it navigates your browser, executes terminal commands, schedules cron jobs, monitors your inboxes, drafts the work, and posts the result to wherever you live: telegram, discord, slack, the email thread you're in right now.

built by

@NousResearch

and it's open source with 150,000 github stars. Currently ranks #1 on OpenRouter for global token usage.

its the framework I have built my entire marketing operation around for the past few weeks, and the article you are about to read is how I would set it up if I were starting today.

![](../../assets/how-to-become-a-hermes-agent-operator/02.jpg)


## what you'll get from this article

what hermes agent is and why marketers (not just developers) should care

the reader-friendly version of the architecture: the brain, the personality, the skillset, and how they all live in one folder

the use cases I am personally running on hermes, with the four posts I have published about them

the four-part mental model (you, control room, agents, optional task bus) and the four levels of setup, from "one agent on your laptop" to "a fully automated agent team on a VPS that you control from your phone"

the prototype → production methodology I use to take a marketing workflow from messy idea to autonomous deployment

the resources I would have wanted on day one: docs, the community atlas, the people to follow, the meetups happening right now

the honest trade-offs and where this still breaks

I am not selling you anything in this article. hermes is open source, Nous Portal has a free tier, and most of the community ecosystem is free too. fork, change, make it yours.


## what hermes agent is

short version:

an autonomous agent that gets more capable the longer it runs.

longer version:

hermes is a framework built by Nous Research that turns a model into a persistent operator. it has its own memory that survives between sessions. it writes its own skills as it works. it ships with 123 skills already built in (github workflows, obsidian, google workspace, linear, notion, typefully, perplexity, deep research, plus 100+ more). it lives wherever you put it, on your laptop, in a docker container, on a VPS, in a serverless runtime. and you can talk to it through 20+ surfaces: telegram, discord, slack, email, voice mode, or just your terminal.

5月13日

Hermes Agent changed how I work it's the highest leverage agent framework you can set up right now what makes it different: > it routes tasks to the right model based on complexity and cost > learns your voice and preferences over time > handles context switching without

![](../../assets/how-to-become-a-hermes-agent-operator/03.jpg)

5月10日

1:51

Nate Herk just wrote the most thorough Hermes Agent setup guide on the internet Here are 12 lessons you need to know before you build your own

x.com/19216745449994…

46

97

710

24万

if you have used claude code or openclaw, hermes is the same shape with a different philosophy.

hermes is rails. opinionated defaults, batteries included, productive on day one with minimal setup, the agent does more thinking for you.

openclaw is linux. primitives, guarantees, explicit control, the agent does exactly what you told it to and nothing more.

both are valid. I run hermes because the bundled defaults compound. every project I start with hermes is one where the agent already knows how to do 100+ things before I write a line of configuration. that head start is worth it for me. I´ve also noticed that hermes doesn´t have where near the same issue with gateway disconnecting or bugging.

proof is in the numbers Nous Research just hit:

#1 on OpenRouter

for global token usage (out of every model and framework on the platform)

150,000 github stars

on the hermes repo

123 bundled skills

before the agent writes one of its own

70+ built-in tools

in the gateway, plus 300+ models through one subscription

6 deployment targets

: local, docker, ssh, daytona, singularity, modal

20+ messaging surfaces

: telegram, discord, slack, email, voice

if you are an AI marketer and you have not started running hermes yet, you are leaving compounding capability on the table every week.


## how it works (the reader-friendly version)

every hermes agent has three things.

a brain.

memory lives at ~/.hermes/memories/. two files, MEMORY.md and USER.md, inject at session start. your voice rubric, your brand notes, your customer language, last week's corrections, all of it loads before the first prompt. sessions are stored in sqlite, recall across sessions is full-text searchable.

a personality.

soul.md is where the vibe lives. concise. sarcastic. blunt. formal. fast or thoughtful. you can spin up six agents and give each one a different soul, same brain underneath. one is your outbound rep with a closer's energy. another is your researcher who likes long sentences. another is your assistant who keeps everything short.

![](../../assets/how-to-become-a-hermes-agent-operator/04.jpg)

a skillset.

123 skills out of the box: github PRs, obsidian, google workspace, linear, notion, typefully, perplexity, deep research, browser control, web scraping, vision, voice, scheduling. and the closed learning loop: as the agent works, it writes new skills along the way. your own skills library grows on top of the 123 without you having to write any of them.

then there is what the agent can talk to.

the

tool gateway

: one subscription, 300+ models, plus web scraping and browser automation built in

MCP integration

: any external service that speaks Model Context Protocol becomes a tool your agent can use

20+ messaging surfaces

: telegram, discord, slack, email, voice, plus the CLI itself

![](../../assets/how-to-become-a-hermes-agent-operator/05.jpg)

and where the agent can live.

your laptop (local)

a docker container (isolated, portable, the way I run mine)

an ssh session on a VPS (so it runs even when your laptop is closed)

daytona, singularity, modal (serverless if you don't want to manage infrastructure)

the closed learning loop is what makes this different from a smart chatbot. the agent watches itself work, writes new skills as it learns the shape of your work, refines its memory periodically, and recalls past context across sessions using a mix of full-text search and LLM summarization. you do not have to re-teach it next week.

the rule I tell people new to hermes is this: do not try to write your own skills on day one. run real work, let the agent watch, and let the harness write the skills. you build a custom skill library faster by working than by writing prompts.


## what I'm running on hermes

I am an AI marketer, not a coder. most of what I run on hermes is marketing infrastructure with the occasional internal tool. here is the actual list:

a personal assistant

that handles business and private, lives in telegram, flags the four emails worth reading every morning, schedules my reminders, summarizes meetings I missed

a marketing workflow prototyping bench

where I test new flows (lead magnet, ad creative review, content sprint) against real work for 2-3 runs before promoting them

specialized marketing agents

: SEO, outbound / BD, design review, content writing, each one with its own soul and its own scope

a company brain

that monitors slack, chats, emails, transcripts, voice memos, and makes all of it queryable. when I ask "what did we say to that client about pricing last month" I get the answer in 3 seconds instead of 30 minutes of digging

an SEO agent

that runs the full pipeline from keyword seed to published article in one docker container, 21 steps, no human in the middle until the final review

a content distribution agent

that takes a piece of long form (this article, for example) and atomizes it across LinkedIn, X, Threads, with platform-specific hooks

an orchestrator agent

that does not produce work itself, just routes requests to the right specialist based on what I'm asking for

the blueprint I posted that summarized it:

5月13日

the org chart for my Hermes Agent company four layers, all isolated docker containers on one vps: 1. company brain - vision, brand, customers, products. the context every other layer inherits 2. orchestrator hermes agent - reads the company brain, picks the right department,

![](../../assets/how-to-become-a-hermes-agent-operator/06.jpg)

5月13日

![](../../assets/how-to-become-a-hermes-agent-operator/07.jpg)

Hermes Agent changed how I work it's the highest leverage agent framework you can set up right now what makes it different: > it routes tasks to the right model based on complexity and cost > learns your voice and preferences over time > handles context switching without

x.com/14559993134531…

54

108

1,038

13万

the SEO agent in particular is worth zooming in on, because it is the one I have shipped publicly and the one that maps cleanest to the architecture in the rest of this article. five layers, all inside one docker container, 21 steps from keyword seed to published article.

the 21 steps look like this in the terminal:


[research + ideate]

- 01 keyword seed
- 02 serp snapshot
- 03 competitor extraction
- 04 intent + format analysis
- 05 content + visual gap
- 06 internal validation
- 07 external validation
[production]

- 08 angle + positioning brief
- 09 visual strategy brief
- 10 outline
- 11 draft
- 12 image gen
- 13 flowchart gen
- 14 visual qa
- 15 article qa
[distribution]

- 16 publish prep
- 17 schema
- 18 internal linking
- 19 syndication
- 20 analytics setup
- 21 monitoring
the layers above this pipeline:

company brain

at the top: vision, brand, audience, products. every agent reads from this

orchestrator hermes agent

: takes the topic or keyword seed and routes it to the seo agent

seo brain

: ranking playbook, voice rules, content formats, visual style guide, success criteria per format. all seo-specific context lives here

three sub-agents

inside the SEO agent, each one handling a phase:

research + ideate

: keyword seed, serp snapshot, competitor extraction, intent and format analysis, content and visual gap, internal and external validation

production

: angle and positioning brief, visual strategy brief, outline, draft, image gen, flowchart gen, visual and article qa

distribution

: publish prep, schema, internal linking, syndication, analytics, monitoring

one docker container

holds all three sub-agents. they share env, memory, and tools. sub-profiles switch context per phase. one process, one filesystem, one set of credentials.

why one container instead of three: seo work is sequential. research feeds the brief, the brief feeds production, production feeds distribution. every step needs memory of what was decided upstream. splitting into three containers means shuttling state across boundaries, which gets expensive and breaks the chain.

every other specialized agent in the company runs on the same template. clone the SEO agent template, swap the brain (seo brain → outbound brain, or → design brain, or → support brain), and you have a new agent for any function with the same five-layer shape.

5月15日

how my hermes seo agent works in the org chart it runs the full pipeline from keyword seed to published article, 21 steps, all inside one docker container the structure: LAYER 1: company brain shared context: vision, brand, audience, products. every agent reads from this

![](../../assets/how-to-become-a-hermes-agent-operator/08.jpg)

5月13日

![](../../assets/how-to-become-a-hermes-agent-operator/09.jpg)

the org chart for my Hermes Agent company four layers, all isolated docker containers on one vps: 1. company brain - vision, brand, customers, products. the context every other layer inherits 2. orchestrator hermes agent - reads the company brain, picks the right department,

x.com/14559993134531…

10

14

189

1.4万

the layers are not decoration. they are the reason the agent does not lose context as the work gets specialized. the company brain stays stable while the worker iterates. the brain layers make the worker disposable.

I also hosted Nous Research at our

@EspressioAI

HQ in Lisbon for a Hermes Agent evening recently.

@yeahfortommy

from Nous ran a Q&A, Simao from noticed .so showed an agent harness with autoresearch, and I walked through how we are using hermes for growth at Espressio.

4月27日

we are hosting

@NousResearch

for a Hermes Agent evening at the Espressio HQ tomorrow

@touchgrassklub

is organizing in collabration with Talent Protocol Here´s the agenda for the evening: > I´ll start by talking about using Hermes Agent for growth, what we're shipping at

![](../../assets/how-to-become-a-hermes-agent-operator/10.jpg)

![](../../assets/how-to-become-a-hermes-agent-operator/11.jpg)

11

4

46

if you are in Lisbon and want to come to the next one, I will post when it's scheduled.


## from one agent to a full fleet

before the levels, the mental model.

the setup has four parts:

you

are the operator. you have direct access to every part of the system.

the

agent control room

is the side control plane. it is not an agent you chat through. it is a folder at /root/vps-agents that documents and governs the whole fleet. you open it, edit it, inspect it, or ask claude, codex, or hermes to use it when you are managing the system.

the

hermes agents

are the workers. some are specialists (seo, dev, cmo, ops). one of them can optionally be an orchestrator.

the

agent task bus

is an optional handoff desk that sits between the orchestrator and the specialists. you only need it once you have an orchestrator in play.

the whole thing looks like this:


┌───────┐

│ YOU │ the operator

└───┬───┘

│

┌─────────────────────────────┼─────────────────────────────┐

│ │ │

control path orchestrated path direct path

│ │ │

▼ ▼ ▼

┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐

│ AGENT CONTROL ROOM │ │ HERMES │ │ SPECIALIST AGENT │

│ /root/vps-agents │ │ ORCHESTRATOR │ │ │

│ │ │ (optional door) │ │ seo · dev · cmo · │

│ docs · rules · │ └─────────┬──────────┘ │ ops · life │

│ runbooks · env-map │ │ delegates │ │

│ · registry │ ▼ │ talk to it │

│ │ ┌────────────────────┐ │ directly, │

│ side control plane │ │ AGENT TASK BUS │ │ no routing │

│ no raw secrets │ │ /srv/agent-bus │ │ │

│ │ └─────────┬──────────┘ │ │

└────────────────────┘ │ │ │

│ routes │ │

└───────────────▶ │

│ │

└────────────────────┘

the agent control room governs every agent in this diagram. it is the

single source of truth, and the place you go to manage the fleet, not

the place you go to run work through it.

the storage split matters more than people think:


/root/vps-agents → control room: docs, rules, runbooks, architecture

no raw secrets, ever

/srv/

<

agent-name

> 
/data → live runtime: secrets, memory, skills, sessions, crons

this is where each hermes agent lives

the control room contains the answers to questions like which agents exist, what they do, what ports they use, what credentials they reference, what each agent can and cannot do, and how to restart, debug, or rebuild any of them. the live runtime contains the actual workings.

the control room is the brain that defines the system. the live runtime is the body that runs it. you can rebuild the body from the brain. you cannot rebuild the brain from the body.

inside the control room:


/root/vps-agents/

README.md

CLAUDE.md

agents/

<

agent-name

> 
/

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

and inside each agent's runtime at /srv/<agent-name>/data/:


.env

config.yaml

SOUL.md

memories/

skills/

cron/

sessions/

logs/

state.db


## three ways you interact


control path:

you ──────► agent control room

(add agents, rotate keys, update docs, debug setup)

direct path:

you ──────► hermes-seo-espressio

(talk to a specialist directly, fastest)

orchestrated path:

you ──► hermes-orchestrator ──► task bus ──► specialists ──► you

(one front door, routes and synthesizes multi-agent work)

control path

is the meta layer. used when adding agents, reviewing docs, checking ports, rotating keys, debugging setup.

direct path

is the fastest. used when you already know which agent does the work.

orchestrated path

is the synthesizer. used when you want one front door that routes and combines work across multiple specialists.


## level 1: one agent

you have one hermes agent. that is it. the control room can still exist (recommended) but it only documents that one agent.

you → one hermes agent

control room → documents that one agent

best for: initial setup, your personal hermes, root install documentation, simple docker migration.

one agent, lived-in, with a personality you have tuned and a memory that has started to build. fill in SOUL.md with the voice you want, MEMORY.md with the stable facts about your business, and USER.md with the stable facts about you. connect it to telegram or discord so it lives where you do. start using it on real tasks. let it touch your tools. let it write its own skills along the way.

MEMORY.md holds the stable facts (what your business is, who your customers are, what your products do). USER.md holds the stable facts about you (timezone, working hours, recurring projects, preferred output formats). both get refined every week as you correct the agent in real conversations.


## level 2: direct specialist agents

you have multiple specialized agents, but you still talk to each one directly. no orchestrator yet.


you → hermes-life

you → hermes-seo-espressio

you → hermes-dev

you → hermes-cmo

the control room documents all of them.

best for: clear role separation, testing which agents are useful, avoiding premature orchestration, keeping credentials scoped per agent.

the trap to avoid here is reaching for an orchestrator before you have proven your specialists are useful. spin up two or three, run them directly, and only add an orchestrator when you find yourself wanting one front door.

when to spin up a new agent vs stay with what you have:


needs its own credentials → new agent

needs its own long-term memory → new agent

ongoing repeated work that is a separate role → new agent

otherwise stay with what you have

bad pattern: one mega-agent with every credential and every memory layer mashed together. you lose isolation, you lose the ability to revoke access cleanly, and the agent gets confused about which voice to use.


## level 3: orchestrator + specialists

you add hermes-orchestrator as a front door. you can still talk directly to specialists, but the orchestrator can route work and synthesize results.

![](../../assets/how-to-become-a-hermes-agent-operator/12.jpg)

the orchestrator reads the control room to know which agents exist, what each one does, where task queues live, what requires approval, which actions are forbidden, and where docs and runbooks are. it does not need to ask you any of that, it reads it.

best for: cross-functional work, delegation, summary and synthesis, one main interface for multi-agent workflows.

the orchestrator is the moment your setup stops being a collection of agents and starts being a team. it is also the moment where the control room earns its keep, because the orchestrator is only as good as the docs it reads.

what a quick check-in on the fleet looks like from my laptop or phone:


$ ssh hermes

welcome to hermes-vps-1.

last login: thu may 15 09:14:22

hermes-vps-1

~

$ cd vps-agents

hermes-vps-1

~

/vps-agents $ docker ps --format \

"table {{.Names}}\t{{.Status}}\t{{.Image}}"

NAMES STATUS IMAGE

hermes-orchestrator up 14 hours hermes-runtime

hermes-seo-espressio up 8 hours hermes-runtime

hermes-cmo up 8 hours hermes-runtime

hermes-outbound up 4 hours hermes-runtime

hermes-life up 12 hours hermes-runtime

hermes-vps-1 ~/vps-agents $ cat agents/hermes-seo-espressio/runbook.md

#

runbook: hermes-seo-espressio

restart: docker compose restart hermes-seo-espressio

logs: docker logs -f hermes-seo-espressio

shell: docker exec -it hermes-seo-espressio bash

...

5月13日

my entire Hermes Agent setup is controlled from one folder on a VPS I can manage it from my laptop or phone in under 10 seconds, spin up isolated agents per project, and never lose context here's the full setup: > bash command "ssh hermes" auto-connects to the VPS > session

![](../../assets/how-to-become-a-hermes-agent-operator/13.jpg)

5月13日

![](../../assets/how-to-become-a-hermes-agent-operator/07.jpg)

Hermes Agent changed how I work it's the highest leverage agent framework you can set up right now what makes it different: > it routes tasks to the right model based on complexity and cost > learns your voice and preferences over time > handles context switching without

x.com/14559993134531…

60

121

1,169

9.7万


## level 4: automated agent team

ame shape as level 3, but with recurring workflows and stronger automation. weekly seo reports run on cron. server health checks fire daily. backup verification runs without you asking. cross-agent business workflows kick off on schedule.

best for: weekly seo reports, content operations, server health checks, backup verification, cross-agent business workflows.

level 4 is what a marketing department in your terminal looks like. it does not need you to start the day. it shows up to work on its own, files reports, checks itself, and only pings you for the decisions that need taste.

![](../../assets/how-to-become-a-hermes-agent-operator/14.png)


## the control hierarchy

one principle to keep in your head as you climb the levels.

the control room is for config, docs, runbooks, and governance. it documents which agents exist, what they do, where they run, which credentials they reference, what each agent can and cannot do. it is the admin panel for the fleet, including the orchestrator. it is not where you go to do work.

for work, you talk to the agents directly. either a specialist (when you know which agent owns the job) or the orchestrator (when you want one front door to route across specialists).


## the setup guide: point your agent at the repo

now you understand the architecture. here is how you build it.

I shipped a public template that holds the exact structure described above, plus the skills your agent needs to set it up for you.

it lives at

github.com/shannhk/hermes-agent-control-room

.

![](../../assets/how-to-become-a-hermes-agent-operator/15.jpg)

you can clone it manually, but the point is that you do not have to. if you have claude code or codex on your laptop, the agents do most of the work after you hand over a Hetzner API key.

the automated flow:


you ──► generate a Hetzner API key

(5 min: sign up, generate a token, drop it in your .env)

│

▼

agent ──► create-vps skill

spins up a Hetzner box, generates an SSH key,

writes the alias to ~/.ssh/config so

`ssh hermes`

works

│

▼

agent ──► setup-control-room skill

installs Node, Docker, Claude Code, Codex CLI,

Hermes Agent, then clones the repo to the VPS

at /root/agent-control-room

│

▼

you ──► finish interactive auth on the VPS

(claude /login, codex, hermes)

│

▼

agent ──► agent-control-room skill

registers your first hermes agent in the docs,

fills in the runbook, sets up the env-map

│

▼

you are at level 1 with a documented agent

within ten to fifteen minutes you have:

a fresh Hetzner VPS with the right tooling installed

the control room cloned at /root/agent-control-room on the VPS

the bundled skills linked into ~/.claude/skills on the VPS

one hermes agent registered, runbook filled in, env-map written

an SSH alias on your laptop so ssh hermes connects instantly


## the prototype → production methodology

most workflows do not start as production ones. they start messy. a flow that runs SEO research, drafts an article, schedules it in Typefully, and posts it to LinkedIn does not exist in your head fully formed. you discover it by running it.

hermes is the prototyping environment for this. here is the four-step path I use to take any new marketing workflow from idea to autonomous deployment:

prototype in hermes

. open your main hermes agent, describe what you want to happen, and let it try. it will get most of it wrong on the first run. that's fine.

run it 2-3 times against real work

, correcting drift each time. the harness watches every correction and starts writing the skill as it learns the shape. by run three the agent is doing most of what you want without coaching.

fine-tune in a dedicated workspace

. pull the workflow into a separate Claude Code workspace (or a fresh hermes agent if you prefer), tighten the prompts, lock the routing, add error handling, decide what should run on cron and what should be triggered.

deploy to a VPS on a schedule

. once it survives a week of real runs without you babysitting it, push it to its own docker container on your VPS, set the cron, walk away.

I learned this pattern after burning a few weekends trying to write production-ready agents from scratch. you cannot write a production agent from scratch. you have to grow one. hermes makes the growing part fast.

![](../../assets/how-to-become-a-hermes-agent-operator/16.png)

prototype in hermes

fine-tune in a dedicated workspace

deploy autonomous on a VPS


## the models I run on hermes

hermes gives you the framework. the model underneath is your choice. through the tool gateway you can route to 300+ models from one subscription, switching per agent or per task.

what I personally run today:

claude opus 4.7

for the creative work: copywriting, voice, hook generation, content drafting, anything where taste and writing quality matter

codex (gpt 5.5)

for the structured work: coding, planning, multi-step workflows, browser automation, scraping, anything where the steps need to be tight and the output predictable

I run both. opus writes. codex builds and plans. hermes makes routing easy, you point each agent at the model that fits the work it does.

if you can only run one, the answer depends on what kind of work your fleet is doing. heavy on content and copy? start with claude opus 4.7. heavy on infrastructure, automation, and engineering workflows? start with codex. you can always add the second model later through the same tool gateway.


## honest trade-offs

I am not going to pretend hermes is perfect. three real trade-offs.

1. the bundled defaults are also

opinions.

hermes

ships with strong defaults for how memory works, how skills get written, how the agent uses tools. that is the whole pitch. but it also means if you want primitives with explicit control over every step, hermes will feel heavy. openclaw is the better fit for that taste. pick the tool that matches your philosophy.

2. level 3 and 4 have a real learning curve.

docker, VPS, SSH, the control room folder structure, the orchestrator skills, none of this is "install and go." you should not jump to level 3 if you are not already running hermes at level 1 daily.

3. the model still

matters.

hermes

is a framework that makes a good model great. it does not make a small model into a strategist. use the strongest models you can afford for the work that matters (your orchestrator, your strategy agent, your brain). drop to cheaper models for the work that does not (research scraping, draft generation, batch processing).

none of this is magic. it is a framework that pays back because the memory persists, the skills accumulate, and the agents stay scoped. apply it to the wrong-sized model and you get a confused team. apply it to the right one and you get a team.


## resources

if you are starting today, here is what I would read in order.

the official docs

:

hermes-agent.nousresearch.com/docs

. start with the install guide, then read the skills page so you understand what ships out of the box

the control room template (my repo)

:

github.com/shannhk/hermes-agent-control-room

. the exact structure I described above, ready to clone. control-room-first template for managing hermes agents from one VPS agent to specialist teams and orchestrated workflows. fork it and make it yours

hermesatlas.com

: the community-curated map of 100+ open source tools, plugins, workspaces, and integrations built on hermes. categorized by domain (memory providers, workspaces, skill registries, deployment, orchestration). also includes the Hermes Handbook, a beginner-friendly walkthrough. weekly updates, free newsletter

@Teknium

on X

: Nous Research founder. ships hermes updates almost daily. the codex runtime integration, the DeepSeek V4 Flash free tier on Nous Portal, the pretext skills, all came through his feed first

@NousResearch

on X

: the official account, official feature announcements

the meetups

: there are hermes meetups happening in person now (Lisbon, Ventura, more cities). worth showing up if there's one near you. you learn more in 90 minutes of side conversations than in a week of reading

![](../../assets/how-to-become-a-hermes-agent-operator/17.jpg)

Hope you got some value out of this, appreciate you reading the whole thing.

-- Shann

上午1:10 · 2026年5月16日

813

查看 9 条回复
