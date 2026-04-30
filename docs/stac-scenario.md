# STAC Scenario

The STAC path is a good concrete example for the Operations Building Block because it is directly relevant to EO platform operators. If STAC is slow or failing, catalogue access and discovery suffer immediately.

## Current STAC Path

In the demo environment, the public STAC endpoint is:

- <https://eoapi.develop.eoepca.org/stac>

The route definition is in [`argocd/eoepca/data-access/parts/route.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/route.yaml).

Operationally, the path looks roughly like this:

```
client
  -> APISIX route data-access_stac_stac-route
  -> eoapi-stac-auth-proxy
  -> eoapi-stac
  -> PostgreSQL / pgSTAC
```

## What We Can Observe Today

The demo environment already gives operators several useful signals for this path.

### Gateway Metrics

The STAC alert rules in [`_rules/stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_rules/stac-alerts.yaml) use APISIX metrics for the route `data-access_stac_stac-route`.

Those rules distinguish between:

- request latency as seen by the client-facing route
- upstream application latency
- APISIX gateway latency

That is already valuable because it helps separate "the whole request is slow" from "the gateway layer is slow" and "the upstream service is slow".

### Database Metrics

The same STAC rules also include database correlation via:

```
ccp_pg_stat_statements_total_mean_exec_time_ms{dbname="eoapi", role="eoapi"}
```

That gives a PostgreSQL-side view which can help explain whether backend query execution is contributing to the symptom.

### Logs

Alloy collects Kubernetes pod logs and forwards them to Loki. Because the Alloy configuration discovers pod logs across the cluster, STAC-related workloads in `data-access` can still be investigated through logs even when native application metrics are missing.

### Synthetic Checks

The data-access deployment also includes a synthetic STAC checker:

- [`deployment-synthetic-api-check.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/deployment-synthetic-api-check.yaml)
- [`check-stac.sh`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/synthetic-api-check/check-stac.sh)

This is useful for black-box confirmation of availability and response time from an operator point of view.

## How We Work With The Available Data

Since we can at least differentiate between request, upstream and gateway latency, we can use Keep to enrich firing alerts with metadata from different sources. This is done by Keep's inherent workflow capability. For each SLO exists a corresponding workflow under [https://alerting.develop.eoepca.org/workflows](https://alerting.develop.eoepca.org/workflows) that is triggered automatically whenever an alert is fired.

Currently, a simple analysis is performed to check whether the database, the application or the gateway is the culprit of the performance degradation. Furthermore, the current mean database latency, gateway burn rate and app burn rate is queried from Prometheus. All this information is then printed to the console together with a link to the corresponding Grafana dashboard.

This way, an operator has a clear picture of the most important metrics and components of an alert. In the future, this information will be used to perform automatic remediation playbooks to enable self-healing mechanisms.

## Current Shortcomings

The main observability gap is at the native application layer.

### No ServiceMonitors In `data-access`

The eoepca-demo cluster has no `ServiceMonitor` objects in the `data-access` namespace. Current ServiceMonitors are present in `operations`, `ingress-apisix`, and `infra`, but not for `eoapi` workloads.

### `eoapi-stac` Does Not Expose A Working Prometheus Endpoint

The `eoapi-stac` service currently exposes only port `8080` in the cluster service definition, and an in-cluster probe against:

```
http://eoapi-stac.data-access.svc.cluster.local:8080/metrics
```

returns `404`.

That does not mean the service is invisible. Because the workload is containerised and runs on Kubernetes, the platform still exposes useful signals around it: pod health, restarts, CPU, memory, network traffic, logs, gateway timings, and database exporter metrics. Those signals already support many operational questions.

The missing piece is application-native instrumentation from `eoapi-stac` itself. For the main STAC application path, there is currently no usable Prometheus metrics endpoint at `/metrics` that exposes STAC-specific behaviour from inside the application.

### Consequence For Operators

Because of that gap, operators can observe STAC mostly from the outside and from surrounding platform layers:

- APISIX request and upstream timings
- Kubernetes pod health, restarts, and resource usage
- database exporter metrics
- logs
- synthetic checks

That is already useful for detecting symptoms and narrowing down the affected layer. What operators cannot yet observe directly from `eoapi-stac` is application-specific behaviour such as:

- handler-level request counters and error classes
- app-native latency histograms
- cache behaviour
- auth or filter decisions inside the STAC application path
- internal queue, worker, or database-pool behaviour

## Why This Matters

The current setup is already useful for detecting that STAC quality is degrading. It is weaker at explaining exactly why the application behaved that way.

That is an important distinction:

- the Operations BB is already able to detect user-visible risk
- the next maturity step is better service-native instrumentation

## What Would Improve The Situation

The natural next step would be to instrument `eoapi` and related components so that operators can scrape them directly.

That would ideally include:

- a Prometheus-compatible metrics endpoint in `eoapi-stac`
- stable service labels and a `ServiceMonitor`
- a similar decision for `eoapi-stac-auth-proxy` if its behaviour is operationally relevant
- EO platform dashboards for STAC based on native application metrics
- alerts that combine gateway, application, and database views more explicitly

Once that exists, the Operations BB can support a much stronger STAC operating model: not only "the path is slow", but also "which layer inside the STAC service is responsible".
