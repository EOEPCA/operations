# Dashboards and Usage

The EOEPCA demo ships Grafana dashboards for the monitoring system, Kubernetes
resources, STAC, workspaces, and security controls. In the
[operating model](operating-model.md), operators use dashboards to compare
Inspect results during Investigate.

## How Dashboards Are Provisioned

The dashboards are generated as ConfigMaps under [`argocd/eoepca/operations/parts/_dashboards`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/_dashboards) and labeled with `grafana_dashboard: "1"`.

Grafana loads them automatically as managed dashboards, so no manual import is
needed.

The live `operations` namespace contains dashboard ConfigMaps for:

- `curated-prometheus-overview`
- `curated-k8s-resources-cluster`
- `curated-k8s-resources-node`
- `curated-k8s-resources-pod`
- `curated-security-compliance`
- `curated-stac-slo`
- `workspaces-overview`

The Security Compliance dashboard combines Kyverno policy results and Trivy
scan findings. It replaces the separate policy-audit and Trivy dashboards.

## Current Dashboards

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

### Kubernetes / Pods

Source: [`k8s-resources-pod.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/k8s-resources-pod.json)

This dashboard focuses on pods and containers. It helps answer:

- which container is using CPU?
- is the workload being throttled?
- how does memory working set compare to requests and limits?
- is network or disk activity unusual for this pod?

Use this after finding the relevant namespace.

### Kubernetes / Nodes

Source: [`k8s-resources-node.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/k8s-resources-node.json)

This dashboard shows node-level data. Use it for cluster saturation, scheduling
pressure, or a node problem rather than a single application fault.

### STAC

Source: [`stac-slo.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/stac-slo.json)

This dashboard has separate sections for `stac-auth-proxy` and `eoapi-stac`.
Each uses that service's request counters and latency histograms to show:

- the share of GET and POST requests within 500 ms over 30 days
- latency error budget use and one-hour and six-hour burn rates
- request rate, HTTP 5xx errors, and p99 latency
- mean query latency for the shared eoapi database in the backend section

The target is 99% of requests within 500 ms. A burn rate above one uses the
budget faster than allowed; more than 100% budget used exceeds the target.
No traffic shows no data. Proxy and backend request counts overlap, so do not
add them. The alert rules still measure the public APISIX route.

### Security Compliance

Source: [`security-compliance.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/security-compliance.json)

This dashboard shows policy results and vulnerability, exposed-secret,
configuration, RBAC, and compliance findings. Start with critical findings,
then use the action tables for secrets and access, image updates, or deployment
configuration. Open the matching Trivy or Kyverno report to confirm the image,
resource, finding, and report age before making a change.

The main views show:

- critical findings that need review first
- high and critical image vulnerabilities and secret findings
- configuration and access findings
- failed policy rules and evaluation errors
- collection errors, report ages, exceptions, and cluster controls

Repeated workload reports for the same image count once per image and
namespace. Secret findings come from image scans, not scans of live Kubernetes
Secrets. Cluster controls and ClusterRole findings apply to all namespaces.

The collapsed review section contains scan health and exception details. Check
these before accepting zero findings. A dashboard total does not prove that
every workload was scanned.

### Workspaces

Source: [`workspaces-overview.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/workspaces-overview.json)

Select one or more workspaces to compare running sessions, CPU and memory use,
requested PVC storage, and owned buckets. The dashboard includes the main
workspace namespace and its session and control-plane namespaces. It opens
with 24 hours of history.

Managed data panels show HTTP requests, successful and failed presigned URL
requests, transfer bytes, and collection health. Activity totals cover the
selected period. Requested PVC storage is allocated capacity; transfer bytes
measure data movement. Neither measures stored data.

The [workspace scenario](workspace-scenario.md) explains the session counts,
recording rules, PodMonitor, and a short investigation example.

## Typical Operator Usage

For a reported problem:

1. Start with the dashboard named in the alert or the dashboard for the
   affected service.
2. Use the Prometheus Overview dashboard first if data is missing or looks
   inconsistent.
3. Use the STAC dashboard for a STAC symptom. Compare proxy, backend, and
   database views; check APISIX metrics for public-route delays.
4. Use Kubernetes / Cluster to locate the affected namespace. Then use
   Kubernetes / Pods for the pod or container.
5. Use Kubernetes / Nodes when the evidence points to scheduling, capacity,
   or one node.
6. Use Security Compliance for a security signal. Confirm the result in the
   source policy or scan report.
7. Use Workspaces for session, resource, or managed data problems.
8. Use Loki when metrics show the symptom but not the component detail.

## Dashboard Checks

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

Application metrics help operators distinguish service problems from
infrastructure problems. The STAC dashboard uses proxy and backend metrics
alongside the database timing. The Workspaces dashboard links Kubernetes
resource use to workspace sessions and managed data activity.

The [STAC example](stac-scenario.md#stac-auth-proxy-metrics) and
[workspace example](workspace-scenario.md) describe how to compare these metrics
with logs, workload health, and service checks.
