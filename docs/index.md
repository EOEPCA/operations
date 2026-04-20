# Operations Building Block

The EOEPCA Operations Building Block exists to make EO platform services observable and operable in a consistent way. The goal is not to invent a new monitoring stack, but to show how common open-source components can be combined into an operator workflow for EO platform services.

This documentation is intentionally split into a few focused pages:

- [Basic Concepts](basic-concepts.md) explains metrics, logs, alerts, and SLOs at a high level
- [Monitoring Stack](monitoring-stack.md) describes what is deployed today in the EOEPCA demo environment
- [Dashboards And Usage](dashboards-and-usage.md) walks through the dashboards we already ship and how operators use them
- [ServiceMonitors](service-monitors.md) explains why they matter and what they enable
- [Alerting And SLOs](alerting-and-slos.md) connects Prometheus, Alertmanager, Keep, and SLO-driven operations
- [EO Platform Scenario](stac-scenario.md) grounds the BB in an EO platform STAC use case and highlights current observability shortcomings

## Deployment Sources

The deployment sources for the demo environment are maintained in [`eoepca-plus/argocd/operations`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/operations). This repository focuses on explanation and documentation, while the operational manifests live there.

## Why This Matters

EO platforms are made of multiple services, gateways, databases, and background components. When those parts expose signals in inconsistent ways, operators lose time during incident response.

The Operations BB helps standardise the operational model so that platform teams can:

- observe service behaviour through common signals
- build dashboards and alerts on top of those signals
- reason about user-facing service quality through SLOs
- enrich alerts with enough context to support mitigation
