# Platform Lifecycle and Continuity Operations

Platform changes need a record of what is deployed, who maintains it, and how
to recover or retire it. Git and Argo CD provide configuration and deployment
history. Service tests, support records, restore exercises, and retirement
checks are also needed to show that a change is ready.

## Deployed Tools and Configuration

| Area | Configuration and use | Configuration source |
| --- | --- | --- |
| Platform and desired state | Kubernetes runs the workloads. Argo CD applies the configuration from Git and reports drift, so changes can be reviewed and repeated. | [Argo CD desired state](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd). |
| Artifact evidence | Argo CD records deployment sources, while Trivy supplies software inventory and vulnerability reports. These help with release and support decisions, but do not establish how an image was built. | [Deployment tree](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd) and [Trivy Operator](https://github.com/EOEPCA/eoepca-plus/blob/deploy-develop/argocd/infra/trivy/parts/app-trivy-operator.yaml). |
| Certificate lifecycle | cert-manager issues and renews certificates. A ready certificate still needs a check through the public TLS route. | [Certificate configuration](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/infra/cert-manager/parts). |
| Backup and restore | RKE2 creates local platform snapshots, and some services have backup settings. There is no common backup inventory or restore procedure, so recovery still needs to be demonstrated. | Platform configuration and service definitions in the deployment tree. |
| Compatibility checks | Repository workflows provide smoke and acceptance tests. Release checks need to exercise representative platform and service paths. | [CI workflows](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/.github/workflows). |

## Sign-off Evidence

The table records findings from the previous live review. Re-run the commands
before sign-off, and save the results of restore, failure-domain, and retirement
exercises. Configuration inspection alone cannot complete those checks.

`✓` marks a deployed automated part of a check, rather than a passed check.
An empty cell means the check needs manual work or has no automation.

| Sign-off check | Automated part | Current state | Next step |
| --- | --- | --- | --- |
| Declarative inventory and drift | ✓ | Git and Argo CD provide a declared inventory and drift status. Ownership, support, and data-class information are incomplete. | Add these fields to a small owned platform inventory and clear drift before release.<br>`kubectl get applications.argoproj.io -n argocd -o 'custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'` |
| Artifact and support trace | ✓ | Deployment sources and Trivy reports exist, but the record does not yet link source, build, and deployed image. | Start with the operations namespace. Record the owner, support status, source, provenance, signature, and deployed image identity.<br>`kubectl get sbomreports.aquasecurity.github.io -n operations -o 'custom-columns=WORKLOAD:.metadata.labels.trivy-operator\.resource\.name,IMAGE:.report.artifact.repository,DIGEST:.report.artifact.digest,UPDATED:.report.updateTimestamp'` |
| Planned change | ✓ | GitOps and test workflows are in place, but no complete upgrade and rollback exercise has been saved. | Run a representative upgrade and save health, compatibility, and rollback results. |
| Isolated restore | ✓ | Local platform snapshots exist, but there is no common backup inventory or saved isolated restore test. | Define the permitted data loss (RPO), recovery time (RTO), backup location, and owner for each data class, then test a restore.<br>`kubectl get etcdsnapshotfiles.k3s.cattle.io,volumesnapshots.snapshot.storage.k8s.io -A` |
| Disaster recovery |  | No complete failure-domain exercise has been saved. | Restore a minimum service in an isolated environment and measure recovery. |
| Decommissioning |  | No complete service-retirement exercise has been saved. | Retire a disposable service and verify removal of access, routes, data, monitoring, and cost. |

## Operator Detail

The [Operating Model](operating-model.md) explains approval and recovery checks
for a change. [Alerting and SLOs](alerting-and-slos.md) describes the service
measurements that help judge whether a rollout has affected users.
