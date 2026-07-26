# End-to-End Example: STAC Service Path

This page applies the Operations Building Block operating model end to end to
a synchronous HTTP request/response path. The STAC example connects a
user-visible request to the gateway, authorisation proxy, application, and
database layers that must cooperate to serve it.

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

This example is useful because one symptom can come from several layers:
gateway routing, authorisation and filtering, the STAC application, or the
database.

## What is Observable Today

The current demo has enough signals to detect STAC degradation and identify
the likely layer.

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

This checks whether the public STAC endpoint responds from outside the
application.

## Inspect, Investigate, and Act

The current STAC setup already supports much of Inspect and provides evidence
for a human-led Investigate. The
[remediation-action library](remediation-actions.md) currently being
established is the next step needed to complete Act.

### Inspect

A practical STAC incident flow is:

1. A STAC burn-rate alert fires from [`stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_rules/stac-alerts.yaml).
2. Alertmanager forwards the alert to Keep through [`alertmanagerconfig-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/alertmanagerconfig-keep.yaml) and [`keep-alertmanager-relay.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/keep-alertmanager-relay.yaml).

Inspect should present these as one service incident: the affected STAC
operation, observed latency or errors, SLO impact, correlated gateway and
dependency signals, and links to the evidence.

### Investigate

The operator opens the STAC SLO dashboard to compare request, upstream,
gateway, and database signals.

- If the issue looks application-related, the operator checks the `eoapi-stac`
  and `eoapi-stac-auth-proxy` pods, logs, and surrounding Kubernetes resource
  dashboards.
- If the issue looks database-related, the operator follows the PostgreSQL
  exporter signal and `pg_stat_statements`-derived timing.
- Recent Flux and deployment changes are useful evidence, but are not treated
  as the cause without supporting observations.

The current setup shows when the STAC path is at risk and helps separate the
main layers. It gives less detail about behaviour inside the application.

### Act and Verify

Once the operator has enough evidence, a remediation library should offer only
actions that are appropriate for the affected layer. STAC actions could
include:

- reconcile the relevant Flux source or release
- restart a stateless STAC workload
- scale a stateless workload within predefined limits
- enable a predefined rate-limiting profile
- open a GitOps change for a durable correction

These actions are proposals and are not yet part of the current demo.
Production changes require human approval.

Verify then repeats the important checks:

- did the workload reach a healthy state?
- does the public STAC request succeed?
- did latency or the error rate return to the expected range?
- did recovery remain stable rather than immediately regress?

If verification fails, the operator returns to Investigate or escalates instead
of treating command completion as recovery.

### Learn

After recovery, the team records what was missing or misleading, whether the
chosen action helped, and what would make the next incident easier. This may
lead to better STAC metrics, SLOs, dashboards, investigation guidance, action
preconditions, or verification checks.

## Main Gap: Application-Specific Metrics

The original STAC scenario showed a strong need for application-specific metrics. Today there are no `ServiceMonitor` objects in the `data-access` namespace, and in-cluster checks against both application endpoints return `404` for `/metrics`:

```text
http://eoapi-stac.data-access.svc.cluster.local:8080/metrics
http://eoapi-stac-auth-proxy.data-access.svc.cluster.local:8080/metrics
```

Operators must therefore infer STAC behaviour from surrounding layers:

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

The next step is to add metrics directly to the STAC application path:

- expose a Prometheus-compatible metrics endpoint in `eoapi-stac`
- expose proxy-specific metrics from `eoapi-stac-auth-proxy` if authorisation and filtering behaviour needs to be operated separately
- add stable labels and a `ServiceMonitor` for each metrics endpoint
- extend the STAC dashboard and alerts with native application metrics

With that in place, the Operations BB can move from "the STAC path is slow"
toward "this specific layer of the STAC application path is contributing to the
degradation".

The next step is to connect that evidence to the remediation-action library.
The operator must be able to trace the evidence, decision, approval, action,
verification, and outcome.
