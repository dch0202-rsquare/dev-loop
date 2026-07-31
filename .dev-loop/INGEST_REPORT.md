# Knowledge flush — 3 insight(s) researched, 2 ingested

Queue drained: `~/.dev-loop/queue/15f1a35f-….jsonl` (2 pending) and
`297b510d-….jsonl` (1 pending). All three were researched against primary
sources and deduped; **two are ingested here and one was dropped as a duplicate
of open PRs #11/#12** (details under "Existing-layer check"). Nothing was merged;
this PR is for review.

> **Dedup note — please read before merging.** The existing-layer check was run
> against `main` *and* against the seven open `knowledge/*` branches, because six
> knowledge PRs (#6–#12) are queued unmerged and their pages are invisible from
> `main`. That check removed one of the three insights from this PR. Merge-order
> notes for the two that remain are at the end of the routing section.

## Verified best-practice

### 1. A prompt-capable CLI called non-interactively blocks on stdin — **DROPPED (duplicate)**

> Researched and verified as below, then **dropped from this PR**: open PRs #11
> and #12 already add `wiki/platforms/processes/non-interactive-cli-invocation.md`
> covering the same case, and their version is better (it adds `ssh -o BatchMode`,
> the GNU-vs-POSIX `nohup` stdin distinction, and a pre-log-rejection branch in
> the client-vs-server table). The research is kept here only so you can fold the
> small delta into that page when you merge it — see "Existing-layer check".

**Claim as queued** — when an interactive-capable CLI agent is invoked with a
non-interactive flag (`-p`/`--print`) and hangs, re-run with stdin closed
(`</dev/null`, `nohup … &`); if it still hangs, check the server-side log for
request arrival to split client from server. The naive read ("the model or
gateway is slow") is wrong because the request was never sent.

**Sources checked**

- [OpenBSD `ssh(1)`](https://man.openbsd.org/ssh) — `-n`: "Redirects stdin from
  /dev/null (actually, prevents reading from stdin). **This must be used when ssh
  is run in the background.**" `-T` disables pty allocation (separate concern).
- [POSIX `nohup`](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/nohup.html)
  — nohup *may* redirect a terminal stdin but is not required to; POSIX
  prescribes `nohup make </dev/null &` for portable applications. This directly
  refutes "nohup already handles stdin".
- [POSIX XBD §11.1.4](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap11.html)
  — a background process group reading its controlling terminal is sent
  `SIGTTIN`, which is the "stopped, no output" variant of the same failure.
- [`debconf(7)`](https://manpages.debian.org/bookworm/debconf-doc/debconf.7.en.html)
  — `DEBIAN_FRONTEND` noninteractive is "the anti-frontend… never interacts with
  you at all, and makes the default answers be used for all questions".
- [git docs](https://git-scm.com/docs/git) — `GIT_TERMINAL_PROMPT=false` stops
  terminal prompting; `GIT_ASKPASS` supplies credentials without a terminal.
- [Claude Code headless docs](https://code.claude.com/docs/en/headless) —
  "Non-interactive mode **reads stdin**, so you can pipe data in"; tool approval
  in scripted runs needs `--allowedTools` or a permission mode. First-party
  confirmation of the exact mechanism for an agent CLI in print mode.

**How verified** — every load-bearing directive is stated in a first-party man
page, standard, or vendor doc (above). The session's own evidence was a
third-party CLI (`pi` 0.79.1) plus a gateway access log showing zero inbound
requests during the hang; that reproduction is consistent with the documented
mechanism, but the page does not rest on it — the page cites the primary sources
and states the client-vs-server split as a general diagnostic.

**Confidence: `verified`** — but not ingested; see the drop note above.

### 2. Pointing a client at an alternate service endpoint

**Claim as queued** — to run Claude Code against an OpenAI-compatible local
model, front it with LiteLLM, set `ANTHROPIC_BASE_URL` to the proxy (port only,
no `/v1`), and lower `CLAUDE_CODE_MAX_OUTPUT_TOKENS` to fit the model's context
window, because the default output cap plus the input exceeds a 128k window and
400s on the first request.

**Sources checked**

- [LiteLLM — Claude Code with non-Anthropic models](https://docs.litellm.ai/docs/tutorials/claude_non_anthropic_models)
  — documented example is `export ANTHROPIC_BASE_URL="http://0.0.0.0:4000"`,
  i.e. **no** version segment, with `ANTHROPIC_AUTH_TOKEN` as the credential.
  Confirms the base-URL form.
- [Ollama OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)
  — the OpenAI SDK's `base_url` **does** include the version segment
  (`http://localhost:11434/v1`) because the SDK appends `/chat/completions`.
  Together with the previous source this establishes the general point: the
  version segment is a per-client convention, not a rule.
- [AWS service-specific endpoints](https://docs.aws.amazon.com/sdkref/latest/guide/feature-ss-endpoints.html)
  — `AWS_ENDPOINT_URL` / `AWS_ENDPOINT_URL_<SERVICE>`, "scheme and host… can
  optionally contain a path component", plus the precedence order. Third
  independent convention; supports generalizing beyond LLM tooling.
- [Anthropic context windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)
  — the window "holds the conversation history **plus the new output**"; when
  input tokens plus `max_tokens` exceed it, pre-4.5 models "return a validation
  error" and 4.5+ models stop with `model_context_window_exceeded`. This is the
  documented mechanism behind the queued 400.
- [Claude Code env vars](https://code.claude.com/docs/en/env-vars) —
  `ANTHROPIC_BASE_URL` routes requests through a proxy or gateway;
  `CLAUDE_CODE_MAX_OUTPUT_TOKENS` caps response tokens.

**Correction made during verification** — the queued insight asserted the client
default output cap is **32000**. Current first-party docs give the default as
**16000** ("values above the model's maximum are capped silently"); the 32000
figure matches what that session's version enforced and what public bug reports
show, so the number is version-dependent. The page therefore states the
version-independent rule ("a max-output setting is reserved from the same
context window as the input; read the client's current documented default rather
than assuming a number") and does not hardcode 32000.

**Confidence: `verified`** for the directives as written (each cited above).

### 3. A verification harness needs a control in the pass direction

**Claim as queued** — when a harness (mutation check, gate, CI check) reports a
uniform result, feed it a known-answer input that must *pass*: a
semantics-preserving no-op mutant must SURVIVE. If it is reported as caught, stop
and report that nothing was measured. A broken isolation environment kills every
case before the rules run, which looks like 100% detection and is 0%.

**Sources checked**

- [Stryker — mutant states and metrics](https://stryker-mutator.io/docs/mutation-testing-elements/mutant-states-and-metrics/)
  — "Killed": at least one test **failed**; "Survived": all tests passed, "You're
  missing a test for it"; **"Runtime error": the run errored rather than a test
  failing, and "It is not represented in your mutation score"** (compile errors
  likewise). Strongest confirmation: a mature tool deliberately distinguishes
  *errored* from *failed* and refuses to count errors as kills — precisely the
  distinction a shell-out harness reading exit codes loses.
- [Stryker configuration](https://stryker-mutator.io/docs/stryker-js/configuration/)
  — `dryRunOnly`: "Execute the initial test run only without doing actual
  mutation testing", a first-class baseline run that exists to validate the setup
  before mutants are introduced.
- [Stryker — what is mutation testing](https://stryker-mutator.io/docs/) —
  killed/survived semantics; "It is expected that your unit tests will now fail."
- [Google Testing Blog — mutation testing](https://testing.googleblog.com/2021/04/mutation-testing.html)
  — inserting faults and requiring failure is what measures detection.

**How verified** — the two load-bearing mechanisms (error ≠ failure; a validated
baseline precedes scoring) are documented tool behavior. The control-group
formulation is the synthesis of those two, and it was reproducibly measured: an
independent audit reproduced RED with a semantics-preserving mutation
(capitalizing a docstring), proving the harness could not report anything but
"caught"; after the fix the score moved to 34/36 and the two survivors were real
unasserted rules. Both a doc basis and a reproduction exist.

**Confidence: `verified`.** The field incident is described in the page body as
context, clearly labelled, rather than being cited as a source.

## Existing-layer check

**Pages read before writing** (routing gate matches, per `AGENTS.md` step 2):

| Read | Overlap verdict |
|------|-----------------|
| `INDEX.md`, `wiki/platforms/index.md`, `wiki/testing/index.md`, `wiki/debugging/index.md` | Routing only |
| `platforms/processes/background-services.md` | Owns *keeping a process alive past the session*; adjacent to insight 1 (dropped) and to insight 2's "the call never left the machine" edge row. Left unmodified by this PR — PRs #11/#12 already edit it. |
| `platforms/environment/path-resolution.md` | **Adjacent, not duplicate.** Owns *which binary* an env input resolves to; insight 2 is *which endpoint* an env input redirects to. Same family (hidden environment inputs), different artifact → linked both ways. |
| `platforms/shells/portable-shell-scripts.md`, `platforms/toolchains/version-management.md` | Non-interactive shells appear in both, but only as "rc files are not read / PATH is missing" — no coverage of stdin, prompting, or pty. No conflict; linked from the new page. |
| `testing/quality/tests-that-cannot-fail.md` | **Closest existing page; explicitly extended rather than duplicated.** It owns the *per-test* question ("can this test go red?") including manual mutation and an edge row pointing at PIT/Stryker. Insight 3 is the *harness-level* question ("can this harness go green?") — a different trigger, and its final `Instead of` row names the distinction so the two do not blur. Added a new edge-case row in the existing page routing to the new one, plus `related:` both ways. |
| `testing/quality/minimum-case-set.md`, `behavior-not-implementation.md`, `testing/data/test-data-and-isolation.md`, `testing/flaky/diagnosing-flaky-tests.md` | Checked via their "load when" lines; no overlap with harness validity. Isolation-tree failures link to `test-data-and-isolation`. |

**Open-PR branches checked (not just `main`).** Six knowledge PRs are open, so
`main` is not the whole existing layer. Fetched all seven `knowledge/*` branches
from the fork and compared:

| Open PR | Adds | Verdict against this flush |
|---------|------|----------------------------|
| #11, #12 | `platforms/processes/non-interactive-cli-invocation.md` | **Duplicate of insight 1 → insight 1 dropped from this PR.** Delta worth folding into their page when you merge it: POSIX `nohup` (they cite the GNU manual; POSIX prescribes `nohup make </dev/null &` for portable use), POSIX XBD §11.1.4 `SIGTTIN` for the "Stopped (tty input)" variant, `DEBIAN_FRONTEND=noninteractive` for debconf tools, and pager/editor env (`PAGER`, `GIT_EDITOR`) as the same class of prompt channel. Also note #11 and #12 add the *same file* as each other. |
| #12 | `backend/common/integrations/externally-owned-defaults.md`, `llm-response-completeness.md` | **Not a duplicate of insight 2.** Those own "a default names a resource someone else controls" and "the response was truncated"; insight 2 owns "which endpoint the client talks to and whether its caps fit that backend". Read both to confirm. |
| #6 | `backend/common/llm/gateway-model-alias-defaults.md` | Same case as #12's `externally-owned-defaults`; neither overlaps insight 2. |
| #7 | `testing/quality/checks-that-cannot-pass.md` | **Closest neighbour to insight 3, and deliberately a different layer.** #7 owns validating *one check* before adopting it ("run it against a known-good input"). Insight 3 owns validating *the harness* that runs many checks in an isolated tree — its last `Instead of` row states the distinction explicitly ("per-check controls ask whether a check can go red; only a harness-level control asks whether the harness can go green"). If you merge #7 first, add a `related:` pair between the two pages; I did not add it here because the id does not exist on `main` and would fail lint as a broken link. |
| #8, #9, #10 | `spec-artifact-checks.md`, `document-conformance-checks.md`, `spec-document-gates.md` | Same family as #7, one level further out (artifact/doc conformance). No duplication with insight 3. |

**Merge-conflict warning:** PRs #7, #8, #10 and this one all touch
`wiki/testing/quality/tests-that-cannot-fail.md`. This PR's change to it is two
lines — one `related:` id and one edge-case row — so the conflict is mechanical,
but it will need resolving depending on merge order.

**Repo-wide greps run** for `stdin|tty|/dev/null|non-interactive`,
`mutation|mutant|control group|no-op`, and `litellm|anthropic|llm|openai|claude`
to catch coverage outside the routed categories. Only pre-existing incidental
mentions were found (listed above); the LLM-tooling grep returned a single
unrelated hit in `security/secrets/secrets-in-code.md`.

**Conflicts flagged: none.** No existing directive is contradicted; the one
factual correction (output-token default) is against the queued candidate, not
against the wiki.

**Related links added (both directions)**

- `platforms-environment-alternate-service-endpoints` ⇄ `platforms-environment-path-resolution`
- `testing-quality-verification-harness-validity` ⇄ `testing-quality-tests-that-cannot-fail`
- One-way outbound: `platforms-processes-background-services`,
  `debugging-methodology-hypothesis-testing`, `infrastructure-config-environment-config`,
  `testing-data-test-data-and-isolation`.

## Routing decision

| Insight | Target | Why |
|---------|--------|-----|
| 1 | **not ingested** — routed to `platforms/processes/`, then found already occupied by open PRs #11/#12 | Correct target existed; the page there is already better sourced than a third copy would be. Fold the delta listed above into that page instead. |
| 2 | `platforms/environment/alternate-service-endpoints.md` (**existing** category `environment`) | Generalized from "Claude Code + LiteLLM" to "a client redirected to an alternate endpoint by env var", which is what makes it reusable and keeps it out of vendor-page territory. `environment` is the category for *hidden environment inputs that change what a tool does*; it already holds `path-resolution` (which binary) and `timezone-and-locale` (which formatting), so "which endpoint" is the same shape. **No new category created.** |
| 3 | `testing/quality/verification-harness-validity.md` (**existing** category `quality`) | `quality` owns "can this test detect anything". A new page rather than a merge because the trigger differs: `tests-that-cannot-fail` triggers on auditing tests, this one on citing a harness's score. Merging would have violated one-case-per-page and buried the harness case under a test-level headline. |

**New categories created: none.** Both ingested pages landed in existing
categories, so no category justification is required.

**Domain-charter edit (wiki layer, deliberate).** `INDEX.md`'s platforms route
line and `wiki/platforms/index.md`'s route paragraph were widened by one phrase —
"environment inputs that redirect a tool (PATH, endpoint/base-URL overrides)" —
so the new page is reachable from the root index (maintenance invariant 1).
Flagging it because it touches how the domain describes itself: **if you would
rather platforms stay strictly OS-difference-scoped, reject that hunk** and
insight 2 needs a home decision from you (my second choice would be
`infrastructure/config/`, which already owns environment configuration).

**Merge-order dependencies** (repeated from the open-PR table): fold insight 1's
delta into #11/#12's `non-interactive-cli-invocation.md`; add a `related:` pair
between `checks-that-cannot-pass` (#7) and `verification-harness-validity` (here)
once both are on `main`; expect a two-line conflict in `tests-that-cannot-fail.md`
against #7/#8/#10.

**Plumbing:** both domain indexes list the new pages with "load when" lines
enumerating their distinct use cases; `log.md` has the dated ingest entry, which
also records the dropped insight. Local lint checks pass — body lengths 69 and 77
lines (limit 120), zero banned vague qualifiers, all `related:` ids and inline
`[page-id]` references resolve against `main` + this branch.
