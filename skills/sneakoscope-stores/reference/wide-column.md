# Wide-column — Cassandra / ScyllaDB

The relational instincts are wrong here — most Cassandra incidents are relational habits
applied to a log-structured, partition-addressed store. Entity/identity doctrine still comes
from `sneakoscope-architecture/reference/data.md`; the physical model below overrides its
normalization default.

## Query-first modeling

- **You model TABLES PER QUERY, not entities.** Design starts from the list of queries the
  product will run — each query shape gets a table whose partition key IS the query's WHERE
  clause. Normalization is not the default here; controlled duplication across query tables
  is, with the app (or a materialized view) writing all copies.
- **A new query shape usually means a new table/MV**, not a clever predicate on an existing
  one. If the roadmap can't enumerate its query shapes, the product isn't ready for this
  engine — stay relational (SKILL.md store-selection rule).
- Every duplicated copy still owes data.md's denormalization triple: sync mechanism, lag
  budget, drift check.

## Partition key doctrine

- **The partition is the unit of locality AND of hotspotting.** All rows in a partition live
  together on the same replicas — reads within one partition are cheap; a hot partition key
  (one celebrity customer, one "default" tenant, today's date) concentrates the cluster's
  load on three nodes.
- **Unbounded partitions are the widening-partition death.** Time-series keyed by
  `(device_id)` with time as clustering column grows forever — reads slow quarter by quarter,
  compaction chokes, then repairs fail. Bucket the partition key by time:
  `((device_id, day_bucket), event_time)` — bucket size chosen so partitions stay bounded.
- **Target partition sizes**: aim ≤ ~100MB and low-hundreds-of-thousands of rows per
  partition; treat multi-GB partitions as an incident in progress. Estimate at design time:
  rows/day × row width × bucket span — written into the design doc, not discovered in
  `nodetool tablehistograms`.
- Clustering columns define within-partition sort order — that order is your ORDER BY; there
  is no other.

## No ad-hoc queries

- **`ALLOW FILTERING` is a table scan wearing a flag** — the engine telling you the query
  doesn't match the model, and you overruling it. In application code it is a finding, always.
  The tell in review: any CQL string containing `ALLOW FILTERING` shipped to production.
- **Secondary indexes are per-node scatter-gather** — a query on one fans out to every node.
  Narrow legitimate use: low-cardinality-ish column, always combined with a partition key.
  As a substitute for a query table: no.
- Analytics/ad-hoc exploration goes through Spark or an OLAP copy, never through CQL scans of
  the serving cluster.

## Write semantics

- **Everything is an upsert.** `INSERT` does not fail on an existing key — it overwrites,
  column by column. Uniqueness is not a constraint the engine gives you (see LWT below);
  "check then insert" is a silent overwrite, not a race you might win.
- **Tombstones come from DELETE, TTL expiry, AND writing nulls.** A read must skim tombstones
  to find live data; the tombstone-scan tell: read latency climbing on a table whose data
  volume didn't, `TombstoneOverwhelmingException`/tombstone warnings in logs. Queue-like
  workloads (write, read, delete, repeat) are the canonical anti-pattern — this engine is not
  a queue.
- **Don't write nulls.** An unset column in a prepared statement bound with null writes a
  tombstone. Use `unset` (driver support) or per-column statements; an ORM that writes every
  mapped column on save is manufacturing tombstones.

## Lightweight transactions (LWT)

- `IF NOT EXISTS` / `IF value = ?` runs Paxos rounds — ~4× the latency and a serialization
  point per partition. **Design around, not on**: LWT for the rare uniqueness-critical write
  (username claim), never in a hot path, never as a general concurrency strategy. If most
  writes need CAS semantics, the workload is relational.

## Consistency levels

- **QUORUM writes + QUORUM reads for read-your-writes** (R + W > RF). That's the default
  posture for anything a user reads back.
- **ONE is eventual, and that's a product decision** — cheap and fast, and the row you just
  wrote may not be there yet. Per read path, the staleness budget is chosen with the product
  owner and written down (data.md §Consistency) — not inherited from whatever the driver
  default was.
- Consistency level is per-statement in the driver; audit that hot paths actually set what
  the design doc claims.

## Driver discipline

- **Page-size/fetch-size defaults silently truncate naive iteration** — the DefaultLimit
  tell: a listing that returns exactly 10/100/5000 rows every time is a driver page size, not
  your data size. Iterate the result set through the driver's paging (it fetches pages
  transparently) — never `LIMIT n` guessed to be "big enough", never collecting one page and
  calling it the result.
- **Prepared statements always** — same injection rule as SQL, plus unprepared statements
  re-parse per request and skip token-aware routing.
- Token-aware load balancing on (default in modern drivers) — verify it's not disabled by a
  copy-pasted legacy config; without it every request takes an extra network hop.
