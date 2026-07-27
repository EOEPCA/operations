# Remediation Actions

A `RemediationAction` is a named, bounded operational procedure. It connects
**Investigate** to **Act** by turning a selected mitigation into a known action
that can be approved, run consistently, and verified.

The remediation-action library is the curated collection of those procedures.

!!! info "Capability being established"

    The remediation-action library is currently being defined and developed.
    It is not yet a finished capability of the deployed Operations Building
    Block.

    The current stack already supports much of Inspect and helps operators
    Investigate. The next step is to establish a small action library and close
    the loop through execution and verification.

    This page covers only the core concept. A stable API, resource model, and
    executor design have not yet been established.

## Why Use an Action Library?

During an incident, an operator should not have to assemble commands or work
out safety rules again. An action library offers named, trusted operational
choices. A script, Kubernetes operation, workflow, or API call may implement an
action, but the operator selects the intended outcome rather than supplying
arbitrary commands.

Read-only checks may run automatically. Production changes require human
approval, operate only on explicit targets and within defined limits, and
include service-level verification and bounded retries. Destructive,
data-affecting, or identity and access operations remain outside the ordinary
library.

## What an Action Needs to Explain

Each action defines:

- its intended outcome and when it may help
- the permitted services and targets
- its preconditions and execution limits
- the required approval and execution method
- how recovery is verified
- when to stop, roll back, or return to Investigate

## The Action Flow

1. **Select:** Investigate identifies a suitable action.
2. **Check and approve:** Preconditions, target, and approval are confirmed.
3. **Execute:** The implementation performs the bounded change.
4. **Verify:** Technical health and the user-facing outcome are checked.
5. **Record:** Evidence, approval, execution, and outcome remain traceable.

If verification fails, the action has not resolved the incident. The response
returns to Investigate or follows the defined stop, rollback, retry, or
escalation path.

## Example: Restart a Stateless Workload

| Question | Example answer |
| --- | --- |
| What does it do? | Replace the running replicas of a selected stateless workload. |
| When may it help? | Replicas are unhealthy or stuck and replacement is a reasonable mitigation. |
| What is checked first? | The target is stateless, no rollout is active, and enough capacity remains. |
| Who approves it? | A human operator. |
| How is it verified? | Replicas become healthy, the external service request succeeds, and relevant SLO signals recover. |
| What if it fails? | Stop repeated attempts and return to Investigate or escalate. |

The implementation may change without changing what the action means to the
operator.
