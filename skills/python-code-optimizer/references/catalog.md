# Python Optimizer Catalog

Citation-backed examples. URLs are the primary sources to quote from (≤ 25 words verbatim).

## Correctness / footguns

### Mutable default argument
Defaults evaluated once at def time — a shared mutable default leaks state across calls.
```python
def add(x, acc=[]):      # BUG: acc shared across calls
    acc.append(x); return acc
```
```python
def add(x, acc=None):
    acc = [] if acc is None else acc
    acc.append(x); return acc
```
Cite: https://docs.python.org/3/reference/compound_stmts.html#function-definitions — "Default parameter values are evaluated ... once, when the function definition is executed."

### Bare / over-broad except
`except:` catches `SystemExit`/`KeyboardInterrupt`; `except Exception` swallowing hides bugs. Catch the narrowest type; re-raise or log.
Cite: https://docs.python.org/3/tutorial/errors.html

### Late-binding closure in loop
```python
fns = [lambda: i for i in range(3)]   # all return 2
```
```python
fns = [lambda i=i: i for i in range(3)]  # or functools.partial
```
Cite: https://docs.python.org/3/faq/programming.html#why-do-lambdas-defined-in-a-loop-with-different-values-all-return-the-same-result

### `is` vs `==`
`is` tests identity, not equality. Use `==` for value comparison; `is` only for `None`/singletons.
Cite: https://docs.python.org/3/reference/expressions.html#is

### Mutating a list while iterating it — iterate over a copy or build a new list.

## Modern idiom adoption

### dataclass over hand-rolled __init__/__eq__/__repr__ (3.7+); `slots=True` (3.10+)
Cite: https://docs.python.org/3/library/dataclasses.html
### enum / StrEnum (3.11+) over string constants
Cite: https://docs.python.org/3/library/enum.html
### pathlib over os.path string juggling
Cite: https://docs.python.org/3/library/pathlib.html
### match/case (3.10+) over long isinstance/elif cascades
Cite: https://peps.python.org/pep-0636/
### X | Y unions (3.10+) over typing.Union / Optional
Cite: https://peps.python.org/pep-0604/
### f-strings over % / str.format
Cite: https://peps.python.org/pep-0498/
### contextlib.suppress / context managers over manual try/finally
Cite: https://docs.python.org/3/library/contextlib.html#contextlib.suppress
### functools.cached_property for expensive derived attrs
Cite: https://docs.python.org/3/library/functools.html#functools.cached_property

## Functional-programming hygiene

- **Comprehension / generator expression** over accumulate-in-loop. Generator = lazy (`needs-benchmark` if the consumer needs a list twice). Cite: https://docs.python.org/3/tutorial/classes.html#generator-expressions
- **itertools** (`chain`, `islice`, `groupby`, `accumulate`, `pairwise` 3.10+) over manual index math. Cite: https://docs.python.org/3/library/itertools.html
- **functools.reduce / partial** for fold / partial application. Cite: https://docs.python.org/3/library/functools.html
- **Pure functions** — return new values instead of mutating arguments; makes reasoning + testing local.
- **Immutability for value objects** — `tuple`, `frozenset`, `@dataclass(frozen=True)`. Cite: https://docs.python.org/3/library/dataclasses.html#frozen-instances
- Prefer a comprehension over `map`/`filter` + `lambda` when it reads clearer; keep `map(func, xs)` when `func` is a named callable.

## Type-safety

- Annotate public function signatures; avoid bare `Any`. Cite: https://docs.python.org/3/library/typing.html
- `Protocol` (structural) over nominal ABC when only shape matters. Cite: https://peps.python.org/pep-0544/
- `TypedDict` / `NamedTuple` over bare dict/tuple payloads. Cite: https://peps.python.org/pep-0589/
- `Literal` / `Final`. Cite: https://peps.python.org/pep-0586/ , https://peps.python.org/pep-0591/
- Builtin generics `list[int]`/`dict[str,int]` (3.9+) over `typing.List`. Cite: https://peps.python.org/pep-0585/

## Performance

- String build in loop → `"".join(parts)`. `+=` on str is O(n²) worst case. Cite: https://docs.python.org/3/library/stdtypes.html#str.join
- Membership test against `list` in a loop → convert to `set` (O(1) vs O(n)). Cite: https://docs.python.org/3/tutorial/datastructures.html#sets
- `collections.Counter` / `defaultdict` over manual `if k in d` counting. Cite: https://docs.python.org/3/library/collections.html
- Hoist repeated global/attribute lookups out of hot loops (bind to local). `needs-benchmark`.
- Don't `list(gen)` if you iterate once — pass the generator. Classify laziness change.
- `dict.setdefault` / `dict.get(k, default)` over key-existence branches.

## Async & concurrency

- No blocking I/O / `time.sleep` inside `async def` — use `await asyncio.sleep`, run blocking work in `asyncio.to_thread`. Cite: https://docs.python.org/3/library/asyncio-task.html
- `asyncio.gather` for independent awaits instead of sequential `await`. Cite: https://docs.python.org/3/library/asyncio-task.html#asyncio.gather
- Forgotten `await` on a coroutine — returns a coroutine object, silent bug.

## Data access / N+1 (high severity)

### Query-in-loop → batch + dict join
```python
for order in orders:
    cust = repo.find_by_id(order.customer_id)   # N round-trips
```
```python
custs = {c.id: c for c in repo.find_by_ids({o.customer_id for o in orders})}  # 1 round-trip
for order in orders:
    cust = custs.get(order.customer_id)
```
Detection: `find_by_id|get_by_id|SELECT .* WHERE id =|session.query` inside `for`/comprehension.
Classification: `improved` (fewer round-trips) — verify a batch API exists; if not, that's the finding.

### Fetch-all-then-filter in Python
`rows = fetch_all(); [r for r in rows if r.status=='OPEN']` → push the predicate into the query/ES filter. Unbounded reads need pagination/limit.

### Per-row DataFrame mutation
`df.append`/row-wise `.loc` writes in a loop → vectorized column ops / `pd.concat` once. `needs-benchmark` only when frames are tiny.
