# Security Operations

Security work includes checking workload settings, reviewing scan findings,
controlling access, and responding to suspicious activity. The controls below
cover different parts of this work. Their presence alone does not establish
that the platform has passed a security review.

## Deployed Tools and Configuration

| Control | Configuration and use | Configuration source |
| --- | --- | --- |
| Workload admission | Pod Security Admission audits and warns about Restricted settings. Kyverno adds policy reports, exceptions, and application policies. These checks evaluate Kubernetes resources during admission. | [Namespace configuration](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/kustomization.yaml) and [Kyverno policies](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/kyverno/policies). |
| Security scanning | Trivy Operator scans for vulnerabilities, configuration, RBAC, and compliance findings. It stores Kubernetes reports for review; it does not block activity in running workloads. | [Trivy Operator](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/trivy/parts/app-trivy-operator.yaml). |
| Runtime detection | Falco, which detects suspicious activity in running workloads, is not deployed. | |
| Identity and gateway | Keycloak authenticates users, OPA and OPAL supply authorization policy, and APISIX enforces access on protected HTTP routes. These controls cover service requests; Kubernetes API access has separate controls. | [IAM](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/iam/parts) and [APISIX](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/apisix/parts). |
| Cluster access | RBAC limits actions through the Kubernetes API. NetworkPolicy limits network paths between Pods and services. | Service definitions in the [Argo CD tree](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd). |
| Secrets and certificates | Sealed Secrets and External Secrets deliver workload secrets. cert-manager issues and renews TLS certificates. | [Sealed Secrets](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/app-sealed-secrets.yaml), [External Secrets](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/eso), and [certificates](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/cert-manager/parts). |
| Evidence and response | Alloy and Loki provide logs for investigation, alongside Prometheus metrics in Grafana. Alertmanager sends notifications to Keep, where operators record the response. | [Monitoring](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/monitoring), [rules](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/_rules), and [alerting](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/alerting). |

## Sign-off Evidence

The table records findings from the previous live review and the work still
needed. Re-run the commands and review fresh results before sign-off.

`✓` marks a deployed automated part of a check, rather than a passed check.
An empty cell means the check needs manual work or has no automation.

| Sign-off check | Automated part | Current state | Next step |
| --- | --- | --- | --- |
| Platform and node identity | ✓ | Kubernetes reports node identity, role, and readiness. | Save the inventory with the sign-off record.<br>`kubectl get nodes -o wide` |
| CIS coverage | ✓ | Trivy produces node compliance reports with findings that need review. | Review each finding and record its resolution or approved exception.<br>`kubectl get clustercompliancereports.aquasecurity.github.io -o 'custom-columns=NAME:.metadata.name,UPDATED:.status.updateTimestamp,PASS:.status.summary.passCount,FAIL:.status.summary.failCount'` |
| Pod admission | ✓ | Pod Security Admission and Kyverno evaluate Restricted controls in audit mode. | Resolve reported findings, then test a controlled move to enforcement.<br>`kubectl get policyreports.wgpolicyk8s.io -A -o 'custom-columns=NAMESPACE:.metadata.namespace,KIND:.scope.kind,RESOURCE:.scope.name,POLICY:.results[0].policy,PASS:.summary.pass,FAIL:.summary.fail'` |
| Fail-closed admission | ✓ | The main Restricted policy permits admission when its webhook is unavailable. | Agree on the failure policy and test it before enforcement.<br>`kubectl get clusterpolicy.kyverno.io pod-security-restricted-audit -o 'custom-columns=NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,FAILURE-POLICY:.spec.failurePolicy,ACTION:.spec.rules[0].validate.failureAction'` |
| Trivy coverage and freshness | ✓ | Workload and compliance reports are produced, with alerts for findings that need attention. | Match deployed workloads to fresh reports and review high-priority findings.<br>`kubectl get vulnerabilityreports.aquasecurity.github.io -A -o 'custom-columns=NAMESPACE:.metadata.namespace,WORKLOAD:.metadata.labels.trivy-operator\.resource\.name,UPDATED:.report.updateTimestamp,CRITICAL:.report.summary.criticalCount,HIGH:.report.summary.highCount'` |
| Exceptions |  | Kyverno exceptions include scope, owner, reason, and review information. Other security exceptions are less consistent. | Use the same exception record for policy, scan, and compliance findings.<br>`kubectl get policyexceptions.kyverno.io -A -o 'custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,OWNER:.metadata.annotations.security\.eoepca\.org/owner,REASON:.metadata.annotations.security\.eoepca\.org/reason,REVIEW-AFTER:.metadata.annotations.security\.eoepca\.org/review-after'` |
| Falco coverage and delivery |  | Falco is not deployed. | Deploy runtime detection and test delivery, or record an approved alternative.<br>`kubectl get daemonsets -A -l app.kubernetes.io/name=falco` |
| Kubernetes API audit | ✓ | Audit collection and alert rules are configured, but current audit records are not available in Loki yet. | Restore audit delivery and test one allowed and one denied action. |
| Identity and gateway enforcement | ✓ | Keycloak, OPA, OPAL, and APISIX are deployed, but no saved access test confirms enforcement. | Record allowed, denied, and direct-backend request tests. |
| RBAC and network boundaries | ✓ | RBAC and NetworkPolicy are configured. Approved access paths and probe results have not been collected in one record. | Record the approved paths and test one allowed and one denied path. |
| Secret handling | ✓ | Secret controllers are deployed, but no delivery and leakage test has been saved. | Test a safe marker from source to workload and confirm that it does not appear in Git or logs. |
| Certificate issue and use | ✓ | Certificate resources report ready, and the public TLS path was valid during inspection. | Keep a renewal alert and repeat the external TLS check before sign-off.<br>`kubectl get certificates.cert-manager.io -A` |
| Dashboard freshness and detail | ✓ | Security dashboards and report-freshness alerts exist. Log-source freshness is incomplete. | Add log-source freshness detection and test dashboard drill-down. |

## Operator Detail

The [dashboard guide](dashboards-and-usage.md#compliance) explains how
to review security findings and report age. The
[alerting reference](alerting-and-slos.md) covers their delivery to Keep.
