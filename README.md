# sneakoscope 🌀

**Artifact-quality doctrine for AI-assisted engineering.** Process skills (superpowers,
brainstorming, TDD flows, subagent-driven development) orchestrate *how* work happens.
Shipshape guards *what gets produced*: the code, the architecture decisions, the SQL, the
pipeline configs, the shell scripts, and the tests — the artifacts that ship, and where
AI-generated slop actually lands.

Inspired by [pbakaus/impeccable](https://github.com/pbakaus/impeccable)'s approach — dense,
concrete, checkable doctrine instead of vague "best practices" — and vendoring it unmodified as
the frontend design register (Apache-2.0, see NOTICE). Everything else is original.

## The name

In the Harry Potter books, a **Sneakoscope** is a Dark Detector — a small glass spinning top
that **lights up, spins, and whistles when someone untrustworthy is near**. Ron sends Harry a
pocket one in *Prisoner of Azkaban*; Mad-Eye Moody keeps Dark Detectors like it in his office;
in *Deathly Hallows* the trio keep one on the tent table, waiting for it to spin. It doesn't
care how convincing the person seems — it reacts to *deception itself*.

That is exactly the job here. AI-generated code is the most convincing liar in your codebase:
it compiles, it's idiomatic, the tests are green — and the test is a tautology, the `.get` is a
production NPE, the query is an OFFSET-10k table scan, and the pipeline config has credentials
in it. A green build is a charming stranger. This toolkit is the thing on the desk that
**whistles anyway**.

(One canon caveat we ignore with pride: cheap Sneakoscopes were notoriously twitchy and went
off constantly. Ours runs cite-or-drop — `file:line` evidence or silence — with explicit
"don't flag when" rules, so when it whistles, it means something.)

Not affiliated with or endorsed by J.K. Rowling, Warner Bros., or Wizarding World — the name is
a fan's homage.

## Skills

| Register | Skill | Guards |
|---|---|---|
| Frontend design | `impeccable` (vendored) | visual/UX design language |
| Architecture | `sneakoscope-architecture` | design-before-code: `init` (generate the repo's ARCHITECTURE.md + ENGINEERING.md), `design` (options → trade-off table → commit → mandatory failure design), `adr`, `critique` (8 scored dimensions), `model`, `api`, `harden` (five questions per hop), `evolve` (no-big-bang migrations). 8 doctrine chapters |
| Code | `scala-code-optimizer` · `python-code-optimizer` · `js-code-optimizer` | per-language idioms, FP hygiene, footguns, perf, N+1, DRY/pattern-fit. Suggest-only, cite-or-drop |
| Pipelines | `sneakoscope-pipelines` | Spark (shuffle/skew/broadcast/serialization/caseSensitive traps), Flink (state/watermarks/exactly-once reality), DAG orchestration (wiring, creds-in-config, idempotent re-run, backfill) |
| Data stores | `sneakoscope-stores` | SQL query-text doctrine (sargability, OFFSET, injection, NULL logic) + engine chapters: MySQL/MariaDB, Cassandra/ScyllaDB, StarRocks/Hive/Trino, Elasticsearch |
| Ops scripts | `sneakoscope-scripts` | bash strict mode, quoting, idempotency, dry-run, traps, secrets, kubectl/helm hygiene |
| Tests | `test-quality-audit` | test honesty: BEHAVIOUR / IMPLEMENTATION / TAUTOLOGY / DEAD verdicts + the falsifiability-mutation check — "100% tested" means behaviours pinned, not lines executed |
| The gate | `pr-gate` | one review gate for your own diff pre-PR or a teammate's PR: routes changed artifacts through every lens above + SOLID-principles sweep + repo conventions → **SHIP / FIX-FIRST / BLOCKED** |

Common contract across all skills: **suggest-only** (never edit), **cite or drop**
(`file:line` evidence, no "likely/appears"), concrete rules with *tells* over principles-prose,
and a "don't flag when" column so doctrine doesn't become zealotry.

## Install

```bash
claude plugin marketplace add ganesh-tt/sneakoscope
claude plugin install sneakoscope@sneakoscope
```

Or vendor individual skills into a repo's `.claude/skills/`.

## The loop (per repo)

1. **Once**: `/sneakoscope-architecture init` → generates the repo's architecture brain
   (`ARCHITECTURE.md` capture + invariants, `ENGINEERING.md` quality bar).
2. **Dev-start**: repo `CLAUDE.md` points at those docs; design work runs
   `/sneakoscope-architecture design` → design doc + ADRs *before* code.
3. **Dev-done**: `/pr-gate` on your branch before opening the PR.
4. **Review**: `/pr-gate <PR#>` on a teammate's PR — same lenses, same standards, and the repo's
   own baseline docs outrank the generic defaults.

Design-time and review-time enforce the *same* doctrine: `sneakoscope-architecture` exists so
that `pr-gate` finds nothing.

## Where this fits with process skills

Use both. Brainstorm/spec/TDD/subagent workflows decide *what to build and in what order*;
their gap is that no step judges the generated artifact against engineering doctrine — a
subagent can complete every process step and still hand back an unsafe `.get`, an
OFFSET-pagination query, a tautological test, and a shell script that rm's unquoted paths.
Shipshape is that judgment layer: wire `pr-gate` at the end of the process (and optionally as
a `gh pr create` blocking hook — documented in `skills/pr-gate/`, opt-in, never silently wired).

## Origin

Built after a release shipped a large bug-tail despite a fully green test suite — audits traced
it to unsafe partial functions, weak FP/effect discipline, schema drift, and tautological tests
that could not fail. These skills are the standards + review loop that came out of it.
