# Alerting and SLOs

Alerts and SLOs support **Inspect** in the
[operating model](operating-model.md). They show which service conditions need
operator attention and help judge the effect on users.

## Prometheus, Alertmanager, and Keep

In the EOEPCA demo:

1. Prometheus scrapes metrics and evaluates recording and alerting rules.
2. Alertmanager groups and routes alerts.
3. Keep enriches selected alerts so operators can triage them with more context.

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

## Alert Context

Alertmanager groups and routes alerts. During investigation, operators also
need to know:

- which user-facing service is actually at risk?
- what related signals should I inspect next?
- is this a gateway problem, an application problem, or a database problem?
- what mitigation step is likely to help?

Keep adds related information to help answer these questions.

## Keep for Alert Enrichment

In the EOEPCA demo, Keep is integrated through:

- [`alertmanagerconfig-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/alertmanagerconfig-keep.yaml)
- [`keep-alertmanager-relay.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/keep-alertmanager-relay.yaml)
- [`app-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/app-keep.yaml)

Selected alerts arrive in Keep with related service information. Use this
information to choose which dashboard, log, or dependency to check next.

In the live cluster, the `keep` `AlertmanagerConfig` sends resolved and firing alerts to:

```
http://keep-alertmanager-relay.operations.svc.cluster.local/alerts/event/prometheus
```

Selected noisy workload alerts are routed to a `null` receiver before the default Keep receiver. Adapt this pattern to the alert noise and operational priorities of your own EO platform.

## SLOs

SLOs focus alerting on user-visible service quality.

A CPU alert can show resource pressure. An SLO checks whether the service
still meets its latency or availability target. Use resource metrics to
explain the condition and request checks to judge its effect on users.

## Burn Rate

Burn rate shows how fast a service uses its error budget and helps operators
judge how soon they need to respond.

For example:

```
stac_get_latency_500ms_burn_rate_1h > 14.4
```

In the current STAC latency rules, the objective is effectively "99% of requests should complete within 500 ms". That leaves a 1% latency error budget: up to 1% of requests may be slower than 500 ms before the objective is missed.

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

The alerting rules currently fire on the request-level GET and POST records. The upstream, gateway, and database records are supporting signals used to understand which layer is contributing to the user-visible symptom.

The [STAC dashboard](dashboards-and-usage.md#stac) uses separate proxy and
backend metrics. Its values can differ from the public-route alert records.

## EO Service Alerts

A healthy pod can still serve slow or failed requests. SLO alerts check the
quality of service paths such as STAC for users and downstream building blocks.

The alert response uses:

- metrics, synthetics, and SLOs detect a meaningful service symptom
- Prometheus and Alertmanager evaluate, group, and route the signal
- Keep enriches the event for investigation
- supporting metrics and logs help the operator find the affected component

These checks supply the results used during Inspect and Investigate.

## From Alerting to Remediation

An alert should lead to an appropriate response.

After **Inspect** raises and enriches the incident, a human operator owns
**Investigate**. The operator uses the prepared signals to decide what is
likely wrong, what remains uncertain, and which mitigation is safe.

Act should then use the
[library of named remediation actions](remediation-actions.md), which is still
in development. Each action will define its checks, approval, execution
limits, and verification. Existing scripts and procedures can run the actions.

Verify completes the action by checking workload health and the user-facing
service again. If the same synthetic check or SLO remains unhealthy, the action
did not solve the incident and the flow returns to Investigate.

After recovery, Learn uses the incident record to improve alerts, dashboards,
investigation guidance, remediation actions, and verification for the next
incident.
