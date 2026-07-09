# Evolve Flow

Plan a migration, replacement, or version transition safely. Input: current state + target state
(schema change, component replacement, datastore move, protocol/version bump). Output: a
migration plan — phases table, verification per phase, rollback per phase, risk register. Evolve
plans; the user-approved implementation executes.

The premise: **no big-bang cutovers.** A migration is a sequence of small, boring releases, each
independently shippable, verifiable, and reversible. If any phase can only be validated by
running the whole plan, the plan is one big bang wearing a phases costume.

## Doctrine

- **Every phase is independently shippable.** It merges to trunk, deploys alone, and the system
  is fully correct with that phase live and the next one never happening. A phase that leaves
  the system half-migrated-and-broken until the next deploy is not a phase; it's a hostage.
- **Every phase states "how we know it worked" BEFORE proceeding** — a check, a metric, a diff
  job result. Written into the plan up front, not improvised after deploy. "Deploy and watch the
  logs" is not a verification; "reconciliation diff = 0 rows over 24h" is.
- **Every phase is reversible, or is explicitly marked point-of-no-return** with sign-off (see
  Rollback doctrine). Reversibility is designed, not assumed.
- **One variable per release.** Migration and behavior change never ship together (see
  Anti-patterns) — if a regression appears, you must be able to attribute it.
- Phases have owners and exit criteria. A migration without a named "contract" (cleanup) phase
  in the plan never finishes — the dual paths become permanent architecture.

## Patterns — and when to use each

**Expand → migrate → contract** — *schema and contract changes (the default).*
Expand: add the new column/table/field/endpoint alongside the old; new code writes both/reads
old. Migrate: backfill; flip reads to new (behind a flag); verify. Contract: remove old
column/code — only after no deployed version reads it. Never rename in place; never drop what
N-1 still reads. Every expand ships with its rollback twin; contract is the point-of-no-return.

**Strangler fig** — *replacing a whole component/service.*
Put a routing seam in front of the old component. Route a small % of traffic (or one
endpoint/tenant) to the new one; **compare** (errors, latency, output diffs); **ratchet** the
percentage up only when the comparison is clean at the current step. The old path stays fully
functional until 100% + soak. The ratchet is the safety: at any step, rollback = set the dial
back.

**Dual-write with reconciliation** — *datastore moves where both stores must converge.
DANGEROUS — use only with both of:* (1) a **diff job** continuously comparing the two stores
and reporting divergence as a metric, and (2) a **kill switch** to drop back to single-write
instantly. Dual-write without reconciliation is **silent divergence**: the second write fails
sometimes (it's a network call; see failure.md), nobody notices, and six months later the
stores disagree and nobody knows which is right. The diff job is not optional tooling; it IS
the pattern. Prefer CDC/outbox-driven replication over app-level dual-write when available —
one write path, replication handles the second store.

**Shadow-read / dark launch** — *validating a new path before trusting it.*
Serve from the old path; also execute the new path on real production traffic, compare results,
record mismatches as metrics, **discard the new result**. Trust is earned by a mismatch rate,
not a code review. Use before any read-path flip in the patterns above. Watch the cost: shadow
traffic is real load on the new system — which is also the point (it's a load test with real
shapes).

**Branch by abstraction** — *long in-code migrations (library swap, engine replacement) where
trunk must stay shippable.* Introduce a seam (interface/port) over the old implementation;
build the new implementation behind it; switch via flag; delete old + seam when done. The seam
is scaffolding — it has a demolition date in the plan, or it becomes permanent indirection.

Choosing: schema/contract → expand-migrate-contract. Whole component with traffic →
strangler fig. Two datastores that must converge → CDC first, dual-write+reconciliation if
forced. Any read-path flip → shadow-read first. Multi-month in-repo rewrite → branch by
abstraction. Most real migrations compose two or three of these.

## Data backfill doctrine

The backfill is a production workload; design it like one.

- **Idempotent**: re-running any chunk produces the same end state (upsert by key, not append).
  The backfill WILL be re-run — after a crash, after a bug fix, after a bad chunk.
- **Resumable (checkpointed)**: progress recorded per chunk (last key / watermark). A crash at
  row 40M resumes at 40M, not at 0. "Restart from the beginning" on a multi-hour job is a plan
  to never finish.
- **Rate-limited**: explicit throughput cap with a runtime knob, and it yields to production —
  don't melt the primary. Run against a replica where possible; watch replication lag and
  p99 of the live traffic as the backfill's own SLO. Off-peak windows are a mitigation, not a
  substitute for the cap.
- **Verified by count + checksum sampling**: row counts match per partition/chunk, AND a random
  sample (e.g. 0.1%, minimum 10k rows) compared field-by-field or by checksum. Counts alone
  pass a backfill that wrote N rows of garbage.
- **Separated from the deploy that enables reads.** Backfill completes and verifies first; the
  read-flip is its own release, days later if needed. Coupling them means a backfill problem is
  a rollback of a deploy instead of a re-run of a job.

## Compatibility windows

- **N and N-1 must coexist.** Rolling deploys *guarantee* both versions run simultaneously —
  this is not an edge case, it is every deploy. Every change is written so that old-code +
  new-schema and new-code + old-schema (whichever the ordering produces) both work.
- **Every serialized artifact survives one version skew**: events in topics, cache entries,
  jobs in queues, session blobs, outbox rows. The queued-job-from-old-version tell: a job
  enqueued by N-1 (old payload shape) is dequeued by N after the deploy — if N can't parse it,
  the deploy just poisoned the queue with everything in flight. Queue retention defines the
  real window: a 4-day-retention topic means N must read N-1 payloads for at least 4 days, not
  "for one deploy".
- Tolerant reader everywhere: ignore unknown fields, default missing ones, never
  strict-schema-reject data written by a version you still support.
- Caches: version the key or the value schema; a deploy that changes the cached shape without
  bumping the key deserializes garbage at 100% cache-hit-rate speed.

## Rollback doctrine

- **Every phase has a tested rollback** — tested meaning executed, in staging at minimum,
  with the time-to-restore measured. An untested rollback is a hypothesis you'll first test
  during an outage.
- **OR is explicitly marked point-of-no-return** — the contract phase (drop old column, delete
  old service, expire old data) — with named sign-off in the plan and its own extended
  verification gate before execution. There is usually exactly one PONR per migration; if the
  plan has three, restructure it.
- **Feature flags for behavior switches**, not deploys: flipping a read path back must be a
  config change (seconds), not a rollback deploy (minutes-hours). But **flags are removed
  within one release after full rollout — flag debt is code debt**: every stale flag doubles
  the paths to test and hides which branch production actually runs.
- **"We'll roll back by restoring the backup" is not a rollback plan.** Restore time = outage
  time — has anyone *timed* the restore of this database at its current size? Plus everything
  written since the backup is lost. Backups are for disasters; migrations roll back by keeping
  the old path alive.

## Anti-patterns

- **Migration + behavior change in one release.** A regression appears — is it the new schema
  or the new logic? You can't attribute it, so you roll back both, including the healthy half.
  One variable per release.
- **Backfill through the ORM at request-path priority.** Per-row loads, hooks and validations
  firing 50M times, connection pool shared with live traffic. Backfills are batch SQL / bulk
  ops on a dedicated connection with their own rate limit.
- **"Restore the backup" as the rollback story.** See above. If it appears in a plan, replace
  it with a kept-alive old path or reject the plan.
- **Renaming while both versions live.** A rename is a drop + add wearing one DDL statement;
  N-1 reads the old name and breaks mid-rolling-deploy. Rename = expand (add new name, dual
  write) → migrate → contract (drop old name), like everything else.
- **Dual-write without a diff job** — silent divergence (see the pattern above).
- **The eternal expand.** Contract never scheduled; both columns/paths live for years; every
  new engineer must learn which one is real. Contract is a phase with a date, or the migration
  failed slowly.

## Output: the migration plan

```markdown
# Migration Plan — <what> (<date>)

Current → target (one paragraph each, with file:line / schema cites). Pattern(s) chosen + why.
Compatibility window: N-1 = <version>, longest-lived serialized artifact = <queue, retention Xd>.

## Phases

| # | Phase | Change | How we know it worked | Rollback | Blast radius |
|---|---|---|---|---|---|
| 1 | Expand | add `amount_v2` col + dual-write behind flag | dual-write error rate 0; col populated for 100% of new rows | flag off; drop col | writes +1 col, no reads |
| 2 | Backfill | checkpointed job, 5k rows/s cap | counts match per partition; 0.1% checksum sample clean; replica lag < 5s throughout | stop job; re-run (idempotent) | replica load only |
| 3 | Shadow read | read both, compare, serve old | mismatch metric < 0.01% over 7d | flag off | +1 read per request |
| 4 | Flip reads | serve `amount_v2` behind flag | error/latency flat vs baseline 48h; mismatch stays 0 | flag back (seconds) | read path |
| 5 | Contract ⚠ PONR | drop old col + dual-write code + flag | old col reader count = 0 for 14d (query log) before drop | none — sign-off: <name> | none if gate held |

Each phase: owner, target date, and "phase N+1 does not start until N's check passes".

## Risk register

| Risk | Likelihood | Impact | Mitigation | Trigger/kill switch |
|---|---|---|---|---|
| dual-write divergence | med | data corruption | diff job, alert on >0 | kill switch → single-write |
| backfill melts primary | med | prod latency | rate cap + replica + off-peak | pause knob, p99 alert |
| N-1 job can't parse new payload | high if unchecked | poisoned queue | tolerant reader test in CI against N-1 fixtures | halt deploy |
```

A plan passes evolve when: every phase is shippable alone, every verification is written before
execution, exactly the PONR phases lack rollback (with sign-off), and the contract phase has a
date.
