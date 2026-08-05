---
id: testing-quality-tests-that-cannot-fail
domain: testing
category: quality
applies_to: [general]
confidence: verified
sources:
  - https://jestjs.io/docs/expect
  - https://jestjs.io/docs/asynchronous
  - https://testing.googleblog.com/2021/04/mutation-testing.html
  - https://martinfowler.com/bliki/TestCoverage.html
  - https://testing.googleblog.com/2013/05/testing-on-toilet-dont-overuse-mocks.html
  - https://martinfowler.com/bliki/DefinitionOfRefactoring.html
  - https://pitest.org/quickstart/mutators/
  - https://docs.pytest.org/en/stable/how-to/monkeypatch.html
last_verified: 2026-08-05
related: [testing-quality-minimum-case-set, testing-quality-behavior-not-implementation, testing-mocking-what-to-mock, testing-async-async-testing, testing-quality-checks-that-cannot-pass, testing-quality-spec-artifact-checks, testing-quality-harness-reverse-controls, qa-document-verification-spec-document-gates, testing-quality-baselines-from-published-artifacts]
---

# Proving a Test Can Fail

## When this applies

You are reviewing tests that always pass, a bug shipped through an area the
suite reported as covered, or you are auditing a suspiciously green suite.

## Do this

1. **A test proves something only if it can fail.** Verify by breaking the code
   under test once: mutate the behavior the test claims to guard (flip the
   condition, change the returned value), rerun the test, and require red. If
   it stays green, the test is decoration — locate its defect in the table
   below and fix the test, then re-verify red before restoring the code.
   This is manual mutation testing; run it whenever a test's value is in doubt.
2. Fix each never-fails pattern with its replacement:

| Never-fails pattern | Fix |
|---------------------|-----|
| Assertion inside a callback or branch that never runs (promise `.then`, event handler, `if` body) | Count assertions with `expect.assertions(n)` / `expect.hasAssertions()`, or restructure to await-then-assert on the test's main path ([testing-async-async-testing]) |
| Assertion swallowed by `try/catch`, or a `.catch` that ignores the error | Remove the catch and let the failure throw; for an expected failure, assert the rejection explicitly (next row) |
| Error-path test that passes when no error is thrown (`expect` sits in the `catch` block; nothing asserts the throw happened) | Use `await expect(...).rejects.toThrow(ErrorType)` / `assertThrows`-style APIs, which fail when the code succeeds |
| Always-true assertion (`toBeDefined`/`toBeTruthy` on a value that is always defined, `expect(arr.length).toBeGreaterThanOrEqual(0)`) | Assert the specific expected value or shape — the observable-outcome rule in [testing-quality-minimum-case-set] |
| Testing the mock instead of the code (mock returns X, test asserts X came back) | Assert the unit's transformation of its inputs, not the pass-through; when no transformation exists at this layer, test the layer that has one ([testing-mocking-what-to-mock]) |
| Copied test body with the name changed but identical inputs and expectation | Give each case distinct inputs and its own expectation; delete exact duplicates — a renamed copy re-proves the same fact and guards nothing new |
| Regression test for a value-preserving refactor (magic literal → named constant or config lookup) that asserts the rendered output (`assert "DB ×1.3" in out`) | Assert the *dependency* instead of the value: substitute a sentinel into the constant for the duration of the test and require the output to follow it. Refactoring is defined as changing internal structure "without changing its observable behavior", so at the shipped config value the reverted refactor is an equivalent mutant — no assertion that pins the value can distinguish the two |

3. **Coverage note:** a covered line is only an executed line. Use coverage to
   find untested code; it cannot certify tested behavior. The proof a test
   works is the red run from step 1, not the coverage report.

## Edge cases

| Case | Then |
|------|------|
| Mutating the code under test is impractical right now (slow build, shared branch) | Invert the expected value in the assertion instead and require red — this proves the assertion executes and compares, though not which code defects it catches |
| Auditing a whole suite, not one test | Run an automated mutation-testing tool (PIT, Stryker) and treat surviving mutants in changed code as missing or defective tests |
| The mutation run is your own script rather than PIT/Stryker | Prove the harness discriminates before citing its score — a semantics-preserving no-op must survive ([testing-quality-harness-reverse-controls]) |
| A test intentionally has no outcome assertion (smoke test: module loads, page renders) | Keep it only when the regression it guards manifests as a throw; name it as a smoke test so reviewers do not count it as behavior coverage |
| The always-green test is a snapshot approved without reading | Snapshot rules → [testing-quality-behavior-not-implementation] |
| Unsure whether a refactor's test can see the refactor | Render the output from the new constant and compare it byte-for-byte with the literal the refactor removed. Identical means the value assertion is decoration; differing means the refactor changed behavior too and that is a separate finding to report |
| Applying the sentinel by hand with `try/finally` | Use the runner's scoped patching (`monkeypatch.setattr` for an attribute, `monkeypatch.setitem` for a config dict) — "All modifications will be undone after the requesting test function or fixture has finished", so an assertion failure cannot leak the sentinel into the next test |
| The sentinel does not reach the code (output keeps the old value) | Patch the reference the consumer looks up, not the defining module — with `from config import RATE` the name is bound in the consumer's namespace at import. When the value is frozen at import time (default argument, module-level f-string), no runtime patch reaches it: that import-time binding is the finding, and the seam to test is the render function's parameter |
| A mutation survives while dozens of assertions pass | The oracle may never read the field the mutation changes — enumerate the returned fields against the assertions ([testing-quality-minimum-case-set]) before concluding the mutation is equivalent |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Add `expect(result).toBeDefined()` to give a test "an assertion" | Assert the specific value/shape the behavior guarantees | `toBeDefined` on an always-defined value passes for every behavior, including broken |
| Prove an error path with `try { await f() } catch (e) { expect(e.message)... }` alone | Use `rejects`/`assertThrows`-style assertion, or add `expect.assertions(1)` above the try | When `f()` succeeds, the catch never runs and the test passes with zero assertions |
| Trust "green suite + high coverage" as proof an area is tested | Break the behavior once and require a red run | Coverage counts execution, not detection; high numbers are reachable with assertion-free tests |
| Delete a suspicious always-green test to clean up | Fix it via the table above, then re-verify it can fail | The test names a behavior someone meant to guard; deletion drops the intent along with the defect |
| Guard a literal-to-constant refactor by asserting the string the constant currently produces | Perturb the constant to a sentinel and assert the output tracks it | The literal and the constant render the same bytes today, so the assertion passes with the literal pasted back — PIT automates the same perturbation as its inline-constant mutator |

## Sources

- https://jestjs.io/docs/expect — `expect.assertions(n)` / `expect.hasAssertions()` guard callback assertions; `.rejects`, `.toThrow`
- https://jestjs.io/docs/asynchronous — un-awaited promises let tests finish early; `.rejects`; `expect.assertions` with try/catch
- https://testing.googleblog.com/2021/04/mutation-testing.html — inserting faults and requiring test failure measures whether tests detect bugs; coverage alone does not
- https://martinfowler.com/bliki/TestCoverage.html — coverage finds untested code; it is not a measure of test quality
- https://testing.googleblog.com/2013/05/testing-on-toilet-dont-overuse-mocks.html — mock-heavy tests can pass while the real code is broken
- https://martinfowler.com/bliki/DefinitionOfRefactoring.html — refactoring is "a change made to the internal structure of software … without changing its observable behavior", so an assertion on observable output cannot distinguish refactored code from the original
- https://pitest.org/quickstart/mutators/ — the Inline Constant mutator "mutates inline constants… a literal value assigned to a non-final variable", replacing `1` with `0` or otherwise incrementing the value: perturbing the constant and requiring detection is the automated form of the sentinel rule
- https://docs.pytest.org/en/stable/how-to/monkeypatch.html — `monkeypatch.setattr`/`setitem` for attributes and config dicts; "All modifications will be undone after the requesting test function or fixture has finished"; "Prefer patching the reference that your code uses instead of patching the original object"
- Field reproduction 2026-08-05 (Python report generator): a report line rendered `"DB ×%s" % E.DB_MULT` with `DB_MULT = 1.3`, producing exactly the `DB ×1.3` literal the refactor had removed, so `assert "DB ×1.3" in out` stayed green with the literal restored. Substituting a sentinel multiplier made the literal version red
