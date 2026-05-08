# STAC Scenario

The STAC path is the clearest EO platform example for the Operations Building Block. If catalogue search or item access is slow, users and downstream building blocks feel it quickly.

## Request Path

In the demo environment, the public STAC endpoint is:

- <https://eoapi.develop.eoepca.org/stac>

The public route is defined in [`argocd/eoepca/data-access/parts/route.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/route.yaml). For the main STAC path, traffic flows through:

```text
client
  -> APISIX route data-access_stac_stac-route
  -> eoapi-stac-auth-proxy
  -> eoapi-stac
  -> PostgreSQL / pgSTAC
```

This is a useful scenario because a single symptom can come from several layers: gateway routing, authorisation and filtering, the STAC application, or the database.

## What is Observable Today

The current demo has enough signals to detect STAC degradation and narrow down the likely layer.

### APISIX Route Metrics

APISIX metrics are scraped through the ServiceMonitor in `ingress-apisix`. The STAC rules in [`_rules/stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_rules/stac-alerts.yaml) use the stable APISIX route label `data-access_stac_stac-route`.

The rule file records burn rates for three views of latency:

- request latency, which is what the client-facing route experiences
- upstream latency, which points toward the proxied application path
- APISIX gateway latency, which points toward the gateway layer itself

The alerting rules currently fire on the request-latency GET and POST records. The upstream and gateway records support diagnosis after an alert fires.

### STAC SLO Dashboard

The curated STAC dashboard is deployed from [`_dashboards/stac-slo.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_dashboards/stac-slo.json). It is built from the same recording rules and shows the route, GET and POST burn rates, gateway and upstream views, and database latency.

### Database Metrics

The STAC rules include a database correlation record based on PostgreSQL exporter data:

```promql
avg(ccp_pg_stat_statements_total_mean_exec_time_ms{dbname="eoapi", role="eoapi"})
```

This does not explain every STAC problem, but it gives operators a database-side signal when the route is slow.

### Logs

Grafana Alloy collects Kubernetes pod logs and forwards them to Loki. STAC-related workloads in `data-access` can therefore be investigated through logs even when the application does not expose native Prometheus metrics.

### Synthetic Checks

The data-access deployment includes a synthetic STAC checker:

- [`deployment-synthetic-api-check.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/deployment-synthetic-api-check.yaml)
- [`check-stac.sh`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/synthetic-api-check/check-stac.sh)

This gives operators a black-box view of whether the public STAC endpoint responds from the outside of the application.

## How Operators Use These Signals

A practical STAC incident flow is:

1. A STAC burn-rate alert fires from [`stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_rules/stac-alerts.yaml).
2. Alertmanager forwards the alert to Keep through [`alertmanagerconfig-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/alertmanagerconfig-keep.yaml) and [`keep-alertmanager-relay.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/keep-alertmanager-relay.yaml).
3. The operator opens the STAC SLO dashboard to compare request, upstream, gateway, and database signals.
4. If the issue looks application-related, the operator checks the `eoapi-stac` and `eoapi-stac-auth-proxy` pods, logs, and surrounding Kubernetes resource dashboards.
5. If the issue looks database-related, the operator follows the PostgreSQL exporter signal and `pg_stat_statements`-derived timing.

The current setup is good at showing that the STAC path is at risk and at separating some broad layers. It is weaker at explaining application-internal behaviour.

## Main Gap: Application-Specific Metrics

The original STAC scenario showed a strong need for application-specific metrics. Today there are no `ServiceMonitor` objects in the `data-access` namespace, and in-cluster checks against both application endpoints return `404` for `/metrics`:

```text
http://eoapi-stac.data-access.svc.cluster.local:8080/metrics
http://eoapi-stac-auth-proxy.data-access.svc.cluster.local:8080/metrics
```

That means operators mainly infer STAC behaviour from surrounding layers:

- APISIX request and upstream timings
- Kubernetes pod health, restarts, and resource usage
- database exporter metrics
- logs
- synthetic checks

APISIX can separate GET and POST traffic, but it cannot see the STAC operation semantics inside those requests. From the outside, modification requests and search requests are both POST requests. GET requests have a similar problem: a single item lookup, a large collection or item listing, and a simple HTTP probe can all look like GET traffic on the same public route.

Those signals are valuable, but they do not expose application decisions. Useful native metrics could be added to `eoapi-stac`, `eoapi-stac-auth-proxy`, or both. Examples include request counters and latency histograms by route template, method, operation, status class, and outcome; auth and filtering decisions in the proxy; cache behaviour; and database-pool or worker saturation inside the STAC application.

## Why Not Use Full URLs From APISIX?

One early idea was to scrape or label full URLs at APISIX to recover more STAC detail from gateway metrics. That quickly runs into high-cardinality problems.

High cardinality means a metric has too many distinct label value combinations. In Prometheus, each unique combination becomes a separate time series. If a label contains full URLs, item IDs, collection IDs, user IDs, or query strings, the number of time series can grow quickly and churn constantly. That increases memory use, storage cost, and query latency, and it can make alerts less reliable.

The current APISIX configuration in [`infra/apisix/parts/values/apisix-values.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/apisix/parts/values/apisix-values.yaml) keeps the useful low-cardinality dimensions, such as route and HTTP method. For deeper STAC insight, the safer pattern is application-native metrics with controlled labels.

## Better End State

The next maturity step is to instrument the STAC application path directly:

- expose a Prometheus-compatible metrics endpoint in `eoapi-stac`
- expose proxy-specific metrics from `eoapi-stac-auth-proxy` if authorisation and filtering behaviour needs to be operated separately
- add stable labels and a `ServiceMonitor` for each metrics endpoint
- extend the STAC dashboard and alerts with native application metrics

With that in place, the Operations BB can move from "the STAC path is slow" toward "this specific layer of the STAC application path is responsible".

Remediation can then be introduced carefully. In the STAC case, that might remain a manual runbook, become a script for common mitigations, follow a decision tree, or use an agent-assisted workflow. The key requirement is the same in every case: the operator must be able to trace the recommendation, approval, action, and outcome.
