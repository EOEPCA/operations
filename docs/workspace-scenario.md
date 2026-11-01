# Workspace Monitoring

When a session is slow or a data request fails, the Workspaces dashboard
brings together the sessions, namespaces, and managed data activity for that
workspace.

## What the Dashboard Shows

Selecting a workspace and the reported time shows its running sessions, CPU
and memory use, requested PVC storage, and owned buckets. Shared services
count once in workspace totals. The managed data panels show whether requests
or presigned URLs failed, how much data was transferred, and whether metrics
are being collected.

PVC requests show allocated capacity, and transfer bytes measure data movement.
Neither shows how much data is stored. Used storage and exact outgoing S3
request counts are not measured per workspace.

## How Metrics Reach the Dashboard

The recording rules in
[`workspace-metrics.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_rules/workspace-metrics.yaml)
link workspace inventory, namespaces, sessions, and pods. The dashboard reuses
this mapping so that each panel includes the same workspace resources.

The
[`workspace-managed-data` PodMonitor](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/workspace-managed-data.yaml)
collects managed data activity from `package-r` pods across workspace namespaces.
The workspace provider creates a network policy that permits Prometheus to
reach the metrics port.

## Example: A Slow Session or Failed Data Request

Start with the workspace and the time reported by the user. Session CPU and
memory use may show resource pressure, while the managed data panels may show
failed requests. If a panel is empty, collection health needs checking before
the absence of data can be interpreted.

The session table identifies the pods whose logs can help explain a slow
session. For a failed data request, `package-r` logs and bucket access checks
can help locate the failure.

After an approved change, repeating the user's request tests whether it helped.
Errors and resource use should also recover; Pod readiness alone does not show
that the session or data access works. The incident record should include the
cause, change, result, and any missing measurements for the later review.
