# adr — record an architecture decision

An ADR captures ONE decision, its forces, and its costs — so a future reader (including a
future you, or an agent with no chat history) doesn't re-litigate it or silently reverse it.
The design doc holds analysis; the ADR holds the decision. Keep them separate.

## The flow

1. Confirm the trigger (table below). Not triggered → say so in one line, record the choice
   in the PR description instead, stop.
2. Locate the ADR directory (`docs/adr/`, `adr/`, `docs/decisions/` — whichever exists;
   create `docs/adr/` if none) and read the last 2–3 ADRs for house style and next number.
3. Draft with the MADR-lite template below, Status `proposed`. Cite the system as it is
   (`file:line`), not as remembered.
4. Show the draft to the user; acceptance follows the lifecycle rules — an agent proposes,
   a human (or the stated team convention) accepts.

## When an ADR is REQUIRED

Any one of these triggers it:

- **New boundary**: a new service, module split, or ownership line for data or a domain.
- **New moving part**: datastore, queue, cache, framework, external vendor — every new part is
  an on-call page; the decision to accept that page gets recorded.
- **Consistency model choice**: eventual vs strong for a flow, saga vs merged aggregate,
  outbox vs synchronous dual-write, arbitration rule for a multi-writer table.
- **Breaking an ARCHITECTURE.md invariant**: never silent. The ADR states the old invariant,
  the new one, and why — and ARCHITECTURE.md is updated citing the ADR.
- **Irreversible or expensive-to-reverse choice**: ID formats, partition/shard keys, event
  schemas, wire-format versions, tenancy model. Anything where "undo" means a migration.
- **Cross-team contract**: an API/event shape another team builds against, including its
  error envelope and compatibility promise.

## When an ADR is NOT required

Decisions cheap to reverse don't need ceremony. Library-internal structure, naming, a helper's
signature, which of two equivalent in-repo utilities to call, test organization — a code
comment or PR description is the right register. The failure mode on this side is ADR-spam:
20 trivial ADRs bury the 3 that matter, and the team stops reading the directory. If reversing
the decision is a refactor, not a migration or a cross-team negotiation, skip the ADR.

## Template — MADR-lite

File: `docs/adr/NNNN-slug.md`. No YAML frontmatter.

```markdown
# NNNN. <Decision as a short active sentence>

Status: proposed | accepted (YYYY-MM-DD, approver) | superseded-by-NNNN | deprecated

## Context
The forces: what problem, what constraints (numbers or labeled assumptions), what the current
system does (file:line cites). 3–8 lines. No solutions here.

## Decision
Active voice, one paragraph. "We will <do X> <scoped how>." Not "it was considered that".

## Consequences
Positive:
- ...
Negative:
- ...

## Alternatives considered
- <Alt A> — 1–2 lines: what it was, why not.
- <Alt B> — 1–2 lines.
```

**The consequences rule**: a Consequences section with no downsides is a sales pitch — reject
it. Every real architectural decision costs something (a new failure mode, an operational
burden, a latency, a coupling, a learning curve). If you can't name a negative consequence,
you haven't understood the decision yet; go back to Context.

**Alternatives rule**: 1–2 entries, each with an honest "why not" traceable to the Context
forces. Zero alternatives means no decision was made — something was merely done.

## Numbering & storage

- `docs/adr/NNNN-slug.md` — `0001-...`, `0002-...`, zero-padded 4 digits, monotonic across
  the repo. Next number = highest existing + 1, even across deleted/deprecated entries.
- **Never renumber.** ADR numbers are citation targets in code comments, design docs, and
  other ADRs; renumbering breaks every reference.
- **Never edit an accepted ADR's Decision/Context/Consequences.** Accepted ADRs are immutable
  history. Wrong or outdated → write a new ADR that supersedes it; the only permitted edits to
  the old one are the Status line (`superseded-by-NNNN`) and a one-line pointer at the top.
  Typo fixes that don't change meaning are fine.

## Lifecycle

- `proposed` → `accepted` requires a named approver on the Status line, or the team's explicit
  convention ("merged to main = accepted") stated once in `docs/adr/README.md`. An agent never
  self-accepts an ADR that breaks an invariant or binds another team — that goes to a human.
- Superseding links **both ways**: new ADR's Context says "Supersedes 0007"; old ADR's Status
  becomes `superseded-by-0012`. One-way links rot.
- `deprecated` = the decision no longer applies and nothing replaced it (the feature died).
- Status changes are commits, so the history is the audit trail.

## Hygiene checks — runnable by an agent

Run these during `critique`/`audit`, or when asked to "check the ADRs":

1. **Decision-vs-code spot check**: for each `accepted` ADR, extract the one falsifiable claim
   in its Decision ("all case events go through the outbox", "reads come from replica X") and
   grep/read for a counterexample — a writer bypassing the outbox, a new synchronous call where
   the ADR mandates async. Check the 3 most load-bearing ADRs deeply rather than all
   superficially. A violated ADR is either a bug (fix the code) or a dead decision (supersede
   it) — report which, never ignore.
2. **Stale proposals**: `proposed` ADRs older than one sprint (~2 weeks by file/commit date)
   get flagged to the user: accept, reject, or delete. A proposal nobody accepted is a decision
   nobody made, and it silently anchors future designs.
3. **Link integrity**: every `superseded-by-NNNN` target exists; every superseding ADR points
   back. Numbering gaps are fine; dangling pointers are not.
4. **Invariant sync**: ADRs that changed an ARCHITECTURE.md invariant are actually reflected
   there (and vice versa — invariants with no originating ADR are flagged for capture).

## Example ADR — the bar to match

```markdown
# 0007. Publish case events via a transactional outbox

Status: accepted (2026-03-02, priya-k)

## Context
Case state changes in `cm/case_service.py:114` must reach the notification service and the
audit pipeline. Today the service POSTs to notifications inside the request path; audit gets
nothing. Dual-writing DB + broker in one request cannot be atomic (no 2PC per our doctrine),
and notification downtime currently fails case updates. Volume: ~40k case events/day
(metrics dashboard, Feb avg); ASSUMPTION: <5× growth this year.

## Decision
We will write case events to an `outbox_events` table in the same transaction as the case
mutation, and a single relay process will publish them to the `case-events` topic with
at-least-once delivery. Consumers deduplicate on `event_id`.

## Consequences
Positive:
- Case mutations and event emission are atomic; notification downtime no longer fails writes.
- One event stream serves notifications, audit, and future consumers.
Negative:
- Consumers see events with relay lag (p99 ~2s measured in staging); UIs reading the topic
  are stale by that much.
- New moving part: the relay needs liveness alerting (oldest-unpublished-row age) and is a
  new on-call surface.
- Duplicate delivery is now guaranteed-possible; every consumer MUST implement dedup — a
  per-consumer tax forever.

## Alternatives considered
- Synchronous dual-write (status quo + audit) — no atomicity; broker outage fails case writes.
- Change-data-capture (Debezium) — atomic too, but a heavier operational footprint than one
  relay process; unjustified at 40k events/day.
```

Note what makes it good: numbers with sources, an assumption labeled, negatives that are real
costs (a forever-tax on consumers, a new on-call surface), and alternatives rejected against
the stated forces — not against strawmen.
