# Monitoring Stack

This page describes the operations stack deployed for the EOEPCA demo environment and how the pieces fit together. The deployment sources live in [`eoepca-plus/argocd/operations`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/operations).

## Deployed Components

The `operations` namespace contains the operator-facing monitoring and alerting stack:

| Component | Purpose | Deployment source |
| --- | --- | --- |
| Prometheus | Scrapes metrics and evaluates recording and alerting rules | [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-kube-prometheus-stack.yaml) |
| Grafana | Dashboards and metric/log exploration | [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-kube-prometheus-stack.yaml) |
| Alertmanager | Alert grouping, routing, and deduplication | [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-kube-prometheus-stack.yaml) |
| Loki | Log storage and querying | [`app-loki-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-loki-stack.yaml) |
| Grafana Alloy | Pod log discovery and forwarding to Loki | [`app-alloy-logs.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-alloy-logs.yaml) and [`config.alloy`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/alloy/config.alloy) |
| Keep | Alert enrichment and triage UI | [`app-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/app-keep.yaml) |
| Keep OAuth2 Proxy | Access control and proxying for Keep | [`app-keep-oauth2-proxy.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/app-keep-oauth2-proxy.yaml) |
| Keep relay | Internal bridge from Alertmanager webhooks to Keep | [`keep-alertmanager-relay.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/keep-alertmanager-relay.yaml) |

## What Each Component Does

### Prometheus

Prometheus is the core metrics engine. In the EOEPCA demo it:

- scrapes platform and application targets
- evaluates baseline and STAC-specific rules
- stores time series for dashboards and alerting

The retention and persistence settings are defined in [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-kube-prometheus-stack.yaml).

### Grafana

Grafana is the operator-facing dashboard layer. In the demo it is configured with OIDC login and a Loki data source alongside Prometheus.

That combination matters because operators usually need both:

- Prometheus for trends, ratios, and alert queries
- Loki for contextual log exploration

### Alertmanager

Alertmanager handles the mechanics of alert delivery:

- grouping
- routing
- deduplication

On its own it is a transport layer for alerts. The Operations BB treats it as necessary, but not sufficient, for operator workflows.

### Loki

Loki stores logs collected from Kubernetes pods. The current deployment uses:

- a single-binary Loki setup
- object storage on S3-compatible backend
- a chart-managed ServiceMonitor for Loki metrics
- Loki canaries to exercise the log pipeline

See [`app-loki-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-loki-stack.yaml).

### Grafana Alloy

Alloy runs as a DaemonSet and discovers pods on each node. The current configuration:

- collects pod logs from Kubernetes
- attaches useful labels such as namespace, pod, container, and app
- forwards logs to Loki

The pipeline is defined in [`monitoring/alloy/config.alloy`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/alloy/config.alloy).

### Keep

Keep is used as an alert enrichment and triage layer. Instead of forcing operators to jump immediately from a firing alert into raw PromQL or cluster state, it provides a place to correlate the event with additional context.

That is especially helpful when several systems can contribute to one visible symptom, which is common on EO platforms.

## Public Operator Endpoints

The demo currently exposes:

- Grafana at <https://monitoring.develop.eoepca.org>, requiring a `monitoring:grafana-admin`, `monitoring:grafana-editor`, or `monitoring:grafana-viewer` role in Keycloak
- Keep at <https://alerting.develop.eoepca.org>, requiring an `alerting:keep-admin` or `alerting:keep-noc` role in Keycloak

These endpoints are exposed through APISIX routes:

- [`monitoring/routes.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/routes.yaml)
- [`alerting/routes.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/routes.yaml)

## What This Gives Operators

With the current stack, operators can already:

- inspect cluster and workload resource behaviour in Grafana
- query logs from EO platform workloads through Loki
- evaluate Prometheus rules and Alertmanager routing
- use Keep to enrich and triage alert events

The remaining challenge is not the absence of a monitoring stack. It is making sure individual EO platform services expose enough native signals for that stack to scrape and interpret well.
