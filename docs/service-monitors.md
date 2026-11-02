# ServiceMonitors and PodMonitors

A metrics endpoint does not become part of monitoring just because a Pod
exposes it. Prometheus must discover and scrape it. A `ServiceMonitor` selects
a Kubernetes Service and its metrics port; a `PodMonitor` selects Pods directly.

## Why ServiceMonitors Matter

A `ServiceMonitor` gives Prometheus Operator a managed target definition,
including the endpoint and scrape settings. Once selected by Prometheus, that
target supplies the metrics used by dashboards, alerts, and SLO calculations.

## Why the Operations BB Uses Them

The Operations BB keeps target definitions with service configuration rather
than maintaining a separate list of scrape endpoints. Teams can add a service's
metrics endpoint and monitor resource together, under the same review.

## What is Scraped Today

In `eoepca-demo`, ServiceMonitors are defined in:

- `operations` for the monitoring and alerting stack
- `ingress-apisix` for APISIX metrics
- `infra` for PostgreSQL exporter metrics
- `data-access` for operation-specific `stac-auth-proxy` metrics

Examples from the current deployment:

- [`eoepca/data-access/parts/servicemonitor-stac-auth-proxy.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/servicemonitor-stac-auth-proxy.yaml) scrapes STAC request and latency metrics
- [`infra/pgo/parts/servicemonitor-postgres-exporter.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/pgo/parts/servicemonitor-postgres-exporter.yaml) scrapes PostgreSQL exporter metrics
- [`infra/apisix/parts/servicemonitor-apisix.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/apisix/parts/servicemonitor-apisix.yaml) scrapes APISIX metrics from `/apisix/prometheus/metrics`
- [`app-keep-oauth2-proxy.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/app-keep-oauth2-proxy.yaml) enables a chart-managed ServiceMonitor for the Keep proxy
- [`app-loki-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/app-loki-stack.yaml) enables Loki monitoring integration

The live `operations` namespace currently has ServiceMonitors for:

- Alloy log collection
- Crossplane core controller metrics
- Keep OAuth2 proxy metrics
- kube-prometheus-stack components, including Alertmanager, Grafana, Prometheus, kube-state-metrics, kubelet, and node exporter
- Loki-related chart-managed metrics

## Crossplane Metrics

The `crossplane-core` ServiceMonitor and `crossplane-providers` PodMonitor in
`operations` select controllers in `infra`. They collect client request
counters for the Kubernetes API and etcd dashboard and alert snapshots.

These client counters help identify controller traffic. The API's resource
breakdown shows which objects were accessed, but cannot identify the caller
by itself. The monitor definitions are in
[`crossplane-metrics.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/crossplane-metrics.yaml).

## Workspace PodMonitor

The `workspace-managed-data` PodMonitor in `operations` selects `package-r`
pods across workspace namespaces and scrapes `/metrics`. This keeps managed
data collection in step with workspace Pod creation and removal. The workspace
provider also creates the network policy that allows Prometheus to reach the
metrics port.

The [workspace scenario](workspace-scenario.md#how-metrics-reach-the-dashboard)
explains how these metrics and the workspace recording rules supply the
dashboard. The PodMonitor definition is in
[`workspace-managed-data.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/workspace-managed-data.yaml).

The demo also has the `infra/pods-by-annotation` PodMonitor.

## Concrete Example: Database Pod

The PostgreSQL database Pod in `infra` shows how the pieces connect:

- the Crunchy `PostgresCluster` enables the `pgmonitor` exporter
- the running database pod includes an `exporter` container with a named `exporter` port on `9187`
- the `default-pods` headless Service selects pods with `postgres-operator.crunchydata.com/cluster: default`
- the `postgres-exporter` `ServiceMonitor` selects that Service and scrapes the `exporter` target port every `30s`

Relevant files:

- [`postgrescluster.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/pgo/parts/postgrescluster.yaml) enables the PostgreSQL monitoring exporter
- [`servicemonitor-postgres-exporter.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/pgo/parts/servicemonitor-postgres-exporter.yaml) defines the scrape target

This supplies PostgreSQL exporter and `pg_stat_statements` measurements for
comparison with STAC request latency.

## Metric Labels

Metric labels need a controlled set of values. A label containing every full
request URL would create too many separate time series. APISIX instead uses
labels such as route and HTTP method, configured in
[`infra/apisix/parts/values/apisix-values.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/apisix/parts/values/apisix-values.yaml).

The STAC auth proxy uses operation, method, and status labels to distinguish
searches and other STAC operations without full request URLs. The
[STAC end-to-end example](stac-scenario.md#compare-events-with-the-charts) shows how
these metrics complement gateway, workload, database, log, and synthetic
signals.
