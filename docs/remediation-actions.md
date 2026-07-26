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

During an incident, an operator should not have to remember commands, find the
right script, and work out its safety rules again.

An action library provides named, trusted operational choices. A script,
Kubernetes operation, Flux reconciliation, workflow, or API call can implement
an action, but the operator selects the intended outcome rather than providing
arbitrary commands.

For example, **restart a stateless workload** is clearer and safer than an
interface that accepts any Kubernetes patch. The named action can restrict
which targets are allowed, check that the workload is stateless, avoid
conflicting rollouts, require approval, and define how recovery will be
verified.

## What an Action Needs to Explain

Action definitions do not need a complicated public schema. They do need to
answer a consistent set of operational questions:

- What does the action do?
- When may it help?
- Which services and targets may use it?
- What must be true before it runs?
- Who must approve it?
- How is it executed?
- How is service recovery verified?
- When should it stop, roll back, or escalate?

These answers help operators understand the action and allow different tools
to run it.

## The Action Flow

A remediation action follows a short, visible flow:

1. **Select:** Investigate identifies a suitable action.
2. **Check:** The current target and action preconditions are checked again.
3. **Approve:** The required person or policy authorises the operation.
4. **Execute:** The implementation performs the bounded change.
5. **Verify:** Technical health and the user-facing service outcome are checked.
6. **Record:** The evidence, approval, action, and outcome remain traceable.

If verification fails, the action has not resolved the incident. The response
returns to Investigate or follows the action's defined stop, rollback, retry, or
escalation path.

## Example: Restart a Stateless Workload

For example, a restart action can be described in practical terms:

| Question | Example answer |
| --- | --- |
| What does it do? | Replace the running replicas of a selected stateless workload. |
| When may it help? | Replicas are unhealthy or stuck and replacement is a reasonable mitigation. |
| What is checked first? | The target is stateless, no rollout is active, and enough capacity remains. |
| Who approves it? | A human operator. |
| How does it run? | Through a bounded Kubernetes executor rather than an arbitrary command. |
| How is it verified? | Replicas become healthy, the external service request succeeds, and relevant SLO signals recover. |
| What if it fails? | Stop repeated attempts and return to Investigate or escalate. |

The implementation may change without changing what the action means to the
operator.

## Candidate Actions

The library should remain small:

| Action | Intended use |
| --- | --- |
| Reconcile a Flux resource | Retry a known-safe source or release reconciliation. |
| Restart a stateless workload | Replace unhealthy application replicas after checking the target. |
| Scale a stateless workload | Add temporary capacity within predefined bounds. |
| Enable a predefined rate limit | Protect a service from an excessive or expensive request pattern. |
| Open a GitOps change | Propose a durable configuration correction for review and Flux reconciliation. |

These are candidate actions, not a claim that they are already implemented.
STAC actions should be selected through controlled failure scenarios and
verified against the actual service path.

## Safety Boundary

The library uses a cautious safety boundary:

- read-only inspection may run automatically
- every production change requires human approval
- actions operate only on explicit targets and within defined limits
- every action includes service-level verification
- retries are limited and visible
- destructive, data-affecting, or identity and access operations remain outside
  the ordinary library

This still allows execution to be automated. It keeps the decision to authorise
a production change explicit while the capability is being established.

## Current Establishment Work

The immediate work is to:

1. select a small set of useful STAC actions
2. document their purpose, preconditions, approval, and verification
3. place existing scripts or procedures behind those action definitions
4. connect actions to the evidence prepared during Inspect
5. exercise the complete flow through controlled incidents
6. use the results to improve both the actions and the surrounding monitoring

This work is tracked in
[`EOEPCA/operations#6`](https://github.com/EOEPCA/operations/issues/6). The issue
and implementation details are still being aligned with the operating model
described in these pages.
