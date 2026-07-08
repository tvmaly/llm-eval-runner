# goeval — Design Plan

A private, minimal-dependency Go library for evaluating LLM capabilities across models, usable as a CLI and as a backend HTTP API, with evals authored as Claude Code–style skill markdown files and outputs designed to be verified by an AI agent in a feedback loop.

Working name: **goeval** (module path is Open Item OI-2). No code in this document — contracts are given in signature notation for Claude Code (Opus 4.8) to implement.

Labeling convention (per author's standard): statements marked **[Inference]** are design judgments or predictions, not externally verified facts. Everything else is either a locked decision or grounded in the sources listed at the end.

---

## 1. Goals and Non-Goals

**Goals**

- Compare LLM capabilities across models and transports with reproducible, deterministic verdicts.
- Library-first: CLI and HTTP API are thin adapters over the same library surface.
- Evals authored as markdown files that structurally resemble Claude Code SKILL.md files (YAML frontmatter + markdown body, optional resource directory).
- Every output machine-verifiable by an AI agent: stable versioned JSON schemas, semantic exit codes, byte-reproducible re-grading.
- Small, composable interfaces; every component replaceable and testable in isolation.

**Non-Goals (v1)**

- No LLM-as-judge grading (Locked Decision LD-4).
- No web UI, no dashboards — reporting is JSON/JSONL/text.
- No multi-turn agentic eval harness (tool-use loops). The Transport contract leaves room for it later. **[Inference]** Multi-turn support can be added as a new `Session` interface without breaking v1 contracts because Transport is stateless per call.
- No distributed execution; single process, concurrent goroutines.

---

## 2. Locked Decisions

These were confirmed with the author and are not open for reinterpretation during implementation.

| ID | Decision |
|----|----------|
| LD-1 | Language: Go. Minimal dependencies — stdlib-first; each third-party dependency requires explicit justification in this plan (see §16 and OI-1). |
| LD-2 | Delivery: library-first. One binary `goeval` with subcommands; `goeval serve` runs the HTTP API. CLI and API call the same library functions — no logic lives in `cmd/`. |
| LD-3 | Transports: pluggable `Transport` interface. v1 ships four implementations: `openrouter`, `anthropic` (direct API), `gateway` (internal AI gateway / Bedrock-style), and `replay` (record/playback for tests and offline verification). |
| LD-4 | Grading: deterministic Go graders only. A verdict must be a pure function of (eval spec, stored model output). No network, no clock, no randomness inside graders. |
| LD-5 | Persistence: pluggable `Store` interface; default implementation is append-only JSONL on the filesystem. |
| LD-6 | Eval authoring: single `EVAL.md` per eval (YAML frontmatter + markdown body). An optional directory pack adds `fixtures/` and `expected/`. Single-file must remain fully self-sufficient. |
| LD-7 | Observability: stdlib `log/slog` for logging now. Tracing/metrics are defined as small locally-owned interfaces whose shapes mirror OpenTelemetry semantics (span start/end, status, attributes, W3C-style trace/span IDs), so an OTel adapter can be added later without changing any library contract. The OTel SDK is **not** a v1 dependency. |
| LD-8 | Agent feedback loop is a first-class requirement: JSON-default machine output, `schema_version` on every emitted record, `goeval verify` re-grades stored outputs and must reproduce verdicts exactly, semantic exit codes (§13.3). |
| LD-9 | Prompt templating: stdlib `text/template` with `missingkey=error` so unresolved variables fail validation, not silently render empty. |
| LD-10 | Config precedence: flags > environment variables > config file > defaults. Config file is optional; the tool must run with zero config given a valid eval path and transport credentials. |
| LD-11 | All authoritative on-disk data (results, run manifests) is append-only; mutation happens only by appending superseding records, mirroring the author's established append-only convention. |

---

## 3. Architecture Overview

Pipeline stages, each behind its own interface:

```
author (EVAL.md / pack)
  → Loader (parse + validate → EvalSpec)
  → Planner (specs × models × attempts → RunPlan)
  → Runner (executes plan via Transport, concurrency/retry/timeout)
  → Grader (deterministic verdicts from raw outputs)
  → Store (append-only persistence of outputs + verdicts)
  → Reporter (summaries: JSON, JSONL, text)
```

Cross-cutting: `obs` (logging/tracing/metrics shapes) and `runid` (ID generation). The CLI and HTTP API drive the pipeline through one façade type (§5.9) so both surfaces are guaranteed to behave identically.

Key property for the agent feedback loop: **execution and grading are separable.** The Runner stores raw model outputs before grading; grading can be re-run at any time from the store alone (LD-4 + LD-8). `run = fetch + grade`, `verify = grade only`.

---

## 4. Package Layout

| Package | Responsibility | Depends on |
|---------|----------------|------------|
| `eval` | `EvalSpec`, `Case`, frontmatter/body parsing, validation, pack resolution | `obs` |
| `grader` | `Grader` interface, registry, built-in graders, combinators | `eval` |
| `transport` | `Transport` interface, `Request`/`Response` types, error taxonomy | `obs` |
| `transport/openrouter`, `transport/anthropic`, `transport/gateway` | HTTP implementations | `transport`, `obs` |
| `transport/replay` | Record/playback cassette transport | `transport` |
| `run` | `Planner`, `Runner`, retry/timeout/concurrency, resume | `eval`, `grader`, `transport`, `store`, `obs` |
| `store` | `Store` interface, record types, `schema_version` constants | — |
| `store/jsonl` | Append-only filesystem implementation | `store` |
| `report` | Aggregation and rendering (json/jsonl/text) | `store` |
| `obs` | `Tracer`/`Span`/`Meter` shapes, slog setup, noop implementations, context propagation helpers | — |
| `api` | HTTP handlers (stdlib `net/http`), request/response DTOs | `run`, `store`, `report` |
| `goeval` (root) | Façade: `Engine` type wiring the pipeline | all above |
| `cmd/goeval` | CLI: flag parsing and output formatting only | root |

Dependency rule: arrows only point downward in this table; `obs` and `store` depend on nothing internal. **[Inference]** This keeps every package unit-testable with fakes and lets Claude Code build in the milestone order of §17 without forward references.

---

## 5. Core Contracts

Notation: `Method(args) → (results)`. All context-accepting methods take `ctx` first and must honor cancellation. All interfaces below are intentionally small (1–3 methods) and composed rather than widened.

### 5.1 Transport

```
Transport
  Complete(ctx, Request) → (Response, error)
  Name() → string
```

- `Request`: `{model, system, messages[], max_tokens, temperature, top_p?, stop?, seed?, metadata{}}`. Messages are `{role, content}` text-only in v1 (OI-3 covers streaming/multimodal).
- `Response`: `{text, model, finish_reason, usage{input_tokens, output_tokens}, raw json.RawMessage, latency, transport_name, request_hash}`.
- `raw` preserves the untouched provider response body — required so re-grading and audits never depend on our parsing being lossless.
- `request_hash` = SHA-256 over a canonical serialization of `Request`; used by replay matching and resume dedupe.
- Error taxonomy: implementations must wrap failures in typed errors — `ErrRateLimited{RetryAfter}`, `ErrTransient`, `ErrAuth`, `ErrBadRequest`, `ErrModelNotFound` — so the Runner's retry policy is transport-agnostic.
- Implementations must not retry internally; retry is the Runner's job (single place, observable).

`transport/replay`: wraps any Transport in record mode (writes cassette JSONL keyed by `request_hash`); in playback mode serves responses from cassette and returns `ErrCassetteMiss` on unknown hashes. This is the backbone of deterministic integration tests and offline agent verification.

### 5.2 Loader

```
Loader
  Load(path) → (EvalSpec, error)          // file or pack directory
  LoadDir(path) → ([]EvalSpec, []LoadError)  // recursive discovery
```

```
Validator
  Validate(EvalSpec) → []Finding   // Finding: {severity, code, path, message}
```

- Severities: `error` (blocks run), `warning` (reported, non-blocking). Machine-stable `code` strings (e.g. `EVAL001 missing description`) so an agent can assert on them.
- `Load` never partially succeeds: parse errors return a typed `ParseError{line, col, reason}`.

### 5.3 Grader

```
Grader
  Grade(spec EvalSpec, c Case, out Output) → (Verdict, error)
  ID() → string        // e.g. "regex@1"
```

- `ID()` embeds a version (`name@major`). Any behavior change bumps the version; verdicts record it (drift detection, §12.3).
- Pure: no ctx parameter by design — graders must not do I/O, read clocks, or use randomness (LD-4). `error` is reserved for grader misconfiguration; a failing check is a failing `Verdict`, not an error.
- Registry: `grader.Register(id, factory)`; factories take `params map[string]any` from the eval file and must reject unknown params (strict decoding) so typos in eval files surface at validation time.

`Verdict`: `{pass bool, score float64 ∈ [0,1], checks []Check}` where `Check` = `{name, pass, expected, got, detail}`. Every failure must be explainable from `checks` alone — this is what the AI agent reads to decide its next action in the feedback loop.

### 5.4 Built-in Graders (catalog)

| ID | Params | Semantics |
|----|--------|-----------|
| `exact@1` | `trim, case_insensitive, normalize_ws` | Byte/normalized equality with expected string |
| `contains@1` | `all[] / any[], case_insensitive` | Substring assertions |
| `regex@1` | `patterns[], must_not_match[]` | Go `regexp` (RE2) full-text matching |
| `json_struct@1` | `assertions[]: {path, op, value}` | Parses output as JSON (tolerates one fenced code block wrapper); asserts on dotted paths with ops `eq, ne, gt, lt, exists, absent, type, len` |
| `numeric@1` | `expected, abs_tol / rel_tol, extract` | Extracts first/last number (or via regex group) and compares within tolerance |
| `set@1` | `expected[], ordered, delimiter` | Line/CSV set equality or ordered sequence |
| `golden@1` | `file, trim, normalize_ws` | Equality against a golden file in the pack's `expected/` |
| `length@1` | `min_chars, max_chars, min_lines, max_lines` | Output size bounds |
| `all_of@1`, `any_of@1`, `not@1`, `weighted@1` | `graders[] (+weights)` | Combinators; nest arbitrarily. `weighted` computes score as weighted mean and pass against a `threshold` param |

**[Inference]** These nine primitives plus combinators cover the large majority of capability checks (formatting, extraction, math, classification, instruction-following) without any LLM judge; gaps should be filled by adding graders, not by weakening determinism.

### 5.5 Planner and Runner

```
Planner
  Plan(specs []EvalSpec, opts PlanOptions) → (RunPlan, error)
```

- `PlanOptions`: `{models[], attempts_override?, filter_tags[], seed}`. Model resolution: an eval's frontmatter `models` field may pin targets; CLI `--models` intersects or overrides per a documented rule (override wins, warning emitted).
- `RunPlan` is a fully materialized, serializable list of `Task{task_id, eval_id, case_id, model, attempt, request}`. `task_id` is deterministic: hash of its coordinates. Plans are stored with the run — resume and verify read the same plan the run used.

```
Runner
  Run(ctx, RunPlan, RunOptions) → (RunSummary, error)
```

- `RunOptions`: `{concurrency, retry{max, base_backoff, jitter}, task_timeout, store, transport_for(model) → Transport, hooks}`.
- Semantics:
  - Worker pool of `concurrency` goroutines over tasks; per-transport serialization or rate limits are OI-6.
  - Retry only on `ErrTransient`/`ErrRateLimited` (honoring `RetryAfter`), with exponential backoff + jitter, capped at `retry.max`. `ErrAuth`/`ErrBadRequest` fail fast.
  - Each attempt's raw output is appended to the Store **before** grading (LD-8 separability).
  - Multiple attempts per case with `pass_policy` from frontmatter: `all | any | majority | at_least: N`.
  - Resume: `Run` with an existing `run_id` loads the plan, skips tasks whose `task_id` already has a stored output, executes the rest. Idempotent by construction.
  - Cancellation: on ctx cancel, in-flight tasks finish or abort per `task_timeout`; the run manifest records `status: interrupted` — resumable.
- `hooks`: `{OnTaskStart, OnTaskEnd, OnVerdict}` function fields (nil-safe) — the extension point for progress bars, API server-sent progress, and agent streaming without touching Runner logic.

### 5.6 Store

```
Store
  AppendOutput(ctx, OutputRecord) → error
  AppendVerdict(ctx, VerdictRecord) → error
  PutRunManifest(ctx, RunManifest) → error        // append-only supersede
  GetRun(ctx, run_id) → (RunManifest, error)
  ListRuns(ctx, ListFilter) → ([]RunManifest, error)
  Outputs(ctx, run_id) → iterator over OutputRecord
  Verdicts(ctx, run_id) → iterator over VerdictRecord
```

- Record envelopes all carry: `schema_version, run_id, task_id, recorded_at (RFC3339, UTC)`.
- `OutputRecord` embeds the full `Response` including `raw`; `VerdictRecord` embeds `Verdict` plus `grader_id`, `output_hash` (SHA-256 of the graded text) and `plan_hash`.

`store/jsonl` layout:

```
<data_dir>/runs/<run_id>/
  manifest.jsonl     # append-only; last line is current state
  plan.json          # immutable after creation
  outputs.jsonl
  verdicts.jsonl
```

- Writes: serialize record, write to `*.tmp` is unnecessary for appends — instead: single `O_APPEND` write of one full line + newline per record, `fsync` policy configurable (`always | on_close`). Multi-file atomicity is not needed because each file is independently append-only. `plan.json` uses write-temp-then-rename.
- Reads tolerate a truncated final line (crash during write): the reader skips it and reports a `warning` finding rather than failing the whole run. **[Inference]** This is the standard failure mode for append-only line-oriented files and the skip-last-partial-line recovery is safe because a partial line can only be the final one.

### 5.7 Reporter

```
Reporter
  Summarize(ctx, run_id) → (RunSummary, error)
Renderer
  Render(RunSummary, format) → ([]byte, error)   // format: json | jsonl | text
```

`RunSummary`: totals, per-eval and per-model pass rates, score distributions (min/mean/max), failure list with `checks` details, token usage and latency aggregates, `schema_version`.

### 5.8 Observability shapes (`obs`)

Locally-owned interfaces mirroring OTel semantics — deliberately *not* implementing the OTel interfaces, because the OTel Go API documents that methods may be added to its interfaces without a major version bump, which would break our zero-dep implementations. An adapter (OI: post-v1) maps ours onto theirs.

```
Tracer
  Start(ctx, name) → (ctx, Span)
Span
  End()
  SetAttr(key, value)
  SetError(err)
  SetStatus(code, msg)    // codes: Unset | OK | Error
TraceIDs
  TraceIDFromContext(ctx) → string   // 32 hex chars (16 bytes), W3C-compatible
  SpanIDFromContext(ctx) → string    // 16 hex chars (8 bytes)
```

- Default implementation: `obs.NoopTracer` (zero cost) and `obs.LogTracer` which emits span start/end as slog records — tracing without any backend.
- ID generation: `crypto/rand`, formatted per W3C trace-context sizes so a future OTel adapter propagates them unchanged.
- Logging: `slog.Logger` injected (never global). Canonical fields on every log line inside a run: `run_id, task_id, eval_id, case_id, model, transport, attempt, trace_id, span_id`. JSON handler by default; text handler with `--log-format text`.
- Metrics shape: `Meter.Counter(name).Add(ctx, n, attrs...)` and `Meter.Histogram(name).Record(ctx, v, attrs...)`; noop default. Counted in v1: tasks started/succeeded/failed/retried, tokens in/out, grader pass/fail, transport errors by class.

### 5.9 Engine (façade)

```
Engine  = New(Config) with:
  RunEvals(ctx, paths, RunConfig) → (RunSummary, error)
  Verify(ctx, run_id) → (VerifyReport, error)
  ValidateEvals(ctx, paths) → ([]Finding, error)
  Report(ctx, run_id, format) → ([]byte, error)
```

The only type `cmd/goeval` and `api` are allowed to call. Guarantees CLI/API parity.

---

## 6. EVAL.md File Format

Structurally mirrors a Claude Code skill: YAML frontmatter with `name` and `description` required, markdown body, optional bundled resources — mapped to the eval domain.

### 6.1 Frontmatter

```yaml
---
name: json-extraction-invoice          # slug: [a-z0-9-], unique within a suite
description: >
  Tests whether a model can extract structured invoice fields into strict JSON.
  Covers nested totals, date normalization, and refusal of missing fields.
version: 1                              # integer; bump on any behavioral change
suite: structured-output                # grouping key
tags: [json, extraction, robustness]
models: any                             # or explicit list of model IDs
sampling:
  temperature: 0.0
  max_tokens: 1024
  attempts: 3
  pass_policy: majority                 # all | any | majority | at_least: N
grader:
  id: json_struct@1
  params:
    assertions:
      - {path: total.amount, op: eq, value: 1042.50}
      - {path: line_items,  op: len, value: 3}
timeout: 60s
---
```

Rules: `name` + `description` required (as in SKILL.md); `description` should state what capability is tested and when the eval applies. Unknown frontmatter keys are a validation `error` (strict decoding, consistent with §5.3).

### 6.2 Body contract

Parsed by heading, deterministically. Recognized `##` sections:

- `## System Prompt` — optional; raw text.
- `## Prompt` — required; `text/template` body, variables from each case's `vars` (LD-9).
- `## Cases` — required; one or more fenced blocks with info string `case`, each containing one JSON object: `{"id": "c1", "vars": {...}, "expected": ..., "grader": {...}?}`. Per-case `expected` feeds the grader; per-case `grader` overrides frontmatter grader for that case only.
- `## Notes` — free text, ignored by the parser (author documentation, agent-readable context).

Anything outside recognized sections is a validation `warning`. Fenced `case` blocks make parsing unambiguous without a markdown AST dependency: the parser needs only heading detection and fence scanning. **[Inference]** This keeps the loader stdlib-only and immune to markdown-renderer ambiguities.

### 6.3 Optional pack directory

```
json-extraction-invoice/
├── EVAL.md            # exactly as above
├── fixtures/          # files referenced by cases via {"vars": {"doc": "@fixtures/inv1.txt"}}
└── expected/          # golden files for golden@1
```

`@`-prefixed var values are resolved as pack-relative file reads at load time (never at grade time — LD-4). A single-file eval must not use `@` references (validation error), keeping LD-6's self-sufficiency guarantee.

### 6.4 Suites

A suite is just a directory tree of evals/packs discovered by `LoadDir`. An optional `SUITE.md` at the root (frontmatter: `name`, `description`, `default_models`, `tags`) provides defaults that individual evals may override.

---

## 7. Transports (implementations)

| Impl | Notes |
|------|-------|
| `openrouter` | OpenAI-compatible chat completions endpoint; API key via `GOEVAL_OPENROUTER_KEY`; model IDs passed through verbatim. |
| `anthropic` | Anthropic Messages API; key via `GOEVAL_ANTHROPIC_KEY`; maps `system` to the API's system field. |
| `gateway` | Internal AI gateway (Bedrock-style). Configurable base URL, auth header name/value, request/response mapping documented in a small per-gateway config struct so corporate gateway quirks live in config, not code. **[Inference]** Given the author's isolated-VDI environment, this transport must support custom CA bundles and proxy env vars; both are plumbed via stdlib `http.Transport`. |
| `replay` | §5.1. Cassettes are JSONL: `{request_hash, request, response, recorded_at}`. Redaction hook strips auth headers before write. |

All HTTP transports share one internal helper package (`transport/internal/httpx`): timeout wiring, typed error mapping from status codes (429 → `ErrRateLimited` with `Retry-After` parsing, 5xx → `ErrTransient`, 401/403 → `ErrAuth`, 4xx → `ErrBadRequest`), and slog/span instrumentation — so implementations stay ~small mapping layers.

---

## 8. Runner Semantics (normative details)

- Determinism inputs: `PlanOptions.seed` feeds task shuffling (optional, default off = stable order) and is recorded in the manifest. Temperature 0 is recommended but not enforced.
- Timeout layering: per-task `timeout` (frontmatter) ≤ CLI `--task-timeout` cap; run-level `--deadline` cancels the whole run.
- Backpressure: bounded task queue; the Runner never buffers unbounded results — records stream to the Store as produced.
- Failure isolation: one task's failure never aborts the run (unless `--fail-fast`).

---

## 9. HTTP API (stdlib `net/http`)

`goeval serve --addr 127.0.0.1:8080 --data-dir ...`

| Method + Path | Semantics |
|---------------|-----------|
| `POST /v1/runs` | Body: `{eval_paths[] or suite, models[], run_options}`. Starts a run asynchronously; returns `202 {run_id}`. |
| `GET /v1/runs` | List manifests (filter query params: `status`, `suite`, `since`). |
| `GET /v1/runs/{id}` | Manifest + live progress counters. |
| `GET /v1/runs/{id}/results` | Verdicts (JSONL stream with `Accept: application/x-ndjson`, else JSON array). |
| `GET /v1/runs/{id}/summary` | `RunSummary`. |
| `POST /v1/runs/{id}/verify` | Re-grade; returns `VerifyReport`. |
| `POST /v1/evals/validate` | Body: eval file content or path; returns `Finding[]`. |
| `GET /v1/evals` | Discovered evals under the configured eval dir. |
| `GET /v1/healthz` | Liveness. |

Conventions: JSON everywhere; errors as `{error: {code, message, details}}` with the same stable code namespace as CLI findings; `X-Trace-Id` response header carries the trace ID (accepts inbound `traceparent`-style ID for propagation); auth per OI-5. One in-process run registry maps `run_id → cancel func` so `DELETE /v1/runs/{id}` (cancel) can be added trivially — include it in v1.

---

## 10. CLI Specification

Binary `goeval`, subcommands:

| Command | Purpose |
|---------|---------|
| `goeval run <paths...> --models m1,m2 [--transport auto] [--concurrency N] [--resume RUN_ID] [--fail-fast] [--record cassette.jsonl \| --replay cassette.jsonl]` | Execute evals |
| `goeval validate <paths...>` | Lint eval files; exit 3 on any `error` finding |
| `goeval verify <run_id> [--strict]` | Re-grade stored outputs; `--strict` fails on any verdict mismatch vs stored verdicts |
| `goeval report <run_id> [--format json\|jsonl\|text]` | Render summary |
| `goeval list [runs\|evals]` | Inventory |
| `goeval serve` | HTTP API |
| `goeval schema [verdict\|output\|summary\|manifest]` | Print the JSON Schema for each record type — lets an agent fetch the contract it should validate against |

Output contract (agent-facing, per the author's agent-CLI conventions): default output is JSON when stdout is not a TTY and text when it is; `--output json|jsonl|text` overrides. Logs go to stderr, data to stdout — never mixed. `--quiet` suppresses stderr logs below WARN.

### 10.3 Exit codes (semantic, stable)

| Code | Meaning |
|------|---------|
| 0 | Success; all graded cases passed (or command succeeded) |
| 1 | Ran successfully but ≥1 eval case failed |
| 2 | Infrastructure error (transport, store, I/O) |
| 3 | Validation error (bad eval file, bad flags, bad config) |
| 4 | Verify mismatch (`verify --strict` found drift) |

---

## 11. Observability (behavioral requirements)

- Span tree per run: `run → task → transport.complete` and `run → grade`. Span attributes mirror the canonical slog fields (§5.8).
- Every stored record and every log line within a run carries the same `run_id`; every task's records carry the same `task_id` and `trace_id` — full joinability across logs, store, and API responses.
- `GET /v1/runs/{id}` exposes progress counters sourced from the same Meter the Runner increments — one source of truth.

---

## 12. AI-Agent Feedback Loop

The design target: an agent (e.g., Claude Code) can run evals, read structured results, decide, and re-verify — without parsing prose.

1. **Stable schemas.** Every emitted record (`OutputRecord`, `VerdictRecord`, `RunSummary`, `Finding`, `VerifyReport`) carries `schema_version`; `goeval schema` prints the JSON Schema. Additive changes bump minor; breaking changes bump major and are called out in a CHANGELOG contract.
2. **Explainable verdicts.** `Verdict.checks[]` names each assertion with `expected` vs `got` — enough signal for the agent to modify a prompt, an eval, or code under test and re-run.
3. **Deterministic re-grading.** `goeval verify <run_id>` re-computes every verdict from stored raw outputs and compares to stored verdicts. Identical grader versions ⇒ byte-identical verdicts required. This is the agent's trust anchor: results can always be independently reproduced offline.
4. **Drift detection.** If the installed grader version differs from `VerdictRecord.grader_id`, verify reports `drift: grader_version_changed` per affected verdict instead of a silent mismatch.
5. **Replay-based loops.** With `--replay`, an agent can iterate on graders/evals against recorded model outputs with zero API cost and zero nondeterminism.
6. **Machine ergonomics.** JSONL streaming for large result sets; exit codes as control flow; stderr/stdout separation; no ANSI color when not a TTY.

---

## 13. Testing Strategy

- Table-driven unit tests for every grader (each catalog row gets positive/negative/edge cases); golden-file tests for the loader (sample EVAL.md corpus in `testdata/`).
- Runner tested exclusively against fake and `replay` transports — including scripted `ErrRateLimited`/`ErrTransient` sequences to prove retry policy, and mid-run cancellation for resume.
- `store/jsonl` crash-injection test: truncate final line, assert recovery behavior of §5.6.
- HTTP API tested via `net/http/httptest` against an Engine wired to fakes.
- One end-to-end test: real EVAL.md pack → replay transport → JSONL store → verify → report, asserting exit codes and schema validity of all outputs.
- TDD per the author's global CLAUDE.md conventions for features and bugfixes.

---

## 14. Dependency Policy

- Allowed unconditionally: Go stdlib.
- Justified third-party (pending OI-1): one YAML parser for frontmatter (`gopkg.in/yaml.v3`), because the stdlib has no YAML support and SKILL.md-style frontmatter is YAML.
- Everything else — HTTP, JSON, templates, regexp, crypto hashing, flag parsing, testing — is stdlib. No CLI framework, no logging framework, no OTel SDK, no HTTP router (stdlib `http.ServeMux` method+pattern routing suffices).

---

## 15. Security and Privacy Notes

- API keys only via env vars or config file with `0600` check (warning otherwise); never logged, never stored in cassettes (redaction hook, §7).
- `serve` binds loopback by default; non-loopback bind without auth (OI-5) emits a startup warning.
- Eval files are trusted input from the repo owner; still, `@fixtures/` resolution must reject path traversal outside the pack directory.

---

## 16. Open Items

Each needs a decision before or during the relevant milestone. Recommendations are the author's to accept or override.

**OI-1 — YAML dependency.** Options: (a) `gopkg.in/yaml.v3` as the single third-party dep; (b) hand-rolled strict YAML-subset parser (scalars, string lists, one nesting level) for zero deps. Recommendation: (a) — frontmatter already uses nested maps (`sampling`, `grader.params`) that a subset parser would handle poorly, and a hand parser is a permanent maintenance liability. **[Inference]** The risk profile of yaml.v3 (mature, widely vetted, no transitive deps) is lower than that of a bespoke parser.
- Given the decision is (a), When `go.mod` is inspected, Then `gopkg.in/yaml.v3` is the only non-stdlib require.

**OI-2 — Module path and final name.** `goeval` is a placeholder. Given a chosen module path, When the repo is initialized, Then all import paths and the binary name match it.

**OI-3 — Streaming and multimodal in Transport.** Recommendation: defer both; v1 is non-streaming text-only. Add later as an optional `Streamer` interface discovered by type assertion, so v1 Transport implementations remain valid.
- Given v1, When any transport is called, Then the full response is returned in one `Response` and no streaming API exists in the public surface.

**OI-4 — File-queue transport for the isolated VDI.** The author has prior art for file-queue bridges. Recommendation: defer to v1.1; confirm now that `Transport.Complete`'s request/response types carry everything a file-queue round trip needs (they do: fully serializable structs, `request_hash` correlation key). **[Inference]** No contract change will be needed.

**OI-5 — API authentication.** Recommendation: v1 supports optional static bearer token via `GOEVAL_API_TOKEN`; if unset and bound to loopback, auth is off.
- Given the env var is set, When a request lacks `Authorization: Bearer <token>`, Then the API returns 401 with error code `AUTH001`.

**OI-6 — Concurrency defaults and per-transport rate limiting.** Recommendation: default `--concurrency 4`; per-transport token-bucket limiter configurable as `requests_per_minute` in transport config, honoring `Retry-After` above it.

**OI-7 — Verdict score semantics for combinators under `pass_policy`.** When attempts > 1, is the case's recorded score the mean across attempts or the score of the deciding attempt? Recommendation: record every attempt's verdict; the case-level summary uses mean score + policy-derived pass. Needs confirmation because it changes `RunSummary` aggregation.

---

## 17. Build Order for Claude Code (milestones)

| # | Milestone | Exit criterion |
|---|-----------|----------------|
| M0 | Repo scaffold, `obs` (slog setup, noop tracer, IDs), `store` types + schema constants | `go vet ./... && go test ./...` green on skeleton |
| M1 | `eval`: frontmatter + body parser, validator, pack resolution | Loader golden tests pass on the testdata corpus |
| M2 | `grader`: registry + all catalog graders + combinators | Table tests per grader; `verify`-grade purity lint (no forbidden imports in grader pkg) |
| M3 | `transport` contract + `replay` + fake; `run`: planner + runner (retry, resume, cancel) | Runner suite green on scripted fakes |
| M4 | `store/jsonl` + `report` | Crash-recovery test green; summary renders in 3 formats |
| M5 | `cmd/goeval`: run/validate/verify/report/list/schema, exit codes, output contract | End-to-end test (§13) green |
| M6 | Real transports: `anthropic`, `openrouter`, `gateway` | Contract test suite (shared across transports) green against recorded cassettes |
| M7 | `api` + `serve` + cancel endpoint | httptest suite green; parity test: API and CLI produce identical `RunSummary` for the same run |
| M8 | Observability polish: span tree, metrics counters, `X-Trace-Id` | Trace/log joinability test green |

Companion document: `goeval-ACCEPTANCE.md` (Gherkin) is the acceptance contract for these milestones.

---

## Sources

- Anthropic skill-creator skill (local: `/mnt/skills/examples/skill-creator/SKILL.md`) — SKILL.md anatomy: required `name`/`description` frontmatter, description as sole trigger mechanism, <500-line body guidance, bundled `scripts/`/`references/`/`assets/`, progressive disclosure levels.
- OpenTelemetry Go trace API — https://pkg.go.dev/go.opentelemetry.io/otel/trace — `Tracer.Start(ctx, name) → (ctx, Span)` shape, TracerProvider pattern, and the explicit warning that API interfaces may gain methods without a major version bump (basis for LD-7's locally-owned shapes).
- OpenTelemetry Tracing API specification — https://opentelemetry.io/docs/specs/otel/trace/api/ — SpanContext = 16-byte TraceId + 8-byte SpanId per W3C TraceContext; span status codes Unset/OK/Error.
