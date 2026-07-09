---
name: sneakoscope-architecture
description: Use when the user wants to design, architect, shape, critique, audit, harden, model, evolve, or document a backend system, service, API, data model, pipeline, or integration — BEFORE or while writing code. Covers architecture design, service/module boundaries, ADRs (architecture decision records), API contract design, data modeling, schema evolution, consistency and transactions, failure design (timeouts, retries, idempotency, backpressure), migration/evolution planning, and architecture review. Trigger on "design this feature/service", "how should we architect X", "write an ADR", "review this design", "model this data", "design the API", "is this architecture sound", "plan the migration", or any feature work that deserves a design before code. The backend-architecture counterpart of a frontend design language — inspired by pbakaus/impeccable's doctrine approach, original content. Not for reviewing already-written code diffs (use pr-gate / the code optimizers for that).
version: 1.0.0
user-invocable: true
argument-hint: "[init · design|shape · adr · critique|audit · model · api · harden · evolve] [target]"
license: Apache 2.0
---

Designs production-grade backend systems: committed decisions, explicit trade-offs, failure-first
thinking, decisions recorded as ADRs. Design is a deliverable, not a preamble to code.

## Setup — you MUST do these steps before proceeding

1. **Load the project's architecture brain.** Look for `ARCHITECTURE.md` (system capture +
   invariants) and `ENGINEERING.md` (quality bar) at the repo root or under `.sneakoscope/code/` (legacy: `.impeccable/code/`),
   and an ADR directory (`docs/adr/`, `adr/`, `docs/decisions/`). If ARCHITECTURE.md is missing
   and the user invoked `init`, `design`, or `shape` — divert into `reference/init.md` first:
   captured system context is the point of those flows. For any other scoped command, proceed
   with what the code shows and offer `init` once as a suggestion. A missing ARCHITECTURE.md
   must never block a scoped request.
2. **If the user invoked a sub-command, read `reference/<command>.md` next. Non-optional.**
   The reference defines the flow; without it you will skip steps the user expects.
   - `init` → `reference/init.md` — capture the system: generate ARCHITECTURE.md + ENGINEERING.md
   - `design` / `shape` → `reference/design.md` — feature/system design flow → design doc + ADR
   - `adr` → `reference/adr.md` — record a decision properly
   - `critique` / `audit` → `reference/critique.md` — scored architecture review
   - `model` → `reference/data.md` — data modeling & schema evolution
   - `api` → `reference/api.md` — contract design
   - `harden` → `reference/failure.md` — failure-mode design pass over an existing design
   - `evolve` → `reference/evolution.md` — migration / strangler / versioning strategy
3. **Read the actual system before designing.** At minimum: the module you're touching, its
   immediate callers/callees, the schema it owns, and one representative test. Designs written
   from folder names are fiction.
4. **Pick the register.** Product service (correctness + evolution dominate) vs data/pipeline
   (throughput + reprocessability dominate) vs library/platform (API stability dominates).
   Say which one you picked; it weights every trade-off below.

## Design doctrine — general rules

These are the backend equivalent of contrast ratios: concrete, checkable, non-negotiable
unless the user explicitly overrides.

#### Boundaries
- A module boundary is real only if it owns its data. Two "services" writing the same tables
  are one service with extra steps.
- Dependency direction: domain logic never imports infrastructure. Infra depends on domain,
  not the reverse. One inversion (an interface/port) is cheaper than a lifetime of mock-hell.
- Synchronous call chains: depth ≤3 across module/service boundaries per request. Deeper means
  you've built a distributed monolith; the 4th hop's latency and failure rate are now your API's.
- Shared libraries may share *mechanisms* (serialization, retry). Shared *domain* types across
  service boundaries couple deploys — copy the DTO instead (rule-of-three before extracting).

#### State & data
- Every table/collection has exactly one writing owner. Multi-writer tables need a documented
  arbitration rule (last-write-wins is a decision, not a default).
- One aggregate per transaction. Cross-aggregate consistency is eventual by default; if the user
  needs it atomic, that's a boundary-drawing error — merge the aggregates or use a saga/outbox
  and say so in the ADR.
- No distributed transactions. Cross-service writes = outbox + idempotent consumer, or a
  documented compensation path. 2PC is not on the menu.
- Schema evolution is expand → migrate → contract. Never rename in place; never drop a column
  the previous release still reads. Every migration ships with its rollback twin.
- Soft-delete, audit, and multi-tenancy are day-one columns, not retrofits — they change every
  index and every query shape.

#### Failure (design for it first, not last)
- Every network call has a timeout, and caller timeout < callee timeout (else callers give up
  while callees keep working — the work is wasted twice).
- Every retried operation is idempotent, keyed by an explicit idempotency key — at-least-once
  delivery is the default reality of every queue you will ever use.
- Retries: bounded (≤3), jittered exponential backoff, and NEVER on non-idempotent writes or
  4xx-class errors. Unbounded retry is a self-inflicted DDoS.
- Backpressure over buffering: an unbounded in-memory queue is an OOM with a delay. Bound it
  and define what happens at the bound (shed, block, spill) — that's a product decision,
  surface it.
- Partial failure is the interface: every fan-out defines per-item failure semantics
  (all-or-nothing vs best-effort + report). "It throws" is not semantics.
- Fallbacks must be *louder* than the primary path, never silent: a defaulted value on parse
  failure without a log line is how silent-corruption bugs ship.

#### API contracts
- Contract-first for anything another team consumes. The contract includes the error shape —
  one machine-readable error envelope (code, message, correlation id) across the whole surface.
- Compatibility rule: adding fields is free; removing/renaming/retyping is a new version.
  Consumers must ignore unknown fields (tolerant reader) — write it into the contract.
- Pagination: cursor-based for anything unbounded or user-scrollable; offset pagination past
  ~10k rows is a database incident scheduled for later.
- Every list endpoint is bounded (default + max page size). Every mutation returns enough to
  avoid an immediate re-fetch.
- Version in the path or header, pick one repo-wide; deprecations get a sunset date and a
  usage-measured removal, not a hope.

#### Observability & operations
- A request is traceable end-to-end from one correlation id, logged at every boundary crossing.
- Every async flow answers: "how do we know it's stuck?" (lag metric, DLQ depth, oldest-message
  age) — before it ships, not after the first silent backlog.
- Config over code for operational knobs (timeouts, limits, flags); code over config for domain
  logic. A YAML file encoding business rules is a programming language without tests.

#### Simplicity (anti-slop, load-bearing)
- The best architecture is the one with the fewest moving parts that meets the *stated* load.
  Design for 10× current scale, not 1000×; write the 1000× escape hatch as one ADR paragraph,
  not as infrastructure.
- No new datastore/queue/framework when an existing one covers the need within measured limits.
  Every new moving part is an on-call page.
- A queue between two components you own and deploy together is usually a function call wearing
  a costume. Justify the decoupling with a real independent-scaling or failure-isolation need.
- Microservices are an org-chart tool. Under ~3 teams, prefer a well-moduled monolith with
  enforced boundaries (the module split IS the future service split).

## Artifacts — design is written down or it didn't happen

| Flow | Produces |
|---|---|
| `init` | `ARCHITECTURE.md` (capture + invariants) + `ENGINEERING.md` (quality bar) |
| `design`/`shape` | a design doc (context → constraints → options → decision → failure design → test strategy) + an ADR for each decision that would surprise a future reader |
| `adr` | `docs/adr/NNNN-<slug>.md` (MADR-lite, see reference/adr.md) |
| `critique` | scored review (dimensions /20) with cited evidence, ranked findings, no edits |
| `evolve` | migration plan with expand/contract phases, rollback points, and a "how we know each phase worked" check |

## Rules of engagement

- **Options before commitment**: a real design presents 2–3 genuinely different options with a
  trade-off table, then COMMITS to one with reasons. One option is a sales pitch; five is
  indecision.
- **Failure design is part of the design**, not an appendix. A design doc without timeout,
  retry, idempotency, and partial-failure answers is unfinished.
- **Evidence discipline**: claims about the existing system cite `file:line`. Claims about load
  ("this table is hot") cite a measurement or are labeled assumption — assumptions get listed
  in the design doc's own section.
- **Preserve invariants**: read ARCHITECTURE.md's invariants list; a design that breaks one
  must say so explicitly and update the invariant via ADR, never silently.
- **Critique never edits.** Design flows produce docs; only the user-approved implementation
  step touches code.
- **The reviewer's counterpart**: pr-gate reviews the code diff against these same doctrines
  post-hoc. This skill exists so that pr-gate finds nothing — the decisions were made, recorded,
  and followed before the code existed.
