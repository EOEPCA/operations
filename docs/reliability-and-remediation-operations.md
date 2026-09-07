# Reliability and Remediation Operations

Reliability and Remediation Operations starts with a user-visible service
result. Metrics, logs, alerts, Kubernetes state, Git history, and GitOps state
help an operator explain the symptom. A safe response also needs a bounded
action and independent proof that users recovered.

## Deployed Tools and Configuration

| Area | Deployed tools and configuration | Why | Configuration source |
| --- | --- | --- | --- |
| Metrics and dashboards | Prometheus collects platform and service metrics. Grafana uses Git-managed data sources and dashboards. | They show health, load, and change over time. They show symptoms and trends but do not prove a cause. | [Monitoring](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/monitoring) and [dashboards](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/_dashboards). |
| Logs | Alloy collects workload and selected node logs. Loki stores them for investigation and alert rules. | They provide event detail for an investigation. They help explain symptoms found in metrics and alerts. | [Loki and Alloy](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/monitoring). |
| Alerts and coordination | Prometheus evaluates rules. Alertmanager routes alerts to Keep, which supports triage and response. | They bring actionable conditions to operators. Alertmanager routes notifications, while Keep supports triage and response tracking. | [Rules](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/_rules) and [alerting](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/alerting). |
| User-service checks | A STAC synthetic checker sends regular requests and logs status and timing. | It checks the service from the user path. It complements internal Pod and component health. | [Deployment](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/deployment-synthetic-api-check.yaml) and [check script](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/data-access/parts/synthetic-api-check/check-stac.sh). |
| GitOps | Argo CD applies desired state and reports reconciliation health. | It detects drift and restores reviewed configuration. It proves desired-state reconciliation, not user-service recovery. | [Argo CD tree](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd). |

## Sign-off Evidence

Each row gives the current evidence and a practical next step. The inspection
was read-only, so it did not create faults or run remediation.

`✓` means that an automated part of the check is deployed. An empty cell means
that automation is not available or the check needs manual work. The mark does
not mean that the complete check has passed.

Only commands that returned HTTP 200 and useful evidence during the live review
are included. Re-run them before sign-off.

| Sign-off check | Automated part | Current state | Next step |
| --- | --- | --- | --- |
| User-facing health | ✓ | The STAC checker logs HTTP status, latency, and response checks. It does not publish a service-health metric. | Review the latest request results, then publish a service-health metric.<br>`kubectl logs -n data-access deployment/synthetic-api-check --tail=20` |
| Alert delivery | ✓ | Alertmanager routes alerts to Keep. A retained delivery test is not available yet. | Confirm the active receiver, then send a safe test alert and record its receipt in Keep.<br>`kubectl get alertmanagerconfig.monitoring.coreos.com keep -n operations -o 'custom-columns=NAME:.metadata.name,RECEIVER:.spec.route.receiver'` |
| Missing telemetry | ✓ | Prometheus can detect missing metric targets. Log-source freshness is not covered in the same way. | Add missing-log detection and test the stale-source view. |
| Investigation path |  | Grafana, Loki, Kubernetes, and GitOps provide investigation evidence. A timed exercise record is not available yet. | Run a service exercise and record the evidence path and elapsed time. |
| Runbook execution |  | General remediation guidance exists, but tested service runbooks are not yet available. | Add a small set of service runbooks with stop and escalation conditions. |
| Workflow control |  | Keep is deployed, but production remediation workflows are not defined in Git. | Add one bounded workflow with approval, scope, retry, and rollback rules. |
| Keep remediation |  | Keep can support response coordination. A retained remediation run is not available yet. | Run the bounded workflow against a safe fault and retain the result. |
| Recovery | ✓ | Metrics and synthetic checks can verify recovery. A complete recovery record is not available yet. | Define stable recovery checks and use them after the workflow test. |
| GitOps reconciliation | ✓ | Argo CD reports desired-state health. A retained change and drift exercise is not available yet. | Test a reviewed change and safe drift, then save the reconciliation result.<br>`kubectl get applications.argoproj.io -n argocd -o 'custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'` |
| Incident record and learning |  | The tools can provide a timeline, but a complete exercise record is not available yet. | Link the alert, decision, action, verification, and improvement in one record. |

## Operator Detail

Use [Observability Basics](basic-concepts.md) for signal limits,
[Dashboards and Usage](dashboards-and-usage.md) for investigation, and
[Remediation Actions](remediation-actions.md) for the current action model. The
[STAC Service Path](stac-scenario.md) is the current end-to-end example.
