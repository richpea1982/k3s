# Vaultwarden — README

Minimum steps to deploy Vaultwarden + Postgres on K3s.

---

1. Prerequisites — Secret Creation

Vaultwarden and Postgres share one Secret, `vaultwarden-secrets`, in the
`vaultwarden` namespace. It must exist **before** the ArgoCD Application
first syncs — ArgoCD creates the namespace (`CreateNamespace=true`) but
will NOT create this secret. Both pods will crash-loop until it's present.

Generate strong random values and store them in Ansible Vault the same
way `vault_k3s_cluster_token` is stored (`ansible-vault edit
ansible/group_vars/all/vault.yml`), so they're recoverable and not just
sitting in your shell history:

    openssl rand -hex 24   # -> vault_vaultwarden_postgres_password
    openssl rand -hex 32   # -> vault_vaultwarden_admin_token

Create the namespace and secret manually:

    kubectl create namespace vaultwarden

    kubectl create secret generic vaultwarden-secrets \
      --namespace vaultwarden \
      --from-literal=POSTGRES_USER=vaultwarden \
      --from-literal=POSTGRES_PASSWORD='<PLACEHOLDER_PG_PASSWORD>' \
      --from-literal=POSTGRES_DB=vaultwarden \
      --from-literal=DATABASE_URL='postgresql://vaultwarden:<PLACEHOLDER_PG_PASSWORD>@postgres:5432/vaultwarden' \
      --from-literal=ADMIN_TOKEN='<PLACEHOLDER_ADMIN_TOKEN>'

Note the password appears twice (once for Postgres' own auth, once
embedded in Vaultwarden's connection string) — they must match.

---

2. Deploying

Vaultwarden is deployed as an ArgoCD Application
(`gitops/apps/vaultwarden.yaml`). Trigger the sync once the namespace and
secret above exist.

---

3. First-Run: Create Your Account

`SIGNUPS_ALLOWED` defaults to `"true"` in the manifest so you can create
your first (and likely only) account. Once you have:

1. Visit `https://vaultwarden.local.lan` (see DNS note in
   `ingressroute.yaml`) and register your account.
2. Edit `vaultwarden-deploy.yaml`, set `SIGNUPS_ALLOWED` to `"false"`.
3. Commit + let ArgoCD self-heal sync the change.

Leaving signups open on a WireGuard-only route is a much smaller exposure
than leaving it open publicly, but there's no reason to leave it open
longer than it takes to make one account.

---

4. Essential Verification Commands

Confirm Pod Health

    kubectl -n vaultwarden get pods

Confirm Postgres Is Actually Being Used (not falling back to SQLite)

    kubectl -n vaultwarden logs deploy/vaultwarden | grep -i postgres

Admin Panel (uses ADMIN_TOKEN from the secret)

    https://vaultwarden.local.lan/admin

---

5. Gotchas to Watch Out For

- Secret Key Names Are Exact: Vaultwarden reads `DATABASE_URL` and
  `ADMIN_TOKEN` directly as env vars via `envFrom` — a typo'd key means a
  crash-loop with no obviously-related error.
- `strategy: Recreate`: the Deployment intentionally does NOT use rolling
  updates. `vaultwarden-data` is a single RWO Ceph RBD volume — a
  rolling update would try to mount it into two pods simultaneously and
  fail to schedule the new one.
- DNS: `vaultwarden.local.lan` won't resolve until Unbound split-horizon
  DNS is live on OPNsense (still planned). Until then, either add a
  manual host entry pointing at a K3s node IP, or use
  `kubectl -n vaultwarden port-forward svc/vaultwarden 8080:80` for
  interim access.
- Backup Strategy: a nightly `pg_dump` CronJob runs at 00:30, writing a
  gzipped SQL dump to a dedicated PVC (`postgres-backup`), 7-day local
  retention. Velero's existing daily backup (01:00) sweeps that PVC up
  automatically — no Velero config changes needed. This is a logical
  (SQL-statement) backup, transactionally consistent by construction, not
  a filesystem-level copy — safe for a live database in a way Kopia's
  fs-backup alone would not have been. Ceph CSI snapshotting (faster
  recovery point, true block-level) remains a planned follow-up, not yet
  built.

Confirm a dump actually ran:

    kubectl -n vaultwarden get cronjob postgres-backup
    kubectl -n vaultwarden get jobs -l job-name=postgres-backup
    kubectl -n vaultwarden exec -it postgres-0 -- sh   # or a throwaway pod
    # then inside: ls -lh /backup   (if mounted) or check via the CronJob's pod logs

Trigger one manually rather than waiting for 00:30:

    kubectl -n vaultwarden create job --from=cronjob/postgres-backup postgres-backup-manual

---

6. Restoring From a Dump

Same scratch-database-then-rename pattern used for the original LXC
migration — verify before touching anything live, never restore directly
into the production database:

    # 1. Copy the dump out of the backup PVC (via a throwaway pod, or
    #    from wherever Velero has archived it in MinIO if restoring after
    #    a full PVC loss)

    # 2. Create a scratch database
    kubectl -n vaultwarden exec -it postgres-0 -- psql -U vaultwarden -d vaultwarden \
      -c "CREATE DATABASE vaultwarden_restore_test OWNER vaultwarden;"

    # 3. Load the dump into it
    gunzip -c vaultwarden-<timestamp>.sql.gz | \
      kubectl -n vaultwarden exec -i postgres-0 -- psql -U vaultwarden -d vaultwarden_restore_test

    # 4. Verify with a throwaway Vaultwarden pod pointed at the restored
    #    scratch DB (same pattern as the migration verification step),
    #    confirm real vault data is present and readable

    # 5. Only once confirmed: cut over via the same rename procedure used
    #    during the original migration (lock connections, rename, restart)
