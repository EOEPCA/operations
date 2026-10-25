# End-to-End Example: STAC Service Path

This page follows a STAC request through the gateway, authorisation proxy,
application, and database. It applies the Operations Building Block operating
model to a slow or failed HTTP request.

!!! info "Application metrics"

    The demo collects request and latency metrics from the proxy and backend.
    EOEPCA supported the
    [upstream transition in eoapi](https://github.com/EOEPCA/operations/issues/11#issuecomment-5083997066)
    to controlled, low-cardinality application metrics.
    `stac-auth-proxy` exposes Prometheus request and latency metrics by STAC
    operation.

    Compare these metrics with the gateway, workload, database, logs, and
    synthetic checks described below.

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

A failed or slow request can have several causes:
gateway routing, authorisation and filtering, the STAC application, or the
database.

## What is Observable Today

The current demo has enough signals to detect STAC degradation and identify
the likely layer.

### APISIX Route Metrics

APISIX metrics are scraped through the ServiceMonitor in `ingress-apisix`. The STAC rules in [`_rules/stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_rules/stac-alerts.yaml) use the stable APISIX route label `data-access_stac_stac-route`.

The rule file records burn rates for three views of latency:

- request latency, which is what the client-facing route experiences
- upstream latency, which points toward the proxied application path
- APISIX gateway latency, which points toward the gateway layer itself

The alerting rules currently fire on the request-latency GET and POST records. The upstream and gateway records support diagnosis after an alert fires.

### STAC Auth Proxy Metrics

The STAC auth proxy exposes request counters and latency histograms with
bounded labels for operation, method, and status. Prometheus collects them
through the
[`ServiceMonitor`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/servicemonitor-stac-auth-proxy.yaml).
This adds operation-specific evidence for search, collection, item, and other
STAC requests without using full URLs as metric labels. During Inspect and
Investigate, operators can compare request rate, errors, and latency by
operation with APISIX timings, workload health, database metrics, logs, and
synthetic checks.

### STAC SLO Dashboard

The STAC dashboard is deployed from
[`_dashboards/stac-slo.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/stac-slo.json).
It has separate sections for `stac-auth-proxy` and `eoapi-stac`, using each
service's request counters and latency histograms. Panels show request rates,
HTTP 5xx errors, p99 latency, and latency budget use. The target is 99% of GET
and POST requests within 500 ms over 30 days.

The dashboard also shows mean query latency for the shared eoapi database.
Proxy and backend request counts overlap; do not add them. Dashboard metrics
come from the services, while alerts still measure the public APISIX route.

### Database Metrics

The STAC rules include a database correlation record based on PostgreSQL exporter data:

```promql
avg(ccp_pg_stat_statements_total_mean_exec_time_ms{dbname="eoapi", role="eoapi"})
```

Use this record to compare a slow route with database query timing. It is a
mean for the shared database and does not identify a single STAC query.

### Logs

Grafana Alloy collects Kubernetes pod logs and forwards them to Loki.
STAC-related workloads in `data-access` can therefore be investigated through
logs alongside the application and surrounding metrics.

### Synthetic Checks

The data-access deployment includes a synthetic STAC checker:

- [`deployment-synthetic-api-check.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/deployment-synthetic-api-check.yaml)
- [`check-stac.sh`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/synthetic-api-check/check-stac.sh)

This checks whether the public STAC endpoint responds from outside the
application.

## Inspect, Investigate, and Act

The STAC setup supports Inspect and supplies metrics and logs for
Investigate. The [remediation-action library](remediation-actions.md) is still
in development. It will define how approved actions run and how recovery is
checked.

### Inspect

When a STAC alert fires:

1. A STAC burn-rate alert fires from [`stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_rules/stac-alerts.yaml).
2. Alertmanager forwards the alert to Keep through [`alertmanagerconfig-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/alertmanagerconfig-keep.yaml) and [`keep-alertmanager-relay.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/keep-alertmanager-relay.yaml).

Inspect should present these as one service incident: the affected STAC
operation, observed latency or errors, SLO impact, correlated gateway and
dependency signals, and links to the evidence.

### Investigate

The operator opens the STAC dashboard to compare proxy, backend, and database
signals. APISIX metrics show the public route, upstream, and gateway timings.

- If the issue looks application-related, the operator checks the `eoapi-stac`
  and `eoapi-stac-auth-proxy` pods, logs, and surrounding Kubernetes resource
  dashboards.
- If the issue looks database-related, the operator follows the PostgreSQL
  exporter signal and `pg_stat_statements`-derived timing.
- Recent deployment changes are useful evidence, but are not treated as the
  cause without supporting observations.

Compare the proxy and backend metrics to locate the delay or errors. If these
do not explain a slow public request, check APISIX timing, the database, and
logs for the same time range.

### Act and Verify

Once the operator has enough evidence, a remediation library should offer only
actions that are appropriate for the affected layer. STAC actions could
include:

- restart a stateless STAC workload
- scale a stateless workload within predefined limits
- enable a predefined rate-limiting profile

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
