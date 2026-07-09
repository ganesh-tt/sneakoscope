# Software-Engineering Principles Checklist — applied by /pr-gate across FE/BE/DE

One shared set of principles, each with the concrete, *detectable* smell to look for in a diff.
Cite `file:line`; a principle finding without a concrete manifestation is not a finding.
Precedence on conflict: Correctness > behaviour preservation > KISS/YAGNI > the rest.
(Language-specific idiom/FP rules live in the optimizer skills; this file is the cross-language layer.)

| Principle | Flag when the diff shows | Don't flag when |
|---|---|---|
| **SRP** (single responsibility) | a class/module gaining a 2nd unrelated reason to change — service doing parsing + persistence + notification; "god" function absorbing another concern | cohesive helpers on one concern grow together |
| **OCP** (open/closed) | modifying a stable dispatch site *again* to add a variant — a match/switch/if-ladder on a type tag that every new feature edits | first or second variant — the ladder is fine until it recurs (rule-of-three) |
| **LSP** (substitutability) | a subtype/`extends`/impl that throws `UnsupportedOperation`, no-ops, or narrows accepted inputs vs its base contract | intentional partial impls documented at the trait/interface |
| **ISP** (interface segregation) | implementors forced to stub methods they never use; wide trait/interface where callers use ≤2 members | small cohesive interfaces even if one impl exists (but see YAGNI) |
| **DIP** (dependency direction) | domain/service layer importing concrete infra (DAO impl, HTTP client, DB driver) instead of receiving it; DE: business rules reading connection config directly | thin modules where indirection would be ceremony (call it out as accepted, not missed) |
| **DRY** | ≥3 copies of logic that must change together (copy-pasted service methods, duplicated SQL fragments, re-implemented validation); single-source-of-truth violations (same constant/threshold defined twice) | 2 accidental look-alikes; different owners/change-cadence — duplication is cheaper than the wrong abstraction |
| **KISS** | nested ternaries, clever one-liners needing a comment to decode, reflection/metaprogramming where a function works, flag-parameter forks | domain complexity that is irreducible |
| **YAGNI** | abstraction with one implementation, config for a constant, speculative generic params, "manager/factory/provider" for nothing (ponytail lens owns this) | extension points a *committed* next ticket needs |
| **Separation of concerns / layering** | controller doing business logic or SQL; DAO doing HTTP; UI component fetching + transforming + rendering + persisting; pipeline UDF writing to stores it shouldn't own | thin pass-through layers being skipped deliberately (document it) |
| **High cohesion / low coupling** | module A reaching into B's internals (`b.dao.conn`), train-wreck chains `a.b().c().d()` (Law of Demeter), cyclic imports/barrel cycles | fluent builders/DSLs (chains on self are fine) |
| **Composition over inheritance** | deep inheritance for code reuse, base-class grab-bags (`BaseService` with unrelated helpers), overriding to *remove* behaviour | genuine is-a hierarchies (sealed ADTs) |
| **Fail fast / no silent fallback** | swallowed exceptions, defaulting on parse failure without logging, `getOrElse(empty)` masking a missing-data bug, DE: empty-DataFrame fallback hiding a dead connector flag | explicit, logged, documented degradation |
| **Encapsulation** | public mutable state, exposing internal collections by reference, setters that break invariants; FE: components mutating props/store directly | DTOs/case classes that are plain data by design |
| **Least astonishment** | a function whose name lies (get* that mutates, find* that creates), side effects in getters/`.map`, overloaded meaning of a boolean param | conventional idioms of the codebase |
| **Idempotency (DE/infra)** | migrations/pipeline jobs/consumers that break on re-run (double-insert, no upsert/guard), non-idempotent retries on at-least-once delivery | genuinely once-only ops with an explicit dedupe key |

## How /pr-gate uses this
During synthesis (step 6), sweep the diff against this table once. Principle findings rank as:
- **Blocker** only when paired with a correctness consequence (silent fallback hiding data loss, non-idempotent migration).
- **Should-fix** for structural violations on new code (new god function, new layering breach, 3rd duplication).
- **Nit** for violations merely *adjacent* to the diff (pre-existing — mention once under Skipped).
Never demand a refactor of untouched code to satisfy a principle.
