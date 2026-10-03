---
name: mcp-server-build
description: Build a Model Context Protocol (MCP) server in Python that exposes custom tools to any MCP client (Claude Code, Claude Desktop, Cursor), containerize it with Docker, and wire it in — plus the security hardening most tutorials skip. Use when the task is "build/expose an MCP server", "give Claude/Cursor a custom tool", "wrap this Python function/API/pipeline as an MCP tool", or "connect a local tool to Claude Desktop". This is for AUTHORING a new MCP server from scratch (FastMCP + stdio). It is NOT for the czlonkowski n8n-mcp bridge (use `n8n-mcp-setup`) and NOT for choosing an agent architecture (use `llm-integration-patterns`).
---

# Build an MCP Server (Python + FastMCP + Docker)

MCP = "a USB-C port for AI": one standard interface so any MCP client discovers and calls your tools. Server exposes tools/resources/prompts; the client (Claude Code/Desktop, Cursor) invokes them on the LLM's behalf via **JSON-RPC 2.0 over stdio** (local) or HTTP (remote). The LLM never touches your server directly.

Sources: freeCodeCamp "MCP Server with Python, Docker, Claude Code" (Brahmandam, 2026-03) and "Privacy-first medical de-id agent" (Mahabaleshwara, 2026-08).

## 1. Server skeleton (FastMCP)

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install "mcp[cli]>=1.25,<2"     # pin <2 — SDK v2 changes the transport layer
```

```python
from mcp.server.fastmcp import FastMCP
import sys, json

mcp = FastMCP("my-server")

@mcp.tool()
def do_thing(name: str, mode: str = "auto") -> str:
    """One-line purpose the LLM reads to decide WHEN to call this.
    Args:
        name: what it is (used as ...).
        mode: one of auto|strict.
    """
    # ... real work ...
    return json.dumps({"status": "ok", "result": ...})   # return a STRING (JSON parses reliably)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

Rules that matter:
- **The docstring IS the prompt.** `@mcp.tool()` builds the JSON schema from the signature + docstring; write it for the model, not for humans.
- **Return strings** (JSON strings), not raw objects.
- **Keep the tool set small and focused** — ~6 good tools beat dozens of scattered ones.

## 2. The #1 footgun: never write to stdout

stdio transport carries JSON-RPC on stdout. **Any** stray `print()` corrupts the stream and the client disconnects — the most common "it won't connect" cause. Log to stderr:
```python
print("debug", file=sys.stderr)
```

## 3. Long-running work → background job + poll

Don't block an MCP call on a slow task (it'll time out). Return a job id immediately, let the client poll:
```python
@mcp.tool()
def start_batch(input_dir: str) -> str:
    job_id = uuid.uuid4().hex[:8]
    threading.Thread(target=_worker, args=(job_id, input_dir), daemon=True).start()
    return json.dumps({"status": "started", "job_id": job_id})

@mcp.tool()
def get_job_status(job_id: str) -> str: ...
```

## 4. Privacy pattern: return summaries, not data

If the tool touches sensitive data, **return counts/decisions/output-paths — never the data itself** (pixels, extracted text, PII field values). That's how the model orchestrates a pipeline it's never allowed to see. Remember conversation metadata (file names, paths, counts) still reaches the model → keep secrets out of filenames and error messages; develop against synthetic data.

## 5. Test before wiring

```bash
python server.py                                     # smoke test; silent = correct (awaiting JSON-RPC)
npx @modelcontextprotocol/inspector python server.py # web UI: list tools, invoke manually, see JSON-RPC
```

## 6. Dockerize

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY server.py .
ENV PYTHONUNBUFFERED=1     # else Python buffers stdout → client hangs (works local, breaks in Docker)
CMD ["python", "server.py"]
```
Run with `docker run -i` — interactive keeps stdin open; without `-i` the server gets EOF and exits.

## 7. Wire into the client

Claude Code (terminal):
```bash
claude mcp add my-server -- docker run -i --rm my-server-image
claude mcp list          # shows my-server / stdio
# inside claude:  /mcp   → confirm connected + tools
```
Claude Desktop: add an `mcpServers` entry in `claude_desktop_config.json` (absolute path to the executable/command, `args`, `env`).

## 8. Security — do NOT skip (43% of tested MCP servers had command injection)

MCP runs code from LLM decisions, so **prompt injection is the core threat** (May 2025: a malicious GitHub issue hijacked the GitHub MCP server into leaking private-repo data via a broad PAT). Real CVEs: CVE-2025-6514 (`mcp-remote` command injection), CVE-2025-6515 (`oatpp-mcp` session hijack), MCP Inspector RCE.
- **Limit filesystem access** — mount only needed dirs (`-v $(pwd)/projects:/app/projects`); never `/` or `~`.
- **Validate every input** — reject path traversal (`..`, `/`, `\`); validate enums against a known set; check existence.
- **Least-privilege tokens** — read-only, single-scope; a broad PAT is what turned the GitHub incident into data loss.
- **Keep approval prompts** for any file-modifying/destructive tool.
- **Trust server sources like any code you run** — a fake "Postmark MCP Server" npm package silently BCC'd all mail to an attacker.

## 9. Consuming an MCP server — the client access layer

Everything above is about *authoring* a server. When your app is the **client** —
calling someone else's MCP server from Python — the shape that keeps the codebase sane
is a **thin access layer with nothing but transport concerns in it.**
(Source: freeCodeCamp, "Build a Market Research Copilot with MCP and Python", 2026-09-21.)

```
project/
├── client.py    ← MCP access only: connect, list tools, call with retry/timeout
├── core.py      ← all domain logic; knows nothing about MCP
└── test.ipynb   ← sanity checks per stage
```

```python
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

class RemoteMCP:
    def __init__(self, url):
        self.url = url
        self._tools = None                      # cache — don't re-list per call

    async def list_tools(self):
        if self._tools is not None: return self._tools
        async with streamablehttp_client(self.url) as (r, w, _):
            async with ClientSession(r, w) as s:
                await s.initialize()
                self._tools = [t.name for t in (await s.list_tools()).tools]
                return self._tools

    async def call_tool(self, name, args, trace_id, timeout_s=25, retries=2):
        last = None
        for attempt in range(retries + 1):
            t0 = time.time()
            try:
                async with streamablehttp_client(self.url) as (r, w, _):
                    async with ClientSession(r, w) as s:
                        await s.initialize()
                        out = await asyncio.wait_for(s.call_tool(name, args), timeout_s)
                        return out, {"trace_id": trace_id, "tool": name,
                                     "args": args, "latency_s": round(time.time()-t0, 3)}
            except Exception as e:
                last = e
                if attempt < retries:
                    await asyncio.sleep(0.5 * (attempt + 1))   # linear backoff
        raise last
```

**The three rules that make this worth isolating:**

1. **Return `(result, metadata)`, never a bare result.** The metadata — trace_id, tool
   name, exact args, latency — is what makes a multi-step agent run *inspectable
   afterwards*. Without it, a finished answer gives you no way to ask "which tool
   produced this, with what arguments, and how slow was it?". This is the
   tool-invocation layer of the 3-layer audit trail in `llm-integration-patterns`.
2. **Cache the tool list.** `list_tools()` on every call is a round trip for data that
   changes on deploy, not per request.
3. **Keep domain logic out.** The moment parsing, scoring or prompt-building leaks into
   `client.py`, you can no longer swap transports or mock the server in tests.

⚠️ **Retry/timeout belong here, not in the agent loop.** If the agent retries, it
re-reasons; if the access layer retries, it just re-requests. Only the second is cheap.

## Checklist
- [ ] FastMCP, `mcp[cli]` pinned `<2`, `transport="stdio"`
- [ ] Tools return JSON strings; docstrings written for the model
- [ ] No stdout writes; logs to stderr
- [ ] Long tasks → background job + status tool
- [ ] Sensitive data → return summaries only
- [ ] Tested via MCP Inspector before wiring
- [ ] Dockerfile has `PYTHONUNBUFFERED=1`; run with `-i`
- [ ] Inputs validated; FS access scoped; least-privilege tokens

## Boundaries
- Choosing single-agent vs gatekeeper vs multi-agent, or RAG/memory wiring → `llm-integration-patterns`.
- The czlonkowski n8n-mcp bridge (exposing n8n's REST API as MCP) → `n8n-mcp-setup`.
- Making the server itself portable across machines/OS → `portable-skills`.
