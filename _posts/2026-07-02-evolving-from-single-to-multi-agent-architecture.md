---
layout: post
title: "When One Agent Isn't Enough: Splitting Into a Multi-Agent System"
date: 2026-07-02
excerpt: "When I started building agents, the instinct was to keep adding tools to one agent. The agent was good at Kubernetes queries — so I added network probe tools, then traffic diagnost"
tag:
- Multi-Agent Systems
- Agent Delegation
- AI Platform Evolution
comments: true
---
# When One Agent Isn't Enough: Splitting Into a Multi-Agent System

When I started building agents, the instinct was to keep adding tools to one agent. The agent was good at Kubernetes queries — so I added network probe tools, then traffic diagnostics, then CloudWatch, then cost analysis. One agent, everything inside it.

This works until it doesn't. Around 15–20 tools, the reasoning quality for tool selection starts degrading noticeably. Different tool categories need different security boundaries. One crashing workflow shouldn't kill the main agent's context. I split into a multi-agent system and here's exactly how it's structured and why.

---

## The Architecture: Sub-Agents via `transfer_to_agent`

The platform has three agents in the EKS agent module:

```
root_agent  (LlmAgent)
├── tools: [mcp_toolset]        ← read-only EKS MCP (15 tools)
└── sub_agents:
    ├── probe_agent             ← write-capable, Python FunctionTools
    └── traffic_agent           ← read-only, own MCP toolset instance
```

The root agent delegates to sub-agents via `transfer_to_agent(agent_name="eks_probe_agent")`. This is crucial — it's *not* `AgentTool` wrapping. The distinction matters enormously for streaming.

---

## `transfer_to_agent` vs `AgentTool`: Why It Matters for Streaming

**AgentTool** wraps a sub-agent as a callable tool. The root agent calls it like any other function. The sub-agent runs, finishes, and returns its output as a single `functionResponse` to the root agent.

```
root_agent calls agent_tool(...)
  → sub_agent runs (complete)
  → returns result as one functionResponse
  → root_agent sees the entire output at once
```

This is wrong for our probe agent, which emits live `PROBE_PROGRESS` updates to the UI. If `AgentTool` wraps it, those progress updates are invisible — they get collapsed into the final response.

**`transfer_to_agent`** delegates control entirely to the sub-agent. The sub-agent streams its events directly to the SSE connection. The UI receives real-time probe steps as they happen.

```
root_agent calls transfer_to_agent("eks_probe_agent")
  → probe_agent takes control
  → emits PROBE_PROGRESS events in real time → UI renders live progress
  → probe_agent completes
```

---

{% include donate.html %}
{% include advertisement.html %}

## The Probe Agent: Write-Capable, Isolated

The network probe agent is the only component with write access to Kubernetes. Its tool set is exhaustive and minimal:

```python
tools=[
    create_probe_pod,    # creates one busybox/curl pod
    get_probe_pod_status,  # polls pod phase
    get_probe_pod_logs,    # reads stdout/stderr
    delete_probe_pod,      # mandatory cleanup
]
```

Nothing else. Not label patching, not deployment scaling, not configmap updates. If I ever add probe variants, they go in `probe_agent.py` — the blast radius is contained.

Why Python `FunctionTools` rather than MCP tools for the probe agent? Because the EKS MCP server is hardcoded `allow_write=False`. Rather than adding a separate write-enabled MCP server with its own security surface, the probe agent directly uses the `kubernetes` Python client in-process. Write operations are scoped to a single RBAC service account with only `pods/create`, `pods/get`, `pods/delete` permissions.

---

## The Traffic Agent: Context Isolation

The traffic routing diagnostic agent uses the same read-only MCP server as the root agent but has its own toolset instance (its own MCP HTTP connection):

```python
traffic_mcp = McpToolset(
    connection_params=StreamableHTTPConnectionParams(
        url=MCP_SERVER_URL,
        terminate_on_close=False,
    ),
)
```

Why a separate toolset instance? Two reasons:

- **Context isolation**: the traffic agent runs a structured 10-step diagnostic protocol. It operates in its own ADK session — the root agent's history of cluster listings and pod inspections doesn't contaminate the traffic agent's flow.
- **Session independence**: if the root agent's MCP session has issues, the traffic agent's session is unaffected.

---

## The System Prompt Pattern for Delegation

Each sub-agent has a deeply specialized system prompt focused on one workflow. The traffic agent's prompt defines a strict step-by-step diagnostic protocol with explicit progress emission requirements at each step.

The root agent's prompt is broader but shallower. It knows *when* to escalate, not *how* to perform the escalation:

---

## When to Introduce a Sub-Agent

Here's a decision guide I use:

| Signal | Action |
|---|---|
| A workflow requires write operations | New sub-agent with minimal write tools |
| A workflow is a rigid multi-step pipeline | Consider sub-agent (or `SequentialAgent`) |
| Different security boundary needed | New sub-agent, separate RBAC |
| Real-time streaming events from sub-task | Sub-agent via `transfer_to_agent` |
| Tool count in root agent exceeds ~15 | Refactor subset into sub-agent |


Use `LlmAgent` for reasoning flexibility during development. Migrate to `SequentialAgent` when the workflow is proven and you want determinism guarantees.

## Gotchas

- `transfer_to_agent` name must match exactly: `agent_name="eks_probe_agent"` must match the `name=` field in the sub-agent's `LlmAgent()` constructor
- Sub-agents must be in `sub_agents=[]` on the root agent, not just imported
- Each sub-agent that uses MCP needs its own toolset instance — they can't share the same `McpToolset` object because connection state is per-instance
- Sub-agent context is separate: the sub-agent does not see the root agent's conversation history unless you explicitly pass context in the delegation message

