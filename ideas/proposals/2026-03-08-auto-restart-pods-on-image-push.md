# Auto-Restart Workloads On New Image Push

## Summary

On our toy homelab cluster, pods can keep running old code after new images are pushed (especially on mutable tags like `:main`). We should adopt a pure-GitOps mechanism that automatically reconciles workloads to new images without manual pod restarts.

## Status

Proposed

## Problem

Today, we sometimes push a new image and still have to manually restart pods. This creates drift between what is in the registry and what is running in cluster. It also causes false confidence after deploy-related changes because the cluster may still be on old runtime code.

## Proposal

Adopt digest-based image automation with Git write-back as the default pattern:

1. Keep ArgoCD as source of truth.
2. Add Argo CD Image Updater for selected apps.
3. Track each app image by digest (`image@sha256:...`) or update pinned values to digest form.
4. Configure Image Updater to open/commit Git changes to the manifests repo when a new digest for allowed tags is available.
5. Let ArgoCD sync those Git commits, which naturally rolls pods.

This keeps deployment behavior auditable and Git-driven, while removing manual restart steps.

## Scope

In scope:
- Homelab-owned app workloads that use `ghcr.io/mvs5465-test/*` images.
- One pilot app first (recommended: `github-pr-slack-notifier`), then rollout to remaining homemade apps.
- Guardrails to avoid surprise rollouts (allowlist and update strategy).

Out of scope:
- Third-party charts/apps not built by us.
- Replacing ArgoCD or changing cluster architecture.
- Auto-remediation unrelated to image drift.

## Risks And Tradeoffs

- Added controller complexity (Image Updater) and credentials management.
- More Git churn from digest updates.
- If allowlists are too broad, rollouts may happen more often than expected.
- Slightly slower end-to-end deploy than imperative restart, but much higher traceability and repeatability.

## Alternatives Considered

1. CI-triggered `kubectl rollout restart` after image push.
- Simpler initially, but not pure GitOps and harder to audit.

2. Nightly scheduled restarts.
- Easy but imprecise; does not guarantee fast adoption of new images.

3. Keep manual restarts.
- Zero setup cost, but error-prone and operationally noisy.

Preferred approach is Image Updater + Git write-back because it aligns with pure-GitOps priority and removes manual drift cleanup.

## Rollout Plan

1. Pilot:
- Enable Image Updater for `github-pr-slack-notifier` only.
- Restrict to `main` tag lineage and digest updates.
- Verify PR/commit write-back and Argo sync behavior.

2. Validation window:
- Run for 1-2 weeks in homelab.
- Track whether manual restarts for pilot drop to zero.

3. Expand:
- Add `cluster-home`, `cluster-lite-wiki`, `cluster-query-router`, `loki-mcp-server`.
- Keep app-level allowlists and rollback instructions documented.

4. Operational docs:
- Add runbook section for "image update not propagating".
- Include ownership and failure triage steps.

## Success Criteria

- 100% of selected homemade workloads roll to new image digests without manual `rollout restart`.
- Manual restart actions for image refresh drop to near-zero within the rollout window.
- Every runtime image change is traceable to a Git commit and Argo sync event.
- No increase in failed deploy incidents attributable to image automation.
