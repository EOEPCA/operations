# Basic Concepts

The Operations Building Block is about turning platform behaviour into something operators can understand and act on.

## Observability In Practice

At a high level, observability answers four different questions:

| Signal | Main question | Typical use |
| --- | --- | --- |
| Metrics | Is the service healthy, fast, or overloaded? | Dashboards, alerts, SLOs, capacity planning |
| Logs | What exactly happened? | Incident investigation, request tracing by context, error inspection |
| Alerts | Does someone need to act now? | Notification and incident response |
| SLOs | Is the service still delivering acceptable quality? | Prioritisation and escalation |

## Metrics

Metrics are numerical time series. They are especially useful when operators need to answer questions such as:

- is request latency rising?
- is the error rate increasing?
- are pods hitting CPU or memory limits?
- how quickly is an error budget being consumed?

Metrics work well when the question is about trend, rate, saturation, or threshold.

## Logs

Logs provide detail that metrics usually cannot. They are useful when an operator needs to understand:

- which request failed
- which component generated the error
- what payload, tenant, or collection was involved
- what changed just before the incident

Metrics usually tell operators that something is wrong. Logs help explain why.

## Alerts

Alerts are the point where monitoring becomes operator action. A good alert should:

- indicate a problem that matters
- carry enough context to triage
- avoid firing on every transient spike

This is one reason the Operations BB cares about signal quality, not just signal collection.

## SLOs

A Service Level Objective (SLO) is a measurable statement of acceptable service quality. Instead of saying "the STAC API should be fast", an operator can say:

- 99% of STAC GET requests should complete within 500 ms
- the API should stay within a defined availability target
- the error rate should remain below an agreed threshold

SLOs are useful because they connect technical behaviour to user impact. They also make it easier to prioritise work: not every warning deserves the same response, but an SLO at risk usually does.

## Why EO Platforms Need This

EO platforms are operationally awkward in a very practical way:

- data access is often latency-sensitive
- workloads are distributed across several building blocks
- failures can sit at gateway, application, database, or infrastructure level
- different teams may own different pieces of the platform

That is why the Operations BB focuses on a shared operating model. It gives teams a common language for signals, dashboards, alerting, and service quality without forcing one rigid implementation everywhere.
