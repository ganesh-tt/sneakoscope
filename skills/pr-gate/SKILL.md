---
name: pr-gate
description: Single review gate to run after any dev work is done — validate your own generated diff/PR before (or right after) opening it, OR review a teammate's PR. Orchestrates every review lens over the changed files — correctness (/code-review), per-language idioms/anti-patterns/FP (Scala/Python/JS-TS optimizers), over-engineering & AI-slop (/ponytail-review), UI/UX anomalies on frontend diffs (impeccable + frontend-design) — plus repo conventions (PR title, target branch, tests, secrets), and returns ONE verdict — SHIP / FIX-FIRST / BLOCKED — with findings ranked by severity. Use whenever the user says "review this PR", "review my teammate's PR", "check my PR", "review my diff before I open it", "validate the PR", "gate this", "review <PR#/URL>", "review code for slop / design patterns / UI/UX", "is this ready for review", or runs /pr-gate. Works on the local branch diff when no PR is given, or on a GitHub PR when a number/URL is given.
---

# PR Gate — one review pass after dev-done

## What this is

You did the work (or a teammate did). Before it merges, this runs every review lens you have over **only the changed lines**, then gives one verdict. It does not reinvent review logic — it routes the diff to skills/tools that already exist:

| Lens | Runs | Finds |
|------|------|-------|
| Correctness | `/code-review` (built-in) | bugs, security, logic errors, missing edge cases |
| Scala quality | `scala-code-optimizer` skill | Scala 3 idioms, anti-patterns, FP, tail-rec, perf |
| Python quality | `python-code-optimizer` skill | PEP idioms, anti-patterns, FP, typing, perf |
| JS/TS quality | `js-code-optimizer` skill | ES/TS idioms, anti-patterns, FP+async, types, perf |
| Anti-slop | `/ponytail-review` | over-engineering, reinvented stdlib, dead abstraction |
| UI/UX (FE diffs only) | `impeccable` + `frontend-design` | visual hierarchy, a11y, responsive, anti-patterns, UX-copy, empty/error states |
| Test quality (when diff touches tests OR adds logic) | `test-quality-audit` skill | tautological/hardcoded/mock-echo tests, verify-only, can-never-fail, missing behaviour coverage |
| SQL / queries (diff touches .sql or embedded SQL) | `sneakoscope-stores` skill (if available) | SELECT *, sargability, OFFSET-depth, injection, engine-specific traps |
| Pipelines (Spark/Flink/DAG-config diffs) | `sneakoscope-pipelines` skill (if available) | shuffle/skew/broadcast, state/watermark/checkpoint, DAG wiring, creds-in-config |
| Scripts (.sh / CI / k8s ops diffs) | `sneakoscope-scripts` skill (if available) | strict mode, quoting, idempotency, dry-run, secret leaks, kubectl context/namespace |
| Conventions | this skill | PR title, target branch, tests present, no secrets/debug |

## Mode detection

- **No argument** → self-review the current branch against its base. Base = the branch the PR targets (the repo's release/integration branch — check `CLAUDE.md` or the default branch). Confirm base with the user if ambiguous.
- **Argument is a PR number or GitHub URL** (`/pr-gate 10746`, `/pr-gate https://github.com/.../pull/10746`) → review that PR.
- **Argument is a branch name** → diff that branch against its base.

## Workflow — do these in order

### 1. Establish the diff and scope
- Self / branch mode: `git merge-base HEAD <base>` then `git diff <merge-base>..HEAD --stat` and capture the changed file list. Read the full diff with `git diff <merge-base>..HEAD`.
- PR mode: `gh pr view <n> --json title,baseRefName,headRefName,files,body,isDraft` and `gh pr diff <n>`. Do **not** check out the PR branch unless you must run the app — read the diff.
- Build a changed-file list bucketed by extension: `.scala` → Scala, `.py` → Python, `.js/.jsx/.ts/.tsx` → JS/TS, everything else → other. **ponytail: only review changed files, never the whole repo.**
- If the diff is huge (>40 files or >2k changed lines), say so and offer to gate by module; don't silently sample.

### 2. Correctness pass (always)
Run `/code-review` scoped to this diff. This is the pass that blocks merges — a real bug is BLOCKED regardless of style. Capture its findings.

### 3. Language quality passes (parallel across languages)
For each non-empty language bucket, invoke the matching optimizer skill on the **changed files only**, and tell it to restrict findings to changed lines:
- Scala bucket → `scala-code-optimizer`
- Python bucket → `python-code-optimizer`
- JS/TS bucket → `js-code-optimizer`

These are suggestion-only and never edit. They give the idioms / anti-pattern / FP / design-pattern / anti-slop findings. If the buckets are independent and large, dispatch them as parallel subagents (one per language) so they run concurrently.

### 4. Over-engineering pass (always)
Run `/ponytail-review` on the diff. Catches the AI-slop failure mode the optimizers miss: code that is idiomatic but shouldn't exist — speculative abstraction, a factory for one product, config for a constant, reinvented stdlib.

### 4b. UI/UX pass (only if the diff touches frontend)
If the changed files include `.jsx`/`.tsx`/`.vue`/`.svelte`/`.css`/`.scss` or React/UI components, run `impeccable` (and `frontend-design` for visual direction) on the changed UI to catch UX/UI anomalies the code lenses don't: broken visual hierarchy, accessibility gaps (labels, contrast, focus, keyboard), responsive breakage, inconsistent spacing/tokens, missing empty/error/loading states, confusing UX copy, layout/alignment drift. Skip entirely for backend-only diffs — say so under `## Skipped`.

### 4c. Test-quality pass (when the diff touches test files OR adds non-trivial logic)
Run the `test-quality-audit` skill on the diff's test files, cross-referenced against the changed source. This catches false confidence the "tests present" convention check can't: mock-echo tests (stub in, same value asserted out), hardcoded oracles pasted from the implementation's own output, tautologies, verify-only-with-any(), can-never-fail asserts, and new source branches with no behaviour test. **A new test that is TAUTOLOGY-class does NOT satisfy the tests-present gate** — treat the logic as untested (FIX-FIRST). Include each flagged test's falsifying-mutation ("flip X at src:line — this test stays green") in the finding.

### 4d. Principles sweep (always)
Sweep the diff once against `references/principles.md` (SOLID, DRY, KISS/YAGNI, layering, cohesion/coupling, fail-fast, encapsulation, least-astonishment, idempotency — each mapped to detectable smells). Rank per that file's rules: blocker only with a correctness consequence; should-fix for structural violations on new code; pre-existing violations → one line under Skipped.

### 4e. Repo baseline standards (when the repo carries them)
If the repo has `.sneakoscope/code/ENGINEERING.md` (legacy `.impeccable/code/`) (quality strategy), `.sneakoscope/code/ARCHITECTURE.md` (system capture + invariants), or `.sneakoscope/proposals/*-conventions.md`, read them and review the diff against THOSE as the codified team bar — e.g. flag a diff that violates a listed architecture invariant, an adopted convention, or re-introduces a pattern an audit called out. Repo baseline docs outrank this skill's generic defaults wherever they conflict.

### 4f. Data & ops artifact passes (when the diff touches them)
Route by artifact type, using each skill's doctrine if installed (skip with a note under Skipped if not):
- `.sql` files or embedded SQL strings -> `sneakoscope-stores` (queries doctrine + the engine chapter matching the actual store).
- Spark/Flink job code or pipeline DAG configs (JSON/YAML) -> `sneakoscope-pipelines` (incl. the creds-in-config and silent-empty-DataFrame connector-flag checks).
- `.sh`/CI steps/kubectl/helm sequences -> `sneakoscope-scripts` (strict mode, idempotency, dry-run, secrets, context/namespace hygiene).

### 5. Convention / gate checks (this skill does these directly)
Read the repo's `CLAUDE.md` / `CONTRIBUTING.md` if present and apply ITS conventions (title format, commit format, branch policy). Generic defaults when the repo defines none:
- **PR title** matches the repo's convention (e.g. conventional-commit prefix + ticket key). If the repo CI enforces a title format, a mismatch is BLOCKED.
- **Commit subjects** follow the repo's documented format (check recent `git log` for the established pattern).
- **Target branch** is a real release/`develop` branch, not another feature branch (unless the user said it's a stacked PR).
- **Tests present** — any non-trivial logic change (branch, loop, parser, money/security path) has a matching test in the diff. Missing test on new logic = FIX-FIRST.
- **No secrets / debug** — scan the diff for tokens, credentials, `console.log`/`println`/`print(` debug, hardcoded hosts, committed `.env`.
- **No commented-out code / TODO dumps** left behind.
- **Ticket key hygiene** — commit/PR references ONLY the target ticket; issue keys in messages auto-link and notify, so stray keys spam unrelated tickets.

### 6. Synthesize — ONE verdict
De-duplicate findings that multiple lenses reported (same file:line). Rank by severity. Then emit the report below.

### 7. Write the gate breadcrumb (self / branch mode only)
After the report, in self-review or branch mode, record that the gate ran so the `gh pr create` hook (`pr-gate-guard.sh`) lets the PR through:
```
touch "/tmp/claude-pr-gate-$(git rev-parse --abbrev-ref HEAD | tr '/ ' '__').done"
```
Do this even on a `FIX-FIRST`/`BLOCKED` verdict — the gate *ran*; the hook only checks that a human saw the review, and any new commit (e.g. your fixes) auto-invalidates the breadcrumb so you re-gate. In PR-review mode (reviewing a teammate's PR by number) do **not** write a breadcrumb.

## Output format

```
# PR Gate: <branch or PR#> → <base>
Verdict: SHIP | FIX-FIRST | BLOCKED
Scope: N files (Scala x / Python y / JS-TS z / other w), ±L lines

## Blockers        (must fix before merge — bugs, security, failing conventions)
- [correctness] file.scala:42 — <one line>. Fix: <one line>
## Should-fix      (idiom/anti-pattern/over-engineering worth fixing now)
- [scala] ...  [ponytail] ...  [python] ...
## Nits            (optional, low)
- ...
## Conventions
- PR title: PASS/FAIL — ...
- Tests: PASS/FAIL — ...
- Secrets/debug: PASS/FAIL — ...
- Target branch: PASS/FAIL — ...
## Skipped
- <what wasn't reviewed and why — e.g. "generated file X", "diff too large, gated to module Y">
```

**Verdict rules:**
- **BLOCKED** — any correctness bug, security issue, secret, or CI-fail convention (bad PR title).
- **FIX-FIRST** — no blockers, but new logic lacks tests, or there are should-fix anti-patterns/over-engineering.
- **SHIP** — only nits or clean.

## Rules

- **Review the diff, not the repo.** Findings must anchor to changed lines. A pre-existing issue in untouched code is out of scope (mention once under Skipped if glaring, don't gate on it).
- **Do not edit or fix anything.** This is a gate — it reports. If the user then says "fix these", apply the blockers first.
- **Do not fabricate.** If a lens couldn't run (no `gh` auth, detached diff, tool missing), say so under Skipped — don't pretend it passed.
- **Self-review vs teammate PR is the same pass** — the only difference is where the diff comes from. Don't go softer on your own diff.
- **Evidence before verdict** — SHIP only after every lens actually ran. Missing lens → say what's unverified, don't claim clean.
- **Big diff**: gate by module and say which modules you covered; never silently truncate.

## Self-check
Trivial run: `/pr-gate` on a branch with one `.py` file changed should → establish base, run /code-review + python-code-optimizer + /ponytail-review + conventions, emit one verdict. If any lens is skipped it must appear under `## Skipped`.
