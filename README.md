# impeccable-stack

Impeccable for the **whole stack** — two registers, one install:

| Register | Skill | What it enforces |
|---|---|---|
| **Design (frontend)** | `impeccable` | The design language for FE/UX work — vendored from [pbakaus/impeccable](https://github.com/pbakaus/impeccable) (Apache-2.0, see NOTICE) |
| **Design (backend)** | `impeccable-backend` | The architecture design language — its true backend counterpart. Sub-commands: `init` (generate the repo's ARCHITECTURE.md + ENGINEERING.md), `design` (options → trade-off table → commit → mandatory failure design → ADR extraction), `adr` (MADR-lite lifecycle), `critique` (8 scored dimensions), `model`, `api`, `harden` (five questions per hop), `evolve` (no-big-bang migration doctrine). 8 doctrine chapters, ~1,500 lines |
| **Engineering** | `scala-code-optimizer` | Scala idioms, FP hygiene (Option/Try/Either, no `.get`/`null`), effect discipline, ScalikeJDBC pitfalls, Spark safety, perf, N+1, DRY/pattern-fit |
| | `python-code-optimizer` | PEP idioms, FP hygiene, typing, footguns (mutable defaults, bare except), perf, N+1, DRY |
| | `js-code-optimizer` | Modern ES/TS idioms, FP + async/Promise hygiene, TS type-safety, perf, N+1, DRY |
| | `test-quality-audit` | Test honesty: BEHAVIOUR / IMPLEMENTATION / TAUTOLOGY / DEAD verdicts + falsifiability-mutation check — so "100% tested" means behaviours pinned, not lines executed |
| | `pr-gate` | The orchestrator: one review gate for your own diff pre-PR or a teammate's PR. Routes changed files through correctness + per-language optimizer + anti-slop + test-quality + UI/UX + SOLID-principles lenses → **SHIP / FIX-FIRST / BLOCKED** |

All engineering skills are **suggest-only** (never edit files), require `file:line` evidence
(cite-or-drop, no "likely/appears"), and classify perf impact
(`equivalent`/`improved`/`needs-benchmark`).

## Install

```bash
claude plugin marketplace add ganesh-tt/impeccable-stack
claude plugin install impeccable-stack@impeccable-stack
```

Or vendor individual skills into a repo's `.claude/skills/`.

## The intended loop (per repo)

1. **Dev-start** — repo `CLAUDE.md` points at the repo's baseline docs
   (`ENGINEERING.md` strategy, `ARCHITECTURE.md` system capture + invariants, per-language
   conventions). `pr-gate` reads these as the review standard when present
   (`.impeccable/code/*`) — repo baseline outranks generic defaults.
2. **Dev-done** — `/pr-gate` on your branch before `gh pr create`.
3. **Review** — `/pr-gate <PR#>` on a teammate's PR. Same lenses, same standards.

Cross-language principles checklist (SOLID, DRY, KISS/YAGNI, layering, cohesion/coupling,
fail-fast, encapsulation, least-astonishment, idempotency — each mapped to *detectable* smells
with "don't flag when" columns): `skills/pr-gate/references/principles.md`.

Optional blocking enforcement (a PreToolUse hook that gates `gh pr create` on a fresh
`/pr-gate` run + deterministic secret/title/tests checks) is documented inside
`skills/pr-gate/` consumers' repos as a proposal — enforcement is opt-in per developer or CI,
never silently wired.

## Origin

Built after a release shipped a large bug-tail despite a fully green test suite — audits traced
it to unsafe partial functions, weak FP/effect discipline, schema drift, and tautological tests
that could not fail. These skills are the standards + review loop that came out of it.
