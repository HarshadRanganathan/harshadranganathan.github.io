---
layout: post
title: "Designing Token-Efficient Tool Responses for LLM Agents: A Field Guide"
date: 2026-07-02
excerpt: "Every MCP tool response becomes conversation history and gets re-billed on every future model call. A field guide to filtering, serializing, and rate-limiting tool responses."
tag:
- Token Optimization
- MCP Optimization
- TPM Optimization
- Smart Filtering
- Pydantic v2
comments: true
---
# Designing Token-Efficient Tool Responses for LLM Agents: A Field Guide

Here's something that isn't obvious when you start building agents: every byte your MCP tool returns doesn't just answer the current question. It gets serialized into conversation history and re-read, in full, on every subsequent model call for the rest of the session.

A tool response with 5,000 tokens of JSON doesn't cost you one model call. It costs you `N × 5,000` tokens over a conversation with N turns. A Kubernetes pod listing that came back from turn 1 is still being billed again at turn 8, even if the user has moved on to a completely different question.

I used to think of token counts as a billing concern — something to optimize after everything else works. Then I started hitting 429 rate-limit errors, watching queries fail midway through because the model was processing 200,000 tokens per call, and realized token count is an operational correctness concern, not just a cost line item.

This post consolidates everything I learned building a Kubernetes operations agent: where tool-response tokens actually go, the filtering stack that gets rid of the waste, the Pydantic v2 serialization trap that quietly undoes your fixes, and why the same waste shows up as rate-limit failures, not just a bigger invoice.

---

## The Core Mental Model: TPM vs Context Window

Two different limits get conflated constantly, and the error message doesn't help:

| Limit | What It Is | Gemini 2.5 Flash Value |
|---|---|---|
| Context window | Max tokens in a single request | 1,000,000 tokens |
| TPM (tokens per minute) | Tokens processed per minute across all requests | Varies by tier |

A multi-cluster scan across 22 clusters rarely hits the context ceiling — a single call is nowhere near 1M tokens. It hits the **TPM ceiling**, because the agent issues many large model calls in rapid succession, and each one re-reads all the accumulated history from the calls before it.

`RESOURCE_EXHAUSTED` / HTTP 429 looks like a context-window error. It usually isn't. It's a rate problem, not a size-of-one-request problem — and the fix is different depending on which one you actually have.

---

## Root Cause: The Accumulating History Problem

Each agent invocation flows like this:

```
Turn 1: [user message] → model call (cost: N tokens)
Turn 2: [user + assistant turn 1 + tool results] → model call (cost: N + history₁ tokens)
Turn 3: [user + assistant + tool results × 2] → model call (cost: N + history₁ + history₂ tokens)
```

If each tool result is ~3,000 tokens and the agent is working through 22 clusters one call at a time:

- Call 14: system prompt + 13 previous tool results + current ≈ 50,000 tokens
- Call 20: ≈ 80,000 tokens
- Call 22: ≈ 90,000 tokens

That's for one user's query. Multiply by concurrent users and you understand why a perfectly reasonable agent plan starts failing around cluster 14 with no code bug anywhere.

Two independent levers reduce this: **shrink what each tool response contains**, and **stop the agent from re-emitting large content inside loops**. The rest of this post works through both.

---

## A Practical Taxonomy of Where Tokens Go

Here's a practical inventory of where tokens go in an agent that talks to a real API — Kubernetes, in this case, but the categories generalize to any backend with optional fields, verbose objects, and metadata nobody asked for.

### 1. Null Fields

Pydantic serializes `Optional[str] = None` as `"field": null` in JSON. Before fixing this, every resource in a listing emitted 5–6 null keys for fields that simply weren't applicable:

```json
{"name": "my-namespace", "namespace": null, "creation_timestamp": null,
 "labels": null, "annotations": null, "status": null}
```

versus the fixed form:

```json
{"name": "my-namespace", "status": {"phase": "Active"}}
```

For a 50-namespace listing across 22 clusters, that's `50 × 22 × ~6 ≈ 6,600` tokens of nothing per query. The correct fix (and a trap that will fool your unit tests) gets its own section below.

### 2. The Annotation Nobody Asked For

Kubernetes resources carry `kubectl.kubernetes.io/last-applied-configuration` — the **complete resource spec as JSON**, used by `kubectl apply` to diff changes. For a typical Deployment that's 500–800 tokens duplicating information already present in the resource's other fields. An LLM will never use it.

**Don't strip all annotations.** My first instinct was to remove everything under `metadata.annotations`. That broke real diagnostic scenarios — `helm.sh/chart`, `deployment.kubernetes.io/revision`, and custom `team`/`owner` annotations regularly *are* the answer to the user's question. The right policy is a blocklist of known-noise keys, not a full strip:

```python
_BLOCKED_ANNOTATION_KEYS = frozenset({
    'kubectl.kubernetes.io/last-applied-configuration',
    'checksum/config',
    'checksum/secret',
})

def _filter_annotations(self, annotations: Dict[str, str]) -> Optional[Dict[str, str]]:
    """Strip large/useless annotation keys; keep everything else."""
    if not annotations:
        return None
    filtered = {
        k: v for k, v in annotations.items()
        if k not in self._BLOCKED_ANNOTATION_KEYS
        and not k.startswith('checksum/')
    }
    return filtered or None
```

### 3. Labels When You're Not Going to Filter By Them

Labels are how you derive a `label_selector` for follow-up queries ("find all pods owned by this deployment"). That's a legitimate reason to include them. But if the user asked "how many namespaces are there?", labels are pure waste — a typical label set is ~40 tokens, and across a listing of 100 pods that's 4,000 tokens of nothing.

```python
exclude_labels: bool = Field(
    False,
    description="""When True, omit metadata.labels from every item.
    Use when you will NOT use labels in a subsequent label_selector filter.
    Always True for Namespace resources.""",
)

# Always strip labels for Namespace resources — never needed for the answer
_exclude_labels = exclude_labels or kind.lower() == 'namespace'
```

At 50 namespaces × 22 clusters, `exclude_labels` alone saves `50 × 22 × 40 = 44,000` tokens on a single namespace-listing query.

{% include donate.html %}
{% include advertisement.html %}

### 4. Verbose Status for Healthy Resources

Kubernetes status blobs aren't uniform, and the naive approach — return the full blob for every resource kind — is both wasteful and noisy. A healthy pod's full status includes 4 conditions and a container-status object, roughly 200 tokens that collectively say "this pod is fine." A Node's full status can be 50+ fields. A Namespace's status is one field.

Extract kind-specific shapes instead of passing the raw blob through:

```python
if kind.lower() == 'pod':
    status_summary = {
        k: v for k, v in {
            'phase': phase,
            'message': status.get('message'),
            'reason': status.get('reason'),
        }.items() if v is not None
    }
    if problem_containers:
        status_summary['containers'] = problem_containers

elif kind.lower() in ['deployment', 'replicaset', 'statefulset', 'daemonset']:
    status_summary = {
        k: v for k, v in {
            'replicas': status.get('replicas'),
            'readyReplicas': status.get('readyReplicas'),
            'availableReplicas': status.get('availableReplicas'),
            'conditions': status.get('conditions') or None,
        }.items() if v is not None
    }
```

A healthy pod now returns `{"phase": "Running"}`. A crashing pod still gets the full diagnostic picture — container statuses only appear when `restartCount > 0` or `ready == false`, i.e. when there's actually something to investigate. A cluster with 500 healthy pods shouldn't generate 100,000 tokens of "ready: true" entries.

### 5. Fields That Are Never Useful for a Given Resource Kind

Not every field matters for every kind. `creationTimestamp` on a Namespace is never the answer to any real user question — strip it server-side unconditionally rather than waiting for a `null`-elimination pass to catch it:

```python
if kind.lower() == 'namespace':
    creation_timestamp = None  # stripped server-side — never relevant
```

Small savings per resource, but it's part of the discipline: question every field for every resource type instead of assuming "more data can't hurt."

### 6. Formatted Output Inside Loops

This is the most expensive pattern, and the hardest to see before it causes problems. When an agent loops over N items and emits a formatted response block after *each* one, those blocks become assistant messages in conversation history — and every subsequent model call in the loop re-reads all of them. The overhead grows roughly quadratically.

For 22 clusters: 22 table blocks × ~600 tokens each = 13,200 tokens in history, each block re-read an average of 11 more times ≈ **145,000 extra tokens** across a single query.

The fix is a prompt-level constraint, not a code change: never emit formatted output inside a loop. Collect all results, emit once at the end. Short `STEP: <description>` progress lines (~10 tokens each) are fine inside the loop — they don't get re-parsed as data, and the frontend strips them from the rendered output.

```
Old approach (emit per cluster): 22 × 600 = 13,200 tokens, re-read ~11 more times
                                 ≈ 145,000 extra tokens
Collect-then-emit:               22 × 10 (a short STEP: line) = 220 tokens total
```

### 7. Accumulated Conversation History

Everything above shrinks individual tool responses. This category is different — it compounds all the others, because every token that makes it into history gets re-billed on every subsequent model call:

```
Turn 5 input = system_prompt + history_turns_1-4 + turn_5_user_message
             ≈ 5,000 + (4 × 10,000) + 50
             = ~45,000 tokens
```

**Context compaction** — replacing accumulated history with a compact LLM-generated summary after a set number of turns — is the systemic fix for this category. Everything else in this post is mitigation until compaction is in place; see the companion post on `EventsCompactionConfig` and `LlmEventSummarizer` for the implementation.

---

## They Stack

None of these optimizations are independent — they compound. Here's the same 22-cluster namespace-listing query, before and after applying all of them:

| Source of waste | Before | After |
|---|---|---|
| Null fields | ~6,600 tokens | 0 |
| `last-applied-configuration` | ~8,800 tokens | 0 |
| Label inflation | ~44,000 tokens | 0 (`exclude_labels=True`) |
| Intermediate table emits | ~145,000 extra tokens re-read | ~2,400 |
| Healthy pod verbosity | ~12,000 tokens | ~800 |
| **Total** | **~216,000 tokens+** | **~3,200 tokens** |

A 65× reduction for this query pattern, with no change in answer quality.

---

## The Filtering Stack: Order of Operations

When multiple filters apply to a single tool call, order matters for efficiency. Each layer should reduce what the next layer has to process:

1. **Server-side field selector** — most efficient, nothing unsuitable ever leaves the API. `field_selector='status.phase!=Running'` returns only the 20 problem pods out of 2,000, without transferring the other 1,980 over the network at all. Gotcha: `field_selector` only supports exact-match on a small set of fields — no partial strings, no range comparisons.
2. **`name_contains` for keyword search** — Kubernetes doesn't support partial name matching natively. Implement it server-side in the MCP process, after the API response, before the data reaches the agent: `name_contains.lower() in item.metadata.name.lower()`. This is how users actually describe resources — "the era fullroster thing," "anything with 'cache' in the name" — not by exact name.
3. **Kind-specific status extraction** — reduce field count per resource (section 4 above).
4. **Annotation blocklist** — remove known noise keys, keep everything else (section 2 above).
5. **Conditional label stripping** — remove labels if this query type won't need them (section 3 above).
6. **Null field elimination** — strip empty fields at serialization time (next section).
7. **Agent reasoning** — apply domain judgment to data that's already been filtered.

If you skip step 2 and let 500 resources through on name alone, step 7 — the LLM, the most expensive layer in the stack — ends up doing filter work that should have happened before the data ever reached it.

### Truncation Must Be Visible, Not Silent

The `limit` parameter prevents catastrophic over-fetching, but truncation has to be communicated to the agent, or it will silently present a partial view as a complete answer — which is worse than no answer at all:

```python
class KubernetesResourceListResponse(CallToolResult):
    total: Optional[int] = None   # total count from the Kubernetes API
    returned: int                  # how many are in this response
    truncated: bool = False        # True if returned < total
    items: List[ResourceSummary]
```

```
When a list response has truncated=True, always tell the user explicitly:
"Results are limited to the first N of M total — use a namespace filter or
name_contains to narrow the search."
```

---

## The Pydantic v2 Null-Stripping Trap

This deserves its own section because it's the bug most likely to fool you: your fix looks correct, your unit test passes, and production keeps emitting nulls anyway.

### The Bug That Hides in Plain Sight

```python
summary = ResourceSummary(name="my-namespace", status={"phase": "Active"})
print(summary.model_dump(exclude_none=True))
# {'name': 'my-namespace', 'status': {'phase': 'Active'}}  ✓ passes
```

You deploy. You inspect the actual SSE stream from the MCP server and see the nulls are still there:

```json
{"name": "my-namespace", "namespace": null, "creation_timestamp": null,
 "labels": null, "annotations": null, "status": {"phase": "Active"}}
```

### The Root Cause: Two Serialization Paths

Pydantic v2 has two serialization paths. **Path A** — top-level `model_dump()` / `model_dump_json()` — respects `exclude_none=True` and any `model_dump()` override you've written. **Path B** — nested serialization, which is what actually happens in production — goes through Pydantic's C-extension serializer (`pydantic_core`), which serializes child models directly and **bypasses `.model_dump()` overrides on the child class entirely**:

```python
class KubernetesResourceListResponse(CallToolResult):
    items: List[ResourceSummary] = Field(...)  # nested!

response.model_dump()
# → serializes ResourceSummary items via the internal C-extension path
# → your ResourceSummary.model_dump() override is never called
```

The wrong fix — the one that appears to work because your test calls `model_dump()` directly on the child — looks like this:

```python
class ResourceSummary(BaseModel):
    def model_dump(self, **kwargs) -> dict:
        data = super().model_dump(**kwargs)
        return {k: v for k, v in data.items() if v is not None}
```

It silently does nothing in production, where the parent model is what actually gets serialized.

### The Correct Fix: `@model_serializer(mode='wrap')`

This registers in Pydantic's core schema at class-definition time and fires during *any* serialization path, including nested parent serialization:

```python
from pydantic import BaseModel, Field, model_serializer
from typing import Any, Callable, Dict, Optional

class ResourceSummary(BaseModel):
    name: str = Field(...)
    namespace: Optional[str] = Field(None)
    creation_timestamp: Optional[str] = Field(None)
    labels: Optional[Dict[str, str]] = Field(None)
    annotations: Optional[Dict[str, str]] = Field(None)
    status: Optional[Dict[str, Any]] = Field(None)

    @model_serializer(mode='wrap')
    def _exclude_none_fields(
        self,
        handler: Callable[['ResourceSummary'], Dict[str, Any]]
    ) -> Dict[str, Any]:
        """Exclude None fields during serialization — fires in ALL serialization paths."""
        return {k: v for k, v in handler(self).items() if v is not None}
```

`handler(self)` runs the default serializer, producing the normal dict including `None` values — you filter and return.

| Mode | Behavior |
|---|---|
| `mode='plain'` | Replaces the serializer entirely — you produce the whole output dict yourself |
| `mode='wrap'` | Wraps the default serializer — `handler(self)` gives you the default output, you post-process it |

`mode='wrap'` is the right choice here: you want the default serialization for everything, plus a null-strip pass afterward. `mode='plain'` would force you to hand-roll every field.

**Verify against the real serialized output, not your unit tests.** Unit tests that call `model_dump()` directly exercise the top-level path and will pass even with the broken child-class override. Check the actual SSE stream after deploying, against real API responses:

```bash
# Correct:
{"name":"argocd","status":{"phase":"Active"}}
# Still broken:
{"name":"argocd","namespace":null,"creation_timestamp":null,"labels":null,...}
```

### One More Layer: Dict Construction

`@model_serializer` strips nulls at *output* time, but nulls can sneak back in from the dict you pass to the constructor for a `Dict[str, Any]` field — Pydantic's field validation doesn't recursively clean values inside an `Any`-typed dict:

```python
# BEFORE — nulls constructed explicitly
status_summary = {
    'phase': phase,
    'message': status.get('message'),  # could be None
    'reason': status.get('reason'),    # could be None
}

# AFTER — filter at construction time too
status_summary = {
    k: v for k, v in {
        'phase': phase,
        'message': status.get('message'),
        'reason': status.get('reason'),
    }.items() if v is not None
}
```

Both layers need filtering: the serializer for the model's own optional fields, and the dict-construction site for anything typed as `Dict[str, Any]`.

### Why This Matters Beyond Token Count

Null fields aren't just token waste — they're **semantic noise**. When the model reads `{"name": "argocd", "namespace": null, "labels": null, "annotations": null}`, it has to reason "these are null, meaning not applicable" before it can get to the signal. Across hundreds of resources in a multi-cluster scan, that's the model's attention being repeatedly pulled toward noise instead of the answer. Clean JSON — `{"name": "argocd", "status": {"phase": "Active"}}` — gives it exactly what it needs and nothing else.

---

## Fixing TPM at the Source

Shrinking tool responses reduces how much enters history in the first place, but three more levers matter specifically for TPM (rate-limit) failures:

**Batch large sets instead of one call per item.** Even with collect-then-emit, 22 individual tool calls in one context eventually grows large. Add an explicit batching rule to the system prompt:

```
If there are 10 or more clusters, process in batches of 5:
- Batch 1 (clusters 1-5): collect all → emit one table-data block for the batch
- Batch 2 (clusters 6-10): collect all → emit one table-data block
```

Each batch's output is ~1,500 tokens, so history grows in bounded increments instead of unbounded accumulation.

**Use the global Vertex AI endpoint.** Regional Gemini endpoints have lower TPM quotas than the global endpoint:

```python
llm_model = LiteLlm(model="vertex_ai/gemini-2.5-flash", vertex_location="global")
```

This routes to Vertex AI's global serving infrastructure, which has access to higher aggregate quota.

**Retry with calibrated exponential backoff on the client, for the 429s that still slip through:**

```typescript
// Attempt 0 → wait 60s → retry
// Attempt 1 → wait 120s → retry
// Attempt 2 → wait 240s → retry
// Attempt 3 → give up, show manual retry button
const RATE_LIMIT_DELAYS = [60, 120, 240]; // seconds, matched to Gemini's quota refresh window

function isRateLimitError(raw: unknown): boolean {
  const text = String(raw ?? '').toLowerCase();
  return (
    text.includes('resource_exhausted') ||
    text.includes('ratelimit') ||
    text.includes('rate limit') ||
    text.includes('429')
  );
}
```

The detector needs multiple signal strings because different layers in the stack serialize the same underlying error differently by the time it reaches the frontend.

---

## Two More Server-Side Habits Worth Adopting

**Add explicit timeouts.** A Kubernetes API call that hangs for 60 seconds doesn't just annoy the user — it leaves the agent in an ambiguous state and often surfaces as an unhelpful HTTP-layer timeout. Set timeouts explicitly at the client level:

```python
configuration.retries = 0
client.rest.RESTClientObject.pool_manager = urllib3.PoolManager(
    timeout=urllib3.Timeout(connect=5.0, read=30.0)
)
```

A `read=30.0` timeout means a clean, fast error instead of an indefinite spinner.

**Keep MCP connections alive between invocations.** By default, `McpToolset` closes its HTTP session to the MCP server when an agent invocation completes — fine for a single agent running once, wasteful for a platform where multiple agents share one MCP server across many tool calls. Set `terminate_on_close=False` on every `McpToolset`, and make sure the MCP server runs with `stateless_http=True` so each tool call remains independently safe to handle over a reused connection.

---

## How to Approach This Systematically

Token optimization is iterative, not a one-time audit. Here's the loop:

1. Run the query and log input token counts from the model provider's response metadata.
2. If the count is higher than expected, identify the biggest contributor using the taxonomy above as a checklist.
3. Fix the biggest source, measure again.
4. The second-biggest source only becomes worth fixing once the first is gone — don't try to fix everything in one pass.

The instinct to over-return data comes from traditional API design, where "more data = more useful." For LLM tool responses, more data = more tokens re-read on every future turn. Reverse the instinct: start with the minimum viable response, and add fields back only when you have a concrete case where the agent gave a wrong or incomplete answer because it lacked that specific data.

## The Rule of Thumb

> Strip a field only if there is **no** scenario where an LLM would use it to answer a user question. If the LLM doesn't need it to answer the question, it shouldn't be in the tool response.

`helm.sh/chart` is always worth keeping — it answers a real, recurring question and isn't derivable from anything else. `last-applied-configuration` is always worth stripping — it's a complete duplicate of data already present elsewhere in the resource. Apply that test field-by-field, and the rest — the blocklists, the conditional excludes, the kind-specific shapes — falls out naturally.
