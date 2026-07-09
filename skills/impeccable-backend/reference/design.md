# design / shape — feature & system design flow

Produces a design doc + ADR stubs BEFORE code. Design is a deliverable: written, cited,
committed. A design that lives in chat scrollback didn't happen.

## When NOT to run this flow (the skip test)

Run the skip test first. All three true → no design doc; say so in one line and go build:

1. The change fits in a single file/module with no new boundary.
2. No schema change, no new datastore/queue/framework, no new external call.
3. No API contract change visible to another team or client.

A bug fix, a new column-with-default on an owned table, a new field on an existing response —
these get a PR description, not a design doc. Writing design docs for one-table features is
process-slop and erodes the team's willingness to read the real ones.

Sizing above the skip line:
- **One new table / one new endpoint on an existing service** → 1-page design: steps 1, 2, 5,
  6 only. Options section collapses to one paragraph ("considered X, rejected because Y").
- **New module, new async flow, new integration** → full flow, all 8 steps.
- **New service / new datastore / consistency-model change** → full flow + every step-8 ADR
  written before code, and the design reviewed by a human before COMMIT is treated as final.

## The flow

### 1. Context capture

- What exists: name the modules/tables/endpoints being touched, each cited `file:line`
  (or `file:symbol`). No citation → you haven't read it → stop and read it.
- Which ARCHITECTURE.md invariants apply. Quote the invariant, don't paraphrase. If the design
  will break one, flag it NOW — that's an ADR + invariant update, never a silent breach.
- Who calls this code today (one level of callers/callees is the minimum).

### 2. Constraints

Four buckets, all mandatory, one line each is fine:

- **Functional**: what must be true after the feature ships. Acceptance-level, not vision-level.
- **Scale**: requests/sec, rows, payload sizes, growth rate — with numbers. No measurement
  available → write `ASSUMPTION: ~N/day, revisit if >10×` and list it in the Assumptions
  section. "High traffic" without a number is not a constraint, it's a mood.
- **Consistency**: which reads must see which writes, and how stale is acceptable (a number:
  "≤5s", "next request", "eventual, hours OK"). Default is eventual across aggregates.
- **Topology**: who deploys this, with what cadence, alongside what. One team + one deployable
  biases hard toward a module, not a service.

### 3. Options — exactly 2–3, genuinely different

Each option: 5–10 lines describing the shape (components, data flow, where state lives, what's
sync vs async). "Genuinely different" means they differ in a boundary, a consistency model, or
a moving part — not in naming. One real option flanked by two strawmen is a sales pitch;
if you can't argue FOR each option for 2 minutes, it's a strawman — replace or delete it.

Then ONE trade-off table:

| Option | Complexity / moving parts | Failure surface | Evolution cost | Fit to stated scale |
|---|---|---|---|---|
| A — inline write | 0 new parts | txn rollback only | schema-coupled | fine to 10× |
| B — outbox + consumer | +1 relay, +1 table | relay lag, dup delivery | independent | fine to 100× |

Fill every cell with a fact or a labeled assumption, not adjectives. "Fit to stated scale"
compares to the step-2 numbers, at 10× — not 1000× (the 1000× escape hatch is one ADR
paragraph, per doctrine).

### 4. COMMIT

- Pick one. Name it. Give the 2–4 reasons that actually decided it (traceable to the table).
- **Reversal triggers**: state what observation would change the decision ("if event volume
  exceeds N/s", "if a second consumer team appears"). A decision without reversal triggers is
  dogma; with them, it's engineering.
- If the user must arbitrate (genuine tie on a product-level trade-off), present the table and
  ask ONCE. Otherwise commit yourself — options-then-indecision is worse than no options.

### 5. Failure design — MANDATORY, the doc is unfinished without it

Not an appendix. Four artifacts:

**Timeout budget table** — every network hop, caller timeout strictly < callee timeout:

| Hop | Caller timeout | Callee timeout/SLA | Budget check |
|---|---|---|---|
| API → svc | 2s | 1.5s p99 | OK |
| svc → DB | 1s | 500ms p99 | OK |

**Retry & idempotency per operation** — for each write/side-effecting op: retried? (bounded ≤3,
jittered, never on non-idempotent writes or 4xx), idempotency key (name the actual key, e.g.
`case_id + event_seq`), and dedup mechanism (unique constraint / conditional write / consumer
inbox). "We'll add idempotency later" is banned — at-least-once delivery starts on day one.

**Partial-failure semantics per fan-out** — for each loop/broadcast over N items: all-or-nothing
(and how rollback works) or best-effort (and how failures are reported to the caller — a
per-item result list, not a thrown exception swallowing item 7 of 40).

**Backpressure** — every queue/buffer/pool gets a bound (a number) and a behavior-at-bound:
shed (which caller sees what error), block (who waits, timeout?), or spill (to where, replayed
how). Unbounded = OOM with a delay; behavior-at-bound is a product decision — surface it.

### 6. Data & contract deltas

- **Schema**: every change expressed as expand → migrate → contract phases, each phase
  independently deployable and shipping with its rollback twin. No in-place renames; no drops
  the previous release still reads. New tables: state the single writing owner.
- **API**: each change classified — `additive` (free), `breaking` (new version + sunset plan),
  `behavioral` (same shape, different semantics — the sneakiest; call it out and version or
  flag it). Error envelope for new endpoints matches the repo-wide shape.

### 7. Test strategy

- **BEHAVIOUR tests**: list the 3–7 behaviours that define the feature (given/when/then, one
  line each). These test outcomes through the public surface, not internals.
- **Contract tests**: each invariant touched (step 1) and each API compatibility promise
  (step 6) gets a test that fails when the promise breaks — e.g. "consumer ignores unknown
  fields", "idempotent replay of event E produces one row".
- Name what is deliberately NOT tested and why (e.g. the vendor SDK's retry internals).

### 8. ADR extraction

Each decision that would surprise a future reader — the datastore choice, the consistency
model, the deliberate invariant change, anything with real reversal cost — becomes an ADR stub
(`docs/adr/NNNN-slug.md`, flow in `reference/adr.md`). The design doc holds the analysis; the
ADR holds the decision. Rule of thumb: if a new hire would ask "why is it like this?", it's
an ADR. If they'd never notice, it isn't.

## Design-doc template

```markdown
# Design: <feature> (<ticket>)
Status: draft | committed   Date:   Author:

## 1. Context
What exists (file:line cites). Applicable invariants (quoted).
## 2. Constraints
Functional / Scale (numbers) / Consistency (staleness numbers) / Topology.
### Assumptions
- ASSUMPTION: ... (revisit trigger)
## 3. Options
### A — <name>  (5–10 lines)
### B — <name>
Trade-off table (option · moving parts · failure surface · evolution cost · scale fit).
## 4. Decision
Chosen: <X>. Reasons. Reversal triggers.
## 5. Failure design
Timeout budget table · retry/idempotency per op · fan-out semantics · backpressure bounds.
## 6. Data & contract deltas
Expand/migrate/contract phases + rollbacks · API changes with compat class.
## 7. Test strategy
BEHAVIOUR list · contract tests per invariant · not-tested list.
## 8. ADRs
- NNNN-<slug>: <one-line decision>
```

## Anti-patterns — refuse on sight

- **Resume-driven options**: one real option plus two strawmen ("do nothing", "rewrite in
  Kafka") staged so the pre-decided answer wins. The tell: the author can't defend the losers.
- **Post-hoc design docs**: written after the code to satisfy process. They record what was
  built, not what was decided; the options table is fiction. If the code exists, write an ADR
  documenting the as-built decision honestly instead.
- **Scale claims without numbers**: "high volume", "hot path", "this table is huge" — cite a
  measurement or label it ASSUMPTION with a revisit trigger. No third state.
- **"We'll add idempotency later"**: later is after the first duplicate payment. The queue is
  at-least-once today; the design answers it today.
- **Option counts outside 2–3**: one option is a sales pitch; five is indecision outsourced
  to the reader.
- **Failure design deferred to implementation**: timeouts and retry policy chosen ad hoc in
  code review are chosen by whoever is tired that day. They're design decisions — table them
  in step 5.
