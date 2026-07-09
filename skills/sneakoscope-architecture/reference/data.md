# Data — model data & evolve a schema

The `model` flow. Data outlives code: every service will be rewritten at least once against the
same tables. Model for the second team, the third year, and the 3 a.m. migration.

## Modeling workflow

Run these steps in order. Each produces a written artifact line in the design doc.

### 1. Entities and ownership

- List the entities. For each: **exactly one writing owner** (module or service). If two
  candidates both "need" to write it, the boundary is wrong — merge them, or split the table
  into two tables each with one writer. Multi-writer without a documented arbitration rule is
  a standing data-corruption invitation (per SKILL.md doctrine, non-negotiable).
- Name the reads too: which modules read this table, and through what (direct SQL, view, API)?
  Direct cross-module reads of another module's tables are schema coupling — every future
  migration now needs their sign-off. Prefer an API/view seam for anything crossing a boundary.

### 2. Identity

- **Surrogate keys by default.** Natural keys (email, account number, national ID) change,
  get retyped, and leak PII into every FK and log line. Use a natural key as the PK only when
  the domain guarantees immutability AND the value is not sensitive (ISO currency code: yes;
  email: never).
- Keep natural keys as UNIQUE-constrained columns — the constraint is the business rule.
- **UUIDv7 or ULID over UUIDv4** for generated IDs: time-ordered prefixes keep B-tree inserts
  append-ish. Random v4 PKs on a hot-insert table fragment the index and blow the buffer pool —
  measurable at ~10k+ inserts/min. Auto-increment integers are fine inside one database; they
  don't survive sharding, merging environments, or client-side ID generation.
- IDs are opaque to consumers. The moment a client parses meaning out of an ID, its format is
  frozen forever.

### 3. Normalize by default

- Third normal form is the starting point, not a debate. Denormalization is an optimization,
  and optimizations require a measurement.
- Every denormalized copy needs, written in the design doc: (a) the measured read-path
  justification (query + latency before/after, or at minimum the query plan), (b) the **sync
  mechanism** that keeps the copy fresh (trigger, outbox consumer, batch job — named, with its
  lag budget), (c) the reconciliation check that detects drift. A copy without all three is a
  future "why do these two screens disagree" ticket.
- Counter/rollup columns (`comment_count`, `total_amount`) are denormalization. Same three
  requirements apply.

### 4. Day-one columns

Retrofits change every index and every query shape — that's why these are day one:

- `created_at`, `updated_at` — UTC, set by the database or a single write path, never by
  callers passing wall-clock time.
- Soft-delete flag (`deleted_at` nullable timestamp beats a boolean — it's the flag AND the
  audit) **if the domain needs recoverability or referential history**. If added, every unique
  index and every default query must account for it from day one.
- Tenant key on every tenant-scoped table if multi-tenant — leading column of most composite
  indexes, part of every unique constraint. Adding it later is a full re-index of the schema.
- Optimistic-lock `version` column on any row that two user sessions can edit.

## Consistency doctrine

- **One aggregate per transaction.** A transaction that touches two aggregates is a
  boundary-drawing error — merge the aggregates or go eventual (outbox + idempotent consumer)
  and record the choice in an ADR. No distributed transactions, ever.
- **Read-your-writes only where the product needs it.** The user who just submitted a form
  must see their submission (route that read to the primary / same session). The list page
  other users see may lag.
- **Staleness budget is a product decision — document it per read path.** "This dashboard may
  be 30 seconds stale", "this balance is transactional" are lines in the design doc, decided
  with the product owner, not defaults inherited from whichever replica the ORM picked. A read
  path with no stated budget defaults to "must be fresh", which is the expensive answer —
  forcing the question is the point.

## Indexing rules

- **Every FK-pattern column gets an index at creation** — actual FK constraints and soft FKs
  (`*_id` columns joined in queries) alike. Databases don't auto-index FK columns; the missing
  index surfaces as a table scan on the child during parent deletes and joins.
- **Composite index column order: equality predicates first, then the range/sort column.**
  `WHERE tenant_id = ? AND status = ? AND created_at > ?` wants
  `(tenant_id, status, created_at)`. Range column first wastes the rest of the index.
- A composite index serves any left prefix; don't add `(a)` when `(a, b)` exists.
- **Missing-index tell**: EXPLAIN shows a full scan (or examined-rows ≫ returned-rows) on a
  query filtering by a WHERE column. Check the plan for every new query shape before shipping,
  not after the incident.
- **Over-indexing tell**: >5–6 indexes on a hot-write table. Every index is a write
  amplification — an insert pays once per index. Audit with the database's unused-index stats
  before adding the seventh; usually two of the six are dead.
- Unique constraints are business rules, not performance features. State them explicitly; the
  application-level "check then insert" race is not a substitute.

## Schema evolution

**Expand → migrate → contract.** Always three deploys minimum; each phase has a safety check
before the next starts.

1. **Expand**: add the new column/table/index, nullable or defaulted, unused. Old code ignores
   it (additive is free). Check: old release runs clean against the new schema.
2. **Migrate**: dual-write from new code; backfill old rows in bounded batches (throttled,
   resumable, idempotent — a 40M-row single-statement UPDATE is a lock incident). Check: a
   reconciliation query proves old and new agree (count + checksum) before proceeding.
3. **Contract**: remove old-column reads, then writes, then — one full release later — the
   column. Check: no release still in the deploy window reads the dropped column.

Hard rules:

- **NEVER rename in place.** A rename is expand (new name) + migrate + contract (old name).
  In-place renames break the previous release the instant the migration runs — rollback is
  now impossible without a second migration.
- **MODIFY/ALTER COLUMN restates the full definition.** MySQL `MODIFY COLUMN` silently drops
  `NOT NULL` and `DEFAULT` unless you re-declare them. A width change that quietly made the
  column nullable is a real production bug class. Restate everything, every time.
- **Every migration ships its rollback twin**, written and tested in the same PR. "We'll write
  the down-migration if we need it" means you'll write it during the incident.
- **Case-consistency rule**: a migration whose column case differs from the base DDL
  (`manual_risk_level` in the migration, `MANUAL_RISK_LEVEL` in base DDL) forks the fleet —
  fresh installs get one casing, upgraded environments get the other, and any case-sensitive
  code path breaks on exactly half of them. Real production bug class. Migrations copy casing
  from base DDL verbatim; a CI check that diffs migration column names against base DDL is
  cheap insurance.
- Migrations are forward-only in sequence, idempotent where possible (`IF NOT EXISTS`), and
  never edited after they've run anywhere beyond a laptop.

## Widths & types

- **Mirror columns share one declared width.** The same logical value stored in N tables
  (denormalized copies, staging tables, an audit table) declares the identical type and width
  everywhere. Drift (`VARCHAR(64)` here, `VARCHAR(32)` there) is silent truncation waiting for
  the first long value. Keep a single source of truth for shared column definitions and audit
  on every width change.
- **Money**: integer minor-units (cents) or `DECIMAL(p, s)` with an explicit scale. Never
  FLOAT/DOUBLE — binary floats cannot represent 0.10 and the reconciliation report will
  eventually be off by a cent, which in a financial domain is a sev-2. Store currency code
  beside every amount.
- **Timestamps**: UTC in storage, always. Timezone conversion happens at the presentation
  edge, using the user's zone, never the server's. A `DATETIME` whose zone is "whatever the
  app server was set to" is unrecoverable after the fact.
- **Enums**: string values with a CHECK constraint or lookup table. Never bare int magic
  (`status = 3`) — unreadable in every query, log, and data export, and renumbering is a data
  migration. Native DB enum types are acceptable only if the team knows their alteration cost;
  the string+CHECK form migrates additively.
- Booleans that will ever grow a third state start as a string status. `is_active` →
  `is_active_but_pending_review` is how boolean columns die.

## Anti-patterns — match and refuse

- **EAV "flexible schema"** (`entity_id, attribute_name, attribute_value`). A database without
  a schema is a pile with an API: no types, no constraints, no usable indexes, and every query
  is a self-join pyramid. If attributes are truly dynamic, use one typed JSON column for the
  overflow and promote any attribute you filter on to a real column.
- **JSON columns for data you filter, join, or index on.** JSON is for payloads you store and
  return whole. The first `WHERE json_extract(...)` in a hot query is the signal to promote
  that field to a column. JSON blobs also dodge every width/type/constraint rule above.
- **Unbounded LONGTEXT/BLOB for structured payloads.** Structured data gets columns; large
  opaque payloads get object storage with a reference row. A LONGTEXT column that "usually
  holds ~2KB of JSON" will one day hold 40MB and take the row cache with it.
- **Multi-writer tables without arbitration.** Two services upserting the same rows need a
  written rule: partitioned key ranges, single-writer-per-column, or last-write-wins with a
  version column — chosen and documented, not discovered.
- **ORM-generated schema as the source of truth.** The migration files are the schema; the ORM
  mapping conforms to them, not the reverse. Auto-sync/auto-migrate in production is a schema
  change nobody reviewed.
- **Nullable-everything.** A column that is logically required but declared nullable "to make
  the migration easy" moves the NOT NULL check into every reader forever. Do the backfill;
  add the constraint.
