# Secret Bootstrap Elimination With Private SOPS Repo

## Summary

Eliminate imperative secret creation in `quick-start.sh` by moving bootstrap and runtime secret material to declarative, encrypted manifests stored in a separate private repo. Use SOPS with age keys so `colima delete && ./quick-start.sh` converges without manual `kubectl create secret` steps.

## Status

Proposed

## Problem

Current bootstrap still creates multiple master secrets imperatively from `~/.secrets/*`. This is the remaining non-GitOps workflow hole in cluster recovery and repeatability:
- secret provisioning is manual and procedural
- recovery depends on local operator steps
- drift can occur between script behavior and declarative cluster state

## Proposal

Adopt a two-repo secrets model:

1. Keep app/infra manifests in existing repos (`local-k8s-argocd`, `local-k8s-apps`).
2. Create a separate private repo for encrypted Kubernetes Secret manifests.
3. Encrypt secret manifests with SOPS using age recipients.
4. Configure ArgoCD to sync that private repo and decrypt at render time.
5. Remove imperative secret creation from `quick-start.sh` once parity is verified.

Target secrets to migrate first:
- `argocd-repo-creds`
- `ghcr-master-secret`
- `github-pr-slack-notifier-master-secret`
- `grafana-alerting-master-secret`
- `velero-minio-master-secret`

## Scope

In scope:
- private secrets repo setup and ArgoCD integration
- SOPS + age key model definition
- migration of current bootstrap-created secrets to encrypted declarative manifests
- `quick-start.sh` cleanup for those secret steps

Out of scope:
- replacing ESO fan-out model
- full external vault migration (AWS/GCP/Vault)
- rotating every app credential immediately beyond migration baseline

## Risks And Tradeoffs

- Decryption key handling becomes the critical trust anchor; key loss blocks decryption.
- Separate private repo adds one more sync dependency and access-control surface.
- Misconfigured SOPS/Argo integration can block first-boot until fixed.

## Alternatives Considered

1. Keep `quick-start.sh` imperative secret creation.
- Lower change effort, but keeps the same reproducibility and drift risks.

2. Sealed Secrets controller.
- Valid approach, but adds a controller dependency and cert lifecycle concerns.

3. External secret manager only.
- Strong long-term direction, but higher setup complexity for immediate bootstrap parity.

Preferred approach: SOPS + age + private repo for minimal-runtime overhead and strong GitOps alignment.

## Rollout Plan

1. Foundation
- Create private repo for encrypted secrets.
- Generate age keypair and define key custody/backup policy.
- Add ArgoCD repo access and AppProject allowlist entry.

2. Pilot
- Migrate one low-risk secret path and validate sync/decrypt/app behavior.

3. Full migration
- Migrate all current bootstrap-created secrets.
- Keep old `quick-start.sh` steps behind a temporary fallback flag during validation window.

4. Cutover
- Remove imperative secret creation blocks from `quick-start.sh`.
- Run full from-scratch cluster rebuild validation.

## Success Criteria

- Fresh cluster bootstrap reaches healthy state without any manual secret creation commands.
- All migrated secrets are sourced from encrypted manifests in the private repo.
- ESO fan-out dependent workloads receive expected runtime secrets after sync.
- `quick-start.sh` no longer contains `kubectl create secret` logic for migrated secret set.
