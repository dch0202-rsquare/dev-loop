# Knowledge flush — 4 insight(s) → 2 pages

Queue drained: 4 pending candidates from 2 sessions (`19484655…`, `dd51281c…`),
harvested 2026-07-30 while writing and auditing RFC-style spec documents.

## Verified best-practice

### A. Insights 1, 2, 4 — grep gates over spec documents (folded into one page)

| # | Claim | Verification | Confidence |
|---|-------|--------------|------------|
| 1 | A gate's red run against a not-yet-written file proves the file is absent, not that the pattern is correct → validate the pattern on a spec-satisfying **sibling** file first | POSIX grep exit status: `0` "one or more lines were selected", `1` "no lines were selected", `>1` "an error occurred" — a missing file returns 2, a wrong pattern returns 1, and both are non-zero. Reproduced locally: missing file → `exit=2`; no match → `exit=1` | verified |
| 2 | Keyword presence does not check the requirement; split gates by structure / modality / polarity / enum completeness / cross-reference | RFC 2119: MUST is "an absolute requirement of the specification"; SHOULD means "there may exist valid reasons in particular circumstances to ignore a particular item". RFC 8174: the meanings apply "only when they are in all capitals". So `필수`→`선택` and `MUST`→`SHOULD` preserve every searched token while destroying the requirement — the requirement level rides on the modal word, not the keyword | verified |
| 4 | Each gate needs a planted-defect copy that must FAIL (mutation self-test), and non-match paths must fail closed | Mutation testing (Google Testing Blog, Goran Petrovic, 2021-04-12) — inserting faults and requiring failure is what measures detection; the repo already cites this on `testing-quality-tests-that-cannot-fail`. Exit-code separation (1 vs >1) verified against POSIX grep as above | verified |

Also verified and folded in as an edge case: `grep -c` "writes only a count of
selected **lines**", so two hits on one line report 1 — an occurrence count needs
`grep -o | wc -l`.

Sources checked (all live-fetched this session):
- https://pubs.opengroup.org/onlinepubs/9699919799/utilities/grep.html
- https://www.rfc-editor.org/rfc/rfc2119
- https://www.rfc-editor.org/rfc/rfc8174
- https://testing.googleblog.com/2021/04/mutation-testing.html (page live, title/author/date confirmed; body not extractable through the fetcher — cited for the mutation-testing concept it is already cited for elsewhere in this repo)

The session evidence (an independent auditor's 8 tampered copies passed 16–17
gates; after adding the four axes, 32/32 tampered copies failed and the clean
document passed 62/62) is production evidence for the *shape* of the fix; the
directives themselves rest on the sources above.

### B. Insight 3 — matching Korean/non-ASCII text

| Claim | Verification | Confidence |
|-------|--------------|------------|
| A composed Hangul syllable is one code point, so a stem prefix is not a substring | Reproduced: `아닌` = U+C544 U+B2CC, `아니` = U+C544 U+B2C8; `"아니" in "아닌"` → `False`; `grep -c '대상이 아니'` → 0 while `grep -c '대상이 아닌'` → 1 on the same line | verified |
| The behaviour flips by normalization form — **a refinement the raw candidate did not state** | In NFD, `아닌` decomposes to U+110B U+1161 U+1102 U+1175 U+11AB and NFD(`아니`) *is* a prefix of it, so the same stem pattern matches under NFD and misses under NFC. Reproduced with `unicodedata.normalize` + grep on NFC and NFD fixtures | verified |
| Pattern and target must be in the same normalization form before matching | UAX #15: "when implementations keep strings in a normalized form, they can be assured that equivalent strings have a unique binary representation"; higher-level processes that compare strings "must respect canonical equivalence or problems will result"; Hangul composition/decomposition is algorithmic | verified |
| macOS filename forms | Apple APFS FAQ: "APFS preserves the normalization of the filename and uses hashes of the normalized form … to provide normalization insensitivity", vs HFS+ which "stores the normalized form … on disk". Lookups are therefore form-insensitive, but `readdir`/`ls` returns the stored form — so a grep over a listing can still miss | verified |

The candidate's own `evidence` line recorded an initial misdiagnosis ("BSD grep
ERE multibyte problem"). That misdiagnosis is **not** carried into the page — it
is inverted into an `Instead of` row (print code points before blaming the engine).

Sources: https://www.unicode.org/reports/tr15/ ,
https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/APFS_Guide/FAQ/FAQ.html ,
POSIX grep (`LC_CTYPE` multibyte interpretation).

## Existing-layer check

Pages read in full: `INDEX.md`, `AGENTS.md`, `templates/page.md`,
`wiki/testing/index.md`, `wiki/qa/index.md`, `wiki/platforms/index.md`,
`wiki/testing/quality/tests-that-cannot-fail.md`,
`wiki/qa/process/release-gates.md`,
`wiki/platforms/environment/timezone-and-locale.md`. Repo-wide dedup greps for
`grep|regex`, `mutation`, `fail-closed|fail-open`, `unicode|normalization|NFC|NFD`,
`definition of done|spec document`.

| Finding | Resolution |
|---------|------------|
| `testing-quality-tests-that-cannot-fail` owns the "prove it can fail" rule — the nearest overlap, but scoped to **test code** (assertion patterns, mocks, coverage) | Not duplicated. The new page reuses that principle for **documents** and links it as the parent; a reciprocal edge-case row was added there pointing here |
| `qa-process-release-gates` also uses the word "gate" | No overlap — that page decides *ship / block* for a build; this one is about the correctness of an automated check. Deliberately not linked, to avoid a false routing hop |
| `platforms-environment-timezone-and-locale` covers locale as a hidden input to **text** code (Turkish-i casing, collation) | Adjacent but a different trigger — normalization is a property of the text, not of `LANG`. Kept separate (that page is already at its section budget); added `related:` both ways plus an edge-case row there for "identical-looking strings compare unequal, no casing involved" |
| `platforms-filesystems-paths-case-and-line-endings` mentions "normalization" only for line endings and path APIs | No Unicode-normalization content exists anywhere in the wiki — genuinely new |
| `platforms-tools-bsd-vs-gnu-cli` owns regex-feature gaps (`grep -P`) | Linked instead of restated |
| No conflicting directive found on any page | Nothing flagged as `contradiction` |

Merge vs create: insights 2 and 4 are the **same situation** (authoring a gate
over a spec document) seen from two failure modes, so they became one page rather
than two; insight 1 is that same situation at authoring time, folded in as
directive 2 plus an `Instead of` row. Insight 3 has a different trigger (text
matching, independent of documents) and became its own page.

## Routing decision

| Insight | Target | Why |
|---------|--------|-----|
| 1, 2, 4 | `testing` / `quality` / **`document-verification-gates.md`** (new page, existing category), id `testing-quality-document-verification-gates` | AGENTS.md routing rule = the domain owning the artifact you change; the artifact is an automated check with assertions, and the governing principle (a check proves something only if it can fail) already lives in `testing/quality`. Rejected `qa`: that domain covers release-process decisions, not check authoring |
| 3 | `platforms` / `environment` / **`text-encoding-and-normalization.md`** (new page, existing category), id `platforms-environment-text-encoding-and-normalization` | Same category as `timezone-and-locale` — hidden inputs to text handling that differ per machine and writing system. Rejected a new category: `environment` already frames exactly this |

No new categories were needed. Plumbing updated: both domain `index.md` files
(page rows + widened domain framing), root `INDEX.md` (the testing and platforms
route lines), reciprocal `related:` on four existing pages, and one `log.md` entry.

Self-lint before commit: both pages carry all 5 template sections, are ≤120 body
lines (66 and 50), contain zero banned vague qualifiers and no bare prohibition
outside `Instead of`, and every `related:` / inline `[page-id]` reference resolves
to an existing page.
