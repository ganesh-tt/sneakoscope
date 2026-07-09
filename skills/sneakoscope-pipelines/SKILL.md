---
name: sneakoscope-pipelines
description: Use when the user wants to design, review, debug, harden, or estimate data-pipeline work — Spark jobs, Flink/streaming jobs, batch pipelines, DAG-orchestrated (JSON/YAML/Airflow-style) pipelines, UDFs, backfills, or reprocessing. Covers pipeline performance (skew, shuffle, OOM, small files), exactly-once vs at-least-once semantics, checkpointing, watermarks, state sizing, idempotent re-runs, backfill planning, and pipeline-config hygiene. Trigger on "design/review this spark job", "debug this flink job", "why is this stage slow/OOMing", "plan the backfill", "is this pipeline idempotent", "review this DAG config", "exactly-once?", "checkpoint/watermark question", or any batch/streaming data-movement work that deserves doctrine before code. The data-pipeline counterpart of sneakoscope-architecture. Not for reviewing already-written non-pipeline code diffs (use pr-gate for that).
version: 1.0.0
user-invocable: true
argument-hint: "[spark · flink|streaming · dag|orchestration|backfill] [target]"
license: Apache 2.0
---

Designs and reviews production-grade data pipelines: idempotent by construction, loud on drift,
sized with numbers. A pipeline is a contract between yesterday's data and tomorrow's re-run —
design the re-run first.

## Setup — you MUST do these steps before proceeding

1. **Read the actual job before opining.** At minimum: the job/operator code, its submit/deploy
   config (memory, parallelism, shuffle partitions, checkpoint settings), and the input AND
   output schemas (DDL, Avro/Protobuf schema, or a sampled row). Advice written from the job's
   name is fiction. Cite `file:line` or config key for every claim about the existing pipeline.
2. **Pick the register — say which one:**
   - **Batch (Spark-style)**: throughput + reprocessability dominate; latency is per-run, not per-record.
   - **Streaming (Flink-style)**: state size + watermark correctness + recovery time dominate.
   - **DAG-orchestrated**: wiring correctness + per-job re-run semantics + config hygiene dominate.
   The register weights every trade-off below.
3. **If the user's ask maps to a sub-command, read the reference next. Non-optional.**
   - `spark` / batch performance / UDF / caching / shuffle → `reference/spark.md`
   - `flink` / streaming / state / watermark / checkpoint / exactly-once → `reference/flink.md`
   - `dag` / orchestration / backfill / pipeline-config / re-run → `reference/orchestration.md`
   Mixed asks (a Spark job inside a DAG) read both relevant references.
4. **State the volumes.** Rows/day, bytes/partition, peak records/sec, state size — with a number
   and a source, or explicitly labeled ASSUMPTION in its own section. A pipeline design without
   volumes is a shape, not a design.

## Core doctrine — cross-cutting, non-negotiable unless the user overrides

1. **Every pipeline is idempotent under re-run.** Reprocessing is not exceptional — it's Tuesday.
   A job whose second run duplicates, double-counts, or corrupts is broken today, not "on retry".
   The re-run story (what gets overwritten, what dedupes, what's keyed) is part of the design.
2. **Overwrite by partition, never append-blind.** Append-mode output + re-run = duplicates.
   Output is written to a deterministic partition (date, batch id) and the re-run replaces that
   partition atomically. If the sink can't do that, the consumer dedupes on an explicit key — say which.
3. **Every job states its input contract and fails loudly on schema drift.** Coercing a missing
   column to null, silently casting, or `select *`-ing whatever arrived is how silent-corruption
   ships. Assert expected columns + types at read time; a drift is a failed run with a named error,
   not a quieter output.
4. **No `collect()` / full materialization on unbounded or unsized data.** Anything pulled to the
   driver / a single node states its max size with a number. "It's small" is an assumption — label it.
5. **Volumes carry numbers or the label ASSUMPTION.** "Large table", "high throughput", "should
   fit in memory" are not inputs to a partition count, a broadcast decision, or a state plan.
6. **At-least-once is the default reality; exactly-once is at-least-once + idempotent/keyed sinks.**
   Any design claiming exactly-once names the mechanism end-to-end (commit protocol, transactional
   sink, dedupe key) — checkpointing alone is not it.
7. **Late and bad data have a named destination.** Side-output, quarantine table, DLQ — with an
   owner and a replay procedure. Silent drop is a product decision nobody made.
8. **Partial-failure semantics are stated per job.** Which jobs/stages are safe to re-run alone,
   which require the whole DAG, which need a cleanup step first. "Restart it and see" is not semantics.
9. **Business logic lives in code with tests; operational knobs live in config.** Parallelism,
   memory, shuffle partitions, checkpoint interval = config. A join condition or a risk rule in a
   JSON pipeline file is a programming language without tests.
10. **No credentials, hostnames, or environment literals in pipeline configs.** Secret indirection
    and parameterization, always — a config that only works in prod is a config that fails silently in staging.
11. **Every output has a freshness/row-count floor.** Per-run duration, row counts in/out, and a
    staleness alert — a pipeline with no anomaly signal fails silently for days and is discovered
    by a downstream customer.
12. **The fewest moving parts that meet the stated volume.** No new engine/queue/format for a job
    the existing stack covers within measured limits; design for 10× current volume, write the
    100× escape hatch as a paragraph, not as infrastructure.

## Rules of engagement

- **Evidence discipline**: claims about the running pipeline cite `file:line`, config key, or a
  Spark UI / Flink UI observation. Claims about data ("this key is hot") cite a count or are labeled assumption.
- **Failure design is part of the design**: re-run, skew, OOM, late-data, and schema-drift answers
  are the pipeline equivalent of timeout/retry/idempotency — a design without them is unfinished.
- **Review never edits.** Design/review flows produce findings and docs; only the user-approved
  implementation step touches code.
- Findings rank P0 (data loss/corruption/duplication) → P1 (outage amplifier: unbounded state,
  skew straggler, retry loop) → P2 (silent degradation) → P3 (observability gap only).
