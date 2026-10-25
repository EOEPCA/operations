# Workspace Monitoring

The Workspaces dashboard groups sessions and namespaces by workspace. Use it
to check resource use and managed data activity.

## What the Dashboard Shows

Select a workspace and time range to view running sessions, CPU and memory
use, requested PVC storage, and owned buckets. Shared services count once in
workspace totals. Managed data panels show requests, presigned URL failures,
transfer bytes, and collection health.

PVC requests show allocated capacity; transfer bytes measure data movement.
Used storage and exact outgoing S3 request counts are not measured per workspace.

## How Metrics Reach the Dashboard

The recording rules in
[`workspace-metrics.yaml`](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/_rules/workspace-metrics.yaml)
link workspace inventory, namespaces, sessions, and pods. They let dashboard
queries reuse the same mapping.

The
[`workspace-managed-data` PodMonitor](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/operations/parts/monitoring/workspace-managed-data.yaml)
collects managed data activity from `package-r` pods across workspace namespaces.
The workspace provider creates a network policy that permits Prometheus to
reach the metrics port.

## Example: A Slow Session or Failed Data Request

1. **Inspect:** Select the workspace and reported time. Check session resource
   use and request failures. If data is missing, check collection health first.
2. **Investigate:** Find the session in the table, then check its pods and logs.
   For data failures, check `package-r` logs and bucket access.
3. **Act and Verify:** After a change, repeat the user's request and confirm
   that errors and resource use recover. Pod readiness alone is not enough.
4. **Learn:** Record the cause, action, and result, including any missing metrics.
