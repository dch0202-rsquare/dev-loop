# Knowledge flush — 4 insights

4 queued candidates → **3 pages** (2 new categories). Candidates 1 and 3 describe
the two failure directions of one case (a doc-spec gate whose verdict does not mean
what it looks like), so they merged into a single page per `AGENTS.md` rule 1
"one case per page" rather than becoming two thin pages.

| # | Candidate (trigger) | Page | Confidence |
|---|---------------------|------|------------|
| 1 | Adopting a grep/regex gate for a document that does not exist yet | `testing/docs-as-spec/gate-falsifiability` | field-tested |
| 3 | A check asserts a mapping table covers every item of a source list | (same page — green-side of the same case) | field-tested |
| 4 | Parsing Markdown table rows programmatically in a doc-as-spec repo | `testing/docs-as-spec/markdown-table-parsing` | **verified** |
| 2 | Persisting the result of `s3.upload()` to a durable store | `backend/common/storage/object-key-persistence` | **verified** |

## Verified best-practice

### 2 — Persist `Key`, not `Location` (verified)

**Claim.** For a managed S3 upload, `Location`'s provenance switches at the
multipart threshold and the two sources encode differently, so the stored
reference must be `Bucket` + `Key`.

Sources checked, and what each established:

- **Reproducible source check, aws-sdk-js 2.829.0** (local `node_modules/aws-sdk`):
  `lib/s3/managed_upload.js` — `minPartSize: 1024 * 1024 * 5`; `finishSinglePart`
  composes `data.Location` from `endpoint.protocol + '//' + endpoint.host +
  httpReq.path` **and** sets `data.Key = request.params.Key`; `finishMultiPart`
  applies `data.Location.replace(/%2F/g, '/')` to S3's value — it undoes only
  `%2F`, so any other form-encoded character survives. `lib/util.js` `uriEscape`
  wraps `encodeURIComponent` (space → `%20`) for the request path.
- **`apis/s3-2006-03-01.min.json`** — `Key` is present in the
  `CompleteMultipartUpload` output shape, so `Key` is available on both paths.
  Cross-checked against
  <https://docs.aws.amazon.com/AmazonS3/latest/API/API_CompleteMultipartUpload.html>.
- **<https://github.com/aws/aws-sdk-js/issues/1158>** — upstream confirmation of
  the divergence (multipart `Location` arrives URI-encoded from S3; single-part is
  SDK-composed). PR #1420 patched `%2F` only, matching the source read above.
- **<https://github.com/aws/aws-sdk-js-v3/issues/5656>** — the same divergence
  still exists in `@aws-sdk/lib-storage` (v3): `__uploadUsingPut` composes
  `Location`, the `CompleteMultipartUploadCommand` path does not. So the directive
  is not a v2-only workaround.
- **<https://github.com/aws/aws-sdk-php/issues/2933>** — the same divergence in
  aws-sdk-php (`ObjectURL` from the request's effective URI below the threshold,
  from the server response above it), reported as inconsistent encoding of spaces
  and plus signs. Two independent SDKs ⇒ an S3-API-level property, not a JS quirk.

**Scoped honestly:** the mechanism (two provenances, switch at `partSize`) and
`Key`'s availability on both paths are documented and reproducible. The specific
"space becomes `+`" symptom is attributed in the page to a production regression
(files >5 MB 404'd on download while smaller ones worked), not to a doc citation.

### 4 — Split table rows on unescaped pipes (verified)

- **<https://github.github.com/gfm/#tables-extension->** — spec rule: "Include a
  pipe in a cell's content by escaping it, including inside other inline spans."
- **Reproducible render check** through GitHub's own renderer (`POST /markdown`,
  `mode: gfm`, <https://docs.github.com/en/rest/markdown/markdown>):
  the row `| EventSource | \`create\|update\|delete\` |` renders as **2 cells**,
  cell 2 holding the literal `create|update|delete`. A naive `split("|")` reports
  **4**; splitting on `(?<!\\)\|` reports **2** — so the "broken table" was the
  checker's defect, not the document's.
- **Additional finding this flush:** the same row with *unescaped* pipes also
  renders as 2 cells — `update` and `delete` are **silently dropped** past the
  header's column count. Added as an edge case (escaping is required even inside
  code spans).
- **Limitation found rather than copied:** the queued candidate proposed
  `(?<!\\)\|` verbatim. Tested, it mis-reads a real delimiter that follows an
  escaped backslash (`\\|` → 1 cell instead of 2). The page carries this as an
  edge case instead of presenting the regex as complete.

### 1 + 3 — Gate falsifiability (field-tested; governing principle sourced)

The principle is sourced; the doc-gate mechanics are field experience, so the page
stays `field-tested` rather than being upgraded to `verified`:

- **<https://www.industriallogic.com/blog/tdd-youre-doing-it-wrong/>** — "If you
  don't start with a test that *fails for the right reason*, you are not doing
  TDD"; the failure must come from the missing behavior. A gate that is red only
  because its target file is absent is exactly a red for the wrong reason.
- **<https://testing.googleblog.com/2021/04/mutation-testing.html>** — inserting a
  fault and requiring failure is what measures detection; grounds the
  one-negative-control-per-check rule.
- **<https://martinfowler.com/bliki/TestCoverage.html>** — coverage locates
  unchecked material, it does not certify correctness; the table-row analogue of
  "a covered line is only an executed line".
- **Field evidence** (described in the page): an RFC-suite doc-verification harness
  (51-rule grammar→IR mapping table, 33 golden rows). Corrupting one mapping cell
  to a nonexistent kind, and swapping a golden node id for a nonexistent one, both
  left the row-coverage checks **green**; the added value-validity checks caught
  both.

## Existing-layer check

Pages read in full: `wiki/testing/index.md`,
`wiki/testing/quality/tests-that-cannot-fail.md`, `wiki/qa/index.md`,
`wiki/backend/index.md`, `wiki/backend/node/index.md`,
`wiki/debugging/methodology/verify-the-fix.md` (plus `INDEX.md`, `AGENTS.md`,
`templates/page.md`, `skills/wiki-ingest/SKILL.md`). Repo-wide greps for
`s3|presigned|object-storage|multipart` and
`negative control|mutation test|falsifi|both directions`.

| Finding | Resolution |
|---------|------------|
| `testing/quality/tests-that-cannot-fail` is the nearest neighbour — same family (prove a check can fail, manual mutation, "coverage counts execution, not detection") | **Not a duplicate.** Its subject is tests over *code* and its method is mutating the code under test; the new page's subject is a gate over a *document*, and its failure modes (absence-driven red, name-only coverage) have no row there. Linked **both ways** — added `testing-docs-as-spec-gate-falsifiability` to that page's `related:` |
| `debugging/methodology/verify-the-fix` carries "flip both directions" (repro must fail pre-fix, pass post-fix) | Same discipline, different trigger (closing a bug vs. adopting a gate). Cited one-way from the new page; no reverse link, to keep that page's routing on bug-fix closure. No conflict |
| `qa/` — the harvester's `domain: qa` hint for candidate 4 | **Rejected as a route.** `qa/index.md` scopes to release-quality *process* (gates, triage, regression scope, exploratory testing). Writing a checker is automated-check code ⇒ `testing/` |
| Object storage / S3 / multipart anywhere in the wiki | **No existing page** (only an unrelated string hit in `qa/process/severity-and-priority.md`). New page; no merge target |
| Conflicting directives | **None found.** Nothing in the wiki says to store `Location`, and nothing claims a coverage check suffices |

Related-links added: new gate page ↔ `testing-quality-tests-that-cannot-fail`
(bidirectional); gate page → `debugging-methodology-verify-the-fix`; the two new
`docs-as-spec` pages ↔ each other; storage page →
`backend-common-jobs-idempotent-handlers`,
`backend-node-boundaries-runtime-validation`. Every `related:` id verified to
resolve.

## Routing decision

| Insight | Target page | New category? |
|---------|-------------|---------------|
| 1 + 3 | `wiki/testing/docs-as-spec/gate-falsifiability.md` | **yes — `testing/docs-as-spec`** |
| 4 | `wiki/testing/docs-as-spec/markdown-table-parsing.md` | (same new category) |
| 2 | `wiki/backend/common/storage/object-key-persistence.md` | **yes — `backend/common/storage`** |

**Why `testing/docs-as-spec` rather than an existing category.** Every existing
`testing/` category takes *code under test* as its artifact: `strategy`
(unit/integration/e2e level), `quality` (cases, assertions, mutation), `data`,
`mocking`, `flaky`, `async`, `e2e`. These two pages take a **document** as the
artifact, and their mechanics are specific to it — grep/regex gates, mapping-table
coverage, cross-document signature checks, GFM cell parsing. Filing them under
`quality` would put document-parsing rules in the category an agent loads for
test-case selection, and would stretch the closest page (`tests-that-cannot-fail`)
past its one-case boundary. Two pages justify the category, and `docs-as-spec`
names the situation rather than a technology, per the naming rule. Placed after
`e2e` in the domain index.

**Why `backend/common/storage` rather than `backend/node`.** The harvested evidence
was aws-sdk-js v2, which pointed at the node subtree — but research showed the same
divergence in `@aws-sdk/lib-storage` (v3) *and* aws-sdk-php, making it a property of
the S3 API rather than a Node quirk. Per `wiki/backend/index.md` ("route by concern
first, stack second"; common owns the principle), it belongs in `common/`. No
existing `common/` category (`api-design`, `reliability`, `caching`, `jobs`,
`errors`, `auth`, `orm`, `concurrency`) covers object-storage references.
`applies_to: [aws-s3]` keeps the scope honest.

**Plumbing updated:** `wiki/testing/index.md` (+ domain header line),
`wiki/backend/index.md` (+ common-subtree route line), `INDEX.md` (testing and
backend route lines), `log.md` (dated ingest entry).

**Verification run.** All three pages satisfy the `AGENTS.md` invariants: body
lines 60 / 56 / 62 (≤120), required sections present, every `related:` id resolves,
no banned vague qualifiers, each page listed in its domain index with a "load when"
line. The repo's `bats` suite covers harness shell scripts and `bats` is not
installed on this machine; this change is docs-only (`wiki/`, `INDEX.md`,
`log.md`), so no script behavior is touched.
