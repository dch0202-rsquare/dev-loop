---
id: testing-quality-document-verification-gates
domain: testing
category: quality
applies_to: [general]
confidence: verified
sources:
  - https://pubs.opengroup.org/onlinepubs/9699919799/utilities/grep.html
  - https://www.rfc-editor.org/rfc/rfc2119
  - https://www.rfc-editor.org/rfc/rfc8174
  - https://testing.googleblog.com/2021/04/mutation-testing.html
last_verified: 2026-07-30
related: [testing-quality-tests-that-cannot-fail, qa-process-acceptance-criteria, platforms-environment-text-encoding-and-normalization]
---

# Automated Gates That Check a Document Meets Its Spec

## When this applies

You are writing or reviewing an automated check (grep/script) that verifies a
**document** satisfies stated requirements — an RFC, API spec, schema doc, or a
plan's definition-of-done — including gates authored before the document exists.

## Do this

A keyword-presence check answers "does this token appear somewhere", which is
not the requirement. Build each gate on the axis that carries the requirement:

| Axis | Check | Defeats |
|------|-------|---------|
| Structure | Parse the table: row count, column count, empty cells | Deleting the table while the token survives in prose |
| Modality | Assert the requirement word itself in the same sentence scope (`MUST` vs `SHOULD`, `필수` vs `선택`) | Downgrading a requirement to a recommendation |
| Polarity | Assert no negation in that same sentence scope (`not required`, `요구하지 않는다`) | Negating the requirement while keeping every token |
| Completeness | Assert an exact count for closed sets (`enum has exactly 5 rows`), not `≥1` | Dropping one member of an enum |
| Cross-reference | Recompute the value in another section and compare the two | A number that is right in one place and stale in the other |

Then make the gate itself falsifiable:

1. **Prove each gate can fail.** Write a copy of the document with that one
   defect planted, run the gate, and require FAIL. A gate with no such negative
   control is untested code guarding a deliverable — the document analogue of
   the red-run rule in [testing-quality-tests-that-cannot-fail]. Keep the
   planted-defect copies as a mutation self-test that runs with the gate.
2. **Validate the pattern on a sibling before the target exists.** When the gate
   is written first (the document is not yet drafted), run the pattern against an
   existing file that already meets the same spec and require the expected count.
   Absence makes every pattern fail, so a red run proves nothing about the
   pattern; a typo'd anchor and a correct anchor both go red.
3. **Fail closed on every non-match outcome.** `grep` exits 0 for a match, 1 for
   no match, and >1 for an error (missing file, bad regex). Treat 1 and >1
   differently: >1 is a broken gate, not an unmet requirement.
4. **Match on the exact text form.** Pattern and document must be in the same
   Unicode normalization form, and stem-prefix matching does not work for
   syllabic scripts — [platforms-environment-text-encoding-and-normalization].

## Edge cases

| Case | Then |
|------|------|
| An `Examples` section repeats the rule the gate is checking | Anchor the gate to the normative section only (line range or heading scope); an example passing while the rule is negated is the exact blind spot |
| The requirement is prose with no fixed wording | Gate on the structural artifact it must produce (a table row, a named heading, a code fence), and review the prose by human read — record which requirements are review-only |
| `grep -c` used to count occurrences | It counts *matching lines*, so two hits on one line report 1; count with `grep -o \| wc -l` when the unit is occurrences |
| The document is generated from a source of truth | Gate the source, and add one check that the generated copy is current (regenerate and diff) |
| The gate is for a plan's definition-of-done | Keep the same axes; criteria wording rules → [qa-process-acceptance-criteria] |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Accept a gate because it goes red against the not-yet-written document | Run it against a sibling file that already meets the spec and require the expected count | Absence fails every pattern; red is evidence the file is missing, not that the pattern is right |
| Assert `grep -q '필수'` to check a field is mandatory | Assert the modality word in that field's row, plus a negation check in the same scope | RFC 2119 requirement level lives in the modal word; `SHOULD` and `MUST` both contain the token being searched for |
| Assert an enum's members exist with `grep -c ≥ 1` | Assert the table has exactly N rows | Deleting one member leaves the count non-zero, so the gate stays green with the enum broken |
| Ship the gate once it passes on the good document | Also run it on a copy with the defect planted and require FAIL | A gate that passes everything is decoration; the fault-injection run is what measures detection |

## Sources

- https://pubs.opengroup.org/onlinepubs/9699919799/utilities/grep.html — exit status 0 "one or more lines were selected", 1 "no lines were selected", >1 "an error occurred"; `-c` writes a count of selected *lines*; `LC_CTYPE` governs multibyte interpretation
- https://www.rfc-editor.org/rfc/rfc2119 — MUST is "an absolute requirement"; SHOULD means "there may exist valid reasons in particular circumstances to ignore a particular item" — the requirement level is carried by the modal word
- https://www.rfc-editor.org/rfc/rfc8174 — the RFC 2119 meanings apply "only when they are in all capitals"; lowercase variants carry no requirement level, so a case-insensitive keyword gate cannot distinguish them
- https://testing.googleblog.com/2021/04/mutation-testing.html — Goran Petrovic, "Mutation Testing" (Google Testing Blog, 2021-04-12): inserting faults and requiring a failure measures whether checks detect defects
