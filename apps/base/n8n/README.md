# n8n

Self-hosted [n8n](https://n8n.io) workflow automation, running on the k3s
cluster and managed by Flux CD. This folder holds the manifests for the n8n
service itself. For what runs on top of it, see [Workflows](#workflows).

## Overview

| Component | Details |
|-----------|---------|
| Image | `docker.n8n.io/n8nio/n8n`, version bumps proposed by Renovate |
| Namespace | `n8n` |
| Database | CloudNativePG Postgres cluster, 2 instances |
| File storage | `n8n-data` Longhorn volume, mounted at `/home/node/.n8n` |
| Port | 3008 |
| Deployed by | Flux, via the apps kustomization |

## Configuration

- **ConfigMap:** n8n and database connection settings (non-sensitive).
- **Secrets:** database credentials, SOPS-encrypted with age and decrypted by
  Flux. No plaintext secrets live in this repo.
- **Encryption key:** n8n encrypts stored credentials with a key that is not in
  the database. Keep a copy outside the cluster. A restored database is not
  usable for credentials without it.

## Storage and backups

| What | How | Where |
|------|-----|-------|
| Postgres database | CloudNativePG with the Barman Cloud plugin: daily base backups, 30-day retention | S3 |
| `n8n-data` volume | Longhorn recurring backup job (opt-in `backup` group) | S3 |

The CNPG backups have been through a test restore. Before any risky change,
also take a manual dump in pgAdmin and a Longhorn snapshot of the three n8n
volumes, since snapshots and backups for a running database volume are
crash-consistent, not clean.

## Upgrades

Renovate opens a PR for each new n8n image. n8n runs database migrations on
startup, and they may not be reversible, so n8n updates are reviewed and
merged by hand, never auto-merged.

1. Confirm all Longhorn volumes are healthy and the CNPG cluster is ready.
2. Back up the database (pgAdmin or `pg_dump`, Custom format) and download it.
3. Take Longhorn snapshots of `n8n-data` and both database volumes.
4. Merge the PR and let Flux roll the deployment.
5. Watch the pod logs until the migrations finish and the editor is reachable.
6. Open a credential and test it, then check that workflows are still listed.

**Rollback:** scale the deployment to zero, revert the image tag in Git, and
restore the database from the backup. Reverting the tag alone is not enough
once migrations have run.

## Workflows

| Workflow | Purpose |
|----------|---------|
| Renovate PR auto-merger | Reviews Renovate PRs on the Forgejo repo with Claude and merges only low-risk patch updates. Everything else is reported on Telegram for manual review. |

Credentials used by workflows are stored in n8n, not in Git. Scrub credential
IDs and chat IDs from any exported workflow JSON before sharing it.

## Known warnings

These appear in the startup log and are harmless here:

- The Python task runner fails to start in internal mode because Python is not
  installed in the image. It only matters if you use Python Code nodes.
- n8n prints deprecation notices for runner and compression settings. Set the
  variables explicitly in the ConfigMap to keep current behavior when defaults
  change.
