# Architecture

## Layers

```
┌ Interface ──────────────────────────────────────────────┐
│  CLI (Typer)                     Web dashboard (Phase 2) │
└───────────────┬─────────────────────────────────────────┘
┌ Orchestration (Python) ─────────▼───────────────────────┐
│  Agent Planner │ Generate/Repair loop │ State Manager    │
└───────────────┬────────────────────────┬────────────────┘
        JSON schema (stdio)              │ LlmClient
┌ Verification Engine (Rust) ─▼──┐   ┌ AI Model Layer ─────▼┐
│ Sandbox │ Compiler │ DiffTest  │   │ Provider adapters,   │
│ │ Benchmark runner             │   │ routing, budgets     │
└───────────────┬────────────────┘   └──────────────────────┘
┌ Storage & Reporting ────────────▼───────────────────────┐
│ Project/Run store │ Artifacts │ Audit log │ Reports      │
└──────────────────────────────────────────────────────────┘
```

## Trust boundary (most important rule)

- **Python orchestrator + LLM output = untrusted.** It proposes; it never decides.
- **Rust verification engine = trusted.** Only it can produce a `VerificationResult`.
  The orchestrator may *read* a result but cannot construct or modify one. Results are written
  to the artifact store by the engine, include a hash of the inputs (source, candidate, test
  corpus, toolchain), and the report generator reads results only from there.
- Generated and original code only ever run **inside a sandbox**, never in the orchestrator
  or engine process.

## Components

### Orchestration (Python, `orchestrator/`)
- **Analyze:** tree-sitter parse → module/function inventory, call/dependency graph, hot-path
  hints, external-dependency list (flag C extensions, dynamic features, I/O, globals).
- **Planner:** orders modules by dependency, selects migration unit (function/module),
  assigns a risk level, chooses model tier. Output is a structured `MigrationPlan`.
- **Generator agent:** produces a Rust candidate for a unit from plan + context + prior diagnostics.
- **Repair loop:** on failure, feeds structured diagnostics (compiler errors, diff of
  mismatching outputs with minimal repro) back to the generator, up to `max_repair_attempts`;
  then escalates to the human gate. Loop is bounded by attempts **and** a token/cost budget.
- **State manager:** checkpoints after every step; resume and rollback; every transition is
  appended to the audit log.

### Verification Engine (Rust, `crates/`)
- **Sandbox:** runs a command with limits (CPU, memory, wall time, no network, read-only FS
  except scratch). Trait-based; implementations: bwrap, later Firecracker.
- **Compiler:** `cargo check`/`cargo build` of candidate in sandbox; captures structured
  diagnostics (`--message-format=json`).
- **Differential tester:** runs original and candidate on the same inputs; compares outputs
  per `verification.md`.
- **Benchmark runner:** measures original vs candidate on the same workload, with warmup,
  repeated runs, and variance reporting.

### AI Model Layer
`LlmClient` protocol → adapters. Router picks a model tier by task (plan, generate, repair,
summarize) from config. Tracks tokens and cost per call, attributed to run/unit/attempt.

### Storage & Reporting
- Project & migration store, artifacts (source snapshots, candidates, logs, bench data),
  results & reports, audit/history, analytics (success rate, repair attempts, cost).
- Report outputs: `report.html`, `bench_results.md`, plus machine-readable `report.json`.

## Core data model (conceptual)

`Project` → `Run` (one pipeline execution) → `Unit` (function/module being migrated) →
`Attempt` (one generate+verify cycle) → `VerificationResult` + `Artifacts`.
`AuditEvent` is append-only and references any of the above. `HumanReview` attaches to a `Unit`.

Unit states: `pending → planned → generating → verifying → (repairing ↔ verifying)* →
awaiting_review → approved | rejected | failed`.

## Engine ⇄ orchestrator contract

- Request/response are JSON, one document per line, over stdio of `verifier-cli`.
- Schemas are generated from Rust types into `schemas/` and checked in; Python models are
  generated or validated against them; CI fails if they drift.
- Every message carries `schema_version`. Breaking changes bump the major version.
- Engine errors are structured (`kind`, `message`, `retryable`); never free-form stderr parsing.

## Human gate

A unit goes to review when: marked critical in the plan, confidence is below threshold, repair
attempts are exhausted, it contains `unsafe`, or the equivalence report has unproven areas.
Reviewer actions: approve, reject with feedback (feeds the next generate attempt), or edit.

## Failure & cost controls

Per-run and per-unit budgets (tokens, dollars, wall time, attempts). Hitting a budget is a
normal terminal state (`failed: budget_exhausted`), reported clearly, never an exception.
