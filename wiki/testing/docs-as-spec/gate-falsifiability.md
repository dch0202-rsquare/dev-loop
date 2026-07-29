---
id: testing-docs-as-spec-gate-falsifiability
domain: testing
category: docs-as-spec
applies_to: [general]
confidence: field-tested
sources:
  - https://www.industriallogic.com/blog/tdd-youre-doing-it-wrong/
  - https://testing.googleblog.com/2021/04/mutation-testing.html
  - https://martinfowler.com/bliki/TestCoverage.html
last_verified: 2026-07-29
related: [testing-quality-tests-that-cannot-fail, testing-docs-as-spec-markdown-table-parsing, debugging-methodology-verify-the-fix]
---

# Proving a Document-as-Spec Gate Can Fail for the Right Reason

## When this applies

You are writing or adopting an automated check over a document that acts as a
spec: a grep/regex gate on an RFC or schema doc, a "every rule has a row"
coverage check on a mapping table, or a cross-document signature check. The
target document is unwritten, half-written, or owned by someone else — so the
gate's first run happens before the thing it checks exists.

## Do this

1. **Pin each check against an artifact whose answer you already know, before you
   trust its verdict.** A gate has two verdicts and each can be right by accident.

| Verdict you have | What it does not yet prove | Make it proof by |
|------------------|----------------------------|------------------|
| Red, and the target file does not exist yet | That the pattern is correct — a typo'd or mis-anchored pattern is equally red against a missing file | Running the same pattern against a **sibling file that already satisfies the spec** and requiring the exact expected count |
| Green on a coverage check (every required row is present) | That the cells are right — set-equality over row *names* does not read cell *contents* | Adding a separate value-validity check: every cell's value must exist in the canonical enum / schema / id set |
| Red after you broke something on purpose | That *this* check caught it — a coarse mutation trips several checks at once | Mutating only what this one check owns, and requiring red from that check alone |

2. **Split presence from validity into two checks with two names.** One check
   answers "is every required row here?", the other answers "does every cell hold
   a value that exists?". A single check that reports both cannot tell you which
   failed, and a single green cannot mean both passed.

3. **Write one negative control per check, mutating only that check's subject.**
   For a coverage check, delete a row. For a value-validity check, leave every row
   in place and corrupt one cell to a value outside the canonical set. Require the
   owning check to go red and record what the other checks said — when a
   coverage-only harness stays green under a corrupted cell, that green is the
   result worth writing down, because it is what your gate would have shipped.

4. **State each check's verdict in terms of what it verified.** A passing
   coverage check licenses "the mapping is *present*", not "the mapping is
   *verified*". Name the check after the claim it can defend.

## Edge cases

| Case | Then |
|------|------|
| No sibling file satisfies the spec yet (the document is the first of its kind) | Write a throwaway fixture that satisfies the spec by construction, pin the pattern's expected count against it, and delete the fixture once the real document passes |
| The canonical enum/id set lives in another document or a code file | Extract it at check time from that source and diff against it; a hand-copied list in the checker drifts and then validates cells against a stale set |
| The gate must pass before the document exists (CI on an early branch) | Make absence a distinct, named outcome (`skipped: target absent`) rather than a red, so a real pattern failure is not hidden inside an expected red |
| A single mutation makes several checks red | Narrow the mutation until exactly one check fails; if that is impossible, the checks share a subject — merge them or re-cut their boundaries |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Accept "exits non-zero today, so the gate works" for a not-yet-written target | Run the pattern against a sibling that already conforms and require the exact expected count | Absence makes every pattern red, so red carries no information about the pattern; the error surfaces later as a gate that fails forever or one that never fires |
| Report "mapping verified" when the row-coverage check passed | Report "mapping present", and add the value-validity check before claiming verified | A table can hold 100% of required rows with a typo'd, empty, or swapped cell |
| Prove a whole harness with one broad mutation | Give each check its own single-subject mutation | A broad mutation that reds the suite leaves each individual check unproven |

## Sources

- https://www.industriallogic.com/blog/tdd-youre-doing-it-wrong/ — "If you don't start with a test that *fails for the right reason*, you are not doing TDD"; the failure must come from the missing behavior, not from incidental breakage
- https://testing.googleblog.com/2021/04/mutation-testing.html — inserting a fault and requiring the test to fail is what measures detection; coverage alone does not
- https://martinfowler.com/bliki/TestCoverage.html — coverage locates unchecked material; it does not certify that what is checked is correct
- Field context: a documentation-verification harness for an RFC suite (grammar→IR mapping table, 51 rules, 33 golden rows). Corrupting one mapping cell to a nonexistent kind and swapping a golden node id for a nonexistent one both left the row-coverage checks green; the added value-validity checks caught both, and the harness prints what the coverage-only gate would have reported.
