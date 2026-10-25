# ServiceMonitors and PodMonitors

A `ServiceMonitor` tells Prometheus Operator which Kubernetes Service exposes
a metrics endpoint. A `PodMonitor` selects pods directly.

## Why ServiceMonitors Matter

A `ServiceMonitor` turns a metrics endpoint into a managed scrape target. Once it exists, Prometheus can:

- scrape the endpoint automatically
- label the target consistently
- build dashboards on those metrics
- alert on them
- include them in SLO calculations

Without this connection, metrics may exist but Prometheus cannot use them.

## Why the Operations BB Uses Them

The Operations BB uses `ServiceMonitor` resources instead of hand-written
scrape settings. This is easier to manage across teams and services. A
component with a stable metrics endpoint and its own `ServiceMonitor` is easier
to add to the platform baseline.

## What is Scraped Today

The eoepca-demo cluster currently has ServiceMonitors in:

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
- Keep OAuth2 proxy metrics
- kube-prometheus-stack components, including Alertmanager, Grafana, Prometheus, kube-state-metrics, kubelet, and node exporter
- Loki-related chart-managed metrics

## Workspace PodMonitor

The `workspace-managed-data` PodMonitor in `operations` selects `package-r`
pods across workspace namespaces and scrapes `/metrics`. It was added to
collect managed data activity as workspace pods are created and removed.
The workspace provider creates a network policy that permits Prometheus to
reach the metrics port.

The [workspace scenario](workspace-scenario.md#how-metrics-reach-the-dashboard)
explains how these metrics and the workspace recording rules supply the
dashboard. The PodMonitor definition is in
[`workspace-managed-data.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/workspace-managed-data.yaml).

The demo also has the `infra/pods-by-annotation` PodMonitor.

## Concrete Example: Database Pod

The PostgreSQL database pod in `infra` uses the following setup:

- the Crunchy `PostgresCluster` enables the `pgmonitor` exporter
- the running database pod includes an `exporter` container with a named `exporter` port on `9187`
- the `default-pods` headless Service selects pods with `postgres-operator.crunchydata.com/cluster: default`
- the `postgres-exporter` `ServiceMonitor` selects that Service and scrapes the `exporter` target port every `30s`

Relevant files:

- [`postgrescluster.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/pgo/parts/postgrescluster.yaml) enables the PostgreSQL monitoring exporter
- [`servicemonitor-postgres-exporter.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/pgo/parts/servicemonitor-postgres-exporter.yaml) defines the scrape target

Operators can use these metrics to compare STAC latency with database query
timing from PostgreSQL exporter and `pg_stat_statements`.

## Metric Labels

A scrape target is only useful when its metric labels are stable. APISIX route
metrics use controlled labels such as route and HTTP method, configured in
[`infra/apisix/parts/values/apisix-values.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/apisix/parts/values/apisix-values.yaml).

The STAC auth proxy follows the same pattern at application level. Its request
and latency metrics use bounded operation, method, and status labels, allowing
operators to distinguish searches and other STAC operations without using full
URLs as metric labels. The
[STAC end-to-end example](stac-scenario.md#stac-auth-proxy-metrics) shows how
these metrics complement gateway, workload, database, log, and synthetic
signals.
