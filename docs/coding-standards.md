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

## 2. Rust (verification engine and everything under `crates/`)

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

**Testing**
- Unit tests next to code; integration tests in `tests/`; `proptest` for comparison/normalization logic.
- Run with `cargo nextest run`. Tests must not depend on network or the host's installed Python version
  unless gated and documented.
- Benchmarks with `criterion`, in `benches/`, never in unit tests.

## 3. Python (orchestrator under `orchestrator/`)

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
  Python. Changing a prompt requires re-running the corpus eval (see §5).
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

## 4. TypeScript / Next.js (`web/`, Phase 2+)

- `strict: true`, no `any` (use `unknown` + narrowing), ESLint + Prettier clean, `pnpm`.
- API client/types **generated from OpenAPI**; never hand-write request/response types.
- Server Components by default; client components only where interactivity requires it.
- Validate all external data at the boundary (e.g. `zod`); no trust in API responses either.
- Accessibility: semantic HTML, keyboard navigable review UI, sufficient contrast.
- Never render source code or LLM output with `dangerouslySetInnerHTML`; escape or use a
  safe highlighter.

## 5. Generated Rust (the product's output) — quality bar

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

## 6. Testing & quality gates

- Required before merge: format, lint, typecheck, unit tests, schema-drift check, `cargo deny`.
- **Corpus eval** (`just eval`): runs the pipeline over `corpus/` and reports pass rate,
  attempts, tokens, cost. Required for changes to prompts, planner, generator, or repair logic;
  results go in the PR. A regression in pass rate blocks the change.
- Bug fixes include a regression test that fails before the fix.
- A flaky test is a bug in the test or code; fix or delete with justification — never retry-until-green.

## 7. Git & PR conventions

- Branch per change; small focused PRs. Conventional commits (`feat:`, `fix:`, `docs:`,
  `refactor:`, `test:`, `chore:`), imperative mood, explain *why* in the body.
- Never commit: secrets, `.env`, large binaries, build output, cloned third-party repos, LLM
  response dumps containing customer code.
- Update `docs/` in the same PR when a decision, contract, or schema changes. Decisions go in the
  decision log in `tech-stack.md`.
- Never force-push shared branches; never rewrite published history.

## 8. Security checklist for any PR

- Does any new path execute untrusted code? → through the sandbox only.
- Does any new input come from a cloned repo or LLM? → validate, size-cap, treat as hostile
  (path traversal, symlinks, zip bombs, prompt injection in source comments).
- Do file paths from repos get joined to a base? → canonicalize and verify containment.
- Could source code containing prompt-injection text change agent *tool permissions*? → agents
  get least-privilege tools; source text is data, never instructions.
- New outbound network call? → documented, configurable, absent from sandboxes.
