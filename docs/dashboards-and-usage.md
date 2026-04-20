# Dashboards And Usage

The EOEPCA demo already ships a small set of curated Grafana dashboards. These are not product-specific dashboards yet. They are operator dashboards that answer broad platform questions reliably.

## How Dashboards Are Provisioned

The dashboards are generated as ConfigMaps under [`argocd/operations/_dashboards`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/operations/_dashboards) and labeled with `grafana_dashboard: "1"`.

That matters because it lets Grafana pick them up automatically as managed dashboards, without manual imports.

The live `operations` namespace contains the expected dashboard ConfigMaps:

- `curated-prometheus-overview`
- `curated-k8s-resources-cluster`
- `curated-k8s-resources-node`
- `curated-k8s-resources-pod`

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

### Kubernetes / Cluster View

Source: [`k8s-resources-cluster.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_dashboards/k8s-resources-cluster.json)

This dashboard is useful for answering questions like:

- which namespaces are consuming most CPU or memory?
- how close is the cluster to requests and limits commitment?
- where is network or storage activity concentrated?

It is a good starting point when an operator knows the platform is unhealthy, but not yet which namespace or workload is involved.

### Kubernetes / Workload View

Source: [`k8s-resources-pod.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_dashboards/k8s-resources-pod.json)

This dashboard focuses on pods and containers. It helps answer:

- which container is using CPU?
- is the workload being throttled?
- how does memory working set compare to requests and limits?
- is network or disk activity unusual for this pod?

For incident response, this is often the next drill-down after identifying the relevant namespace.

### Kubernetes / Physical View

Source: [`k8s-resources-node.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/operations/_dashboards/k8s-resources-node.json)

This dashboard shifts the view to the node level. It is useful when the issue looks like cluster saturation, scheduling pressure, or a node-local problem rather than a single application fault.

## Typical Operator Usage

A practical operator workflow often looks like this:

1. An alert or symptom points to a service problem.
2. Grafana Cluster View is used to locate the affected namespace or workload area.
3. Workload View is used to inspect the specific pod or deployment.
4. Physical View is used when the problem may be node-related.
5. Loki is used to confirm what the affected component was actually doing.

## EO Platform Dashboards

The current dashboards are a strong platform baseline. EO platform dashboards for specific use cases, for example STAC, are covered in the [EO Platform Scenario](stac-scenario.md).
