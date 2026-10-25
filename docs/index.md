# Operations Building Block

EOEPCA separates platform operations into three disciplines. They use many of
the same cluster, GitOps, monitoring, and incident records. Each discipline has
its own responsibilities and checks.

The pages give a short description of each discipline, show the main deployed
tools and configuration, and assess each sign-off check.

| Discipline | Main question | EOEPCA page |
| --- | --- | --- |
| Security Operations | Is the platform protected, and are security events detected and handled? | [Security Operations](security-operations.md) |
| Reliability and Remediation Operations | Are user services healthy, and can service be restored safely? | [Reliability and Remediation Operations](reliability-and-remediation-operations.md) |
| Platform Lifecycle and Continuity Operations | Can the platform be changed, supported, recovered, and retired safely? | [Platform Lifecycle and Continuity Operations](platform-lifecycle-and-continuity-operations.md) |

A security finding can require a planned release, and that release can cause a
service incident. The discipline that owns the immediate risk leads the work.
Each team remains responsible for its part of the work.

## Operator Reference

Use the [Operating Model](operating-model.md) for the Inspect, Investigate,
Act, Verify, and Learn loop. The [Monitoring Stack](monitoring-stack.md),
[Dashboards and Usage](dashboards-and-usage.md), and
[Alerting and SLOs](alerting-and-slos.md) give tool-level detail. The
[STAC Service Path](stac-scenario.md) applies the loop to a user request.
[Workspace Monitoring](workspace-scenario.md) shows how to check sessions,
resource use, and managed data activity.
