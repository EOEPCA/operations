# Remediation Actions

A `RemediationAction` will describe a prepared procedure: when it may help,
what it can change, and how to check the result. An operator will select and
approve it after investigating the problem.

!!! info "In development"

    The library is not deployed. This page describes its intended use;
    the API, resource model, and execution design are still open.

## Why Use an Action Library?

Preparing a procedure before an incident gives the operator time to review
its targets, risks, and recovery checks. It could run through a script,
Kubernetes operation, workflow, or API call. The library will describe the
required result and the limits that apply, regardless of how the procedure
runs.

Read-only checks may run automatically. A production change will require human
approval, explicit targets, execution and retry limits, and a service-level
recovery check. Destructive changes, changes to data, and identity or access
operations remain outside the ordinary library.

## What an Action Needs to Explain

Before an action can be used, its procedure must explain:

- its intended outcome and when it may help
- the permitted services and targets
- its preconditions and execution limits
- the required approval and execution method
- how recovery is verified
- when to stop, roll back, or return to Investigate

## The Action Flow

1. **Select:** The operator chooses an action based on the investigation.
2. **Check and approve:** The operator confirms the target and preconditions,
   then approves the change.
3. **Execute:** The procedure runs within its defined limits.
4. **Verify:** Health checks and a user-facing request confirm whether the
   service has recovered.
5. **Record:** The incident record keeps the evidence, approval, execution
   details, and result for later review.

If the recovery check fails, the response returns to Investigate or follows
the procedure's stop, rollback, retry, or escalation instructions.

## Example: Restart a Stateless Workload

| Question | Example answer |
| --- | --- |
| What does it do? | Replace the running replicas of a selected stateless workload. |
| When may it help? | Replicas are unhealthy or stuck and replacement is a reasonable mitigation. |
| What is checked first? | The target is stateless, no rollout is active, and enough capacity remains. |
| Who approves it? | A human operator. |
| How is it verified? | Replicas become healthy, the external service request succeeds, and relevant SLO signals recover. |
| What if it fails? | Stop repeated attempts and return to Investigate or escalate. |
