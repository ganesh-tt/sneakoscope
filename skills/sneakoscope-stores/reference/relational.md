# Relational — MySQL / MariaDB (InnoDB) specifics

Engine-level behavior that generic SQL advice misses. Query-text rules: `queries.md`. Key
choice, evolution phasing, widths: `sneakoscope-architecture/reference/data.md` — this file is
what InnoDB does with those choices.

## Clustered PK = physical layout

- **InnoDB stores the table AS the PK B-tree.** PK choice is not an identity decision only —
  it's the physical row order on disk. PK-ordered range reads are sequential; anything else
  hops pages.
- **Monotonic keys for insert-heavy tables.** Auto-increment or UUIDv7/ULID append to the
  right-most page. **UUIDv4 PK = page-split storm**: every insert lands at a random B-tree
  position, splitting pages, fragmenting the index, and thrashing the buffer pool — measurable
  degradation from ~10k inserts/min. If v4 IDs are imposed externally, store them as a UNIQUE
  secondary column over a surrogate PK.
- **Secondary indexes carry the PK.** Every secondary index entry ends with the full primary
  key (that's how it locates the row). **Fat PK = fat every-index**: a 4-column composite or a
  36-char VARCHAR PK is silently multiplied into each of your six secondary indexes — index
  size, buffer-pool pressure, write amplification. Keep the PK narrow even when a natural
  composite "feels right".

## DDL semantics

- **`MODIFY COLUMN` silently drops `NOT NULL` and `DEFAULT`** unless restated. A width bump
  that quietly made the column nullable is a production bug class (data.md hard rule — restate
  the full definition, every time). Review tell: any MODIFY/CHANGE whose new definition is
  shorter than the current `SHOW CREATE TABLE` line.
- **ALTER on big tables: know the algorithm before running.** `ALGORITHM=INPLACE, LOCK=NONE`
  covers most adds (columns, secondary indexes) on modern versions — but the coverage matrix
  differs by version and between MySQL and MariaDB, so declare the algorithm explicitly and
  let the server *refuse* rather than silently fall back to `COPY` (full table rebuild under
  lock, hours on a 100M-row table). For anything that forces COPY on a hot table:
  gh-ost/pt-online-schema-change class tooling, not a maintenance-window prayer.
- Adding an index still costs a full build even INPLACE — schedule it, watch replica lag.

## Charset & collation

- **`utf8` is 3-byte broken — `utf8mb4` always.** MySQL's `utf8` rejects astral-plane
  characters (emoji, some CJK); the failure is an insert error or silent truncation at the
  first real-world user string. There is no reason to create a new `utf8` column in this
  decade.
- **Collation mismatch kills index use on joins** (the cross-collation tell in queries.md).
  One collation per schema, declared at database level, verified in every migration — a table
  created without an explicit collation inherits whatever the server default was that day.

## Replication

- **Read-your-writes on replicas is a lie without session pinning.** The user who just
  submitted must read from the primary (or a session pinned past their write's GTID); the
  freshness budget per read path is a product decision (data.md §Consistency). A load-balanced
  read pool with no pinning WILL serve the pre-write row to its author.
- **Big transactions stall replicas.** A replica replays each transaction as a unit — one
  10M-row UPDATE means the replica applies nothing else until it finishes. This is the second
  reason (after primary lock storms) that batch mutations chunk with LIMIT.
- Monitor `Seconds_Behind_Master`-class lag as a first-class SLO if anything user-facing
  reads replicas.

## Connection pooling

- **Every pool sets a validation strategy** (`validationQuery`/`testOnBorrow` or JDBC
  `isValid`). The stale-connection bug class: server-side `wait_timeout` (default 8h) reaps
  idle connections overnight; the pool hands out a corpse; the first request of the morning
  throws "Communications link failure". Intermittent, unreproducible after retry,
  first-login-of-the-day shaped — if you hear that symptom, check the pool config before
  anything else.
- Pool max size is a capacity decision against `max_connections`, per instance × per app
  replica — N app pods × pool-max must fit under the server limit with headroom for humans.

## Isolation — REPEATABLE READ defaults

- InnoDB's default is **REPEATABLE READ**, not READ COMMITTED. Consequences people trip on:
  - Long-running transactions read a snapshot from their first read — a "why is this SELECT
    stale" inside a transaction opened minutes ago is working as designed.
  - **Gap locks on range UPDATE/DELETE**: a range mutation locks the gaps *between* index
    records, blocking inserts into the range — deadlocks between writers that never touch the
    same row. Range mutations on hot tables → narrow indexed predicates, small transactions,
    or drop that workload to READ COMMITTED explicitly.
  - Locking reads (`FOR UPDATE`) still see the *latest* committed row, not the snapshot —
    mixed plain/locking reads in one transaction can observe two different states.

## Identifier case

- **Table-name case sensitivity varies by OS/config** (`lower_case_table_names`: Linux 0 =
  sensitive, macOS/Windows ≠ 0). Code that works on a dev laptop can 
  "table doesn't exist" on the Linux primary. Never rely on it: pick one case convention
  (lower_snake) for tables AND columns and enforce it in review. Column names are always
  case-insensitive in lookup but case-*preserving* — case-sensitive application code
  (`'MANUAL_RISK_LEVEL' in columns` checks) breaks against half the fleet when migrations and
  base DDL disagree on casing (data.md case-consistency rule).
