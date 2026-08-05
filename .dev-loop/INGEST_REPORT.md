# Knowledge flush — 3 insight(s)

All three came out of the same week's estimation-engine work and share one shape:
**a check that runs, passes, and cannot see the thing it was written to see.** They
route to `testing/quality` as one amendment each to the two pages that own
assertion design, plus one new page for the artifact-baseline case.

## Verified best-practice

### 1. Value-preserving refactor (literal → config constant) — assert the dependency, not the value
**Claim:** when a refactor replaces a magic literal with a constant read from
config/SSOT, first compute whether the rendered output is byte-identical to the
removed literal. If it is, an assertion on that output cannot fail when the literal
is pasted back; substitute a sentinel into the constant (scoped patch) and assert the
output follows it.

**Sources checked**
- <https://martinfowler.com/bliki/DefinitionOfRefactoring.html> — refactoring is "a
  change made to the internal structure of software … without changing its observable
  behavior". Fetched; quoted verbatim. This is the mechanism: the assertion observes
  exactly the thing the refactor is defined not to change.
- <https://pitest.org/quickstart/mutators/> — the Inline Constant mutator "mutates
  inline constants… a literal value assigned to a non-final variable", replacing `1`
  with `0` or otherwise incrementing. Fetched; confirms the sentinel substitution is
  the hand-run form of an established mutation operator, i.e. the directive is not a
  local invention.
- <https://docs.pytest.org/en/stable/how-to/monkeypatch.html> — `setattr`/`setitem`,
  "All modifications will be undone after the requesting test function or fixture has
  finished", and "Prefer patching the reference that your code uses instead of
  patching the original object". Fetched. This **upgraded** the queued directive: the
  candidate said manual `try/finally`; the runner's scoped patch is leak-safe, and the
  namespace warning became the `from x import CONST` edge case.
- Field evidence (session): `"DB ×%s" % E.DB_MULT` with `DB_MULT = 1.3` renders
  exactly the removed `DB ×1.3` literal — computed, not assumed; the literal version
  went red only under sentinel substitution.

**Confidence: verified** (official docs for both the mechanism and the technique).

### 2. Composite return values — the oracle must read the field the fault reaches
**Claim:** when a function returns a dict/tuple/record, enumerate the returned fields
against the assertions; a field no assertion reads is unguarded regardless of
assertion count. Confirm by mutation, then assert the cross-field invariants over an
exhaustive grid.

**Sources checked**
- <https://arxiv.org/html/2411.09846v1> — RIPR: a fault must "be executed
  (Reachability), infect the program states (Infection), propagate the infection
  (Propagation), and have appropriate test oracles to reveal the fault
  (Revealability)"; the developer must "create a test assertion to detect an infected
  state that Propagated back to the test"; where the oracle is insufficient,
  "assertion amplification may help reveal the fault". Fetched; verbatim. This is the
  named mechanism the queued candidate lacked, and it supplies the remedy's term.
- <https://doi.org/10.1145/1543134.1411292> (Runciman, Naylor & Lindblad, Haskell '08,
  SIGPLAN Notices 44(2):37–48) — "instead of using a sample of randomly generated
  values they test properties for all values up to some limiting depth". Verified via
  the York institutional record (abstract + full citation + DOI), which is why the
  citation is by DOI rather than a paywalled PDF link.
- <https://hypothesis.works/articles/what-is-property-based-testing/> — fetched;
  supports the invariant-over-many-inputs framing and the "grid too large" edge case.
- Attempted and **not** cited: Li & Offutt's oracle-strategy paper
  (`albany.edu/faculty/offutt/research/papers/testOracle.pdf`) — the PDF downloaded but
  could not be rendered locally (no poppler), so nothing from it is quoted. The RIPR
  wording above comes from a source actually read.
- Field evidence (session): 58 assertions all reading `sp`; four mutations to the
  `lo`/`hi` percentile selection survived, a no-op control confirmed the harness
  discriminated, and an exhaustive grid found 13 combinations violating `lo ≤ sp ≤ hi`.

**Confidence: verified.**

### 3. A previously published artifact as the "before" baseline
**Claim:** date the artifact (provenance stamp, or schema fields the old code could
not emit) before using it; compare row-by-row on a stable key rather than trusting an
aggregate match; when it predates the change, rebuild `before` by reverting only the
change under test and re-running.

**Sources checked**
- <https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html> — validation
  "compares each row in the source with its corresponding row at the target, verifies
  the rows contain the same data, and reports any mismatches", "requires that the table
  has a primary key or unique index", and records failures per row key with "all
  source/target column values which do not match for the given key". Fetched. Supplies
  the keyed row-level comparison shape, including why a stable key is a precondition.
- <https://slsa.dev/spec/v1.0/provenance> — provenance exists to "describe how an
  artifact or set of artifacts was produced so that consumers of the provenance can
  verify that the artifact was built according to expectations"; `resolvedDependencies`
  records the source repo and resolved commit. Fetched. Supports "stamp what you emit"
  as the general fix rather than a local convention.
- <https://michaelfeathers.silvrback.com/characterization-testing> — "The purpose of
  characterization testing is to document your system's actual behavior, not check for
  the behavior you wish your system had"; the expected value comes from running the
  code. Fetched. Names the re-run baseline (step 4) as an established practice.
- Field evidence (session): `estimated.json` matched the current run's counted total
  (211.48) but one row differed in both category and value; a `dead` field null for
  every row dated the artifact to before a regex-lookbehind fix; the differing issue
  was a roll-up parent excluded from the total, which is why the aggregate agreed.

**Confidence: verified** for the comparison and provenance mechanics. The
aggregate-masking claim rests on the reproduction plus the DMS keyed-comparison
design, and the page states it as the literal claim an aggregate match supports rather
than as a general theorem.

## Existing-layer check

**Pages read in full:** `wiki/testing/index.md`, `quality/tests-that-cannot-fail.md`,
`quality/checks-that-cannot-pass.md`, `quality/harness-reverse-controls.md`,
`quality/minimum-case-set.md`; plus a directory survey of `wiki/qa/` and
`wiki/debugging/` and a repo-wide grep for `baseline|snapshot|before/after`.

**Also checked the 12 open `dev-loop:knowledge` PRs**, since unmerged pages are
invisible to `main` but not to the reviewer:

| Overlap candidate | Finding |
|---|---|
| `quality/tests-that-cannot-fail.md` — amended by open PRs #23, #28, #29 | **Textual conflict expected, not semantic.** #28 adds edge cases about *applying* a mutation (sed patterns, guard direction, interpreter mismatch); #23 and #29 only extend `related:`. The rows added here (value-preserving refactor, sentinel scoping) are disjoint from all three. All four touch the frontmatter `related:`/`sources:` block, so a conflict there is likely — resolve by keeping the union of ids and source URLs |
| `quality/differential-run-agreement.md` (open PR #22, not on `main`) | Adjacent, **not** a duplicate: that page compares two implementations of one spec on the same input; this one compares one implementation against its own stale output. No `related:` link added, because linking a page that does not exist on `main` would be a broken reference if #22 is rejected — worth adding if #22 merges first |
| `quality/harness-reverse-controls.md` (open PR #24) | Covers the no-op control that made insight 2's mutation run trustworthy. Linked one-way (new page → it) rather than editing it, to keep #24 conflict-free |
| `qa/process/regression-scope.md`, `debugging/methodology/verify-the-fix.md` (both in open PR #25) | Genuinely adjacent to the new page; linked one-way from the new page only, for the same reason |
| `quality/minimum-case-set.md` | No open PR touches it — safe to amend |

**Conflicts with existing directives:** none. Insight 1 extends the page's
break-the-code rule to a case where breaking the code is *undetectable by
construction*; insight 2 extends its step 2 ("assert an observable outcome") from one
outcome to the whole returned shape.

**Related links added:** `tests-that-cannot-fail` ↔ new baselines page (both are
comparisons that cannot show the change); `tests-that-cannot-fail` →
`minimum-case-set` (a surviving mutant may mean the oracle never reads the field);
`minimum-case-set` → `tests-that-cannot-fail` + `harness-reverse-controls`; new page →
`tests-that-cannot-fail`, `harness-reverse-controls`, `minimum-case-set`,
`qa-process-regression-scope`, `debugging-methodology-verify-the-fix` (all five ids
confirmed present on `main`).

## Routing decision

| Insight | Target | Action | Why |
|---|---|---|---|
| 1 — value-preserving refactor test | `testing/quality/tests-that-cannot-fail.md` | **Merge** — 1 never-fails row, 4 edge cases, 1 Instead-of row, 3 sources, `last_verified` → 2026-08-05 | The page's whole subject is "an assertion that cannot detect a defect". This is one more instance, not a new situation — a separate page would split the never-fails table across two files |
| 2 — composite return values | `testing/quality/minimum-case-set.md` | **Merge** — new step 6 with four sub-steps, 3 edge cases, 2 Instead-of rows, 3 sources | The question is *which assertions* a result needs, which is this page's remit (step 2 already says "assert an observable outcome"). Placed here rather than in `tests-that-cannot-fail` because the deliverable is an assertion-selection rule; the detection half is cross-linked |
| 3 — published artifact as baseline | **New page** `testing/quality/baselines-from-published-artifacts.md` (75 body lines) | Create | No existing page covers validating a *baseline*. `tests-that-cannot-fail` is about assertions inside a suite, `harness-reverse-controls` about a harness's own score, `differential-run-agreement` (unmerged) about two implementations. Folding it into any of those would break the one-case-per-page rule |

**New category:** none. `testing/quality` already holds the "can this check see
anything" family (`tests-that-cannot-fail`, `checks-that-cannot-pass`,
`spec-artifact-checks`, `harness-reverse-controls`), and the new page asks the same
question of a comparison baseline.

**Plumbing:** `wiki/testing/index.md` — new "load when" row plus the domain
"route here for" line extended with baseline judgement; `log.md` — one `ingest` entry
naming the new page, both amendments, and the verified sources.

**Format checks run:** body lengths 75 / 89 / 67 lines (limit 120); banned vague
qualifiers (`usually`, `generally`, `consider`, `might want`, `as appropriate`,
`typically`, `probably`) grep-clean across all three files; every anti-pattern sits in
an `Instead of` row paired with its replacement; all five `related:` ids resolved
against `main`.
