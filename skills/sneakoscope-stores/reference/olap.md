# OLAP — StarRocks-class columnar + Hive/Trino

Two families: real-time columnar MPP (StarRocks/ClickHouse/Doris class) and lake SQL
(Hive tables queried by Hive/Trino/Spark). Both are scan engines — they reward pruning and
batch ingestion, and punish point access and small writes. Store-selection boundaries:
SKILL.md; this file is what you do once the workload legitimately lives here.

## StarRocks-class columnar

### Table model choice — pick by mutation pattern

- **Duplicate key**: append-only facts (events, logs). No dedup, cheapest writes, default for
  immutable data.
- **Aggregate key**: pre-aggregated rollups (SUM/MAX per key). Only when the raw rows are
  never needed — you can't un-aggregate.
- **Unique key**: merge-on-read upserts. **The unique-key merge cost lands on READ** — every
  query pays version-merging across rowsets; heavy upsert + heavy read on unique-key is a
  latency surprise. Prefer primary-key model where available.
- **Primary key**: merge-on-write upserts + delete-by-key — pay at ingest, read clean. The
  default for mutable dimensions/state tables on modern versions.
- Review tell: a unique/primary-key model chosen for data that never mutates — you bought
  merge machinery for nothing; use duplicate.

### Partition + bucket doctrine

- **Partition prune or die.** Partition by the dominant filter column (almost always the date
  column); **every production query must carry a partition filter** — a query without one
  scans the table, and one such dashboard tile at refresh interval is a cluster outage. Make
  "partition filter present" a query-review checklist item.
- **Bucket count sized to data volume**, targeting roughly single-GB tablets — not "one per
  CPU because the example said 10". The **too-many-tablets tell**: metadata bloat, slow
  ingest, FE memory pressure on a cluster whose *data* volume is modest — thousands of tiny
  tablets from copy-pasted `BUCKETS 32` on hundreds of small partitioned tables.
- Bucket key = the high-cardinality column you also join/group on (colocation wins); a
  low-cardinality bucket key skews tablets.

### No point-lookup serving

- **Columnar scan engines serve aggregates.** A user-facing "fetch this one row by ID" per
  click belongs in the row store; the OLAP table answers the dashboard, fed asynchronously.
  The tell: an application DAO with `WHERE id = ?` against the OLAP connection.

### Ingestion

- **Per-row INSERTs are an anti-pattern** — each tiny load creates a version/rowset the
  engine must compact; row-at-a-time trickle is a compaction death spiral. Batch or stream
  load in micro-batches (seconds-to-minutes cadence, thousands of rows per batch).
- **Exactly-once via labels**: every load job carries an idempotency label; retry with the
  same label is deduplicated. A retrying ingest pipeline without labels double-loads on the
  first network blip — and in an aggregate-key table that's silently wrong sums, not
  duplicate rows you can see.

### Schema change cost

- Light-schema-change (add/drop column) is cheap on modern versions; type changes and
  key-column changes rewrite data. Same discipline as relational big-ALTERs: know which class
  the change is before running it on a TB-scale table, and follow data.md's
  expand→migrate→contract for anything readers depend on.

## Hive / Trino

### Partition discipline

- **Partition columns in every WHERE.** The full-table-scan-on-unpartitioned-date tell: table
  partitioned by `dt`, query filters `event_time` (a data column) — reads every partition ever
  written, correct results, thousandfold cost. Filter on the partition column itself, with a
  literal or pushdown-able expression, in every query.
- **Over-partitioning creates the small-files problem**: partition by (date, hour, customer)
  and each leaf holds kilobyte files — HDFS/S3 metadata pressure, per-file open cost dominates,
  planners crawl. Target file sizes in the 100MB–1GB range; compact small files as a scheduled
  job; partition only on columns queries actually filter by.

### File formats

- **Columnar ORC or Parquet + compression (zstd/snappy), always** — text/CSV/JSON "tables"
  forfeit column pruning, predicate pushdown, and statistics in one move. A production Hive
  table in TEXTFILE is a finding.
- **Schema evolution rules are per-format**: Parquet/ORC support add-column-at-end and
  renames-by-name vs by-position differently depending on reader configuration. Verify the
  exact engine's resolution mode before renaming or reordering anything — a mismatched
  rename reads the wrong column's bytes *silently*. Additive-only is the safe posture;
  anything else gets a tested read-back on real files.

### Trino federation

- **Verify predicate pushdown per connector — don't assume it.** The federated-join tell: a
  Trino query joining a lake table to a MySQL/ES connector pulls the ENTIRE remote table when
  the predicate doesn't push down, melting the OLTP store you were told never to scan. Check
  `EXPLAIN` for the pushed-down filter in the remote scan node before shipping any federated
  query.
- **CTAS for repeated expensive queries**: the same heavy federation/aggregation run per
  dashboard refresh gets materialized once (CTAS / scheduled INSERT) and served from the
  copy. The copy owes the data.md sync + drift-check triple.

### Statistics

- **ANALYZE after significant loads** — Trino/Hive CBOs plan joins from table/column stats;
  **stale stats = bad plans** (wrong join order, broadcast of a "small" table that grew 100×).
  Stats collection is part of the ingest pipeline, not a manual afterthought; a plan that
  suddenly degraded after a backfill is stale statistics until proven otherwise.
