# Knowledge flush — 3 insight(s)

Drained 3 queued `★ Insight` candidates from `~/.dev-loop/queue` (sessions
`4de22ef2…` and `c55e6d60…`). Each was researched and verified against primary
sources before ingest. 2 new pages + 1 merge; one PR.

## Verified best-practice

### 1. Kubelet container metrics missing though scrape target is up (infrastructure/observability)
- **Claim:** On kube-prometheus-stack, a non-standard kubelet (OrbStack, Docker
  Desktop, minimal k3s) can serve `/metrics/cadvisor` with only `machine_*` and
  zero `container_*` series while the scrape target stays green. Enable
  `kubelet.serviceMonitor.resource: true` **and**
  `resourcePath: /metrics/resource` (the default `/metrics/resource/v1alpha1`
  404s on k8s ≥ 1.18); its series lack an `image` label, so kubernetes-mixin
  dashboards (which filter `image!=""`) hide them → custom dashboards needed.
- **Sources checked:**
  - kube-prometheus-stack `values.yaml` — fetched: `resource: false` (default),
    `resourcePath: "/metrics/resource/v1alpha1"` with the comment "From
    kubernetes 1.18, /metrics/resource/v1alpha1 renamed to /metrics/resource",
    `cAdvisor: true`. → confirms the two required toggles + the 404 trap.
  - Kubernetes docs *Metrics For Kubernetes System Components* — kubelet exposes
    separate `/metrics/cadvisor` and `/metrics/resource` endpoints with
    different lifecycles.
  - kubernetes-mixin — recording rule
    `node_namespace_pod_container:container_memory_working_set_bytes` filters
    `{job="cadvisor", image!=""}` (drops pause containers); confirms
    `/metrics/resource` series (no `image` label) are excluded by the mixin.
  - Candidate's own reproduction: cAdvisor endpoint 0 `container_` series;
    `/metrics/resource` returned `container_memory_working_set_bytes` = 502MB
    for querypie-agent; default `resourcePath` target Down → fixed to
    `/metrics/resource`.
- **Confidence:** verified (chart source + docs + mixin rule + reproduction).

### 2. Container PID1 losing buffered log output at exit (infrastructure/containers)
- **Claim:** A container PID1 bash script using `exec > >(tee -a "$LOG")` loses
  output (short jobs all, long jobs the tail) because bash does not wait for the
  process-substitution child and the container is killed the instant PID1 exits.
  Fix: save `$!`, close fds and `wait` the tee in an `EXIT` trap, plus
  `trap 'exit 143' TERM` so orchestrator stops run the trap.
- **Sources checked:**
  - Greg's Wiki *ProcessSubstitution* — fetched: "it will continue to run when
    your script exits (unless you manage your child processes)"; since bash 4.4,
    `wait "$!"` synchronizes a process substitution. → confirms mechanism + fix.
  - GNU Bash manual, Process Substitution — asynchronous subshell over a pipe/FIFO.
  - Candidate's reproduction on an OrbStack container: no trap → 0/10 runs
    captured the tail; EXIT-trap `wait` → 10/10.
- **Confidence:** verified (documented bash behavior + reproduction).

### 3. Detect GNU `date` by `%3N` output, not `%N` support (platforms/tools)
- **Claim:** Modern macOS `date` prints nanosecond digits for `%N` but the
  literal string `3N` for the width form `%3N`; feature-detecting GNU by "`%N`
  is supported" is wrong and corrupts millisecond timestamps to `…08.3NZ`.
  Detect by whether `date +%3N` yields three digits.
- **Sources checked:**
  - **Direct reproduction on this host (macOS 26.5.1):** `date +%N` →
    `908606000` (digits), `date +%3N` → `3N` (literal),
    `date +%s.%3N` → `1784091784.3N`. Exactly matches the candidate.
  - GNU coreutils manual (time conversion specifiers) — documents the `%N`
    width form that BSD `date` does not implement.
- **Confidence:** verified (reproduced first-hand + GNU manual).

## Existing-layer check

- **Read for routing/dedup:** `INDEX.md`; `wiki/infrastructure/index.md`
  (observability + containers categories); `wiki/platforms/index.md`;
  `wiki/infrastructure/observability/logs-metrics-signals.md` +
  `alerting.md`; `wiki/infrastructure/containers/resource-limits-and-probes.md`;
  `wiki/platforms/tools/bsd-vs-gnu-cli.md`;
  `wiki/platforms/shells/portable-shell-scripts.md`.
- **Insight 1:** no existing observability page covers a green scrape target
  with absent series or the kube-prometheus-stack kubelet config —
  `logs-metrics-signals` is app-instrumentation, `alerting` is paging policy.
  → **new page**, cross-linked to both (which now `related:`-link back).
- **Insight 2:** no containers page covers PID1 process-model / log flushing;
  `resource-limits-and-probes` is limits/probes (adjacent on SIGTERM only),
  `portable-shell-scripts` is shell portability (adjacent on the bash
  process-sub angle). No single-case fit → **new page**, with reciprocal
  `related:` links added to both adjacent pages.
- **Insight 3:** `platforms/tools/bsd-vs-gnu-cli.md` already owns the `date`
  BSD-vs-GNU case (relative-date row). Same trigger, additive fact → **merged**
  (one Do-table row for ms stamps + one Edge-case row), not a new page.
- **Conflicts:** none found; all edits additive.

## Routing decision

| Insight | Target | Action |
|---------|--------|--------|
| 1 | `infrastructure/observability/kubelet-container-metrics-missing.md` | new page (existing observability category) |
| 2 | `infrastructure/containers/pid1-log-flushing.md` | new page (existing containers category) |
| 3 | `platforms/tools/bsd-vs-gnu-cli.md` | merge into existing page |

No new categories created — all three fit existing categories
(observability, containers, tools). Plumbing updated:
`wiki/infrastructure/index.md` gained "load when" rows for both new pages;
reciprocal `related:` links added on `logs-metrics-signals`, `alerting`,
`resource-limits-and-probes`, `portable-shell-scripts`; `log.md` has the ingest
entry. Body sizes 63 and 54 lines (≤120). All `related:` ids resolve.
