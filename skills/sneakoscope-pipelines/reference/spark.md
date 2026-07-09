# Spark Doctrine

Batch Spark (and Spark-on-YARN/K8s) design and review rules. Every rule ships with its TELL —
the observable symptom that says the rule is being violated right now. Evidence for any claim:
Spark UI stage/task view, `explain()` plan, or `file:line` in job code/config.

## Partitioning & shuffle

- **Know your partition count — as a number, derived from bytes.** Target 128–512MB per
  partition post-shuffle. `total_shuffle_bytes / 256MB` is the starting count; the default 200
  (`spark.sql.shuffle.partitions`) is right only by coincidence.
  - TELL (too few): tasks each processing GBs, long GC, spill-to-disk metrics climbing, executor OOM.
  - TELL (too many): thousands of sub-second tasks, scheduler overhead dominating stage time,
    and the **small-files problem** downstream — an output directory of 10k × 2MB files that
    murders the next reader's listing + open cost. Fix at write time, not with a compaction job you'll forget.
- **`repartition` vs `coalesce` — they are not synonyms.** `repartition(n)` = full shuffle,
  even distribution, can increase n. `coalesce(n)` = narrow merge of existing partitions, no
  shuffle, can only decrease — and **collapses parallelism upstream**: a `coalesce(1)` before a
  write can drag the whole preceding stage onto one task.
  - TELL: "we added coalesce to fix small files and the job got 10× slower" — the coalesce
    swallowed the parallelism of the stage before it. Use `repartition` there, or coalesce after an action boundary.
- **Partition the output by the read pattern, not the write convenience.** Daily-read data
  partitioned by `date`; if consumers filter by `client_id`, say why it isn't a partition/bucket
  column (cardinality? skew?) — with numbers.
- AQE (`spark.sql.adaptive.enabled`) is the default assumption on Spark 3.x — verify it's on
  before hand-tuning partition counts; verify it's honored (some clusters pin it off).

## Skew

- **The one-hot-key tell**: a stage where 199 tasks finish in seconds and 1 task runs for an
  hour. That task holds the null key, the default value, or the mega-customer. Confirm with a
  `GROUP BY key ORDER BY count DESC LIMIT 10` before designing the fix — never fix skew you haven't counted.
- Fix ladder, cheapest first:
  1. **Filter/split the hot key** if it's a null/sentinel (nulls in join keys join with nothing useful — pre-filter).
  2. **AQE skew-join** (`spark.sql.adaptive.skewJoin.enabled`) — free on 3.x, check it fired in the plan.
  3. **Salting**: append `rand % N` to the hot side's key, explode the other side ×N, join, strip.
     N is derived from the measured skew ratio, not vibes.
  4. **Broadcast the small side** — skew in a shuffle join vanishes when there's no shuffle.
- TELL of unfixed skew shipped to prod: job duration variance run-to-run with identical volume —
  the hot key's size fluctuates and the straggler with it.

## Broadcast joins

- **Broadcast only what you've sized.** `spark.sql.autoBroadcastJoinThreshold` (default 10MB) is
  a plan-time *estimate* threshold — stats can be stale/absent and Spark will happily broadcast a
  "10MB" table that decompresses to 4GB.
  - TELL: driver OOM or `Broadcast exceeds spark.driver.maxResultSize` on a join that "was fine last month" — the small table grew.
- **Never broadcast an unbounded input** (a table with no size ceiling by construction:
  customer lists, event dims that grow with the business). A broadcast is a bet the table stays
  small; the design states the ceiling and what enforces it.
- Explicit `broadcast()` hints beat threshold roulette for known-small dims — but a hint is a
  size assertion; cite the size.
- **Broadcast variables are read-only.** Mutating a broadcast collection inside a task mutates a
  local copy per executor — no error, divergent results. TELL: results differ across runs/executor counts.

## Serialization

- **The task-not-serializable tell**: `org.apache.spark.SparkException: Task not serializable`,
  usually because a lambda captured `this` (the enclosing class), a `SparkContext`/`SparkSession`,
  a DB connection, or a logger. Fix: extract the needed field to a `val` before the closure, or
  make the helper an `object`. Never fix by marking the whole service class `Serializable` —
  you just shipped your connection pool to 200 executors.
- Connections/clients inside tasks: create per-partition (`mapPartitions`, lazy singleton per
  executor), never per-record, never captured from the driver.
- Kryo (`spark.serializer=KryoSerializer`) + registered classes for shuffle-heavy jobs with
  custom types — measurable shuffle-bytes win; register or `registrationRequired=true` so
  unregistered classes fail loudly instead of falling back to slow paths.

## Caching discipline

- **Cache only if BOTH hold**: the dataset is consumed by ≥2 actions, AND it fits (memory ×
  executors, measured — check the Storage tab, not hoped). One-action caches are pure overhead.
  - TELL: Storage tab shows fraction cached < 100% — you're paying cache cost and recomputing anyway.
- **Every `cache()` has a paired `unpersist()`** at the end of its live range. Leaked caches
  evict each other under memory pressure and the job gets slower the longer it runs.
- **Cache before checkpoint.** `checkpoint()` recomputes the lineage to write it; without a
  preceding `cache()`, the whole upstream runs twice. TELL: a checkpointed job whose upstream stage appears twice in the UI.
- `count()` as a "materialize the cache" idiom is fine — but say that's what it's for; a bare
  count in the middle of a pipeline reads as debris and gets deleted.

## UDF discipline

- **Built-ins first, always.** Catalyst cannot see into a UDF: no predicate pushdown, no
  constant folding, no null-skip, no codegen. 90% of UDFs are a `when/coalesce/regexp_extract`
  written the long way. TELL in review: a UDF whose body is expressible in `F.*` functions — reject it.
- **Python UDFs pay per-row serialization** (JVM→Python→JVM). If Python is genuinely needed,
  `pandas_udf` (Arrow, vectorized) — order-of-magnitude cheaper. A row-at-a-time Python UDF on a
  billion-row frame is a design bug, not a tuning problem.
- **UDFs handle null explicitly.** Built-ins skip nulls; your UDF gets them raw. A UDF that
  NPEs on null kills the task, the retry gets the same row, ×4 attempts, stage dead.
  TELL: task failures all citing the same partition — one poison row.
- UDFs are pure functions of their inputs. No I/O, no shared mutable state, no wall-clock reads
  (task retries would produce divergent rows — non-deterministic UDFs also break AQE reuse; mark
  `asNondeterministic()` if unavoidable, and justify).

## Case-sensitivity traps

- **`spark.sql.caseSensitive=false` (the default) makes `withColumn("col", …) REPLACE
  case-insensitively.** `withColumn("MANUAL_RISK_LEVEL", lit(null))` silently overwrites an
  existing `manual_risk_level` — a real silent-corruption class, no error, no log.
  - TELL: a column "mysteriously null after the transform" only on environments whose DDL casing
    differs (migrated vs fresh-install schemas).
- Any code that checks column existence (`'X' in df.columns`) is doing a **case-sensitive**
  check against a case-insensitive engine — resolve case-insensitively at every boundary where
  schemas come from external DDL. One canonical `resolve(df, name)` helper; ban raw `in df.columns` in review.
- Normalize column casing once at ingest (contract check, rule 3 of core doctrine) so the rest
  of the job works against one casing.

## Memory / OOM triage order

Diagnose in this order — each step's tell is distinct; skipping to "add memory" hides the bug:

1. **Driver or executor?** Driver OOM → `collect()`, oversized broadcast, giant task-result
   (`maxResultSize`), or plan explosion. Executor OOM → skew, partition too big, cache pressure, UDF leaks.
2. **collect/broadcast** (driver): find every materialization to the driver, size each with a number.
3. **Skew** (executor): one task's shuffle-read dwarfs the others in the stage view → skew section above.
4. **Partition size** (executor, uniform): all tasks big → raise partition count; check for
   `coalesce` upstream collapsing parallelism.
5. Only after 1–4: memory config (`executor.memory`, `memoryOverhead` — off-heap/PySpark lives in
   overhead; TELL: container killed by YARN/K8s ("exit 137"/OOMKilled) while heap looked fine → raise overhead, not heap).

## Write semantics

- **Idempotent overwrite by partition**: dynamic partition overwrite
  (`spark.sql.sources.partitionOverwriteMode=dynamic`, or table-format equivalent) replaces
  exactly the partitions the run produced. Static overwrite mode + partitioned write TRUNCATES
  THE WHOLE TABLE first — TELL: "the backfill for March deleted January".
- **Exactly-once via commit protocols, not hope.** Table formats with atomic commit (Iceberg/
  Delta/Hive-ACID) or a staging-dir + atomic-rename/pointer-swap pattern. A job that writes
  files in place dies mid-write and leaves a half-output the next reader consumes as truth.
- **`_SUCCESS` markers (or the format's snapshot) gate downstream reads.** A consumer that
  lists a directory the producer is mid-writing reads a partial dataset — every downstream
  states what it keys readiness on.
- Speculative execution + non-idempotent sinks (JDBC upserts without keys, external API calls in
  `foreachPartition`) = duplicate writes by design. Either the sink is keyed-idempotent or
  speculation is off for that job — stated, not assumed.

## Config vs code

- **Config**: shuffle partitions, memory, cores, AQE flags, broadcast threshold, paths,
  connection targets — per-environment, no redeploy to tune.
- **Code**: joins, filters, business rules, schema contracts — tested, reviewed, versioned.
- TELL of the line being crossed: a config value that changes the *answer* (a threshold that
  filters rows, a mapping table inlined in JSON) rather than the *cost*. Answer-changing values
  are code or reference data with lineage — never a tuning knob.
- Every non-default config value in the job carries a one-line why. An unexplained
  `spark.sql.shuffle.partitions=873` outlives the incident that set it by years.

## Anti-patterns — reject on sight

| Anti-pattern | Why it's fatal | Replace with |
|---|---|---|
| `collect()` then loop on driver | driver OOM at volume; single-threaded | `mapPartitions` / join / window fn |
| `count()` per iteration for logging | full job re-execution per count (no cache) | accumulators or one counted checkpoint |
| Row-at-a-time Python UDF on big frames | per-row ser/deser tax | built-ins, else `pandas_udf` |
| `coalesce(1)` to "make one file" | serializes the whole job onto one task | repartition(1) post-aggregation, or downstream compaction with stated cost |
| Append-mode + scheduled re-runs | duplicates on every retry | partition overwrite (core doctrine 2) |
| `dropDuplicates()` with no subset on wide frames | full-row shuffle + hides the real key question | dedupe on the stated business key |
| Schema inferred from data (`inferSchema`, schemaless JSON read) in prod | drift becomes silent coercion | explicit schema + contract assert |
| Catch-all `try/except` around the transform returning empty DF | data loss wearing an uptime costume | fail the run; quarantine bad rows to a named sink |
| Joining on columns of different types (silent cast) | nulls out matches silently | explicit cast + count-match assertion |
| One giant job "so we shuffle once" | unrecoverable; 6h in, one failure = 6h retry | stage boundaries at durable checkpoints (see orchestration.md) |
