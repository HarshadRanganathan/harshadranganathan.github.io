---
layout: post
title: "Where Does the Logic Actually Belong? Three Layers, Real Tradeoffs"
date: 2026-07-02
excerpt: "When you build an agent that talks to an API, you face a recurring architectural question: should this behavior live in the UI, in the agent prompt, or in the MCP server (your data"
tag:
- System Design
- AI Agent Boundaries
- Architecture Decisions
comments: true
---
# Where Does the Logic Actually Belong? Three Layers, Real Tradeoffs

When you build an agent that talks to an API, you face a recurring architectural question: should this behavior live in the UI, in the agent prompt, or in the MCP server (your data access layer)?

The wrong answer leads to fragile systems. Logic too far left (in the UI) creates a system that can't operate autonomously. Logic too far right (in the MCP server) bakes in assumptions that prevent the server from being useful beyond one specific query type. Logic in the agent that the API should handle is wasteful and slow.

Here are concrete examples from building a Kubernetes agent platform, and the reasoning behind where each piece of logic ended up.

---

## The Three-Layer Model

```
UI (React)  ←→  Agent (ADK / LLM)  ←→  MCP Server (FastMCP / Python)
   ↑                    ↑                         ↑
presentation         reasoning                data access
```

---

## Example 1: "Find Failing Pods" — why the MCP server should stay dumb

My first instinct was to add a `list_failing_pods` tool to the MCP server:

```python
@mcp.tool()
async def list_failing_pods(cluster_name: str) -> ...:
    # fetch all pods, filter for non-running, return list
```

Seems clean. But it bakes a specific definition of "failing" into the data layer — and that definition is wrong in an important way. Kubernetes `phase=Running` doesn't mean a pod is healthy (a pod can be `phase=Running` while crash-looping). Whether a pod is "failing" requires reasoning over multiple fields: phase, restart count, container readiness, waiting state. The definition is nuanced domain knowledge, not a simple data filter.

The right design: a general-purpose `list_k8s_resources` tool that returns raw pod status, and an agent whose system prompt knows what failure actually looks like. The MCP server stays generic and reusable. The reasoning lives with the LLM that's equipped to do it.

**General rule**: *if the determination requires judgment or domain knowledge, it belongs in the agent prompt. If it's a mechanical data query, it belongs in MCP.*

---

{% include donate.html %}
{% include advertisement.html %}

## Example 2: Live Progress Steps — why the agent should emit them, not the backend

I needed a live progress stepper in the UI — something that shows "Querying cluster..." while the agent is making tool calls. The naive approach is to intercept SSE events in the FastAPI backend and inject progress messages.

Don't do this. The backend doesn't know what the agent is doing. Any progress messages it injects will be generic ("Processing...") rather than meaningful ("Fetching pods with restart count >5 in kube-system").

The right approach: the agent emits `STEP:` markers in its own text stream, and the UI intercepts and renders them.

```
# Agent emits (in its text stream):
STEP: Querying available EKS clusters
STEP: Fetching pods in kube-system
STEP: Analyzing container restart counts
```

```typescript
// UI strips STEP: lines from rendered markdown, renders them as a progress stepper
const [steps, setSteps] = useState<string[]>([]);
const lines = chunk.split('\n');
for (const line of lines) {
  if (line.startsWith('STEP: ')) {
    setSteps(prev => [...prev, line.replace('STEP: ', '')]);
    // don't add to renderedText
  } else {
    setRenderedText(prev => prev + line + '\n');
  }
}
```

The agent knows its own plan in real time. The infrastructure doesn't. Let the agent be the source of progress events.

**General rule**: *the LLM is the event source for UI state. The UI catches and renders structured markers. The backend is just a pipe.*

---

## Example 3: Cluster Discovery — let the agent populate the sidebar

I needed the sidebar to show clickable buttons for every cluster the user's conversation touches. I could have built a REST endpoint that loads all cluster names at startup. Instead, the agent emits `CLUSTER: <name>` markers whenever it mentions a cluster, and the UI accumulates them:

```typescript
if (line.startsWith('CLUSTER: ')) {
  const clusterName = line.replace('CLUSTER: ', '').trim();
  addCluster(clusterName);  // Zustand store action
}
```

Why not pre-load cluster names at startup? Because the UI doesn't know which clusters matter for *this* conversation. The agent discovers them contextually — a "list all clusters" query finds 22, while a specific pod investigation might touch only one. Letting the agent drive sidebar state keeps the UI out of the domain logic business.

**General rule**: *contextual UI state belongs to the agent. Static UI state (layout, theme) belongs to the frontend.*

---

## Example 4: Name Normalization — the agent should handle fuzzy input

When a user types "era full roster", the agent needs to find the Kubernetes resource named `era-fullroster`. Spaces to hyphens, try compound form first, retry with variants. This is domain-specific reasoning.

I briefly considered pre-processing user input in the frontend before sending it to the agent:

```typescript
// Don't do this
const normalized = userInput.replace(/\s+/g, '-').toLowerCase();
sendMessage(normalized);
```

Don't do that. You lose the user's original phrasing, which the agent needs for context. Send the raw query, teach the agent the normalization strategy in its system prompt.

**General rule**: *fuzzy input handling is a reasoning task, not a preprocessing task.*

---

## Example 5: Write Operations — isolate them in a dedicated sub-agent

My EKS MCP server is read-only. The one write-capable feature I built — spinning up a temporary "probe pod" to test network connectivity — lives in a completely separate sub-agent with its own Python function tools and its own restricted Kubernetes RBAC service account.

**Why not add write tools to the MCP server?**

- The MCP server is shared across all agents. Adding write tools there exposes them to every agent.
- Write capability needs tighter control: the probe agent has exactly 4 tools — `create_probe_pod`, `get_probe_pod_status`, `get_probe_pod_logs`, `delete_probe_pod`. That's the complete blast radius.
- The parent agent escalates to the probe agent only when explicitly asked and confirms intentions first.

**General rule**: *write operations define your blast radius. Isolate them as tightly as possible. The tightest possible isolation is a purpose-built sub-agent with an explicit and minimal toolset.*

---

## Example 6: Semantic Color Mapping — pure presentation stays in the UI

The chart component maps status values like `"CrashLoopBackOff"` and `"Running"` to red and green. This is obviously presentation logic. It belongs in the frontend component.

```typescript
// frontend/src/components/chat/ModernChart.tsx
const SEMANTIC_BUCKETS = {
  danger: ['error', 'failed', 'crashloopbackoff', 'oomkilled'],
  warning: ['pending', 'unknown', 'warning'],
  success: ['running', 'active', 'succeeded'],
};
```

The agent returns raw status values. The frontend decides what color they should be. Neither one needs to know about the other's concerns.

---

## The Decision Framework

When a new feature comes up, I run through these questions:

| Question | UI | Agent | MCP |
|---|---|---|---|
| Needs multi-step reasoning? | ✗ | ✓ | ✗ |
| Has domain knowledge requirements? | ✗ | ✓ | ✗ |
| Is pure data access / transformation? | ✗ | ✗ | ✓ |
| Is pure presentation / layout? | ✓ | ✗ | ✗ |
| Needs to persist across turns? | ✗ | ✓ (session) | ✗ |
| Needs to be reusable across agents? | ✗ | ✗ | ✓ |
| Has a dangerous blast radius? | ✗ | ✓ (controlled) | minimize |

The key question for anything that's tempting to put in MCP: "does more than one agent need this?" If yes, MCP is the right place. If it's only needed by one agent with specific domain knowledge, it probably belongs in that agent's prompt or as a Python function tool, not in the shared MCP server.

