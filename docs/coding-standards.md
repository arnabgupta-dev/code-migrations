# Coding Standards

Rules for humans and AI agents writing code in this repo. Where a rule has a reason, the reason
is given so you can judge edge cases. Tooling enforces what it can; CI is the final word.

## 1. Universal rules

1. **Verification-first.** Anything touching verification (comparison, sandbox, result
   handling) needs tests written *with* the change, including a negative test (a case that must fail).
2. **Never weaken a check to make something pass.** No loosening tolerances, deleting test cases,
   adding `#[ignore]`/`skip`/`xfail`, or widening comparisons without an explicit human decision
   recorded in the PR/audit trail.
3. **Untrusted code runs only in the sandbox.** Original code, generated code, and anything from
   a cloned repo is untrusted. Never `exec`/`eval`/`subprocess` it from the orchestrator or engine
   process directly; always through the `Sandbox` trait.
4. **No secrets in code, logs, prompts, fixtures, or reports.** API keys come from env/secret store.
   Redact before logging. Cloned repos may contain secrets — never send files matching secret
   patterns to an LLM.
5. **Config over constants for anything that changes**: model IDs, budgets, retry limits,
   timeouts, thresholds. Never hardcode model names in source.
6. **Determinism.** Seed all randomness, record seeds, avoid wall-clock in logic (inject a clock).
   Same inputs ⇒ same verdict.
7. **Small, boring, explicit.** Prefer straightforward code over clever abstraction. No premature
   generality: abstract on the *third* use, except at designed seams (`Sandbox`, `LlmClient`,
   repositories) which exist for planned swaps.
8. **Fail loudly and structurally.** Typed errors with context; no silent fallbacks, no
   swallowed exceptions, no `except Exception: pass`.
9. **Comments explain *why*, not *what*.** Document invariants, trade-offs, and non-obvious
   constraints. No commented-out code, no TODOs without an issue reference or owner.
10. **Match the surrounding code** — naming, structure, comment density, idiom.
11. **Every public item has a doc comment** stating contract, errors, and edge cases.
12. **Don't add dependencies casually.** Justify each in the PR. Prefer std/existing deps.
    Rust: `cargo-deny` must pass (licenses, advisories). Python: pin via `uv.lock`.

## 2. Design principles & patterns

Principles are tools for a *present* problem, not a checklist to satisfy. If applying one makes
the code harder to read than the problem it solves, don't. Rule 1.7 (abstract on the third use,
except at the designed seams) still holds: **patterns are introduced to solve a concrete
need, never speculatively.**

### SOLID — and how it maps to this codebase

| Principle | Rule here | Concrete application |
|-----------|-----------|----------------------|
| **S**ingle Responsibility | One reason to change per module/type/function. Split I/O from logic. | `verifier-core` = pure comparison logic; `sandbox` = process isolation; `difftest` = orchestrating a comparison run. Planner ≠ generator ≠ repairer. |
| **O**pen/Closed | Extend by adding an implementation, not by editing a growing `match`/`if` chain in the caller. | New sandbox backend, LLM provider, source-language analyzer (Python → TS → C), or report format = new impl behind an existing trait/protocol. |
| **L**iskov Substitution | Every implementation honors the full contract, including errors, limits, and timeouts. | Any `Sandbox` must enforce the same `Limits` semantics (bwrap and Firecracker are interchangeable); any `LlmClient` reports tokens/cost the same way. Write **one shared contract test suite** run against every implementation. |
| **I**nterface Segregation | Small, role-specific traits/protocols. Callers depend only on what they use. | Separate `ArtifactStore`, `RunRepo`, `AuditLog` (not one `Storage` god-trait); separate `Compiler`, `Runner`, `Comparator`. |
| **D**ependency Inversion | High-level policy depends on abstractions; concrete deps are injected at the composition root. | Pipeline/graph code receives `LlmClient`, `Sandbox`, repositories as constructor/function arguments. Construction happens in one place (`main`/CLI/app factory). No hidden globals or singletons. |

**Rust specifics:** Rust has no inheritance — achieve all of the above with traits + composition.
Default to **generics (static dispatch)**; use `dyn Trait` where you need runtime selection
(e.g. choosing the sandbox backend from config) or to cut compile-time bloat. Keep traits small and
object-safe where they may be used as `dyn`. Provide default methods to minimize what implementors
must write.
**Python specifics:** use `typing.Protocol` (structural typing) for seams; **constructor
injection**, no DI framework. Prefer composition over inheritance; inheritance only for true
"is-a" with shared behavior, and never more than one level deep.

### Patterns we expect to use (and where)

| Pattern | Use it for | Notes |
|---------|-----------|-------|
| **Strategy** (traits/protocols) | Interchangeable algorithms: sandbox backend, LLM provider, comparison normalizer, model-routing policy | In Rust this is just a trait — no class hierarchy needed |
| **Repository** | Persistence (`RunRepo`, `ArtifactStore`) | Lets SQLite→Postgres be a swap |
| **Adapter** | Wrapping provider SDKs / external tools behind our `LlmClient`, `Compiler` | Keep SDK types from leaking out |
| **Factory / composition root** | Building the object graph from config | One place; everything else receives its deps |
| **Builder** (type-state where it helps) | Complex config/request types (`SandboxSpec`, `RunConfig`) | Required fields enforced at compile time in Rust |
| **Newtype** | Distinct IDs/units (`UnitId`, `AttemptNo`, `Bytes`, `Millis`) | Zero-cost; prevents mixing up arguments |
| **Type-state / enum state machines** | Unit lifecycle, run phases, sandbox lifecycle | Make illegal transitions unrepresentable (Rust) or explicit transition tables (Python) |
| **RAII / `Drop`** | Sandbox cleanup, temp dirs, process groups | Cleanup must be guaranteed on every path |
| **Command / event log** | Audit trail, checkpoint & resume, replay | Append-only; state is derivable from events |
| **Pipeline / chain of responsibility** | Analyze → Plan → Generate → Verify stages; comparison normalizer chain | Each stage: typed input → typed output |
| **Observer / pub-sub (light)** | Progress events to CLI/web (SSE later) | Don't add a framework for it |
| **Circuit breaker / bulkhead** | Failing LLM provider, runaway repair loops | See §9 |
| **Visitor / tree walking** | tree-sitter AST analysis | Use the library's cursor; don't hand-roll |
| **Null Object / `Option`** | Absent optional behavior (e.g. no benchmark requested) | Prefer over `None` checks scattered everywhere |

**Do not use:** Singleton/global mutable state (use injection), deep inheritance hierarchies,
service locators, "manager"/"util" grab-bag classes, reflection/monkey-patching for control flow,
metaclass magic, or macros where a function works.

### Anti-patterns to reject in review

- **Cloning to silence the borrow checker** (Rust). Restructure ownership/borrowing first;
  clone only with a stated reason.
- **`#![deny(warnings)]` in source** (Rust). It breaks builds on new compiler lints. Enforce
  warnings-as-errors in CI (`-D warnings`), not in the crate.
- **`Deref` to fake inheritance** (Rust). Only smart pointers implement `Deref`.
- **Stringly-typed APIs and boolean flags.** Use enums/newtypes/types that convey meaning.
- **God objects / god modules**, and "just in case" parameters or config knobs.
- **Primitive obsession** at module boundaries.
- **Premature optimization** — measure first (the project has a benchmark runner; use it).
- **Catch-all exception handling / ignoring `Result`.**
- **Boolean-returning validators** that discard *why* something failed; return typed errors.
- **Leaky abstractions**: provider/sandbox/DB specifics visible past their seam.
- **Copy-paste of near-identical code across the three language layers** — fix by sharing
  schemas (generated), not by duplicating logic.

## 3. Rust (verification engine and everything under `crates/`)

**Toolchain & lints**
- Edition 2024; `rust-toolchain.toml` pins channel. MSRV documented in the workspace manifest.
- `cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings` must be clean.
- Workspace lints in root `Cargo.toml`: `unsafe_code = "forbid"` (override per-crate only with
  justification), `clippy::unwrap_used`, `clippy::expect_used`, `clippy::panic`,
  `clippy::todo`, `clippy::dbg_macro` = `deny` in non-test code.

**Style**
- Errors: `thiserror` enums in library crates; `anyhow` only in binaries at the top level.
  Every error carries context (what operation, which unit/path).
- No panics on any input from outside the process (files, stdin, subprocess output, LLM text).
  Parse, don't assume; return `Result`.
- Use newtypes/enums to make invalid states unrepresentable (`UnitId`, `Verdict`, `Limits`) instead of
  stringly-typed or boolean-flag APIs.
- `pub` surface minimal; `pub(crate)` by default. Public types in `verifier-core` are
  schema-stable: changing them requires a schema version decision.
- Async (`tokio`) only for I/O-bound orchestration of subprocesses; keep comparison logic
  synchronous and pure (easy to test, no I/O in `verifier-core`).
- Subprocess: always with explicit timeout, output size cap, and killed process group on timeout.
  Never build shell strings; pass argv arrays.
- Logging via `tracing` with structured fields; no `println!` in libraries.
- Derive `Debug` everywhere; `serde` types use `#[serde(deny_unknown_fields)]` for inbound messages.

**API design** (from the official Rust API Guidelines; applies to every `pub` item)
- Naming: RFC 430 casing; conversions `as_`/`to_`/`into_`; getters without `get_`; iterators
  `iter`/`iter_mut`/`into_iter`; consistent word order across names.
- Eagerly implement common traits (`Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `Default`,
  `Display` where sensible); use `From`/`AsRef` for conversions; make types `Send + Sync` where possible.
- Type safety: newtypes for static distinctions, custom enums/types instead of `bool`/`Option`
  arguments, builders for complex values, **private struct fields** (future-proofing).
- Constructors are associated functions (`new`, `with_*`, `from_*`); functions with a clear
  receiver are methods; no out-parameters.
- Validate arguments at the boundary; make error types meaningful and well-behaved
  (`std::error::Error`, `Send + Sync`, `source()` chained).
- Be generic over what you need (`impl AsRef<Path>`, `impl Read`) and let callers choose where
  data is copied/allocated; make traits object-safe if they may be `dyn`.
- Rustdoc on every public item with an example that uses `?` (not `unwrap`) and `# Errors`,
  `# Panics`, `# Safety` sections where relevant.

**Idioms**
- Prefer iterator transforms and `Option`/`Result` combinators over index loops and verbose `match`
  where they read more clearly; don't golf.
- Don't panic (see above); minimize visibility; no wildcard imports (except in tests/preludes);
  re-export dependencies whose types appear in your public API.
- Understand lifetimes rather than fighting them; avoid `unsafe`; be wary of shared-state
  parallelism — prefer message passing/ownership transfer, `Arc<Mutex<_>>` only when justified.
- Avoid the temptation to over-optimize: correctness first, then profile with the benchmark runner.
- Listen to Clippy (it is part of the gate); write more than unit tests (integration, property,
  doc tests).
- Use `Drop` for RAII cleanup (sandbox dirs, child processes) — `Drop` must never fail or block.
- Dependency hygiene: keep the graph small, watch feature creep, prefer well-maintained crates;
  `cargo deny` + `cargo audit` in CI.

**Testing**
- Unit tests next to code; integration tests in `tests/`; `proptest` for comparison/normalization logic.
- Run with `cargo nextest run`. Tests must not depend on network or the host's installed Python version
  unless gated and documented.
- Benchmarks with `criterion`, in `benches/`, never in unit tests.

## 4. Python (orchestrator under `orchestrator/`)

**Toolchain**
- `uv` for env/deps; `pyproject.toml` is the single config; Python ≥3.12.
- `ruff format` + `ruff check` (broad rule set incl. `B`, `UP`, `S`, `ASYNC`, `PTH`, `SIM`, `RUF`)
  and `pyright --strict` must be clean. CI blocks on any of them.

**Style**
- Full type annotations on everything; no untyped `dict` crossing a module boundary — use
  Pydantic v2 models (`model_config = ConfigDict(extra="forbid", frozen=True)` by default).
- **Domain logic is plain functions/classes; LangGraph only wires.** No business logic inside
  graph node definitions beyond calling a domain function and updating state.
- LLM calls only via `LlmClient`. Parse model output with a schema (structured output /
  Pydantic validation); on parse failure retry within budget, never `eval` model text.
- Prompts are files in `orchestrator/prompts/` with a version header; no prompt text inline in
  Python. Changing a prompt requires re-running the corpus eval (see §7).
- Async only where it buys concurrency (parallel units, LLM calls); don't mix sync/async in one
  module. No blocking calls in `async def`.
- Use `pathlib`, `subprocess.run([...], check=..., timeout=...)` with argv lists — never
  `shell=True`, and never run untrusted code here (rule 3).
- Logging via `structlog`; never log full prompts/source by default (size, secrets); log hashes + sizes,
  full text only to the artifact store.
- Exceptions: define a small hierarchy rooted at `MigratorError`; catch narrowly.

**Testing**
- `pytest`; use `hypothesis` for parsing/planning invariants. LLM calls are **mocked in unit
  tests** with recorded fixtures; real-model tests are marked `@pytest.mark.live` and excluded
  from default runs and CI unless explicitly enabled (cost).
- Tests don't touch the network or real repos; use fixtures in `tests/fixtures/`.

## 5. TypeScript / Next.js (`web/`, Phase 2+)

- `strict: true`, no `any` (use `unknown` + narrowing), ESLint + Prettier clean, `pnpm`.
- API client/types **generated from OpenAPI**; never hand-write request/response types.
- Server Components by default; client components only where interactivity requires it.
- Validate all external data at the boundary (e.g. `zod`); no trust in API responses either.
- Accessibility: semantic HTML, keyboard navigable review UI, sufficient contrast.
- Never render source code or LLM output with `dangerouslySetInnerHTML`; escape or use a
  safe highlighter.

## 6. Generated Rust (the product's output) — quality bar

Code the platform emits must meet these; the engine/gate enforces them where mechanical.
- Compiles with `cargo check`, passes `cargo clippy -- -D warnings` and `cargo fmt --check`.
- **Idiomatic**: ownership/borrowing over cloning everything, iterators over index loops where
  clearer, `Option`/`Result` instead of sentinel values, enums for tagged data. Not a line-by-line
  transliteration of Python.
- **`unsafe` forbidden by default.** If required, isolate in a small module with a `// SAFETY:`
  justification; its presence forces the human gate and is listed in the report.
- No `unwrap()`/`expect()` on fallible operations in library paths unless the invariant is
  documented; preserve the original program's error behavior as specified by the equivalence rules.
- Integer/float semantics chosen deliberately (see `verification.md`); document each choice
  that differs from the source language.
- Preserves public behavior and names where sensible; module layout mirrors the source structure
  unless the plan says otherwise. Doc comments carry over the original docstrings.
- Dependencies in generated crates: minimal, well-known, permissively licensed, listed in the report.
- No test-input special-casing, no shelling out to the original runtime (see `verification.md`).

## 7. Testing & quality gates

- **Every milestone has tests written for it, before implementation** (see
  [milestones.md](milestones.md)): unit, integration, acceptance, and negative tests. Test-first for
  acceptance criteria; tests ship in the same branch as the code.
- Required before merge: format, lint, typecheck, **full test suite incl. the milestone's acceptance
  tests**, schema-drift check, `cargo deny`. Red gate ⇒ no merge to `main`.
- **Corpus eval** (`just eval`): runs the pipeline over `corpus/` and reports pass rate,
  attempts, tokens, cost. Required for changes to prompts, planner, generator, or repair logic;
  results go in the PR. A regression in pass rate blocks the change.
- Bug fixes include a regression test that fails before the fix.
- A flaky test is a bug in the test or code; fix or delete with justification — never retry-until-green.

## 8. Git & PR conventions

- **Milestone branches merge to `main` only when their tests pass** — acceptance tests written
  first, full suite green, gate clean. Never edit/skip/delete a failing acceptance test to get
  green. Full workflow and per-milestone criteria: [milestones.md](milestones.md).
- Branch per change (`milestone/m<phase>.<n>-<slug>` for milestones); small focused PRs. Conventional commits (`feat:`, `fix:`, `docs:`,
  `refactor:`, `test:`, `chore:`), imperative mood, explain *why* in the body.
- Never commit: secrets, `.env`, large binaries, build output, cloned third-party repos, LLM
  response dumps containing customer code.
- Update `docs/` in the same PR when a decision, contract, or schema changes. Decisions go in the
  decision log in `tech-stack.md`.
- Never force-push shared branches; never rewrite published history.

## 9. Reliability & LLM-safety rules

**Retries, timeouts, budgets** (agent loops multiply cost and blast radius)
- Every external call (LLM, subprocess, network, DB) has an **explicit timeout**. No unbounded waits;
  human-review pauses also have a timeout/escalation.
- Retry only **idempotent** operations or those protected by an **idempotency key**. Generation
  and verification attempts are idempotent by design (attempt id + input hash); side-effecting
  steps (publishing a report, writing to a customer repo) are not retried blindly.
- Retry policy = capped **exponential backoff with jitter**, a max attempt count, *and* a max total
  retry time. Only retry transient errors (429, 5xx, timeouts), never validation/auth failures.
- Repair loops are bounded by attempts **and** token/cost budget **and** wall time; hitting
  any bound is a normal terminal state with a clear reason.
- Add a **circuit breaker** per LLM provider: after repeated failures, fail over per routing
  config or pause, instead of hammering a down service.
- Checkpoint after every pipeline step. Checkpoints store **references** (artifact ids/hashes), not
  large payloads. Resume must be deterministic: replaying a step must not repeat side effects.
- Log every retry, fallback, and budget decision to the audit log.

**LLM safety (mapped to the OWASP Top 10 for LLM Applications, 2025 edition)**
- *Prompt injection (LLM01)* and *indirect injection*: all repo content (code, comments, README,
  filenames) is hostile data. Keep it separate from instructions; never let it grant tools or
  change policy. Agents get **least-privilege tools** (*Excessive Agency, LLM06*).
- *Improper output handling (LLM05):* model output is untrusted input — schema-validate
  (Pydantic), size-cap, never `eval`/exec/shell it, never use it as a path or command without
  validation. It reaches execution only via compile + sandbox.
- *Sensitive information disclosure (LLM02):* scrub secrets/PII before prompts; honor a per-project
  "do not send" path list; log hashes not content.
- *Unbounded consumption (LLM10):* enforce the budgets above plus per-project rate limits.
- *Supply chain (LLM03):* pin model identifiers and dependencies; review generated crates'
  dependencies (they appear in the report).
- *System prompt leakage (LLM07):* assume prompts are readable; never put secrets in prompts.
- *Misinformation (LLM09):* the model's claim that code is equivalent is worthless — only the
  engine's verification result counts. Authorization and policy are enforced by code outside the model.
- Security controls are **enforced in code, outside the model**; they are never delegated to the
  model's judgment.

## 10. Security checklist for any PR

- Does any new path execute untrusted code? → through the sandbox only.
- Does any new input come from a cloned repo or LLM? → validate, size-cap, treat as hostile
  (path traversal, symlinks, zip bombs, prompt injection in source comments).
- Do file paths from repos get joined to a base? → canonicalize and verify containment.
- Could source code containing prompt-injection text change agent *tool permissions*? → agents
  get least-privilege tools; source text is data, never instructions.
- New outbound network call? → documented, configurable, absent from sandboxes.

## 11. References (researched 2026-10-21)

Primary sources used to shape these rules; re-check them when revising.

- [Rust API Guidelines & checklist](https://rust-lang.github.io/api-guidelines/checklist.html) — naming, interop, type safety, docs, future-proofing
- [Rust Design Patterns (rust-unofficial)](https://rust-unofficial.github.io/patterns/) — idioms, patterns, anti-patterns (e.g. no strategy class hierarchy; traits suffice)
- [Effective Rust](https://www.lurklurk.org/effective-rust/) — newtype, builders, generics vs trait objects, don't panic, avoid `unsafe`, listen to Clippy
- [OWASP Top 10 for LLM Applications (2025)](https://genai.owasp.org/llm-top-10/) — prompt injection, improper output handling, excessive agency, unbounded consumption
- Retry/durable-execution guidance: [AWS Durable Execution best practices](https://docs.aws.amazon.com/durable-execution/patterns/best-practices/), [Diagrid durable agent retries/timeouts](https://www.diagrid.io/faq/durable-execution/what-operational-best-practices-apply-to-retries-timeouts-and-pauses-in-durable-agent-work)

Caveats: SOLID-in-Rust and Python DI/`Protocol` guidance came from secondary sources (blogs,
tutorials), so §2's SOLID mapping is project convention informed by them, not an official
standard. Retry numbers (base delay, attempt caps) are tunable defaults, not universal rules —
put them in config. A 2026 OWASP LLM list revision was mentioned by one source; verify against
OWASP before citing category numbers beyond the 2025 edition.
