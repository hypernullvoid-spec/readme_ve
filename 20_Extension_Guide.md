# 20 — Extension Guide

How to extend each subsystem, following the conventions the codebase already uses.

## Add a new tool (the primary extension point)

1. In `agent/runtime/tools.py` (or a new module imported by it):

```python
@tool(
    description="One paragraph the model reads to decide when to use this.",
    schema={
        "type": "object",
        "properties": {"arg": {"type": "string", "description": "…"}},
        "required": ["arg"],
    },
)
def my_tool(arg: str) -> str:
    from agent.my_subsystem import get_my_subsystem   # lazy import
    return get_my_subsystem().do(arg)                  # return str; never raise
```

2. Conventions to follow (all load-bearing):
   - **Return strings; never raise.** Errors as `"Error: …"` so the correction policy
     classifies them.
   - **Lazy-import** the subsystem inside the function body.
   - Resolve user paths with `_safe_path` if the tool touches workspace files.
   - Heavy subsystems get a module with a class + `get_xxx()` singleton accessor
     (copy the pattern at the bottom of any capability module).
3. Optionally document the tool + workflow in `prompts.SYSTEM_PROMPT`'s catalogue (the
   model behaves better with the workflow hints) and add it to the relevant role lists in
   `roles.py` if team mode should see it.
4. Nothing else: `AgentLoop` picks it up automatically via `get_tool_definitions()`.

**Alternative: lazy registration.** If the tool pulls in a heavy dependency (matplotlib,
scipy, pdfplumber), don't decorate it in `tools.py` — importing `tools.py` must stay cheap
for a caller that only wanted `read_file`. Follow the analyst modules instead: put the tools
in their own module with a `_SWARN_TOOLS` dict of `name -> (description, schema, fn)` and a
`register_*()` function that inserts them into `TOOL_REGISTRY`, skipping names already
present so a repeat call is a no-op and a built-in wins any collision. Then call
`register_*()` from the CLI's boot path.

**Alternative: a capability.** If the unit of work has its own schemas, artifacts and CLI
surface, and should run without an agent at all, it belongs in `swarn/capabilities/` — see
[05_Module_Design.md](05_Module_Design.md). Keep the import direction one-way (`agent/`
imports `swarn/`, never the reverse at import time), import nothing eagerly from
`capabilities/__init__.py`, and reach back into `agent.*` helpers only via a lazy import
inside the function that needs one.

## Add a new role to the team pipeline

1. `agent/core/roles.py`: define `<ROLE>_TOOLS` (allow-list; typos are silently dropped, so
   double-check names) and `<ROLE>_PROMPT = _SHARED_CORE + "━━━ Your role: … ━━━ …"`,
   then add the entry to `ROLES`.
2. `agent/core/orchestrator.py`: wire the role into `run()`'s pipeline — role sequencing is
   explicit code, not configuration. Follow the existing shape: build task string from
   blackboard summaries, `self._run_role("myrole", task, state)`, branch on
   outcome/summary.
3. If the role emits verdicts, keep the `APPROVED/NEEDS_CHANGES/PASS/FAIL` keyword
   convention so `_verdict_is_approval` works, or extend that method.

## Add a new LLM backend

1. `agent/llm/my_client.py`: subclass `BaseLLMClient`, implement
   `_call_api(system, messages, tools, max_tokens, temperature, tool_choice) -> LLMResponse`.
   Convert from/to the Anthropic-style message shape (`base.py` docstring documents it);
   `openai_client.py` is the reference implementation.
2. `agent/llm/router.py`: add a branch in `create_client()` (the current router
   intentionally hard-routes everything to the deployed endpoint — restoring
   spec-based selection means re-introducing parsing here). Keep the client cache keyed on
   your spec.
3. Tool-choice forcing (`{"type":"tool","name":…}`) must be supported if the backend will
   serve the search reviewer / knowledge reflection.

## Change the deployed endpoint

No code: set `SWARN_DEPLOYED_MODEL`, `SWARN_DEPLOYED_BASE_URL`, `SWARN_DEPLOYED_API_KEY`
(env or `.env`); there are no in-code defaults — the banner at the top of
`agent/llm/router.py` says the same.

## Add a new execution backend

1. `agent/runtime/execution.py`: new class with `name`, `exec_python(code, timeout) -> ExecResult`,
   `exec_shell(command, timeout) -> ExecResult`, `close()`.
2. Extend the `ExecutionBackend` union and `make_backend()` selection (plus a
   `SWARN_SANDBOX` value for forcing).
3. Keep `ExecResult.as_text()` semantics (`[exit N]`, "timed out after") — the correction
   policy pattern-matches those strings.

## Add a search-stage or policy tweak

- Tree policy knobs are all in `SearchConfig` — prefer adding a field with an env-backed
  default (pattern: `field(default_factory=lambda: os.environ.get(...))`).
- New stage (beyond draft/debug/improve): add a prompt builder + branch in
  `SearchAgent.propose`, extend `choose_action`, and ensure `Node.stage` values flow into
  `report.py`/`journal.render_tree` (they render any stage string as-is).
- New static-gate rules: extend `static_check()` — only reject *guaranteed* failures
  (documented invariant: "must never veto a script that could have worked").

## Add a memory/knowledge implementation

- **Session storage:** `SessionStore` is instantiable with a custom `sessions_dir`; for a
  different backend entirely, keep `new_session/close_session/list_sessions/get_session/
  recall_as_text/subscribe_to_all_sessions` — that's the surface `AgentLoop`, tools,
  REPL/CLI, and dashboard use. Swap the singleton in `get_session_store()`.
- **Knowledge:** `KnowledgeStore(root=...)` already supports relocation
  (`SWARN_KNOWLEDGE_DIR`). A different store must keep `context_for_task`, `index_run`,
  `add_lessons`, `playbook` (used by runner, CLI, dashboard).

## Add middleware around tool execution

There is no formal middleware chain; the insertion point is `AgentLoop.run()`'s per-tool
block (order matters — see the Phase 4/15 ordering comment) or, for *all* callers
including roles, a wrapper around `tools.run_tool`. Follow the policy-object pattern:
optional constructor arg, `None` = off.

## Add a dashboard endpoint

`agent/web/dashboard.py` — plain FastAPI. Read-only data should come from disk
(`sessions/`, `runs/`, `knowledge/`) or `get_session_store()`; anything triggering agent
work must run on a worker thread to keep the loop responsive (copy `JobRegistry.submit`
or `POST /api/team`).

## Add an MCP-exposed capability

`agent/integrations/mcp_server.py` — add a `@mcp.tool()` function. For long work, follow the
`_TaskRecord` + daemon-thread pattern so the tool returns immediately and is pollable.

## Add configuration

- Env-var: read via `os.environ.get` at *use* time if it should be changeable per call, or
  at import for constants; document it in `agent/config.py` and [11_Configuration.md](11_Configuration.md).
- Search-related: add a `SearchConfig` field and thread it through the `solve_ml_task` tool in `agent/runtime/tools.py`.

## Testing your extension

- Add a `test_*.py` with plain `test_*` functions and register the module in
  `tests/run_tests.py::MODULES` (the runner does not auto-discover files).


## Add a specialist to the team — as a plug-in file (`agent/multiagent/plugins.py`)

No code change. A specialist is one JSON (or YAML) file; the catalogue is rebuilt from the
built-ins plus every plug-in directory at start-up and on `/agents reload`.

1. `/agents new my_agent` at the `swarn` prompt writes `<repo>/specialists/my_agent.json` with
   every field explained (`plugins.write_template()` does the same from code). The fields are
   `AgentSpec`'s: `name` (lowercase identifier), `title`, `group` (a built-in one or a new
   lowercase name — only `generic` means anything: the router never keyword-matches generic
   roles), `description`, `mission` (the role section of the prompt), `tool_categories`
   (names from `agent/runtime/tool_categories.py`; `finish_task` is always added),
   optional `extra_tools` / `excluded_tools`, `capabilities` (substrings the rule-based router
   matches against the task), and the run settings (`approval_mode`, `max_iterations`,
   `memory`, `middleware`, `model`, `temperature`). Keys starting with `_` are comments.
2. Edit the mission and the keywords, then `/agents reload`. The report names every problem
   per file — a misspelt field, an unknown tool category, a name clash with a built-in, a
   keyword set that overlaps another specialist's (the rule-based router would then split
   their tasks) — and loads everything else. `/agents` lists the catalogue with each entry's
   source file; `/agents show my_agent` prints it with the tools it resolved to.
3. Where files live, in load order: `~/.config/swarn/specialists/` (yours, across checkouts),
   then `<repo>/specialists/` (commit these with the project). `SWARN_SPECIALISTS_DIR`
   (`os.pathsep`-separated) replaces the list; empty disables loading. A file with
   `"enabled": false` stays out; one with `"replace": true` takes the place of a built-in (or
   an earlier plug-in) of the same name — otherwise the clash is an error.
4. `team.json` in a plug-in directory is the manifest: `"disabled": [names]` takes built-ins
   out of the catalogue; `"roles": {role: name}` rebinds the **roles** the runtime asks for
   (`specialists.ROLES`: `planner`, `implementer`, `reviewer`, `tester`, `analyst`,
   `investigator`, `data_scientist`, `ml_builder`, `debugger`). The pipeline, the phase loop
   and the router never name a specialist — they call `role("implementer")` and so on — so
   rebinding is how a plug-in takes a built-in's seat. Manifests are merged across directories
   (`disabled` is a union, a later `roles` entry wins) and applied after the spec files, so one
   manifest can bind `implementer` to `my_coder` and disable `coder`.
5. What is refused: removing or disabling a **core** specialist — one a role is bound to —
   because the role would resolve to nothing. The error names the role; rebind it first.
   `is_core(name)` / `roles_of(name)` say which these are. A role name is also an alias:
   `get_spec("implementer")`, a plan assignee of `reviewer`, `/team` with a role name — all
   resolve to whoever fills the role.
6. The web interface shows the same catalogue (`GET /api/meta` → `specialists[].source`,
   `.core`, `.roles`, plus `specialist_roles`, `disabled_specialists`, `plugins`) and
   `POST /api/specialists/reload` re-reads the directories.

### … or in code (`agent/multiagent/specialists.py`)

1. Write an `AgentSpec` and `register_spec(spec)` (or `spec_from_json(dict)`, which validates
   the way the loader does). `unregister_spec(name)` / `disable_spec(name)` remove one —
   never a core one; `enable_spec(name)` restores a disabled built-in; `reset_catalogue()`
   puts the shipped catalogue back. `set_roles({...})` rebinds roles; every target must exist.
2. A built-in belongs in `specialists.py`'s `_add(AgentSpec(...))` list; `tool_names()` must
   resolve only to registered tools — `tests/test_specialists.py` checks every spec; run it.
   If the new built-in is something the runtime should rely on by name, give it a role
   (`DEFAULT_ROLES`) and call `role(...)` at the use site instead of writing its name.
3. Optional: a Skills.md file under `knowledge/memory/skills/` scoped to the agent with
   `<!-- agents: my_agent -->`; middleware configs in `spec.middleware`.
4. Expose it to other agents with `register_specialist_tool("my_agent")` →
   `delegate_to_my_agent`.

## Add an orchestration pattern (`agent/multiagent/patterns.py`)

1. Subclass `Pattern`, set `pattern_name`, implement `run_stream(task, *, control, context,
   decision, answer, parent)`: create the state with `self._state()`, the board with
   `self._board()`, call `self._deliver_notes()` at every boundary, check
   `self._switch_requested(state)`, run children with `result = yield from
   self._stream_child(...)`, handle `result.needs_human` (store `state.pending`, finish
   with the awaiting outcome) and `cancelled`, and end with `self._finish(...)`.
2. Implement `to_config()` / `from_config()` and register with `register_agent_type`.
3. Teach `TaskRouter._rules` / `build()` when to pick it, and add it to `PATTERNS`.

## Add a middleware (`agent/multiagent/middleware.py`)

Subclass `Middleware`, override the moments you need (`before_model`, `after_model`,
`before_tool`, `after_tool`, `on_event`), raise `MiddlewareBlock` to refuse, implement
`to_config()` / `from_config()`, and `register_middleware("MyMiddleware", MyMiddleware)` so
saved agent configurations can restore it.

## Add a new role to the team pipeline — note

The Phase 11 pipeline now also runs on the runtime (`build_team_pipeline()` in
`agent/multiagent/orchestrator.py`, a `SequentialWorkflow` with review gates). A new role
there is a `Step` with an `AgentSpec`; the legacy `agent/core/orchestrator.py` procedure
above still applies to callers that use it directly.
