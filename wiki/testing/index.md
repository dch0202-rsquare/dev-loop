# testing — Domain Index

Route here for: writing or structuring automated tests — choosing the test
level, selecting cases and assertions, test data and isolation, mock/fake
decisions, fixing flaky tests, verifying tests can actually fail, testing
async code (promises/timers/events), browser E2E selector/wait/setup
strategy, and automated checks over documents that act as spec.
Release-process quality (gates, manual testing, bug triage) → wiki/qa/.

Match your situation to a "load when" line; load only matching pages.

## strategy

| Page | Load when |
|------|-----------|
| [test-level-choice](strategy/test-level-choice.md) | Deciding at which level (unit/integration/e2e) to test new or changed behavior; reviewing a test plan's level distribution; logic buried in a controller or framework wiring needs coverage |

## quality

| Page | Load when |
|------|-----------|
| [minimum-case-set](quality/minimum-case-set.md) | Writing tests for a function/endpoint/change and choosing which cases to cover; reviewing whether coverage suffices; picking boundary values by input type; adding a regression test for a bug fix |
| [behavior-not-implementation](quality/behavior-not-implementation.md) | Deciding what a test should assert; a behavior-preserving refactor broke tests; tempted to expose privates for testing; deciding whether a snapshot test is appropriate |
| [tests-that-cannot-fail](quality/tests-that-cannot-fail.md) | Reviewing tests that always pass; a bug shipped through an area the suite reported as covered; auditing a suspiciously green suite; judging whether an assertion, error-path test, or mock-based test can actually detect a defect |

## data

| Page | Load when |
|------|-----------|
| [test-data-and-isolation](data/test-data-and-isolation.md) | Tests need fixture data and you are choosing how to create it; tests pass alone but fail together (or vice versa); DB cleanup, shared fixtures, time-dependent logic, or unique-value collisions |

## mocking

| Page | Load when |
|------|-----------|
| [what-to-mock](mocking/what-to-mock.md) | Deciding whether to mock/stub/fake a dependency or use the real one; mocks breaking on refactors; testing handling of a third-party's failure modes; the same mock setup is copy-pasted across tests |

## flaky

| Page | Load when |
|------|-----------|
| [diagnosing-flaky-tests](flaky/diagnosing-flaky-tests.md) | A test fails intermittently with no code change: on retry, in CI only, or only when run with other tests; deciding policy for a newly identified flaky test (quarantine vs retry) |

## async

| Page | Load when |
|------|-----------|
| [async-testing](async/async-testing.md) | Testing async code — promises, timers, retries, debounce, event-driven flows; the runner warns about assertions after completion or un-awaited promises; an async test intermittently interferes with the next test; deciding between fake timers and condition waits |

## e2e

| Page | Load when |
|------|-----------|
| [e2e-stability](e2e/e2e-stability.md) | Writing browser E2E tests (Playwright/Cypress-style); an E2E suite is flaky or slow; choosing selectors (role/label vs test id vs CSS), wait strategy, auth/data setup layer, or what to stub at the network edge |

## docs-as-spec

| Page | Load when |
|------|-----------|
| [gate-falsifiability](docs-as-spec/gate-falsifiability.md) | Writing or adopting an automated check over a spec document (grep/regex gate, mapping-table coverage check, cross-document signature check); the target document does not exist yet so the gate's first run is red; deciding whether a passing coverage check licenses "verified"; designing one negative control per check |
| [markdown-table-parsing](docs-as-spec/markdown-table-parsing.md) | A checker reads Markdown tables programmatically — counting cells, extracting a column, asserting table shape; a row containing an escaped pipe (`\|` in EBNF/enum/type-union cells) is reported as broken while it renders correctly; deciding whether the row or the checker is wrong |
