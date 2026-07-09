# Search — Elasticsearch / OpenSearch

An index, not a database. Every rule below follows from that: the store of record is
elsewhere, mappings are immutable-in-practice, and reads are near-real-time — design for all
three on day one because none can be retrofitted quietly.

## Index is rebuildable

- **Source of truth lives elsewhere** (SKILL.md rule). Every document must be reconstructible
  from a store of record, and the **reindex path is written and tested** — not "we'd replay
  from the DB somehow". You WILL reindex: mapping changes force it (below). A field that
  exists only in ES is a data-loss design.
- The sync path (CDC, outbox consumer, batch job) owes data.md's denormalization triple:
  named mechanism, lag budget, drift check (count + spot-checksum between store of record and
  index, scheduled).

## Mapping discipline

- **Explicit mappings, always.** Dynamic mapping guesses from the first value it sees —
  wrong once and forever: the first numeric-looking string makes the field `long` and every
  subsequent real string is a mapping exception; a date-ish string becomes `date` and rejects
  everything else. `dynamic: strict` (or at least `false`) on production indices; new fields
  are a reviewed mapping change.
- **`keyword` vs `text` is the load-bearing choice.** `text` is analyzed for full-text
  matching; `keyword` is exact-value for terms filters, sorting, aggregations. The
  aggregation-on-text failure: aggregating/sorting on a `text` field either errors or —
  with `fielddata: true` — silently aggregates on analyzed *tokens* ("New York" buckets as
  "new" and "york") while eating heap. IDs, statuses, codes = keyword; searchable prose =
  text; fields needing both = multi-field (`text` + `.keyword` sub-field).
- **Mapping changes on existing fields = reindex.** You cannot change a field's type in
  place. Therefore: **plan aliases from day one** — applications never talk to a concrete
  index name.

## Alias-based zero-downtime reindex

- Concrete index names are versioned (`alerts_v7`); the app knows only aliases. Read-alias
  and write-alias split: create `alerts_v8` with the new mapping → point write-alias at both
  (or dual-write from the ingest path) → `_reindex` v7→v8 → verify counts → flip read-alias
  → retire v7. An application configured with a bare index name has already forfeited this —
  that's the day-one review check.

## Query doctrine

- **Filter context over query context** for every yes/no condition (status, tenant, date
  range): filters skip scoring and are cached; the same condition in `must` computes
  relevance nobody reads. Tell: a `bool.must` full of `term`/`range` clauses on a query whose
  results are sorted by a field, not by score — move them to `filter`.
- **Deep pagination: `search_after`, not `from`/`size`.** `from + size` past 10k hits the
  `max_result_window` wall, and everything before the wall pays coordinate-and-discard cost
  per page (same disease as SQL OFFSET). Cursor with `search_after` + a deterministic sort
  including a tie-breaker (`_id`-like field).
- **Missing-index silent-failure class**: in multi-index searches, a per-index error or a
  client mapping a `Left`/error to an empty result **hides data loss as "no matches"** — the
  user sees zero hits and trusts it. Fail loud: treat index-not-found and shard failures
  (`_shards.failed > 0`) as errors, never as empty; `ignore_unavailable: true` is an explicit
  reviewed decision, not a default.

## Refresh & near-real-time

- A written document is searchable only after a refresh (default ~1s). **Read-after-write
  needs an explicit refresh policy** — `refresh=wait_for` on the write, or a UI that doesn't
  immediately re-query — and which one is a **product decision** per flow (data.md staleness
  budget). `refresh=true` per write on a hot ingest path is a performance foot-gun; a test
  suite that passes only with sleeps has this bug.

## Shard sizing

- **Target tens-of-GB per shard** (~10–50GB working band); shard count is fixed at index
  creation, so size from projected data, and use time-based indices (ILM/rollover) for
  append-heavy data instead of one ever-growing index.
- **The thousand-tiny-shards tell**: cluster with modest data but thousands of shards —
  every shard costs heap and cluster-state; master updates crawl, recoveries stall. Usually
  caused by daily indices × many small tenants × default shard counts. Fix with rollover by
  size, fewer primaries, and shrink/merge of the backlog.

## Aggregations cost

- **`cardinality` on high-cardinality fields is approximate (HyperLogLog) and
  memory-priced** — treat the number as an estimate (it is one) and don't put
  unique-count-of-user-id-per-term on an auto-refreshing dashboard without a
  `precision_threshold` decision.
- **Deep/unbounded `terms` aggregations** (high-cardinality field, large `size`) strain
  coordinating-node memory and return *approximate* top-N across shards. To page through all
  buckets, use `composite` aggregation with `after` — it's the search_after of aggregations;
  cranking `terms.size` to 100000 is the anti-pattern it replaces.
