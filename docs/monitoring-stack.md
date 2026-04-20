# Monitoring Stack

This page describes what is deployed today for the EOEPCA demo operations stack and how the pieces fit together.

## Deployed Components

The `operations` namespace in the eoepca-demo cluster includes these main components:

| Component | Purpose | Deployment source |
| --- | --- | --- |
| Prometheus | Scrapes metrics and evaluates rules | [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/2473e90aad88ef76de8cbd947b124e404fb1281b/argocd/operations/monitoring/app-kube-prometheus-stack.yaml#L71-L82) |
| Grafana | Dashboards and exploration UI | [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/2473e90aad88ef76de8cbd947b124e404fb1281b/argocd/operations/monitoring/app-kube-prometheus-stack.yaml#L17-L60) |
| Alertmanager | Alert routing and grouping | [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/2473e90aad88ef76de8cbd947b124e404fb1281b/argocd/operations/monitoring/app-kube-prometheus-stack.yaml#L65-L69) |
| Loki | Log storage and querying | [`app-loki-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-loki-stack.yaml) |
| Grafana Alloy | Log collection from Kubernetes pods | [`app-alloy-logs.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-alloy-logs.yaml) |
| Loki Canary | Synthetic log path checks for Loki ingestion and querying | [`app-loki-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-loki-stack.yaml) |
| Keep | Alert enrichment and operator workflow UI | [`app-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/app-keep.yaml) |
| Keep OAuth2 Proxy | Access control and proxying for Keep | [`app-keep-oauth2-proxy.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/app-keep-oauth2-proxy.yaml) |

## Current Cluster Snapshot

The live `operations` namespace is healthy at workload level:

- Prometheus `v3.11.1` and Alertmanager `v0.31.1` are both reconciled and available
- Grafana `12.4.2`, kube-state-metrics `v2.18.0`, and Prometheus Operator `v0.90.1` are running
- Alloy runs as a four-pod DaemonSet using `grafana/alloy:v1.12.1`
- Loki runs as a single-binary StatefulSet using `grafana/loki:3.6.7`, with four Loki canary pods
- Keep backend and frontend run on `0.49.0`
- Prometheus has a bound `50Gi` PVC and Keep has a bound `5Gi` PVC

The owning Argo CD applications for the operations stack were also synced and healthy: `alloy-logs`, `keep`, `keep-oauth2-proxy`, `kube-prometheus-stack`, and `loki-stack`.

## What Each Component Does

### Prometheus

Prometheus is the core metrics engine. In the EOEPCA demo it:

- scrapes platform and application targets
- evaluates baseline and STAC-specific rules
- stores time series for dashboards and alerting

The current deployment keeps 30 days of retention and persists data on a PVC. See the values in [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/2473e90aad88ef76de8cbd947b124e404fb1281b/argocd/operations/monitoring/app-kube-prometheus-stack.yaml#L73-L82).

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
- a 168 hour retention period
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

These endpoints are exposed through APISIX routes, not OpenShift `Route` objects. The routing manifests are here:

- [`monitoring/routes.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/routes.yaml)
- [`alerting/routes.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/routes.yaml)

## What This Gives Operators

With the current stack, operators can already:

- inspect cluster and workload resource behaviour in Grafana
- query logs from EO platform workloads through Loki
- evaluate Prometheus rules and Alertmanager routing
- use Keep to enrich and triage alert events

The remaining challenge is not the absence of a monitoring stack. It is making sure individual EO platform services expose enough native signals for that stack to scrape and interpret well.
