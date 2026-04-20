# ServiceMonitors

`ServiceMonitor` is the main way the Prometheus Operator learns what to scrape in Kubernetes.

## Why ServiceMonitors Matter

A `ServiceMonitor` turns a metrics endpoint into a managed scrape target. Once it exists, Prometheus can:

- scrape the endpoint automatically
- label the target consistently
- build dashboards on those metrics
- alert on them
- include them in SLO calculations

Without that wiring, metrics may exist but remain operationally invisible.

## Why The Operations BB Uses Them

The Operations BB prefers declarative discovery over ad hoc scrape configuration because it scales better across teams and services. A component that exposes a stable metrics endpoint and ships its own `ServiceMonitor` is much easier to integrate into a common platform baseline.

## What Is Scraped Today

The eoepca-demo cluster includes ServiceMonitors in:

- `operations` for the monitoring and alerting stack
- `ingress-apisix` for APISIX metrics
- `infra` for PostgreSQL exporter metrics

Examples from the current deployment:

- [`infra/pgo/parts/servicemonitor-postgres-exporter.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/2473e90aad88ef76de8cbd947b124e404fb1281b/argocd/infra/pgo/parts/servicemonitor-postgres-exporter.yaml#L1-L14) scrapes PostgreSQL exporter metrics every `30s`
- [`app-keep-oauth2-proxy.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/app-keep-oauth2-proxy.yaml) enables a chart-managed ServiceMonitor for the Keep proxy
- [`app-loki-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/monitoring/app-loki-stack.yaml) enables Loki monitoring integration

The live `operations` namespace currently has ServiceMonitors for:

- Alloy log collection
- Keep OAuth2 proxy metrics
- kube-prometheus-stack components, including Alertmanager, Grafana, Prometheus, kube-state-metrics, kubelet, and node exporter
- Loki

There are no `PodMonitor` objects in `operations`. The only live `PodMonitor` observed in the demo cluster was `infra/pods-by-annotation`.

## Concrete Example: Database Pod

The PostgreSQL database pod in `infra` is a good example of the full pattern:

- the Crunchy `PostgresCluster` enables the `pgmonitor` exporter
- the running database pod includes an `exporter` container with a named `exporter` port on `9187`
- the `default-pods` headless Service selects pods with `postgres-operator.crunchydata.com/cluster: default`
- the `postgres-exporter` `ServiceMonitor` selects that Service and scrapes the `exporter` target port every `30s`

Relevant files:

- [`postgrescluster.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/2473e90aad88ef76de8cbd947b124e404fb1281b/argocd/infra/pgo/parts/postgrescluster.yaml#L42-L44) enables the PostgreSQL monitoring exporter
- [`servicemonitor-postgres-exporter.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/2473e90aad88ef76de8cbd947b124e404fb1281b/argocd/infra/pgo/parts/servicemonitor-postgres-exporter.yaml#L9-L14) defines the scrape target

Because that pattern is in place, operators can correlate STAC symptoms with database-side signals such as PostgreSQL exporter metrics and `pg_stat_statements`-derived query timing.

## What A Metrics Endpoint Enables

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

Those signals are still useful, but they do not replace native application metrics. They usually tell operators that a path is slow or failing, but not enough about why the service itself behaved that way.
