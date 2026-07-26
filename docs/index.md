# Operations Building Block

The EOEPCA Operations Building Block turns evidence of service problems into
safe, verified action and learning.

![The EOEPCA operating model moves from Inspect to Investigate and Act. Verify completes Act, and Learn improves the next response.](img/operating-model.svg){ .operations-model }

Start with the [Operating Model](operating-model.md) to learn what each stage
means. The concept pages explain the evidence used during
[Inspect and Investigate](basic-concepts.md) and the
[remediation actions](remediation-actions.md) used during Act.

## Documentation

The documentation is organised from concepts to implementation:

- **Foundations** defines the operating model, observability basics, and the
  remediation-action concept.
- **Current Implementation** describes the monitoring, dashboard, scraping,
  alerting, and SLO capabilities deployed in the EOEPCA demo.
- **End-to-End Example** applies the complete model to the STAC service path.

## Deployment Sources

The deployment sources for the demo environment are in
[`eoepca-plus/argocd/operations`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/operations).
This repository contains the documentation.

## Why This Matters

EO platforms contain many services, gateways, databases, and background
components. Operators lose time when these parts expose different or unclear
signals.

The Operations BB helps standardise the operational model so that platform teams can:

- detect service degradation through common signals
- prepare evidence and context for investigation
- reason about user-facing service quality through SLOs
- choose from known remediation actions instead of ad hoc commands
- verify recovery and improve the response after each incident
