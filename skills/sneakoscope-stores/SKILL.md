---
name: sneakoscope-stores
description: Use when designing or reviewing SQL/CQL queries, schemas, or data-store usage for a specific engine — BEFORE writing DDL/queries or while reviewing them. Covers query performance ("why is this query slow", EXPLAIN reading, sargability, pagination, locking), per-engine schema design, and store selection ("should this go in X or Y", "which database for this feature"). Engines: MySQL/MariaDB (InnoDB), Cassandra/ScyllaDB, StarRocks/ClickHouse-class OLAP columnar, Hive/Trino, Elasticsearch/OpenSearch. Trigger on "design this schema/table", "review this query", "this query is slow", "add an index", "partition key", "should we cache this", "ES mapping", "which store", or any PR touching DDL, DAO/repository query text, or pipeline persistence. Engine-specific + query-text-level counterpart of sneakoscope-architecture's cross-engine modeling doctrine (reference/data.md there) — that file owns keys, normalization, evolution, widths; this skill owns what the engine does with your bytes and your query text.
version: 1.0.0
user-invocable: true
argument-hint: "[queries · mysql|mariadb · cassandra|scylla · olap|starrocks|hive|trino · elasticsearch|search] [target]"
license: Apache 2.0
---

Per-engine data-store doctrine: what each engine rewards, what it punishes, and the query-text
tells that predict incidents. Cross-engine modeling (identity, normalization, evolution,
consistency) lives in `sneakoscope-architecture/reference/data.md` — read it for schema *shape*;
read this for engine *behavior*.

## Setup — you MUST do these steps before proceeding

1. **Identify the ACTUAL engine and version before opining.** "SQL" is not an engine. MySQL 5.7
   vs 8.0, MariaDB vs MySQL, Cassandra vs ScyllaDB, Hive vs Trino-on-Hive all diverge on the
   exact points that matter (optimizer, DDL semantics, defaults). Find it in the connection
   config, docker-compose, Helm values, or infra docs — never assume from the query dialect.
2. **Read the real schema, not the ORM model.** `SHOW CREATE TABLE` / migration DDL / `DESCRIBE`
   output is the truth; the ORM mapping is a hopeful summary that omits collations, index
   definitions, and column defaults — the three things most advice hinges on.
3. **If the user invoked a sub-command, read the matching reference. Non-optional.**
   - `queries` (any SQL query design/review, "why slow") → `reference/queries.md`
   - `mysql` / `mariadb` → `reference/relational.md` (read queries.md too for query text)
   - `cassandra` / `scylla` → `reference/wide-column.md`
   - `olap` / `starrocks` / `clickhouse` / `hive` / `trino` → `reference/olap.md`
   - `elasticsearch` / `opensearch` / `search` → `reference/search.md`
4. For store-selection questions, answer from the doctrine below; the per-engine references
   justify the boundaries.

## Store-selection doctrine

Concrete, checkable, non-negotiable unless the user explicitly overrides.

- **Point read/write of a business entity → row-store OLTP** (MySQL/MariaDB class). It has the
  transactions, constraints, and secondary indexes the entity lifecycle needs.
- **Aggregation scans over many rows → OLAP columnar** (StarRocks/ClickHouse class). Never run
  analytics on the OLTP primary: one dashboard query holding a scan can starve the buffer pool
  and lock queue that serves users.
- **Never serve user-facing point lookups from a scan engine.** A columnar store answering
  "get me row X" per user click is paying full-segment-scan tax per request — that lookup
  belongs in the row store, with the OLAP copy fed asynchronously.
- **Search engine is an INDEX, not a source of truth.** Every ES/OpenSearch document must be
  rebuildable from a store of record, and the reindex path must exist and be tested. If a
  field lives only in ES, that's a data-loss design, not a design.
- **Wide-column (Cassandra/Scylla) only for known-query-shape, high-write workloads.** You
  model tables per query there; if the product still discovers new query shapes monthly, it's
  too early — stay relational.
- **Cache is an optimization, never a source of truth.** Anything in Redis/EhCache must be
  reconstructible from a store below it, with an explicit invalidation owner (the canonical
  change-event writer invalidates, not every reader guessing TTLs).
- **One store of record per entity.** Copies in OLAP/search/cache each need the sync mechanism
  + drift check from data.md's denormalization rule — a copy without both is a future
  "why do these two screens disagree" ticket.
- **No new engine when an existing one covers the measured need.** Every additional store is
  an on-call page, a backup regime, and a consistency seam. "Postgres/MySQL can't do this" is
  a claim that cites a measurement, not a vibe.

## Rules of engagement

- **Performance claims cite an EXPLAIN / query profile**, not intuition. "This will be slow"
  without a plan or a row-count estimate is labeled an assumption.
- **Advice names the engine and version it applies to.** Cross-engine folklore ("indexes make
  everything fast", "denormalize for NoSQL") is where incidents come from.
- **Review flows never edit** — findings with `file:line` + the tell that fired, ranked by
  blast radius (data loss > silent wrong results > outage > latency).
- Schema evolution, migration phasing, and width/type rules: defer to
  `sneakoscope-architecture/reference/data.md` — restate its conclusion, don't fork it.
