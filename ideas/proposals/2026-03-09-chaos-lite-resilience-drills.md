# Chaos Lite Resilience Drills

## Summary

Introduce a safe, homelab-friendly chaos program that runs controlled failure drills on a schedule and measures recovery behavior. The goal is to continuously validate resilience and reduce surprise outages through low-blast-radius experiments.

## Status

Proposed

## Problem

We currently infer resilience from normal operation, but we do not routinely validate recovery under failure conditions. This creates gaps in:
- confidence that self-healing and restart behavior works as expected
- understanding of recovery time under common fault scenarios
- early detection of brittle dependencies

## Proposal

Implement a "Chaos Lite" drill system with guardrails:

1. Controlled Experiments
- Start with safe scenarios:
  - single pod termination for selected stateless workloads
  - restart drills for selected stateful components with rollback checks
  - temporary dependency disruption (for example DNS/service path fault) in limited scope

2. Recovery Measurement
- Track and visualize:
  - mean time to recovery (MTTR)
  - failure rate by experiment type
  - services exceeding recovery SLO thresholds

3. Operational Signal
- Post drill outcomes to Slack with:
  - experiment executed
  - pass/fail and recovery time
  - remediation hint when thresholds are exceeded

## Scope

In scope:
- scheduled drill runner with strict allowlist
- guardrails (single active experiment, maintenance window, auto-abort rules)
- dashboard + Slack result reporting

Out of scope:
- broad production-grade chaos mesh rollout
- destructive multi-failure experiments in phase 1
- auto-remediation beyond defined rollback/abort behavior

## Risks And Tradeoffs

- Even safe drills can create user-visible blips if scheduling is poor.
- Poorly scoped experiments can produce misleading conclusions.
- Added control logic and reporting introduces maintenance overhead.

## Alternatives Considered

1. No chaos testing.
- Lowest effort but leaves resilience assumptions untested.

2. Manual occasional failure tests.
- Useful but inconsistent and hard to trend.

3. Full-featured chaos platforms.
- Powerful but heavier than required for homelab phase 1.

Preferred approach: small, controlled, repeatable drills with strong guardrails.

## Rollout Plan

1. Phase 1: Foundation
- Define experiment allowlist and safety policies.
- Add one weekly stateless pod-kill drill.

2. Phase 2: Measurement
- Add MTTR and pass/fail dashboards.
- Add Slack reporting and threshold alerts.

3. Phase 3: Coverage Expansion
- Add 2-3 additional experiment types with explicit rollback criteria.

4. Phase 4: Operationalization
- Add monthly resilience review loop and backlog items from drill findings.

## Success Criteria

- Weekly drills execute reliably with guardrails enforced.
- Recovery metrics are visible and trendable over time.
- At least 90% of drills recover within defined SLO target windows.
- Drill findings produce concrete follow-up fixes that improve measured recovery outcomes.
