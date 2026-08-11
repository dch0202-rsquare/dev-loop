# Knowledge flush — 4 candidate(s): 3 ingested, 1 dropped as pending-duplicate

Queue drained: `~/.dev-loop/queue/231f63bc-….jsonl` (2 rows), `~/.dev-loop/queue/498e892c-….jsonl` (2 rows).

## Verified best-practice

### 1. Assert a signed link by verifying it, not by matching its shape → `verified`

**Claim.** When a test covers code that assembles a URL carrying a signature or
token, asserting the token's *presence/shape* cannot detect a link signed with
the wrong key, for the wrong subject, or under the wrong purpose; only passing
the token to the real verifier — plus shape-preserving negative arms — can.

**Sources checked (all opened this session):**

| Source | What it establishes |
|--------|---------------------|
| https://docs.djangoproject.com/en/stable/topics/signing/ | Verification is an explicit call: "If the signature or value have been altered in any way, a `django.core.signing.BadSignature` exception will be raised". Salt namespacing gives the wrong-purpose arm a primary source: "A signature that comes from one namespace (a particular salt value) cannot be used to validate the same plaintext string in a different namespace that is using a different salt setting" |
| https://www.rfc-editor.org/rfc/rfc7519 | The wrong-subject arm: "If the principal processing the claim does not identify itself with a value in the 'aud' claim when this claim is present, then the JWT MUST be rejected". The expiry arm: `exp` "identifies the expiration time on or after which the JWT MUST NOT be accepted for processing" |
| https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html | The third-party-receiver arm: `SignatureDoesNotMatch` troubleshooting instructs verifying "that all request parameters—including the HTTP method, headers, and query string—match exactly between URL generation and usage" — i.e. the link's correctness is only decidable at the receiver. Also the credential-bound-lifetime edge case |
| https://stryker-mutator.io/docs/mutation-testing-elements/supported-mutators/ | The hand-seeding requirement: the nearest generated operator is String Literal, "FilledStringToEmpty: `\"foo\"` (filled string) becomes `\"\"` (empty string)" — an empty key still yields a well-formed signature, so no generated mutant exercises this class |

**How verified.** Each URL fetched and the quoted sentences read in place; no
quote is carried over from another document. The queue row's own field evidence
(6 hand-seeded mutants on a Python link-assembly module: wrong-key and
wrong-subject killed only by the verifier assertion and three live-receiver
`200` assertions, no-op control survived) is recorded in the page as a dated
field reproduction, kept separate from the doc-verified claims.

**Confidence: `verified`** — the mechanism is stated by primary docs; the
testing directive is the direct consequence plus one reproduction.

### 2. Census cross-module consumers of new public symbols, then classify the zeros → `verified`

**Claim.** A symbol built by one parallel task and never wired by another passes
type-check, its own tests, and merge; the detector is a census of cross-module
production references. But a raw zero is not a defect — most zeros are
module-internal helpers, so the census needs a filter (a written consumer claim)
before anything is reported.

**Sources checked:**

| Source | What it establishes |
|--------|---------------------|
| https://github.com/import-js/eslint-plugin-import/blob/main/docs/rules/no-unused-modules.md | The criterion is cross-module, not any reference: the rule reports "individual exports not being statically `import`ed or `require`ed from other modules in the same project"; `unusedExports` finds "exports without any static usage within other modules" |
| https://github.com/jendrikseipp/vulture | The false-positive half: "Due to Python's dynamic nature, static code analyzers like Vulture are likely to miss some dead code. Also, code that is only called implicitly may be reported as unused"; functions/classes/variables are rated at 60% confidence — the tool itself treats a zero as a candidate, not a verdict |
| https://knip.dev/explanations/why-use-knip | Corroborates the false-positive load in real projects: "In large and/or legacy projects, Knip may report false positives and require some configuration" |

**How verified.** Fetched each; the eslint-plugin-import rule doc is the exact
statement of the cross-module criterion the directive rests on. Field
measurement from the queue row (14 zero-consumer public functions, exactly 1
real gap once filtered by documented-consumer claim) is recorded as dated field
evidence, not as a doc claim.

**Confidence: `verified`** — criterion and false-positive rate are documented by
the tools that implement it; the filter step is grounded in vulture's own
confidence model and one measurement.

### 3. Sweep the review's defect class over your own fix diff → `verified`

**Claim.** A review names the instances present in the tree it read. The fix
diff adds code that tree never contained, so a fix can close the class in one
function and reopen it in a sibling added by the same commit. The remedy is to
turn the reviewer's wording into a class query and run it over *both* the repo
and the diff's added lines, proving the query discriminates against the pre-fix
site first.

**Sources checked:**

| Source | What it establishes |
|--------|---------------------|
| https://codeql.github.com/docs/codeql-overview/about-codeql/ | The seed-to-class technique this page applies to review feedback: "Variant analysis is the process of using a known security vulnerability as a seed to find similar problems in your code"; such a query lets you "identify all of the places where that vulnerability exists" |
| https://www.eecg.utoronto.ca/~yuan/papers/incorrect_fix_abstract.html | The risk weighting on the fix diff: "at least 14.8% to 24.4% of sampled fixes for post-release bugs in these large OSes are incorrect"; "27% of the incorrect fixes are made by developers who have never touched the source code files associated with the fix" |
| https://dl.acm.org/doi/10.1145/2025113.2025121 | The paper those figures come from — Yin, Yuan, Zhou, Pasupathy, Bairavasundaram, "How do fixes become bugs?", ESEC/FSE 2011 |

**How verified.** CodeQL doc and the authors' abstract page fetched and quoted
verbatim; the ACM DOI is the paper record for the same figures. One source I
tried and did **not** use: the ISTQB Foundation syllabus PDF (defect clustering)
— `WebFetch` returned it as binary and local PDF rendering is unavailable here,
so rather than quote it from a secondary blog I dropped it. Nothing in the page
depends on it.

**Confidence: `verified`** — the technique and the incorrect-fix rates are both
primary-sourced. The synthesis (apply variant analysis to your own fix diff,
scoped to added lines) is stated as a directive with the field incident that
produced it attached and dated.

### 4. `.pyc` cache invalidation in a Python mutation harness → not ingested (see Open-PR check)

Researched anyway before deciding: the mechanism is already documented at
https://docs.python.org/3/reference/import.html (timestamp **and size** metadata
validation) and https://peps.python.org/pep-0552/ (hash-based invalidation) —
both already cited by the existing page. Confidence would have been `verified`;
the reason it is not ingested is duplication, not weak evidence.

## Existing-layer check

Routed via `INDEX.md` → read `wiki/testing/index.md`, `wiki/qa/index.md`,
`wiki/infrastructure/index.md`, `wiki/backend/index.md` route rows, then opened
every page whose "load when" line overlapped a candidate.

Pages read: backend-python-language-bytecode-cache-staleness, backend-common-change-impact-call-site-enumeration, testing-quality-tests-that-cannot-fail, testing-quality-write-path-assertions, testing-quality-harness-reverse-controls, qa-process-regression-scope, qa-process-scope-purity-checks, debugging-methodology-verify-the-fix

| Candidate | Overlap found | Resolution |
|-----------|---------------|------------|
| 1 — signed links | No page covers testing link-minting code. `testing-quality-write-path-assertions` is the closest shape (assert past the status code) but its trigger is a persisting HTTP endpoint. `backend-common-auth-jwt-server-side` covers *implementing* verification, not testing a link builder | **Created new** `testing/quality/signed-link-assertions.md`; `related:` to write-path-assertions, tests-that-cannot-fail, harness-reverse-controls, jwt-server-side, secrets-in-code |
| 2 — unconsumed symbols | `backend-common-change-impact-call-site-enumeration` enumerates callers of a callee whose **contract you are changing** — a different trigger (change-time scoping vs completion-time gap detection) and a different failure (silent rebinding vs never-wired). `qa-process-scope-purity-checks` is the same *shape* (a completion gate built by filtering a programmatic listing) and is the natural sibling | **Created new** `qa/process/new-symbols-without-a-consumer.md`, with an explicit "contract changes → change-impact" pointer in both its trigger and the index line so the two do not compete |
| 3 — defect-class sweep | `debugging-methodology-verify-the-fix` has a "Scope check" row — "List the other inputs/paths the confirmed mechanism implies (same root cause elsewhere)" — which covers the *pre-existing* arm only. The arm this candidate adds (the class reopening inside the fix's own added lines) is absent. `qa-process-regression-scope` chooses which **tests** to re-run, not which static class to re-query | **Created new** `qa/process/defect-class-sweep-over-a-fix.md`, cross-pointing to both from its trigger section so the boundary is explicit |
| 4 — `.pyc` cache | `backend-python-language-bytecode-cache-staleness` already carries the mechanism, the equal-size mutation case ("Mutations designed to preserve byte length … are exactly the ones the timestamp+size check cannot see"), the `PYTHONDONTWRITEBYTECODE` row, and the no-op-control requirement | **Dropped**, not merged — see Open-PR check |

**Conflicts flagged:** none. No candidate contradicts an existing directive.

**Reciprocal `related:` links added** (chosen among adjacent pages that no open
PR is currently editing, to keep this PR's conflict surface small):
`testing/quality/write-path-assertions.md`, `qa/process/scope-purity-checks.md`,
`debugging/methodology/verify-the-fix.md`, `backend/common/auth/jwt-server-side.md`.
The strongly adjacent `testing-quality-tests-that-cannot-fail` and
`backend-common-change-impact-call-site-enumeration` are edited by 4 and 4 open
PRs respectively, so the new pages link *to* them one-way rather than adding a
fifth concurrent edit.

## Open-PR check

Listed 18 open `knowledge/*` heads via
`gh pr list --repo choiyounggi/dev-loop --state open --search "head:knowledge/"`.
Five of them (#74, #73, #72, #52, #49) are not fetchable from `origin` —
`fatal: couldn't find remote ref …` — so their contents were read with
`gh pr diff <n>` instead of `git diff origin/main origin/<head>`. Treating those
five as "no overlap" because the fetch failed would have missed the one real
duplicate below.

| Open PR | wiki/ paths it touches (overlapping only) | Overlaps which candidate | Verdict |
|---------|-------------------------------------------|--------------------------|---------|
| #52 | `backend/python/language/bytecode-cache-staleness.md`, `testing/quality/source-text-wiring-assertions.md`, `testing/quality/harness-reverse-controls.md` | **4** (same page, same rows — #52 adds a `shutil.copy2` mtime-restore row to the exact edge-case table candidate 4 would extend). Also inspected for **2**: `source-text-wiring-assertions` asserts *within one file* that a wiring call's text is present; candidate 2 counts *cross-module* consumers of a new symbol — different trigger, different subject | **drop** for 4; **new** for 2 |
| #49 | `testing/quality/unasserted-return-fields.md` + 6 other testing/quality pages | **1**, checked: it covers composite return fields no assertion reads. A signed link is one string, and the defect is the assertion's *kind* (shape vs verification), not an unread field | **new** |
| #61 | `testing/data/harness-vs-run-path-fixtures.md`, `testing/quality/harness-reverse-controls.md` | none of the three | **new** |
| #47, #73 | `testing/quality/tests-that-cannot-fail.md`, `testing/quality/guard-shape-vs-consequence.md`, mocking/db pages | none — this PR does not edit those files | **new** |
| #58, #68, #51, #50 | `backend/common/change-impact/call-site-enumeration.md`, `qa/process/regression-scope.md`, `qa/index.md` | adjacency to **2** and **3**, but no page with either trigger | **new**; this PR does not edit call-site-enumeration or regression-scope |
| #74, #72, #69, #66, #64, #62, #57, #56, #55 | databases / infrastructure / platforms / qa-deliverables / backend-ml paths | none | **new** |

**Candidate 4 — dropped as pending-duplicate, with its delta recorded here.**
The one thing the merged page and #52 do not say: a harness that re-imports via
`importlib.util.spec_from_file_location` builds a *fresh module object* and so
bypasses `sys.modules` staleness — which reads as immunity — but it still goes
through the `.pyc` cache, so a byte-length-preserving mutant is re-measured from
the previous mutant's bytecode. Adding that row here would collide directly with
#52's pending edit to the same table. **Recommendation for the owner:** fold one
edge-case row into `backend-python-language-bytecode-cache-staleness` when #52
merges, phrased against the existing "stale module was already imported in a
long-lived process" row. No sibling duplicate PR was opened.

## Routing decision

| # | Insight | Domain / category | Page | New category? |
|---|---------|-------------------|------|---------------|
| 1 | Assert a signed link by verifying it | `testing` / `quality` | **new** `wiki/testing/quality/signed-link-assertions.md` | No — `testing/quality` already holds the "what does this assertion actually prove" family (tests-that-cannot-fail, write-path-assertions, unasserted return fields) |
| 2 | Census cross-module consumers of new public symbols | `qa` / `process` | **new** `wiki/qa/process/new-symbols-without-a-consumer.md` | No — this is a completion gate over a diff, the same category as `scope-purity-checks`. Considered and rejected: `backend/common/change-impact` (that category is change-*time* scoping of an existing callee, and the page would compete with call-site-enumeration's trigger) and `infrastructure/agent-orchestration` (the insight is not tied to any orchestration substrate — it applies to any parallel split, including several humans on several PRs) |
| 3 | Sweep the review's defect class over your own fix diff | `qa` / `process` | **new** `wiki/qa/process/defect-class-sweep-over-a-fix.md` | No. Considered and rejected: merging a row into `debugging/methodology/verify-the-fix` — its trigger is "you believe a bug is fixed and are about to close it", while this one fires on *any* review-response diff including pure-refactor feedback, and the directive needs its own two-arm table |
| 4 | `.pyc` cache in a mutation harness | — | none (dropped) | — |

Plumbing updated: `INDEX.md` (testing + qa route lines), `wiki/testing/index.md`
(quality table), `wiki/qa/index.md` (route line + process table, 2 rows),
`log.md` (one dated `ingest` entry covering all 4 candidates including the drop).

Format checks run before commit: body length 78 / 71 / 73 lines (limit 120);
zero banned vague qualifiers in all three pages; every `related:` id and every
relative index link resolves against `wiki/` in this checkout.
