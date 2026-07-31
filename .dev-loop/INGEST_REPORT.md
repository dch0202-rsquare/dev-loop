# Knowledge flush — 3 insight(s)

Drained `~/.dev-loop/queue` (3 pending candidates, sessions `15f1a35f…` and
`51223260…`). Each was researched against primary sources, checked against the
existing wiki layers, routed, and ingested. Result: 2 new pages, 1 merge into an
existing page, 1 new category.

## Verified best-practice

### 1. An OpenAI-compatible 200 response is not a usable answer
**Claim** — before using chat-completion text as a deliverable, fail on
`finish_reason == "length"` and on blank `content`, and use a populated
`reasoning` / `reasoning_content` field to distinguish "a reasoning model was
routed in and spent the output budget" from other empty-output causes.

**Sources checked (all fetched live, 2026-07-31):**
- `https://developers.openai.com/api/docs/api-reference/chat/object` — confirms
  the `finish_reason` vocabulary: `stop` = "the model hit a natural stop point or a
  provided stop sequence"; `length` = "the maximum number of tokens specified in
  the request was reached"; plus `tool_calls` and `content_filter`.
- `https://developers.openai.com/api/docs/guides/reasoning` — confirms the
  mechanism: reasoning tokens "occupy space in the model's context window and are
  billed as output tokens"; hitting the limit returns `status: incomplete` with
  `incomplete_details.reason = max_output_tokens`; and explicitly, "this might
  occur before any visible output tokens are produced, meaning you could incur
  costs for input and reasoning tokens without receiving a visible response."
  Also the ≥25,000-token reservation guidance now in the page.
- `https://docs.vllm.ai/en/latest/features/reasoning_outputs/` — **corrected the
  candidate on a real point**: the field the queued insight named
  (`reasoning_content`, DeepSeek's name) has been renamed to `reasoning`
  ("`reasoning` used to be called `reasoning_content`"). The page therefore
  directs reading **both** names, notes the field requires `--reasoning-parser`
  server-side, and notes the official OpenAI client types declare neither.

**How verified** — the general mechanism is documented by the vendor above; the
concrete failure shape came from measurement in this environment (LiteLLM
gateway, 2026-07-28): a 9,317-token summarization prompt returned HTTP 200 with
`finish_reason=length`, `content` 0 chars, `reasoning_content` 8,173 chars, while
the same prompt on a non-reasoning model returned `finish_reason=stop` with 2,418
chars. **Confidence: `verified`** (vendor docs + reproducible measurement).

### 2. A non-interactive flag does not detach a child CLI from the terminal
**Claim** — when a script/CI/agent call to an interactive-capable CLI hangs with
no output, wire stdin explicitly (`</dev/null`, or `nohup … & disown`) and bound
it with a timeout; if it still hangs, use the server's own arrival record to split
client hang from server latency.

**Sources checked (fetched live, 2026-07-31):**
- `https://www.gnu.org/software/coreutils/manual/html_node/nohup-invocation.html`
  — nohup redirects stdin when it is a terminal "so that terminal sessions do not
  mistakenly consider the terminal to be used by the command", and makes the
  substitute descriptor unreadable so the command errors rather than reading stdin.
  This is the documented, vendor-blessed form of the directive.
- `https://man.openbsd.org/ssh.1` — `-n`: "Redirects stdin from /dev/null
  (actually, prevents reading from stdin). This must be used when `ssh` is run in
  the background." A second independent tool encoding the same rule as mandatory.

**How verified** — documented behaviour of two standard tools, plus the field
reproduction behind the candidate: `pi 0.79.1 --tools` hung twice in the
foreground (300s and 150s, 0 bytes of output) while the LiteLLM gateway logged
**zero** requests from that IP in the window; the identical command under `nohup`
(stdin `/dev/null`) completed a tool call immediately.
**Confidence: `verified`.**

### 3. A dated "verified end to end" claim expires when the name is owned elsewhere
**Claim** — when a default names a resource owned outside the repo, re-resolve it
against the live provider at review/merge/gate time rather than trusting the
change's own earlier measurement.

**Sources checked (fetched live, 2026-07-31):**
- `https://developers.openai.com/api/docs/deprecations` — substantiates the
  mechanism with dated examples: `gpt-3.5-turbo-0613` shut down 2024-09-13;
  `dall-e-2` / `dall-e-3` removal scheduled 2026-05-12. An identifier that
  resolved earlier stops resolving with no change on the caller's side.
- `https://sre.google/sre-book/release-engineering/` — already cited by the target
  page; supports expressing this as a fixed gate layer rather than per-release
  judgement.

**How verified** — the provider-side mechanism is documented; the "code, tests and
build all stay green while the default is dead" half is production experience
(meetingSummary PR#15: the default summarization alias `dgxb/kanana` had
disappeared from the gateway catalog and returned 400, after the PR body's own
measurement was taken and was true on its date). The target page's existing
`confidence: field-tested` is unchanged and correct for this addition.
**Confidence: `field-tested`** (provider-side mechanism `verified`; the
green-CI-with-dead-default half rests on production experience).

Nothing was left `unverified`; no candidate was dropped.

## Existing-layer check

**Pages read in full:** `AGENTS.md`, `INDEX.md`, `templates/page.md`,
`wiki/backend/index.md`, `wiki/platforms/index.md`, `wiki/debugging/index.md`,
`wiki/qa/index.md`, `wiki/platforms/processes/background-services.md`,
`wiki/platforms/shells/portable-shell-scripts.md`,
`wiki/debugging/signals/logs-and-correlation.md`,
`wiki/infrastructure/config/environment-config.md`,
`wiki/testing/mocking/what-to-mock.md`, `wiki/qa/process/release-gates.md`.
Also grepped the whole `wiki/` tree for
`llm|reasoning_content|finish_reason|openai|stdin|tty|non-interactive`.

**Overlaps found and how each was resolved:**

| Existing page | Overlap | Resolution |
|---|---|---|
| `platforms/processes/background-services` | Owns *detaching* a process (nohup/launchd/systemd) and has an edge case for harness background tasks; says nothing about a foreground call blocking on an inherited TTY stdin | Kept separate (different case: invocation, not lifetime). Cross-linked both ways; the new page delegates its "must outlive the session" row to it |
| `platforms/shells/portable-shell-scripts` | Covers quoting, `set -e` blind spots, zsh word-splitting; no stdin/TTY content | No merge. Linked as `related` |
| `debugging/signals/logs-and-correlation` | **Directly constrains** the new page's step 5 — it already warns: "Conclude 'service B never received it' because B has no log → check whether B logs that path at all". Honored rather than duplicated: the new page's arrival table carries an explicit third row ("cannot confirm the path is always logged → absence is not evidence; send a known-good probe first") pointing back at that page. Linked both ways |
| `infrastructure/config/environment-config` | Nearest neighbour to insight 3 — item 5 and an `Instead of` row cover *missing* / dev-only defaults. Distinct failure: insight 3 is a default that is **present and syntactically fine but no longer resolves upstream** | No merge; left untouched so its "required keys get no default" rule stays undiluted |
| `testing/mocking/what-to-mock` | Explains mocking unowned I/O — the reason CI stays green — but is about test-double choice, not verifying a live external identifier | No merge; the consequence is captured as a release-gates edge case instead |
| `qa/process/release-gates` | Same case as insight 3 (a fixed checklist deciding ship/merge) | **Merged here** rather than creating a page |
| `backend/common/{api-design,reliability,errors}` | api-design owns the API *you* serve; reliability owns timeouts/retries against a dependency; errors owns exception structure. None owns the *response shape* of a model-serving API | New category (see below); linked `reliability/timeouts-and-retries` both ways |

**Conflicts flagged:** none. No existing directive is contradicted or overwritten.

**Reciprocal `related:` links added:** `background-services` ↔ new platforms page;
`logs-and-correlation` ↔ new platforms page; `reliability/timeouts-and-retries` ↔
new backend page; `qa-process-release-gates` ↔ new backend page.

## Routing decision

| Insight | Target | New? |
|---|---|---|
| 1 — LLM response completeness | `backend` / **new category `llm`** / `wiki/backend/common/llm/response-completeness-validation.md` (`backend-common-llm-response-completeness-validation`) | New page + new category |
| 2 — non-interactive CLI hang | `platforms` / `processes` / `wiki/platforms/processes/non-interactive-cli-invocation.md` (`platforms-processes-non-interactive-cli-invocation`) | New page, existing category |
| 3 — externally-owned names go stale | `qa` / `process` / `wiki/qa/process/release-gates.md` | **Merged** into the existing page: +1 gate layer, +2 edge cases, +1 `Instead of` row, +1 source, `last_verified` bumped |

**New category `backend/common/llm` — why the existing ones don't fit.** The case
is "a response from a model-serving API I don't own, whose shape changes with which
model was routed in". `api-design` governs the contract *this* service publishes;
`reliability` governs transport concerns (timeouts, retries, concurrency caps) and
its page is about failing calls, not 200s with unusable bodies; `errors` governs
where exceptions are caught and translated; `caching`, `jobs`, `auth`, `orm`,
`concurrency` are unrelated. Filing it under any of those would make the routing
line lie about the page's trigger. The concern is language-agnostic, so it belongs
in the `common` subtree rather than a stack subtree.

**Routing for insight 2 — why `platforms/processes`, not `debugging`.** The
directive changes how you *write the call site* (stdin wiring, timeout), the same
concern `processes` already owns via `background-services`; the diagnostic half
delegates to `debugging/signals/logs-and-correlation` by reference instead of
restating it.

**Plumbing updated:** `wiki/backend/index.md` (new `### llm` section; the `common`
routing line now names LLM/model-API responses), `wiki/platforms/index.md` (new row
under `## processes`), `log.md` (2 `ingest` + 1 `revise` entry). `INDEX.md` needed
no change — both domains were already listed and seeded.

**Invariant checks run:** every new page is listed in its domain index with a
"load when" line consistent with its "When this applies"; all `related:` ids across
`wiki/` resolve to existing page ids (0 dangling); new pages are 78 and 79 lines
(limit 120); no banned vague qualifier appears in a directive sentence (the two
grep hits are a quoted man-page sentence and a pre-existing quoted anti-pattern
phrase).
