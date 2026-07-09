# Test-Slop Catalog — false-confidence patterns with detection greps and rewrites

Per pattern: what it looks like (Scala/Python/JS), how to grep for candidates, and the
behaviour-test rewrite. Greps find *candidates* — always read the test before flagging.

## 1. Mock echo (tests the mocking library)

```scala
when(dao.findAlert("A1")).thenReturn(Some(alert))
service.getAlert("A1") shouldBe Some(alert)   // asserts the stub, service logic untested
```
```python
mock_repo.get.return_value = {"status": "OPEN"}
assert svc.fetch("A1") == {"status": "OPEN"}   # same object back
```
```js
fetchMock.mockResolvedValue({ok: true});
expect(await load()).toEqual({ok: true});
```
Grep candidates: the same literal/fixture appearing in both the stub arrange and the assert.
`grep -n "thenReturn" f | ...` then eyeball; JS: `mockResolvedValue\|mockReturnValue`.
**Rewrite:** make the SUT *transform* — assert the derived/mapped/filtered output, error mapping,
or a value the stub did NOT contain verbatim. If the method is a pure pass-through, say so and
recommend deleting the test.

## 2. Hardcoded oracle from implementation output

Expected value was produced by running the code and pasting the result (magic
floats/hashes/long JSON blobs nobody derived by hand; snapshot tests committed blind).
```scala
score(txns) shouldBe 0.7342118           // where did this number come from?
```
Detection: high-precision literals, base64/hash literals, `toMatchSnapshot()` with no reviewed
snapshot file, expected JSON identical to a fixture the code loads.
**Rewrite:** derive the oracle from the SPEC (hand-computed small case, closed-form bound,
property: `score ∈ [0,1]`, monotonicity), or assert against an independently-constructed value.

## 3. Tautological assert

```scala
result shouldBe result
parse(render(x)) shouldBe parse(render(x))
```
```python
assert build_config() == build_config()   # both sides same call
```
Detection grep: identical expression both sides of `shouldBe`/`==`/`toEqual`; asserting a
fixture field against the same fixture field.
**Rewrite:** one side must be an independent expectation. Round-trip tests are fine ONLY as
`parse(render(x)) == x` with `x` hand-built.

## 4. Over-mocking / mocking the SUT

Everything the method touches is stubbed — the only unmocked lines are glue; or a spy stubs the
very method under test. React: rendering a component with all children/hooks mocked, asserting
mock props.
Detection: count mocks vs real collaborators; `spy(sut)` / `jest.spyOn(component, methodUnderTest)`.
**Rewrite:** mock only the process boundary (DB/HTTP/clock/random). Pure logic runs real. If the
class can't be tested without mocking half of it, that's a design finding — extract the logic.

## 5. Verify-only with any() matchers

```scala
verify(alertDao).save(any[Alert])      // saved WHAT? wrong fields still pass
```
```python
mock_publisher.publish.assert_called_once()   # payload unchecked
```
Detection grep: `verify(.*any[(\[]` · `assert_called` without `assert_called_with` ·
`toHaveBeenCalled()` (no `With`).
**Rewrite:** capture the argument (ArgumentCaptor / `assert_called_once_with(expected)` /
`toHaveBeenCalledWith(expected)`) and assert the fields that matter to the behaviour.

## 6. Can-never-fail

- No assert at all (test = "it doesn't throw"). Detection: test body without any
  assert/should/expect token.
- Assert inside a lambda that never executes: `futures.foreach(f => f.map(_ shouldBe x))` —
  un-awaited; pytest `async def` without the async plugin marker; JS missing `await expect(...)`.
- Swallowed failure: `try { assert... } catch { case _ => }`; `except Exception: pass` in a test.
- `@Ignore`/`xit`/`test.skip`/`@pytest.mark.skip` left permanently. DEAD — report count.
**Rewrite:** await the future/promise (`whenReady`, `Await.result`, `await`), remove the
swallow, or delete the test.

## 7. Happy-path-only

Source has `if (xs.isEmpty)` / `Option` / error branches; test file exercises only the populated
happy case. Detection: diff the source's branch conditions against test inputs.
**Rewrite:** one test per observable behaviour class: empty input, boundary (0, 1, max, off-by-one),
error propagation, null/None. Cite the untested source `file:line` branch.

## 8. Copy-paste matrix padding

12 near-identical tests, different input literal, same asserted output — usually generated to
inflate counts. Detection: >3 tests differing only in one literal with identical assert shape.
**Rewrite:** table-driven/parameterized test with cases that each pin a DIFFERENT behaviour
(boundary, sign flip, category change). If all cases assert the same thing, most are padding — delete.

## 9. Shared mutable fixture / order dependence

`var` / class-level mutable state reused across tests; suite passes only in order. Detection:
`var ` in test class scope, `beforeAll` populating shared mutable collections, module-level
mutation in pytest.
**Rewrite:** fresh fixture per test (`beforeEach`, pytest function-scope fixture).

## 10. Sleep-based async

`Thread.sleep(2000)` / `time.sleep` / `setTimeout` before asserting. Flaky-green: passes when
the race happens to resolve. Detection grep: `sleep(` inside test dirs.
**Rewrite:** await the condition — `eventually`, `whenReady`, `waitFor`, awaitility-style polling
with timeout.

## 11. Structure-mirroring (implementation coupling)

One test per private method, asserting each internal step; refactor breaks 40 tests, real bug
breaks none. Also: asserting exact SQL strings, exact log lines, internal call order.
**Rewrite:** test the public contract of the module; private methods get covered through it.
Keep at most deliberate seam tests, flagged as IMPLEMENTATION knowingly.

## 12. Assertion-free integration test

Spins up docker/ES/DB, hits the endpoint, asserts `status == 200` only — body, DB row, emitted
event unchecked. Detection: integration specs whose only matcher is on status/response-code.
**Rewrite:** assert the observable contract: response body fields, the row actually persisted
(read it back), the Kafka message actually published (consume it).

## The falsifiability check (use on every suspicious test)

Name the smallest source mutation that SHOULD fail the test:
- flip a comparison (`>` → `>=`), off-by-one a boundary, return `Nil`/`None`/empty, drop a filter,
  swap two fields in the mapper.
If no such mutation would fail it → TAUTOLOGY/DEAD. Put the mutation in the finding so the
author can verify in 30 seconds.

## Scoring a suite

`real_coverage = behaviours_with_BEHAVIOUR_test / critical_behaviours` (enumerate critical
behaviours from the spec/AC, not from the code). Report alongside the raw test count so
"245 tests" and "9 behaviours actually pinned" can be seen in one line.
