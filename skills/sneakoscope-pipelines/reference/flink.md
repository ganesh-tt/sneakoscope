# Flink / Streaming Doctrine

Streaming (Flink and Flink-shaped: keyed state + event time + checkpoints) design and review
rules. Every rule ships with its TELL. Evidence: Flink UI (backpressure/checkpoint/watermark
panels), job config, `file:line` in operator code.

## State

- **Keyed state size IS your capacity plan.** `keys × bytes-per-key × operators-holding-state`
  — computed with numbers before the job ships, re-checked when key cardinality assumptions
  change. "State is small" is an assumption; label it.
- **TTL on every state, or a written justification for unbounded.** Keys arrive forever;
  without TTL, state grows forever. The justification names the natural bound (key space is
  finite and sized) — "we clear it in the business logic" counts only if the clearing path is
  cited and covers abandoned keys (sessions that never close, orders that never complete).
  - TELL: checkpoint size growing monotonically across weeks at flat traffic — that's a state
    leak, not growth. Plot checkpoint size vs traffic; diverging curves = leak.
- **RocksDB vs heap is a stated choice.** Heap: state must fit in JVM memory across restarts and
  rescale — fast, GC-coupled. RocksDB: state >> memory, per-access serialization cost, disk +
  incremental checkpoints. Rule of thumb: projected state > ~1–2GB/slot → RocksDB. The design
  says which and shows the state-size number that decided it.
- State is per-key private; anything "shared across keys" is a broadcast state or an external
  lookup — a plain static map in the operator is per-TaskManager, divergent, and wrong on rescale.

## Watermarks & lateness

- **Lateness is a product decision, not a config default.** Someone chooses: window fires at
  watermark, late events within `allowedLateness` update the result (downstream must tolerate
  retractions/updates — say how), and later-than-that events go to a **side output** with a named
  consumer. **Never silent drop** — a dropped-late-event with no counter is data loss nobody approved.
  - TELL: `numLateRecordsDropped` > 0 with no alert and no side output — the product decision was
    made by a default.
- Watermark strategy states its out-of-orderness bound with a number derived from measured event
  lag (p99 of `event_time - ingest_time`), not a copied `Duration.ofSeconds(5)`.
- **The idle-source watermark stall**: watermark = min across all inputs/partitions; one idle
  Kafka partition or quiet source pins the watermark forever and **no window ever fires**.
  - TELL: job "running" healthily, zero output, watermarks frozen in the UI. Fix:
    `withIdleness(...)` on the strategy — and an alert on watermark lag regardless.
- Watermark lag (`current_time - current_watermark`) is a first-class metric with an alert
  threshold. It is the streaming equivalent of consumer lag; without it, stalls are discovered by
  downstream staleness.

## Exactly-once reality

- **Checkpointing gives exactly-once for Flink's internal state only.** End-to-end
  exactly-once additionally requires the sink to be **transactional** (two-phase commit,
  e.g. Kafka transactions) or **idempotent** (keyed upsert). A design claiming exactly-once
  names the sink mechanism or it's claiming at-least-once with better marketing.
- **2PC sinks buy exactness with latency**: output commits only on checkpoint completion, so
  end-to-end latency ≥ checkpoint interval, and consumers must read `read_committed`. State the
  latency cost next to the exactness claim; if the consumer can't wait, choose idempotent-sink +
  at-least-once and say so.
- Side effects in operators (HTTP calls, non-transactional DB writes, notifications) **replay on
  recovery** — every restore re-executes since the last checkpoint. Side effects are idempotent-keyed
  or moved behind the sink; an email-sending operator double-sends by design.
  - TELL: "duplicates only after deploys/failures" — that's checkpoint replay meeting a non-idempotent effect.

## Checkpointing discipline

- **Interval is a recovery-time trade**: recovery replays up to one interval of input.
  Interval 10min = up to 10min re-ingest at restore, at source-read speed. Pick the interval
  from the tolerable recovery gap, not from "the example used 1 minute". Also set
  `minPauseBetweenCheckpoints` so back-to-back checkpoints can't starve processing.
  - TELL: checkpoint duration approaching the interval — the job spends its life checkpointing;
    raise interval, go incremental, or shrink state.
- **Checkpoint size growth at flat traffic = state leak** (see State). Alert on checkpoint
  size and duration trends, not just failures.
- **Savepoint before every deploy. No exceptions.** Deploy = savepoint → stop → new version →
  restore from savepoint. Restarting from "latest checkpoint" across a code change is gambling
  with state compatibility.
- **State-schema evolution rules**: every stateful operator has an explicit stable `uid()`
  (auto-generated uids change with topology edits and orphan your state — TELL: restore
  "succeeds" with empty state); state types evolve compatibly (add-with-default; never rename/
  retype in place — POJO/Avro evolution rules, or a state-migration job); dropping an operator
  needs `allowNonRestoredState`, consciously.
- Checkpoint failures have a tolerance budget (`tolerableCheckpointFailureNumber`) and an alert.
  A job silently failing every checkpoint runs fine until the crash, then replays hours.

## Backpressure

- **Read the backpressure panel backwards**: the operator *showing* backpressure is a victim;
  the bottleneck is the first *downstream* operator that is busy (high busy-time, not
  backpressured). Blaming the red operator is the classic mis-fix.
- **The slow-sink domino**: sink degrades → buffers fill upstream hop by hop → sources stall →
  consumer lag climbs → checkpoints time out (barriers stuck behind queued data) → job restarts
  → replays into the same slow sink. One slow sink presents as "flaky checkpoints".
  Unaligned checkpoints relieve the barrier symptom; the sink is still the disease.
- **Never blocking calls inside operators.** A synchronous HTTP/DB lookup in a `map` caps
  throughput at `parallelism / latency` and manufactures backpressure. Enrichment lookups use
  **async I/O** (`AsyncDataStream`, ordered vs unordered stated, timeout + capacity set, timeout
  handler defined — the default throws and fails the job) — or better, join against a
  broadcast/temporal table and skip the RPC entirely.
  - TELL: throughput ceiling that scales linearly with parallelism but never reaches input rate,
    operator busy-time pegged, CPU idle — threads parked on I/O.

## Restart strategies & poison records

- **Bounded restarts with backoff** (`failure-rate` or `exponential-delay`). The default-ish
  fixed-delay-forever turns a deterministic bug into an infinite crash loop that looks "up" in
  the process list. Restart storms also hammer sources/sinks with replay.
- **A poison record fails the job deterministically on every replay** — restart strategy can't
  save you; the record is waiting in the source at the same offset.
  - TELL: crash loop where every failure cites the same exception ± the same offset/key.
  - Fix: deserialize/validate at the edge, route failures to a **dead-letter output** with the
    raw bytes + error + offset, count + alert on it. Never `try/catch → skip` without the DLQ
    and counter — that's silent drop again.
- Distinguish restart (transient, bounded, automatic) from stop-and-page (schema drift, DLQ rate
  above threshold, state restore failure). The second list is written down per job.

## Event time vs processing time

- **Pick explicitly, per job, in the design.** Event time: reproducible, replayable, requires
  watermarks and lateness handling. Processing time: simpler, lower latency — and
  **non-reproducible**: a replay/backfill produces different windows than the original run,
  because "now" moved. If anyone will ever reprocess this stream (they will — it's Tuesday),
  processing-time aggregates are unverifiable. Say this in the design if choosing processing time.
- Mixed-time bugs: comparing event-time fields against `System.currentTimeMillis()` inside
  operators reintroduces processing time silently. TELL: results that differ between live run and
  replay of the same input.

## Anti-patterns — reject on sight

| Anti-pattern | Why it's fatal | Replace with |
|---|---|---|
| Keyed state with no TTL and no bound argument | OOM/disk-full with a delay | TTL, or cited natural bound |
| Blocking RPC in map/flatMap | throughput cap + fake backpressure | async I/O or broadcast/temporal join |
| No `uid()` on stateful operators | topology edit orphans state on restore | explicit stable uids, always |
| "Exactly-once" with a plain HTTP/JDBC sink | duplicates on every recovery | transactional or keyed-idempotent sink |
| Dropping late data by default, no counter | unapproved data loss | allowedLateness + side output + alert |
| Fixed-delay infinite restart | crash loop wearing an "up" costume | failure-rate strategy + DLQ for poison records |
| Processing-time windows on replayable business data | backfills produce different answers | event time + watermarks, or a written waiver |
| Checkpoint interval copied from a tutorial | recovery gap nobody chose | interval derived from tolerable replay window |
| Static mutable map in operator "for caching" | per-TM divergence, lost on restart | keyed/broadcast state |
