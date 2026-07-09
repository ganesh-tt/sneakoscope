# Queries — SQL query-text doctrine (engine-agnostic)

Rules that hold across MySQL/MariaDB/Postgres-class engines. What the optimizer can and cannot
use is decided by the query *text* — most slow queries are self-inflicted at the keyboard.
Schema-side rules (index design, composite order) live in
`sneakoscope-architecture/reference/data.md` §Indexing; this file is the query side.

## Column selection

- **No `SELECT *` in application code.** Two failure modes: (a) a schema change silently
  reshapes every result set — added columns break positional readers, widen payloads, and leak
  new PII fields into logs/exports nobody reviewed; (b) it forecloses index-only scans — the
  engine must visit the row even when the index alone could answer. Name the columns; the
  column list is part of the code's contract.
- `SELECT *` is acceptable in ad-hoc investigation and nowhere that ships.

## Sargability — keep the indexed column naked

An index is usable only when the indexed column stands alone on its side of the predicate.

- **No function or cast on the column side.** `WHERE DATE(created_at) = '2026-07-09'` scans;
  `WHERE created_at >= '2026-07-09' AND created_at < '2026-07-10'` seeks. Move the
  transformation to the literal side, always.
- **Implicit-type-conversion tell**: a string column compared to an int literal
  (`WHERE account_no = 12345` on `VARCHAR account_no`) forces a per-row cast = full scan, and
  it's invisible in the query text — you must know the column type. Any cross-type predicate
  in review is a finding until the DDL is checked.
- **Leading-wildcard `LIKE '%foo'` = scan**, no exceptions. `LIKE 'foo%'` can seek. If the
  product needs contains-search, that's a search-engine requirement (see search.md), not a
  LIKE.
- **`OR` across different columns** usually kills index use — the optimizer can't merge two
  indexes cheaply everywhere. Rewrite as `UNION ALL` of two indexed queries (dedupe only if
  the branches can overlap). `OR` on the *same* column is fine (it's an IN).
- **IN-list bounds**: IN-lists grow with the code that builds them. Past a few hundred values,
  plan quality degrades and parse cost climbs; past a few thousand you can hit statement-size
  limits. Load the values into a temp table and JOIN — that also gives the optimizer
  statistics to plan with.

## JOINs

- **Join on indexed keys of the same type AND collation.** The cross-collation join tell:
  `utf8mb4_general_ci` column joined to `utf8mb4_unicode_ci` column silently casts per row —
  index dead, scan born, and the query text looks perfectly innocent. Same for INT-to-VARCHAR
  joins. Check both DDLs before approving any new join.
- **Explicit `JOIN ... ON` over comma syntax.** Comma joins with WHERE-clause conditions hide
  the join predicate; a forgotten condition is a cartesian product, and it's much easier to
  forget in a WHERE list.
- **Know your expected row multiplicity before writing the join.** 1:1, 1:N, N:M — say which.
  The **distinct-after-explode bug pattern**: join fans out rows (1:N), then `DISTINCT` or
  `GROUP BY` collapses them back. The query returns correct-looking data while scanning N×
  rows, and any aggregate computed before the collapse (SUM, COUNT) is silently multiplied.
  A `DISTINCT` whose purpose is "the join duplicates rows" is a bug marker — use
  EXISTS/IN-subquery for the filter, or aggregate the child side first.

## Pagination & ordering

- **No `OFFSET` beyond ~10k rows.** `LIMIT 20 OFFSET 100000` reads and discards 100k rows on
  every page — cost grows linearly with page number, and deep crawlers will find it. Use
  keyset/cursor pagination: `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC
  LIMIT 20`. The cursor is the last row of the previous page; the tie-breaker column is
  mandatory or pages skip/duplicate rows.
- **Every ORDER BY needs an index that produces the order, or the sort spills.** The tells in
  EXPLAIN: `Using filesort` (MySQL) — the engine sorted materialized rows, possibly on disk.
  Fine for 200 rows, an incident at 2M. ORDER BY on a different column than the WHERE-index
  usually means the composite index is in the wrong order (equality first, then sort column —
  data.md rule).

## Aggregation

- **Filter before you aggregate.** Conditions on raw rows go in `WHERE`; `HAVING` runs after
  grouping, on the aggregated result. A row-level condition in HAVING forces the engine to
  group everything first and discard after — a scan wearing a GROUP BY.
- **`COUNT(*)` vs `COUNT(col)` are different questions.** `COUNT(*)` counts rows;
  `COUNT(col)` counts rows where col IS NOT NULL. A nullable column turns them into different
  numbers, silently. If you mean rows, write `COUNT(*)`.
- Row-limit-before-distinct: applying `LIMIT` before deduplication caps the *input*, not the
  distinct output — result count varies with physical row order. LIMIT goes after DISTINCT /
  GROUP BY semantics are settled.

## Locking & transactions

- **`SELECT ... FOR UPDATE` scope minimal**: exact rows by indexed key, shortest possible
  transaction. FOR UPDATE on an unindexed predicate locks every row it scanned, not every row
  it returned.
- **No user interaction — and no external I/O (HTTP call, queue publish) — inside an open
  transaction.** Lock hold time becomes network latency; one slow downstream and the lock
  queue behind you is the outage.
- **Batch UPDATE/DELETE chunked with LIMIT** (loop until rows-affected < chunk, sleep between
  chunks). A single-statement million-row UPDATE is a lock storm on the primary and a
  replication-lag cliff on every replica — replicas replay it as one serialized unit.

## EXPLAIN discipline

- **Every performance claim cites an EXPLAIN, not vibes.** "Should be fine, it's indexed" is
  an assumption until the plan confirms the index is *chosen*.
- Read, minimum: **access type** (`ALL` = full scan — finding; `index` = full index scan —
  usually also a finding; `range`/`ref`/`eq_ref` = healthy), **rows** (examined estimate —
  compare to rows *returned*; a 100000-examined / 20-returned ratio is a missing or wrong
  index), **filtered** (% surviving the WHERE — low % on a big rows count = the index isn't
  doing the filtering).
- Extra-column tells: `Using filesort` (sort not served by an index), `Using temporary`
  (materialized intermediate — GROUP BY/DISTINCT without index support). Either on a hot
  query is a fix-before-ship.
- Run EXPLAIN with production-shaped parameters — an empty dev table makes every plan look
  great.

## Injection

- **Bind parameters ALWAYS.** SQL built by string concatenation with any user-influenced
  value is a P0 security finding, not a style comment — no "it's internal", no "it's
  validated upstream". This includes ORDER BY / column-name injection: those can't be bound,
  so they pass through an allowlist map, never through the request string.
- LIKE patterns are user input too: escape `%` and `_` before binding or users can wildcard
  your table.

## NULL semantics

- **`NOT IN` with a NULL in the list returns nothing** — the tell:
  `WHERE id NOT IN (SELECT ref_id FROM t)` silently returns zero rows the day `ref_id` gets
  its first NULL. Use `NOT EXISTS`, which has the semantics everyone thought NOT IN had.
- WHERE is three-valued: a predicate evaluating to UNKNOWN drops the row, so `col != 'x'`
  excludes NULL rows too. If NULLs should pass, say so: `col != 'x' OR col IS NULL`.
- `NULL = NULL` is not true. Equality on nullable join keys silently drops NULL-keyed rows —
  usually correct, occasionally the bug; decide, don't discover.
