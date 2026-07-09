# Init Flow

The setup command for a backend repo. One codebase crawl feeds two artifacts:

- **ARCHITECTURE.md** (capture): what the system IS — modules, layering, effect model, datastores, auth, version pins, invariants. Answers "what exists and what must not silently change".
- **ENGINEERING.md** (quality bar): how work on this system is judged — precedence-ordered principles, per-language bars, scoring scheme. Answers "what good looks like here".

Both live at the repo root or under `.impeccable/code/`. Every other impeccable-backend command reads them before doing any work. Init is a **capture**, not a critique: you record the system as-is, in neutral language, with citations. Improvement proposals belong to `critique`; migration plans to `evolve`.

## Step 1: Load current state

Check whether ARCHITECTURE.md / ENGINEERING.md already exist (root, `.impeccable/code/`, `docs/`). Also note an ADR directory (`docs/adr/`, `adr/`, `docs/decisions/`).

- **Neither exists**: full flow, Steps 2–7.
- **ARCHITECTURE.md exists, ENGINEERING.md missing**: refresh nothing silently — do Steps 6–7 only, cross-checking the existing capture's claims against the current code (stale `file:line` cites get re-verified or marked "not verified").
- **Both exist**: STOP and ask which to refresh. Never silently overwrite.

If init was invoked as a blocker by `design`/`shape`, complete init, then resume the original command — your own writes are the freshest source; no reload needed.

## Step 2: Inventory the build

Enumerate what the repo actually builds before reading any source:

- **Build files**: `build.sbt`, `pom.xml`, `build.gradle*`, `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, workspace/multi-module descriptors. Each build unit = one row in the module map.
- **Deploy units**: Dockerfiles, Helm charts, CI workflows (`.github/workflows/*`) — which modules ship, which are dev-only. A module the release workflow never packages is "not shipped"; say so.
- **Entry points**: main classes, WAR/servlet descriptors, serverless handlers, cron/scheduler registrations, queue consumers. Every entry point is a boundary the capture must account for.

Record per module: name, path, language, what it produces (service, library, job, pipeline config), and whether it ships.

## Step 3: Map layering and effect model

Open 2–3 representative request paths end-to-end (controller → service → DAO, or handler → use-case → repository — whatever the repo actually does):

- **Layering**: name the layers as the code names them (packages/directories), not as textbook ideals. Note the dependency direction and cite one import that proves it (`file:line`).
- **Effect/concurrency model**: `IO`/`ZIO`/`Future`, async/await, thread pools, blocking JDBC on request threads, actor systems. If two models coexist (e.g. `IO` in module A, `Future` in module B), record BOTH — that split is observed state, not a defect to fix here.
- **Error channel**: exceptions vs `Either`/`Try` vs error codes; where errors become HTTP responses; whether one error envelope exists.
- **Transactions**: where the transaction boundary sits (annotation, session block, manual commit) and which layer owns it.

## Step 4: Map datastores and ownership

For each datastore (RDBMS, KV, search, queue, cache, object store):

- Which module **writes** it, which modules **read** it — cite the DAO/client class per path. Multi-writer tables are recorded neutrally: "written by X (`file:line`) and Y (`file:line`)".
- Where the **schema** lives (DDL dirs, migration tool, ORM-generated) and how it evolves (versioned migrations vs manual).
- Read path ≠ write path divergences (e.g. writes via ORM, reads via raw SQL; sync API reads table A while batch reads its Hive twin) — capture them; they are the landmines every future design must respect.

## Step 4b: Cross-cutting conventions

Sweep for the conventions every future design must either follow or deliberately break. For each, record the dominant pattern, the minority pattern if one exists, and one cite per pattern:

- **Error shape**: the envelope(s) actually returned — grep the exception-to-response mappers, not the docs.
- **Pagination**: offset vs cursor, default and max page sizes, and whether the cap is enforced server-side.
- **Serialization**: the one (or several) JSON libs in play; strict vs tolerant deserialization defaults.
- **Logging & correlation**: is there a correlation/request id? Does it survive async hops? Cite the filter/middleware that sets it and one queue consumer that does or doesn't propagate it.
- **Idempotency & retries**: any existing retry wrapper, idempotency-key convention, or dedup table — future designs should reuse, not reinvent.
- **Testing conventions**: test frameworks per module, whether integration tests exist and what they spin up (Testcontainers, in-memory, shared env). A design's test strategy must be writable in the repo's own vocabulary.

## Step 5: Auth model and version pins

- **Auth**: authN mechanism (JWT/session/mTLS), where it's enforced (filter, middleware, per-controller), authZ model (roles, scopes, resource-based), and the one place the permission strings are defined. Cite each.
- **Version pins**: for each load-bearing dependency (framework, runtime, DB driver, serialization lib), record the version AND the file that pins it (`build.sbt:41`, `Dockerfile:1 FROM ...`). If the same dependency is pinned in two places (pom BOM + Dockerfile FROM), record both pins — dual-pin drift is a classic silent failure.
- **Config**: where runtime config comes from (env vars, config files, dyn-properties table) and which module reads which.

## Step 6: Interview — only what code can't reveal

STOP and ask the user. 5–7 questions max, 2–3 per round. For each, infer from code FIRST and lead with your hypothesis; ask only to confirm or fill the gap.

1. **SLAs / latency budgets**: "I see no timeout config beyond X (`file:line`) — what latency/availability targets does this system actually have?" (Infer first: gateway timeouts, healthcheck settings.)
2. **Scale numbers**: rows in the biggest tables, requests/sec, batch sizes. (Infer first: partition schemes, index count, pool sizes hint at scale — state your read.)
3. **Team topology**: who owns which module; is the module split an org split? (Infer first: CODEOWNERS, commit authorship clusters.)
4. **Known pain**: "Which part of this system pages you / do you dread touching?" — code can't reveal operational scar tissue.
5. **In-flight migrations**: anything half-moved (Scylla→MariaDB, v1→v2 API) that the code shows both sides of? (Infer first: dual read paths, `*_v2` names, deprecated annotations.)
6. **Compliance/tenancy constraints**: data residency, audit retention, multi-tenant isolation guarantees. (Infer first: tenant columns, audit tables.)
7. **What must not change**: any behaviour clients depend on that looks accidental in code (field ordering, error strings, undocumented endpoints).

Do NOT ask about anything Steps 2–5 already answered. Do not ask style/preference questions — ENGINEERING.md's bar is derived from the doctrine plus observed conventions, not taste polling.

## Step 7: Write ARCHITECTURE.md

Every claim cites `file:line` or a command output. Anything you could not verify is written as **"not verified"** — never guessed. Observed inconsistencies are recorded as **observed state** ("module A uses IO, module B uses Future"), never as fixes ("module B should migrate to IO" — that sentence belongs in a critique, not here).

```markdown
# Architecture

> Captured <date> from <commit sha>. Claims cite file:line; unverified claims say so.

## Module map
| Module | Path | Language | Produces | Ships? | Layering |
|---|---|---|---|---|---|

## Effect & concurrency model
[Per module: effect type, thread/pool model, error channel, transaction boundary. Cited.]

## Datastores & ownership
| Store | Written by | Read by | Schema lives at | Evolution mechanism |
|---|---|---|---|---|
[Below the table: read/write path divergences, multi-writer notes — neutral, cited.]

## Auth model
[AuthN mechanism + enforcement point; authZ model + permission-string source. Cited.]

## Cross-cutting conventions
[Error envelope, logging/correlation-id, pagination style, config source, serialization. Cited. Coexisting conventions listed side by side as observed state.]

## Version pins
| Dependency | Version | Pinned by |
|---|---|---|
[Dual pins get two rows and a note.]

## Observed state (inconsistencies, neutral)
[Splits, divergences, half-migrations — facts only, no recommendations.]

## Invariants worth preserving
[5–10 bullets. Each: the invariant + why breaking it hurts + the cite that establishes it.
e.g. "Every unified_alerts write goes through AlertDao (dao/AlertDao.scala:88) — the CM copy table is synced from it, direct writes desync CM."]
```

## Step 8: Write ENGINEERING.md

```markdown
# Engineering Bar

## Precedence (when principles conflict, higher wins)
1. Correctness — data integrity, no silent failure, security.
2. Behaviour preservation — existing observable behaviour changes only deliberately, with a test proving old and new.
3. FP / effect discipline — [the repo's own model from ARCHITECTURE.md]: effects in the effect type, no bare side effects in domain logic, errors on the typed channel.
4. Consistency — new code matches the module it lives in, even where modules differ from each other.
5. Anti-slop — no speculative abstraction, no new moving part without a stated need, no dead flexibility.

## Per-language bar
[One pointer block per language in the module map: idiom source (e.g. scalafmt config at path, ruff/eslint config at path), the 3–5 repo-specific rules that CI does NOT enforce but reviewers do. Cited to real examples in the repo.]

## Scoring scheme
- Findings: P0 (correctness/data-loss/security — blocks) · P1 (will bite in production) · P2 (debt, fix when touching) · P3 (style).
- Health score: /20 per critique dimension (see reference/critique.md), reported with evidence.

## Guardrails proposed, not wired
[Checks worth automating (arch-unit tests, migration linters, timeout-config assertions) listed as PROPOSALS with what they'd catch. Init never adds CI steps, hooks, or failing tests — wiring is a user decision taken later.]
```

## Capture discipline — the rules that make the artifacts trustworthy

- **Verify before you write.** A claim goes in only after you've read the cited line in this session. `git grep` hits are leads, not evidence — open the file. For claims about a release branch, `git show origin/<branch>:<path>`; the working tree may be a different branch.
- **"Not verified" beats plausible.** A capture with 30 verified claims and 5 honest "not verified" markers is worth more than 35 confident guesses — the next design will be built on it.
- **Neutral voice on inconsistencies.** The observed-state section uses "X does A (`cite`); Y does B (`cite`)". Banned words in ARCHITECTURE.md: "should", "unfortunately", "legacy" (unless the code says it), "needs to be". If you feel a fix coming on, note it privately for the wrap-up recommendation to run `critique` — do not smuggle it into the capture.
- **Invariants are the payload.** The 5–10 invariants are what future designs check themselves against; each must be falsifiable ("every write to T goes through D") and consequence-stated ("direct writes desync the CM copy"). "Code should be clean" is not an invariant.
- **Cap the size.** ARCHITECTURE.md ≤ ~300 lines, ENGINEERING.md ≤ ~120. Past that, agents skim and the artifact stops doing its job. Depth goes into linked per-module notes only if the user asks.
- **Stamp the capture.** Header carries date + commit sha. Any command loading a capture older than ~50 commits on the mainline should re-verify the cites it relies on before trusting them.

## Step 9: Wrap up — menu

Summarize in ≤6 lines: modules captured, invariants count, notable observed state, what was written. Then recommend the 2–4 best next commands, drawn from what the crawl surfaced — not a menu dump:

- **Design a feature**: `/impeccable-backend design <feature>` — now grounded in the capture.
- **Get the system scored**: `/impeccable-backend critique` — turns the neutral "observed state" section into ranked, scored findings. Lead with this if Step 7 recorded ≥3 inconsistencies.
- **Record the first decision**: `/impeccable-backend adr <decision>` — start the ADR log with the biggest undocumented decision the capture exposed (e.g. the dual effect model, the copy-table sync).
- **Plan a half-done migration**: `/impeccable-backend evolve <migration>` — if Step 6 Q5 surfaced one.

One line each on why, with the exact command to type.
