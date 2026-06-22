---
title: "I think Anthropic and OpenAI have found product-market fit"
source: "https://simonwillison.net/2026/May/27/product-market-fit/"
author: "Simon Willison"
published: "2026-05-27"
saved: "2026-05-28"
tags: [clipping, ai, product-market-fit, llm]
---

# I think Anthropic and OpenAI have found product-market fit

27th May 2026

Anthropic are [strongly rumored](https://techcrunch.com/2026/05/20/anthropic-says-its-about-to-have-its-first-profitable-quarter/) to be about to have their first profitable quarter. Stories [are circulating](https://www.theinformation.com/newsletters/applied-ai/uber-cto-shows-claude-code-can-blow-ai-budgets) of companies surprised at how expensive their LLM bills are becoming from usage by their staff. I think this is because OpenAI and Anthropic have both found product-market fit.

- [Enterprise customers are now paying API prices](https://simonwillison.net/2026/May/27/product-market-fit/#enterprise-customers-are-now-paying-api-prices)

- [I think they’ve found product-market fit](https://simonwillison.net/2026/May/27/product-market-fit/#i-think-they-ve-found-product-market-fit)

- [And they’re ramping up](https://simonwillison.net/2026/May/27/product-market-fit/#and-they-re-ramping-up)

- [The AI-failure stories around this are pretty thin](https://simonwillison.net/2026/May/27/product-market-fit/#the-ai-failure-stories-around-this-are-pretty-thin)

- [We also know the labs are spending a lot](https://simonwillison.net/2026/May/27/product-market-fit/#we-also-know-the-labs-are-spending-a-lot)

- [API revenue is becoming less important](https://simonwillison.net/2026/May/27/product-market-fit/#api-revenue-is-becoming-less-important)

- [April is a new inflection point](https://simonwillison.net/2026/May/27/product-market-fit/#april-is-a-new-inflection-point)

## Enterprise customers are now paying API prices

I currently subscribe to the $100/month Max plan from Anthropic and the $100/month Pro plan from OpenAI. If you are a heavy user of coding agents these plans are a fantastic deal. I just ran the [ccusage](https://github.com/ryoppippi/ccusage) tool on my laptop to get an estimate of how much I would have spent if I were to pay for API tokens in the past 30 days and got:

- $1,199.79 for Anthropic Claude Code

- $980.37 for OpenAI Codex

That’s $2,180.16 worth of tokens for $200—not bad at all! I’m a moderately heavy user of these tools, but I’m certainly not running agents every hour of the day and night.

I had assumed that companies making extensive use of agents were getting similar discounts. It turns out I *could not have been more wrong* about that.

I haven’t been able to track down the exact date, but at some point in the last six months Anthropic switched their Enterprise plan (originally [“Claude seats include enough usage for a typical workday” back in August 2025](https://www.anthropic.com/news/claude-code-on-team-and-enterprise)) to $20/seat/month plus API pricing for usage. This story about the change [from The Information](https://www.theinformation.com/articles/anthropic-changes-pricing-bill-firms-based-ai-use-amid-compute-crunch) is dated Apr 14, 2026, but cites an Anthropic spokesperson claiming that the pricing change occurred in November 2025. Existing customers are finding out about the change as they renew their contracts.

OpenAI made a similar pricing change in April. The [Codex rate card](https://help.openai.com/en/articles/20001106-codex-rate-card) ([Internet Archive copy](https://web.archive.org/web/20260519062438/https://help.openai.com/en/articles/20001106-codex-rate-card)) currently says:

> **Note**: On April 2, 2026, we updated Codex pricing to align with API token usage, instead of per-message pricing. This change was applicable to new and existing Plus, Pro, ChatGPT Business and new ChatGPT Enterprise plans.

> On April 23, 2026, we made this update for all existing ChatGPT Enterprise plans as well, inclusive of Edu, Health, Gov, and ChatGPT for Teachers.

It’s a little harder to decode as they quote prices in “credits”, but as far as I can tell those credit costs are an exact match for the API token costs listed for those models.

All of which is to say that as of April 2026 the “Enterprise” cost for both OpenAI Codex and Anthropic Claude Code/Cowork is the same as the listed API price.

GPT-5.5 (released April 23rd) is 2x the API price of GPT-5.4. Opus 4.7 (April 16th) is [around 1.4x](https://simonwillison.net/2026/Apr/20/claude-token-counts/) the price of Opus 4.6 when you take their new tokenizer into account.

So April saw both leading model companies release new frontier models with a higher API price, *and* both companies now have measures to lock their enterprise customers (who tend to sign year-long deals) at those API prices, not the previous extreme discounts.

## I think they’ve found product-market fit

Why these sudden aggressive moves on pricing? Both Anthropic and OpenAI are planning to IPO, but I suspect there’s a more important factor here: I think they’ve finally found product-market fit, with the coding/general-purpose agent products embodied by Claude Code/Cowork and Codex.

Tools like ChatGPT are wildly popular, but that wild popularity has been difficult to turn into revenue. In February [OpenAI boasted](https://finance.yahoo.com/news/chatgpt-almost-1-billion-weekly-212157499.html) more than 900 million weekly active users for ChatGPT, but only 50 million—5.6% of that—were paying consumer subscribers.

Charging $10-$20/month per user is an OK business, but you’d need 1-2 billion subscribers sticking around for four years to cover [$1 trillion in infrastructure](https://openai.com/global-affairs/seizing-the-ai-opportunity/).

Companies spending $200+/month/user will get you there a whole lot faster—and as noted above, as a power-user I’m at ~$1,000/month in API costs per vendor already.

Coding agents really did change everything. These are tools which burn *vastly* more tokens, but are also quickly becoming daily drivers for the work carried out by extremely well-compensated professionals. Right now that’s still mostly software engineers, but a coding agent is a tool that can automate anything you can do by typing commands into a computer... so they are clearly applicable to a much wider set of skilled knowledge workers.

As I’ve [discussed on this site at length](https://simonwillison.net/tags/november-2025-inflection/), the models released in November 2025 elevated agents to being genuinely useful. We’ve had six months to get used to that idea now—it’s no wonder companies are beginning to spend real money on this technology.

You could argue that ChatGPT achieved product-market fit when it became the [fastest-growing consumer app in history](https://www.reuters.com/technology/chatgpt-sets-record-fastest-growing-user-base-analyst-note-2023-02-01/) back in February 2023... but it certainly wasn’t making any actual money back then. Coding agents plus enterprise pricing marks the point when these companies start making *very* real revenue. Maybe even enough to start covering their costs!

## And they’re ramping up

As further evidence that enterprise agents represent product-market fit for these companies, consider their open job listings.

OpenAI have [703 open jobs](https://openai.com/careers/search/) right now, of which I’d categorize 229 (32.6%) as relating to enterprise sales and support—account executives, “Go To Market”, “Forward Deployed Engineers” and the like.

Anthropic have [390 open jobs](https://www.anthropic.com/careers/jobs), 105 (26.9%) of which look enterprisey to me.

It’s pleasingly ironic that these AI labs have picked a business model with such a heavy demand on human labor—enterprise sales contracts don’t close themselves without a whole lot of humans in the mix!

(I ran this analysis by scraping their job sites with Claude Code, then having it use Datasette’s [JSON API](https://docs.datasette.io/en/latest/json_api.html) to pipe that data into Datasette Cloud where I used [Datasette Agent](https://agent.datasette.io/) for the analysis, [exported here](https://gist.github.com/simonw/5632d208d76b3c8b34f1fdbaf69eb1b8#agent-4). Dogfood!)

## The AI-failure stories around this are pretty thin

I started digging into this in response to [a growing volume](https://news.ycombinator.com/item?id=48287025#48287219) of stories claiming that large companies were sounding the alarm because their AI usage costs had grown so large.

The most widely cited of these stories appear quite overblown to me.

The most discussed has been Uber, based on [this report](https://www.theinformation.com/newsletters/applied-ai/uber-cto-shows-claude-code-can-blow-ai-budgets) where CTO Praveen Neppalli Naga indicated that Uber had “maxed out its full year AI budget just a few months into 2026”, mostly thanks to Claude Code.

Given that Claude Code only got *really* good in November it’s entirely unsurprising to me that a budget set in 2025 may have failed to predict demand for that tool in 2026!

That Uber story was further fueled by comments made by Uber’s COO, Andrew Macdonald, on the Rapid Response podcast. I tracked down [the segment](https://www.youtube.com/watch?v=y_mQ6xLcKyc&t=1616s) and there really isn’t much there. Here’s what Andrew said:

> But then you sometimes go and talk to your senior engineering leaders and you’re saying, OK, how many projects that were on the cutting room floor got moved above the line because of the productivity gains because 25% of our code commits were via Claude Code last quarter?

> That link is not there yet, right? I think maybe implicitly there’s more that is getting shipped. But it’s very hard to draw a line between one of those stats and, OK, now we’re actually producing like 25% more useful consumer features, right? And that line is hard to draw.

Somehow this fragment turned into headlines like [Uber’s COO says it’s getting harder to justify the money spent on AI tokenmaxxing](https://www.businessinsider.com/uber-coo-andrew-macdonald-ai-token-spending-harder-justify-2026-5), because the market for stories about AI failures remains enormous.

The other popular story around this is [Microsoft starts canceling Claude Code licenses](https://www.theverge.com/tech/930447/microsoft-claude-code-discontinued-notepad), ostensibly to encourage their engineers to dogfood their own Copilot CLI agent instead—but The Verge reporter Tom Warren says “sources tell me the decision is also a financial one”, triggered by the June 30th end of Microsoft’s financial year.

I think both of these stories support my “product-market fit” hypothesis. The best advice I ever heard on pricing a product was that your customer should *suck air through their teeth* and then say yes. Uber’s budget overrun and Microsoft’s seat cancellations look like that effect playing out in practice.

## We also know the labs are spending a lot

The big AI labs spend billions of dollars on both training and inference. Credible figures are hard to come by, but we did get one huge hint as to the figures involved from, oddly enough, the recent [SpaceX S-1](https://www.sec.gov/Archives/edgar/data/1181412/000162828026036936/spaceexplorationtechnologi.htm):

> [...] in May 2026, we entered into **Cloud Services Agreements with Anthropic PBC** (“Anthropic”), an AI research and development public benefit corporation, with respect to access to **compute capacity across COLOSSUS and COLOSSUS II**. Pursuant to these agreements, the customer **has agreed to pay us $1.25 billion per month** through May 2029 [...]

The [Anthropic announcement](https://www.anthropic.com/news/higher-limits-spacex) said that this deal meant they could “increase our usage limits for Claude Code and the Claude API”, heavily implying that Colossus is being used for inference, not model training.

Anthropic already have vast amounts of compute from other providers. The fact that they’re willing to spend $1.25 billion per month for extra capacity from just *one* of their vendors hints at how big these inference budgets have become.

## API revenue is becoming less important

Over the past two years my impression has been that OpenAI made more of their income from subscription revenue while Anthropic made more from their API.

Anthropic’s API revenue was historically quite dependent on a small number of large API customers—[this VentureBeat story from August 2025](https://venturebeat.com/ai/anthropic-revenue-tied-to-two-customers-as-ai-pricing-war-threatens-margins) quotes “sources familiar with the matter” suggesting that just Cursor and GitHub Copilot were responsible for $1.2 billion of the company’s then-$4 billion revenue.

Today Anthropic are rumored to hit [$10.9 billion in the second quarter](https://www.wsj.com/tech/ai/mind-blowing-growth-is-about-to-propel-anthropic-into-its-first-profitable-quarter-7edbf2f4), potentially even operating at a profit for the first time.

This pivot-to-Enterprise suggests that the labs have realized that the real money lies in cutting out the middlemen. Anthropic’s Claude Code directly competes with Cursor and Copilot. No wonder Cursor are [investing in their own models](https://cursor.com/blog/composer-2)!

## April is a new inflection point

I’ve called November 2025 the [November inflection point](https://simonwillison.net/tags/november-2025-inflection/) because that was when GPT-5.1 and Opus 4.5, combined with their respective coding agent harnesses, got *good*—good enough that we’ve spent the last six months adapting to agent systems that can reliably get useful work done.

I think April 2026 is a new inflection point where the revenue implications of this have started to land, to the benefit of the frontier AI labs and with material impacts on the budgets of large companies.

We’ll know for sure how real this moment is when the S-1 documents for the upcoming Anthropic and OpenAI IPOs give us some real, audited numbers to get our teeth into.
