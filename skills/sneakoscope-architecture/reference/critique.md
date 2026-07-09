# Critique Flow

Scored, read-only architecture review of an existing system or a design doc. Produces a report; touches nothing.

## Hard rules

- **Read-only.** No edits, no branch switches, no commits, no "quick fixes while I'm here". Findings are proposals.
- **Every finding cites `file:line`** (or doc section for design-doc reviews). A finding you cannot cite is a hypothesis — label it as one or drop it.
- **Working code needs a correctness consequence to be flagged.** "This would be cleaner as X" without a failure mode it prevents is a P3 at most, and usually silence. Never demand a refactor of working code without naming what breaks or degrades if it stays.
- **Respect ARCHITECTURE.md.** Read its invariants and "observed state" first. A documented, deliberate inconsistency is not a finding; an invariant violation is automatically ≥P1.
- **Score what's there against the stated load**, not against a FAANG fantasy. If the user gave no scale numbers, ask once or state the assumed load in the verdict.

## Scope

Target = whole system, one module, or a design doc (from `design`/`shape`). For a design doc, run the same dimensions against the doc's claims and check each claim about the existing system against the code — a design citing fiction is itself a P1.

## Evidence-gathering order

Read in this order before scoring anything — findings discovered later re-rank, they don't get bolted on:

1. **ARCHITECTURE.md + ENGINEERING.md** (if present): invariants, observed state, the bar. Missing → offer `init` once, then proceed from code with lower confidence, saying so in the verdict.
2. **The write paths**: every DAO/repository/producer for the stores in scope. Data-loss findings (P0s) live here; find them before polishing anything else.
3. **2–3 request paths end-to-end** including one failure branch: follow an exception from throw to HTTP response, follow a timeout from client construction to caller.
4. **One async flow end-to-end**: publish → broker → consume → side effect, checking id propagation, redelivery handling, and stuck-detection.
5. **Migrations + deploy artifacts**: latest 2–3 schema migrations against the previous release's code; Dockerfiles/CI against build-file pins.
6. **Tests for the riskiest path found**: a scary path with a real integration test drops one severity level; with no test, it keeps it.

Budget: for a whole-system critique read ~15–25 files deliberately, not 200 shallowly. A dimension you couldn't gather evidence for is scored `n/a — not assessed`, never guessed.

## The 8 dimensions — each scored /20

Score each dimension independently; the verdict reports all eight plus an overall (mean, rounded down — one 4/20 dimension should drag the headline). 17+ = sound, 12–16 = serviceable with named debt, 8–11 = at-risk, <8 = redesign conversation.

### 1. Boundaries & data ownership

- Does every table/collection have exactly one writing owner? **Tell**: two modules importing the same DAO or issuing INSERTs to the same table (`grep` the table name across write paths).
- Is each module boundary backed by data it owns? **Tell**: a "service" whose every method reads another module's tables — a facade, not a boundary.
- Do shared domain types cross deploy boundaries? **Tell**: a `common-models` jar whose change forces lockstep deploys of 3+ services.
- Is multi-writer arbitration documented where it exists? **Tell**: last-write-wins by accident — two writers, no version column, no comment.
- **Don't flag when**: a read-only reporting path reads another module's tables (reads don't break ownership); or ARCHITECTURE.md records the multi-writer state with an arbitration rule.

### 2. Dependency direction & layering

- Does domain logic import infrastructure? **Tell**: `import java.sql.*` / HTTP client / Kafka producer inside a service/domain class rather than behind a port.
- Are layer skips one-way and rare? **Tell**: controller calling DAO directly in some endpoints but not others — two conventions, pick from evidence which is dominant and flag the minority.
- Is the synchronous call-chain depth ≤3 across module/service boundaries? **Tell**: request path A→B→C→D where the 4th hop's p99 and failure rate silently become A's SLA.
- Are cycles absent between build units? **Tell**: module A's build file depending on B and B's tests depending on A.
- **Don't flag when**: the "layer skip" is a deliberate CQRS read path documented as such; or the infra import sits in a module ARCHITECTURE.md classifies as infra.

### 3. State & consistency

- One aggregate per transaction? **Tell**: a transaction block spanning writes to two aggregates in different consistency domains, or worse, a DB write + HTTP call inside one transaction.
- Cross-service writes via outbox/saga, never 2PC or hope? **Tell**: `save(); publish(event)` as two statements — a crash between them loses the event forever.
- Are dual writes actually transactional? **Tell**: a for-comprehension over two DAOs that reads like a transaction but commits independently (each `.update()` autocommits).
- Do caches have a defined invalidation owner? **Tell**: cache populated at read time, invalidated nowhere, "TTL will handle it" with a 24h TTL on user-visible data.
- Is read-your-writes needed and provided where the UX implies it? **Tell**: write to primary, immediate read from a lagging replica/search index feeding the confirmation screen.
- **Don't flag when**: eventual consistency is documented AND the consumer tolerates it (append-only analytics, metrics); or the "dual write" targets are one physical store.

### 4. Failure design

- Does every network call have an explicit timeout? **Tell**: default HTTP client construction (infinite or library-default timeout) — cite the client construction line.
- Caller timeout < callee timeout everywhere along each chain? **Tell**: gateway 30s calling a service whose DB statement timeout is 60s — the caller's gone, the work completes into the void.
- Are all retried operations idempotent with an explicit key? **Tell**: retry wrapper around a POST that inserts a row; queue consumer without dedup on redelivery.
- Are retries bounded (≤3), backed off with jitter, and skipped on 4xx? **Tell**: `while` retry loop, fixed sleep, retrying validation errors.
- Is every internal buffer/queue bounded with a defined at-bound behaviour? **Tell**: unbounded in-memory queue between producer and slow consumer — an OOM with a delay.
- Do fan-outs define per-item failure semantics? **Tell**: `Future.sequence`/`Promise.all` over N items where one failure discards N−1 successes with no report.
- **Don't flag when**: the call is to a co-deployed in-process component; or the retry wraps a read that is idempotent by nature and says so.

### 5. API contract quality

- One machine-readable error envelope across the surface? **Tell**: three handlers returning `{error: msg}`, `{message}`, and a bare 500 string.
- Are list endpoints bounded (default + max page size)? **Tell**: `findAll()` behind an endpoint, or a `limit` param with no server-side cap.
- Cursor pagination for unbounded sets? **Tell**: `OFFSET :page*:size` on a table whose row count Step-6 numbers put past ~10k.
- Tolerant-reader compatibility: adding fields free, removing/renaming = new version? **Tell**: strict deserialization (`failOnUnknownProperties=true`) on a consumer of an external contract.
- Do mutations return enough to avoid an immediate re-fetch? **Tell**: POST returns `201` + id only, and the FE's next line is a GET for the object it just sent.
- **Don't flag when**: the endpoint is internal-only with one known consumer and both sides deploy together; or offset pagination sits on a bounded reference table.

### 6. Evolution safety

- Do schema changes follow expand → migrate → contract? **Tell**: a migration that renames or drops a column the previous release's code still reads (check the release branch, not develop).
- Does every migration ship a rollback twin? **Tell**: `ddl/v6.4.0.sql` with no `rollback/v6.4.0.sql`.
- Is the same dependency pinned in one place? **Tell**: pom BOM says 26.5.7 while Dockerfile `FROM` says 26.5.5 — the runtime ships the old one.
- Are deprecated paths on a measured removal plan? **Tell**: `@Deprecated` since 3 versions ago, still called from live code, no usage metric.
- Can two versions of the code run against one schema during deploy? **Tell**: rolling deploy + a NOT NULL column added without default.
- **Don't flag when**: the repo deploys with full downtime windows and says so; or the "missing rollback" is for a data backfill explicitly documented as forward-only.

### 7. Observability & operability

- Is one correlation id traceable end-to-end? **Tell**: async hop (queue publish) that drops the id — the trace dies at the broker.
- Does every async flow answer "how do we know it's stuck"? **Tell**: consumer with no lag/DLQ-depth/oldest-message-age metric — the first symptom will be a user report.
- Are errors logged with context at the boundary where they're handled, once? **Tell**: same exception logged at DAO, service, AND controller (triple noise), or swallowed with `catch { }` (silence).
- Are operational knobs config, domain rules code? **Tell**: timeout hardcoded in source (redeploy to tune) or a YAML file encoding business branching (a programming language without tests).
- Can a new on-call answer "is it healthy" from endpoints/dashboards named in the repo? **Tell**: healthcheck that returns 200 unconditionally.
- **Don't flag when**: the flow is a dev-only tool; or the "hardcoded" value is genuinely invariant (protocol constant, not a knob).

### 8. Simplicity — moving parts vs stated load

- Is every datastore/queue/framework carrying load an existing part couldn't? **Tell**: Redis cache in front of a table serving <10 rps; Kafka between two components that deploy together (a function call in a costume).
- Is the design sized for ~10× stated load, not 1000×? **Tell**: sharding/CQRS/event-sourcing machinery with no measurement justifying it and no ADR.
- Are abstractions used ≥2 times or scheduled to be? **Tell**: an interface with one implementation and no test double using it.
- Does the microservice count exceed what the team topology supports? **Tell**: more services than engineers; shared DB behind "separate" services.
- **Don't flag when**: the extra part is a documented compliance/isolation requirement; or an ADR records the speculative capacity with a revisit date. Never score simplicity down for parts ARCHITECTURE.md's invariants require.

## Severity mapping

- **P0** — correctness, data loss, security: lost events (save-then-publish), non-idempotent retries on writes, auth bypass, migration that eats data, multi-writer clobber. Blocks; lead the report with these.
- **P1** — will bite in production: timeout inversions, unbounded queues/lists, no stuck-detection on async flows, dual-pin drift, missing rollback on destructive migration.
- **P2** — debt: convention splits, offset pagination approaching its cliff, single-impl abstractions, noisy logging. Fix when touching.
- **P3** — style/polish: naming, doc gaps, error-message wording. Never lead with these; cap at 5 in the report.

Invariant violations from ARCHITECTURE.md: minimum P1 regardless of dimension.

## Scoring calibration

Anchor each dimension's /20 to findings, not vibes:

- **18–20**: no P0/P1 in the dimension; at most scattered P3s. Reserve 20 for "checked hard, found nothing" — not "didn't look".
- **14–17**: no P0, ≤2 P1s, each with a plausible mitigation already half-present in the code.
- **9–13**: one P0 OR ≥3 P1s; the dimension works today by luck or low load.
- **≤8**: multiple P0s or a P0 with no cheap containment; this dimension is the redesign conversation.
- A single P0 caps its dimension at 13 and the overall at 15 — a system that loses data cannot be "sound" no matter how pretty the layering.
- Design-doc reviews score the same scale; an unanswered failure-design question in the doc (no timeout/retry/idempotency answer) counts as a P1 against dimension 4 — per the doctrine, that doc is unfinished.

## Anti-zealotry — the reviewer's own bans

- No finding phrased as a command ("must", "always") unless it's a P0. P1↓ are proposals with the failure scenario attached.
- No dogma citations without a local consequence: "violates hexagonal architecture" is not a finding; "domain class constructs its own Kafka producer (`file:line`), so unit tests need a broker and a broker outage fails order validation" is.
- No re-litigating recorded ADRs. If an ADR chose the thing you'd flag, the finding becomes at most "conditions since the ADR changed: <evidence>" — or silence.
- Count your own P3s: more P3s than P1s+P2s combined means you're padding; cut to the 5 best.
- One consolidated finding per root cause. Ten endpoints missing timeouts because one shared client has none = ONE finding at the client construction line, not ten.

## Output format

```markdown
# Architecture Critique — <target> @ <commit/doc version>, <date>

## Verdict
- **Overall: NN/20** (assumed load: <stated or assumed, one line>)
- Per dimension: Boundaries N · Layering N · State N · Failure N · API N · Evolution N · Observability N · Simplicity N
- **Top 3 risks**: one line each — the failure scenario, not the code smell.

## Findings (ranked)
### P0
- [DIM] <finding> — `file:line` — failure scenario — proposed direction (1 line, propose-only)
### P1 / P2 / P3
[same shape]

## Ranked backlog
1..N — ordered by (severity, then blast radius, then cost). Each entry: finding ref, effort guess (S/M/L), what unlocks it.

## Not flagged (deliberate)
[2–5 things that look wrong but were skipped, with the "don't flag when" reason — proves the review read the context.]

## Assumptions
[Load numbers, SLA guesses, anything the user should correct — corrections may re-rank the backlog.]
```

Close by offering the natural next commands: `harden <flow>` for the worst failure-design findings, `evolve <migration>` for evolution findings, `adr` to record any decision the critique forced into the open.
