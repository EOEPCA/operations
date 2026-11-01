# When the Kubernetes API Slows Down

Several workspaces stay pending, processing jobs do not start, and controllers
report errors. When these symptoms start together, the Kubernetes API or its
database, etcd, may be the shared dependency that needs attention.

The EOEPCA demo has had API and etcd stalls that stopped controllers from
renewing their leader leases. Reducing avoidable API traffic reduced load,
but did not establish the cause of those stalls.

## Review the Alert in Keep

In the [Keep alert feed](https://alerting.develop-v2.eoepca.org/alerts/feed),
alerts for `kubernetes-control-plane` can help explain why several services
are affected. Look for alerts around the time the failures started. When you
take an alert, assign it to yourself and acknowledge it so other operators
can see that someone is handling it.

Keep enriches a new firing alert with a snapshot of API traffic, its earlier
baseline, etcd latency, busy resources, and Crossplane and Kyverno activity.
The alert's Grafana link opens charts for that period, while the saved values
remain in Keep after acknowledgement and resolution.

Open the alert, select **View Alert**, then enable **Enriched Fields Only**
to see the saved links and measurements.

[![Keep shows the saved links and API/etcd measurements.](img/control-plane/keep-enrichment.png)](img/control-plane/keep-enrichment.png)

*The screenshots show **Manual Test: API and etcd**, an info alert sent directly
to Keep on 1 November 2026. This checked enrichment and chart markers without
causing a platform fault. Delivery through Alertmanager was not part of the test.*

## Compare Events with the Charts

Follow the alert's Grafana link to compare the saved values with the charts
before and after the notification. If you open the
[Kubernetes / API and etcd dashboard](https://monitoring.develop-v2.eoepca.org/d/eoepca-control-plane)
directly, set its time range to the incident period.

**Keep events** is on by default. It adds blue markers when Keep handles a
firing or resolved notification. Hover over a marker to see its time, status,
and link back to the alert. These markers come from Keep's event log, which
Alloy sends to Loki for Grafana to read. The saved measurements stay in Keep.

[![API/etcd charts with Keep events enabled and a test marker open.](img/control-plane/keep-events-on.png)](img/control-plane/keep-events-on.png)

*Keep events on, 17:37–17:42 UTC: the test fired and resolved around 17:39.*

Turning **Keep events** off gives the same charts without the markers, as
shown below. This only changes the display; metrics and alert delivery continue.

[![The same API/etcd charts with Keep events disabled.](img/control-plane/keep-events-off.png)](img/control-plane/keep-events-off.png)

*Keep events off, with the same time range and measurements.*

| Signal | What to check |
| --- | --- |
| API traffic and earlier baseline | Did traffic rise before errors or latency? |
| Request errors | Are API or backend requests failing? |
| Etcd latency | Is the API waiting for its database? This is backend request latency measured by the API server. |
| Busy API resources | Which resource types and operations account for the requests? |
| Crossplane and Kyverno requests | Which measured clients were busy? |
| Control-plane disk writes | Did node storage latency rise in the same period? |

The resource breakdown shows which API objects were accessed. Client metrics
and logs are needed to identify the callers. A rise in traffic may help explain
a stall, but storage latency can also matter. Compare the timing with care:
markers show when Keep logged the notification, including delivery delay.

## Check Recovery

After an approved change, retry the failed workspace or processing job. Its
completion is the user-facing recovery check. API errors and latency should
also fall, and controllers should resume their work before the incident is
closed. The alert may resolve before all of these checks pass.

The Keep record should explain what changed, whether it helped, and which
measurements were missing. If the problem is limited to STAC requests, the
[STAC scenario](stac-scenario.md) follows that request path in more detail.

The [dashboard guide](dashboards-and-usage.md#keep-events) covers missing
markers. The [alerting reference](alerting-and-slos.md#keep-for-alert-enrichment)
explains the shared workflows and snapshot limits. The
[deployment sources](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts)
contain the rules and dashboard.
