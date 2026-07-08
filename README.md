# LLM Eval Runner

A design plan for a Go-based LLM evaluation runner that helps teams compare model behavior with deterministic, repeatable checks.

> Status: This repository currently contains a design plan only. It is not an implemented system. The CLI commands, HTTP API, package layout, and contracts described here are proposed design targets.

## Who this is for

This project is for developers, engineering teams, and AI-agent builders who need a simple way to test LLM behavior across models and providers.

It is especially useful for teams that want:

- Reproducible model comparisons
- Markdown-authored evals that are easy to review in Git
- Deterministic grading instead of LLM-as-judge scoring
- A CLI that agents can run and understand
- A backend API that can be used by internal tools
- Minimal dependencies and a design that is easy to test
- A path to support private gateways, replay mode, and offline verification

## What the system is intended to do

The planned system is a private, minimal-dependency Go library and CLI for evaluating LLM capabilities across different models and transports.

Evals are written as markdown files with YAML frontmatter. Each eval describes the prompt, cases, expected outputs, and grader settings. The runner sends those cases to selected models, stores the raw outputs, grades them with deterministic Go graders, and writes machine-readable results.

The same core library is intended to power:

- A command-line tool
- A local HTTP API
- Replay-based testing
- Append-only JSONL result storage
- Agent-friendly verification loops

The design favors boring parts, small interfaces, and clear contracts over a large framework.

## Why this design is useful

### It makes model quality easier to prove

A good eval system should help a team say, "This model handled these cases better, and here is the evidence." This design stores raw outputs, verdicts, checks, and run metadata so results can be inspected instead of argued over.

### It reduces trust problems

The design avoids LLM-as-judge grading in v1. That makes the result easier to trust because verdicts come from deterministic Go code, not another probabilistic model.

### It gives agents a clean feedback loop

The output is planned to be JSON-first, versioned, and tied to semantic exit codes. That makes it easier for an AI coding agent to run evals, read failures, adjust code or prompts, and verify the result without scraping prose.

### It keeps the authoring format approachable

Markdown eval files are easy to read, review, diff, and store in a normal repository. A developer can understand an eval without learning a new DSL or opening a dashboard.

### It makes failures actionable

The planned verdict format includes named checks with expected and actual values. That matters because a failed eval should point to what broke, not just say that the run failed.

### It supports private and corporate environments

The planned transports include direct provider APIs, an internal gateway-style transport, and replay mode. That makes the design a better fit for teams that cannot send everything through one public SaaS tool.

### It lowers adoption friction

The system is designed as a Go library first, with thin CLI and HTTP adapters. Teams can start with a command-line workflow and later embed the same engine into internal tools.

### It gives engineering teams auditability

Append-only JSONL storage, immutable run plans, raw response preservation, and deterministic re-grading all support later review. This is useful when model decisions need to be explained to engineers, managers, auditors, or stakeholders.

### It separates execution from grading

The runner stores raw model output before grading. That means teams can replay and re-grade stored outputs without spending more API money or changing the original evidence.

### It is easy to test

Small interfaces, replay transport, fake transports, deterministic task IDs, and file-backed storage make the planned system testable without live model calls.

### It is positioned around a clear pain

The central promise is simple: compare LLMs with evidence you can reproduce. That is stronger than a vague promise like "better AI evaluation" because it speaks to a real buyer problem: confidence before changing models, prompts, or agent behavior.

## Major benefits from a copywriting and persuasion perspective

### Clear value proposition

The system has a direct promise: run repeatable evals and get deterministic verdicts. That is easy to explain in one sentence and easy for a technical audience to understand.

### Strong trust signal

Deterministic graders, replay mode, raw output storage, and `verify` all support the same message: the result can be checked again. That creates trust.

### Strong contrast against noisy eval tools

Many LLM evaluation tools lean on model-graded results. This design can position itself as the simpler, more predictable alternative for teams that want hard checks first.

### Low-risk adoption story

Because the design is library-first, CLI-first, and minimal-dependency, it can be presented as something a team can adopt gradually. That reduces resistance.

### Good fit for developer buyers

The system speaks in terms developers already care about: markdown files, JSONL, exit codes, package boundaries, replay tests, typed errors, and Git-friendly artifacts.

### Concrete proof points

The design includes concrete mechanisms that can be used in marketing copy or documentation:

- Deterministic Go graders
- Markdown eval files
- Replay transport
- Append-only JSONL store
- Machine-readable verdicts
- CLI and HTTP API parity
- Stable schema versions
- Semantic exit codes
- Offline verification

These are more persuasive than broad claims because they make the system feel specific and buildable.

### Built-in answer to "Why should I trust it?"

The answer is: raw model outputs are stored, grading is deterministic, and `verify` can reproduce verdicts. That is a strong trust loop.

### Built-in answer to "Will this work with our stack?"

The planned transports, gateway support, and Go library design make the system easier to position for internal engineering teams rather than only hobby projects.

### Built-in answer to "Can an agent use it?"

Yes. The design is intentionally agent-readable: JSON output, schema versions, stable findings, replay mode, and exit codes. That makes it useful for Claude Code-style implementation loops.

### Good naming angle

The best name should be plain and searchable. This is an engineering tool, so clarity matters more than cleverness.

Recommended repository name:

```text
llm-eval-runner-go
```

Good alternatives:

```text
llm-eval-kit-go
model-eval-runner
agent-eval-kit-go
eval-runner-go
deterministic-llm-evals
```

I would avoid `goeval` as the final name because it is too broad and already appears to be used by other public projects.

## Planned high-level design

The proposed pipeline is:

```text
EVAL.md / eval pack
  -> Loader
  -> Planner
  -> Runner
  -> Transport
  -> Store
  -> Grader
  -> Reporter
```

The core design rule is that the CLI and HTTP API should call the same library facade. That keeps behavior consistent and avoids duplicated logic.

## Planned components

### Eval loader

Reads a single `EVAL.md` file or an eval pack directory. The file format uses YAML frontmatter and markdown sections.

### Planner

Expands eval specs, models, attempts, and cases into a fully materialized run plan.

### Runner

Executes the plan with concurrency, retries, timeouts, cancellation, resume support, and transport selection.

### Transports

The planned v1 transports are:

- OpenRouter
- Anthropic direct API
- Internal gateway
- Replay mode

### Store

Stores run data in append-only JSONL files. The design preserves raw outputs and verdict records separately.

### Graders

Deterministic Go graders check exact matches, substrings, regexes, JSON structure, numeric tolerance, sets, golden files, length bounds, and grader combinations.

### Reporter

Produces summaries in JSON, JSONL, or text.

### HTTP API

Provides local API access for runs, results, summaries, validation, verification, and health checks.

## Current repository contents

At this stage, this repository should be treated as a planning repository. It may contain:

- The design plan
- Acceptance tests
- Architecture notes
- Future implementation milestones

It should not be presented as a working CLI, library, or service until code is implemented and tested.

## Suggested next steps

1. Confirm the final repository name.
2. Confirm the Go module path.
3. Add the design plan as `PLAN.md`.
4. Add acceptance tests as `ACCEPTANCE.md`.
5. Open issues for the major milestones.
6. Start implementation with the store types, observability shapes, and eval loader.
