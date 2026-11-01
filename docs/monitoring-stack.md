# Monitoring Stack

The monitoring tools run in the `operations` namespace of `eoepca-demo`.
Prometheus metrics and Loki logs are available together in Grafana, while
Keep holds alerts and their response records. Deployment sources are in
[`eoepca-plus/argocd/eoepca/operations`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations).

## Deployed Components

The main components are:

| Component | What it does | When it helps |
| --- | --- | --- |
| Prometheus | Collects metrics and evaluates rules. | Its queries show trends; target status helps explain missing measurements. |
| Grafana | Shows metrics and logs in dashboards. | Charts place service and component behaviour on the same timeline. |
| Alertmanager | Groups alerts and routes notifications. | Routing, silences, inhibition, and delivery status can explain a missing notification. |
| Loki | Stores searchable logs. | Request and component events can explain a symptom found in metrics. |
| Grafana Alloy | Forwards Pod and selected node-service logs; API audit collection is configured but unverified. | Collection checks help when an expected source is absent from Loki. |
| Keep | Stores alerts, saved measurements, and response status. | An operator can see related alerts, who is handling them, and what was saved when they arrived. |
| Keep OAuth2 Proxy | Restricts access to the Keep interface. | It checks the user's Keycloak roles before allowing access. |
| Keep relay | Adapts Alertmanager webhooks to the Keep event endpoint. | Relay errors can explain why a delivered webhook has no matching Keep alert. |

The deployment definitions are under
[`parts/monitoring`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/monitoring)
and
[`parts/alerting`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/alerting).

## What Each Component Does

### Prometheus

Prometheus scrapes platform and application targets, then evaluates the
baseline, service, and workspace rules against those measurements. The stored
time series supply dashboards, recording rules, and alerts.

The retention and persistence settings are defined in [`app-kube-prometheus-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/app-kube-prometheus-stack.yaml).

### Grafana

Grafana uses OIDC login and has Prometheus and Loki data sources. This lets an
operator compare a metric trend with request errors or component events from
the same period without moving to a separate log tool.

### Alertmanager

Alertmanager groups repeated alerts and routes their notifications to Keep.
If an expected alert is missing there, its routes, silences, inhibition rules,
and delivery status are the first part of the notification path to check.

### Loki

Loki stores workload and selected node-service logs. An API audit collection
path is also configured in Alloy, but delivery of those records remains
unverified.
The current deployment uses:

- a single-binary Loki setup
- object storage on S3-compatible backend
- a chart-managed ServiceMonitor for Loki metrics
- Loki canaries to exercise the log pipeline

See [`app-loki-stack.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/app-loki-stack.yaml).

### Grafana Alloy

Alloy runs as a DaemonSet on each node. It sends Kubernetes Pod logs and
selected node-service journal records to Loki, with labels such as namespace,
pod, container, and app. Its API audit pipeline is configured to normalize and
forward audit events, subject to the delivery gap noted above.

The pipeline is defined in [`monitoring/alloy/config.alloy`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/alloy/config.alloy).

### Keep

Keep brings related alerts and their response status into one place. For
API/etcd and STAC alerts, it also saves measurements and Grafana links, so the
assigned operator does not have to rebuild the initial queries. This is useful
when several services fail together, as in the
[control-plane scenario](control-plane-scenario.md).

For API/etcd and STAC alerts, a shared Keep workflow writes firing and resolved
events to the backend log. Alloy sends these to Loki. Grafana reads them as
[chart markers](dashboards-and-usage.md#keep-events), with links back to Keep.
The alerting reference describes the [event path](alerting-and-slos.md#keep-events-in-grafana).

## Use the Stack During an Incident

An alert's time and affected service give the investigation a starting point.
Its Grafana link opens the relevant charts, if one is configured. For a user
report without an alert, the service dashboard is the starting point.
The charts show whether the symptom is visible at the gateway,
application, workload, or a shared dependency; Loki can then supply event and
request details for that period.

When expected data is missing, the collection path needs checking before the
data can support a decision. Prometheus target status, Alloy, and Loki canaries
help distinguish a quiet service from a collection failure.

After a change, repeating the user's operation tests recovery. Comparing the
same metric and log queries before and after the change helps confirm the
result. Pod readiness alone cannot show that the service works for users.

## Public Operator Endpoints

The public endpoints use Keycloak login:

- Grafana at <https://monitoring.develop-v2.eoepca.org>, requiring a `monitoring:grafana_admin`, `monitoring:grafana_editor`, or `monitoring:grafana_viewer` role in Keycloak
- Keep at <https://alerting.develop-v2.eoepca.org/alerts/feed>, requiring an `alerting:keep_admin` or `alerting:keep_noc` role in Keycloak

These endpoints are exposed through APISIX routes:

- [`monitoring/routes.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/routes.yaml)
- [`alerting/routes.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/alerting/routes.yaml)

The [ServiceMonitors and PodMonitors](service-monitors.md) page explains scrape
configuration. The [STAC](stac-scenario.md) and [workspace](workspace-scenario.md)
examples show how to use the measurements during an investigation.
