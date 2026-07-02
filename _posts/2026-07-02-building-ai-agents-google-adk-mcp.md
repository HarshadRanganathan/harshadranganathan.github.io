---
layout: post
title: "Building AI Agents with Google ADK and the Model Context Protocol — What I Wish I Knew First"
date: 2026-07-02
excerpt: "I built an ops AI assistant that lets engineers talk to their AWS Kubernetes clusters in plain English. Type 'why are my pods crashing in production?' and the agent queries the cluster"
tag:
- Google ADK
- Model Context Protocol
- AI Agent Architecture
comments: true
---

# Building AI Agents with Google ADK and the Model Context Protocol — What I Wish I Knew First

I built an ops AI assistant that lets engineers talk to their AWS Kubernetes clusters in plain English. Type "why are my pods crashing in production?" and the agent queries the cluster, reasons over the results, and explains the root cause — no `kubectl` required.

That sounds clean and simple. In practice, I spent weeks learning how the two underlying frameworks — **Google ADK** and the **Model Context Protocol (MCP)** — actually fit together, what their defaults do, and where they silently bite you.

If you're starting down this path, here's every important thing I wish someone had told me up front.

---

## The Three-Layer Stack

Before anything else, it helps to see the full flow:

```
React UI  →  FastAPI backend  →  ADK Agent  →  MCP Server  →  AWS EKS / Kubernetes
```

The **ADK Agent** is the brain — it receives a user message, decides which tools to call, calls them, reasons over the results, and generates a response. The **MCP Server** is a separate process that owns the actual access to data sources (in my case, Kubernetes clusters). The agent never talks to Kubernetes directly — it only knows about tools that the MCP server exposes.

Two frameworks do the heavy lifting: **Google ADK** and the **Model Context Protocol (MCP)**.

---

## What is Google ADK?

ADK (`google-adk`) is Google's opinionated Python framework for building LLM-powered agents. Think of it as the plumbing that sits between your LLM and your tools: it serializes the LLM's JSON tool call into a Python function call, runs the function, feeds the result back into the conversation, and manages the multi-turn loop so you don't have to.

The four concepts you'll use constantly:

| Concept | What it does |
|---|---|
| `LlmAgent` | An agent wired to an LLM with a system prompt and a set of tools |
| `McpToolset` | Connects to an MCP server and automatically exposes all its tools to the agent |
| `App` | A session container that wraps an agent — needed for advanced features like history compaction |
| `runner.run_async()` | Drives a single user invocation — feeds input, streams events back |

The minimal working setup looks like this:

```python
from google.adk.agents import LlmAgent
from google.adk.models.lite_llm import LiteLlm
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StreamableHTTPConnectionParams

llm_model = LiteLlm(model="vertex_ai/gemini-2.5-flash", vertex_location="global")

mcp_toolset = McpToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="http://localhost:8002/mcp",
        terminate_on_close=False,  # more on this default below
    ),
)

root_agent = LlmAgent(
    model=llm_model,
    name="eks_agent",
    instruction="You are an EKS expert...",
    tools=[mcp_toolset],
)
```

That's it. ADK reads the tool schemas from the MCP server, registers them with the LLM, and handles the call-and-response cycle automatically.

---

{% include donate.html %}
{% include advertisement.html %}

## What is MCP?

The **Model Context Protocol** is a vendor-neutral standard for exposing tools to LLMs over HTTP. An MCP server is just a Python (or other language) process that publishes a list of callable functions — along with their JSON schemas — over an HTTP endpoint. Any MCP-compatible client, including ADK, can connect and use those tools.

You might wonder: why bother with a separate server? Why not just write Python functions and pass them directly to the agent?

For a toy project, direct functions are fine. For a production system, the MCP boundary buys you three things:

1. **Isolation** — The MCP server has its own process, its own credentials, and its own dependencies. If it crashes, the agent process doesn't crash with it. You can restart the MCP server without restarting agents.

2. **Reusability** — Multiple agents can connect to the same MCP server. In my setup, the main EKS agent, a network probe sub-agent, and a traffic diagnostic sub-agent all share one MCP server. Same tools, three different agents, one codebase to maintain.

3. **Security boundary** — AWS credentials, Kubernetes kubeconfig, and IAM role arns live in the MCP server environment. The agent only knows tool schemas. If the agent's LLM is ever fooled into doing something unexpected, it still can't access credentials directly.

Building an MCP server with `fastmcp` is straightforward:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("eks-server", stateless_http=True)

@mcp.tool()
async def list_k8s_resources(ctx: Context, cluster_name: str, kind: str, ...) -> KubernetesResourceListResponse:
    # actual Kubernetes API call here
    ...
```

The `@mcp.tool()` decorator automatically generates the JSON schema from the type annotations and docstring. The LLM sees that schema and knows how to call this function.

---

## The Things That Will Trip You Up

### 1. `terminate_on_close=False` — set it or else

By default, `McpToolset` closes and tears down its HTTP session to the MCP server when an agent invocation finishes. Sounds reasonable — until you have more than one agent connecting to the same server.

In my setup, the root agent, the probe sub-agent, and the traffic sub-agent all use `McpToolset`. If the root agent invocation ends, the default behavior closes the session — mid-conversation, while sub-agents might still need it.

The fix is one line:

```python
McpToolset(
    connection_params=StreamableHTTPConnectionParams(
        url=MCP_SERVER_URL,
        terminate_on_close=False,  # keep connection alive across invocations
    )
)
```

Set this on every `McpToolset` in your system. It's a silent failure — the tool call just dies with a connection error, and the agent either confabulates or reports an unhelpful error.

### 2. `App` vs plain `root_agent` — know when you need it

If you export just `root_agent` from your agent module, ADK runs it in a basic session with no extras. That's fine to start. But once you want session-level features — like automatic history compaction (more on that in another post) — you need to export an `App` instance instead:

```python
from google.adk.apps.app import App, EventsCompactionConfig

app = App(
    name='agents',
    root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(...),
)
```

ADK's runner checks for an `app` variable first. If it doesn't find one, it falls back to `root_agent`. The catch is that **the `App.name` must be `'agents'`**, not your agent's logical name. That's because ADK infers the expected name from the `LlmAgent` class's module path (`google.adk.agents`), and the last segment is always `'agents'`. If your name doesn't match, ADK emits a warning on every startup — not a crash, just noise you'll chase for a while.

### 3. ADK version instability — pin it

ADK is a young framework and it moves fast. Within a few weeks I went through versions 1.18, 1.19, and 1.20. Each upgrade changed the streaming event format — the shape of JSON that comes out of the SSE endpoint. Version 1.20 introduced an envelope format (`{ content: { parts: [...] } }`) that completely broke the frontend's message parser. The UI went blank.

I ended up pinning to `google-adk==1.18.0` while I fixed the parser to handle both formats, then re-upgraded. The lesson here: **treat ADK version upgrades like database migrations** — test your client-side parsing against the new format in a staging environment before shipping.

### 4. `vertex_location="global"` for higher quota

When using Gemini via Vertex AI, the model endpoint matters for rate limits. The regional endpoints (`us-central1`, `europe-west1`, etc.) have lower tokens-per-minute quotas than the global endpoint. Using `vertex_location="global"` routes to Google's global serving infrastructure, which has access to higher aggregate TPM.

```python
llm_model = LiteLlm(model="vertex_ai/gemini-2.5-flash", vertex_location="global")
```

For a lightly used personal project this doesn't matter. For anything with multiple concurrent users it absolutely does.

---

## Architecture Principles That Held Up

After running this in production, a few design principles consistently proved their worth:

**Keep the MCP server read-heavy.** Write operations dramatically increase what can go wrong when the LLM misbehaves. My EKS MCP server is hardcoded `allow_write=False`. The one write-capable agent (a Kubernetes probe that creates/deletes temporary pods) is a completely separate sub-agent with exactly 4 tools and its own RBAC service account. If you need writes, isolate them.

**Credentials live in the MCP server, not the agent.** The agent only knows about tool schemas. AWS credentials, kubeconfig, IAM role ARNs — they never leave the MCP server's environment. This is a meaningful security boundary.

**Run MCP servers as sidecars or separate deployments.** The agent needs stable HTTP access to the MCP endpoint. A sidecar container in the same pod is the simplest setup; a separate Kubernetes deployment is better if you need independent scaling or rollout control.

