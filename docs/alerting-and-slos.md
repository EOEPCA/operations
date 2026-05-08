# Alerting and SLOs

The Operations BB is not only about collecting data. It is about making that data actionable for operators.

## Prometheus, Alertmanager, and Keep

The current EOEPCA demo uses a layered model:

1. Prometheus scrapes metrics and evaluates recording and alerting rules.
2. Alertmanager groups and routes alerts.
3. Keep enriches selected alerts so operators can triage them with more context.

This lets each component do the job it is good at.

## Baseline Alerts

The baseline rules live in [`_rules/baseline-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_rules/baseline-alerts.yaml). They cover platform-level concerns such as:

- missing Prometheus targets
- Prometheus ingest problems
- rule evaluation failures
- node pressure and disk exhaustion
- crash looping pods
- replica mismatches
- persistent volume exhaustion

These rules are useful because they give operators a stable default baseline before service-specific alerting is added.

## Why Alertmanager Alone Is Not Enough

Alertmanager is very good at routing and grouping alerts, but it does not automatically answer questions such as:

- which user-facing service is actually at risk?
- what related signals should I inspect next?
- is this a gateway problem, an application problem, or a database problem?
- what mitigation step is likely to help?

That is why the Operations BB also motivates an enrichment layer.

## Keep for Alert Enrichment

In the EOEPCA demo, Keep is integrated through:

- [`alertmanagerconfig-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/alertmanagerconfig-keep.yaml)
- [`keep-alertmanager-relay.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/keep-alertmanager-relay.yaml)
- [`app-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/app-keep.yaml)

This setup allows selected alerts to arrive in Keep, where they can be correlated with additional operational context. The value is not only visibility. The value is reducing the time between "alert fired" and "operator knows what to inspect next".

In the live cluster, the `keep` `AlertmanagerConfig` sends resolved and firing alerts to:

```
http://keep-alertmanager-relay.operations.svc.cluster.local/alerts/event/prometheus
```

Selected noisy workload alerts are routed to a `null` receiver before the default Keep receiver. Adapt this pattern to the alert noise and operational priorities of your own EO platform.

## SLOs

SLOs move alerting closer to user-visible service quality.

Instead of saying:

- alert when CPU is above 80%

an SLO-oriented approach asks:

- is the service still delivering acceptable latency or availability?

That distinction matters. Resource pressure is only important when it threatens service quality.

## Burn Rate

Burn rate is a practical way to express urgency. It shows how fast a service is consuming its error budget.

For example:

```
stac_get_latency_500ms_burn_rate_1h > 14.4
```

In the current STAC latency rules, the objective is effectively "99% of requests should complete within 500 ms". That leaves a 1% latency error budget: up to 1% of requests may be slower than 500 ms before the objective is missed.

The burn-rate recording rule divides the observed slow-request ratio by that allowed 1% budget. A burn rate of `1` means the service is consuming the budget exactly as fast as the objective allows. A burn rate of `14.4` means it is consuming the budget 14.4 times faster than allowed.

For a 30-day SLO window, sustaining a `14.4` burn rate would use the whole 30-day error budget in about two days:

```
30 days / 14.4 = 2.08 days
```

Because the example uses the one-hour burn-rate record, it is intended to catch sharp, urgent degradation. That is more meaningful than a generic infrastructure threshold because it directly connects to service quality.

The deployed STAC rules are defined in [`stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_rules/stac-alerts.yaml). The live `stac-alerts` PrometheusRule contains:

- one-hour and six-hour request-latency burn-rate records for STAC GET and POST requests
- matching upstream application and APISIX gateway burn-rate records for diagnosis
- a PostgreSQL mean execution latency record for database correlation
- warning alerts where the six-hour request burn rate is above `6` for `30m`, unless the one-hour critical condition is already true
- critical alerts where the one-hour request burn rate is above `14.4`

The alerting rules currently fire on the request-level GET and POST records. The upstream, gateway, and database records are supporting signals used to understand which layer is contributing to the user-visible symptom.

## Why This Fits EO Platforms

EO platform operators care about service paths such as STAC, not only about whether a pod is alive. SLO-driven alerting helps focus attention on what users and downstream building blocks actually experience.

This is the core idea of the Operations BB:

- metrics and logs provide signals
- Prometheus and Alertmanager detect and route problems
- Keep enriches the event
- SLOs tell operators what matters most

## Outlook: Remediation

The same operating model can later support remediation. Remediation does not have to mean a fully autonomous system from the beginning. It can start as a manual operator action linked from an alert, then grow into scripts, decision trees, or agent-assisted workflows.

The important constraint is transparency. Operators must be able to see why a remediation path was suggested, what inputs were used, what action was taken, who or what approved it, and what changed afterwards. Whether the action is manual or automated, the result should be traceable enough to support audit, review, rollback, and learning from incidents.
