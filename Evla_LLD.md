# Evla_LLD — LLM Evaluation & Benchmarking (`swarn eval`)

Low-level design of `swarn/capabilities/llm_eval/`. Format matches
[05_Module_Design.md](05_Module_Design.md): purpose → public API → internals →
callers → dependencies → error handling → extension points, followed by the
data contracts, sequence diagrams, concurrency model, and test map specific to
this capability. The *why* behind the architecture (build-on vs reimplement,
framework choices) lives in `swarn/capabilities/llm_eval/PLAN.md`; usage in
`swarn/capabilities/llm_eval/README.md`.

---

## 1. Placement & integration contract

The module is a **capability** per `swarn/__init__.py`: self-contained, its own
schemas/artifacts/CLI, registered by the agent, never depending on it at import
time. Every `agent.*` reuse is a lazy import inside the function that needs it.

```
agent/cli.py ──add_typer──▶ swarn.capabilities.llm_eval.cli:eval_app
agent/runtime/tools.py ──@tool──▶ design_llm_eval / evaluate_llm  (wraps this module)
agent/core/approval_policy.py: "evaluate_llm" ∈ SIDE_EFFECTING_TOOLS

llm_eval ──lazy──▶ agent.config (DEPLOYED_* endpoint)        [providers.py: provider="deployed"]
llm_eval ──lazy──▶ agent.llm.create_client                    [autonomous.py: design LLM]
llm_eval ──lazy──▶ agent.runtime.execution.make_backend       [metrics/code_exec.py: pass@k sandbox]
```

Internal dependency graph (module level, no cycles; `metrics/judge.py` is the
only metrics module the engine layer imports back from, and only lazily inside
`run_eval`):

```
paths ◀── config ◀── dataset ◀── validation ◀── progressive
                │        │            │
                ▼        ▼            ▼
            providers  metrics/*   estimate
                │        │            │
                ▼        ▼            ▼
              cache ◀── runner ──▶ report ──▶ gates
                          │          │  ▲
                          ▼          ▼  │
                     autonomous   compare, calibrate, analysis, stats
              harness, synth (leaf utilities)
```

---

## 2. Per-module design

### `paths.py`
- **Purpose:** filesystem identity — run/cache/calibration directories, run ids.
- **Public API:** `eval_base_dir()`, `cache_dir()`, `calibration_dir()`,
  `new_run_id(config_hash) -> "YYYYmmdd-HHMMSS-<hash6>"`, `run_dir(run_id, create)`,
  `find_run(prefix)` (unique-prefix resolution), `sha256_text`.
- **Internals:** repo root derived from `__file__` (`parents[3]`) so the
  capability never imports `agent.paths`; `SWARN_EVAL_DIR` env var overrides
  everything (tests point it at a tempdir). Default base:
  `<repo>/workspace/eval_runs/` (inside the gitignored workspace).
- **Error handling:** `find_run` raises `FileNotFoundError` listing recent run
  ids, or an ambiguity error listing all matches.

### `config.py`
- **Purpose:** the single schema of `eval.yaml` (pydantic v2); identity of a run.
- **Public API:** models `EvalConfig`, `DatasetConfig`, `FieldMapping`,
  `SamplingConfig`, `ModelConfig`, `PricingConfig`, `JudgeConfig`,
  `RubricCriterion`, `MetricConfig`, `ExecutionConfig`, `GateConfig`,
  `ReportingConfig`, `TrackingConfig`, `SafetyConfig`;
  `load_eval_config(path)`, `parse_eval_config(dict)`, `ConfigError`,
  `EXAMPLE_YAML` (the fully commented reference config);
  `EvalConfig.config_hash()` / `.canonical_json()`.
- **Internals / invariants enforced at parse time:**
  - `MetricConfig.name` validated against the metric registry with
    did-you-mean suggestions (`difflib`).
  - `ModelConfig`: `name` defaults to the model id; `n_samples_per_input > 1`
    with `temperature == 0` is rejected (identical samples); provider
    `deployed`/`mock` synthesize a model id.
  - `JudgeConfig._force_determinism`: judge and ensemble temperatures are
    **overwritten to 0.0** — §2.2 determinism is not configurable.
  - `EvalConfig`: judge-backed metric names require a `judge:` section;
    candidate names must be unique (they are cache namespaces).
  - `config_hash()` = sha256 of the sorted-key JSON dump → run id suffix,
    snapshot identity, reproducibility anchor.
- **Error handling:** every pydantic `ValidationError` is flattened to
  `ConfigError` with one `path: message` line per issue (exit-code 2 at the CLI).

### `dataset.py`
- **Purpose:** streaming row access + normalization into `Sample`.
- **Public API:** `Sample` (`id, input, reference, context: list[str]|None,
  turns, trajectory, metadata`; `content_hash` property), `MalformedRow`,
  `iter_rows(cfg) -> Iterator[(index, dict|MalformedRow)]`,
  `map_row(index, row, mapping) -> Sample`, `iter_samples(cfg, skip_ids, keep_ids)`,
  `detect_format(path)`, `dataset_content_hash(cfg)`, `estimate_tokens(text)`
  (chars//4 heuristic — estimates only; real counts come from providers).
- **Internals:**
  - Loaders, all generators: JSONL (line-at-a-time; bad lines yield
    `MalformedRow`, never raise), CSV (`csv.DictReader`; column-count
    mismatches flagged), Parquet (`pyarrow.iter_batches(1024)`), HF
    (`datasets.load_dataset(streaming=True)`), directory (file = sample,
    `<stem>.expected*` sibling = reference), traces (OTLP/OpenInference JSON:
    `resourceSpans→scopeSpans→spans` flattening, attribute list→dict
    normalization, root span → input/output, all spans → `trajectory`).
  - `map_row` field resolution: first present name from each `field_mapping`
    list wins; multi-turn rows take the last user turn as `input`; non-string
    references are JSON-serialized; unmapped scalar fields become `metadata`
    (or exactly `metadata_fields` when set).
  - `Sample.content_hash` = sha256 over (input, reference, context, turns) —
    the sample part of every cache key.
- **Error handling:** `map_row` raises `MalformedRow` naming the fields it
  looked for and the fields the row has; `iter_samples` silently skips
  malformed/empty rows (the validation pass has already counted them).

### `validation.py`
- **Purpose:** the always-first streaming pass (§3.1); doubles as the dataset
  inspection autonomous mode reasons over.
- **Public API:** `validate_dataset(cfg) -> ValidationReport`,
  `ValidationReport` (counters + `errors` fatal list + `summary_lines()` +
  `to_dict()`), `simhash64(text)`, `hamming`, `NearDupIndex`.
- **Internals (memory budget is the design driver):**
  - `simhash64`: char 4-gram shingles → vectorized numpy pipeline
    (`sliding_window_view` → polynomial code → splitmix64 finalizer →
    `unpackbits` vote). Deterministic across processes (no salted `hash()`);
    ~100× faster than a per-shingle digest loop.
  - `NearDupIndex`: 8 bands × 8 bits. Pigeonhole guarantees candidate lookup
    for hamming ≤ 7; 8–12 caught probabilistically (>80%) — acceptable for a
    duplication-*rate* report. Storage: `array('Q')` hashes + id list +
    per-band `array('I')` index → ~50 B/row. `add_and_check` returns the
    earlier duplicate's id **or `None`** (callers must use `is not None`;
    row id `"0"` is falsy).
  - Exact-dup / contamination sets: 8-byte int keys
    (`sha256(text)[:8]`) instead of hexdigests.
  - Label distribution: reference strings ≤ 40 chars counted; tracking stops
    past 10 000 distinct values (free text, not classes); categorical when
    `len(labels) ≤ max(20, √n)` and coverage ≥ 50 %.
  - Contamination: `train_path` is hashed with the *same field mapping*
    (exact keys + its own `NearDupIndex`), then each eval row is probed.
  - Fatal errors appended to `report.errors`: empty dataset, malformed
    fraction > threshold, oversized inputs under `truncation_policy: error`.
  - Verified scale point: 1 M rows / 174 MB JSONL → ~37 s, +183 MB RSS
    (`tests/test_llm_eval_scale.py` enforces `< max(400, 2×file)` MB).

### `providers.py`
- **Purpose:** async model access with rate limiting, retries, timeouts, cost.
- **Public API:** `Provider.call(messages, system, max_retries, meta) ->
  Generation`, `Generation(text, tokens_in/out, latency_s, cost_usd, error,
  cached; .ok)`, `create_provider(ModelConfig)`, `TokenBucket`,
  `resolve_pricing`, `model_family`, `DEFAULT_PRICING`.
- **Internals:**
  - Subclasses implement `_generate` (one raw attempt):
    `OpenAICompatProvider` (openai `AsyncOpenAI`; retries once without `seed`
    if the server rejects it), `AnthropicProvider` (lazy SDK),
    `HFLocalProvider` (transformers pipeline in `asyncio.to_thread`),
    `MockProvider` (behaviours `echo | reference | fixed | fail`; `reference`
    reads `meta["reference"]` — the runner passes the sample's reference so a
    "perfect model" exists for tests/demos), and `deployed` → an
    `OpenAICompatProvider` bound to `agent.config.DEPLOYED_*` (lazy import).
  - `call()` wraps `_generate` with: two `TokenBucket`s (RPM by request, TPM
    by estimated tokens; continuous refill, ~10 s burst capacity),
    `asyncio.wait_for(timeout_s)`, exponential backoff with jitter capped at
    30 s on `_RETRYABLE` markers (429/5xx/timeout/connection), immediate
    failure on `_PERMANENT` markers (context length, auth, quota).
  - **Never raises**: exhausted retries return `Generation(error=...)` —
    partial-failure semantics (§4) are implemented here, not in the runner.
  - `cost_of` = tokens × `resolve_pricing` (config pricing wins; else a small
    prefix table for known hosted models; else $0, flagged "unpriced" in
    estimates).
  - `model_family` buckets model ids (gpt/claude/qwen/…) for the self-judge
    guard.

### `cache.py`
- **Purpose:** content-addressed response cache (§3.2.5) shared across runs.
- **Public API:** `ResponseCache(enabled, base)`, `.get(key) ->
  Generation|None`, `.put(key, gen)`, static `key(model_cfg, messages, system,
  sample_hash, sample_index)`, `CACHE_VERSION`.
- **Internals:** key = sha256 over `{CACHE_VERSION, namespace=candidate name,
  provider, model, mock behaviour, generation params (incl. system + base_url),
  messages, sample content hash, sample_index}`. The **candidate name as
  namespace** is deliberate (§4: per-candidate isolation) and fixed a real
  collision where two mock candidates shared responses. Layout:
  `_cache/<k[:2]>/<k>.json`, atomic tmp-then-replace writes. Failures are
  never cached. Cached hits are re-billed at $0 (`cached=True` in the ledger).
  `sample_index` keys the n-th sample when `n_samples_per_input > 1`; the
  judge appends a hashed `cache_tag` (metric/criterion/judge/attempt) on top.

### `estimate.py`
- **Purpose:** the dry-run estimator behind `swarn eval estimate` and the HITL
  budget gate. Calls no model.
- **Public API:** `estimate_run(cfg, report, rows_override) -> CostEstimate`
  (`models: [ModelEstimate]`, `judge`, totals, `est_minutes`, `summary_lines()`),
  `expected_output_tokens`.
- **Internals:** per candidate: `rows × n_samples × (avg_input + template/system
  overhead)` in, `rows × expected_output` out (references predict answer
  length: `2×avg_ref` clamped to `[16, max_tokens]`; else 256). Judge calls per
  row are metric-aware: `judge_rubric` → `len(rubric)` calls when decomposed ×
  judges × candidates; `judge_pairwise` → `2 × C(n_candidates, 2)`;
  other judge/rag metrics → judges × candidates; plus fixed prompt/output
  overhead constants. Wall time = max(latency-bound `requests×2.5s/concurrency`,
  RPM bound).

### `progressive.py`
- **Purpose:** §3.2 progressive evaluation — stage-1 selection and stage-2 sizing.
- **Public API:** `select_sample_ids(cfg, report, sample_size, exclude) ->
  list[str]`, `choose_stratify_field`, `stage2_recommendation(scores, ci_low,
  ci_high, question_value) -> dict|None`, `required_n_for_ci_halfwidth`.
- **Internals:** one streaming pass collects `(id, stratum)` only. Stratum key:
  explicit `stratify_by` > categorical reference (`__label__`) > lowest-cardinality
  categorical metadata > input-length buckets. Allocation: proportional with
  floor 1 per stratum + largest-remainder; draw via `random.Random(seed)` over
  *sorted* ids → byte-identical selection for a given (dataset, seed).
  `exclude` powers stage-2 top-ups (`resume --add N` never re-picks stage-1).
  `stage2_recommendation` returns None when the CI is decisively on one side of
  the gate threshold; otherwise `n = ⌈(1.96σ/target)²⌉` with target = distance
  from mean to threshold (floor 0.01).

### `metrics/` package
- **Registry (`metrics/__init__.py`):** `_REGISTRY: name -> (module, class)`;
  `available_metrics()` **never imports metric modules** (config validation
  must stay dependency-free); `create_metric(MetricConfig, judge_pool)` imports
  lazily and injects `judge_pool` when `cls.requires_judge`.
- **Interface:** `Evaluator` — class attrs `name`, `requires_reference/
  context/judge`; sync `evaluate(sample, output)`; async `aevaluate` (default
  wraps sync; **the runner always awaits `aevaluate`**). `ModelOutput(text,
  samples, error, latency_s, tokens_*, cost_usd, cached)`;
  `EvalResult(metric, score: float|None, passed: bool|None, details, error)` —
  `score=None` + `error` means the metric itself could not judge (kept separate
  from a score of 0.0). `_result()` helper applies `pass_threshold`.
  Evaluators are **stateless** (10.6: same objects must later score live traces).
- **`deterministic.py`:** two tokenizers, deliberately distinct —
  `_tokens` (SQuAD normalization: lowercase, no punctuation, **no articles**)
  for `exact/normalized/contains/token_f1`; `_tokens_simple` (articles kept)
  for BLEU/ROUGE, because those reference formulas do not drop stopwords.
  `sentence_bleu`: up-to-4-gram, ε-smoothing, brevity penalty, n capped at
  candidate length. `rouge_n`/`rouge_l` (O(n·m) LCS with 1-D DP rows).
  `extract_json`: raw → fenced → first balanced `{…}`/`[…]` (shared by judge
  and structured metrics). `_check_schema`: `jsonschema` if installed, else a
  built-in subset checker (type/required/properties/items/enum).
  `Classification` stores `{predicted, reference}` in details — the aggregate
  P/R/F1/confusion work happens in `analysis.py`, not per sample.
- **`judge.py`:** see §4 sequence below. `JudgePool` owns judge providers
  (main + ensemble), shares the run's cache/cost/audit; `ask(prompt,
  required_keys, sample_hash, cache_tag, mcfg)` = cache → call → `extract_json`
  → key check → on failure re-prompt with the parse error appended, up to
  `judge.max_retries`; each attempt has its own cache slot. `JudgeRubric`
  (decomposed = one binary yes/no call per criterion — 10.3 — else one scaled
  call, scores normalized `(raw-1)/(scale-1)`, weighted mean; ensemble =
  majority vote on pass, mean on score), `JudgeReference` (1–5 vs golden
  answer), `JudgePairwise` (registry placeholder — real work is
  `run_pairwise_stage`, a post-stage over `results.jsonl` since it needs both
  candidates' outputs: both orderings per pair, verdicts mapped back to
  order-1 space; inconsistent ⇒ `winner="tie", flipped=True`;
  `summarize_pairwise` folds `pairwise.jsonl` into win/flip rates).
- **`rag.py`:** claim-extraction prompt shared by faithfulness (supported/total)
  and hallucination (unsupported/total, `passed = rate ≤ threshold`, i.e. an
  inverted metric); relevance (1–5 normalized); context precision (per-chunk
  boolean list) and recall (golden-answer claims vs context).
  `params.engine: ragas` delegates when installed, actionable error otherwise.
- **`code_exec.py`:** `extract_code` (fenced-block aware); `SyntaxValid` via
  `compile()`; `CodePassAtK` — harness from reference or `tests` metadata,
  candidate code + harness + `print('__SWARN_PASS__')` executed per generated
  sample through `agent.runtime.execution.make_backend()` (one shared backend;
  plain-subprocess fallback when the agent package is absent); unbiased
  estimator `pass_at_k(n, c, k) = 1 − C(n−c,k)/C(n,k)`; demands
  `n_samples_per_input ≥ k` with an actionable error.
- **`safety.py`:** `Refusal` (regex on the first 600 chars; `params.desired:
  refuse` flips polarity for adversarial sets), `Toxicity` (judge, 0–1,
  lower-better), `PiiLeakage` (Presidio if installed, else email/phone/SSN/card
  regexes; engine recorded in details).
- **`trajectory.py`:** `ToolCorrectness` (0.7·expected-tool recall +
  0.3·argument validity; args-only without expectations), `StepEfficiency`
  (optimal/actual capped at 1; metadata or budget fallback), `TaskCompletion`
  (judge over a ≤30-step rendered trajectory; 0.5·completed + 0.5·quality).

### `runner.py` — the execution engine
- **Public API:** `run_eval(cfg, yes, no_cache, max_cost_override,
  resume_run_id, sample_ids_override, progress) -> Path` (the run dir);
  `RunAborted`; `Checkpoint`; `CostTracker`; `AuditLog`; `load_run_config(run_id)`
  (always the **snapshot**, never the possibly-edited original);
  `build_messages`, `prepare_samples`.
- **Control flow of `run_eval`:**
  1. `validate_dataset` — `report.errors` ⇒ `RunAborted`.
  2. New run: create dir, write `config.snapshot.yaml`, `environment.json`,
     `dataset_meta.json` (content hash + validation dict); estimate the full
     set; decide sampling (`sample` mode, or `auto` when est > `budget_usd`
     and rows > sample_size → `select_sample_ids`); re-estimate the actual
     rows; HITL gate if est > budget (`_confirm_or_abort`: `--yes` →
     auto-approve+audit; TTY → `typer.confirm`; neither → `RunAborted` telling
     the user to pass `--yes`); write `Checkpoint(selected_ids, seed)`.
     Resume: load the existing checkpoint, log `resumed`.
  3. Build `ResponseCache`, `CostTracker` (seeded with `cost_so_far`),
     optional `JudgePool` (+ `_self_judge_guard`: judge family ==
     any candidate family ⇒ `RunAborted` unless `allow_self_judge`),
     evaluators via `create_metric` (**`judge_pairwise` filtered out** —
     it runs as the post-stage).
  4. `asyncio.run(_run_all())`: candidates sequentially; per candidate
     `_execute_candidate` skips ids already in `checkpoint.completed[name]`;
     then the pairwise stage if requested, ≥2 candidates, and not over cap.
  5. `KeyboardInterrupt` → checkpoint + audit + re-raise (CLI exits 130).
     Cost cap hit → audit `halted_at_cost_cap`, progress names the resume
     command. Finally `report.finalize_run`.
- **`_execute_candidate` (per candidate):** one provider, `asyncio.Semaphore
  (concurrency)`, one `asyncio.Lock` guarding *file append + checkpoint
  mutation* as a unit; per sample: `_generate_output` (loop over
  `n_samples_per_input`: cache key with `sample_index=i`, provider call with
  `meta={"reference": …}`, `cost.record` per generation) → every evaluator's
  `aevaluate` in `try/except` (a crashing metric becomes an
  `EvalResult(error="metric crashed: …")`, never a dead run) → append the
  result row → `checkpoint.completed[name].append(id)`; checkpoint saved every
  `checkpoint_interval` completions and at candidate end. Cooperative halt:
  `cost.over_cap` is checked before and after semaphore acquisition — nothing
  new starts, in-flight samples finish and are recorded.
- **`prepare_samples`:** materializes only the selected rows (progressive
  sampling is what keeps huge sets out of memory — full-run memory is
  O(selection), documented as such); applies `truncation_policy`
  (`truncate` = char-level cut at `max_input_tokens×4`, `skip` = drop;
  `error` was already fatal in validation).
- **`build_messages`:** multi-turn samples send their turns verbatim;
  otherwise `prompt_template.format_map(_SafeDict)` over
  `{input, context, **metadata}` — unknown placeholders survive literally
  instead of raising.
- **`CostTracker.record`:** appends a ledger line `{ts, role:
  candidate|judge, model, sample_id, tokens, cost_usd, cached, error,
  running total}`; sets `halted` when `total ≥ max_cost_usd`.
- **`AuditLog`:** append-only JSONL of `run_created / hitl_* / progressive_stage1 /
  self_judge_allowed / judge_malformed_json / judge_call_failed / interrupted /
  halted_at_cost_cap / execution_finished / resumed / self_heal`.

### `stats.py`
- **Public API:** `bootstrap_ci(scores, seed, resamples=10_000) -> CI(mean,
  low, high, n)`, `paired_bootstrap_diff`, `paired_tests(a, b, seed) ->
  PairedTestResult(test, p_value, effect_size, effect_label, diff_ci, n,
  significant)`, `cohens_kappa`, `spearman`, `pearson`.
- **Internals:** percentile bootstrap on the mean, vectorized
  (`rng.integers((resamples, n))` index matrix — one allocation, no Python
  loop). `paired_tests` picks the test from the data: both arrays binary ⇒
  exact McNemar (binomial on discordant pairs); else Wilcoxon signed-rank
  (`zero_method="pratt"`, all-ties → p=1). `significant` requires **both**
  p < 0.05 and the paired-bootstrap diff CI excluding 0. Effect label =
  Cohen's d against the pooled std. Kappa handles the single-class case
  (pe=1 ⇒ 1.0) so degenerate calibrations don't NaN.

### `analysis.py`
- **Public API:** `load_results(path) -> {candidate: [row]}`,
  `aggregate_candidate(rows, seed)`, `slice_metrics(rows, fields, seed,
  min_slice=5)`, `verbosity_bias`, `format_bias`, `flip_rates`,
  `classification_report`, `failure_analysis(rows, worst_k)`,
  `worst_examples(rows, metric, k)`.
- **Internals:** aggregates = per-metric CI + `pass_rate` + metric-error count,
  plus candidate-level `pass_rate` (all pass-bearing metrics passed),
  `failure_rate`, latency/token/cost/cache totals. Slices skip fields with
  < 2 groups or groups under `min_slice`. Judge diagnostics: verbosity =
  Pearson(len(output), judge score) per judge metric (≥ 5 points); format
  bias = mean judge score per detected output format (table/json/code/prose,
  ≥ 5 each). `classification_report` rebuilds y_true/y_pred from per-sample
  details → sklearn accuracy, macro/micro/weighted P/R/F1, confusion matrix,
  ROC-AUC when binary. `failure_analysis`: generation errors bucketed by
  family (timeouts/rate limits/context/permanent/retries); wrong answers
  (any score < 0.5) clustered — MiniLM embeddings + KMeans when available,
  category/leading-token lexical fallback otherwise — each cluster labeled by
  its most distinctive tokens, top clusters summarized in plain sentences.

### `gates.py`
- **Public API:** `evaluate_gates(gates, summary_candidates) -> [GateResult]`,
  `gates_exit_code`.
- **Internals:** address grammar `[candidate/]metric[.aggregate]`, aggregate
  defaulting to `.mean`; bare `pass_rate`/`failure_rate` hit candidate-level
  aggregates. Unqualified metrics with multiple candidates are rejected as
  ambiguous (the error lists valid prefixes). `n < min_samples` ⇒ "not
  evaluable", which **fails** the gate (§3.3: no verdicts from tiny samples).
  Exit code 0 iff every gate passed.

### `report.py`
- **Public API:** `finalize_run(rdir, cfg, validation, progress) -> summary
  dict`, `build_summary`, `regenerate(rdir)` (config from snapshot; no model
  calls), `render_markdown`, `render_html`, `TINY_DATASET_N = 30`.
- **summary.json composition:** run/config/dataset identity + seed + sampled
  flag; per-candidate aggregates (+slices from `reporting.slice_by` ∪
  validation's categorical metadata, +classification); pairwise-tested
  `comparisons` with a `verdict` string built by `_verdict` (< 30 shared
  samples ⇒ "directional only"; not significant ⇒ says so with the diff CI —
  **never a bare winner**); `judge_diagnostics` (verbosity/format/flip rates,
  `summarize_pairwise` output, calibration record via
  `calibrate.load_calibration` incl. staleness); `gates` + `gates_passed`;
  `stage2_recommendation` (first gate whose threshold sits inside a CI);
  `caveats` (tiny dataset / synthetic / contamination / failure-rate > 5 %);
  per-candidate `failure_analysis`; `environment`.
- **Rendering:** one markdown renderer (headline tables with unicode CI bars,
  comparison tables, slice tables, judge diagnostics with the kappa < 0.6 red
  flag, gates, failure analysis, `<details>`-collapsed worst-K with judge
  reasoning, and a fixed "how to read this report" footer); HTML = built-in
  minimal md→html converter (headings/tables/lists/bold/code/details, inline
  CSS) — no external renderer dependency.
- **Evidence bundle (10.5):** `evidence/` = copies of snapshot, environment,
  dataset meta, summary, audit, ledger (+ judge calibration) plus a
  `MANIFEST.json` describing each file, gate outcome, caveats, model versions.
- **MLflow (`tracking.mlflow`):** lazy import; params (hashes, seed), metric
  means, pass/failure rates, full artifact dir. Missing package = one skipped
  line, never a failure.

### `compare.py`
- **Public API:** `compare_runs(baseline_dir, candidate_dir, max_drop,
  p_value, seed) -> ComparisonReport(pairs, regressions, warnings, passed)`,
  `render_comparison`.
- **Internals:** joins on candidate name across runs, then on `sample_id`
  within (falls back to first-vs-first with a warning when no names match;
  warns when < 30 shared or zero shared ids). Regression rule per metric:
  `effect_size < −max_drop ∧ p < p_value` ⇒ `passed=False` (CLI exit 1).
  Both runs' cached generations make re-scoring free.

### `calibrate.py`
- **Public API:** `calibrate_run(run_dir, judge_cfg, human_labels_path) ->
  record`, `load_calibration(judge_cfg)`, `save_calibration`,
  `calibration_key(judge_cfg)`, thresholds `KAPPA_UNRELIABLE=0.6`,
  `KAPPA_PRODUCTION=0.8`.
- **Internals:** store keyed by sha256 of (judge model+provider, mode,
  decompose, full rubric dump)[:16] under `_calibration/` — editing the rubric
  invalidates old agreement numbers by construction. Joins human labels
  (`{sample_id, human_pass|human_score}`) with the run's judge metrics: kappa
  on pass pairs (tercile-discretized kappa when only scalar), Spearman on
  score pairs, verdict string. **Judge drift (10.3):** the record stores the
  exact judge model id; `load_calibration` marks the record `stale` (with a
  re-calibrate instruction) when the config's pinned judge differs. Empty
  joins raise `ValueError` explaining the id-mismatch possibilities.

### `autonomous.py` (§6)
- **Public API:** `design_eval(goal, dataset_path, candidates, train_path,
  progress) -> (EvalConfig, rationale)`, `autonomous_eval(...) -> run dir`,
  `MALFORMED_JUDGE_THRESHOLD = 0.10`.
- **Internals:**
  - Design: validation report → `_llm_design` (SWARN's deployed model via lazy
    `agent.llm.create_client`, temp 0, JSON-only prompt listing the metric
    registry) with `_heuristic_design` fallback (categorical refs ⇒
    classification; short refs ⇒ NEM+F1; no refs ⇒ judge rubric; context ⇒
    +rag_faithfulness; `NN%` in the goal ⇒ pass_rate gate). An LLM design
    that fails config validation (unknown metrics etc.) falls back to the
    heuristic rather than erroring.
  - Loop: write proposed `eval.yaml` next to the dataset → design-approval
    HITL → `run_eval` → up to 2 self-heal iterations via `_diagnose`
    (audit-driven: > 10 % malformed judge JSON ⇒ force `decompose=True`, then
    raise `max_retries`; > 5 % context-overflow errors ⇒
    `truncation_policy: truncate`) — each patch audited (`self_heal`) and
    re-run **cheaply through the cache** → if `stage2_recommendation` exists,
    offer the top-up (auto-approved under `--yes` only up to 1000 rows) →
    `_grow_selection` + resume.

### `harness.py` (§2.5) & `synth.py` (10.4)
- **harness:** `run_harness(tasks, ModelConfig, limit, num_fewshot)` shells
  out to `lm_eval` (`--model local-chat-completions` against the candidate's
  OpenAI-compatible endpoint; deployed endpoint resolved lazily), captures
  stdout to the run dir, finds `results*.json`, and `ingest_harness_results`
  reshapes it into the same summary schema (per-metric mean ±1.96·stderr,
  provenance and caveat noted). Missing tool ⇒ `RuntimeError(INSTALL_HINT)`.
- **synth:** `generate_synthetic_dataset` chunks source docs (6 kB), asks the
  generator model for Q/A pairs (JSON-only), writes JSONL rows carrying
  `{synthetic: true, synthetic_source: file#chunk, synthetic_generator,
  reviewed: false}`; `review_sample` (seeded sample for the human) +
  `mark_reviewed` (approve/drop, rewrites the file). Gating configs must set
  `dataset.synthetic: true`, which pins the synthetic caveat to every report.

### `cli.py`
- **Public API:** `eval_app` (Typer sub-app) with `validate` (+
  `--write-example`), `estimate`, `run`, `resume` (+ `--add`), `report`,
  `compare`, `calibrate`, `auto`, `harness`, `synth`, `synth-review`,
  `metrics`.
- **Internals:** heavy modules imported inside commands (`swarn --help` stays
  fast). `_print_run_outcome` reads `summary.json` and prints caveats →
  headline metrics → comparison verdicts → stage-2 suggestion → gates,
  returning the gate exit code. Exit codes: 0 gates pass, 1 gate failure or
  regression, 2 config/dataset/abort errors, 130 Ctrl-C.

---

## 3. Data contracts

### Run directory (`workspace/eval_runs/<run_id>/`)

| file | writer | mode | contents |
|---|---|---|---|
| `config.snapshot.yaml` | runner (create) | once | full `EvalConfig.model_dump()` — resume/report/compare read only this |
| `environment.json` | runner (create) | once | python/platform/package versions, seed, candidate + judge model ids, timestamp |
| `dataset_meta.json` | runner (create) | once | dataset content hash + full validation report |
| `checkpoint.json` | runner | atomic replace | `{selected_ids, completed: {candidate: [ids]}, cost_so_far, seed, stage}` |
| `results.jsonl` | runner | append-only | one row per (candidate, sample) — schema below |
| `pairwise.jsonl` | judge post-stage | append-only | `{a, b, sample_id, winner: a\|b\|tie\|null, flipped, detail?, error?}` |
| `cost_ledger.jsonl` | CostTracker | append-only | every model call: tokens, cost, cached, role, running total |
| `audit.jsonl` | AuditLog | append-only | HITL decisions, judge failures, halts, self-heals |
| `summary.json` | report | replace | machine-readable aggregates (schema in §2 report.py) |
| `report.md` / `report.html` | report | replace | human report |
| `evidence/` | report | replace | compliance bundle + `MANIFEST.json` (10.5) |

`results.jsonl` row:

```json
{"candidate": "...", "sample_id": "...", "input": "...", "output": "...",
 "output_samples": null, "reference": "...",
 "scores":  {"metric": 0.0},           // null = metric errored
 "passed":  {"metric": true},           // only pass-bearing metrics
 "metric_details": {"metric": {"reasoning": "...", "...": "..."}},
 "metric_errors":  {"metric": "..."},
 "error": null,                          // generation-level failure
 "latency_s": 0.0, "tokens_in": 0, "tokens_out": 0,
 "cost_usd": 0.0, "cached": false, "metadata": {"category": "..."}}
```

### Crash-safety / resume invariants
- Every artifact is append-only or atomically replaced ⇒ kill -9 at any point
  leaves a resumable run.
- A sample id enters `checkpoint.completed[candidate]` in the same locked
  section that appends its result row ⇒ resume never duplicates and never
  re-bills (verified by `test_checkpoint_resume_idempotent`).
- `selected_ids` lives in the checkpoint ⇒ stage-1/stage-2 membership is
  stable across resumes regardless of RNG.
- Reproducibility identity = `config_hash + dataset content hash + seed`
  (all three in `summary.json`; sampling, bootstrap, and provider `seed`
  params all derive from `sampling.seed`).

---

## 4. Sequence diagrams

### `swarn eval run` (fresh, auto sampling, judge configured)

```
CLI.run ─▶ load_eval_config ──ConfigError?──▶ exit 2
        ─▶ runner.run_eval
             ├─ validate_dataset ──errors?──▶ RunAborted ▶ exit 2
             ├─ mkdir run_dir; snapshot config/env/dataset_meta
             ├─ estimate_run(full) ─ over budget & big? ─▶ select_sample_ids (stage 1)
             ├─ estimate_run(selection) ─ over budget? ─▶ _confirm_or_abort (HITL)
             ├─ Checkpoint(selected_ids).save
             ├─ JudgePool + _self_judge_guard ; evaluators (pairwise filtered)
             ├─ asyncio.run:
             │    for candidate:                       (sequential)
             │      gather(sample tasks)               (Semaphore-bounded)
             │        sample ─▶ cache.get ∥ provider.call ─▶ cost.record
             │               ─▶ [evaluator.aevaluate]* ─▶ EvalResults
             │               ─▶ lock: append results.jsonl + checkpoint tick
             │    judge_pairwise? ─▶ run_pairwise_stage ─▶ pairwise.jsonl
             ├─ finalize_run ─▶ summary.json, report.md/html, evidence/, mlflow?
             └─ CLI._print_run_outcome ─▶ exit = gates_exit_code
```

### One judged rubric check (decomposed)

```
JudgeRubric.aevaluate(sample, output)
  for judge in [main, *ensemble]:
    for criterion in rubric:
      JudgePool.ask(binary_check_prompt, ("reasoning","verdict"), tag)
        attempt 0..max_retries:
          key = sha256(cache_key(judge_model, msgs, attempt) + tag)
          cache.get(key) ∥ provider.call ─▶ cost.record(role="judge")
          extract_json ∧ keys present? ─▶ return dict
          else audit("judge_malformed_json"); re-prompt with parse error
    score_j = Σ weight·[verdict==yes] / Σ weight
  score = mean(score_j); passed = majority(score_j ≥ threshold)
```

### `resume --add N` (stage 2) and cost-cap halt

```
resume: find_run ─▶ load_run_config(snapshot)
        --add? ─▶ validate ─▶ select_sample_ids(N, exclude=stage1) ─▶ checkpoint.stage+=1
        run_eval(resume_run_id=...) ─▶ skips checkpoint.completed ids ─▶ finalize_run

cap:    cost.record ⇒ total ≥ max_cost ⇒ halted=True
        pending tasks see over_cap before starting ⇒ no new calls
        audit("halted_at_cost_cap"); checkpoint intact ⇒ `resume --max-cost` continues
```

---

## 5. Concurrency model

- One event loop per `run_eval` (`asyncio.run`); candidates run sequentially
  (isolates rate limits and keeps the cost ledger attributable), samples
  concurrently under `Semaphore(execution.concurrency)`.
- Shared mutable state and its guard:
  - results file + checkpoint: one `asyncio.Lock` per candidate execution.
  - `TokenBucket`: internal `asyncio.Lock`; RPM bucket pays 1/request, TPM
    bucket pays estimated tokens; waiters sleep ≤ 5 s slices.
  - `CostTracker`/`AuditLog`: append via synchronous `open(..., "a")` —
    atomic-enough single writes within one process; called only from the loop
    thread.
- Blocking work is kept off the loop: HF local inference in
  `asyncio.to_thread`; deterministic metrics are microsecond-level sync calls
  inside `aevaluate` (accepted); sandbox pass@k is sync inside `evaluate`
  (bounded by its own timeout).
- No cross-process coordination: a run dir belongs to one process at a time
  (nothing enforces this — see §8).

---

## 6. Error taxonomy & exit codes

| layer | failure | behaviour |
|---|---|---|
| config | bad YAML / schema / unknown metric | `ConfigError`, actionable per-field lines → exit 2 |
| validation | empty set, malformed > threshold, oversized+`error` policy | `report.errors` → `RunAborted` → exit 2 |
| HITL | over budget, no TTY, no `--yes` | `RunAborted` naming the flag → exit 2 |
| provider | retryable (429/5xx/timeout) | backoff+jitter, then `Generation(error)` — sample failed, run continues |
| provider | permanent (auth/context/quota) | immediate `Generation(error="permanent: …")` |
| metric | evaluator raises | caught per metric → `EvalResult(error="metric crashed…")` |
| judge | malformed JSON | per-attempt retry with parse error; exhausted → `score=None`; rate tracked for self-heal |
| optional deps | ragas/bert-score/nltk/presidio/mlflow/lm-eval missing | actionable install-hint error string (metric) or skip line (mlflow) — never a crash |
| gates | failed / not-evaluable | exit 1 |
| Ctrl-C | anywhere in execution | checkpoint + audit → exit 130 |

Design rule inherited from the repo: nothing in the execution path raises for
a single sample; a run only aborts before spending (validation/HITL) or by
explicit cap/interrupt.

---

## 7. Security & HITL surface

- Spend gates: estimate-vs-`budget_usd` confirmation (interactive) /
  `--yes` (CI, audited); `max_cost_usd` hard cap with graceful halt;
  `evaluate_llm` is in `SIDE_EFFECTING_TOOLS` so the agent loop's approval
  prompt covers tool-initiated runs (the tool then passes `yes=True` — the
  human approved the call itself).
- Judge integrity: self-judge family guard (explicit opt-in flag), forced
  temperature 0, dual-order pairwise, calibration verdicts surfaced in-report.
- API keys: read from env vars named in config (`api_key_env`), never stored
  in run artifacts; `config.snapshot.yaml` contains the env var *name* only.
- Synthetic data cannot silently gate: provenance on every row + human-review
  command + permanent report caveat.

---

## 8. Known limitations (accepted, documented)

- Full-dataset (non-sampled) runs materialize the selected sample list in
  memory (`prepare_samples`); progressive sampling is the intended path for
  huge sets — the streaming guarantee applies to validation/selection passes.
- No cross-process run-dir locking; concurrent `resume` of the same run is
  undefined.
- Near-dup recall between hamming 8–12 is probabilistic (~80 %+) by design.
- `safety.injection_set` is a documented config field; adversarial sets run
  as their own dataset rather than being auto-merged into a run.
- 10.3 "cross-family audit" is realized via `judge.ensemble` (per-judge scores
  recorded in details), not as a scheduled background process.
- Online scoring (10.6) is design-only: evaluators are stateless and consume
  the trace-compatible `Sample`; no live pipeline exists.

---

## 9. Extension points

| to add | touch |
|---|---|
| a metric | new `Evaluator` subclass + one `_REGISTRY` line (config validation, `create_metric`, estimator naming rules pick it up automatically; prefix `judge`/`rag_` if it needs the JudgePool) |
| a provider | `Provider` subclass implementing `_generate` + a branch in `create_provider` + optional `DEFAULT_PRICING` rows |
| a dataset format | `_iter_<fmt>` generator + `detect_format`/`iter_rows` branches + the `DatasetConfig.format` literal |
| a gate aggregate | expose it in `aggregate_candidate` — the gate resolver addresses any key in the metrics entry |
| a self-heal rule | new branch in `autonomous._diagnose` returning `(description, patched_cfg)` |
| a report section | extend `build_summary` + `render_markdown` (HTML renders from the markdown) |

---

## 10. Test map (`python tests/run_tests.py`)

| module | covers |
|---|---|
| `test_llm_eval_metrics.py` (27) | known-answer metric math (F1/BLEU/ROUGE/JSON/pass@k), normalization semantics, judge retry-on-malformed + exhaustion, pairwise flip detection, bootstrap/McNemar/Wilcoxon/kappa/Spearman, gate addressing/min-samples, registry completeness |
| `test_llm_eval_data.py` (14) | every loader, field mapping errors, multi-turn + OTLP trace ingestion, validation edge cases (malformed/empty/dupes/oversized/threshold), SimHash + banded index, contamination, deterministic proportional stratified sampling, stage-2 math |
| `test_llm_eval_engine.py` (15) | mocked end-to-end run + full artifact set, cache hits on re-run, `--no-cache`, checkpoint/resume idempotency (incl. rewound results file), cost-estimate math, headless HITL block, cost-cap halt + resume, partial-failure semantics + caveat, self-judge guard, auto progressive trigger, tiny-dataset caveat, compare regression gate, calibrate → report surfacing → drift, pairwise stage, gates in summary |
| `test_llm_eval_scale.py` (1) | 1 M-row JSONL validation: streams in ~37 s, RSS growth < max(400 MB, 2×file) |

All engine tests use the mock provider — zero network calls in CI (§8
integration-test requirement).
