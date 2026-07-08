# goeval — Gherkin Acceptance Tests

Companion to `goeval-PLAN.md`. Section numbers reference the plan. Scenarios are the exit contract for milestones M0–M8; each should map to at least one automated test. Scenarios tagged `@open-item` depend on an Open Item decision and must be revisited if the recommendation is overridden.

Conventions: "the store" = `store/jsonl` unless stated; "a fake transport" = an in-memory scripted Transport; all CLI scenarios assume non-TTY stdout unless stated (JSON default output per §10).

---

## Feature: Eval loading and validation (§5.2, §6)

```gherkin
Scenario: Load a valid single-file eval
  Given a valid EVAL.md with required frontmatter and Prompt and Cases sections
  When Loader.Load is called with its path
  Then an EvalSpec is returned with name, description, version, grader config, and all cases populated
  And no error is returned

Scenario: Missing required frontmatter field
  Given an EVAL.md whose frontmatter omits "description"
  When the eval is validated
  Then a Finding with severity "error" and a stable code is returned
  And the Finding's path points at the frontmatter

Scenario: Unknown frontmatter key is rejected
  Given an EVAL.md with frontmatter key "samplng" (typo)
  When the eval is validated
  Then a Finding with severity "error" identifies the unknown key by name

Scenario: Malformed YAML frontmatter yields a positioned parse error
  Given an EVAL.md whose frontmatter is not valid YAML
  When Loader.Load is called
  Then a ParseError containing line and column is returned
  And no partial EvalSpec is returned

Scenario: Case block with invalid JSON
  Given an EVAL.md whose Cases section contains a fenced case block with invalid JSON
  When the eval is validated
  Then a Finding with severity "error" identifies the offending case block by position

Scenario: Duplicate case IDs
  Given an EVAL.md with two cases sharing id "c1"
  Then validation returns an "error" Finding naming the duplicate id

Scenario: Unrecognized body section is a warning
  Given an EVAL.md containing a "## Scratch" section
  When the eval is validated
  Then a Finding with severity "warning" is returned
  And the eval remains runnable

Scenario: Duplicate eval names within a suite
  Given two EVAL.md files in one suite directory sharing name "x"
  When LoadDir is called on the suite
  Then a Finding with severity "error" reports the collision and both file paths

Scenario: Per-case grader override
  Given a frontmatter grader "exact@1" and a case-level grader "regex@1" on case "c2"
  When the EvalSpec is inspected
  Then case "c2" resolves to "regex@1" and all other cases resolve to "exact@1"

Scenario: validate command exit code
  Given a directory containing one invalid eval
  When "goeval validate <dir>" runs
  Then stdout contains a JSON array of Findings
  And the exit code is 3
```

## Feature: Pack directories and fixtures (§6.3)

```gherkin
Scenario: Fixture reference resolves at load time
  Given a pack whose case vars include {"doc": "@fixtures/inv1.txt"}
  And fixtures/inv1.txt exists in the pack
  When the eval is loaded
  Then the case var "doc" contains the file's contents
  And no file I/O occurs later during grading

Scenario: Fixture reference in a single-file eval is rejected
  Given a standalone EVAL.md (no pack directory) using an "@" var reference
  Then validation returns an "error" Finding citing the self-sufficiency rule

Scenario: Path traversal in fixture reference is rejected
  Given a pack case var {"doc": "@../../etc/passwd"}
  When the eval is loaded
  Then loading fails with an "error" Finding and no file outside the pack is read

Scenario: Golden file grader reads from expected/
  Given a pack using grader golden@1 with file "expected/out1.txt"
  When the eval is loaded
  Then the golden content is materialized into the EvalSpec at load time
```

## Feature: Prompt templating (LD-9, §6.2)

```gherkin
Scenario: Variables render into the prompt
  Given a Prompt section "Summarize: {{.doc}}" and a case with vars {"doc": "hello"}
  When the request for that case is built
  Then the rendered prompt is "Summarize: hello"

Scenario: Missing template variable fails validation
  Given a Prompt referencing {{.missing}} and a case that does not define "missing"
  When the eval is validated
  Then an "error" Finding names the unresolved variable and the case id

Scenario: System prompt is passed through unrendered vars only if templated
  Given a System Prompt section with static text
  When the request is built
  Then Request.system equals the section text verbatim
```

## Feature: Deterministic graders (§5.3, §5.4)

```gherkin
Scenario: exact grader with normalization
  Given grader exact@1 with params {trim: true, case_insensitive: true}
  And expected "Paris" and model output "  paris \n"
  When Grade is called
  Then the Verdict passes with score 1.0

Scenario: contains grader all/any
  Given grader contains@1 with params {all: ["alpha", "beta"]}
  And output containing "alpha" but not "beta"
  Then the Verdict fails
  And checks[] contains a failing check named for "beta" with expected and got populated

Scenario: regex must_not_match
  Given grader regex@1 with {patterns: ["^\\d+$"], must_not_match: ["[a-z]"]}
  And output "42abc"
  Then the Verdict fails with both a pattern miss and a forbidden match recorded in checks[]

Scenario: json_struct tolerates one fenced code block
  Given grader json_struct@1 asserting {path: total.amount, op: eq, value: 1042.5}
  And output wrapping valid JSON in a single ```json fence
  Then the JSON is parsed and the assertion evaluates against it

Scenario: json_struct on unparseable output
  Given output that is not JSON and not a fenced JSON block
  Then the Verdict fails with a single check "parse" explaining the failure
  And no grader error is returned

Scenario: numeric tolerance
  Given grader numeric@1 with {expected: 3.14159, abs_tol: 0.001, extract: "last"}
  And output "The value of pi is approximately 3.1416."
  Then the Verdict passes

Scenario: set grader unordered
  Given grader set@1 with {expected: ["a","b","c"], ordered: false, delimiter: "\n"}
  And output "c\na\nb"
  Then the Verdict passes

Scenario: length bounds
  Given grader length@1 with {max_lines: 3}
  And a five-line output
  Then the Verdict fails and checks[] reports expected "<=3 lines" and got "5 lines"

Scenario: Grader misconfiguration is an error, not a verdict
  Given grader regex@1 with an invalid pattern "("
  When Grade is called
  Then an error is returned and no Verdict is produced

Scenario: Unknown grader param rejected at validation
  Given frontmatter grader exact@1 with param "trimm": true
  Then validation returns an "error" Finding naming the unknown parameter

Scenario: Grader purity is enforced structurally
  Given the grader package source tree
  When the purity check runs (forbidden-import lint for net, os, time.Now, math/rand)
  Then the check passes for every registered grader
```

## Feature: Grader combinators (§5.4)

```gherkin
Scenario: all_of fails when any child fails
  Given all_of@1 over [exact@1 (passing), regex@1 (failing)]
  Then the composite Verdict fails
  And checks[] nests both children's checks with distinguishable names

Scenario: any_of passes when one child passes
  Given any_of@1 over [exact@1 (failing), contains@1 (passing)]
  Then the composite Verdict passes

Scenario: not inverts
  Given not@1 over contains@1 with {all: ["I cannot"]} and output "I cannot help"
  Then the Verdict fails
  And the same grader over output "Sure, here's the answer" passes

Scenario: weighted score and threshold
  Given weighted@1 with children scoring 1.0 and 0.0 and weights 3 and 1 and threshold 0.7
  Then the composite score is 0.75 and the Verdict passes

Scenario: Combinators nest
  Given all_of@1 containing any_of@1 containing two graders
  When Grade is called
  Then evaluation succeeds and nested checks preserve the hierarchy in their names
```

## Feature: Transport contract and replay (§5.1, §7)

```gherkin
Scenario: Typed error taxonomy from HTTP status codes
  Given an HTTP transport receiving status 429 with Retry-After "7"
  When Complete is called
  Then the returned error is ErrRateLimited with RetryAfter of 7 seconds
  And status 503 maps to ErrTransient, 401 to ErrAuth, 400 to ErrBadRequest

Scenario: Transports never retry internally
  Given a fake HTTP backend that fails once with 503 then succeeds
  When Complete is called once
  Then exactly one HTTP request was made and ErrTransient was returned

Scenario: Raw provider body is preserved
  Given any successful completion
  Then Response.raw equals the untouched provider response body byte-for-byte

Scenario: Record mode writes a cassette
  Given a replay transport wrapping a fake transport in record mode
  When two distinct requests complete
  Then the cassette JSONL contains two entries keyed by their request_hash
  And no Authorization header value appears anywhere in the cassette

Scenario: Playback serves by request hash
  Given a cassette recorded from a request R
  And a replay transport in playback mode
  When Complete is called with an identical request R
  Then the recorded response is returned without any network access

Scenario: Cassette miss is a distinct error
  Given playback mode and a request absent from the cassette
  Then ErrCassetteMiss is returned identifying the request_hash

Scenario: Request hash canonicalization
  Given two Request values equal in content but constructed with different map orderings in metadata
  Then their request_hash values are identical
```

## Feature: Planner and Runner (§5.5, §8)

```gherkin
Scenario: Plan is fully materialized and deterministic
  Given 2 evals with 3 cases each, 2 models, and attempts 2
  When Plan is called
  Then the RunPlan contains exactly 24 tasks
  And task_ids are stable across repeated Plan calls with identical inputs

Scenario: Concurrency bound respected
  Given a fake transport that records concurrent in-flight calls
  And RunOptions with concurrency 3 over 20 tasks
  When the run executes
  Then the maximum observed in-flight count is 3

Scenario: Retry on transient errors with backoff
  Given a fake transport scripted to return ErrTransient twice then succeed for one task
  And retry max 3
  When the run executes
  Then the task succeeds and the store shows attempt metadata reflecting 3 transport calls
  And ErrAuth on another task fails fast with exactly one call

Scenario: Rate limit honors Retry-After
  Given a fake transport returning ErrRateLimited with RetryAfter 2s then success
  When the run executes with an injected fake clock
  Then the retry waited at least 2s of fake-clock time

Scenario: Raw output stored before grading
  Given a grader wired to panic for a specific case
  When the run executes with panic recovery
  Then that case's OutputRecord exists in the store
  And its VerdictRecord records a grading failure rather than being absent

Scenario: pass_policy majority across attempts
  Given attempts 3 with scripted outputs passing, failing, passing
  And pass_policy "majority"
  Then the case-level result is pass
  And all three attempt-level verdicts are individually stored

Scenario: Resume skips completed tasks
  Given a run interrupted after 5 of 12 tasks completed
  When Run is invoked with the same run_id
  Then exactly 7 transport calls are made
  And the final manifest status is "completed"

Scenario: Cancellation marks the run resumable
  Given a run in progress
  When the context is cancelled
  Then in-flight tasks stop within task_timeout
  And the manifest's last state is "interrupted"

Scenario: One failing task does not abort the run
  Given one task whose transport call always returns ErrBadRequest
  When the run executes without --fail-fast
  Then all other tasks complete and the summary reports one infrastructure-failed task
```

## Feature: JSONL store (§5.6, LD-11)

```gherkin
Scenario: Records are append-only single lines
  Given a completed run
  When outputs.jsonl is inspected
  Then every record occupies exactly one line of valid JSON
  And every record carries schema_version, run_id, task_id, and an RFC3339 UTC timestamp

Scenario: Manifest supersession by append
  Given a run whose status changed running → interrupted → completed
  Then manifest.jsonl contains three lines in order
  And GetRun returns the state of the final line

Scenario: Truncated final line is tolerated
  Given an outputs.jsonl whose last line is truncated mid-record
  When Outputs is iterated
  Then all complete records are returned
  And a warning Finding reports one skipped partial record
  And no error aborts the read

Scenario: Plan file is immutable
  Given a stored run
  When resume executes
  Then plan.json's content hash is unchanged from run creation

Scenario: ListRuns filtering
  Given three stored runs with differing suites and statuses
  When ListRuns is called with a suite filter
  Then only matching manifests are returned, newest first
```

## Feature: Observability (§5.8, §11)

```gherkin
Scenario: Canonical fields on every in-run log line
  Given a run executed with the JSON slog handler
  Then every log line emitted during task execution contains run_id, task_id, eval_id, case_id, model, attempt, trace_id, and span_id

Scenario: Span tree shape
  Given a run of one task using the LogTracer
  Then span start/end records exist for run, task, transport.complete, and grade
  And the task span's trace_id equals the run span's trace_id

Scenario: Trace and store joinability
  Given a completed run
  Then the trace_id logged for a task equals a field retrievable for that task's records via the store or manifest

Scenario: Noop tracer is the default
  Given an Engine constructed without a tracer
  When a run executes
  Then no tracing output is produced and no nil dereference occurs

Scenario: W3C-sized IDs
  When a trace_id and span_id are generated
  Then the trace_id is 32 lowercase hex characters and the span_id is 16
  And neither is all zeros

Scenario: Metrics counters increment
  Given a run of 4 tasks where 1 fails transiently once before succeeding
  Then the Meter records tasks_started 4, tasks_succeeded 4, retries 1
```

## Feature: CLI contract (§10)

```gherkin
Scenario: run happy path exit code and JSON output
  Given a valid eval and a replay cassette covering all requests
  When "goeval run eval.md --models m1 --replay c.jsonl" runs with non-TTY stdout
  Then stdout is a single JSON RunSummary with schema_version
  And stderr contains only log lines
  And the exit code is 0

Scenario: Eval failures exit 1
  Given a run where at least one case fails grading
  Then the exit code is 1 and the summary lists the failing case with its checks

Scenario: Infrastructure failure exits 2
  Given a transport whose credentials are invalid
  When "goeval run" executes
  Then the exit code is 2 and the error output carries a stable error code

Scenario: TTY defaults to text output
  Given stdout is a TTY
  When "goeval report <run_id>" runs without --output
  Then the output is human-readable text
  And with "--output json" it is the JSON RunSummary

Scenario: schema command emits JSON Schema
  When "goeval schema verdict" runs
  Then stdout is a valid JSON Schema document describing VerdictRecord
  And a stored VerdictRecord from a real run validates against it

Scenario: list runs as JSONL
  Given two stored runs
  When "goeval list runs --output jsonl" runs
  Then stdout contains one manifest JSON object per line
```

## Feature: HTTP API (§9)

```gherkin
Scenario: Start a run asynchronously
  Given the server is running with a replay-backed Engine
  When POST /v1/runs is sent with eval paths and models
  Then the response is 202 with a run_id
  And GET /v1/runs/{id} subsequently reports progress counters until status "completed"

Scenario: Results as NDJSON stream
  Given a completed run
  When GET /v1/runs/{id}/results is sent with Accept: application/x-ndjson
  Then the body streams one VerdictRecord per line

Scenario: Error envelope and codes
  When GET /v1/runs/nonexistent is sent
  Then the response is 404 with body {error: {code, message}} using a stable code

Scenario: Trace ID header
  When any request is sent
  Then the response carries X-Trace-Id
  And an inbound trace ID is propagated into the run's spans and logs

Scenario: Cancel a run
  Given a run in progress
  When DELETE /v1/runs/{id} is sent
  Then the response is 202 and the manifest reaches status "interrupted"

Scenario: CLI/API parity
  Given the same eval, cassette, and seed
  When a run is executed via CLI and another via POST /v1/runs
  Then their RunSummary documents are identical except run_id and timestamps

@open-item
Scenario: Bearer token auth when configured (OI-5)
  Given GOEVAL_API_TOKEN is set
  When a request lacks the bearer token
  Then the response is 401 with error code AUTH001
  And a request with the token succeeds
```

## Feature: Agent feedback loop (§12)

```gherkin
Scenario: verify reproduces verdicts byte-identically
  Given a completed run with stored outputs and verdicts
  When "goeval verify <run_id> --strict" runs with unchanged grader versions
  Then every recomputed verdict is byte-identical to the stored verdict
  And the exit code is 0

Scenario: verify detects tampering
  Given a stored VerdictRecord whose pass field was manually flipped
  When "goeval verify <run_id> --strict" runs
  Then the mismatch is reported with task_id and differing fields
  And the exit code is 4

Scenario: Grader version drift is named, not conflated
  Given stored verdicts produced by regex@1 and an installed grader registered as regex@2
  When verify runs
  Then affected verdicts are reported as drift "grader_version_changed" rather than mismatches

Scenario: Offline iteration via replay
  Given a cassette from a previous run
  When an eval's grader params are modified and "goeval run --replay" executes
  Then new verdicts are produced with zero network access
  And outputs are identical to the original run's outputs

Scenario: Checks carry actionable expected/got
  Given any failing verdict in a run summary
  Then each failing check includes non-empty name, expected, and got fields

Scenario: schema_version present on every emitted record
  Given a completed run
  Then every OutputRecord, VerdictRecord, manifest line, and the RunSummary contains schema_version
```

## Feature: Configuration and dependency policy (LD-10, §14)

```gherkin
Scenario: Precedence flags over env over file
  Given a config file setting concurrency 2, env GOEVAL_CONCURRENCY=4, and flag --concurrency 8
  When the effective config is resolved
  Then concurrency is 8
  And removing the flag yields 4, and removing the env var yields 2

Scenario: Zero-config run
  Given only an eval path and transport credentials in env
  When "goeval run" executes
  Then the run completes using documented defaults

@open-item
Scenario: Single third-party dependency (OI-1)
  When go.mod is inspected
  Then the only non-stdlib requirement is the YAML parser

Scenario: Secrets never logged or persisted
  Given a run against an HTTP transport with an API key set
  Then the key value appears in no log line, no store record, and no cassette
```
