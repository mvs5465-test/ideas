# Secret Bootstrap Elimination With Private SOPS Repo

## Summary

Minimize imperative secret creation in `quick-start.sh` by moving runtime/master secret material to declarative, encrypted manifests stored in a separate private repo. Use SOPS with age keys and a defined ArgoCD decryption plugin path so bootstrap is reduced to a small documented root-of-trust step.

## Status

Proposed

## Problem

Current bootstrap still creates multiple master secrets imperatively from `~/.secrets/*`. This is the remaining non-GitOps workflow hole in cluster recovery and repeatability:
- secret provisioning is manual and procedural
- recovery depends on local operator steps
- drift can occur between script behavior and declarative cluster state

Current security baseline (important):
- Secrets live in local `~/.secrets/` and are not committed to Git.
- This is an intentional trust boundary with lower remote attack surface for a homelab setup.

## Proposal

Adopt a two-repo secrets model:

1. Keep app/infra manifests in existing repos (`local-k8s-argocd`, `local-k8s-apps`).
2. Create a separate private repo for encrypted Kubernetes Secret manifests.
3. Encrypt secret manifests with SOPS using age recipients.
4. Configure ArgoCD repo-server for `kustomize-sops` (KSOPS) decryption:
   - repo-server init container installs `sops` + `ksops`
   - custom kustomize plugin path enabled for repo-server
   - age private key provided to repo-server from a bootstrap secret
5. Keep only minimal bootstrap for root trust material (`age` private key and private-repo access credential), then remove other imperative secret creation from `quick-start.sh` once parity is verified.

Target secrets to migrate first:
- `ghcr-master-secret`
- `github-pr-slack-notifier-master-secret`
- `grafana-alerting-master-secret`
- `velero-minio-master-secret`

Bootstrap carve-out (explicitly not migrated in phase 1):
- `argocd-repo-creds` (or equivalent private repo access secret) remains bootstrap-provided to avoid circular dependency when ArgoCD clones the private secrets repo.

Security decision gate:
- This migration only proceeds if it is shown to be materially safer than the current local-secrets model for this homelab threat model.
- \"More GitOps\" alone is not sufficient justification.

## Scope

In scope:
- private secrets repo setup and ArgoCD integration
- SOPS + age key model definition and key custody policy
- explicit ArgoCD decryption mechanism (`kustomize-sops`/KSOPS in repo-server)
- migration of current bootstrap-created secrets to encrypted declarative manifests
- `quick-start.sh` cleanup for non-bootstrap secret steps
- ESO compatibility: keep secret names and `external-secrets` namespace targets unchanged so existing fan-out manifests continue to work
- explicit security comparison against current model, including compromise probability and blast radius

Out of scope:
- replacing ESO fan-out model
- full external vault migration (AWS/GCP/Vault)
- rotating every app credential immediately beyond migration baseline
- eliminating all bootstrap imperatives in phase 1 (root-of-trust bootstrap remains)

## Risks And Tradeoffs

- Decryption key handling becomes the critical trust anchor; key loss blocks decryption.
- Separate private repo adds one more sync dependency and access-control surface.
- Misconfigured SOPS/Argo integration can block first-boot until fixed.
- Private repo availability becomes part of recovery path; Git provider outage can delay secret sync.
- Encrypted secret blobs in Git create historical ciphertext exposure; if age private key is later compromised, historical commits become decryptable.

## Alternatives Considered

1. Keep `quick-start.sh` imperative secret creation.
- Lower change effort, but keeps the same reproducibility and drift risks.
- Strong current security boundary for homelab: secrets never stored in Git.

2. Sealed Secrets controller.
- Valid approach, but adds a controller dependency and cert lifecycle concerns.

3. External secret manager only.
- Strong long-term direction, but higher setup complexity for immediate bootstrap parity.

Preferred approach: SOPS + age + private repo for minimal-runtime overhead and strong GitOps alignment.

Security caveat:
- Preferred only if the security decision gate is passed. If not, keep local `~/.secrets` as the trust boundary and pursue declarative ergonomics without storing secret material in Git.

## Rollout Plan

1. Foundation
- Create private repo for encrypted secrets.
- Generate age keypair and implement key custody baseline:
  - primary key in local secure secret storage (for example password manager secure note)
  - second encrypted offline backup copy
  - recovery test documented
- Add ArgoCD repo access and AppProject allowlist entry for the private repo.
- Add repo-server KSOPS/SOPS plugin wiring in `local-k8s-argocd`.
- Document the one remaining bootstrap step (provision age private key + private-repo access secret in `argocd`).

2. Pilot
- Migrate one low-risk secret path and validate sync/decrypt/app behavior.

3. Full migration
- Migrate master/runtime secrets to encrypted manifests in private repo (excluding explicit bootstrap carve-outs).
- Keep old `quick-start.sh` steps behind a temporary fallback flag during validation window.

4. Cutover
- Remove imperative secret creation blocks from `quick-start.sh` for migrated secret set.
- Run full from-scratch cluster rebuild validation.
- Run security review checkpoint before final cutover:
  - compare local-secrets baseline vs private-repo model
  - document accepted risks and why they are acceptable for homelab
  - if not accepted, stop rollout and keep current trust boundary

## Success Criteria

- Fresh cluster bootstrap reaches healthy state with only documented root-of-trust bootstrap secrets.
- All migrated secrets are sourced from encrypted manifests in the private repo.
- ESO fan-out dependent workloads receive expected runtime secrets after sync from unchanged secret names in `external-secrets`.
- `quick-start.sh` no longer contains `kubectl create secret` logic for migrated non-bootstrap secret set.
- Recovery runbook explicitly documents and verifies bootstrap prerequisites (`age` key + private repo credential).
- Security decision gate signed off with explicit rationale that risk is improved or acceptable vs current local-secrets model.
