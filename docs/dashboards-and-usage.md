# Dashboards and Usage

The EOEPCA demo ships curated Grafana dashboards for the monitoring system, Kubernetes resources, and the STAC SLO scenario.

## How Dashboards Are Provisioned

The dashboards are generated as ConfigMaps under [`argocd/operations/_dashboards`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/operations/_dashboards) and labeled with `grafana_dashboard: "1"`.

That matters because it lets Grafana pick them up automatically as managed dashboards, without manual imports.

The live `operations` namespace contains dashboard ConfigMaps for:

- `curated-prometheus-overview`
- `curated-k8s-resources-cluster`
- `curated-k8s-resources-node`
- `curated-k8s-resources-pod`
- `curated-stac-slo`

## Current Curated Dashboards

### Prometheus / Overview

Source: [`prometheus-overview.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_dashboards/prometheus-overview.json)

This dashboard helps operators understand whether the monitoring system itself is healthy. It includes views such as:

- discovery
- target sync
- scrape failures
- appended samples
- query rate
- Prometheus storage state

This is usually the first place to look when monitoring results seem incomplete or suspicious.

### Kubernetes / Cluster

Source: [`k8s-resources-cluster.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_dashboards/k8s-resources-cluster.json)

This dashboard is useful for answering questions like:

- which namespaces are consuming most CPU or memory?
- how close is the cluster to requests and limits commitment?
- where is network or storage activity concentrated?

It is a good starting point when an operator knows the platform is unhealthy, but not yet which namespace or workload is involved.

### Kubernetes / Workload

Source: [`k8s-resources-pod.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_dashboards/k8s-resources-pod.json)

This dashboard focuses on pods and containers. It helps answer:

- which container is using CPU?
- is the workload being throttled?
- how does memory working set compare to requests and limits?
- is network or disk activity unusual for this pod?

For incident response, this is often the next drill-down after identifying the relevant namespace.

### Kubernetes / Node

Source: [`k8s-resources-node.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_dashboards/k8s-resources-node.json)

This dashboard shifts the view to the node level. It is useful when the issue looks like cluster saturation, scheduling pressure, or a node-local problem rather than a single application fault.

### STAC / SLO

Source: [`stac-slo.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_dashboards/stac-slo.json)

This dashboard is the EO platform example dashboard. It focuses on the STAC route and uses the recording rules from [`stac-alerts.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_rules/stac-alerts.yaml) to show:

- STAC GET and POST latency burn rates
- the route currently used by the STAC APISIX rule
- database mean execution latency from PostgreSQL exporter data
- request, upstream application, and gateway views that help locate the likely layer of degradation

## Typical Operator Usage

A practical operator workflow often looks like this:

1. An alert or symptom points to a service problem.
2. The STAC SLO dashboard is used first when the alert is STAC-specific.
3. Grafana Cluster View is used to locate the affected namespace or workload area.
4. Workload View is used to inspect the specific pod or deployment.
5. Node View is used when the problem may be node-related.
6. Loki is used to confirm what the affected component was actually doing.

## EO Platform Dashboards

The STAC dashboard is intentionally built from the metrics available today. It is useful for gateway, upstream, and database correlation, but the [STAC Scenario](stac-scenario.md) explains why application-native metrics would make it much stronger.
