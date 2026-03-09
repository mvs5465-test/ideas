# Homelab Cost And Energy Intelligence

## Summary

Add a lightweight cost and power intelligence layer for the cluster that translates resource usage into estimated dollars and watts by namespace/workload. This gives clear visibility into always-on cost, spike events, and optimization opportunities without introducing heavyweight billing systems.

## Status

Proposed

## Problem

The cluster is operationally healthy, but we currently lack a clear answer to:
- which workloads are the biggest always-on resource and energy consumers
- what a change in workload profile means in monthly cost terms
- where optimization effort has the highest return

Without this, efficiency improvements are mostly intuition-driven.

## Proposal

Build a homelab-focused cost/energy model with three outputs:

1. Cost and Power Dashboard
- Grafana dashboard for per-namespace and per-app estimates:
  - estimated CPU/RAM/storage cost
  - estimated watt draw and monthly kWh
  - top always-on cost contributors

2. Weekly Summary Signal
- Scheduled summary to Slack with:
  - top 5 most expensive workloads
  - week-over-week deltas
  - notable spikes/regressions

3. Optimization Candidate Queue
- Auto-generated "high-impact candidates" list based on:
  - persistent high usage with low SLO sensitivity
  - request/limit skew and idle overhead

## Scope

In scope:
- metric collection from existing Prometheus signals
- model config for local assumptions (`$/kWh`, baseline host watts, overhead factors)
- Grafana dashboard and weekly summary delivery

Out of scope:
- cloud-provider billing parity
- exact hardware-level power metering
- automatic rightsizing actions in phase 1

## Risks And Tradeoffs

- Estimates are model-based, not meter-perfect.
- Poor assumptions can produce misleading numbers if not calibrated.
- Additional dashboarding/summary logic increases maintenance surface.

## Alternatives Considered

1. Manual periodic audits.
- Low setup effort, but poor continuity and weak trend visibility.

2. Full Kubernetes cost platforms.
- Rich features, but heavier than needed for a single-owner homelab.

3. No dedicated cost layer.
- Keeps stack simple but leaves efficiency decisions unguided.

Preferred approach: simple model-driven intelligence built on existing observability stack.

## Rollout Plan

1. Phase 1: Baseline Dashboard
- Define model assumptions and publish first Grafana dashboard.
- Validate output against observed cluster behavior.

2. Phase 2: Weekly Reporting
- Add scheduled Slack summaries with top contributors and deltas.

3. Phase 3: Optimization Queue
- Add recommendation panel for high-impact tuning candidates.

4. Phase 4: Calibration
- Recalibrate model assumptions quarterly with observed host behavior.

## Success Criteria

- Dashboard shows per-namespace/app estimated cost and power with stable daily updates.
- Weekly summary is delivered consistently with meaningful trend deltas.
- At least three optimization changes are identified from the model and validated as resource-reducing.
- Team can answer "what is our most expensive always-on workload" in under 1 minute.
