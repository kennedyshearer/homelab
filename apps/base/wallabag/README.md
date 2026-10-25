# Wallabag

Self-hosted read-it-later app, running on the homelab cluster and managed by Flux.

- **URL:** https://wallabag.kennedyhomelab.net
- **Image:** `wallabag/wallabag:2.6.14` (pinned; Renovate manages upgrades)
- **Database:** CloudNativePG cluster `wallabag-postgres-cluster` (Postgres), reached at `wallabag-postgres-cluster-rw`
- **Storage:** Longhorn PVC `wallabag-data`, mounted at `/var/www/wallabag/web/assets/images`
- **Backups:** CNPG Barman Cloud backups to S3 (the database is the part that matters; the images volume only holds downloaded article images)
- **Public access:** Cloudflare Tunnel, configured under `staging/wallabag/`

## Layout

| Path | Purpose |
|---|---|
| `apps/base/wallabag/` | Deployment, ConfigMap, Service, PVC, CNPG cluster and backup resources |
| `staging/wallabag/cloudflare.yaml` | Tunnel route for the wallabag hostname |
| `staging/wallabag/cloudflare-secret.yaml` | SOPS-encrypted Cloudflare credentials |
| `staging/wallabag/kustomization.yaml` | Wires the staging resources together |

## How startup works

The Deployment has an **init container** (`wallabag-db-init`) that runs before the app starts:

1. Rebuilds `parameters.yml` from the environment variables.
2. Waits for Postgres to accept connections.
3. Checks for the `wallabag_user` table.
   - **Missing** (empty database): runs `wallabag:install`, which creates the schema and the default `wallabag` / `wallabag` user.
   - **Present**: runs `doctrine:migrations:migrate`, so image upgrades update the schema. Safe to repeat.

This is needed because the image's own entrypoint only auto-populates Postgres when given superuser credentials and a database that doesn't exist yet. CNPG pre-creates the database, so that path is skipped and the app would otherwise return a `500` on an empty database.

## Configuration

Non-secret settings live in the ConfigMap `wallabag-configmap`:

| Variable | Value |
|---|---|
| `SYMFONY__ENV__DATABASE_DRIVER` | `pdo_pgsql` |
| `SYMFONY__ENV__DATABASE_HOST` | `wallabag-postgres-cluster-rw` |
| `SYMFONY__ENV__DATABASE_PORT` | `5432` |
| `SYMFONY__ENV__DATABASE_NAME` | `wallabag` |
| `SYMFONY__ENV__DOMAIN_NAME` | `https://wallabag.kennedyhomelab.net` |

The database user and password come from the Secret `wallabag-postgres-creds` (`secretKeyRef`). Editing the ConfigMap or Secret does not restart the pod on its own: run `kubectl rollout restart deployment/wallabag -n wallabag`.

User registration is closed. Accounts are created from the command line (below), not from a sign-up form.

## Create an admin user

Run the console as `nobody`, never as root. Root-owned cache files under `var/` break the app later with confusing permission errors.

```bash
kubectl exec -it -n wallabag deploy/wallabag -- \
  su -s /bin/sh nobody -c "/var/www/wallabag/bin/console fos:user:create --super-admin --env=prod"
```

It prompts for a username, email, and password. If the command rejects `--super-admin`, check the exact spelling with:

```bash
kubectl exec -it -n wallabag deploy/wallabag -- \
  su -s /bin/sh nobody -c "/var/www/wallabag/bin/console fos:user:create --help --env=prod"
```

### After a from-scratch rebuild (empty database)

`wallabag:install` recreates the default `wallabag` / `wallabag` account, whose password is public knowledge.

1. Log in through a port-forward so the public tunnel never serves the default credentials:
   ```bash
   kubectl port-forward -n wallabag svc/wallabag 8080:80
   ```
   Then open http://localhost:8080.
2. Create your own admin (command above) or via the user management page.
3. Log in as your own admin, confirm it works, **then** delete the default `wallabag` user.

A restore from the Barman backup brings the existing accounts back, so none of this is needed for a restore.

## Useful checks

```bash
# Is the schema there? (prints wallabag_user, or empty if missing)
kubectl exec -n wallabag wallabag-postgres-cluster-1 -c postgres -- \
  psql -d wallabag -c "SELECT to_regclass('public.wallabag_user')"

# What did the init container do on the last start?
kubectl logs -n wallabag deploy/wallabag -c wallabag-db-init

# Is the images volume mounted and writable?
kubectl exec -n wallabag deploy/wallabag -- df -h /var/www/wallabag/web/assets/images
kubectl exec -n wallabag deploy/wallabag -- \
  su -s /bin/sh nobody -c "touch /var/www/wallabag/web/assets/images/t && rm /var/www/wallabag/web/assets/images/t && echo writable"

# Application errors behind a 500
kubectl exec -n wallabag deploy/wallabag -- tail -n 50 /var/www/wallabag/var/logs/prod.log
```

## Gotchas

- **Single replica with a ReadWriteOnce volume** needs `strategy: Recreate`. With the default rolling update, a new pod on another node hits a Multi-Attach error and the rollout hangs.
- **Switching an existing Deployment to `Recreate`** can fail with `spec.strategy.rollingUpdate: Forbidden`, because the API server had defaulted that block. Setting `rollingUpdate: null` is in the manifest; if Flux still rejects it, delete the Deployment (not the PVC) and let Flux recreate it.
- **The container starts as root** (s6 supervises nginx and php-fpm, which drop to `nginx` and `nobody` for their workers). Do not set `runAsUser` or `runAsNonRoot`, and do not drop all capabilities. Pod-level `fsGroup: 65534` handles volume permissions instead.
- **Probes hit `/`**, which returns `500` when the database is empty. The init container prevents that; if probes are ever failing, check the init container logs and `prod.log` first.
- **Keep both image tags in sync.** The init container and the app container must run the same wallabag version, or the init container may migrate to a schema the app doesn't match.
- **Take a fresh CNPG backup before an image upgrade.** Migrations run automatically on start, and the only way to undo one is to restore.
