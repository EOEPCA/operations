# Security Operations

Security Operations owns platform posture, preventive controls, detection,
security evidence, and response. EOEPCA uses separate controls for workload
admission, deployed-state scanning, request authorization, secret and
certificate handling, and operator evidence.

## Deployed Tools and Configuration

| Control | Deployed tools and configuration | Why | Configuration source |
| --- | --- | --- | --- |
| Workload admission | Kubernetes Pod Security Admission uses Restricted audit and warning controls. Kyverno adds policy reports, exceptions, and application policies. | It reduces risk from unsafe workload settings. It evaluates Kubernetes resources, not user requests to services. | [Namespace configuration](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/eoepca/kustomization.yaml) and [Kyverno policies](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/kyverno/policies). |
| Security scanning | Trivy Operator runs scheduled vulnerability, configuration, RBAC, and compliance scans. It stores results as Kubernetes reports. | It finds known risks and compliance gaps. It inspects deployed resources and images but does not block live activity. | [Trivy Operator](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/trivy/parts/app-trivy-operator.yaml). |
| Runtime detection | Falco is not part of the current deployment. | It detects suspicious activity in running workloads. It complements admission and image scanning after a workload starts. | |
| Identity and gateway | Keycloak authenticates users. OPA and OPAL supply authorization policy. APISIX enforces access at the gateway. | It permits only authenticated and authorized requests on protected inbound HTTP routes to deployed workloads. This is separate from `kubectl` access to the Kubernetes API server. | [IAM](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/iam/parts) and [APISIX](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/apisix/parts). |
| Cluster access | Kubernetes RBAC and NetworkPolicy limit API actions and network paths. | It limits damage from excessive access. RBAC controls Kubernetes API actions, while NetworkPolicy controls Pod network paths. | Service definitions in the [Argo CD tree](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd). |
| Secrets and certificates | Sealed Secrets and External Secrets deliver secrets. cert-manager issues and renews certificates. | It protects sensitive data and trusted connections. These controls manage workload secrets and TLS, not user authorization. | [Sealed Secrets](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/app-sealed-secrets.yaml), [External Secrets](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/eso), and [certificates](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/cert-manager/parts). |
| Evidence and response | Alloy and Loki collect logs. Prometheus and Grafana provide metrics and views. Alertmanager and Keep route and manage alerts. | It helps operators detect, investigate, and respond. It reports activity and risk but does not enforce access. | [Monitoring](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/monitoring), [rules](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/_rules), and [alerting](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts/alerting). |

## Sign-off Evidence

Each row gives the current evidence and a practical next step. Confirm the
evidence again before a formal sign-off.

`✓` means that an automated part of the check is deployed. An empty cell means
that automation is not available or the check needs manual work. The mark does
not mean that the complete check has passed.

Only commands that returned HTTP 200 and useful evidence during the live review
are included. Re-run them before sign-off.

| Sign-off check | Automated part | Current state | Next step |
| --- | --- | --- | --- |
| Platform and node identity | ✓ | Kubernetes reports node identity, role, and readiness. | Save the inventory with the sign-off record.<br>`kubectl get nodes -o wide` |
| CIS coverage | ✓ | Trivy produces node compliance reports with findings that need review. | Review each finding and record its resolution or approved exception.<br>`kubectl get clustercompliancereports.aquasecurity.github.io -o 'custom-columns=NAME:.metadata.name,UPDATED:.status.updateTimestamp,PASS:.status.summary.passCount,FAIL:.status.summary.failCount'` |
| Pod admission | ✓ | Pod Security Admission and Kyverno evaluate Restricted controls in audit mode. | Resolve reported findings, then test a controlled move to enforcement.<br>`kubectl get policyreports.wgpolicyk8s.io -A -o 'custom-columns=NAMESPACE:.metadata.namespace,KIND:.scope.kind,RESOURCE:.scope.name,POLICY:.results[0].policy,PASS:.summary.pass,FAIL:.summary.fail'` |
| Fail-closed admission | ✓ | The main Restricted policy permits admission when its webhook is unavailable. | Agree on the failure policy and test it before enforcement.<br>`kubectl get clusterpolicy.kyverno.io pod-security-restricted-audit -o 'custom-columns=NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,FAILURE-POLICY:.spec.failurePolicy,ACTION:.spec.rules[0].validate.failureAction'` |
| Trivy coverage and freshness | ✓ | Trivy creates workload and compliance reports. Security alerts show that findings need attention. | Match deployed workloads to fresh reports and work through high-priority findings.<br>`kubectl get vulnerabilityreports.aquasecurity.github.io -A -o 'custom-columns=NAMESPACE:.metadata.namespace,WORKLOAD:.metadata.labels.trivy-operator\.resource\.name,UPDATED:.report.updateTimestamp,CRITICAL:.report.summary.criticalCount,HIGH:.report.summary.highCount'` |
| Exceptions |  | Kyverno exceptions include scope, owner, reason, and review information. Other security exceptions are less consistent. | Use the same exception record for policy, scan, and compliance findings.<br>`kubectl get policyexceptions.kyverno.io -A -o 'custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,OWNER:.metadata.annotations.security\.eoepca\.org/owner,REASON:.metadata.annotations.security\.eoepca\.org/reason,REVIEW-AFTER:.metadata.annotations.security\.eoepca\.org/review-after'` |
| Falco coverage and delivery |  | Falco runtime detection does not exist yet. | Deploy runtime detection and test delivery, or record an approved alternative.<br>`kubectl get daemonsets -A -l app.kubernetes.io/name=falco` |
| Kubernetes API audit | ✓ | Audit collection and alert rules are configured, but current audit records are not available in Loki yet. | Restore audit delivery and test one allowed and one denied action. |
| Identity and gateway enforcement | ✓ | Keycloak, OPA, OPAL, and APISIX are deployed. A retained access test is not available yet. | Record allowed, denied, and direct-backend request tests. |
| RBAC and network boundaries | ✓ | RBAC and NetworkPolicy controls are present. A complete access matrix and probe record are not available yet. | Record approved access paths and test one allowed and one denied path. |
| Secret handling | ✓ | Secret controllers are deployed. A retained delivery and leakage test is not available yet. | Test a safe marker from source to workload and confirm that it does not appear in Git or logs. |
| Certificate issue and use | ✓ | Certificate resources report ready, and the public TLS path was valid during inspection. | Keep a renewal alert and repeat the external TLS check before sign-off.<br>`kubectl get certificates.cert-manager.io -A` |
| Dashboard freshness and detail | ✓ | Security dashboards and report-freshness alerts exist. Log-source freshness is incomplete. | Add log-source freshness detection and test dashboard drill-down. |

## Operator Detail

Use [Dashboards and Usage](dashboards-and-usage.md) for the Kyverno and Trivy
views. Use [Alerting and SLOs](alerting-and-slos.md) for alert routing and Keep.
