# Harden Flow

A failure-mode design pass over a design doc or an existing flow. Input: a request path or data
path (from a design doc, or traced from code with `file:line` cites). Output: a harden report —
per-hop answers to five questions, gaps ranked P0–P3. Harden never edits code; it produces the
report and, where the input was a design doc, proposes amendments to it.

The premise: **failure is the normal case at scale.** Every dependency you call will time out,
return garbage, and go down for five minutes — this quarter. A flow is hardened when the design
answers what happens then, before it does.

## The workflow

1. **Draw the path.** List every hop the request/message crosses: client → edge → service →
   service → datastore/queue/third-party. Include the async legs (outbox, consumers, schedulers).
   A hop you didn't list is a hop you didn't harden. Cite `file:line` for each hop when working
   from code; from a design doc, cite the section.
2. **At each hop, answer the five questions.** No hop is done until all five have concrete
   answers (a number, a mechanism, a metric name) — "handled by the framework" is not an answer
   until you've named the framework default and checked it.
3. **Rank the gaps.** Every unanswered or wrong answer becomes a finding: P0 (data loss /
   corruption / unbounded resource), P1 (outage amplifier: retry storms, pool starvation,
   unbounded queue), P2 (degraded UX / silent failure), P3 (observability gap only).
4. **Write the report** (template at the bottom). Findings cite the hop and the question.

## The five questions — asked at every hop

1. **What's the timeout?** Connect and read, separately. Is the caller's total budget smaller
   than the callee's? If not, callers give up while callees keep working — the work is wasted
   twice and the callee's capacity is burned for nobody.
2. **What happens on retry?** Is the operation idempotent? Keyed how? If it's a write and the
   answer is "it isn't", the retry policy at this hop is: none.
3. **What happens when the downstream is down for 5 minutes?** Queue it (bounded, where)? Shed
   it (which requests)? Error to the user (what message, what status)? "It'll error" is a P2
   until the error shape and user experience are stated.
4. **What happens at 10× load?** Where is the bound (pool size, queue depth, rate limit)? What
   backs up behind it? What degrades first, and did a human choose that ordering?
5. **How do we KNOW it's failing?** Name the metric, the alert threshold, and the log line with
   correlation id. A failure mode without a detection mechanism is discovered by customers.

## Timeout budget doctrine

Budgets are allocated **top-down** from the user-facing surface, never accreted bottom-up from
library defaults. Pick the end-to-end budget first, then divide it across hops with headroom.

| Surface | End-to-end budget |
|---|---|
| Interactive read (page/API GET) | 3s |
| Interactive write (form submit, action) | 10s |
| Long-tail sync (report gen, search, export kick-off) | 30s — above this, go async + poll |
| Service-to-service internal hop | ≤ half of what remains of the caller's budget |
| Batch/async consumer per message | explicit, own budget; never "whatever the client lib does" |

Rules:

- **Connect timeout ≠ read timeout.** Connect: short (250ms–1s) — a host that won't accept a
  socket won't answer either. Read: sized to the callee's p99, not its average. One combined
  "timeout" setting is a smell; find both knobs.
- **caller_timeout < callee_timeout is the doctrine written per hop.** At every edge, the
  upstream deadline must expire *after* the downstream's own budget, so the downstream either
  answers or fails cleanly inside the window. Propagate the remaining budget (deadline header)
  where the stack supports it.
- **The double-timeout trap.** Worst case per hop = attempts × (connect + read) + backoff sum.
  3 attempts × 2s read + backoff ≈ 7–10s — inside a 3s caller budget, attempts 2 and 3 are
  talking to a caller that already hung up. Compute the worst case for every hop with retries;
  if it exceeds the caller's remaining budget, cut attempts or cut the per-try timeout.
- Every timeout is config, not code, and every value in the report has a source
  (`file:line` or config key) — "the default" means you don't know it.

## Retry doctrine

Retries convert transient blips into successes — and sustained failures into self-inflicted
DDoS. **Retries amplify load exactly when the system can least afford it**: the downstream is
slow because it's saturated, and every retry adds to the saturation. Design for that.

- ≤3 attempts total (initial + 2 retries). Exponential backoff with **full jitter**
  (`sleep = rand(0, base × 2^attempt)`); deterministic backoff synchronizes clients into waves.
- **Retry budget**: cap retries as a fraction of traffic (≤10%). When the budget is spent, fail
  fast — a downstream returning 100% errors must not receive 3× traffic.
- Never retry: non-idempotent writes without an idempotency key; 4xx-class errors (the request
  is wrong, repeating it is spam); calls through an open circuit breaker; anything after the
  caller's deadline has passed.
- Retry at **one layer only**. Client-retries × mesh-retries × app-retries = 27 attempts from
  3×3×3. Pick the layer that has the idempotency key; disable the rest.

## Idempotency mechanics

- **Explicit key**, not vibes: client-generated (UUID minted on the form/first attempt) or
  derived (hash of business identity — `order_id + event_type`). The key travels in a header or
  message attribute and is part of the API contract.
- **Dedupe store**: keyed table or cache recording key → result. TTL ≥ the maximum redelivery
  window of every queue/retry path that can carry the key (queue retention + DLQ redrive window
  + client retry horizon). A 1h TTL under a 4-day redelivery window is a duplicate generator
  with a delay.
- On duplicate: return the **stored result** of the first execution, same status code — not an
  error, not a re-execution.
- **At-least-once is the only real queue semantics.** "Exactly-once" is a marketing term for
  at-least-once plus dedupe you still have to build. Design every consumer to tolerate
  duplicates: idempotent handler, or dedupe-on-key at the top of the handler. A consumer that
  double-charges on redelivery is broken today, not "under rare conditions".

## Backpressure & load shedding

- **Every queue and buffer is bounded, with a stated behavior at the bound**: block (propagate
  backpressure upstream), shed (reject with a retryable status), or spill (to disk/overflow
  topic). Unbounded in-memory buffering is an OOM with a delay. Which of the three is a
  *product* decision — name it in the report, don't let the library default decide.
- **Shed lowest-value work first, and name it.** "Under pressure we drop X before Y" —
  prefetch before user actions, analytics before writes, anonymous before authenticated. If
  nobody has named the shedding order, the order is "random", which means "whatever was
  unlucky".
- **Circuit breakers** on every external dependency: open on error-rate/latency threshold,
  half-open probes, and a **fallback defined per call site** — cached value, default, degraded
  response, or explicit error. Per the doctrine, the fallback is *louder* than the primary:
  metric + log line every time it engages. A silent fallback is silent corruption's front door.
- **Bulkheads**: one pool (threads/connections) per dependency. The shared-pool starvation
  tell: one slow downstream, and suddenly *unrelated* endpoints time out — because every worker
  in the shared pool is parked waiting on the slow one. If the design has one global pool in
  front of N dependencies, that's a P1 regardless of current load.
- Admission control at the edge beats graceful collapse in the middle: rate-limit / concurrency-
  limit at ingress with a 429 + Retry-After, so the interior never sees the load it can't take.

## Partial failure semantics (fan-out)

- Every fan-out (N downstream calls, batch of N items) declares its semantics **explicitly**:
  **all-or-nothing** (any failure fails the whole; requires rollback/compensation) or
  **best-effort + report** (response enumerates per-item success/failure; caller decides).
  "It throws on the first failure" is an accident, not semantics.
- **Multi-step writes get a compensation path (saga outline)**: for each step, the compensating
  action; the order of compensation (reverse); what happens when a *compensation* fails (park +
  alert — a human finishes it, the system must say so). No compensation path = the design is
  claiming a distributed transaction it doesn't have.
- **Poison messages**: a message that fails N (bounded!) deliveries goes to a DLQ. Alert on DLQ
  depth > 0 and on oldest-message age. Never infinite redelivery — one poison message pinning a
  partition is a full-consumer outage wearing a single message's face. DLQ items have an owner
  and a replay procedure, or the DLQ is just a slower /dev/null.

## Observability floor

The minimum below which a flow is not shippable:

- **Correlation id at every boundary**: generated at ingress, propagated through every hop
  including queues (message attribute) and scheduled jobs, present in every log line. One id
  reconstructs the whole request.
- **RED metrics per endpoint**: Rate, Errors, Duration (p50/p95/p99 — averages hide the pain).
  Per endpoint, not per service; one hot endpoint drowning in a service-level average is
  invisible.
- **Staleness lag for every async consumer**: consumer lag (messages or time-behind-head) as a
  first-class metric with an alert. Throughput without lag tells you it's busy, not that it's
  keeping up.
- **"How do we know it's stuck?" answered for every queue**: lag metric + DLQ depth +
  oldest-message age, each with an alert threshold. A queue with no staleness alarm fails
  silently for days — the first detector is a customer asking where their data went.
- Circuit-breaker state transitions and fallback engagements are metrics + logs, per the
  louder-than-primary rule.

## Output: the harden report

```markdown
# Harden Report — <flow name> (<date>)

Path: client → api-gw → order-svc → payment-svc → payments-db / outbox → billing-consumer

## Per-hop answers

| # | Hop | Timeout (conn/read; caller<callee?) | Retry (idempotent? key?) | Down 5 min | 10× load | How we know |
|---|---|---|---|---|---|---|
| 1 | api-gw → order-svc | 250ms/2.5s; ✓ (gw 3s) | 2×, jitter; GET only | 503 + Retry-After | 429 at 500 rps ingress limit | RED per route, corr-id |
| 2 | order-svc → payment-svc | 250ms/1s; ✓ | NONE — non-idempotent write, key TODO | breaker open → fail order, user msg X | bulkhead pool=20, shed anon first | breaker-state metric, p99 alert |
| 3 | outbox → billing-consumer | n/a (async, per-msg budget 30s) | redelivery ≤5 → DLQ | backlog bounded 100k, alert at 10k | lag alert 5 min | consumer-lag + DLQ depth + oldest-age |

Each cell cites its source (file:line / config key / design §) or is marked ASSUMED.

## Gaps

| Rank | Hop | Question | Gap | Fix |
|---|---|---|---|---|
| P0 | 2 | retry | payment call retried by mesh with no idempotency key → double charge | key = order_id; dedupe table TTL 7d; disable mesh retry |
| P1 | 1–2 | timeout | 3 attempts × 2.5s > gw 3s budget (double-timeout trap) | attempts→2, read→1s |
| P2 | 2 | down-5-min | breaker fallback silent (returns cached quote, no log) | metric + WARN log on every fallback |
| P3 | 3 | know | no oldest-message-age alert on billing DLQ | add alert, owner = billing team |

## Assumptions
- payment-svc p99 assumed 400ms — no measurement cited; verify before sizing read timeout.
```

A flow passes harden when every hop has all five answers cited, and no P0/P1 remains open.
