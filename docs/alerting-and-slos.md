# Alerting and SLOs

The demo sends Prometheus alerts through Alertmanager to Keep. Platform rules
cover conditions such as disk pressure and missing targets; SLO alerts cover
service quality, such as slow requests.

## Prometheus, Alertmanager, and Keep

Prometheus evaluates rules against collected metrics. When an alert fires,
Alertmanager groups it with related alerts and sends a notification to Keep.
Routing, silences, and inhibition in Alertmanager determine which notifications
reach Keep.

## Why Use Keep?

A slow Kubernetes API can delay both workspace creation and processing jobs.
Reviewing their alerts together in Keep helps an operator see a possible
shared failure. Assignment and acknowledgement show who has taken the work,
so another operator can see its status or take it over.

For API/etcd and STAC alerts, Keep saves measurements when the notification
arrives. The Grafana link opens the surrounding period for comparison with
metrics and logs. The [control-plane example](control-plane-scenario.md) shows
this for API load and etcd latency, including the saved snapshot and chart markers.

## Baseline Alerts

The baseline rules live in [`_rules/baseline-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_rules/baseline-alerts.yaml). They cover platform-level concerns such as:

- missing Prometheus targets
- Prometheus ingest problems
- rule evaluation failures
- node pressure and disk exhaustion
- crash looping pods
- replica mismatches
- persistent volume exhaustion

Service-specific rules add checks for request quality and application
behaviour to these platform alerts.

## Recording Rules and Alert Rules

A recording rule saves a calculated metric for reuse. For example, the
control-plane rules record API request rate and an earlier hourly baseline.
Grafana can show both on the same chart.

An alert rule checks whether a condition needs attention. The API traffic
alert requires traffic to exceed both a minimum rate and twice the earlier
baseline for a sustained period. Separate rules check API errors,
availability, and etcd latency. A traffic spike alone does not show service
impact or its cause.

Chart thresholds need to follow rule changes so that an operator can still
see why the alert fired.

## Alert Context

Enrichment adds measurements and investigation links to an alert. Keep saves
them with the alert, so the operator can use the same starting information
even after the condition has passed.

## Keep for Alert Enrichment

The Keep connection and workflows are defined in:

- [`alertmanagerconfig-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/alertmanagerconfig-keep.yaml)
- [`keep-alertmanager-relay.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/keep-alertmanager-relay.yaml)
- [`app-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/app-keep.yaml)

Alerts arrive with the labels and annotations from their rules. Three native
YAML workflows use these settings to add investigation links, save an optional
metrics snapshot, and write firing/resolved events. The API/etcd and STAC SLO
rules specify which dashboard and measurements to use.

Prometheus records a small set of context measurements every 30 seconds as
`operations:alert_context`. Keep reads the selected dashboard's measurements
once for a new firing occurrence. It saves the result in `metrics_context`
and the receipt time in `metrics_context_at`. Repeated notifications,
acknowledgement, and resolution retain the snapshot. If the query fails after
one retry, the context stays `unavailable`.

The [control-plane example](control-plane-scenario.md) explains the API/etcd
signals. The [STAC example](stac-scenario.md#review-the-alert-in-keep) follows the
gateway, auth proxy, API, and database.

The `keep` `AlertmanagerConfig` sends firing and resolved notifications to:

```
http://keep-alertmanager-relay.operations.svc.cluster.local/alerts/event/prometheus
```

Earlier routes send selected noisy workload alerts to a `null` receiver.
The remaining alerts use the default Keep receiver.

## Keep Events in Grafana

The API/etcd and STAC dashboards place firing and resolved notifications next
to the measurements as chart markers. They reach Grafana through the log
collection path:

1. When a covered Prometheus alert fires or resolves, the native Keep workflow
   writes a structured event to the backend Pod log.
2. Alloy collects that log and sends it to Loki with the cluster, namespace,
   and container labels.
3. Grafana's **Keep events** annotation query selects those entries from Loki
   for the dashboard's time range. Each returned entry becomes a chart marker.

Each log entry contains `event_type: keep_alert_event`, the dashboard ID,
alert name, status, and Keep link. Grafana selects the entries for its own
`${__dashboard.uid}`. It reads chart measurements from Prometheus and markers
from Loki. The saved snapshot stays with the alert in Keep.

Only firing and resolved events produce markers; assignment and acknowledgement
remain in Keep. Native change filters skip ordinary repeated notifications,
but do not guarantee exactly one marker per event. The markers therefore give
an investigation timeline rather than a complete audit of response actions.

See [Keep Events](dashboards-and-usage.md#keep-events) for the toggle, marker
contents, and checks for missing events. The annotation query is maintained in
[`kubernetes-api-and-etcd.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/kubernetes-api-and-etcd.json).

### Shared Configuration

Each alert selects a context profile with `dashboard: stac` or
`dashboard: control-plane`. Its `dashboard_url` points to the Grafana dashboard.
Keep uses the profile to query measurements and reads the dashboard ID from
the URL for chart markers. Changing an ID in the dashboard JSON can break
existing bookmarks and alert links.

Add `context_snapshot: "true"` to save measurements when a new occurrence
fires. To cover another service, define its context recording rules and set
the profile and dashboard URL on its alerts. The Keep workflows stay the same.

Public URLs are set in the dashboard links, alert rules, and Keep configuration.
The [deployment reference](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/README.md)
explains the settings and checks. The workflows use built-in Keep providers.
MySQL retains alert state across backend Pod replacements; the demo has no
database backup or restore guarantee.

## SLOs

An SLO sets a service target, such as a maximum share of slow requests. It
measures the result that users receive. CPU and memory metrics can help
explain a failure or delay, but do not measure that result directly.

## Burn Rate

Burn rate shows how fast a service uses its error budget. A high value means
that it will miss its target sooner if the problem continues.

For example:

```
stac_get_latency_500ms_burn_rate_1h > 14.4
```

The STAC latency target is 99% of requests within 500 ms. This leaves a 1%
error budget for slow requests.

The burn-rate rule divides the observed slow-request ratio by the allowed 1%
budget. A burn rate of `1` means the service is using the budget at the allowed
rate. A burn rate of `14.4` means it is using the budget 14.4 times faster.

For a 30-day SLO window, sustaining a `14.4` burn rate would use the whole 30-day error budget in about two days:

```
30 days / 14.4 = 2.08 days
```

The one-hour record detects a rapid increase in slow requests. The six-hour
record helps detect a problem that persists for longer.

The deployed STAC rules are defined in [`stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_rules/stac-alerts.yaml). The live `stac-alerts` PrometheusRule contains:

- one-hour and six-hour request-latency burn-rate records for STAC GET and POST requests
- matching upstream application and APISIX gateway burn-rate records for diagnosis
- a PostgreSQL mean execution latency record for database correlation
- warning alerts where the six-hour request burn rate is above `6` for `30m`, unless the one-hour critical condition is already true
- critical alerts where the one-hour request burn rate is above `14.4`

Alerts fire on the request-level GET and POST records. The upstream, gateway,
and database records help locate the layer that contributes to the delay.

The [STAC dashboard](dashboards-and-usage.md#stac) shows the public-route
records above separate proxy and backend metrics. Their values can differ
because they measure different points in the request path.

## From Alerting to Remediation

The investigation determines which response is suitable. The planned
[remediation-action library](remediation-actions.md) will define approved
procedures with limits and recovery checks; it is still in development.

A change is complete only after the service request works again and the
relevant measurements confirm recovery. If the problem remains, the operator
returns to investigation. The action and result belong in the incident record,
along with any alert or dashboard changes needed for the next incident.
