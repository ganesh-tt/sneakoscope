# DAG Orchestration Doctrine

Rules for DAG-orchestrated pipelines — JSON/YAML-config DAGs, Airflow-style schedulers, or
custom orchestrators (jobsList + jobsDetail shapes). The DAG config is production code that
never gets a compiler: review it like code, validate it like input. Every rule ships with its
TELL. Evidence: the config file `path:key`, the orchestrator's run history, or job logs.

## DAG wiring correctness

- **Every reference resolves — mechanically checked, not eyeballed.** Job indexes, output
  indexes, task ids, dataset names: a validator (CI step or pre-deploy check) asserts every
  `dependencies: [{jobIndex, outputIndex}]`-style edge points at an existing job and an output
  that job actually produces. Index-based wiring is one insertion away from silently shifting
  every downstream edge.
  - TELL: a job reading the *wrong upstream's* output after an unrelated job was added to the
    list — off-by-one in index wiring, no error anywhere.
- **No orphan jobs, no dangling deps.** A job nothing depends on and nothing consumes is either
  dead (delete) or a hidden side-effect writer (document its output as a real edge). A declared
  dependency on a job not in the DAG fails at parse time, loudly, before the run starts.
- The DAG is acyclic and its parallelism is intentional: jobs with no edge between them WILL run
  concurrently — if two jobs write the same table/path, that's a race the wiring must serialize.
  TELL: intermittent corrupt/partial output that "fixes itself on re-run".
- Fan-in semantics stated: does a join-point job run on ALL parents succeeded, or ANY? The
  orchestrator has a default; name it and check it — an any-parent default turns one upstream
  failure into a run over half the inputs.

## Config hygiene

- **NO credentials in pipeline configs. Ever.** Secret indirection only: vault path, secret
  name, env var reference — resolved at runtime by the executor. A password in a JSON DAG is in
  git history forever; grep for it in review (`password|secret|token|key.*=`), don't trust the diff.
- **No hardcoded IPs or hostnames.** DNS names or parameters. TELL: a pipeline that breaks when
  a database fails over or a pod reschedules — the config memorized an address.
- **Environment-specific values are parameterized**, never branched (`if env == prod` inside a
  config is two pipelines wearing one file). One config + per-env parameter/overlay set; the
  diff between environments is enumerable in one place.
- Every tunable in the config carries units in its name or schema (`timeoutMs`, `batchSizeRows`).
  A bare `"timeout": 300` has caused both 300ms and 300-minute incidents.
- Business logic (filter expressions, mapping rules, thresholds that change answers) embedded in
  DAG config is untested code — pull it into the job's tested code or versioned reference data.
  Config decides *cost and wiring*, not *answers* (same rule as spark.md).

## Idempotent re-run — per job AND per DAG

- **Partial-failure restart semantics are written down per job**: safe to re-run alone /
  requires upstream re-run / requires cleanup first. The 3am operator restarting job 7 of 12
  must not need to reverse-engineer this from the code.
  - TELL: a runbook (or tribal knowledge) saying "if it fails at step X, first go delete Y" —
    that's a missing cleanup-on-start in job X, not an ops procedure.
- Each job is idempotent standalone (overwrite-by-partition, keyed upsert — core doctrine 1–2),
  AND the DAG is idempotent as a whole: re-running the full DAG for a period produces the same
  end state, including side-effect jobs (notifications, downstream triggers) which must be
  keyed/deduped or gated on first-run.
- **Run identity is explicit**: every run carries a logical date / batch id that flows into
  every job's output partitioning. "Now()" inside a job breaks re-runs by definition — the
  re-run of Tuesday must write Tuesday's partitions, not Friday's.
  - TELL: a job whose output path or filter contains `current_date` instead of the run parameter.
- Retries at the orchestrator layer are bounded with backoff, and only for jobs declared
  idempotent. Orchestrator-retrying a non-idempotent job is duplication-as-a-service.

## Backfill doctrine

Backfills are where pipelines meet their re-run story at scale. A backfill is:

- **Bounded**: explicit start/end, enumerated partitions, and a written blast-radius statement
  (which tables/partitions will be overwritten, which downstream consumers see churn).
- **Checkpointed**: progress recorded per partition/chunk so a failure at 80% resumes at 80%,
  not at 0%. A 3-week backfill that restarts from scratch on one failure never finishes —
  TELL: backfill "attempt 4" in the run history.
- **Rate-limited**: a backfill shares sources, sinks, and cluster with the steady-state
  pipeline. Uncapped, it starves tonight's production run and DDoSes the shared database.
  Concurrency/rate cap stated with a number.
- **Verified**: per-partition row counts (and a checksum/aggregate spot-check) compared against
  expectation or the pre-backfill baseline, recorded. A backfill without verification is a
  hope-fill.
- **Separately configured**: backfill parallelism/rate/window is its own parameter set or
  overlay — never achieved by hand-editing the steady-state config and (maybe) reverting it.
  TELL: git history showing the prod config toggled and restored around a date — one day someone forgets the restore.
- Streaming note: backfilling through a processing-time pipeline produces different answers
  than the original run (see flink.md) — event-time or waiver first.

## Snapshot / versioned-config discipline

- **Wholesale-copied config versions drift.** `v1/`, `v2/`, `incremental/v6.3/` directories of
  full pipeline copies guarantee a fix lands in one copy and not the others. Single source +
  per-version/per-env overlays; the delta between versions is a readable diff, not a
  vimdiff-archaeology session.
  - TELL: the **byte-identical-duplicate tell** — two config files that are 95% identical.
    Either the 5% is the real overlay (extract it) or the twin is dead (delete it). Run the
    check mechanically (`diff`/hash) across the config tree in review.
- Know which copy is LIVE. In snapshot-heavy repos, top-level = live and `v*/` = historical (or
  the inverse) — the convention is written at the config root, because the default assumption
  will be wrong exactly once, in production.
- Config changes deploy like code: reviewed, versioned, and attributable to a run ("run 4812
  used config sha abc123"). An orchestrator that reads mutable config at run-start with no
  recorded version makes every incident unreproducible.

## Connector flags & defaults

- **A default that silently returns an empty result is a data-loss bug wearing a config
  costume.** A connector flag whose false/absent value yields an empty DataFrame lets the whole
  DAG "succeed" on nothing: joins produce zero rows, output overwrites good data with empty
  partitions, all green.
  - TELL: run succeeded, duration suspiciously short, output row count ≈ 0 — and nobody was paged.
- Defaults fail loud: an unconfigured connector **errors**, it does not no-op. If skip-a-source
  is a legitimate mode, it's an explicit `enabled: false` with an INFO log and a metric, not an
  absence.
- Flag names match their semantics. A `scyllaConnectFlag` that actually gates a MariaDB read is
  a trap for every future reader — rename or die by triage-time.
- Every boolean flag's default is stated in the config schema/docs, not discovered in the
  code's parameter list — positional/implicit defaults are where "why is this table empty"
  investigations go to spend a week.

## Monitoring floor

The minimum below which a DAG is not shippable:

- **Per-job duration + rows-in/rows-out metrics**, recorded per run. Duration alone says busy;
  row counts say whether it did anything.
- **Row-count anomaly detection between runs**: today's output vs trailing baseline (e.g. ±3σ
  or a stated %) → alert. Catches the empty-connector class, upstream truncation, and drift the
  schema check can't see. This is the single highest-value pipeline alert; a DAG without it
  fails silently until a downstream customer asks where the data went.
- **Freshness/lag alert per output**: "partition for date D exists by HH:MM or page" — per
  consumed output, not per DAG. A DAG that runs but skips one output is green on the DAG panel
  and stale to the consumer.
- Failure alerts route to a named owner per pipeline. An alert channel nobody owns is a log file
  with a notification sound.
- The orchestrator's own health is monitored separately (scheduler up, queue not stuck): a dead
  scheduler produces zero failures — TELL: no alerts at all AND no runs; alert on absence-of-run,
  not only on failed-run.

## Anti-patterns — reject on sight

| Anti-pattern | Why it's fatal | Replace with |
|---|---|---|
| Index-wired deps with no validator | silent off-by-one on any insertion | CI reference-resolution check |
| Credentials/hostnames inline in DAG config | leaked forever; env-fragile | secret indirection + DNS/params |
| `current_date` inside job logic | re-runs write the wrong partitions | run-parameter (logical date) everywhere |
| Backfill by editing prod config | forgotten revert = misconfigured steady state | separate backfill parameter set |
| Full-copy version directories | fixes land in one copy | single source + overlays; diff-check twins |
| Connector default = empty output | DAG succeeds on nothing, overwrites good data | fail-loud default + row-count anomaly alert |
| Orchestrator retries on non-idempotent jobs | duplication-as-a-service | idempotency first, then retries |
| "Green DAG" as the only signal | skipped/empty outputs invisible | per-output freshness + row-count alerts |
| Runbook step "first delete Y" | missing cleanup-on-start in the job | job cleans/overwrites its own outputs |
