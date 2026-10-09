# 15 — Concurrency

The agent cores are synchronous. Concurrency exists in five well-bounded places.

## 1. Parallel tree search (`search/runner.py`)

- `ThreadPoolExecutor(max_workers=cfg.parallel_workers)`.
- **Shared mutable state:** the `Journal` and the `in_flight` map. One
  `threading.Lock` guards (a) action selection + reservation computation and (b)
  `journal.append` + `journal.save`. Proposal, gate, execution, and review run **outside**
  the lock (that's the parallelism).
- **Reservation-aware policy:** `choose_action(reserved=frozenset(parent_ids in flight),
  pending_drafts=count of in-flight drafts)` prevents two workers debugging the same leaf
  and prevents drafting past `num_drafts`. Verified by
  `test_parallel_resume.py::test_parallel_search_valid_journal` (dense unique steps, exactly
  `num_drafts` roots).
- **Budget exhaustion:** the scheduler stops launching, then *drains* in-flight futures
  into the journal before returning.
- **Thread-safety caveat:** each worker shares the same two cached LLM clients
  (`total_usage.add` is not locked — token counts can race benignly) and the same backend
  (Docker execs serialize on the container; subprocess execs run concurrently in the
  workspace with unique script names).

## 2. Docker exec watchdog (`execution.DockerBackend._run`)

Each exec runs `container.exec_run` in a daemon **watcher thread**; the caller
`join(timeout)`s it. On timeout the container is killed under `_lock`
(`_recycle_container`) — the abandoned thread's eventual result is discarded via the
`slot` dict. Rationale documented: Docker cannot kill an individual exec.

## 3. MCP client bridge (`mcp_integration.py`)

```
caller thread (sync)                     mcp-event-loop thread (asyncio, daemon)
────────────────────                     ───────────────────────────────────────
connect_server ──run_coroutine_threadsafe──▶ _server_task_main  (ONE task per server)
call_mcp_tool  ──_run_coro(_submit_call)───▶   queue.get() → session.call_tool → future.set_result
disconnect     ──queue.put(None)───────────▶   finally: exit_stack.aclose()  (same task)
```

- One persistent event loop in a named daemon thread (`mcp-event-loop`), started
  idempotently under `_lock` with a `threading.Event` readiness handshake.
- **Why one owner task per server:** anyio (under the MCP SDK) binds a context manager's
  cancel scope to the task that entered it; a context opened in one coroutine cannot be
  closed from another, even on the same loop. Hence all calls and the disconnect are routed
  through the owner task's queue. This is documented as "required, not a style choice".
- Sync callers block on `concurrent.futures.Future.result(timeout)`.

## 4. Dashboard sync→async bridge (`dashboard.py`)

- `POST /api/jobs` hands the blocking team run to a `JobRegistry` worker thread, and
  `POST /api/team` to a `TeamSession` thread, so the event loop (and the websocket broadcast
  task) stays responsive. (The blocking `POST /api/run` that offloaded `AgentLoop.run` to
  `run_in_executor` was removed on 2026-09-21.)
- Step delivery: the store subscriber `on_step` runs on the **agent's executor thread**; it
  enqueues via `asyncio.run_coroutine_threadsafe(queue.put(payload), loop)`; the async
  `broadcast_loop` task drains the queue and fans out to websockets, pruning dead ones.
  Mirror-image of the MCP bridge (sync producer → async consumer), and noted as such in the
  module docstring.

## 5. MCP server task threads (`mcp_server.py`)

One daemon `threading.Thread` per submitted task; results written into the shared
`_TASKS[rec.id]` record (insert under `_LOCK`; subsequent mutation single-writer by the
worker thread; readers are the MCP tool handlers). Agent-mode output capture uses
`redirect_stdout` — note this swaps `sys.stdout` **process-wide**, so concurrent tasks'
output can interleave into the wrong transcript (see Tech Debt).

## Locks inventory

| Lock | Guards |
|---|---|
| `runner.run_search.lock` | journal append/save + action selection |
| `DockerBackend._lock` | container create / recycle |
| `MCPManager._lock` | loop startup + `_servers` map |
| `mcp_server._LOCK` | `_TASKS` insertion |
| *(none)* | singleton initializers, `TOOL_REGISTRY` mutation, `Usage` accumulation, `SessionStore` |

## Things that look async but aren't

- `ToolAgent.run_stream`, the patterns, all tools, all LLM calls: synchronous/blocking on
  their worker thread (the `arun*` wrappers bridge to asyncio through a queue).
- `Session.add_step` callbacks: synchronous, on the caller's thread.
- `POST /api/jobs` and `POST /api/team` return immediately; liveness comes from the
  websocket and the status endpoints.

## Background threads at a glance

```mermaid
graph TD
    MAIN[Main thread<br/>REPL / CLI / uvicorn loop]
    POOL[Search ThreadPool<br/>N workers, per run]
    WATCH[Docker exec watcher<br/>per exec, daemon]
    MCPL[mcp-event-loop<br/>1 daemon thread, lazy]
    MCPT[MCP server task threads<br/>per submitted task, daemon]
    EXECU[Dashboard default executor<br/>runs AgentLoop]
    MAIN --> POOL
    POOL --> WATCH
    MAIN --> WATCH
    MAIN -.lazy.-> MCPL
    MAIN --> MCPT
    MAIN --> EXECU
```


## 6. Multi-agent runtime threads (`agent/multiagent/`) — added 2026-09-21

| Thread | Created by | Lifetime | Notes |
|---|---|---|---|
| `team-<id>` | `TeamSession.start()` | one run | drives `agent.run_stream`; events are buffered under `TeamSession._lock` and pushed to listeners on this thread |
| `<parallel>:<branch>` | `ParallelWorkflow.run_stream` | one branch | one per branch (bounded by `max_workers`); each has its own child `RunControl`; events cross a `queue.Queue` back to the pattern's generator |
| `llm-<agent>` | `ToolAgent._call_model` | one model call | the call runs here so the loop can wait on the cancellation token; on cancel the thread is orphaned (daemon) and its result discarded |
| `stream-<agent>` | `BaseAgent.arun_stream` | one run | bridges the sync generator to an async consumer through a queue |
| `job-<id>` (existing) | `JobRegistry.submit` | one job | a `team` job runs `MultiAgentOrchestrator` here with `CancellationToken.from_event(job._stop)` |

Synchronisation added: `CancellationToken._lock` (callbacks/children), `RunControl._lock`
(inbox), `TaskBoard._lock` (RLock, transitions), `FTSMemoryStore._lock` (RLock, writes; one
SQLite connection per operation, closed), `MemoryManager._lock` (RLock),
`TeamSession._lock` (RLock, event buffer and status), `SessionRegistry._lock`,
`AgentTool._lock`, `RateLimitMiddleware._lock`, `AuditMiddleware._lock` (hash chain).

Pause semantics: `PauseGate` is a `threading.Event` held open; a paused worker blocks in
`wait_if_paused()` and a cancel releases it. `PauseGate.on_pause/on_resume` propagate a
parent's pause to child controls.

"Things that look async but aren't", updated: `ToolAgent.run_stream` is a synchronous
generator; `arun`/`arun_stream` are thread bridges, not async I/O. The LLM client and every
tool remain blocking.
