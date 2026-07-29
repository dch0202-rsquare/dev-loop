---
id: testing-docs-as-spec-markdown-table-parsing
domain: testing
category: docs-as-spec
applies_to: [general]
confidence: verified
sources:
  - https://github.github.com/gfm/#tables-extension-
  - https://docs.github.com/en/rest/markdown/markdown
last_verified: 2026-07-29
related: [testing-docs-as-spec-gate-falsifiability, testing-quality-tests-that-cannot-fail]
---

# Parsing Markdown Table Rows in a Checker

## When this applies

You are writing a check that reads Markdown tables programmatically — counting
cells per row, extracting a column, or asserting a table's shape — in a repo
where documents act as spec. The documents contain EBNF, enums, type unions, or
CLI grammar, so cells hold characters that collide with table syntax.

## Do this

1. **Split rows on unescaped pipes only, then unescape each cell.** GFM treats
   `\|` inside a cell as a literal pipe, not a delimiter, so a plain
   `split("|")` over-counts every row that contains one:

   ```python
   cells = [c.strip().replace(r"\|", "|")
            for c in re.split(r"(?<!\\)\|", row.strip().strip("|"))]
   ```

2. **Assert cell counts against the header row's count, not a literal.** GFM
   drops cells beyond the header's column count and pads short rows, so a row
   the renderer accepted can still differ from a hardcoded expectation.

3. **Confirm the document renders as intended before you change it.** When a
   checker flags a row, render that row through a GFM renderer and read the cell
   count from the output — the rendered result decides whether the row or the
   checker is wrong. `POST /markdown` with `{"mode":"gfm"}` returns GitHub's own
   rendering of a snippet.

4. **Match the file's existing escape convention when you add rows.** A document
   that already writes `create\|update\|delete` has settled the question; a new
   row using a code span without escaping renders differently from its neighbors.

## Edge cases

| Case | Then |
|------|------|
| A cell needs a pipe inside a code span (`` `a\|b` ``) | Escape it there too — GFM requires the escape inside inline spans, and an unescaped pipe splits the cell even within backticks, silently truncating the content past the header's column count |
| A cell legitimately ends with a backslash before a delimiter (`\\|`) | The `(?<!\\)` lookbehind reads that delimiter as escaped and under-counts the row; use a parser that consumes escapes left-to-right, or forbid trailing backslashes in cells and check for them |
| The table is inside a fenced code block | Skip fenced regions before scanning for tables; a fenced example table is illustration, not spec, and flagging it produces noise that hides real breakage |
| Cells contain a full EBNF alternation and escaping hurts readability | Move the grammar to a fenced block and keep the table cell as a reference to it — the table stays parseable and the grammar stays readable |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Use `row.split("|")` and assert a fixed cell count | Split on `(?<!\\)\|`, unescape, and compare against the header count | An escaped pipe reports the row as over-wide, raising a "broken table" on a document that renders correctly |
| Fix a row the checker flagged, before checking the render | Render the row and read the cell count from the output first | Editing a correct row to satisfy a broken checker breaks the document and leaves the checker wrong |
| Add an unescaped pipe inside a code span in a table cell | Escape it (`\|`) inside the span | The cell splits and everything past the header's column count is dropped without an error |

## Sources

- https://github.github.com/gfm/#tables-extension- — "Include a pipe in a cell's content by escaping it, including inside other inline spans"
- https://docs.github.com/en/rest/markdown/markdown — `POST /markdown` with `mode: gfm` renders a snippet through GitHub's renderer, for confirming a row's real cell count
- Reproducible check (2026-07-29): the row `| EventSource | \`create\|update\|delete\` |` renders through `POST /markdown` as two cells, the second containing the literal `create|update|delete`; a naive `split("|")` reports four cells, while splitting on `(?<!\\)\|` reports two. The same row with unescaped pipes renders as two cells with `update` and `delete` dropped.
