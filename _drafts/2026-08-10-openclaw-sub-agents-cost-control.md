---
layout: post
title: "OpenClaw Sub-Agents: Cutting AI Token Costs Without Cutting Capability"
date: 2026-08-10 09:00:00 -0500
categories: [openclaw, optimization, cost, selfhosted]
tags: [openclaw, sub-agents, cost-optimization, tokens, ai-infrastructure, llm]
---

I run a full-time AI agent on a VPS — it trades stocks, writes blog posts, checks news, manages a Gumroad shop, and monitors my email. Each morning it wakes up, checks market data, reads headlines, reviews positions, and decides whether to trade. That's a lot of API calls, and the bill adds up fast when every tool call runs on a premium reasoning model.

A few weeks ago I realized my per-trade cost was eating more in API tokens than the trades themselves were making. Not great for a paper-trading setup where the whole point is to practice disciplined decision-making, not to burn through credits.

The fix was counterintuitive: use a *worse* model for most of the work. Specifically, OpenClaw's `sessions_spawn` facility, which lets the main agent fork off a cheap "sub-agent" to do the grunt work — fetch prices, scan news, check the calendar — then report back with structured results. The expensive model only engages for the actual reasoning step: *should I trade or not*.

Here's how I set it up and what the numbers look like.

## The Problem: Expensive Tools

My primary model is a high-end reasoning model — the kind that costs $15-30 per million tokens. That's great for complex decisions like "is this news article a legitimate reason to trim a position, or is it noise?" It's terrible for "what was the closing price of AAPL yesterday?"

Every time the model calls `alpaca__get_stock_bars`, `alpaca__get_news`, or `alpaca__get_market_movers`, it burns tokens at the premium rate — input tokens for the API call itself, output tokens for the response, plus all the thinking tokens the model spent deciding *which* symbol to look up. A single price check that costs $0.001 in raw API usage can end up costing $0.05-0.10 after the model's reasoning overhead. Over dozens of checks per day, that's not trivial.

## The Fix: sessions_spawn

OpenClaw's `sessions_spawn` kicks off a new agent session as a sub-task. By default, it uses the system's cheaper model — in my case, a flash-tier model that costs about $0.30-0.50 per million tokens. That's 30-50x cheaper than the primary model.

The pattern is simple. In my agent's trading workflow, instead of doing:

```
1. (primary model) fetch news for watchlist symbols → $0.05
2. (primary model) get latest quotes for interesting symbols → $0.05  
3. (primary model) check market calendar → $0.03
4. (primary model) decide whether to trade → $0.15
Total: ~$0.28 per cycle
```

I do:

```
1. (sub-agent) fetch news, quotes, calendar, return as JSON → $0.005
2. (primary model) review the results and decide → $0.08
Total: ~$0.085 per cycle
```

The sub-agent is told exactly what to fetch and how to format it. No reasoning overhead, no back-and-forth. It fetches, formats, and returns. The primary model gets a clean, structured report and makes the call.

## The Code: What It Looks Like

In practice, the agent calls `sessions_spawn` with a task like:

> Check the market calendar for today to see if markets are open. If open, get the latest quotes for AAPL, MSFT, and GOOG. Also fetch any recent news for those symbols. Return everything as a structured summary.

The sub-agent runs its tools, compiles the result, and the primary session picks it up via a `sessions_yield` / completion event. The primary model never touches the price-checking tools directly.

The same pattern works for any tool-heavy task:

- **Email triage:** sub-agent checks inbox, returns "3 unread, 1 from your bank about a statement, 2 newsletters"
- **Portfolio review:** sub-agent fetches all positions, current prices, and day change, returns a formatted table
- **News scanning:** sub-agent fetches headlines for all watchlist symbols, filters for keywords like "lawsuit" or "upgrade"

## The Results

I've been running this for about two weeks. My average daily API cost dropped from about $1.80 to $0.55 — a 70% reduction. The trade decisions themselves haven't changed because the reasoning model still makes the final call with the same data it would have fetched itself. The only difference is that it gets that data pre-packaged instead of discovering it through its own tool calls.

The sub-agent runs on a flash model that's perfectly adequate for "run three tools and format the output." It doesn't need to *reason* about what it finds. It just needs to find it and hand it over.

## The Catch

Sub-agents are isolated sessions. They don't share memory, tools, or context with the primary agent unless you explicitly pass it. This is usually fine — you give the sub-agent a clear task and let it work — but it means the sub-agent can't use long-term memory or reference past decisions. If a task requires context from yesterday, the primary agent needs to include that context in the task prompt.

Also, `sessions_spawn` creates a new session each time, which has a small overhead — maybe 500-1000 tokens of system instructions. That's negligible compared to what you save, but worth knowing about if you're spawning dozens of them per cycle. For my use case (3-5 sub-agent tasks per day), it's a non-issue.

## When Not to Use It

This works well when the sub-agent task is well-scoped: "fetch X, filter Y, return Z." It does not work well when the task requires judgment — "evaluate whether this earnings report is bullish or bearish." That kind of reasoning is exactly what the premium model is for, and trying to offload it will just produce lower-quality results.

Rule of thumb: if you'd trust a junior to do it, give it to a sub-agent. If you'd want a senior's opinion, use your primary model.

---

*I run my entire agent stack on a single VPS with this sub-agent pattern wired into every automated workflow. The configs and automation scripts are included in the Self-Hosted AI Assistant Ops Kit at <https://brainscratch.gumroad.com/l/snjbhd>.*
