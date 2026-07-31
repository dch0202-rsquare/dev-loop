---
id: testing-quality-verification-harness-validity
domain: testing
category: quality
applies_to: [general]
confidence: verified
sources:
  - https://stryker-mutator.io/docs/mutation-testing-elements/mutant-states-and-metrics/
  - https://stryker-mutator.io/docs/stryker-js/configuration/
  - https://stryker-mutator.io/docs/
  - https://testing.googleblog.com/2021/04/mutation-testing.html
last_verified: 2026-07-31
related: [testing-quality-tests-that-cannot-fail, testing-data-test-data-and-isolation, debugging-methodology-hypothesis-testing]
---

# Trusting the Harness That Grades Your Tests

## When this applies

You built a harness that runs a suite against modified inputs — a mutation
script, a rule/spec checker, a CI quality gate — and you are about to cite its
numbers as evidence. Especially when the result is uniform: every case caught,
or every case passing.

## Do this

A harness reports two things at once: what your tests detect, and whether the
harness itself works. A broken isolation environment kills every case *before
the rules run*, which prints as a perfect score and measures nothing.

1. **Run a positive control before citing any score.** Feed the harness an input
   whose answer you already know in the *pass* direction. For a mutation-style
   harness that is a semantics-preserving change — reformat a line, rename a
   local, capitalize a docstring — and it **must be reported as surviving**. If
   the harness reports it as caught, stop: nothing was measured. Report that,
   not the score.
2. **Separate "the test failed" from "the test errored."** Established tools do:
   Stryker gives runtime errors and compile errors their own mutant states and
   excludes them from the mutation score, counting only genuine test failures as
   kills. A homegrown harness that shells out and reads a non-zero exit code
   cannot tell a detected defect from a crashed environment — classify the run
   (collection error, no tests ran, non-zero exit without a failing assertion)
   before counting it.
3. **Require a green baseline inside the harness's own environment.** Run the
   unmodified suite in the isolated tree first and require it to pass — the
   equivalent of Stryker's initial test run, which `dryRunOnly` exists to
   exercise on its own. Comparing mutated runs against a baseline you never
   observed compares nothing.
4. **Build the isolated tree from what the tests resolve at runtime**, not from
   what you think they need: the repo root they locate themselves against,
   fixture and example data, config files, installed dependencies. Directive 3
   is what proves you got this right.
5. **Read uniformity as a harness signal.** 100% caught, 0% caught, or every
   case producing byte-identical output means the same thing until proven
   otherwise: suspect the harness, run directive 1, and only then record the
   number.

Field context: a harness that copied only the implementation directory into its
temp tree — while the tests located their example data relative to the real
repo — killed all 105 tests with a file-not-found error, so every mutant was
"caught" and the reported 36/36 was a detection rate of zero. A no-op control
(capitalizing a docstring) reproduced the red run and exposed it; after the fix
the score moved to 34/36, and the two survivors were real rules no test asserted.

## Edge cases

| Case | Then |
|------|------|
| The no-op control is reported as caught | Fix the harness (missing data, deps, or paths in the isolated tree), then re-measure from scratch — every earlier number from that harness is void, including ones already published |
| A mutant survives that you expected to be caught | That is the finding, not a defect: a behavior no test asserts. Add the assertion, then re-run — [testing-quality-tests-that-cannot-fail] |
| Some mutants fail to compile or crash the runner | Report them as their own category and exclude them from the score; a mutant the runner cannot execute proves nothing about the tests |
| No semantics-preserving change is available for this input type | Use the baseline instead: run the unmodified input through the full harness path and require the pass verdict (directive 3) |
| The score is the evidence in a PR, report, or README | Publish the control run beside it; a score without its control is an unverified claim |
| Tests pass alone but fail inside the isolated tree | Isolation and shared-state rules — [testing-data-test-data-and-isolation] |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Cite "N/N caught" as proof the suite is strong | Cite it together with a control run showing the harness can also report *not* caught | A harness that can only produce one verdict has no information content, and the perfect score is its failure mode |
| Treat a non-zero exit code from the test command as "mutant killed" | Classify failure versus error, and count only assertion failures as kills | An import error, a missing fixture, and a detected bug all exit non-zero; only one of them is a test doing its job |
| Re-run the same harness to confirm a suspicious result | Feed it a known-answer input in each direction | Repeating a broken measurement reproduces the same wrong number with more confidence |
| Copy only the code under test into the harness's temp tree | Copy the whole working tree, then prove it with a green baseline run | Tests resolve fixtures, config, and data paths at runtime; the copy that "obviously suffices" is what silently breaks them |
| Add a negative control to each individual check and call the harness validated | Also control the harness as a whole | Per-check controls ask "can this check go red?"; only a harness-level control asks "can this harness go green?" |

## Sources

- https://stryker-mutator.io/docs/mutation-testing-elements/mutant-states-and-metrics/ — "Killed": at least one test failed; "Survived": all tests passed, "You're missing a test for it"; "Runtime error": the run errored rather than a test failing, and "is not represented in your mutation score"; compile errors likewise excluded
- https://stryker-mutator.io/docs/stryker-js/configuration/ — `dryRunOnly`: "Execute the initial test run only without doing actual mutation testing", the baseline run that validates the setup before mutants are introduced
- https://stryker-mutator.io/docs/ — mutants are inserted and the suite re-run per mutant; failing tests kill a mutant, passing tests let it survive
- https://testing.googleblog.com/2021/04/mutation-testing.html — inserting faults and requiring test failure is what measures detection; coverage does not
