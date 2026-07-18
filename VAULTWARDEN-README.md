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
- No Backups Yet (Deliberate): see the note at the bottom of
  `postgres-statefulset.yaml`. Velero's Kopia fs-backup is not
  file-consistency-safe for a live Postgres data directory. Don't assume
  this data is protected until either a Ceph CSI VolumeSnapshotClass or a
  `pg_dump` CronJob exists — track this the same way etcd-snapshot-to-S3
  was tracked before it was actually verified working.
