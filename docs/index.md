# Operations Building Block

These pages describe operations for the `eoepca-demo` cluster: how to check a
problem, restore a service, and prepare for platform changes. They cover the
deployed tools and the checks still needed before sign-off.

The work is divided into three areas:

| Discipline | Main question | EOEPCA page |
| --- | --- | --- |
| Security Operations | Is the platform protected, and are security events detected and handled? | [Security Operations](security-operations.md) |
| Reliability and Remediation Operations | Are user services healthy, and can service be restored safely? | [Reliability and Remediation Operations](reliability-and-remediation-operations.md) |
| Platform Lifecycle and Continuity Operations | Can the platform be changed, supported, recovered, and retired safely? | [Platform Lifecycle and Continuity Operations](platform-lifecycle-and-continuity-operations.md) |

A security finding may need a software update, which must then be deployed and
checked for service failures. In cases like this, the team responsible for the
immediate risk leads the work, with help from the other teams.

## Operator Reference

The [Operating Model](operating-model.md) explains how an incident moves from
Inspect to Investigate and Act, with Verify as part of Act and Learn after
recovery. The [Monitoring Stack](monitoring-stack.md),
[Dashboards and Usage](dashboards-and-usage.md), and
[Alerting and SLOs](alerting-and-slos.md) describe the tools used during that work.

For worked examples, follow a slow request in
[STAC Service Path](stac-scenario.md), or a session or data problem in
[Workspace Monitoring](workspace-scenario.md).
[When the Kubernetes API Slows Down](control-plane-scenario.md) covers failures
that affect several services at once, with Keep alerts and Grafana screenshots.
