---
layout: post
title: "The Unexpected Complexity of Streaming Agent Responses to a Browser"
date: 2026-07-02
excerpt: "Server-Sent Events looked like the simple part of this project. It's a well-supported browser API, it's built into most HTTP frameworks, and the mental model is simple: server push"
tag:
- Server-Sent Events
- Streaming UX
- Realtime AI Interfaces
- Chat Bots
comments: true
---

# The Unexpected Complexity of Streaming Agent Responses to a Browser

Server-Sent Events looked like the simple part of this project. It's a well-supported browser API, it's built into most HTTP frameworks, and the mental model is simple: server pushes text, client displays it.

Then I added tool calls, an nginx proxy, a framework that changed its event format between minor versions, and a Gemini API that sends rate limit errors in-band over the event stream rather than as HTTP status codes. Here's what I found.

---

## Why SSE for Agent Streaming?

SSE is the right transport for agent responses because agent output is one-directional during a turn (server → client), incremental (tokens arrive as they're generated), and potentially long-lived (a multi-cluster scan can run 30–60 seconds).

WebSockets add bidirectionality you don't need. HTTP request streaming works but has buffering behavior that varies by proxy. SSE is the simplest fit: native browser support, automatic reconnect on network drops, and straightforward server implementation in any async framework.

ADK's runner exposes an SSE endpoint at `/run_sse`. The frontend holds an `EventSource` connection, receives events, and renders them incrementally.

---

## The Streaming Envelope Problem

This caused the most debugging time of any feature in the entire project.

**ADK 1.18** emitted agent events in a simple format:

```json
{"mime_type": "text/plain", "data": "The EKS clusters are: ...", "turn_complete": false}
```

**ADK 1.19/1.20** changed to a streaming envelope format:

```json
{
  "content": {
    "parts": [
      {"text": "The EKS clusters are: ..."}
    ]
  },
  "partial": true
}
```

The legacy client code completely broke — `message.data` was undefined, `message.mime_type` was undefined. The UI showed nothing.

I downgraded to ADK 1.18 to buy time, then fixed the parser to handle both formats:

```typescript
const raw = JSON.parse(event.data);

// ADK 1.19+ envelope: { content: { parts: [...] }, partial?: boolean }
if (raw?.content && Array.isArray(raw.content.parts)) {
  const parts = raw.content.parts;
  const texts: string[] = [];
  const seen = new Set<string>();
  
  for (const p of parts) {
    // Skip function call / tool response parts
    if (p.functionCall || p.functionResponse) continue;
    
    let txt = p.text;
    if (typeof txt === 'string' && txt.length > 0) {
      if (!seen.has(txt)) {  // deduplicate
        texts.push(txt);
        seen.add(txt);
      }
    }
  }
  
  onMessage({
    mime_type: 'text/plain',
    data: texts.join(''),
    turn_complete: raw.partial === false || raw.partial === undefined,
  });
  return;
}

// ADK 1.18 format: { mime_type, data, turn_complete }
if (raw?.data && raw?.mime_type) {
  onMessage(raw as Message);
  return;
}
```

Then ADK 1.20 reverted some of these changes, and I stayed on 1.18 as the stable baseline. The lesson: **pin your ADK version and test the client-side parser against every version you upgrade to**.

---

{% include donate.html %}
{% include advertisement.html %}

## The Deduplication Problem

ADK sometimes emits the same text part twice in rapid succession. This happened with thought/reasoning events — the model's internal chain-of-thought appears as a text part, then the same text appears again as the "published" output.

Without deduplication:

```
STEP: Querying available EKS clusters
STEP: Querying available EKS clusters      ← duplicate
Fetching pods...
Fetching pods...                           ← duplicate
```

The `seen` set in the parser above handles within-event deduplication. For cross-event deduplication (the same text arriving in consecutive SSE events), track the last emitted chunk:

```typescript
// In the store
let lastChunk = '';
// In the SSE handler
if (message.data !== lastChunk) {
  lastChunk = message.data;
  appendToMessage(message.data);
}
```

---

## Tool Calls in the Stream

When the agent calls a tool, ADK emits function call events into the SSE stream alongside text events. For a Kubernetes query, the stream looks like:

```
event: message  {"content": {"parts": [{"text": "STEP: Querying pods..."}]}}
event: message  {"content": {"parts": [{"functionCall": {"name": "list_k8s_resources", "args": {...}}}]}}
  ... (MCP server processes the call) ...
event: message  {"content": {"parts": [{"functionResponse": {"name": "list_k8s_resources", "response": {...}}}]}}
event: message  {"content": {"parts": [{"text": "Found 3 failing pods..."}]}}
```

The function call and response events must be filtered out on the frontend — they're infrastructure events not intended for direct rendering. Our parser skips them:

```typescript
if (p.functionCall || p.functionResponse) continue;
```

But here's the important part: during a tool call, the SSE stream **pauses**. The MCP server is processing the request and the user sees nothing for 2–5 seconds. Without explicit progress signals, this looks like a hang.

This is exactly why the system prompt requires the agent to emit a `STEP:` line *before* each tool call, so the user sees "Querying EKS clusters..." while the tool call is in flight.

---

## The `turn_complete` Signal

`turn_complete: true` tells the client that the agent has finished generating its response for this turn. The UI uses this to:

1. Stop displaying the typing indicator
2. Enable the message input
3. Finalize the message rendering (convert partial Markdown to full render)

In the ADK 1.18 format, `turn_complete` is explicit. In the 1.19+ envelope format, it's inferred from `partial: false`. The parser normalizes this:

```typescript
turn_complete: raw.partial === false || raw.partial === undefined
```

If `turn_complete` never fires (due to a network error or agent crash), the UI locks up. I added a timeout: if no event arrives within 30 seconds, assume the connection dropped and show "Connection lost — retry?" UI.

---

## The nginx Proxy Challenge

In production, the React frontend is served by nginx, which proxies API calls to the FastAPI backend. Standard nginx buffering breaks SSE:

```nginx
# Without this, nginx buffers SSE events until the connection closes
# User sees nothing until the entire response is complete
location /api/ {
    proxy_pass http://backend:8000/;
    proxy_buffering off;           # ← critical for SSE
    proxy_cache off;               # ← prevents caching of event stream
    proxy_set_header Connection ''; 
    proxy_http_version 1.1;
    chunked_transfer_encoding on;
}
```

`proxy_buffering off` is non-negotiable for SSE. Without it, nginx buffers the entire response before forwarding, effectively turning streaming into batch delivery. You won't catch this in local development where nginx isn't in the path.

---

## The 429 and Error Events in the Stream

Rate limit errors from Gemini arrive as SSE error events, not HTTP 4xx responses. The SSE connection status is always 200 — errors come in-band as event payloads. Your error detection must parse the event data:

```typescript
if (event?.error) {
  const isRateLimit = isRateLimitError(event.error);
  if (isRateLimit) {
    // trigger auto-retry UI with backoff timer
  } else {
    // surface as generic error
  }
}
```

Detecting Gemini's various rate-limit error strings requires a multi-pattern check because different clients serialize it differently:
- Vertex AI SDK: `RESOURCE_EXHAUSTED`
- HTTP layer: `429`
- LiteLLM wrapping: `rate limit exceeded`

---

## What to Watch For

- **Pin ADK version** — the streaming event format changed between minor versions; test the parser on every upgrade
- **Parse both old and new envelope formats** — the fallback is your regression safety net
- **Deduplicate text parts** — ADK sometimes emits the same part twice
- **Filter `functionCall` / `functionResponse` parts** — they're infrastructure, not content
- **`proxy_buffering off` in nginx** — essential and easy to forget; hard to catch in local dev
- **Errors arrive in-band** — the SSE connection is always 200; parse event data for rate limit and API error signals
- **`turn_complete` timeout** — if it never fires, the UI must not hang forever
- **`STEP:` lines before tool calls** — prevent the "silent hang" during MCP round trips

SSE itself is simple. An agent that calls external APIs, streams reasoning steps, handles rate limits, and runs through an nginx proxy in production is not. Each of these issues is invisible in a demo but certain to appear under real usage.

