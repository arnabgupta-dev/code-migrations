# Tech Stack & Decisions

Status: **proposed baseline, 2026-10-21**. Change a decision by editing this file and adding a row
to the decision log at the bottom, not by silently diverging in code.

Versions: pin exact versions in lockfiles (`Cargo.lock`, `uv.lock`, `pnpm-lock.yaml`); this doc
names tools, not versions. Use latest stable at the time a component is created.

## Summary

| Layer | Choice | Phase |
|-------|--------|-------|
| Verification engine | **Rust** (edition 2024), Cargo workspace | 0 |
| Orchestration & agents | **Python ≥3.12**, `uv`, **LangGraph** for the pipeline graph | 0 |
| Python ⇄ Rust boundary | Versioned **JSON schema over stdio** (JSON Lines); `axum` HTTP service later | 0 → 2 |
| Sandbox | **bubblewrap + rlimits + cgroups** behind a `Sandbox` trait; **Firecracker** later | 0 → 3 |
| Source analysis | **tree-sitter** (Python first; TS, C later) | 0 |
| Differential test inputs | **Hypothesis** + recorded traces/examples; fixed seeds | 0 |
| LLM access | Thin in-house **`LlmClient` protocol** + provider adapters, config-driven routing | 0 |
| CLI | **Typer** (Python) shipping the Rust engine binary | 1 |
| Persistence | **SQLite + local FS** → **Postgres + S3-compatible** behind repository interfaces | 0–1 → 2 |
| API | **FastAPI** (Python) + OpenAPI | 2 |
| Job queue | Postgres-backed queue (`SKIP LOCKED`); no Redis initially | 2 |
| Web | **Next.js** (App Router) + TypeScript + Tailwind + shadcn/ui, **pnpm** | 2 |
| Live progress | Server-Sent Events | 2 |
| Task runner | **just** | 0 |
| CI | GitHub Actions (assumed; confirm hosting) | 0 |

## Rationale per decision

### Rust for the verification engine
The plan makes this the trust engine: sandboxing untrusted generated code, comparing outputs,
running compilers and benchmarks. Needs speed, memory safety, small footprint, strong typing for
the result schemas. Also dogfoods the language we're promoting. Libraries: `tokio`, `clap`,
`serde`/`serde_json`, `thiserror` (library crates) + `anyhow` (binaries only), `tracing`,
`criterion` (benches), `proptest` (engine's own tests), `cargo-nextest`, `cargo-deny`.

### Python + LangGraph for orchestration
Best agent ecosystem. LangGraph fits because the pipeline *is* a graph with a conditional
repair loop and needs **checkpointing/resume** (plan's State Manager: checkpoints, rollback,
audit trail). Guardrail to avoid framework lock-in: **domain logic lives in plain functions and
Pydantic models; LangGraph only wires them together.** If it becomes a drag, replacing it must
touch one module. Libraries: `pydantic` v2, `httpx`, `structlog`, `typer`, `pytest`, `hypothesis`,
`ruff`, `pyright` (strict).

### JSON-over-stdio now, HTTP service later
Simplest boundary that is language-agnostic, debuggable (a human can replay a request), and
needs no daemon for the CLI. Schemas are **versioned and generated from Rust types**
(`schemars`) so Python models cannot drift. When the web platform needs a long-running verifier
fleet, the same schemas are served by `axum` — no redesign. Avoid PyO3/FFI: it couples the
untrusted orchestrator to the trust engine's process and complicates distribution.

### Sandbox: bubblewrap first, Firecracker later, WASM rejected for now
Dev machine and CI are Linux with `bwrap` and `/dev/kvm` available.
- **Phase 0–1:** `bubblewrap` (namespaces: no network, read-only root, tmpfs workdir) + rlimits +
  cgroup CPU/memory/time limits. Fast, no daemon, good enough for single-user CLI.
- **Phase 2–3:** Firecracker microVMs for multi-tenant hosted runs (stronger isolation boundary
  for untrusted code from strangers). Same `Sandbox` trait, so it is an implementation swap.
- **WASM rejected as the primary sandbox:** equivalence testing must run the *original* program
  in its *native* runtime (CPython, Node, C). Running it under a WASM port can change behavior
  (floats, threading, syscalls, C extensions) and would undermine the proof. May reappear as an
  extra isolation layer for the Rust candidates only.

### Differential testing: custom, in Rust
Cases are `(function/entrypoint, JSON inputs)`; the engine runs the original and the Rust
candidate in sandboxes and compares **return value, raised error class, stdout/stderr, exit
code**, and optionally side effects (files, env). Input sources: developer-provided tests,
recorded traces, and **Hypothesis**-generated inputs (seeded, saved so runs are reproducible).
Full spec in `verification.md`.

### LLM layer: own thin protocol, not a heavyweight gateway dependency
Plan requires multi-provider routing to avoid lock-in. A minimal `LlmClient` protocol
(`complete`, tool-calling, token/cost accounting) with adapters (Anthropic first, then an
OpenAI-compatible adapter that also covers local/DeepSeek-style endpoints) is small, auditable,
and avoids a large third-party dependency sitting in the path of every prompt and API key.
Routing rules (task → model tier, budget caps, fallbacks) live in **config files**; model IDs
are never hardcoded in Python/Rust source. Prompts are versioned files under `orchestrator/prompts/`.

### Storage
CLI phases need zero infrastructure: **SQLite + a local artifacts directory**. Everything goes
through repository interfaces (`ProjectRepo`, `RunRepo`, `ArtifactStore`, `AuditLog`) so Phase 2
swaps in **Postgres + S3-compatible (MinIO locally)** without touching pipeline code.
SQLAlchemy 2.x + Alembic for schema/migrations. The audit log is **append-only**.

### Web (Phase 2, not before)
Next.js + TypeScript per the plan. API types are **generated from FastAPI's OpenAPI** so the
frontend never hand-writes request/response types. Auth: decide at Phase 2 start (open item).
No web code before Phase 2 — the CLI is the product until then.

## Repository layout (target)

```
code-migration/
├── CLAUDE.md
├── docs/                    # plan, architecture, tech-stack, verification, coding-standards
├── crates/                  # Rust workspace (Apache-2.0, open-source candidate)
│   ├── verifier-core/       # result/case schemas, comparison logic (no I/O)
│   ├── sandbox/             # Sandbox trait + bwrap impl (+ firecracker later)
│   ├── difftest/            # differential tester
│   ├── bench/               # benchmark runner
│   └── verifier-cli/        # JSON-over-stdio binary
├── orchestrator/            # Python package (uv project)
│   ├── src/migrator/{analyze,plan,generate,repair,llm,state,cli}/
│   ├── prompts/
│   └── tests/
├── corpus/                  # Python→Rust migration examples + adversarial cases (ground truth)
├── web/                     # Next.js (Phase 2)
├── schemas/                 # generated JSON schemas (checked in)
└── justfile
```

## Open decisions (need the owner)

1. **License.** Recommendation: Apache-2.0 for `crates/` + `corpus/` (open verifier = credibility,
   per plan); orchestrator/web license TBD (proprietary or source-available).
2. **CI/hosting provider** (GitHub assumed).
3. **Web auth approach** (before Phase 2).
4. **Default LLM provider/tier for Phase 0** (config only; no code impact).
5. **Python version floor** for *migrated* projects the tool supports (separate from our own ≥3.12).

## Decision log

| Date | Decision | Alternatives rejected |
|------|----------|-----------------------|
| 2026-10-21 | Rust verifier / Python orchestrator split, per PDF | All-Python (weak trust story); all-Rust (slow agent iteration) |
| 2026-10-21 | LangGraph with domain logic outside it | Fully custom state machine (re-implements checkpointing) |
| 2026-10-21 | JSON-over-stdio, schemas generated from Rust | PyO3/FFI; gRPC from day one |
| 2026-10-21 | bwrap → Firecracker; WASM not primary | WASM-only (changes original behavior); Docker-per-run (slow, daemon) |
| 2026-10-21 | Own `LlmClient` protocol | LiteLLM/LangChain as mandatory gateway |
| 2026-10-21 | SQLite→Postgres behind repo interfaces | Postgres from day one (infra tax for a CLI) |
| 2026-10-21 | Python → Rust first | Node/C first (less demand/AI support, per PDF) |
