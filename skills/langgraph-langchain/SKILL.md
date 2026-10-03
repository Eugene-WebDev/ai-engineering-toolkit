---
name: langgraph-langchain
description: Current (v1.x, verified 2026-10-03) reference for building agents in Python with LangChain and LangGraph — which layer to pick (Deep Agents vs create_agent vs raw LangGraph vs n8n), create_agent + middleware (HITL, retries, fallback, summarization, PII, call limits), structured output, LangGraph StateGraph with durable checkpoints, interrupt()/Command(resume) human-in-the-loop, 1.2 fault tolerance (RetryPolicy, TimeoutPolicy, error handlers), event streaming (stream_events v3), short/long-term memory (checkpointer vs Store), multi-agent patterns, MCP tools (langchain.mcp), and deployment (LangSmith Deployment / Agent Server). Use when writing or reviewing LangChain/LangGraph code, porting an n8n agent workflow to code, answering "should we use LangGraph here", or when old tutorials use create_react_agent / langgraph-supervisor / chains (those are legacy). Complements llm-integration-patterns (that skill picks the architecture; this one is the code-level how-to).
---

# LangChain + LangGraph (v1.x)

Verified against docs.langchain.com and PyPI on **2026-10-03**. Versions then:
`langchain 1.4.3` · `langchain-core 1.6.6` · `langgraph 1.2.12` · `langgraph-checkpoint-postgres 3.1.2` ·
`deepagents 0.7.21` · `langsmith 0.14.4` · JS: `@langchain/langgraph 1.4.18`, `langchain 1.5.15`. Python ≥ 3.10.

Smoke-tested offline in a clean venv (Python 3.14): every import in this file resolves; interrupt → `version="v2"` interrupts → `Command(resume)`, `stream_events(version="v3").interrupted`, `error_handler` + `RetryPolicy`, async `TimeoutPolicy` → handler, and sync-node timeout → `ValueError` at compile all behave as written below.

**Most tutorials online are pre-v1 and wrong now.** Before trusting a snippet, check it against the "legacy → current" table below.

## 1. Which layer?

They're stacked, not competing: **LangGraph** = runtime (graph, state, checkpoints, interrupts). **LangChain** = agent + model/tool abstractions built on it. **Deep Agents** = batteries-included harness built on LangChain. **LangSmith** = tracing, evals, deployment.

| Need | Use |
|---|---|
| Long multi-step task, files/scratchpad, subagents, planning, human steering | **Deep Agents** `create_deep_agent` |
| A tool-calling agent with some policy (approval, retries, limits, PII) | **LangChain** `create_agent` + middleware |
| Bespoke flow mixing deterministic steps and LLM steps, branching, parallel fan-out, compensation | **LangGraph** `StateGraph` |
| Integrations-heavy business automation, non-dev maintainers, visual ops, webhooks/schedules | **n8n** (keep it there; call a LangGraph service from n8n only for the part that needs code-level control) |

Rule of thumb: start at the highest layer that fits, drop down only when you hit a wall. Don't reach for LangGraph to chain two prompts.

## 2. Legacy → current

| Old (tutorials) | Now |
|---|---|
| `langgraph.prebuilt.create_react_agent` | `langchain.agents.create_agent` |
| `LLMChain`, `RetrievalQA`, `langchain.hub`, old retrievers/indexing | moved to `langchain-classic` (`from langchain_classic.chains import ...`) — don't start new code on it |
| `langgraph-supervisor`, `langgraph-swarm` | not in current docs (last releases 2025); use subagents-as-tools / handoffs / Deep Agents |
| `langchain-mcp-adapters` `MultiServerMCPClient` | `from langchain.mcp import MCPAdapter` (langchain ≥ 1.4, `pip install "langchain[mcp]"`) |
| Reading `result["__interrupt__"]` | `graph.invoke(..., version="v2")` → `GraphOutput(.value, .interrupts)` |
| `stream(stream_mode=...)` branching on tuples | `stream_events(..., version="v3")` typed projections (1.2+) |
| `response.content` parsing per provider | `response.content_blocks` (unified: text, reasoning, tool_call, citations) |
| "LangGraph Platform / Cloud" | **LangSmith Deployment**, runtime = **Agent Server** |

## 3. create_agent + middleware

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    HumanInTheLoopMiddleware, ModelRetryMiddleware, ModelFallbackMiddleware,
    SummarizationMiddleware, ModelCallLimitMiddleware, PIIMiddleware,
)
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="anthropic:claude-sonnet-5-5",
    tools=[read_email, send_email],
    system_prompt="You triage the support inbox.",
    checkpointer=InMemorySaver(),          # required for HITL / resume
    middleware=[
        ModelRetryMiddleware(max_retries=3),
        ModelFallbackMiddleware("openai:gpt-5.4-mini"),
        ModelCallLimitMiddleware(run_limit=15),          # cost/loop guard
        SummarizationMiddleware(model="anthropic:claude-haiku-4-5",
                                trigger=("tokens", 4000), keep=("messages", 20)),
        HumanInTheLoopMiddleware(interrupt_on={
            "send_email": {"allowed_decisions": ["approve", "edit", "reject"]},
            "read_email": False,
        }),
    ],
)
```

Key params: `model`, `tools`, `system_prompt`, `middleware`, `response_format`, `context_schema`, `checkpointer`.

**Prebuilt middleware** (`langchain.agents.middleware` unless noted):
- Reliability: `ModelRetryMiddleware`, `ToolRetryMiddleware` (`max_retries`, `backoff_factor`, `initial_delay`), `ModelFallbackMiddleware`, `ToolErrorMiddleware` (exceptions → error message the model can react to).
- Cost/loop guards: `ModelCallLimitMiddleware`, `ToolCallLimitMiddleware` (`thread_limit`, `run_limit`).
- Context: `SummarizationMiddleware`, `ContextEditingMiddleware` (clear old tool outputs), `LLMToolSelectorMiddleware` (pre-pick relevant tools when you have many), `ProviderToolSearchMiddleware`.
- Safety/control: `HumanInTheLoopMiddleware`, `PIIMiddleware` (`pii_type`, `strategy`, `detector`), `ShellToolMiddleware` (`workspace_root`, `execution_policy`).
- Planning/files: `TodoListMiddleware`, `FilesystemFileSearchMiddleware`; Deep Agents adds `FilesystemMiddleware`, `SubAgentMiddleware`, `RubricMiddleware` (self-grade until rubric passes).
- Testing: `LLMToolEmulator` (LLM fakes tool results — useful for offline evals).

**Custom middleware:** subclass `AgentMiddleware`, implement any of `before_agent`, `before_model`, `wrap_model_call`, `wrap_tool_call`, `after_model`, `after_agent`. `before_*` hooks run in list order. This is where cost-based routing goes (`wrap_model_call` → swap model by complexity) and where audit logging goes (`after_model` / `wrap_tool_call`).

**Structured output** — no extra LLM call:
```python
from langchain.agents.structured_output import ToolStrategy   # or ProviderStrategy
agent = create_agent(model, tools=[...], response_format=ToolStrategy(MyPydanticModel))
```

## 4. LangGraph StateGraph

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.postgres import PostgresSaver

builder = StateGraph(MessagesState)
builder.add_node("draft", draft)
builder.add_node("review", review)
builder.add_edge(START, "draft")
builder.add_edge("draft", "review")
builder.add_conditional_edges("review", route)   # returns next node name or END
with PostgresSaver.from_conn_string(DB_URI) as cp:
    cp.setup()                                   # creates tables once
    graph = builder.compile(checkpointer=cp)
    graph.invoke(inputs, {"configurable": {"thread_id": "job-42"}})
```

- Every node transition is checkpointed → resume after crash, inspect `graph.get_state(config)`, replay `graph.get_state_history(config)`, `checkpointer.delete_thread(id)`.
- Checkpointers: `InMemorySaver` (dev), `SqliteSaver`, `PostgresSaver` (prod), also Redis/Mongo/Oracle.
- Nodes return partial state updates or a `Command(update=..., goto=...)` to update and route in one step. `Send` = dynamic fan-out (map-reduce).

## 5. Human-in-the-loop: interrupt / resume

```python
from langgraph.types import interrupt, Command

def approve(state):
    decision = interrupt({"draft": state["draft"], "ask": "approve / edit / reject?"})
    return {"decision": decision}

cfg = {"configurable": {"thread_id": "post-7"}}
out = graph.invoke(inputs, cfg, version="v2")
if out.interrupts:                       # paused, payload goes to Slack/Telegram/UI
    ...
graph.invoke(Command(resume={"type": "approve"}), cfg)   # later, same thread_id
```

Rules that bite:
1. Needs a checkpointer **and** the same `thread_id` on resume.
2. **The node re-runs from the top on resume** — anything before `interrupt()` executes again. Make it idempotent or move side effects to a later node.
3. Never wrap `interrupt()` in `try/except` (it works by raising).
4. Multiple interrupts in one node are matched **by index** — don't skip one conditionally; no `while True` loops with interrupts inside a node.
5. Payloads must be JSON-serializable.

This is the code equivalent of an n8n "Wait for webhook" approval gate, with the state persisted for you. Retry-with-feedback = `Command(resume={"type":"reject","feedback":...})` → route back to the drafting node with the feedback in state.

## 6. Fault tolerance (langgraph ≥ 1.2)

```python
from langgraph.types import RetryPolicy, TimeoutPolicy, Command
from langgraph.errors import NodeError

def on_fail(state, error: NodeError) -> Command:
    return Command(update={"status": f"compensated: {error.error}"}, goto="finalize")

graph = (StateGraph(State)
    .set_node_defaults(retry_policy=RetryPolicy(max_attempts=3),
                       timeout=TimeoutPolicy(run_timeout=30))
    .add_node("charge", charge, timeout=TimeoutPolicy(run_timeout=120, idle_timeout=30),
              error_handler=on_fail)
    .add_node("finalize", finalize)
    .add_edge(START, "charge")
    .compile(checkpointer=cp))
```

- Order: attempt → timeout (`NodeTimeoutError`, writes of the failed attempt discarded) → `RetryPolicy` → only after retries are exhausted, `error_handler`.
- **Timeouts only work on async nodes** — a timeout on a sync node fails at compile time.
- Error handlers survive host crashes (durable resume). Use them for saga-style compensation instead of try/except inside nodes.

## 7. Streaming

New code (1.2+): event streaming with typed projections.
```python
stream = graph.stream_events(inputs, cfg, version="v3")
for msg in stream.messages:
    for token in msg.text:
        print(token, end="", flush=True)
if stream.interrupted:
    print(stream.interrupts)
final = stream.output
```
Projections: `messages`, `values`, `output`, `subgraphs`, `interrupts`, `extensions`. Async: `await graph.astream_events(...)` and consume projections concurrently with `asyncio.gather`. Sync mixed order: `stream.interleave()`.
Older stream-mode API still works: `graph.stream(..., stream_mode=[...], version="v2")` → every chunk `{"type", "ns", "data"}`; modes `values | updates | messages | custom | checkpoints | tasks | debug`; emit custom progress with `get_stream_writer()`.

## 8. Memory

- **Short-term** (one conversation) = checkpointer + `thread_id`. Keep it bounded: `trim_messages(..., strategy="last", token_counter=count_tokens_approximately, max_tokens=N)` from `langchain_core.messages.utils`, `RemoveMessage(id=...)` to delete, or `SummarizationMiddleware`.
- **Long-term** (across threads/users) = **Store**: `InMemoryStore` (dev) / `PostgresStore`; `graph.compile(store=store)`. Namespaced: `store.put(("user_123","memories"), key, {...})`, `store.search(ns, query=..., limit=3)`. Semantic search: `InMemoryStore(index={"embed": init_embeddings("openai:text-embedding-3-small"), "dims": 1536})`.
- In a node, reach both via `runtime: Runtime[Context]` → `runtime.context.user_id`, `runtime.store.asearch(...)`. Pass per-run context (user id, tenant) through `context_schema`, not through state.

## 9. Multi-agent

Docs' five patterns: **subagents** (main agent calls specialists as tools — parallel, central control), **handoffs** (agents transfer control via tool call — multi-turn with the user), **skills** (one agent loads domain prompts on demand — cheapest for simple tasks), **router** (classify then dispatch — stateless), **custom workflow** (LangGraph graph).
Cost note from the docs: single request ≈ 3 model calls for handoffs/skills/router vs 4 for subagents (aggregation step); multi-domain tasks favour subagents/router (~9K tokens) over skills (~15K, context accumulates).
Default recommendation now: **Deep Agents** for subagents + planning + filesystem context.

```python
from deepagents import create_deep_agent
agent = create_deep_agent(model="anthropic:claude-sonnet-5-5", tools=[...],
                          system_prompt="...", subagents=[...], interrupt_on={...})
```
Built in: `ls/read_file/write_file/edit_file/glob/grep`, `task` (spawn isolated subagent), optional `write_todos` via `TodoListMiddleware`, sandboxed `execute` when the backend supports it; pluggable backends (state, local disk, Store, composite) with permission rules.

## 10. MCP tools

```python
from langchain.mcp import MCPAdapter          # langchain >= 1.4, pip install "langchain[mcp]"
async with MCPAdapter("https://mcp.example.com/mcp") as mcp:   # URL → streamable HTTP
    tools = await mcp.list_tools()
agent = create_agent("anthropic:claude-sonnet-5-5", tools)    # tools keep the client; usable after exit
await agent.ainvoke({"messages": [...]})                       # async only
```
Target types: URL string (HTTP), `Path` (stdio subprocess), in-process `FastMCP` instance (tests), or `{"mcpServers": {...}}` dict for several servers. Auth: see docs page `/oss/python/langchain/mcp/auth`. Building the server side → `mcp-server-build`.

## 11. Observability, evals, deployment

- **Tracing:** set `LANGSMITH_TRACING=true`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT` — `create_agent`/graphs trace with no code change. Self-hosted alternative: Langfuse. Details → `llm-integration-patterns` (observability section).
- **Evals:** LangSmith datasets + evaluators (LLM-as-judge, deterministic checks) run in CI. Same shape as a hand-rolled harness: fixed cases → retrieval / deterministic / judge scores → thresholds fail the build. `LLMToolEmulator` lets the agent loop run offline.
- **Deployment = LangSmith Deployment** (formerly LangGraph Platform); runtime is the **Agent Server** (Assistant → Thread → Run model, `RemoteGraph` client). Options: Cloud (Plus plan+), Hybrid (you host servers + data), Self-hosted with control plane (Enterprise), **Standalone server** container (Docker/Compose/K8s; bring Postgres + Redis + a LangSmith license). Config in `langgraph.json`; CLI `langgraph dev` (local), `build`, `up`, `deploy`.
- **Cheapest self-host without a license:** plain FastAPI around `graph.invoke` + `PostgresSaver` — you lose the managed queue/cron/assistant APIs, keep durability and HITL.

## 12. Gotchas checklist

- `PostgresSaver` import fails with "no pq wrapper available" unless you install `psycopg[binary]` (or system libpq).
- `langchain.mcp` emits `LangChainBetaWarning` and `stream_events(version="v3")` warns too — both are beta; pin and expect changes.
- Pin versions; the API moved a lot between 0.x and 1.x and keeps adding (1.2 timeouts, 1.4 MCP). Re-verify against docs before quoting.
- No checkpointer → no HITL, no resume, no time-travel. Use Postgres in prod, not SQLite on a multi-worker server.
- Side effects before `interrupt()` re-run on resume.
- Sync nodes can't have timeouts.
- Context (user id, tenant, API keys) → `context_schema` / runtime, never into persisted state (it ends up in checkpoints).
- PII in prompts → `PIIMiddleware` or mask before the call (see `local-llm-ollama`).
- Always set a call limit (`ModelCallLimitMiddleware` / recursion limit) on agents that loop.

## Sources
- https://docs.langchain.com/oss/python/releases/langchain-v1
- https://docs.langchain.com/oss/python/langchain/middleware/built-in
- https://docs.langchain.com/oss/python/langgraph/interrupts
- https://docs.langchain.com/oss/python/langgraph/fault-tolerance
- https://docs.langchain.com/oss/python/langgraph/event-streaming
- https://docs.langchain.com/oss/python/langgraph/streaming
- https://docs.langchain.com/oss/python/langgraph/add-memory
- https://docs.langchain.com/oss/python/langchain/multi-agent
- https://docs.langchain.com/oss/python/deepagents/overview
- https://docs.langchain.com/oss/python/langchain/mcp
- https://docs.langchain.com/langsmith/deployments
- https://www.langchain.com/blog/langchain-langgraph-1dot0
