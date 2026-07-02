---
layout: post
title: "Context Compaction in Google ADK: Taming the Conversation History That Keeps Coming Back to Haunt You"
date: 2026-07-02
excerpt: "Here's the problem with multi-turn LLM agent sessions: every tool call, every model response, every step of reasoning gets appended to conversation history. And every subsequent mo"
tag:
- Conversation Compaction
- Google ADK
- LlmEventSummarizer
comments: true
---
# Context Compaction in Google ADK: Taming the Conversation History That Keeps Coming Back to Haunt You

Here's the problem with multi-turn LLM agent sessions: every tool call, every model response, every step of reasoning gets appended to conversation history. And every subsequent model call re-reads that entire history from the beginning.

In practice for my Kubernetes assistant, a three-turn diagnostic session might look like:

1. "List all clusters" → agent calls the MCP server 22 times, gets back JSON for each cluster
2. "Find failing pods in the production cluster" → agent calls MCP 5 more times
3. "Get logs for the crashing auth-service pod" → 2 more calls

By turn 4, every model call starts with ~15,000 tokens of previous tool results and reasoning text that gets re-read in full. By turn 8, it's 40,000+ tokens. This is what drives the `RESOURCE_EXHAUSTED` (429) errors on long sessions — not the context window, but the tokens-per-minute rate limit from re-reading accumulated history.

**Context compaction** solves this by replacing the accumulated history with a short, LLM-generated summary at regular intervals. You keep the findings, discard the raw tool response JSON that produced them.

---

## How It Works

Google ADK 1.18+ includes `EventsCompactionConfig`. When it fires (after a configurable number of turns), it:

1. Takes the entire accumulated conversation history
2. Sends it to an LLM with a summarization prompt
3. Replaces the raw history with the summary
4. The agent continues the conversation from there, with the summary as its "memory"

Vizually, the before and after:

```
Before compaction (turn 4 starts with):
  [22 cluster listings JSON] + [5 pod listing JSON] + [2 log snippets JSON] + [prose] = 15,000 tokens

After compaction:
  "The session found 22 clusters. The production cluster (pes-prod-eks-cluster)
   has 3 failing pods: auth-service (CrashLoopBackOff, OOMKilled),
   payment-worker (ImagePullBackOff — ECR access issues),
   cache-warmer (restartCount=47). Investigated auth-service logs:
   NPE in JVM heap allocation." = ~250 tokens
```

---

## How to Wire It Up in ADK

Compaction requires exporting an `App` object from your agent module, not just `root_agent`. ADK's runner looks for `app` first and uses its configuration.

```python
# backend/agents/eks_agent/__init__.py
from google.adk.apps.app import App, EventsCompactionConfig
from google.adk.apps.llm_event_summarizer import LlmEventSummarizer
from .agent import root_agent, llm_model

_EKS_SUMMARY_PROMPT = (
    'The following is a conversation between a user and an Amazon EKS expert'
    ' AI assistant. Summarize the conversation history below, preserving:\n'
    '- All EKS cluster names mentioned (exact AWS names)\n'
    '- Namespaces and their associated clusters\n'
    '- Any failing or problematic pods by name, namespace, and root cause\n'
    '- Key findings and conclusions reached\n'
    '- Any open/unresolved questions or follow-up tasks\n'
    '- Recommendations already given\n\n'
    'Be concise. Do NOT include raw JSON tool responses or log lines — only'
    ' the interpreted findings. Use bullet points.\n\n'
    'Conversation:\n{conversation_history}'
)

app = App(
    name='agents',  # must match ADK runner's inferred name (see below)
    root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,   # trigger after every 3 user turns
        overlap_size=1,          # retain 1 turn for continuity
        summarizer=LlmEventSummarizer(
            llm=llm_model,
            prompt_template=_EKS_SUMMARY_PROMPT,
        ),
    ),
)

__all__ = ['root_agent', 'app']
```

---

{% include donate.html %}
{% include advertisement.html %}

## The `name='agents'` Trap

This one took me a while to track down. ADK's runner infers the expected `App` name by inspecting the `LlmAgent` class's module path:

```python
# Inside ADK runners.py:
module = inspect.getmodule(agent.__class__)
# module.__name__ = "google.adk.agents"
# origin_dir.name = "agents"
```

If your `App(name=...)` doesn't match this inferred value, ADK emits a startup warning on every invocation:

```
App name mismatch detected. The runner is configured with app name "eks_agent",
but the root agent was loaded from ".../google/adk/agents", which implies app name "agents"
```

The fix: always use `App(name='agents')`. Not your agent's logical name, not your module folder name — the string `'agents'`. It's the last segment of ADK's own internal module path (`google.adk.agents`). This is confusing because it looks like a generic name, but it has to match exactly or you'll see this warning printed on every agent startup.

---

## Writing a Compaction Prompt That Actually Preserves What Matters

The default `LlmEventSummarizer` uses a generic prompt that produces summaries like:

> "The user asked about Kubernetes clusters and pods."

That's useless. You need a domain-specific prompt that explicitly tells the summarizer what information must be preserved verbatim. For my EKS agent, the non-negotiables were:

- **Exact cluster names** (AWS cluster names are not guessable — if the summary drops them, the agent has to re-discover them on the next query)
- **Specific pod names and their root causes** ("auth-service: CrashLoopBackOff due to OOMKilled, NPE in JVM heap allocation")
- **Open questions and unresolved follow-ups** (so the user doesn't get contradicted later)
- **Recommendations already given** (so the agent doesn't repeat them)

A generic summary loses all of this. A domain-specific prompt like the one above preserves what the user actually needs to continue the session.

---

## Tuning the Interval and Overlap

`compaction_interval=3` means: compact after turn 3, then again after turn 6, then turn 9, etc.

`overlap_size=1` means: keep the most recent turn's events verbatim in the history alongside the summary. Without this, the agent might get a summary and then immediately lack context for the user's *next* sentence.

**How to tune for your use case:**
- Large tool responses (like Kubernetes listings)? Use a smaller interval (2–3) to prevent history from bloating before the first compaction
- Mostly short chat exchanges? A larger interval (5–10) is fine and reduces the overhead of the compaction call itself
- `overlap_size=1` is the safe default for most conversational agents

---

## How to Confirm It's Working

Look for this line in your agent logs:

```
2026-04-13 22:59:01 | INFO | Running event compactor.
```

A behavioral signal: after compaction fires, the next query involving clusters discovered earlier should not trigger re-discovery tool calls — the agent knows the cluster names from the summary.

## What Compaction Doesn't Solve (Be Honest With Yourself About This)

**It doesn't fix large single-turn costs.** If one query generates 50,000 tokens of tool results in a single turn, compaction doesn't touch that until *after* the turn completes. The "collect-then-emit" pattern (never emit formatted output inside a loop) handles this — it's a prompt-level constraint that prevents individual turns from ballooning.

**Compaction itself costs tokens.** You're spending model calls to summarize. With a `compaction_interval=3` and typical sessions, the break-even is a few additional turns. It's worth it for long sessions and not worth it for sessions that almost never go past 3 turns.

**Summaries can drop facts.** LLMs occasionally omit details from long histories during summarization. This is why the compaction prompt is explicit about what to preserve, and `overlap_size=1` ensures the most recent full context is always present verbatim.

