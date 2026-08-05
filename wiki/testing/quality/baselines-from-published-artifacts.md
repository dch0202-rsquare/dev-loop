---
id: testing-quality-baselines-from-published-artifacts
domain: testing
category: quality
applies_to: [general]
confidence: verified
sources:
  - https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html
  - https://slsa.dev/spec/v1.0/provenance
  - https://michaelfeathers.silvrback.com/characterization-testing
last_verified: 2026-08-05
related: [testing-quality-tests-that-cannot-fail, testing-quality-harness-reverse-controls, testing-quality-minimum-case-set, qa-process-regression-scope, debugging-methodology-verify-the-fix]
---

# Using a Previously Published Artifact as the "Before" Baseline

## When this applies

You are measuring what a code change does to an output — an export, report,
estimate file, CSV/JSON snapshot — and you plan to use a file produced by an
earlier run as the "before" side. Also when a headline aggregate (a total, a
row count, a score) matches between that file and the current run and you are
about to read that match as "no other differences".

## Do this

1. **Date the artifact before comparing against it.** Determine which revision
   produced it, and require an answer from the artifact itself, not from its
   mtime or filename:

| Signal available | How to date it |
|------------------|----------------|
| The artifact records its producing revision (provenance/version stamp) | Read it, and diff that revision against the code you are measuring |
| No stamp, but the current code emits fields the old code did not | Check those fields: a column absent, or present-but-null for every row, dates the artifact to before that field existed |
| No stamp and no schema change | Re-run the producing command at the artifact's suspected revision and require a byte-identical file before trusting it as a baseline |

2. **Compare row by row on a stable key, not by aggregate.** Align both sides on
   the identity column, then report per-key differences with the fields that
   differ — the shape AWS DMS validation uses (it "compares each row in the
   source with its corresponding row at the target", requires a primary key or
   unique index, and records each failure as a key plus the columns that
   mismatch). Run the aggregate check as well, and treat agreement there as one
   assertion among many rather than a summary of them.
3. **Read an aggregate match as its literal claim: the excluded and offsetting
   rows cancel.** Any rule that removes rows from the total (roll-up parents,
   cancelled or superseded records, de-duplication) lets a wrong row sit inside
   a right total, and two errors in opposite directions do the same.
4. **When the artifact turns out to predate the change, build the baseline by
   re-running the current code with only the change under test reverted** —
   in memory, in one process, over the same inputs. That is a characterization
   run: it "document[s] your system's actual behavior" at the revision you
   actually want to compare against, so every difference it shows belongs to
   your change.
5. **Report the old artifact's numbers in their own section, labelled with the
   revision that produced them.** Keep them out of the before/after table so a
   reader cannot add a historical figure to a change-impact figure.
6. **Stamp the artifacts you produce** with the producing revision and the
   input set, so the next comparison reads the stamp instead of doing
   archaeology. Provenance exists to let a consumer "verify that the artifact
   was built according to expectations", which is the same question a baseline
   comparison asks.

## Edge cases

| Case | Then |
|------|------|
| The artifact has no stable per-row key | Align on the natural key the producer sorts by, and verify the alignment by requiring equal row counts per key group before comparing values; when no alignment holds, use the re-run baseline from step 4 |
| Row-level and aggregate comparison both agree | Cite the row-level result — it subsumes the aggregate — and record the key set compared, so a later reader knows which rows were in scope |
| The published artifact is itself the contractual deliverable (it was sent to someone) | The diff against it is the finding, not a nuisance: report which delivered rows the change would alter, separately from the impact measurement |
| A field was renamed between the two generations | Map the old name to the new one explicitly in the comparison, and record the mapping — an unmapped rename reads as every row differing |
| The producer's output order is nondeterministic | Sort both sides by the key before diffing; an order-only diff is not a value difference |
| Inputs also changed between the two runs (new rows arrived) | Restrict the comparison to the key set present in both, and report the added and removed keys as their own counts |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Accept a matching total as evidence the two runs agree | Diff row by row on the key and report per-key differences | Exclusion and roll-up rules keep wrong rows out of the total, so the total can match while individual rows disagree |
| Use the newest artifact on disk as "before" because it is the newest | Date it by its schema fields or provenance stamp first | The newest file can still predate the change you are measuring; its mtime records when it was written, not which code wrote it |
| Attribute every difference from the old artifact to your change | Re-run the current code with only your change reverted, and diff against that | Differences from intervening revisions land in your impact number otherwise, inflating it silently |
| Fold the old artifact's headline figures into the before/after table | Put them in a separate, revision-labelled section | Two numbers in one table read as comparable; one is a historical output and the other is a measurement |

## Sources

- https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html — validation "compares each row in the source with its corresponding row at the target, verifies the rows contain the same data, and reports any mismatches"; it "requires that the table has a primary key or unique index"; failures are recorded per row key with "all source/target column values which do not match for the given key"
- https://slsa.dev/spec/v1.0/provenance — provenance exists to "describe how an artifact or set of artifacts was produced so that consumers of the provenance can verify that the artifact was built according to expectations"; `resolvedDependencies` records the source repository and the commit it resolved to
- https://michaelfeathers.silvrback.com/characterization-testing — "The purpose of characterization testing is to document your system's actual behavior, not check for the behavior you wish your system had"; the expected value comes from running the code and reading the actual result
- Field reproduction 2026-08-05 (Python estimation engine): a published `estimated.json` agreed with the current run on the counted total (211.48) but differed on one row, whose category and value both changed. The artifact predated a regex lookbehind fix; a `dead` field that was null for every row dated it. The differing issue was a roll-up parent excluded from the total, which is why the aggregate matched
