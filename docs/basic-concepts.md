# Observability Basics

The Operations Building Block is about turning platform behaviour into
something operators can understand and act on. Metrics, logs, alerts, and SLOs
primarily support **Inspect** and **Investigate** in the
[operating model](operating-model.md).

## Observability In Practice

Observability answers four questions:

| Signal | Main question | Role in the operating model |
| --- | --- | --- |
| Metrics | Is the service healthy, fast, or overloaded? | Inspect trends and service quality; investigate likely failure domains |
| Logs | What exactly happened? | Investigate requests, errors, and component behaviour |
| Alerts | Does someone need to act now? | Turn an Inspect condition into operator attention |
| SLOs | Is the service still delivering acceptable quality? | Focus Inspect on user impact and urgency |

## Metrics

Metrics are numerical time series. They help answer questions such as:

- is request latency rising?
- is the error rate increasing?
- are pods hitting CPU or memory limits?
- how quickly is an error budget being consumed?

Metrics work well for trends, rates, saturation, and thresholds.

## Logs

Logs provide detail that metrics usually cannot. They are useful when an operator needs to understand:

- which request failed
- which component generated the error
- what payload, tenant, or collection was involved
- what changed just before the incident

Metrics usually tell operators that something is wrong. Logs provide evidence
for Investigate; they do not automatically establish a root cause.

## Alerts

Alerts are the point where Inspect asks for operator attention. A good alert
should:

- indicate a problem that matters
- carry enough context to triage
- avoid firing on every transient spike

This is one reason the Operations BB cares about signal quality, not just signal collection.

## SLOs

A Service Level Objective (SLO) is a measurable statement of acceptable service quality. Instead of saying "the STAC API should be fast", an operator can say:

- 99% of STAC GET requests should complete within 500 ms
- the API should stay within a defined availability target
- the error rate should remain below an agreed threshold

SLOs connect technical behaviour to user impact. They also help prioritise
work: not every warning needs the same response, but an SLO at risk usually
does.

## Why EO Platforms Need This

EO platforms can be hard to operate:

- data access is often latency-sensitive
- workloads are distributed across several building blocks
- failures can sit at gateway, application, database, or infrastructure level
- different teams may own different pieces of the platform

That is why the Operations BB focuses on a shared operating model. It gives
teams a common language for inspecting service quality, investigating
degradation, acting safely, verifying recovery, and learning from incidents
without forcing one rigid implementation everywhere.
