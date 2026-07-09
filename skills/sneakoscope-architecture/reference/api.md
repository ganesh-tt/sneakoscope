# API — design a contract

The `api` flow. A contract is a promise to people you can't see and can't redeploy. Design it
as one — the code is an implementation detail of the contract, never the reverse.

## Contract-first workflow

Run in order; each step is written into the contract doc before any handler exists.

### 1. Name the consumers first

- List every consumer by name: which teams, which apps, which jobs. "The frontend" is not a
  consumer list — the mobile app on a 6-week release train and the web app that deploys daily
  have different compatibility needs, and the strictest consumer sets the rules.
- For each consumer: can it be redeployed on your schedule? Anything you can't redeploy
  (mobile, partner integrations, on-prem installs) makes every breaking change a multi-quarter
  project. Know this before drawing the resource model.

### 2. Resource and verb model

- Nouns are resources, HTTP verbs are the operations. `POST /orders/{id}/cancel` is acceptable
  for a genuine domain action; `POST /doThing?mode=cancel` is not (see god-endpoints below).
- One resource shape per resource. If two endpoints return "an order" with different fields,
  either they're different resources (name them differently) or one is a documented partial
  representation (`?fields=` / summary vs detail) — not an accident of two serializers.
- URL depth ≤2 nesting levels (`/customers/{id}/orders` fine; deeper means the sub-resource
  wanted its own top-level collection).

### 3. Error envelope BEFORE the happy path

Design the error shape first — it's the part every consumer must handle and the part teams
improvise last. One machine-readable envelope across the entire surface:

```json
{
  "error": {
    "code": "ORDER_ALREADY_CANCELLED",     // stable, machine-matched, SCREAMING_SNAKE
    "message": "Order ord_123 was cancelled on 2026-07-01.",  // human, not for parsing
    "correlationId": "req_8f3a...",         // matches server logs end-to-end
    "fieldErrors": [                        // optional, validation failures only
      { "field": "amount", "code": "MUST_BE_POSITIVE", "message": "..." }
    ]
  }
}
```

- Consumers branch on `code`, never on `message` text. The message is free to change; the code
  is contract.
- Every documented endpoint lists its possible error codes. An undocumented error code is a
  breaking change deployed by surprise.

### 4. Compatibility rules written INTO the contract

- **Tolerant reader is a contract clause, not a hope**: "consumers MUST ignore unknown fields
  in responses and unknown enum values where marked extensible." Write it in the doc; a
  consumer that hard-fails on a new optional field has violated the contract, and that's only
  enforceable if the sentence exists.
- The contract doc (OpenAPI/proto/schema) is versioned in the repo and reviewed like code —
  because it is the code that other teams compile against.

## Compatibility classification

Classify every proposed change before implementing it. When in doubt, it's breaking.

| Class | Rule | Examples |
|---|---|---|
| **Additive — free** | Ship anytime, no version bump | New optional request field with a default; new response field; new endpoint; new error code on a new endpoint |
| **Deprecating — sunset date + measurement** | Old form keeps working; announce sunset; measure usage to zero before removal | Replacing a field with a better-named twin (ship both, deprecate old); tightening a doc'd-but-unenforced limit |
| **Breaking — new version** | Never in place | Making an optional field required; removing/renaming/retyping a field; changing an error code; narrowing accepted input; changing default sort order |

Traps that look additive but aren't:

- **Widening an enum is breaking for validating consumers.** Adding `PARTIALLY_SHIPPED` to a
  status enum breaks every client that exhaustively matches or validates the old set. Either
  the contract marked the enum extensible from day one (tolerant reader clause + a documented
  fallback value), or the widening is a versioned change.
- Adding a field the consumer's deserializer treats as strict (some generated clients default
  to fail-on-unknown) — this is why the tolerant-reader clause is written down and verified.
- Changing serialization config (see anti-patterns) — omitting nulls where they were present,
  renaming via a global naming strategy — is a breaking change wearing a refactor's clothes.

## Pagination doctrine

- **Cursor pagination for anything unbounded or user-scrollable.** The cursor is opaque
  (encoded keyset position, signed if it must not be forged) — never an exposed offset, never
  a raw last-ID the client is invited to construct. Offset pagination past ~10k rows is a
  database incident scheduled for later (per SKILL.md).
- **Offset is acceptable only for small bounded admin lists** — a few hundred rows, internal
  users, no growth path. Say so in the contract; "temporarily offset" lists grow.
- **Every list endpoint declares default AND max page size** (e.g. default 25, max 100). An
  unbounded `?limit=` is a memory exhaustion endpoint with a friendly name.
- Response carries a has-more signal: `nextCursor` present/absent, or an explicit `hasMore`.
  Total counts are a separate, optional, cacheable concern — a `COUNT(*)` on every page of a
  hot table is self-harm; omit or estimate unless the product needs it.
- Stable sort key under the cursor (unique tiebreaker column) — a cursor over a non-unique
  sort skips or duplicates rows at page boundaries.

## Mutation semantics

- **Every mutation is idempotent** — naturally (PUT full-replace, DELETE) or via an
  `Idempotency-Key` header the server persists with the first response. Retry with the same
  key returns the stored result; same key with a *different* body is `409` (see errors). This
  is not optional: every client that retries on timeout (all of them) double-submits without it.
- **Return the resulting resource state** in the mutation response. Forcing an immediate
  GET-after-POST doubles the traffic and opens a read-replica staleness window where the
  client can't see what it just created.
- **PATCH merge semantics are documented, explicitly**: JSON Merge Patch (RFC 7386) or JSON
  Patch (RFC 6902) — named in the contract. The **null-vs-absent distinction is stated**:
  absent field = leave unchanged, `null` = clear the value. A PATCH endpoint that treats them
  identically silently erases data on the first client that omits fields it didn't load.
- Server-generated fields (id, timestamps, computed values) in a write body are ignored or
  rejected — pick one, document it; silently ignoring some and honoring others is how clients
  learn to fear you.

## Async APIs

- Any operation that can exceed ~5s (report generation, bulk import, downstream fan-out)
  returns **`202 Accepted` + a status resource**: `{ "operationId": ..., "status": "PENDING" }`
  with a `Location`/URL to poll. Status values have explicit terminal states
  (`SUCCEEDED` / `FAILED` with the standard error envelope embedded) — a status enum where the
  client can't tell "still working" from "wedged forever" is not a status.
- Poll endpoints are cheap, cacheable, and state the recommended interval. Completed operations
  link to the result resource; they don't inline a 50MB payload into the status body.
- **Webhooks/events carry, minimum**: event `id` (unique — this IS the idempotency key), `type`
  (versioned, e.g. `order.cancelled.v1`), `occurredAt` (UTC), and the affected resource id.
  Payload is either full state or a pointer the consumer fetches — pick per event type,
  document it.
- **Consumers dedupe by event id** — the contract says delivery is at-least-once and ordering
  is not guaranteed, so the consumer's handler is idempotent. Write both sentences into the
  webhook doc; every webhook consumer that skipped them has processed a payment twice.
- Webhook deliveries are signed (HMAC over body + timestamp) and retried with backoff to a
  bounded limit, with a dead-letter/redelivery story the consumer can trigger.

## Versioning

- **Path (`/v2/`) or header — one choice, repo-wide.** Path is visible and cache/proxy-friendly;
  header is purer REST. The wrong choice is having both.
- A version bump = a new contract doc, not a diff comment. `v2` gets its own spec file;
  consumers of v1 read a spec that never mutates under them.
- Version the surface, not each endpoint — per-endpoint version soup (`/v1/orders`,
  `/v3/customers`) makes every client hold a version matrix in its head.
- **Deprecation = `Sunset` header (RFC 8594) + `Deprecation` header on responses + measured
  removal.** Log per-consumer usage of the deprecated surface; removal happens when measured
  usage hits zero or the sunset date passes with named stragglers notified — never on a hope.

## Error doctrine

- **4xx = the caller can fix it; never retry-worthy.** A client retrying a `400` in a loop is
  a bug the contract invited by mis-coding a transient failure as 4xx. Get the class right.
- **5xx = the caller may retry** — bounded, jittered exponential backoff (per SKILL.md failure
  doctrine), and only against idempotent operations (which, per mutation semantics, is all of
  them).
- **`409 Conflict`** for: optimistic-lock version mismatch, unique-constraint collision, state
  conflicts ("order already shipped"), and **idempotency-key replay with a divergent body**.
- **Timeouts return `504`, not `500`.** A `504` tells the caller "retry may work"; a generic
  `500` tells them nothing. Similarly: `429` with `Retry-After` for rate limits, `503` for
  planned unavailability — the status code is the retry policy.
- **Never leak internals**: no stack traces, SQL fragments, ORM class names, hostnames,
  internal IPs, or dependency error text in any response body, any environment. The
  `correlationId` is the bridge to the internal detail — that's what it's for. Internal error
  text in a 500 body is both an information-disclosure finding and a consumer that will parse it.
- Validation failures report **all** field errors in one response (the `fieldErrors` array),
  not first-failure-only — one round trip per field is a hostile form.

## Anti-patterns — match and refuse

- **Boolean-blind endpoints**: `200 OK` with `{"success": false}`. Status codes are the
  contract's first field; monitoring, retries, caches, and every HTTP client on earth key off
  them. A 200-wrapped failure is invisible to all of it. Errors use error status codes and the
  envelope, period.
- **Chatty N+1 shape**: a list endpoint returning ids that force one GET per item. Provide
  batch fetch (`?ids=a,b,c`, bounded) or documented expansion (`?expand=customer`) — the
  contract shapes the client's loop; design it out.
- **God-endpoints with mode parameters**: `POST /process?action=create|update|cancel`, or one
  request body whose meaning flips on a `type` field. Each mode is a separate endpoint with
  its own contract, validation, and error set. Mode parameters make every change a breaking
  change to every mode's consumers simultaneously.
- **Breaking-change-by-serializer-config**: flipping a global naming strategy
  (camelCase→snake_case), toggling include-nulls, changing date format via a framework upgrade
  default. The wire format IS the contract — pin it with serialization tests that assert
  literal JSON, so a dependency bump can't renegotiate the contract silently.
- **Leaking storage through the contract**: response fields mirroring column names, enums
  mirroring DB ints, ids exposing auto-increment sequence (business intelligence for free to
  anyone who can count). The contract is a projection of the domain, not a `SELECT *`.
- **Anonymous ad-hoc responses**: endpoints returning shapes that exist nowhere in the
  contract doc. If it's not in the spec, consumers are coupling to an accident.
