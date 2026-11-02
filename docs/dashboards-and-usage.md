# Dashboards and Usage

Grafana dashboards show service behaviour and the components behind it. The
dashboard for the reported problem is usually the best starting point, with
the time range set to include the first symptom and any recent change.

## How Dashboards Are Provisioned

The dashboards are generated as ConfigMaps under [`argocd/eoepca/operations/parts/_dashboards`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/_dashboards) and labeled with `grafana_dashboard: "1"`.

Grafana loads them automatically, so no manual import is needed. Each dashboard
has the `baseline` tag. Public links are set in the manifests, and stable
dashboard IDs preserve bookmarks and alert links. The
[configuration reference](alerting-and-slos.md#shared-configuration) explains
how Keep uses those links.

The live `operations` namespace contains dashboard ConfigMaps for:

- `curated-prometheus-overview`
- `curated-k8s-resources-cluster`
- `curated-k8s-resources-node`
- `curated-k8s-resources-pod`
- `curated-control-plane`
- `curated-crossplane`
- `curated-security-compliance`
- `curated-stac-slo`
- `workspaces-overview`

The Compliance dashboard combines Kyverno policy results and Trivy
scan findings. It replaces the separate policy-audit and Trivy dashboards.

## Current Dashboards

<span id="prometheus-overview"></span>

### Prometheus

Source: [`prometheus.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/prometheus.json)

When monitoring data is missing or looks wrong, this dashboard helps check
Prometheus itself. It shows target discovery and sync, scrape failures,
appended samples, query rate, and storage state.

### Kubernetes Cluster

Source: [`kubernetes-cluster.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/kubernetes-cluster.json)

This dashboard compares namespaces by CPU and memory use, requested and
limited capacity, and network and storage activity. It helps locate an affected
namespace when the report does not yet identify a workload.

### Kubernetes Pods

Source: [`kubernetes-pods.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/kubernetes-pods.json)

Once the namespace is known, the Pod dashboard shows container CPU use and
throttling, memory working set against requests and limits, and network and
disk activity. These measurements help narrow the investigation to a workload
or container.

### Kubernetes Nodes

Source: [`kubernetes-nodes.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/kubernetes-nodes.json)

Node capacity and scheduling pressure can explain why workloads do not start
or run slowly. This dashboard is also useful when several affected workloads
share a node.

### Kubernetes API and etcd

Source: [`kubernetes-api-and-etcd.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/kubernetes-api-and-etcd.json)

The [demo dashboard](https://monitoring.develop-v2.eoepca.org/d/eoepca-control-plane)
shows API traffic against its earlier baseline, API errors, availability, and
etcd latency. Resource views show what the API handles. Crossplane and Kyverno
client activity helps identify measured sources of requests.

When several services report errors or resource creation becomes slow, these
charts can show a shared API or etcd problem. Controller activity needs to be
compared over the same time range before attributing load to a controller. The
[control-plane scenario](control-plane-scenario.md) explains the investigation
and the links between Keep alerts and Grafana chart markers.

### Crossplane Reconciliation

Source: [`crossplane-reconciliation.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/crossplane-reconciliation.json)

The [demo dashboard](https://monitoring.develop-v2.eoepca.org/d/eoepca-crossplane)
shows reconciliation rates and p99 duration for Datalab and Storage composition
controllers and provider managed-resource controllers. Error panels show the
total for the selected period and errors per minute. The **Provider** selector
filters provider charts. CRD gate controllers are excluded.

Rates and p99 duration use five-minute windows. Composition counts include
periodic checks and runs triggered by watched changes. They combine all
resources of each type; they do not identify an individual workspace or the
trigger for a run. Successful scheduled requeues are not errors.

### STAC

Source: [`stac.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/stac.json)

The **STAC** dashboard follows the request path from APISIX ingress to
`stac-auth-proxy` and `eoapi-stac`. The ingress section shows public request and
server-error rates, request/upstream/gateway p99, and GET/POST latency budget
burn. Separate proxy and backend sections use each service's own counters and
histograms to show:

- the share of GET and POST requests within 500 ms over 30 days
- latency error budget use and one-hour and six-hour burn rates
- request rate, HTTP 5xx errors, and p99 latency
- mean query latency for the shared eoapi database in the backend section

The target is 99% of requests within 500 ms. A burn rate above one uses the
budget faster than allowed; more than 100% budget used exceeds the target.
Periods without traffic show no data. Proxy and backend counts include some
of the same requests, so adding them would count requests more than once.
The alert rules measure the public APISIX route.

<span id="security-compliance"></span>

### Compliance

Source: [`compliance.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/compliance.json)

This dashboard brings Kyverno policy results together with Trivy vulnerability,
exposed-secret, configuration, RBAC, and compliance findings. Critical findings
appear first, followed by action tables for secrets and access, image updates,
and deployment configuration. Before a change, the matching Trivy or Kyverno
report is needed to confirm the affected image or resource and the report age.

The main views show:

- critical findings that need review first
- high and critical image vulnerabilities and secret findings
- configuration and access findings
- failed policy rules and evaluation errors
- collection errors, report ages, exceptions, and cluster controls

Repeated workload reports for the same image count once per image and
namespace. Secret findings come from image scans, not scans of live Kubernetes
Secrets. Cluster controls and ClusterRole findings apply to all namespaces.

The collapsed review section contains scan health and exception details. Zero
findings are meaningful only when the expected workloads have fresh reports;
a missing scan can also leave the total at zero.

### Workspaces

Source: [`workspaces.json`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_dashboards/workspaces.json)

The workspace selector compares running sessions, CPU and memory use, requested
PVC storage, and owned buckets for one or more workspaces. Each workspace
includes its main, session, and control-plane namespaces. The dashboard opens
with 24 hours of history.

Managed data panels show HTTP requests, successful and failed presigned URL
requests, transfer bytes, and collection health. Activity totals cover the
selected period. Requested PVC storage is allocated capacity; transfer bytes
measure data movement. Neither measures stored data.

The [workspace scenario](workspace-scenario.md) explains the session counts,
recording rules, PodMonitor, and a short investigation example.

## Keep Events

Both **Kubernetes API and etcd** and **STAC** have a **Keep events** control,
enabled by default. With the incident period selected, it places alert
notifications on the charts alongside the measurements.

On time-series charts such as API traffic and etcd latency, blue lines and
small markers show these events. Hover over a marker for the alert name,
`firing` or `resolved` status, time, and **Open alert in Keep** link. Both states
use blue, so the status text distinguishes them. Stat cards and tables have
no markers. If the link opens only Keep's feed, the matching alert can be
selected there.

For example, `EtcdRequestLatencyHigh: firing` marks a notification handled by
Keep. A later `EtcdRequestLatencyHigh: resolved` marks its recovery notification.
The measurements around these points show what was happening when Keep handled
each notification. Each marker is a separate event; the chart does not shade
the whole incident period.

Turning off **Keep events** hides the markers. Alerts still reach Keep, and
the metric charts continue to work. Assignment, acknowledgement, and response
notes stay in Keep. The saved metrics snapshot also stays with the alert.

Grafana reads up to 100 matching log entries for the selected period. If there
are more events, a shorter range is needed to see them. Marker times come from
the workflow log and include alert delivery and processing delay.

If markers are missing, check the same period in Grafana Explore with the
Loki data source:

```logql
{namespace="operations",container="keep"}
  | json | event_type="keep_alert_event"
```

No matching events can mean that no covered alert changed state, or that the
workflow or log collection failed. Check the workflow execution in Keep and
the Alloy/Loki collection path. Marker history follows Loki log retention.
If Keep history is cleared, older markers can remain after their linked alerts
have been removed.
The `dashboard_uid` field identifies the target dashboard; each dashboard's
query selects it with `${__dashboard.uid}`. The
Kubernetes API and etcd dashboard also reads the earlier event format to
retain history.
The [alerting reference](alerting-and-slos.md#keep-events-in-grafana) describes
the workflows that produce these events. Grafana calls the marker query a
[Loki annotation query](https://grafana.com/docs/grafana/latest/datasources/loki/annotations/).

## Typical Operator Usage

An alert's dashboard link provides a starting point. For a user report without
an alert, the affected service determines where to start: STAC for searches,
Workspaces for sessions and managed data, or Compliance for scan and
policy findings. Missing or inconsistent measurements need a Prometheus
check first.

For a broader resource problem, Kubernetes Cluster helps locate the
namespace, then Kubernetes Pods gives container detail. Kubernetes Nodes
helps when the affected workloads share a node or show scheduling pressure.
If workspace creation and several controllers fail together, API/etcd charts
may reveal a shared dependency problem. Crossplane Reconciliation helps check
controller rates, duration, and errors when managed resources are slow to become
ready.

Loki logs and source scan or policy reports provide details behind the charts.
The [STAC](stac-scenario.md) and [control-plane](control-plane-scenario.md)
scenarios show this investigation with Keep and Grafana screenshots.

## Dashboard Checks

- Set the time range to include the first symptom and the most recent relevant
  change.
- Match the service, namespace, workload, and route filters across the panels
  being compared. These dashboards cover the single `eoepca-demo` cluster.
- Treat an empty panel as unknown until you confirm the target, query, and data
  source.
- Open the source metric, log, policy report, or scan report before you approve
  a change.
- Reuse the same dashboard and filters during Verify so that before-and-after
  evidence is comparable.
