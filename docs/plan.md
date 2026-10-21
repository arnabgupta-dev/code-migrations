# Plan: AI-Assisted Code Migration Platform

Source: `AI_Code_Migration_Platform_Plan.pdf` (kept in repo root). This file is the working
distillation; if the two disagree, discuss and update this file deliberately.

## 1. What we are building

A platform that takes existing source code (Python first, later TypeScript/Node, C) and produces
**clean, idiomatic Rust that is verified to behave the same**, plus evidence:
equivalence report, performance deltas, and a list of remaining risks.

The value is **not** "AI rewrites code". The value is the **proof**. Companies pay only if the
equivalence evidence is convincing. Everything in the design serves that.

## 2. Pipeline (every migration runs this)

```
Source ─▶ 1 Analyze ─▶ 2 Plan ─▶ 3 Generate ─▶ 4 Verify ─▶ pass? ─yes─▶ 6 Human Gate ─▶ 7 Output
                                     ▲            │ no
                                     └─ 5 Repair ◀┘  (up to N attempts, then escalate to human)
```

| # | Step | What happens |
|---|------|--------------|
| 1 | Ingest & Analyze | Clone repo, parse, build dependency graph, find hot paths and module boundaries, infer intent |
| 2 | Plan | AI proposes migration order, independent modules, type/layout strategy, risk assessment |
| 3 | Generate | Agents write Rust candidates per module using plan + context |
| 4 | Verify | `cargo check`/compile → differential tests (same inputs ⇒ same outputs) → unit/integration tests → benchmarks |
| 5 | Repair | Failure diagnostics go back to the agent; retry up to a limit |
| 6 | Human Gate | Critical modules and low-confidence results require human review/approval |
| 7 | Output | Rust project (`rust/`), `report.html`, `bench_results.md`, remaining-risk list |

## 3. Layers (see `architecture.md`)

Interface (CLI + Web) → Orchestration (Python) → Verification Engine (Rust) → AI Model Layer →
Storage & Reporting. The Rust verification engine is the **trust engine**; the Python layer is
untrusted-by-design (it is AI-driven) and must never be able to "declare" a pass itself.

## 4. Roadmap

### Phase 0 — Foundation (4–6 wks)
Prove the core verification loop on small examples.
- Working differential tester (Python vs Rust on simple functions)
- Basic agent that rewrites one function and is repaired until tests pass
- **Success:** 10/10 small modules migrate with zero behavioral difference
- **Deliverable:** side-by-side demo with proof of equivalence

### Phase 1 — MVP CLI (8–12 wks)
Single pair: **Python → Rust**, for technical users.
- CLI accepting a folder or Git repo; full pipeline for modules under ~500 lines
- HTML/Markdown report with equivalence results and performance numbers
- Open-source the verification core
- **Success:** 5–10 external developers migrate a real small service

### Phase 2 — Web Platform Beta (12–16 wks)
- Web dashboard: Git connect, live progress, review UI
- Multi-module support with dependency ordering
- Human review gates + comments
- Basic auth + project history
- **Success:** first paid pilot customers

### Phase 3 — Production hardening & expansion (ongoing)
- Node/TypeScript → Rust
- Incremental migration for large codebases
- Private deployment (VPC / on-prem)
- Audit logs, SLAs, team roles
- **Success:** repeatable revenue from mid-market/enterprise

Each phase is broken into milestones with their own acceptance tests, and **a milestone merges to
`main` only when its tests pass** — see `milestones.md`.

## 5. Immediate next actions (weeks 1–2)

1. ✅ First language pair: **Python → Rust** (decided, see `tech-stack.md`)
2. Build the smallest verification loop: one function in, one function out, prove they match
3. ✅ Open-source strategy: open verifier core, keep orchestration/web proprietary (recommended, **needs owner confirmation**)
4. Identify 5–10 design partners (teams with Python/Node performance pain)

## 6. Key risks

| Risk | Why it matters | Mitigation direction |
|------|----------------|----------------------|
| Verification quality | Make-or-break feature | Invest in the Rust engine first; adversarial test corpus; see `verification.md` |
| Agent reliability on real code | Toy examples ≠ real repos | Start with <500-line modules; strict module boundaries; human gates |
| AI token cost at scale | Repair loops multiply cost | Per-run budget caps, model routing by cost/quality, cache analysis results |
| Trust | No convincing proof ⇒ no customers | Report states exactly what was and wasn't proven; never overclaim |

## 7. Notes on the PDF

- The architecture diagram lists specific models (GPT-4o, Claude 3.5, Gemini 1.5, DeepSeek Coder V2).
  Treat these as illustrative only; model names live in config, never in code (see `tech-stack.md`).
- The PDF offers "Firecracker or WASM" for sandboxes. We chose a phased approach; see `tech-stack.md`.
