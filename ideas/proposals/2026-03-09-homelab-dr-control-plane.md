# Homelab Disaster Recovery Control Plane

## Summary

Build a GitOps-managed disaster recovery layer for the local cluster that continuously validates we can recover from real failures, not just back up data. The feature combines scheduled backups, restore drills, and a recovery runbook generator so cluster recovery is repeatable within a target RTO/RPO and does not depend on memory.

## Status

Rejected

## Problem

We already have strong GitOps bootstrap and observability, but recovery confidence is still largely implicit. If a cluster rebuild, namespace deletion, PVC corruption, or secret loss happens, we lack a fully validated and routine path to recover quickly with predictable downtime and data-loss bounds.

Current gap:
- Backups may exist but are not routinely restore-tested.
- Recovery steps are spread across scripts, manifests, and operator knowledge.
- There is no single health signal proving recoverability over time.

## Proposal

Add a DR control plane with three core capabilities:

1. Backup Fabric
- Use Velero with CSI snapshots and Restic fallback where needed.
- Back up cluster-scoped resources and selected namespaces on schedule.
- Store backup artifacts in a local S3-compatible store (MinIO) with retention tiers.
- Require backup storage to survive cluster loss: MinIO data must live on host-mounted persistent storage (for example `~/clusterstorage/velero`) or an external S3-compatible endpoint, not an ephemeral in-cluster-only PVC.
- Require Velero object-store credentials to be managed by External Secrets using the existing ClusterSecretStore pattern.

2. Restore Verification
- Run scheduled restore drills into an isolated `dr-verify` namespace.
- Validate critical services (ArgoCD, Grafana, Loki, wiki/home, notifier paths) with automated checks.
- Emit pass/fail metrics and Slack summaries after each drill.
- Keep drill orchestration fully declarative in GitOps-managed manifests (Kubernetes CronJobs/Workflows), not ad hoc operator scripts.

3. Recovery UX
- Generate a versioned "Recovery Plan" artifact from cluster config:
  - bootstrap order
  - required secrets and dependencies
  - one-command rebuild flow
- Track recovery objectives in code (`target_rto_minutes`, `target_rpo_minutes`).
- Add a "recoverability score" card in Cluster Home backed by drill results.
- Reuse existing notification paths (`github-pr-slack-notifier` and Grafana/Alertmanager routing) for drill-result messaging.

## Scope

In scope:
- Velero/backup operator install and config via GitOps.
- Backup schedules for high-value namespaces and cluster resources.
- Automated weekly restore drill pipeline and validation checks.
- Recovery runbook generation and surfaced recoverability score.
- Resource controls for drill workloads (requests/limits, concurrency policy, and off-peak schedules) so DR checks fit homelab constraints.
- Cross-repo implementation coordination: app and metrics surface work in `local-k8s-apps` and any required policy/wave adjustments in `local-k8s-argocd`.

Out of scope:
- Multi-cluster active/active failover.
- Off-site encrypted replication to paid cloud storage (future phase).
- Full HA redesign of every stateful component.

## Risks And Tradeoffs

- More infrastructure components (Velero + MinIO + jobs) increase operational footprint.
- Restore drills consume disk and CPU during execution windows.
- Snapshot portability differs across storage backends; fallback paths must be tested.
- Security hardening is required because backup stores contain sensitive objects.
- Host-mounted backup storage improves survivability but still shares a physical host; it protects against cluster reset, not full host failure.

## Alternatives Considered

1. Keep ad hoc backups and manual restore docs.
- Lowest setup cost but weak confidence under real incident pressure.

2. Use only GitOps re-bootstrap (no data backup).
- Works for stateless config but loses stateful telemetry/history and app data.

3. Nightly etcd-only snapshots.
- Helpful for control-plane state but incomplete for workload PVC and app-level recovery.

Preferred approach: Velero-based backup + automated restore drills because it proves real recovery behavior, not just backup artifact existence.

## Rollout Plan

1. Phase 1: Foundation
- Deploy MinIO backup bucket and Velero with least-privilege credentials.
- Mount MinIO data to host-persistent storage (or external S3 endpoint) and verify backup artifacts survive a full cluster rebuild.
- Add backup schedules for `argocd`, `monitoring`, `services`, and cluster-scoped resources.
- Wire Velero credentials through ESO-managed secrets.

2. Phase 2: Drill MVP
- Restore one critical app path weekly into `dr-verify` via declarative CronJob/Workflow manifests.
- Validate readiness with smoke tests and publish result to Slack.
- Set strict requests/limits and schedule drills for off-peak windows.

3. Phase 3: Full Recoverability
- Expand drill set to core platform apps and secret restoration paths.
- Publish recovery score and trend to Grafana/Cluster Home.
- Document one-command rebuild from empty cluster.
- Coordinate Cluster Home score-card implementation in `local-k8s-apps` with DR stack rollout.

4. Phase 4: Hardening
- Tune retention, encryption-at-rest, and backup pruning.
- Add quarterly game day with intentional failure scenarios.

## Success Criteria

- Weekly restore drills succeed at least 90% over a rolling 8-week window.
- Median drill recovery time meets a target RTO of 45 minutes or less.
- Backup freshness for protected namespaces stays within target RPO of 30 minutes or less.
- Recovery runbook can rebuild cluster services from a fresh Colima cluster without undocumented manual steps.
- Recoverability score is visible in Cluster Home and alerts if drill success drops below threshold.
- At least one full-cluster-loss simulation confirms backups remain recoverable after destroying and recreating the cluster.
