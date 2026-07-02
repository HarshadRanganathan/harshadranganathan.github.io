---
layout: post
title: "Gemini Implicit Caching: What It Actually Does, and the Part That Will Catch You Off Guard"
date: 2026-07-02
excerpt: "Gemini 2.5 Flash has a feature called implicit caching. Unlike explicit caching — where you manually define a cache key and TTL — implicit caching works automatically. Google's ser"
tag:
- Context Caching
- Gemini Caching
- LLM Cost Optimization
comments: true
---
# Gemini Implicit Caching: What It Actually Does, and the Part That Will Catch You Off Guard

Gemini 2.5 Flash has a feature called implicit caching. Unlike explicit caching — where you manually define a cache key and TTL — implicit caching works automatically. Google's servers detect when your API call shares a common prefix with a recent call from the same project, and bill the matched portion at a steep discount (~75% off). No configuration needed.

When I first read about it I thought: great, my agent sends the same long system prompt on every call, so I'll automatically save money on that part. That's basically true. But there's a part of how LLM agents work that makes implicit caching much less useful than it sounds, and understanding it changed how I structured my prompts.

---

## Why Caching Breaks After Turn 1

The implicit cache is keyed on the **exact API call prefix**. With an LLM agent framework like ADK, every model invocation sends:

```
[system prompt] + [conversation history] + [tool schemas] + [current user message]
```

Between turn 1 and turn 2, here's what changes:
- System prompt: unchanged ✓
- Tool schemas: unchanged ✓  
- Conversation history: **changed** — turn 1's response is now in history ✗
- Current user message: different ✗

The prefix diverges as soon as conversation history is added. Every turn after the first gets a cache miss on everything after the system prompt. The longer the conversation, the more tokens you're re-billing in full.

I saw this in the Vertex AI usage logs for a multi-cluster namespace query:

| Turn | Input tokens | Cached tokens | Effective token cost |
|---|---|---|---|
| 1 | 8,500 | 7,200 (system prompt cached) | 1,300 |
| 2 | 14,200 | 0 (history changed prefix) | 14,200 |
| 3 | 22,800 | 0 | 22,800 |

Turn 2 costs roughly 10× more than turn 1 in effective billed tokens. The tool results from turn 1 get fully re-billed. By turn 3 it's worse.

This is not a problem you can solve with caching alone. Caching saves you money on the static system prompt. It doesn't help with the history that keeps growing.

---

## What You Can Actually Cache: The System Prompt

The system prompt is the first thing in the API request. If it's long enough (Gemini requires a minimum of ~1,024 tokens for implicit caching to activate) and identical across calls from the same project, it'll be cached. My EKS agent's system prompt is 300+ lines — comfortably above the threshold. That portion is effectively cached across all concurrent users hitting the same agent.

That's genuinely useful. It's just not the whole picture.

---

{% include donate.html %}
{% include advertisement.html %}

## How to Maximize the Cache Hit Rate

**Rule 1: Never put dynamic content in the system prompt.**  
This is the most important one. If you inject the current date, the user's name, or any other per-call value into the system prompt, every call gets a different prefix and the cache never hits. Instead, inject dynamic context into the *user message*.

I built a cost analysis agent across this project that needed to know the current date to handle relative references like "last month" or "last 30 days". The tempting implementation:

---

## The Actual Bottleneck: History Growth

Here's the honest framing: implicit caching solves the system prompt cost. It doesn't solve the conversation history growth problem.

By turn 10 of a Kubernetes diagnostic session, you might have 80,000 tokens of tool results sitting in conversation history. Those get fully re-billed on every model call regardless of caching, because they're different every turn and will never match a cache prefix.

The real fix for that is **context compaction** — replacing accumulated conversation history with a compact LLM-generated summary after a set number of turns. I cover how to implement that with Google ADK in a separate post. But it's worth understanding the relationship: caching saves you money on the static parts (system prompt, tool schemas); compaction manages the cost of the dynamic parts (conversation history).

They solve different problems. You need both for a production agent running long sessions.

---

## Cost Explorer Agent: Date Context

The Cost Explorer agent (which queries AWS Cost Explorer API) needs to know the current date to resolve relative date references like "last month" or "last 30 days." This is dynamic context that *must not* go in the system prompt.

```python
# backend/agents/cost_explorer_agent/agent.py
# Inject the current date at invocation time, not in the static system prompt
```

The agent's system prompt instead says:
```
You will receive the current date in the user turn as:
CURRENT_DATE: <YYYY-MM-DD>
Use this as your authoritative reference for date calculations.
```

This keeps the system prompt prefix stable and cache-eligible, while still giving the agent accurate date context.

---

## The Short Version

- Implicit caching is real and useful — but only for the static prefix of your request (system prompt + tool schemas)
- Every turn, conversation history changes — breaking the prefix for everything after the system prompt
- Never put dynamic values in the system prompt; use the user message instead
- A longer, stable system prompt is more cache-eligible than a short one
- Caching doesn't solve history bloat; that requires compaction

