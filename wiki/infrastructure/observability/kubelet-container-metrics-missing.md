---
id: infrastructure-observability-kubelet-container-metrics-missing
domain: infrastructure
category: observability
applies_to: [kubernetes, prometheus]
confidence: verified
sources:
  - https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml
  - https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/
  - https://github.com/kubernetes-monitoring/kubernetes-mixin
last_verified: 2026-07-15
related: [infrastructure-observability-logs-metrics-signals, infrastructure-observability-alerting, infrastructure-containers-resource-limits-and-probes]
---

# Kubelet Container Metrics Missing Though the Scrape Target Is Up

## When this applies

kube-prometheus-stack is installed and every kubelet scrape target reports
healthy (`up == 1`), but `container_cpu_*` / `container_memory_*` series are
empty and pod CPU/memory dashboards are blank. Common on non-standard kubelets
(OrbStack, Docker Desktop, some minimal/k3s runtimes) whose cAdvisor endpoint
emits only `machine_*` series.

## Do this

1. Separate two checks that are easy to conflate: **target UP** (the scrape
   succeeded) versus **expected series present** (the metric exists). A kubelet
   can serve `/metrics/cadvisor` with only `machine_*` and zero `container_*`
   series while the target stays green — health monitoring cannot see the gap.
   Verify a representative series directly: query
   `count(container_memory_working_set_bytes)`.
2. When cAdvisor emits no per-container series, scrape the kubelet **resource
   metrics** endpoint instead. In `kube-prometheus-stack` values set both:

   ```yaml
   kubelet:
     serviceMonitor:
       resource: true
       resourcePath: /metrics/resource
   ```

   `resource` defaults to `false`; `resourcePath` defaults to
   `/metrics/resource/v1alpha1`, which **404s on Kubernetes ≥ 1.18** (the path
   was renamed to `/metrics/resource`) — leave the default and the new target
   comes up **Down**. The resource endpoint exposes
   `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`,
   and the `pod_*` / `node_*` equivalents independently of cAdvisor.
3. Build a **custom** dashboard/recording rule for these series. The
   `/metrics/resource` series carry a different label set than cAdvisor — no
   `image`, `id`, or `name` label. kubernetes-mixin "Compute Resources"
   dashboards and its recording rules filter
   `container_memory_working_set_bytes{job="cadvisor", image!=""}` (to drop
   pause containers), so the resource-endpoint series are silently excluded.

## Edge cases

| Case | Then |
|------|------|
| cAdvisor endpoint returns `machine_*` but zero `container_*` | Not a scrape/auth failure — this runtime's cAdvisor emits no per-container series; switch to `/metrics/resource` |
| After `resource: true`, the kubelet-resource target is Down | The default `resourcePath: /metrics/resource/v1alpha1` 404s on k8s ≥ 1.18; set `resourcePath: /metrics/resource` |
| Built-in Compute Resources dashboard blank while PromQL returns rows | The mixin queries require `image!=""`; `/metrics/resource` has no `image` label — query without that filter in a custom panel |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Alert only on `up == 0` for the kubelet | Also alert on `absent(container_memory_working_set_bytes)` | A green target with an empty series set looks healthy but shows nothing |
| Assume cAdvisor is the only source of container usage | Scrape `/metrics/resource` when cAdvisor lacks it | The resource endpoint exposes container CPU/memory usage on its own |
| Rely on the mixin dashboards to render `/metrics/resource` data | Add a custom dashboard without the `image!=""` filter | The mixin rules require the `image` label only cAdvisor adds |

## Sources

- https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml — `kubelet.serviceMonitor.resource` (default `false`), `resourcePath` (default `/metrics/resource/v1alpha1`, comment: renamed to `/metrics/resource` since k8s 1.18), `cAdvisor: true`
- https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/ — kubelet exposes separate `/metrics/cadvisor` and `/metrics/resource` endpoints with different lifecycles
- https://github.com/kubernetes-monitoring/kubernetes-mixin — recording rule `node_namespace_pod_container:container_memory_working_set_bytes` filters `{job="cadvisor", image!=""}`
