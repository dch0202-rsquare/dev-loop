---
id: qa-process-defect-class-sweep-over-a-fix
domain: qa
category: process
applies_to: [general]
confidence: verified
sources:
  - https://codeql.github.com/docs/codeql-overview/about-codeql/
  - https://www.eecg.utoronto.ca/~yuan/papers/incorrect_fix_abstract.html
  - https://dl.acm.org/doi/10.1145/2025113.2025121
last_verified: 2026-08-11
related:
  [
    debugging-methodology-verify-the-fix,
    qa-process-regression-scope,
    backend-common-change-impact-call-site-enumeration,
  ]
---

# Sweeping a Review's Defect Class Over Your Own Fix

## When this applies

You are about to submit a diff that responds to review feedback or fixes a
reported defect, and that diff adds new functions, branches, or call sites.
Also when a later review round raises the same defect the previous round
closed, in a different function of the same change.

Choosing which *tests* to re-run for a change → [qa-process-regression-scope].
Confirming the original symptom is gone → [debugging-methodology-verify-the-fix].

## Do this

1. **Name the class from the reviewer's own words**, not the location they
   pointed at: "external string reaches the sink unwrapped", "missing null
   guard before the accessor", "query without a bound". The named class is the
   seed — variant analysis is "the process of using a known security
   vulnerability as a seed to find similar problems in your code".

2. **Run the class query over both arms**, and require zero on each:

| Arm | How to scope it | Why the review did not cover it |
|-----|-----------------|---------------------------------|
| Pre-existing code | The class query over the repo | The reviewer listed the instances they saw, not the set |
| Code this diff adds | The class query restricted to added lines (`git diff -U0` `+` lines, or the diff's new files) | This code did not exist in the tree the reviewer read |

   The second arm is the one reviews structurally cannot reach, and it is where
   a fix reopens the class it just closed: a guard added to one handler while
   the sibling handler extracted in the same commit ships without it.

3. **Prove the query discriminates before trusting a zero.** Point it at the
   flagged site in its pre-fix state — `git stash` is not needed, the blob is at
   `HEAD~`/the review's base — and require a hit. A query that returns zero
   everywhere reports the query's shape, not the code's.

4. **When the class is machine-expressible, land the rule with the fix** — a
   lint rule, a CodeQL query, a test — instead of a one-time grep. The rule
   covers the next diff; the grep covers this one.

5. **Give the fix diff the review depth of the original change, not less.**
   Measured across four operating systems, "at least 14.8% to 24.4% of sampled
   fixes for post-release bugs in these large OSes are incorrect", and "27% of
   the incorrect fixes are made by developers who have never touched the source
   code files associated with the fix".

## Edge cases

| Case | Then |
|------|------|
| The review names one instance rather than a rule | Ask the reviewer for the rule before sweeping; a search for the flagged symbol returns exactly the site you already fixed |
| The class is a judgment call no query expresses (naming, layering, readability) | Enumerate the diff's added functions by hand and check each against the reviewer's stated rule — the enumeration is the substitute for the query |
| The sweep returns zero on a class you know exists | The query is wrong; re-run step 3 against the pre-fix site before recording zero |
| The sweep finds instances outside this change's scope | File them as their own work; growing the fix diff to cover them puts more unreviewed code in the highest-risk diff |
| The fix was authored by someone new to these files | Add the sweep's output to the review request explicitly — unfamiliarity is the measured correlate of an incorrect fix |
| The reviewer marked the thread resolved | Resolution records a reply; confirm the class query is zero over the file before treating the thread as closed |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Re-check only the sites the reviewer listed | Run the class query over the repo *and* over the diff's added lines | The reviewer's list is bounded by the tree at review time, and the fix changed that tree |
| Treat "all review comments addressed" as the completion condition | Require the class query to return zero over the added lines too | A comment is answered per instance; the defect exists per class |
| Fix each flagged site individually as the comments arrive | Derive the class first, then fix every member in one pass | Per-instance fixes leave the class intact and reopen it in the next round |
| Ship the fix under a lighter review because it is small | Route it through the same depth as the original change | Fix diffs carry a measured 14.8–24.4% incorrect rate and have no prior verification history |

## Sources

- https://codeql.github.com/docs/codeql-overview/about-codeql/ — "Variant analysis is the process of using a known security vulnerability as a seed to find similar problems in your code"; running such a query across codebases lets you "identify all of the places where that vulnerability exists" — the seed-to-class step this page applies to review feedback
- https://www.eecg.utoronto.ca/~yuan/papers/incorrect_fix_abstract.html — "at least 14.8% to 24.4% of sampled fixes for post-release bugs in these large OSes are incorrect"; "27% of the incorrect fixes are made by developers who have never touched the source code files associated with the fix"
- https://dl.acm.org/doi/10.1145/2025113.2025121 — Yin, Yuan, Zhou, Pasupathy, Bairavasundaram, "How do fixes become bugs?", ESEC/FSE 2011, the study those figures come from
- Field incident 2026-08-11 (Python review round): round 1 flagged one function for passing an unescaped external string into a report line; the fix closed that site and, in the same commit, added a sibling validator that opened the identical hole. An independent audit reproduced it with a control-character-bearing input; a class sweep restricted to the diff's added lines would have listed it before submission
