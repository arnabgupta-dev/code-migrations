# Verification & Equivalence Spec

The plan calls verification quality "the make-or-break feature". This doc defines what we claim
and how. **Never claim more than we proved.**

## What "equivalent" means

For a migration unit with entrypoints `E`, the Rust candidate is **observationally equivalent
on test set T** if for every input `i ∈ T`, original and candidate agree on:

1. **Return value** (structurally compared, see normalization)
2. **Raised error** — same error *category* (mapped, e.g. `ValueError → Err(ValueError-equivalent)`); message text compared only if marked significant
3. **stdout / stderr** (byte-exact unless a normalizer is configured)
4. **Exit code** (for program entrypoints)
5. **Declared side effects** (files written, env, network calls recorded via stubs)

Equivalence is **empirical, not formal**: it is evidence over T. Reports must say so and show
coverage. Formal proof is out of scope until explicitly planned.

## Comparison rules

- Compare **structured values**, not strings of printed reprs. Common representation: a tagged
  JSON value (`int`, `float`, `str`, `bytes`, `list`, `dict`, `none`, `bool`, plus `set`/`tuple`).
- **Integers:** Python ints are arbitrary precision; Rust fixed width. Candidates must use a type
  chosen by the planner; the tester includes boundary values (min/max/overflow) to expose mismatch.
- **Floats:** bit-exact by default (including NaN/±0/inf handling). Tolerance only via an explicit,
  per-unit, human-approved setting that is shown in the report.
- **Strings:** compare as Unicode scalar sequences; include non-ASCII, empty, and surrogate-ish cases.
- **Dict/set ordering:** order significant only where the source semantics make it so
  (Python dicts are insertion-ordered; sets are not).
- **Nondeterminism** (time, randomness, hash ordering, threads): must be stubbed/seeded via the
  harness or the unit is marked `non-deterministic` and **cannot** be auto-approved.

## Input sources (in priority order)

1. Existing project tests, adapted to run against both implementations
2. Recorded traces from running the original (real inputs)
3. Hypothesis-generated inputs from type hints/signatures — **seeded; seed and generated cases
   are saved as artifacts** so any failure is reproducible
4. Boundary/adversarial corpus (`corpus/adversarial/`): empty, huge, unicode, negative, NaN,
   deep recursion, large collections

Coverage of the **original** code during these runs is measured and reported. Unexecuted
branches are listed as "unverified" in the report.

## Result object (conceptual)

`VerificationResult { unit, attempt, input_hash{source,candidate,cases,toolchain}, compile{ok,diagnostics},
cases{total,passed,failed,mismatches[minimal_repro]}, coverage, bench{...}, verdict, limitations[] }`

`verdict ∈ {verified, failed_compile, failed_equivalence, failed_timeout, inconclusive}`.
`inconclusive` is a real outcome (e.g. non-deterministic, insufficient coverage) and routes to
the human gate. There is no verdict "probably fine".

## Anti-cheating rules (the repair loop will try shortcuts)

- The agent **cannot modify** the test cases, the comparison config, or the harness. They are
  read-only inputs to the engine, hashed into the result.
- Generated Rust is checked for **special-casing test inputs** (hardcoded outputs, input
  lookup tables) via a static check + review of any large literal tables; held-out cases not
  shown to the agent are part of every final verification.
- Weakening tolerances or normalizers requires human approval and is recorded in the audit log.
- A candidate that shells out to Python/Node/C to "compute" the answer fails verification.
  The sandbox for the candidate has no interpreter of the original language available.

## Benchmarks

- Same workload and inputs for original and candidate; separate processes; warmup; ≥N repeats.
- Report median, p95, variance, and environment (CPU, OS, toolchain versions, build profile).
- Rust measured with `--release`; Python measured in its normal production configuration.
- Never report a speedup from a failed or inconclusive unit.
- Benchmarks run after correctness passes, never as a substitute for it.

## Trust & reproducibility

- Every verification is **reproducible**: given the saved artifacts (source, candidate, cases,
  seeds, toolchain versions), re-running yields the same verdict.
- Toolchain versions (rustc, CPython/Node/gcc) are recorded in the result.
- The report lists: what was tested, what was not, assumptions, tolerances used, and unsafe
  blocks (with justification) in the candidate.

## Phase 0 acceptance (concrete)

10 small Python functions in `corpus/` with hand-verified Rust equivalents → engine reports
`verified` for all; plus ≥10 deliberately wrong Rust variants (off-by-one, overflow, float
rounding, wrong error type, ordering) → engine reports a failure for **every one**. The
negative corpus matters as much as the positive one.
