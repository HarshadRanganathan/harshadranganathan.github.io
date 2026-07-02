---
layout: post
title: "Writing System Prompts for Agents calling tools"
date: 2026-07-02
excerpt: "Prompt Engineering nuances for tool calling agents"
tag:
- Prompt Engineering
- Tool Calling Agents
- LLM Reliability
comments: true
---
# Writing System Prompts for Agents That Actually Use Tools Well

The first system prompt I wrote for my Kubernetes AI assistant was two sentences long:

> "You are an EKS expert. Help users manage their Kubernetes clusters."

It worked great in demos. It failed in production in about fifteen different ways — each one a lesson in how LLMs actually behave when you point them at real tools.

A system prompt for a tool-using agent is not a description of personality. It's closer to a specification for a coworker who is very smart, but has never worked in your domain before, never uses your tools the efficient way, and will always do the reasonable-looking thing rather than the correct thing unless you tell them otherwise.

Here's what I learned, failure mode by failure mode.

---

## Lesson 1: LLMs Do the Human Thing, Not the Efficient Thing

My tool `list_k8s_resources` accepts a `namespace=None` parameter that searches across all Kubernetes namespaces in one call. Without instruction, Gemini ignored it and called the tool once per namespace — ten times instead of once — because that's what a human engineer doing manual debugging would do. It's reasonable behavior. It's also expensive and slow.

The fix is being extremely explicit about what *not* to do:

```
CRITICAL: namespace=None means all namespaces. NEVER loop through individual 
namespaces manually. Always call list_k8s_resources(namespace=None) first.
```

The word "NEVER" in all-caps is load-bearing. I tested the same constraint written as "please avoid looping over namespaces" — it was ignored roughly half the time. Framing it as a hard constraint with CRITICAL/NEVER significantly reduced the failure rate. It's not a perfect solution, but it works better than polite suggestions.

This generalizes to any multi-step tool call pattern where a correct but inefficient path exists. The LLM will find that path unless you close it explicitly.

---

## Lesson 2: Your Agent Doesn't Know What Your Data Source Lies About

This was the most consequential failure mode I hit. Kubernetes has a concept called pod `phase`, which reports the lifecycle state of a container. `phase=Running` sounds like it means the pod is healthy and doing its job.

It doesn't. `phase=Running` only means the container process was started at least once. A container can be `phase=Running` while it's in `CrashLoopBackOff` — crashing and restarting every 30 seconds. It can be `phase=Running` while its readiness probe is failing and no traffic is being routed to it. Kubernetes just doesn't update the phase in those cases.

I found this out the hard way when the agent confidently reported "all pods are healthy" on a cluster where 12 pods were in `CrashLoopBackOff`. Every pod's phase was `Running`. The agent had no reason to look further.

The fix was to embed the domain knowledge directly in the system prompt:

```
CRITICAL: phase=Running does NOT mean healthy. Kubernetes reports phase=Running 
as long as the container process exists — even if it is crash-looping, OOM-killed, 
or silently failing.

Treat restartCount > 5 as a failure regardless of phase.
Treat containerStatuses[].ready == false as a failure.
```

Now the agent knows to run two passes: one for pods with non-Running phases, and a second pass inspecting container-level status for pods that are nominally `Running` but have high restart counts or failing readiness probes.

The general lesson: **your LLM doesn't know the semantics of your data source's lie.** Kubernetes lying about `Running` is a domain-specific trap. SQL `NULL` semantics are a different trap. HTTP 200 with error payloads is another. Whatever your data source does that's counterintuitive — document it in the prompt.

---

{% include donate.html %}
{% include advertisement.html %}

## Lesson 3: Show an Example, Don't Write a Rule

I spent a long time writing rules like "always normalize user input before searching" and "try multiple search strategies before giving up." The agent half-followed them, generalized them incorrectly, or applied them to situations where they didn't fit.

The thing that actually worked was a fully-spelled-out example of the reasoning chain I wanted:

```
WORKED EXAMPLE — commit this to memory:
User says: "give me the port of era full roster"
→ Step 1: normalize spaces to hyphens: "era full roster" → "era-full-roster"
→ Step 2: call list_k8s_resources(kind='Deployment', name_contains='era-fullroster')
   (try compound form first — remove middle word)
→ Step 3: tool returns matching resources
→ Step 4: use that result immediately. Do NOT ask the user anything.
```

The phrase "commit this to memory" isn't decorative — it tells the model this example is a reference pattern to retrieve when user input looks similar. Generic rules get abstracted away. Concrete examples stay retrievable.

For any multi-step workflow that needs to happen exactly right, write it out as a worked example, not as a list of rules.

---

## Lesson 4: Don't Let the Agent Emit Intermediate Results Mid-Loop

This one cost me real money before I caught it.

When asked "list namespaces across all clusters", the agent's natural instinct is:
1. Query cluster 1, get namespaces, immediately show the user a formatted table
2. Query cluster 2, show another table
3. ...repeat for 20 clusters

Here's the problem: every formatted table the agent emits becomes part of conversation history. Every subsequent model call re-reads that entire growing history. By cluster 14, the agent is processing ~80,000 input tokens on each tool call — because it's re-reading all previous tables on every step. This is what triggers `RESOURCE_EXHAUSTED` (HTTP 429) errors on multi-item queries, not running out of context window.

The fix is a simple pattern I now call **Collect-Then-Emit**, and it goes in the system prompt as an explicit rule:

```
WRONG — emit a table block after every cluster call
CORRECT:
1. Emit STEP: Querying all clusters — will emit one consolidated table at the end
2. For each cluster, emit only: STEP: ✓ cluster-name — N namespaces
3. After the final cluster, emit ONE consolidated table with a "Cluster" column
```

---

## Lesson 5: The Agent Can Drive UI State Through Text

One of the most useful patterns I found: the agent's text stream can carry structured markers that the UI intercepts and uses to update state — without the frontend needing any domain knowledge.

In my system, the agent emits lines like:

| Marker | Purpose |
|---|---|
| `STEP: <message>` | Live progress stepper in the UI |
| `CLUSTER: <cluster-name>` | Sidebar clickable cluster buttons |
| `table-data { ... }` | Rendered as a sortable table |
| `PROBE_PROGRESS: { ... }` | Real-time probe progress bar |
| `SUGGEST_ACTION: probe` | "Run network probe" button |

The frontend strips these lines from the visible response and uses them to update UI state — progress stepper, sidebar cluster buttons, rendered tables, live probe progress bars. The agent doesn't know about React components. The frontend doesn't know about Kubernetes. The text stream is the interface between them.

This pattern works for any agent with a rich UI. Define your markers in the system prompt. Parse them on the client. The LLM becomes the event source for your UI, not just a text generator.

---

## Lesson 6: Your Prompt is a Specification That Grows With the System

The EKS agent's system prompt went from 50 lines to 300+ lines over the course of building this. That's not bloat — every line traces back to a concrete production failure.

Here's the discipline I landed on: track which prompt change fixed which failure. Write it like a changelog — "added `exclude_labels` rule after agent caused 429 error on multi-cluster listing." Six months later you'll know why each section exists and whether it's still needed.

A system prompt is not documentation. It's a specification. Treat it with the same rigor you'd give to a contract.

---

## Quick Reference

- **NEVER / CRITICAL in ALL CAPS** is more reliable than "please don't" for hard constraints
- **Concrete worked examples** beat abstract rules for multi-step reasoning — the model can pattern-match to a specific example better than generalizing from a rule
- **Embed domain-specific gotchas** that the LLM can't know from training (e.g., Kubernetes phase semantics, SQL NULLs, API quirks in your domain)
- **Collect-then-emit** — for any loop that runs N times, never emit formatted output inside the loop. Collect everything, emit once. This is the single biggest lever for preventing rate-limit errors on bulk queries
- **Structured text markers** let the agent drive UI state changes without coupling the frontend to agent internals
- **If your tool returns `truncated=True`**, the agent must tell the user — silently omitting that caveat is worse than no answer

