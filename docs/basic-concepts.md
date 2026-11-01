# Observability Basics

When a request is slow, a latency chart shows when the problem started. Logs
can then show which component reported a delay or error. An alert brings a
condition to an operator's attention; an SLO sets the service quality expected
by users. Together, these support **Inspect** and **Investigate** in the
[operating model](operating-model.md).

## Observability In Practice

Each source answers a different question:

| Signal | Main question | Role in the operating model |
| --- | --- | --- |
| Metrics | Is the service healthy, fast, or overloaded? | Inspect trends and service quality; find affected components |
| Logs | What exactly happened? | Investigate requests, errors, and component behaviour |
| Alerts | Does someone need to act now? | Notify the operator of a condition that needs attention |
| SLOs | Is the service still delivering acceptable quality? | Focus Inspect on user impact and urgency |

## Metrics

Metrics record values over time, such as request duration or memory use.
They help answer questions such as:

- is request latency rising?
- is the error rate increasing?
- are pods hitting CPU or memory limits?
- how quickly is an error budget being consumed?

These values show whether a problem is growing and which components are busy.
They usually cannot explain why a particular request failed.

## Logs

Logs record individual events. After finding a rise in errors on a chart,
an operator can look at logs from that period to find:

- which request failed
- which component generated the error
- what payload, tenant, or collection was involved
- what changed just before the incident

An error message can help explain the symptom, but it may also be a result of
a failure elsewhere. The timing, affected requests, and other measurements
help establish whether it explains the problem.

## Alerts

An alert calls for attention when a measured condition needs a response.
Its rule should:

- indicate a problem that matters
- identify the affected service and give a useful starting point
- avoid firing on every transient spike

Repeated notifications that need no action make important alerts harder to
see. Rules need review when this happens, or when a service fails without an
alert.

## SLOs

A Service Level Objective (SLO) defines acceptable service quality in measurable
terms. For STAC, the latency target is 99% of GET and POST requests within
500 ms. The remaining 1% is the error budget for slow requests.

An SLO can also set a target for availability or request errors. It helps the
operator judge the effect on users and how soon a response is needed.

## Why EO Platforms Need This

A slow data request can involve the gateway, application, database, or
infrastructure. Different teams may own these components. A shared timeline
of measurements, logs, and changes helps them investigate the same problem.
The operating model gives them a common process for choosing a change and
checking that users can use the service again.
