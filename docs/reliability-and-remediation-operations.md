# Reliability and Remediation Operations

When a user cannot complete a request, reliability work starts with that
failure. Measurements, logs, Kubernetes state, and recent GitOps changes help
explain it. The operator then needs a procedure with clear limits and a way
to check that the user can complete the request again.

## Deployed Tools and Configuration

| Area | Configuration and use | Configuration source |
| --- | --- | --- |
| Metrics and dashboards | Prometheus collects platform and service measurements for Git-managed Grafana dashboards. Changes in health and load help locate the problem, but need other evidence to establish its cause. | [Monitoring](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/monitoring) and [dashboards](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/_dashboards). |
| Logs | Alloy sends workload and selected node logs to Loki. These events can explain symptoms first seen in metrics or alerts. | [Loki and Alloy](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/monitoring). |
| Alerts and coordination | Prometheus rules detect conditions that need attention. Alertmanager sends notifications to Keep, where operators can review related alerts and record the response. | [Rules](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/_rules) and [alerting](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/alerting). |
| User-service checks | The STAC synthetic checker sends regular requests and logs their status and timing. It tests the request path that internal Pod health cannot confirm. | [Deployment](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/deployment-synthetic-api-check.yaml) and [check script](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/synthetic-api-check/check-stac.sh). |
| GitOps | Argo CD applies the declared configuration and reports drift and reconciliation health. A successful sync still needs a separate service check. | [Argo CD tree](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd). |

## Sign-off Evidence

Previous tests checked that Keep retained alerts across a backend Pod
replacement. Separate API/etcd and STAC test alerts then checked repeated
delivery, acknowledgement, and resolution, with saved snapshots and Loki
markers. Those alerts went directly to Keep: the tests did not cover the
complete Alertmanager route or a remediation procedure.

The table records review results and the work still needed. Re-run the
commands and review fresh results before sign-off. `✓` marks a deployed
automated part of a check, rather than a passed check. An empty cell means
the check needs manual work or has no automation.

| Sign-off check | Automated part | Current state | Next step |
| --- | --- | --- | --- |
| User-facing health | ✓ | The STAC checker logs HTTP status, latency, and response checks. It does not publish a service-health metric. | Review the latest request results, then publish a service-health metric.<br>`kubectl logs -n data-access deployment/synthetic-api-check --tail=20` |
| Alert delivery | ✓ | Alertmanager routes alerts to Keep. Marked API/etcd and STAC alerts sent directly to Keep passed their workflow lifecycle checks. | Retain a separate test through the complete Alertmanager delivery route.<br>`kubectl get alertmanagerconfig.monitoring.coreos.com keep -n operations -o 'custom-columns=NAME:.metadata.name,RECEIVER:.spec.route.receiver'` |
| Missing telemetry | ✓ | Prometheus can detect missing metric targets. Log-source freshness is not covered in the same way. | Add missing-log detection and test the stale-source view. |
| Investigation path |  | The tools provide measurements, logs, and change history, but no timed service exercise has been saved. | Run an exercise and record how the evidence was found and how long it took. |
| Runbook execution |  | General remediation guidance exists, but tested service runbooks are not yet available. | Add a small set of service runbooks with stop and escalation conditions. |
| Workflow control |  | Git defines Keep workflows for enrichment and event logs. Production remediation workflows are not yet defined. | Add a remediation workflow with approval, explicit targets, retry limits, and rollback rules. |
| Keep remediation |  | Keep records the response, but no remediation run has been saved. | Test that workflow against a safe fault and save the result. |
| Recovery | ✓ | Metrics and synthetic checks can show recovery, but no complete recovery record has been saved. | Define how long recovery must remain stable and use these checks after the workflow test. |
| GitOps reconciliation | ✓ | Argo CD reports desired-state health, but no change and drift exercise has been saved. | Test a reviewed change and safe drift, then save the reconciliation result.<br>`kubectl get applications.argoproj.io -n argocd -o 'custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'` |
| Incident record and learning |  | The tools can supply a timeline; no complete exercise record brings it together yet. | Link the alert, decision, action, recovery checks, and follow-up work in one record. |

## Operator Detail

[Observability Basics](basic-concepts.md) explains what metrics, logs, and
alerts can tell you. The [dashboard guide](dashboards-and-usage.md) covers the
available views, and [Remediation Actions](remediation-actions.md) describes
the planned action library.

The [STAC](stac-scenario.md), [workspace](workspace-scenario.md), and
[control-plane](control-plane-scenario.md) scenarios follow specific problems
from the first report through investigation and recovery checks.
