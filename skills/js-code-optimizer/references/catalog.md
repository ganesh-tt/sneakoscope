# JS/TS Optimizer Catalog

Citation-backed examples. URLs are the primary sources to quote from (≤ 25 words verbatim).
MDN = https://developer.mozilla.org · TS = https://www.typescriptlang.org/docs · TC39 = https://tc39.es

## Correctness / footguns

### == vs ===
`==` applies type coercion; `===` does not. Default to `===`/`!==`.
```js
if (x == null) {}   // ok idiom for null|undefined; otherwise ===
```
Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Strict_equality

### var hoisting / loop capture
```js
for (var i=0;i<3;i++) setTimeout(()=>console.log(i)); // 3 3 3
```
```js
for (let i=0;i<3;i++) setTimeout(()=>console.log(i)); // 0 1 2
```
Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let

### Floating `this` — arrow functions capture lexical `this`; `function` re-binds it. Don't swap them blindly (changes `this`).
Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions

### Missing await / unhandled rejection — a dropped Promise fails silently. Always `await` or `.catch`.
Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises

### for...in over arrays — iterates enumerable keys incl. inherited; use `for...of` / index loop.
Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of

## Modern ES idiom adoption

- `const`/`let` over `var`. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const
- Optional chaining `?.`. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining
- Nullish coalescing `??` (differs from `||` — only null/undefined). Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing
- Destructuring + defaults. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment
- Template literals over `+` concat. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals
- Spread/rest over `apply`/`arguments`. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax
- `Object.entries`/`fromEntries`, `Array.prototype.flatMap`/`at`. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/flatMap
- `structuredClone` for deep copy. Cite: https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone
- `Map`/`Set` over object-as-dictionary (key ordering, non-string keys, no proto collisions). Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map
- `#private` class fields. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_properties

## Functional-programming & effect hygiene

- `map`/`filter`/`reduce`/`some`/`every`/`find` over imperative accumulation. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce
  - **Perf**: each chained `.map().filter()` allocates a new array — on large data classify `needs-benchmark`; fuse into one `reduce`/loop if hot.
- Pure functions — return new values; don't mutate arguments. Immutable update via spread / `structuredClone`, not in-place.
- No side effects inside `.map` (use `.forEach` or a loop when the point is an effect).
- Early return over deep nesting; small composed functions over one god function.
- Lazy sequences via generators / `Array.from(iterable, fn)`. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/from

## Async / Promise hygiene

- `async/await` over nested `.then` / callbacks. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
- `Promise.all` for independent awaits (not sequential). Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all
- `Promise.allSettled` when one failure must not abort the rest. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled
- No `async` function as `new Promise` executor (rejections lost). No `await` in a `for` loop for parallelizable work → collect promises, `await Promise.all`.

## Type-safety (TypeScript)

- Replace `any` with `unknown` + narrowing. Cite: https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown
- Discriminated unions over boolean-flag objects. Cite: https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions
- `as const` for literal narrowing. Cite: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-4.html#const-assertions
- `satisfies` — check against a type without widening the inferred literal. Cite: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator
- `readonly` / `Readonly<T>` for value objects. Cite: https://www.typescriptlang.org/docs/handbook/2/objects.html#readonly-properties
- Exhaustive `switch` with `never` default to catch unhandled union members. Cite: https://www.typescriptlang.org/docs/handbook/2/narrowing.html#exhaustiveness-checking
- Avoid non-null `!` assertions — narrow instead.

## Performance

- Fuse array chains on large data (one `reduce`/loop vs `.map().filter().map()`). `needs-benchmark`.
- Spread-in-loop is O(n²): `acc = [...acc, x]` inside a loop → `acc.push(x)`. Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax
- `Set.has` / `Map.get` over `Array.includes` for repeated membership.
- Don't create functions/objects inside hot render paths (React) — hoist or memoize.
- Batch DOM reads/writes to avoid layout thrash. Cite: https://developer.mozilla.org/en-US/docs/Web/Performance

## Data access / N+1 (high severity)

### Await-in-loop fetch → batch + Map join
```js
for (const o of orders) {
  const c = await api.getCustomer(o.customerId);   // N sequential round-trips
}
```
```js
const custs = new Map((await api.getCustomers([...new Set(orders.map(o => o.customerId))]))
  .map(c => [c.id, c]));                            // 1 round-trip
for (const o of orders) { const c = custs.get(o.customerId); }
```
No batch endpoint? `Promise.all` the lookups (parallel, still N calls — say so) and flag the missing batch API.
Detection: `await .*(getById|findById|fetch\()` inside `for`/`.map`.

### Nested-array lookup O(n·m)
```js
items.map(i => users.find(u => u.id === i.userId))       // quadratic
```
```js
const byId = new Map(users.map(u => [u.id, u]));
items.map(i => byId.get(i.userId))                        // linear
```
Cite: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map

### Fetch-all-then-filter client-side
Pull the predicate/pagination into the API query params; unbounded fetches need paging.
