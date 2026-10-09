# 00 — Executive Summary

## What this system is

**Swarn** (package name `swarn`, version 2.0.0 per `pyproject.toml`) is a from-scratch
autonomous AI engineering agent focused on machine-learning and data work. It is two Python
packages — `agent/` (the loop and everything it depends on) and `swarn/` (self-contained
*capabilities* the agent merely registers) — offering **two complementary agent paradigms**
plus a multi-agent coordination layer:

> **2026-09-21:** paradigms 1 and 3 below were **removed**. The specialist team (4) is the only agent
> engine and the interactive REPL is the only way to run it; the tree search (2) remains as
> the team's `solve_ml_task` tool — the ML specialists' default for "best model" requests since
> 2026-10-05, when the `swarn solve` subcommand was removed. The descriptions are kept as history.

1. **A ReAct tool-calling loop** (`agent/core/agent_loop.py`, removed) — the LLM is given a registry of
   **104 tools** (file I/O, sandboxed code execution, repo-RAG semantic search, a full tabular-ML
   pipeline from ingestion to deployment packaging, a human-in-the-loop cleaning/analysis/
   reporting layer, multimodal indexing, LoRA fine-tuning,
   MCP client integration) and iterates *think → call tool → observe result* until it calls
   `finish_task`, errors out, or hits an iteration cap. Full catalog:
   [../TOOLS.md](../TOOLS.md) and [09_Tool_Execution.md](09_Tool_Execution.md).

2. **An AIDE-style solution tree search** (`agent/search/`) — for "build the best model on
   this data" tasks. The engine drafts several complete solution scripts, executes each in a
   sandbox, has a second LLM call review the result (`{is_bug, metric, ...}` via forced tool
   call), then iteratively **debugs** buggy leaves or **improves** the best node until a
   step/time/token budget is exhausted. Every attempt is a node in a persisted tree
   (`runs/<id>/journal.json`), enabling crash-safe **resume**.

3. **A fixed multi-agent pipeline** (`agent/core/orchestrator.py` + `agent/core/roles.py`, removed) —
   Planner → Coder → Reviewer → Tester, where each role is an `AgentLoop` instance with a
   role-specific system prompt and a restricted tool allow-list, coordinated through a small
   "blackboard" of role summaries. Kept for direct callers; superseded by (4).

4. **A multi-agent team runtime** (`agent/multiagent/`, added 2026-09-21) — fourteen
   specialist agents (analysis, ML building, LLM evaluation, debugging) as data-defined `ToolAgent`s over the same registry; workflow, conversation
   and plan-based coordination patterns chosen per task by a router; streaming events,
   cancellation tokens, pause / inject / switch, stateless human-in-the-loop exits, middleware
   (logging, PII, rate limits, audit), a long-term memory system (`agent/memory/`), and
   provenance views. Reached through the interactive REPL (the only way to give it a task),
   and — for observation and background jobs — `/api/jobs`, `/api/team/*` and the MCP server.
   See [24_Multi_Agent_Runtime.md](24_Multi_Agent_Runtime.md) and
   [25_Memory_System.md](25_Memory_System.md).

## How it is used

Four entry points (see [03_Startup_Sequence.md](03_Startup_Sequence.md)):

| Entry point | File | Purpose |
|---|---|---|
| `swarn` / `python main.py` | `agent/cli.py` | Interactive REPL — the only way to run the team. `main.py` is a thin shim forwarding argv — there is one REPL implementation, in the package. |
| `swarn <cmd>` | `agent/cli.py` | Utility subcommands (no agent task runs headless): `sessions`, `prune-sessions`, `recall`, `index`, `playbook`, `watch`, `eval`, `config`, `web`, `serve`, `mcp-serve`, `guardrail-benchmark` |
| `swarn web` | `webui/server.py` | The browser equivalent of the REPL: conversations on the team, approval and question cards, steering bar, inspector (plan / story / report / trace / files), history, settings. One person, one run at a time |
| `swarn serve` | `agent/web/dashboard.py` | FastAPI web dashboard: live websocket step feed, session/search-run browsing, playbook view |
| `swarn mcp-serve` | `agent/integrations/mcp_server.py` | MCP server exposing the platform (submit/status/messages/list) to Claude Code, Cursor, etc. |

## Key design decisions (verified in code)

- **One deployed LLM endpoint.** Every LLM call in the codebase is hard-routed to a single
  OpenAI-compatible endpoint routed by `agent/llm/router.py` (set through the `SWARN_DEPLOYED_*`
  env vars, usually in `.env`; there are no in-code defaults). All `--model` flags and
  model parameters are display-only. There are no exceptions.
- **The tool registry is the only thing that grows.** New capability = new `@tool`-decorated
  function in `agent/runtime/tools.py` (or dynamic MCP registration). `AgentLoop`'s control flow is
  never modified for new capabilities.
- **Errors are strings, never exceptions.** `run_tool()` catches everything and returns
  `"Error ..."` strings; the self-correction policy, guardrail scanner, and doom-loop
  detector then *enrich* those strings so the model can react.
- **Docker-optional execution.** `agent/runtime/execution.py` auto-detects Docker; falls back to a
  cross-platform subprocess backend with hard timeouts. Both return a structured `ExecResult`.
- **Per-process singletons.** Nearly every subsystem (session store, data pipeline, feature
  engine, model trainer, evaluator, packager, context engine, MCP manager, sandbox) is a
  lazily-created module-level singleton (`get_xxx()` accessor).
- **Self-improvement across runs.** `agent/memory/knowledge.py` maintains a hard-capped playbook of
  distilled lessons (post-run LLM reflection) and an SQLite FTS5 archive of past runs; both
  are injected into future search prompts.
- **Safety layers on the ReAct loop.** Self-correction hints + abort budget
  (`self_correction.py`), prompt-injection scanning of tool results (`observability.py`),
  doom-loop repetition detection (`doom_loop.py`), deterministic context compaction
  (`agent_loop.py`), optional OpenTelemetry tracing.

## Scale of the codebase

~11,000 lines of Python. Largest modules: `tools.py` (1,243 — mostly declarative tool
schemas), `dashboard.py` (505, includes embedded HTML), `model_training.py` (479),
`mcp_integration.py` (440), `multimodal_rag.py` (428). Tests: 10 files, run by a
zero-dependency runner (`tests/run_tests.py`) or pytest.

## What is NOT in this system (verified)

- No database beyond SQLite (knowledge archive) and JSON files (sessions, journals, index).
- No authentication/authorization on the dashboard or MCP server (localhost tooling).
- No async *core*: the LLM client and the tool layer are synchronous. The multi-agent
  runtime adds an async surface (`arun` / `arun_stream` bridge a worker thread) and
  event streaming, but not an async tool layer.
- No fine-tuning of the main LLM — `finetuning.py` trains small *local* HuggingFace models.
- No token-level streaming of LLM responses — all calls are blocking request/response
  (`BaseLLMClient.call()`); progress streaming is at the event level (per model call,
  tool call, step).
- Cost tracking only when prices are configured (`SWARN_PRICE_*`; `Usage.cost()` returns
  None otherwise — an unpriced endpoint reports nothing rather than zero).
