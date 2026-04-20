# Alerting And SLOs

The Operations BB is not only about collecting data. It is about making that data actionable for operators.

## Prometheus, Alertmanager, And Keep

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

## Keep For Alert Enrichment

In the EOEPCA demo, Keep is integrated through:

- [`alertmanagerconfig-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/alertmanagerconfig-keep.yaml)
- [`keep-alertmanager-relay.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/keep-alertmanager-relay.yaml)
- [`app-keep.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/alerting/app-keep.yaml)

This setup allows selected alerts to arrive in Keep, where they can be correlated with additional operational context. The value is not only visibility. The value is reducing the time between "alert fired" and "operator knows what to inspect next".

In the live cluster, the `keep` `AlertmanagerConfig` sends resolved and firing alerts to:

```text
http://keep-alertmanager-relay.operations.svc.cluster.local/alerts/event/prometheus
```

As an example, `KubeJobFailed` is routed to a `null` receiver to suppress noisy alerts. Adapt this pattern to the alert noise and operational priorities of your own EO platform.

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

```text
stac_get_latency_500ms_burn_rate_1h > 14.4
```

In the current STAC latency rules, the objective is effectively "99% of requests should complete within 500 ms". That leaves a 1% latency error budget: up to 1% of requests may be slower than 500 ms before the objective is missed.

The burn-rate recording rule divides the observed slow-request ratio by that allowed 1% budget. A burn rate of `1` means the service is consuming the budget exactly as fast as the objective allows. A burn rate of `14.4` means it is consuming the budget 14.4 times faster than allowed.

For a 30-day SLO window, sustaining a `14.4` burn rate would use the whole 30-day error budget in about two days:

```text
30 days / 14.4 = 2.08 days
```

Because the example uses the one-hour burn-rate record, it is intended to catch sharp, urgent degradation. That is more meaningful than a generic infrastructure threshold because it directly connects to service quality.

The current STAC rules include both one-hour and six-hour burn-rate records for GET and POST requests. Critical alerts fire on the fast one-hour burn rate, while warning alerts use the slower six-hour view unless the critical condition is already true.

## Why This Fits EO Platforms

EO platform operators care about service paths such as STAC, not only about whether a pod is alive. SLO-driven alerting helps focus attention on what users and downstream building blocks actually experience.

This is the core idea of the Operations BB:

- metrics and logs provide signals
- Prometheus and Alertmanager detect and route problems
- Keep enriches the event
- SLOs tell operators what matters most
