---
name: sneakoscope-scripts
description: Use when writing or reviewing shell scripts, CI steps, cron jobs, migration/backfill runners, kubectl/helm operations, or any operational glue script — the artifacts AI generates constantly and reviews never. Covers bash safety (strict mode, quoting, traps), idempotency, dry-run discipline, k8s/helm operational hygiene, and secrets handling in scripts. Trigger on "write a script to", "add a CI step", "cron job", "runbook", "kubectl/helm command sequence", or any .sh/.bash file being created or reviewed.
version: 1.0.0
user-invocable: true
license: Apache 2.0
---

Operational scripts are production code with worse tooling and better access. They get the same
bar: evidence, idempotency, and a failure story — in one file, because scripts are small and
doctrine should be too.

## Bash floor (non-negotiable in every generated script)

- `set -euo pipefail` first line after the shebang. Deviations (a step that may fail) are
  explicit: `cmd || true` with a comment saying why failure is acceptable.
- **Quote every expansion**: `"$var"`, `"$@"`, `"$(cmd)"`. Unquoted `$var` is a word-splitting
  bug that waits for a filename with a space. Arrays for building command lines, never a string
  you later re-split.
- `[[ ]]` over `[ ]`; `$( )` over backticks; `printf` over `echo` for data (echo mangles `-n`,
  escapes, varies by shell).
- Variables that must exist: `${VAR:?message}` at the top — fail at line 5, not at the `rm` on
  line 80.
- `trap 'cleanup' EXIT` for anything that creates temp files, port-forwards, locks, or partial
  state. Cleanup runs on failure too — that's the point.
- `mktemp` for temp files, never predictable `/tmp/foo.$$` paths (symlink attacks, collisions).
- shellcheck-clean or each warning suppressed with a directive + reason. A script that's never
  seen shellcheck isn't reviewed.
- Portability is a decision: `#!/usr/bin/env bash` and bash-isms are fine — but then don't call
  it `sh`. macOS ships ancient bash and BSD userland (`sed -i ''`, no `timeout`); name the
  target platform in the header comment.

## Behavior doctrine

- **Idempotent by default**: running twice must be safe or the script refuses the second run
  (lockfile/marker). "It'll only be run once" is how it gets run twice.
- **Dry-run for anything destructive**: `--dry-run`/`DRY_RUN=1` prints the mutating commands
  instead of executing. Destructive = deletes, writes to shared state, k8s mutations, DB DML.
  The dry-run path is the default when the script's args are ambiguous.
- **Fail loud, halfway matters**: after a partial failure, the script's output must say what
  completed and what didn't — a `set -e` death mid-loop with no context is an incident
  investigation. For loops over items: collect failures, report a summary, exit non-zero.
- **Confirmation gates scale with blast radius**: prod-touching scripts echo the target
  (cluster/namespace/DB/host) and require typed confirmation or an explicit `--yes`.
- **Logs are the UI**: timestamped, to stderr for diagnostics and stdout for data (so pipes
  work). Every mutating action logged with its target before it runs.
- **No secrets in scripts**: no inline passwords/tokens — env vars or secret-manager lookups;
  no secrets in `echo`/`set -x` output (mask before tracing); never commit a script with a
  credential "temporarily".

## k8s / helm hygiene

- Every `kubectl` call carries explicit `--context` and `-n <namespace>` — inherited context is
  how prod gets the staging command. Print both before mutating.
- No `kubectl edit/patch/delete` against shared environments in scripts — cluster changes go
  through the chart/manifest repo (GitOps); scripts that must mutate directly say why in a
  header comment and log the diff.
- `helm upgrade` with pinned chart version + `--timeout` + post-upgrade verification (rollout
  status + one functional check) — an exit-0 helm command is not a landed deploy.
- Rollback story stated in the script header: what to run when this goes wrong.
- Jobs/CronJobs the script creates: `backoffLimit` bounded, `ttlSecondsAfterFinished` set,
  resource requests/limits present — an unbounded retrying Job is a cluster-eating gremlin.

## SQL-from-scripts

- Scripts running DML/DDL: transaction + row-count echo (`SELECT ROW_COUNT()`), chunked
  UPDATE/DELETE with LIMIT loops on big tables, and a WHERE clause reviewed by a second pair of
  eyes — the unbounded-UPDATE incident class. Engine-level doctrine: see `sneakoscope-stores`.

## Review checklist (when reviewing, not writing)

1. strict mode? quoting? shellcheck?
2. what happens on re-run? on Ctrl-C halfway? (trap/cleanup/idempotency)
3. what does it print before mutating? (target, dry-run, confirmation)
4. secrets: any inline? any leaked via trace/logs?
5. k8s: context+namespace explicit? GitOps bypassed without justification?
6. exit codes honest? (a `|| true` swallowing the one failure that matters)

Findings cite `file:line`; propose-only when reviewing; classification P0 (data
loss/destructive-unsafe/secret leak) → P3 (style).
