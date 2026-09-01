# Monitoring Stack

This page explains why each monitoring component exists, what an operator can
do with it, and how the pieces fit together in the EOEPCA demo. In the
[operating model](operating-model.md), this stack implements much of Inspect
and provides the evidence and tools used during Investigate. The deployment
sources live in
[`eoepca-plus/argocd/eoepca/operations`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations).

## Deployed Components

The `operations` namespace contains the operator-facing monitoring and alerting
stack:

| Component | Why it exists | What an operator does with it |
| --- | --- | --- |
| Prometheus | Converts metrics endpoints into queryable time series and evaluates rules. | Check target health, query trends and ratios, and inspect active rules and alerts. |
| Grafana | Puts related metric and log evidence in one interface. | Start from a service or platform dashboard, then narrow the time range and affected resource. |
| Alertmanager | Prevents every matching rule from becoming a separate notification. | Inspect grouped alerts, routing, inhibition, and delivery status. |
| Loki | Keeps searchable logs without using Prometheus for high-volume event text. | Query the affected namespace, workload, request, or audit identifier during Investigate. |
| Grafana Alloy | Discovers and forwards Pod, node-service, and Kubernetes API audit logs to Loki. | Check the collection source when expected logs are missing. |
| Keep | Adds context and ownership to alert triage. | Review the incident, related signals, assignee, status, and response progress. |
| Keep OAuth2 Proxy | Restricts access to the Keep interface. | Use the approved Keycloak roles to access Keep. |
| Keep relay | Adapts Alertmanager webhooks to the Keep event endpoint. | Check it when Alertmanager delivery succeeds but the event does not appear in Keep. |

The deployment definitions are under
[`parts/monitoring`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/monitoring)
and
[`parts/alerting`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/alerting).

## What Each Component Does

### Prometheus

Prometheus is the core metrics engine. In the EOEPCA demo it:

- scrapes platform and application targets
- evaluates baseline and STAC-specific rules
- stores time series for dashboards and alerting

The retention and persistence settings are defined in [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/app-kube-prometheus-stack.yaml).

### Grafana

Grafana is the operator-facing dashboard layer. In the demo it is configured with OIDC login and a Loki data source alongside Prometheus.

That combination matters because operators usually need both:

- Prometheus for trends, ratios, and alert queries
- Loki for contextual log exploration

### Alertmanager

Alertmanager groups, routes, and deduplicates alerts. It delivers alerts, but
it does not guide the operator through the response.

### Loki

Loki stores logs collected from Kubernetes pods. The current deployment uses:

- a single-binary Loki setup
- object storage on S3-compatible backend
- a chart-managed ServiceMonitor for Loki metrics
- Loki canaries to exercise the log pipeline

See [`app-loki-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/app-loki-stack.yaml).

### Grafana Alloy

Alloy runs as a DaemonSet on each node. The current configuration:

- collects pod logs from Kubernetes
- collects normalized Kubernetes API audit events and selected node-service
  journal records
- attaches useful labels such as namespace, pod, container, and app
- forwards logs to Loki

The pipeline is defined in [`monitoring/alloy/config.alloy`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/alloy/config.alloy).

### Keep

Keep enriches alerts and supports triage. It lets operators see an alert and
related context in one place instead of immediately searching raw PromQL or
cluster state.

This helps when several systems can cause one visible symptom, which is common
on EO platforms.

## Apply the Stack Well

Use the tools as one evidence path:

1. Start with the affected service, operation, and time window in Keep or the
   relevant Grafana dashboard.
2. Use Prometheus to confirm the trend, user impact, and affected labels.
3. Compare gateway, application, workload, and dependency signals. Do not infer
   a root cause from one graph.
4. Use Loki when you need request, error, audit, or component detail.
5. Check scrape targets, Alloy, and Loki canaries when expected evidence is
   missing. An empty query can mean a broken collection path.
6. After an action, repeat the user-facing check and the relevant metric and log
   queries. Do not use Pod readiness as the only recovery test.

## Public Operator Endpoints

The demo currently exposes:

- Grafana at <https://monitoring.develop.eoepca.org>, requiring a `monitoring:grafana-admin`, `monitoring:grafana-editor`, or `monitoring:grafana-viewer` role in Keycloak
- Keep at <https://alerting.develop.eoepca.org>, requiring an `alerting:keep-admin` or `alerting:keep-noc` role in Keycloak

These endpoints are exposed through APISIX routes:

- [`monitoring/routes.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/routes.yaml)
- [`alerting/routes.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/routes.yaml)

## What This Gives Operators

With the current stack, operators can already:

- inspect cluster and workload resource behaviour in Grafana
- query logs from EO platform workloads through Loki
- evaluate Prometheus rules and Alertmanager routing
- use Keep to enrich and triage alert events

EO platform services contribute service-specific evidence through managed
scrape targets. The [ServiceMonitor](service-monitors.md) page explains this
pattern, and the [STAC end-to-end example](stac-scenario.md) shows it in use
with operation-specific application metrics. The same signals can then Verify
whether an operational action restored the service.
