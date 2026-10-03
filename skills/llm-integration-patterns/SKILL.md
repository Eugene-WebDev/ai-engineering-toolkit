---
name: llm-integration-patterns
description: Decision guide for choosing an LLM/agent architecture in production and wiring in memory/retrieval. Picks between the four integration patterns (chained requests, single agent with tools, multi-agent gatekeeper, multi-agent team) using explicit "use when / avoid when" rules, then wires RAG, vector stores, and MCP — mapped to concrete n8n nodes. Also covers production hardening: resilience/fallback patterns (retry+backoff, timeout, circuit breaker, key rotation, provider failover, failure classification), cost-based model routing (classify complexity → cheapest capable model), auditability + observability (the three logging layers, EU AI Act / SOC2, LangSmith/Langfuse tracing), input/output guardrails (the validator sandwich + structured outputs), cost gating (rate limit -> cache -> queue -> circuit breaker), self-hosted serving with vLLM (continuous batching, PagedAttention, prefix caching) for concurrent agent workloads, and self-hosting the routing/compression layer itself (9router tiered fallback, Headroom reversible context compression). Use when scoping ANY new LLM feature or automation and deciding "how should the model connect to the rest of the system", when a workflow is getting complex enough that a single prompt no longer fits, when someone asks which agent architecture to use, or when making an AI automation reliable/auditable for production. Complements n8n-workflow-build (that skill pushes a workflow; this one decides its shape) and langgraph-langchain (code-level LangChain/LangGraph how-to).
---

# LLM Integration Patterns

Pick the **simplest** architecture that fits, then add memory/retrieval only where it earns its cost. Most teams over-engineer this and rebuild later. Source: n8n "Patterns that work in production" (2026-05-20).

## The core rule

> Start with **chained requests** or a **single agent**. Escalate to multi-agent only when the simpler pattern demonstrably hits its limit. More coordination = more latency, more failure surface, harder debugging. Starting simple gives you a working system to learn from and a clear signal when you've outgrown it.

## The four patterns

### 1. Chained requests — fixed sequence, flexible components
A predetermined sequence of model calls; each output feeds the next. No routing logic.
- **Use when:** steps are known/ordered/stable; different models or modalities per stage; you want per-step debugging and iteration.
- **Avoid when:** input variability changes the number/order of steps; the model must decide the next action from intermediate results.
- **Main risk:** error propagation — a bad output at step 2 silently reaches step 3. **Validate output between every stage.**
- Example: transcribe → summarize → classify → store.

### 2. Single agent with tools — runtime reasoning for variable tasks
LLM is a reasoning engine that chooses which tool to call and in what order, looping until done.
- **Use when:** step count/order varies by input; the task needs to evaluate intermediate results before deciding next steps.
- **Avoid when:** the sequence is predictable and fixed; debugging transparency is mandatory.
- **Main challenge:** the model picks its own path → harder to debug. Needs a **scratchpad** (temp state for one run); watch its size against the context window.
- Example: research agent — search, summarize, decide whether to search again, format.

### 3. Multi-agent gatekeeper — centralized routing, distributed execution
A coordinator classifies each request and delegates to specialized sub-agents.
- **Use when:** diverse request types needing different models/data; you need to add capabilities incrementally without restructuring; centralized control + consistent interface matter more than speed.
- **Avoid when:** routing is simple enough for a single agent with tools; cross-step latency is a hard constraint.
- **Tradeoff:** the gatekeeper is load-bearing — misclassification/failure hits everything downstream; round-tripping through it adds latency.
- Example: support system with billing / technical / escalation sub-agents; new category = new sub-agent, gatekeeper unchanged.

### 4. Multi-agent team — distributed coordination, no central control
Role-specialized agents talk directly to each other on a shared task.
- **Use when:** the task is too complex/open-ended for one agent within context limits; needs autonomous operation over long horizons with distributed decisions.
- **Avoid when:** a gatekeeper already handles routing; you lack observability tooling to trace multi-agent behavior.
- **Main cost:** no central control → debugging means tracing interactions across agents. Only do this with real tracing in place.
- Example: planning + research + writing + review agents producing shared output.

### Quick selector

| Dimension | Chained | Single agent | Gatekeeper | Team |
|-----------|---------|--------------|-----------|------|
| Step predictability | Fixed | Variable | Varies by type | Highly variable |
| Input variability | Low | High | Med–High | High |
| Debugging complexity | Low | Medium | Med–High | High |
| Context management | Simple | Critical | Moderate | Complex |

Decision shortcut: **Fixed steps? → Chained.** Variable path, one domain? → **Single agent.** Many domains, stable routing? → **Gatekeeper.** Open-ended, no clean router, and you have tracing? → **Team.**

## Adding memory & retrieval

LLMs have no built-in memory; **vector stores** add it. (freeCodeCamp, vector-stores 2026-06-03; RAG+MCP 2026-05-24.)
- **RAG** = retrieve relevant chunks → inject into context. Fundamentals that matter: chunking strategy, embeddings, and a retriever the agent can call as a *tool*.
- **MCP (Model Context Protocol)** exposes tools/data sources to the agent through a standard interface — pairs naturally with the gatekeeper/single-agent patterns.
- **Document-heavy pipelines:** LlamaParse (verified n8n community node, 2026-06-30) consolidates parse / classify / extract / split / retrieve behind one credential, each a callable agent tool — good for "classify a doc, then extract fields" routing.
- **Browser-aware agents:** Safari MCP (freeCodeCamp 2026-05-28) bridges an agent to a logged-in browser session vs. headless tooling.

## Mapping to n8n

n8n implements all four patterns in one canvas:
- **AI Agent node** (tool calling) → single-agent and the building block of multi-agent.
- **AI Agent Tool node** (an agent callable as a tool) → simplifies the gatekeeper.
- **Call n8n Workflow Tool node** → turn any workflow into an agent tool; add capability without rebuilding the pipeline.
- **Sub-workflows** → keep each chained-request step modular and independently debuggable.
- **Error-handling workflows** → catch step-level failures before they propagate (this is the fix for Pattern 1's core risk).

For actually building/pushing the workflow, hand off to the `n8n-workflow-build` skill.

## Production reliability — fallback patterns

Whichever pattern you pick, in production it *will* hit a failing LLM/tool/retrieval call. Resilience means **containing failure, not eliminating it** — accept a degraded response over a dead pipeline. (Source: n8n "Designing Resilient AI Automation", 2026-07-03.)

Three core patterns:
- **Retry with exponential backoff** — re-attempt transient errors at doubling intervals; add **jitter** so many clients don't retry in a synchronized wave.
- **Timeout** — a deadline per call so a stalled LLM doesn't block the thread; calibrate against the provider's SLA to avoid false failures that trigger needless retries.
- **Circuit breaker** — once failure rate crosses a threshold, stop routing. States: Closed (normal) → Open (blocked) → Half-Open (test recovery). Stops one bad dependency cascading through the pipeline.

**Classify the failure — it decides the response:**
- `429` rate limit → retry with backoff (transient, per-key)
- `429` quota exhausted → **don't** retry (permanent until billing resets)
- `5xx` → provider-side → consider provider failover
- auth/billing → permanent, needs human intervention

**Fallback mechanisms:** key rotation (cycle multiple API keys past a rate limit) and provider failover (switch to a secondary model/provider). Both only work with accurate failure classification.

**In n8n:** Retry on Fail + Enable Fallback Model + Error Trigger implement all three with no custom code — see the `n8n-workflow-build` skill's "Production hardening" section for the node-level wiring (not repeated here).

## Cost-based model routing

Failover keeps you alive; **routing keeps you cheap.** Don't send every request to your strongest model — classify first, route to the cheapest model that can handle it, and keep the failover chain underneath. (Source: freeCodeCamp "Build AI Applications That Switch Models Automatically", 2026-07-29.)

Three tiers:
1. **Classify complexity offline (<5ms, no API call):** word-count bands (e.g. SIMPLE <80 / MEDIUM 80–300 / COMPLEX >300), code-block detection (```), and keyword regex (refactor, debug, algorithm…). Deterministic — never spend an LLM call to decide which LLM to call.
2. **Routing table:** map each tier → {primary, fallback}. e.g. SIMPLE → gpt-4o-mini / haiku; COMPLEX → sonnet / gpt-4o. Add a **local tier** (Ollama, via the `local-llm-ollama` factory) for cheap/private tasks.
3. **Execute with fallback** (reuses the failover section above) + a 5–10s timeout so a hang doesn't block the fallback.

Discipline: keep classification <5ms; **normalize every provider's output into one schema** so downstream code is provider-agnostic; log the routing decision + cost delta per request to tune thresholds.

## Auditability — log three layers

For any agent that makes consequential decisions (and mandatory for EU AI Act high-risk domains: hiring, credit, health, insurance), log enough to reconstruct *why* it acted. (Source: n8n "Audit AI Chatbot Decisions", 2026-06-26.)
- **Layer 1 — what the model received:** write user input + start timestamp *before* the agent runs, so a failed run still leaves a partial audit record proving it initiated but never returned.
- **Layer 2 — what it decided:** enable **"Return intermediate steps"** to capture tools called, params, tool responses, final output; require the model to **justify** its decision (info used, alternatives considered) and store that.
- **Layer 3 — what it invoked:** already inside the intermediate steps (tool name + exact params + response). Your audit responsibility ends at invocation; downstream systems own their trails.

**Use a dedicated audit store, not execution/provider logs:** execution retention prunes fast and cross-run filtering is limited; provider logs (OpenAI ~30d / Anthropic ~7d) are for abuse monitoring, stop at the model boundary, and are empty under Zero Data Retention. In n8n this is a Data Table — see `n8n-workflow-build` "Production hardening" for the node wiring.

**Auditability (the compliance "what") vs observability (the debugging "why") are different jobs — run both.** The 3-layer audit trail answers "what did the agent decide and can I prove it 6 months later." A tracing tool answers "why is this slow/failing right now." (Source: freeCodeCamp "Trace and Monitor AI Agents with LangSmith", 2026-07-22.)
- **LangSmith** (LangChain's platform): Project → Trace (one request) → Run (one LLM/tool step) → Thread (session). Captures request/response, tool calls+args, token use, per-step latency, errors, plus a metrics dashboard with regression alerts. With LangChain's `create_agent` it auto-traces via env vars (`LANGSMITH_TRACING/API_KEY/PROJECT`) — **no code changes**.
- **Langfuse** — open-source, self-hostable alternative (traces/sessions/metadata/dashboards); use when data can't leave your infra (pairs with `local-llm-ollama`).

## Guardrails — wrap the probabilistic core in deterministic code

The model is ~20% of the solution; the system around it is the other 80%. "Use a better model" is not a reliability strategy. (Source: freeCodeCamp "How to Build Reliable AI Systems", 2026-08-26.)

**Validator sandwich pattern** — three layers, deterministic on both ends:
1. **Top bun — input guardrails.** Validate *before* spending a token: type checks, format checks (email/URL regex), range checks (min/max body length). Reject garbage immediately — fail fast and cheap. Return a typed, normalized object (trim, lowercase) so the prompt always sees the same shape.
2. **Meat — the LLM.** Force **structured outputs** (JSON schema / tool-call mode), never free text you plan to regex later. The classic failure: the same "extract the customer email" prompt returns `john@example.com` on Monday, `"Customer email: john@example.com (verified)"` on Tuesday, a bare name on Wednesday, and a nested object on Thursday. All arguably correct, none programmatically parseable.
3. **Bottom bun — output guardrails.** Re-validate the model's output against the schema, check enum membership and value ranges, and reject/retry on violation. Never let unvalidated model output reach a routing decision or a downstream API.

**Sharpest instance — text-to-SQL:** when the model's output is an executable query, the output guard is the only thing between a question and an arbitrary write. Read-only connection, single-`SELECT` allowlist, enforced `LIMIT`. See the `sqlite` skill.

**The deterministic rule:** if a decision *can* be made with `if/else`, it **must** be. Only send to the model what genuinely requires judgment. Every rule you move out of the prompt and into code is a failure mode you deleted.

### Model the gap, not just the answer

A retrieval pipeline that returns only *what it found* quietly implies the rest was
checked and clean. Make absence a first-class output instead.
(Source: freeCodeCamp, "Market Research Copilot with MCP and Python", 2026-09-21.)

Instead of `retrieve → summarize`, sort every signal into **three buckets** before any
verdict is written:

| Bucket | Question it answers |
|---|---|
| **support** | what in the data backs the claim |
| **contradiction** | what in the data weakens it |
| **missing evidence** | what would be needed to judge it, and wasn't there |

Only then assign a verdict. The third bucket is the one that matters: it converts
"the model didn't mention it" into "the system states it was unavailable" — which is
the difference between a confident-sounding answer and an auditable one. It also gives
the user a concrete next action (go find *that*), which a summary never does.

Applies well beyond finance — any time the input is a **claim to test** rather than a
topic to describe: contract review, diagnostics, compliance checks, research memos.

## Gated pipeline — cost control before the call

Uncontrolled spend is a first-class failure mode, not a billing footnote. Put four gates in front of every LLM call, in this order:
1. **Rate limit** — per-user/per-tenant cap, enforced before anything else.
2. **Cache check** — return the stored answer for a repeated/near-identical request; the cheapest call is the one you don't make.
3. **Request queue** — smooth bursts instead of fanning them straight at the provider.
4. **Circuit breaker** — the same breaker from the failover section, doubling as a spend guard: a failing provider you keep retrying burns money as well as latency.

Pair with the **silent-failure** guard: an AI pipeline that returns *something* on every path hides its own breakage. Log at each stage boundary and alert on distribution shifts (schema-violation rate, fallback rate, empty-result rate) — not just on exceptions.

## Self-hosted serving — when provider APIs stop fitting

Agents multiply inference: one user turn can become 10–20+ model calls (plan → select tool → interpret result → summarize → recover → decide → answer). Request shapes are also uneven — a short question next to a long system prompt + history + retrieved docs + tool results. That's a dynamic workload, not a queue of similar jobs. (Source: freeCodeCamp "How to Scale LLM Inference for AI Agents Using vLLM", 2026-08-27.)

**vLLM** is an OpenAI-compatible serving layer that separates agent logic from inference infrastructure (HTTP call instead of in-process `model.generate()`). Four features that matter for agents:
- **Continuous batching** — the active batch is updated as requests arrive/finish, so a short request isn't stuck behind a long one.
- **PagedAttention** — KV cache in fixed-size blocks instead of one contiguous region per request; less fragmentation, easier reuse.
- **Automatic prefix caching** — requests sharing a prompt prefix (system prompt, tool definitions, conversation history, retrieved docs) reuse KV blocks. Cuts *prefill* work only — it does not speed up new-token generation, so the payoff scales with prefix length. Agent workloads have unusually long shared prefixes, which is exactly why this pays off here.
- **OpenAI-compatible API** — existing agent frameworks point at it with a base-URL change.

**Use when:** concurrent multi-user agent traffic, long shared prefixes, data that can't leave your infra, or provider per-token cost dominating. **Skip when:** single-user prototype or low/bursty traffic — a provider API is cheaper than a GPU sitting idle. For single-user local/private work use `local-llm-ollama` instead; vLLM is the concurrent-serving tier above it.

## Self-hosting the routing and compression layer

The cost-routing and failover sections above describe the *architecture*. These are the
concrete components that implement it on your own box — relevant when provider spend
dominates, or when data can't leave your infra. (Source: Fireship, "5 open source tools
that replaced my $320/mo AI stack", 2026-09-07.)

```
application / agent
      │
   Headroom      ← context compression (before tokens are billed)
      │
   9router       ← one OpenAI-compatible endpoint, tiered fallback
      ├─ tier 1: the subscription you already pay for (Claude Max, GPT Pro)
      ├─ tier 2: cheap pay-per-token backup
      ├─ tier 3: free providers / trial credits
      └─ Ollama  ← local, zero inference cost, prompts never leave
```

**9router** (`decolua/9router`) — self-hosted gateway exposing one OpenAI-compatible
`localhost` endpoint instead of juggling many keys; fronts Claude Code, Codex, Cursor,
Cline, Copilot. Its distinctive feature is the **tiered fallback ladder**: exhaust the
subscription tier and traffic drops to the next tier automatically. This is the
*subscription-first* variant of the cost-routing table above — route by remaining quota,
not only by task complexity. Also tracks usage and compresses tool output.

**Headroom** (`chopratejas/headroom-*`) — **context compression as its own layer**, sitting
between the app and the provider, squeezing tool outputs and logs *before they become
billable input tokens*. The canonical waste case: an agent reads 50k lines of
`package-lock.json` to conclude a dependency is missing.

> **The design property that matters: the compression is reversible.** The original is
> cached locally, so the model can pull it back if it genuinely needs it. That makes this
> a *deferral* of cost, not a loss of information — and it is why compression can be
> applied aggressively by default rather than tuned per call.

Pair it with the **gated pipeline** above: the gates stop calls you shouldn't make,
Headroom shrinks the ones you do.

**When to self-host this layer:** provider spend dominates · data can't leave your infra ·
you already pay for a subscription whose quota goes unused · you want one endpoint across
many agent tools. **When not to:** single-user prototype — a provider API plus the
cost-routing table is simpler and cheaper than another service to operate.

**Adjacent, not part of the routing layer:** *Dify* (`langgenius/dify`) — visual workflow
builder that exposes a pipeline as an API; an alternative to n8n when the deliverable is
an API endpoint rather than an automation. *OpenHands* (`OpenHands/OpenHands`) — autonomous
coding agent, strong on SWE-bench Verified, self-hostable against cloud or local models.

## Anti-patterns
- Reaching for multi-agent because it sounds capable — pay the coordination tax only when forced.
- Chained pipeline with no inter-step validation → silent error propagation.
- Single agent with an unbounded scratchpad → context-window blowups.
- Gatekeeper doing real work instead of just routing → single point of failure.
- Multi-agent team with no tracing → undebuggable.
- No fallback/timeout in production → one rate-limited provider takes down the whole pipeline.
- Retrying a permanent failure (quota exhausted, auth) → wasted calls, masked root cause.
- Consequential agent with no audit trail → can't explain a decision when it matters (or when regulators ask).
- Parsing free-text model output with regex instead of demanding a structured schema → breaks silently on the next model version.
- Asking the model to decide something `if/else` could decide → paying tokens for nondeterminism.
- No spend gate in front of the model → a retry loop or a traffic spike becomes an invoice.
- Standing up a GPU serving stack for a single-user prototype → idle hardware beats no hardware only in a demo.
- Paying per token for context the model never needed — no compression layer in front of a tool-heavy agent.
- Running a subscription to its cap while a cheaper tier sits idle, because routing keys off task complexity only and never off remaining quota.
- Returning only what was found, with no "missing evidence" channel → silence reads as confirmation.
