# ServiceMonitors

A `ServiceMonitor` is the main way the Prometheus Operator learns what to scrape in Kubernetes.

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

There are no `data-access` ServiceMonitors for `eoapi` workloads yet.

Examples from the current deployment:

- [`infra/pgo/parts/servicemonitor-postgres-exporter.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/pgo/parts/servicemonitor-postgres-exporter.yaml) scrapes PostgreSQL exporter metrics
- [`infra/apisix/parts/servicemonitor-apisix.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/apisix/parts/servicemonitor-apisix.yaml) scrapes APISIX metrics from `/apisix/prometheus/metrics`
- [`app-keep-oauth2-proxy.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/app-keep-oauth2-proxy.yaml) enables a chart-managed ServiceMonitor for the Keep proxy
- [`app-loki-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-loki-stack.yaml) enables Loki monitoring integration

The live `operations` namespace currently has ServiceMonitors for:

- Alloy log collection
- Keep OAuth2 proxy metrics
- kube-prometheus-stack components, including Alertmanager, Grafana, Prometheus, kube-state-metrics, kubelet, and node exporter
- Loki-related chart-managed metrics

There are no `PodMonitor` objects in `operations`. The only live `PodMonitor` observed in the demo cluster is `infra/pods-by-annotation`.

## Concrete Example: Database Pod

The PostgreSQL database pod in `infra` is a good example of the full pattern:

- the Crunchy `PostgresCluster` enables the `pgmonitor` exporter
- the running database pod includes an `exporter` container with a named `exporter` port on `9187`
- the `default-pods` headless Service selects pods with `postgres-operator.crunchydata.com/cluster: default`
- the `postgres-exporter` `ServiceMonitor` selects that Service and scrapes the `exporter` target port every `30s`

Relevant files:

- [`postgrescluster.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/pgo/parts/postgrescluster.yaml) enables the PostgreSQL monitoring exporter
- [`servicemonitor-postgres-exporter.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/pgo/parts/servicemonitor-postgres-exporter.yaml) defines the scrape target

Because that pattern is in place, operators can correlate STAC symptoms with database-side signals such as PostgreSQL exporter metrics and `pg_stat_statements`-derived query timing.

## Metric Quality Matters Too

A scrape target is only useful when its metric labels are stable. For example,
APISIX route metrics use controlled labels such as route and HTTP method. The
labels are configured in
[`infra/apisix/parts/values/apisix-values.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/apisix/parts/values/apisix-values.yaml).

The STAC scenario originally showed a strong need for application-specific metrics. Trying to derive that detail at the gateway by adding a full URL label creates high cardinality: every distinct path, query, or identifier becomes another time series. That increases Prometheus memory, storage, and query cost, especially when values churn quickly.

That is why the better long-term pattern is to expose semantic, low-cardinality metrics from the application itself, then scrape them with a `ServiceMonitor`.

## What a Metrics Endpoint Enables

When a service exposes a useful Prometheus endpoint and is scraped through a `ServiceMonitor`, operators can:

- measure request rate, errors, and latency directly from the service
- create dashboards that reflect application internals rather than only infrastructure symptoms
- build SLOs closer to the service boundary
- correlate platform symptoms with app-level behaviour
- distinguish gateway issues from application issues more confidently

## What Happens Without It

When a service does not expose native metrics or does not ship a `ServiceMonitor`, operators fall back to indirect signals such as:

- gateway timings
- pod health and resource usage
- logs
- synthetic checks

Those signals are still useful, but they do not replace native application
metrics. They can show that a path is slow or failing, but often not why.
