# Milestones & Merge Gates

Rule: **a milestone branch merges to `main` only if its tests pass.** `main` is always green and
always releasable. No exceptions, no "merge now, fix tests later".

## Workflow (every milestone)

1. **Define acceptance tests first.** Before writing implementation, write the milestone's
   acceptance tests (they should fail) and commit them as the first commit on the branch.
   They encode the milestone's "done" definition from the list below.
2. **Branch:** `milestone/m<phase>.<n>-<slug>` from an up-to-date `main`
   (e.g. `milestone/m0.2-bwrap-sandbox`). Work in small commits.
3. **Every feature/fix inside the milestone ships its own tests** (unit + integration; negative
   cases for anything verification-related) — see `coding-standards.md` §7.
4. **Gate** (run `just milestone-check <id>` locally; CI runs the same on the PR):
   - format, lint, typecheck (both languages) clean
   - the **entire** test suite passes (not just this milestone's) — no regressions
   - this milestone's **acceptance tests** pass
   - schema-drift check and `cargo deny` pass
   - corpus eval shows no pass-rate regression (once the corpus exists)
   - docs updated for any changed decision/contract/schema
5. **Merge via PR only after the gate is green.** Squash or merge commit; tag the merge `m<phase>.<n>`.
   Delete the branch.
6. **If the gate is red:** fix the code. Do **not** edit, weaken, skip, or delete acceptance
   tests to get green. Changing an acceptance test requires an explicit human decision with the
   reason recorded in the PR and the decision log.
7. Direct pushes to `main` are not allowed once the repo has CI. Configure GitHub branch protection:
   require PR, require the CI check to pass, require branch up to date, no force-push
   (owner action; needs repo admin).

## Test classes per milestone

| Class | Purpose | Gate? |
|-------|---------|-------|
| Unit | Logic in isolation | Yes |
| Integration | Real components together (engine + sandbox + toolchain) | Yes |
| **Acceptance** | The milestone's "done" criteria, listed below | **Yes — decides merge** |
| Negative / adversarial | Wrong inputs/candidates must be *rejected* | Yes |
| Property-based | Invariants (proptest / Hypothesis) | Yes |
| Live-LLM eval | Real model quality/cost on the corpus | **Reported, not a hard gate** (non-deterministic, costs money); the gate uses recorded LLM fixtures. Results are pasted in the PR |

Acceptance tests must be deterministic and runnable offline. Anything needing a real model uses
recorded fixtures for the gate.

## Phase 0 — Foundation

Goal: prove the verification loop on small Python → Rust functions.

| ID | Milestone | Acceptance tests (merge criteria) |
|----|-----------|-----------------------------------|
| M0.1 | Workspace, contracts, CI skeleton | `cargo`/`uv` projects build; `just check`/`test` run; JSON schemas generated from Rust and Python models validate against them; schema-drift check fails on a deliberate change; CI runs green |
| M0.2 | Sandbox (bwrap) | Runs a command with CPU/mem/time limits; **network blocked**; FS read-only outside scratch; timeout kills the whole process group; output size cap enforced; cleanup guaranteed on success, failure, and timeout; shared `Sandbox` contract tests pass |
| M0.3 | Comparator (`verifier-core`) | Structured value comparison per `verification.md`: ints (incl. boundaries), floats (NaN/±0/inf bit-exact), strings (unicode), ordering rules, error-category mapping; proptest invariants (reflexive, symmetric where applicable); every mismatch kind has a negative test |
| M0.4 | Differential tester end-to-end | Python function vs hand-written Rust on fixed inputs → `verified`; deliberately wrong Rust → `failed_equivalence` with minimal repro; compile failure → `failed_compile` with structured diagnostics; result includes input hashes + toolchain versions; re-run reproduces the same verdict |
| M0.5 | Corpus & negative corpus | **10/10** correct small modules report `verified`; **≥10/10** deliberately wrong variants (off-by-one, overflow, float rounding, wrong error type, ordering, hardcoded outputs) are all rejected; held-out cases enforced |
| M0.6 | Generate + repair loop | With recorded LLM fixtures: agent output compiles, fails, gets structured diagnostics, repairs, and passes; loop stops at attempt/budget limit with the right terminal state; agent cannot modify tests/harness (attempt to do so is rejected); orchestrator cannot construct a `VerificationResult` |
| M0.7 | Phase 0 demo & report | End-to-end CLI run on the corpus produces side-by-side demo + `report.json`/Markdown with equivalence evidence and limitations; live-LLM eval results recorded (reported); Phase 0 success metric (10/10, zero behavioral difference) met |

## Phase 1 — MVP CLI (detail these milestones when Phase 0 merges)

| ID | Milestone (draft) | Acceptance tests (draft) |
|----|-------------------|--------------------------|
| M1.1 | Repo ingest + analysis (tree-sitter, dependency graph) | Fixture repos analyzed to expected inventory/graph; hostile repo fixtures (symlinks, path traversal, huge files, prompt-injection text) handled safely |
| M1.2 | Planner (module ordering, risk, model tier) | Deterministic plan for fixture repos; dependency order respected; critical modules flagged for human gate |
| M1.3 | Multi-unit pipeline + state/checkpoint/resume | Kill mid-run → resume without repeating completed steps or side effects; audit log complete and append-only |
| M1.4 | Human gate (CLI review flow) | Gate triggers per rules in `architecture.md`; approve/reject/feedback paths; timeout escalation |
| M1.5 | Benchmarks + HTML/Markdown reports | Bench methodology per `verification.md`; no speedups reported for non-verified units; report lists unverified areas |
| M1.6 | CLI packaging + folder/Git input | Install in clean env; run on a real small service (<~500-line modules); budgets enforced |
| M1.7 | Open-source readiness for verifier core | License, docs, examples; `crates/` builds standalone; no secrets/customer code in history |

## Phase 2 / 3

Break into milestones with their own acceptance tests when the phase starts (web dashboard,
auth, Postgres/queue, review UI; then TS→Rust, incremental migration, private deployment,
audit/roles). Same rules apply.

## Definition of done for a milestone

Branch merged to `main` through a green gate, tagged, docs updated, acceptance tests live in the
repo as permanent regression tests, and the roadmap in `plan.md` reflects the new state.
