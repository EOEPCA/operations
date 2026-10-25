# Remediation Actions

A `RemediationAction` is a named operational procedure with defined targets
and limits. It connects **Investigate** to **Act**: the operator selects an
action that can be approved, run consistently, and verified.

The remediation-action library holds these procedures.

!!! info "In development"

    The remediation-action library is still being defined and developed.
    It is not yet available in the deployed Operations Building Block.

    The current stack already supports much of Inspect and helps operators
    Investigate. The next step is to add a small action library with execution
    and verification checks.

    This page describes what the library should do. The API, resource model,
    and execution design are not yet settled.

## Why Use an Action Library?

During an incident, an operator should not have to assemble commands or work
out safety rules again. An action library provides prepared procedures with
known checks and limits. A script, Kubernetes operation, workflow, or API call
may implement an action. The operator selects the intended outcome from the
library.

Read-only checks may run automatically. Production changes require human
approval, operate only on explicit targets and within defined limits, and
include service-level verification and a retry limit. Destructive,
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
3. **Execute:** The system runs the change within the defined limits.
4. **Verify:** Technical health and the user-facing outcome are checked.
5. **Record:** Save the evidence, approval, execution details, and result.

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
