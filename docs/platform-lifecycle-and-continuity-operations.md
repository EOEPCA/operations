# Platform Lifecycle and Continuity Operations

Platform Lifecycle and Continuity Operations owns planned change,
supportability, recovery readiness, and complete retirement. GitOps supplies a
strong baseline for desired state, review, deployment history, and drift. It
does not prove service health, software support, data recovery, or safe
decommissioning.

## Deployed Tools and Configuration

| Area | Deployed tools and configuration | Why | Configuration source |
| --- | --- | --- | --- |
| Platform and desired state | Kubernetes runs the platform. Argo CD applies the declared configuration and reports drift. | It makes platform changes reviewable and repeatable. Kubernetes runs workloads, while Argo CD reconciles their declared configuration. | [Argo CD desired state](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd). |
| Artifact evidence | Argo CD records deployment sources. Trivy supplies software inventory and vulnerability reports. | It supports risk, support, and release decisions. This evidence does not by itself prove build provenance. | [Deployment tree](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd) and [Trivy Operator](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/trivy/parts/app-trivy-operator.yaml). |
| Certificate lifecycle | cert-manager issues and renews platform certificates. | It keeps service identity and encryption valid. Certificate readiness does not prove that the public route works. | [Certificate configuration](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/cert-manager/parts). |
| Backup and restore | RKE2 creates local platform snapshots. Some services have their own backup settings, but there is no common continuity inventory or restore process. | It provides the data needed to recover service. A backup is not proof that restore works. | Platform configuration and service definitions in the deployment tree. |
| Compatibility checks | Repository workflows provide smoke and acceptance test entry points. | They find release problems before users are affected. They must cover representative platform and service paths. | [CI workflows](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/.github/workflows). |

## Sign-off Evidence

Each row gives the current evidence and a practical next step. Live inspection
cannot replace restore, failure-domain, or retirement exercises.

`✓` means that an automated part of the check is deployed. An empty cell means
that automation is not available or the check needs manual work. The mark does
not mean that the complete check has passed.

Only commands that returned HTTP 200 and useful evidence during the live review
are included. Re-run them before sign-off.

| Sign-off check | Automated part | Current state | Next step |
| --- | --- | --- | --- |
| Declarative inventory and drift | ✓ | Git and Argo CD provide a declared inventory and drift status. Ownership, support, and data-class information are incomplete. | Add these fields to a small owned platform inventory and clear drift before release.<br>`kubectl get applications.argoproj.io -n argocd -o 'custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'` |
| Artifact and support trace | ✓ | Deployment sources and Trivy reports are available. A complete source-to-build-to-deployment trace is not yet available. | Start with the operations namespace. Record the owner, support status, source, provenance, signature, and deployed image identity.<br>`kubectl get sbomreports.aquasecurity.github.io -n operations -o 'custom-columns=WORKLOAD:.metadata.labels.trivy-operator\.resource\.name,IMAGE:.report.artifact.repository,DIGEST:.report.artifact.digest,UPDATED:.report.updateTimestamp'` |
| Planned change | ✓ | GitOps and test workflows support planned change. A complete upgrade and rollback record is not available yet. | Run one representative upgrade and retain health, compatibility, and rollback evidence. |
| Isolated restore | ✓ | Local platform snapshots exist. A common backup inventory and isolated restore record are not available yet. | Define RPO, RTO, backup location, and owner for each data class, then test one restore.<br>`kubectl get etcdsnapshotfiles.k3s.cattle.io,volumesnapshots.snapshot.storage.k8s.io -A` |
| Disaster recovery |  | A complete failure-domain exercise record is not available yet. | Restore a minimum service in an isolated environment and measure recovery. |
| Decommissioning |  | A complete service-retirement record is not available yet. | Retire a disposable service and verify removal of access, routes, data, monitoring, and cost. |

## Operator Detail

Use the [Operating Model](operating-model.md) for change and verification. Use
[Alerting and SLOs](alerting-and-slos.md) to define the service check that must
stay healthy during a rollout.
