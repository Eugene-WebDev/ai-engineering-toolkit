# AI Engineering Toolkit — Custom Claude Code Skills + MCP Integrations

A demonstration of how I build with agentic AI: a personal toolkit of **custom Claude Code skills** and **MCP (Model Context Protocol) servers** that turn an LLM into an operator for real infrastructure, APIs, and workflows — with human-in-the-loop control.

> **Philosophy:** AI drafts, I own the architecture and verification. Every solved problem becomes a reusable skill, so each project starts from a higher baseline.

---

## Custom Claude Code Skills

> Five of them are published in [`skills/`](skills/): **llm-integration-patterns**, **langgraph-langchain**, **mcp-server-build**, **portable-skills**, **website-architect**.

20+ skills I authored — reusable, version-controlled, and portable across machines (macOS ↔ Linux). A representative slice:

**AI / automation engineering**
- **n8n-workflow-build** — designs, validates, and pushes *new* n8n workflows via the official n8n MCP (form triggers, AI agents, HITL approvals, multi-stage pipelines).
- **langgraph-langchain** — code-level reference for LangChain/LangGraph v1.x (verified against the docs and smoke-tested): create_agent + middleware, durable checkpoints, interrupt/resume approval gates, retries/timeouts/error handlers, streaming, memory, multi-agent, MCP, deployment — plus when to stay in n8n instead.
- **n8n-mcp-setup** — deploys/troubleshoots a self-hosted n8n-MCP bridge for *surgical* workflow edits over the n8n REST API.

**Infrastructure & ops**
- **headless-server-build** — end-to-end playbook for a headless Linux server: blind-install workarounds, networking via Tailscale, headless device-code auth.
- **delegate-hp** — delegates a self-contained task to a Claude instance on a remote server over SSH/Tailscale; runs locally there and returns the result. Read-only by default.
- **backup** — timestamped, self-contained snapshots with restore instructions.

**Databases & web**
- **sqlite** / **turso** — production playbooks (WAL/concurrency/footguns; distributed-SQLite trade-offs).
- **website-architect** — designing sites to be agent-friendly (screenshots, raw HTML, accessibility tree).

**Authoring & utilities**
- **portable-skills** — patterns for skills that run on any OS (capability detection, layered secret lookup, graceful degradation).
- **yt** (YouTube transcript→summary), **md-fetch**, **tts** (voice output), **project-playbook**, and more.

*What this demonstrates: skill design, capability detection, disciplined secret handling, and turning repeatable work into reusable tooling — the same patterns a team wants for internal AI tooling.*

---

## MCP Server I built

- **Custom n8n MCP server** — a TypeScript/Node MCP server (built on the official MCP SDK) that lets Claude manage a self-hosted n8n instance in natural language: list, create, update, activate workflows. Runs locally; credentials in env; no third-party data sharing.

## MCP Servers I deploy & integrate (third-party)

Honest distinction — self-hosted and wired into my workflows, but authored by others:
- **czlonkowski/n8n-mcp** — self-hosted on my VPS (patched docker-compose + Traefik); my deploy config is version-controlled.
- **chigwell/telegram-mcp** — Telethon-based; run locally to let Claude read and triage documents from Telegram.

---

## How it fits together

A trigger hands a task to an agent. The agent runs against real systems (n8n, a VPS, a database, Telegram) through MCP tools, reaches an **approval gate**, a human approves or returns it with feedback, and the next step runs autonomously — with logging and verification throughout. The reusable skills mean I don't rebuild the plumbing each time.

**Stack:** Python (FastAPI / Django) · TypeScript / Node · n8n · Make.com · LangChain / LangGraph · MCP · RAG · Docker · self-hosted Linux / Tailscale.

---

*Eugene Melnychenko — AI Automation & Integrations Engineer · [linkedin.com/in/eugene-webdev](https://www.linkedin.com/in/eugene-webdev)*
