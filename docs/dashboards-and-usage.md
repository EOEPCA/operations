# Dashboards and Usage

The EOEPCA demo ships Grafana dashboards for the monitoring system, Kubernetes
resources, the STAC SLO scenario, and security controls. In the
[operating model](operating-model.md), dashboards turn Inspect signals into
evidence an operator can use during Investigate.

## How Dashboards Are Provisioned

The dashboards are generated as ConfigMaps under [`argocd/eoepca/operations/parts/_dashboards`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/_dashboards) and labeled with `grafana_dashboard: "1"`.

Grafana loads them automatically as managed dashboards, so no manual import is
needed.

The live `operations` namespace contains dashboard ConfigMaps for:

- `curated-prometheus-overview`
- `curated-k8s-resources-cluster`
- `curated-k8s-resources-node`
- `curated-k8s-resources-pod`
- `curated-kyverno-policy-audit`
- `curated-stac-slo`
- `curated-trivy-security`

The Kyverno chart also provides the `kyverno-grafana-grafana` dashboard. Use
the chart dashboard for controller behaviour and the curated policy-audit
dashboard for EOEPCA policy results.

## Current Curated Dashboards

### Prometheus / Overview

Source: [`prometheus-overview.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/prometheus-overview.json)

This dashboard helps operators understand whether the monitoring system itself is healthy. It includes views such as:

- discovery
- target sync
- scrape failures
- appended samples
- query rate
- Prometheus storage state

Start here when monitoring results look incomplete or wrong.

### Kubernetes / Cluster

Source: [`k8s-resources-cluster.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/k8s-resources-cluster.json)

This dashboard is useful for answering questions like:

- which namespaces are consuming most CPU or memory?
- how much cluster capacity is committed through resource requests and limits?
- where is network or storage activity concentrated?

Start here when the platform is unhealthy but the affected namespace or
workload is not yet known.

### Kubernetes / Workload

Source: [`k8s-resources-pod.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/k8s-resources-pod.json)

This dashboard focuses on pods and containers. It helps answer:

- which container is using CPU?
- is the workload being throttled?
- how does memory working set compare to requests and limits?
- is network or disk activity unusual for this pod?

Use this after finding the relevant namespace.

### Kubernetes / Node

Source: [`k8s-resources-node.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/k8s-resources-node.json)

This dashboard shows node-level data. Use it for cluster saturation, scheduling
pressure, or a node problem rather than a single application fault.

### STAC / SLO

Source: [`stac-slo.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/stac-slo.json)

This dashboard is the EO platform example dashboard. It focuses on the STAC route and uses the recording rules from [`stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_rules/stac-alerts.yaml) to show:

- STAC GET and POST latency burn rates
- the route currently used by the STAC APISIX rule
- database mean execution latency from PostgreSQL exporter data
- request, upstream application, and gateway views that help locate the likely layer of degradation

### Security / Kyverno Policy Audit

Source: [`kyverno-policy-audit.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/kyverno-policy-audit.json)

This dashboard shows the results of the Restricted Pod Security audit. Use it
to compare pass, fail, warning, error, and skipped results and to find the
affected namespace or policy. Check the age of the underlying policy reports
before you trust a quiet dashboard.

The main views show:

- policy failures
- policy exceptions and skipped results
- policy-engine errors
- result rate by policy, rule, and result

### Security / Trivy

Source: [`trivy-security.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/trivy-security.json)

This dashboard summarizes vulnerability, exposed-secret, configuration, RBAC,
and compliance findings. Use it to find an area that needs investigation. Then
open the matching Trivy report resource to confirm the image, object, finding,
and report age. A dashboard total is not proof that every workload was scanned.

The main views show:

- critical image vulnerabilities
- high or critical exposed secrets
- critical configuration and RBAC findings
- failed compliance checks
- security-finding trends
- vulnerability and configuration findings by namespace

## Typical Operator Usage

A practical operator workflow often looks like this:

1. Start with the dashboard named in the alert or the dashboard for the
   affected service.
2. Use the Prometheus Overview dashboard first if data is missing or looks
   inconsistent.
3. Use the STAC SLO dashboard for a STAC symptom. Compare request, upstream,
   gateway, and database views.
4. Use Cluster View to locate the affected namespace. Then use Workload View
   for the Pod or controller.
5. Use Node View only when the evidence points to scheduling, capacity, or one
   node.
6. Use the Kyverno or Trivy dashboard for a security signal. Confirm the result
   in the source policy or scan report.
7. Use Loki when metrics show the symptom but not the component detail.

## Apply Dashboards Well

- Set the time range to include the first symptom and the most recent relevant
  change.
- Keep service, namespace, workload, route, and cluster filters consistent when
  you compare panels.
- Treat an empty panel as unknown until you confirm the target, query, and data
  source.
- Open the source metric, log, policy report, or scan report before you approve
  a change.
- Reuse the same dashboard and filters during Verify so that before-and-after
  evidence is comparable.

## EO Platform Dashboards

Application-specific metrics from EO services are especially valuable in
Grafana. In the eoAPI STAC path, `stac-auth-proxy` now exposes request and
latency metrics that can be filtered by operation. Native metrics from
`stac-fastapi-pgstac` are still being established.

The curated STAC SLO dashboard does not yet use these operation-specific
series; it remains based on the APISIX route and supporting upstream and
database records.

The [end-to-end STAC example](stac-scenario.md#stac-auth-proxy-metrics)
describes how the new metrics complement the existing dashboard signals.
