# Seafile (custom images) — README

Deploys Seafile Community Edition on K3s using
[`ggogel/seafile-containerized`](https://github.com/ggogel/seafile-containerized)'s
images (`seafile-server`, `seahub`, `seahub-media`) instead of the
official all-in-one image or Seafile's own bundled Caddy — Traefik does
the path-splitting here instead, and MariaDB runs on Ceph RBD (not a
Docker volume) with the library data itself on NFS.

Chosen over building fully from source because Seafile 13 dropped
non-Docker manual installation entirely; ggogel's project already
solves the from-source packaging problem with actively maintained,
Alpine-based, security-conscious images — adapting it is faster and
lower-maintenance than re-deriving a from-source build against a moving
upstream target, while still giving full control over every layer
above the base images themselves (ingress, storage, secrets, backups).

---

## 1. Prerequisites

### NAS export (infra-homelab repo, Ansible)

The `seafile` ZFS dataset must be NFS-exported to VLAN20 and have its
`data`/`avatars`/`custom` subdirectories created before the first sync —
this is the `nas_setup` role change bundled alongside this manifest set.
Re-run it against the NAS before syncing this ArgoCD Application:

    ansible-playbook -i hosts/hosts.ini nas_setup.yml

Confirm the export actually took effect:

    ssh <nas-host> exportfs -v | grep seafile

### Secret creation

Everything in this deployment shares one Secret, `seafile-secrets`, in
the `seafile` namespace. It must exist **before** the ArgoCD Application
first syncs — ArgoCD creates the namespace (`CreateNamespace=true`) but
will NOT create this secret. mariadb, seafile-server, and seahub will
all crash-loop until it's present.

Generate strong random values and store them in Ansible Vault the same
way `vault_k3s_cluster_token` is stored:

    openssl rand -hex 24   # -> vault_seafile_mysql_root_password
    openssl rand -hex 24   # -> vault_seafile_admin_password

Create the namespace and secret manually:

    kubectl create namespace seafile

    kubectl create secret generic seafile-secrets \
      --namespace seafile \
      --from-literal=mysql-root-password='<PLACEHOLDER_MYSQL_ROOT_PASSWORD>' \
      --from-literal=admin-email='<your-admin-email>' \
      --from-literal=admin-password='<PLACEHOLDER_ADMIN_PASSWORD>'

Note `mysql-root-password` is read by **both** mariadb's
`MYSQL_ROOT_PASSWORD` and seafile-server's `DB_ROOT_PASSWD` — same key,
referenced twice, so there is no way for the two to drift out of sync.

---

## 2. Version pin — verify before first sync

The manifests pin `ggogel/seafile-server:11.0.13`,
`ggogel/seahub:11.0.13`, and `ggogel/seahub-media:11.0.13` — matched
against what their `master` branch compose file referenced at the time
these manifests were written, not necessarily their current latest tag.
Check Docker Hub before syncing for the first time:

    https://hub.docker.com/r/ggogel/seafile-server/tags
    https://hub.docker.com/r/ggogel/seahub/tags
    https://hub.docker.com/r/ggogel/seahub-media/tags

All three image tags must match exactly — mixing versions across
seafile-server/seahub is not supported upstream.

---

## 3. Deploying

Seafile is deployed as an ArgoCD Application
(`gitops/apps/seafile.yaml`). Trigger the sync once the NAS export and
the secret above exist.

---

## 4. First-Run

1. Visit `https://seafile.pearsalls.fr` (resolve manually via
   `/etc/hosts` pointing at a K3s node IP, or WireGuard once that
   migration lands — no public DNS record, same pattern as
   `vault.pearsalls.fr`).
2. Log in with the `admin-email` / `admin-password` from the secret.
3. **Important — `SEAFILE_URL` is a one-time value.** Per ggogel's own
   docs, changing the `SEAFILE_URL` env var after first deployment has
   no effect, because the value gets written into the database on first
   boot and takes priority over the manifest from then on. To change it
   later, use the System Admin section of the web UI instead of editing
   this manifest. If file upload/download breaks after any future
   domain change, this is the first thing to check.

---

## 5. Essential Verification Commands

Confirm pod health:

    kubectl -n seafile get pods

Confirm seafile-server and seahub can both reach mariadb:

    kubectl -n seafile logs deploy/seafile-server
    kubectl -n seafile logs deploy/seahub

Confirm the fileserver path-split is actually working (should return
Seafile's fileserver banner, not a 404 or the seahub login page):

    curl -k https://seafile.pearsalls.fr/seafhttp/protocol-version

---

## 6. Gotchas to Watch Out For

- **No DB env vars on seahub — this is intentional.** seahub reads its
  database connection details from config files seafile-server already
  wrote into the shared `/shared` volume on its own first boot. Do not
  add `DB_HOST`/`DB_ROOT_PASSWD` to the seahub container on the
  assumption they're missing.
- **`strategy: Recreate` on both seafile-server and seahub**: both mount
  the same RWX `seafile-data` PVC. A rolling update briefly running two
  pod generations against it risks the same class of conflict as
  Vaultwarden/Traefik's single-RWO-volume Deployments, even though the
  access mode here is RWX rather than RWO — the application itself is
  not designed for concurrent multi-instance writers.
- **`/media/*` routing**: Traefik's IngressRoute sends `/media/avatars`
  and `/media/custom` directly to `seahub-media`, and `/seafhttp/*`
  (prefix stripped) to `seafile-server`. Everything else falls through
  to `seahub`. If avatars/custom logos stop rendering after any Traefik
  or middleware change, check the IngressRoute priorities first —
  Traefik does not reliably auto-prioritize `PathPrefix` specificity
  over a bare `Host()` match, which is why priorities are pinned
  explicitly rather than left to default ordering.
- **CrowdSec bouncer included on this route** — Vaultwarden and ArgoCD
  don't have it yet per the tracked backlog, so this jumps ahead of
  those. Remove the `crowdsec-bouncer` middleware reference from
  `ingressroute.yaml` if you'd rather roll it out to Seafile in the same
  batch as the others instead of ahead of them.
- **NFS PV paths must exist before first sync** — Kubernetes' static NFS
  PV type does not create the remote directory itself; if
  `/mnt/storage/seafile/{data,avatars,custom}` don't exist on the NAS,
  pods will hang at `ContainerCreating`. The bundled `nas_setup` Ansible
  change creates these idempotently — confirm it actually ran.
- **Backup strategy**: a nightly `mysqldump --all-databases` CronJob runs
  at 00:15 (ahead of Vaultwarden's 00:30 and Velero's 01:00), writing a
  gzipped SQL dump to a dedicated PVC (`mariadb-backup`), 7-day local
  retention, swept up by Velero's existing daily backup automatically.
  This is a logical, transactionally-consistent backup — safe for a live
  database in a way Kopia's fs-backup directly against the MariaDB Ceph
  RBD volume would not be. `--all-databases` is deliberate here (unlike
  Vaultwarden/GLPI's single named database) since ggogel's images don't
  document a fixed, version-stable list of database names created under
  the shared root credentials.
- **Container UID/GID unconfirmed**: the NFS export subdirectories are
  created with `0777` as a pragmatic starting point (relying on
  `no_root_squash` on the export). Worth tightening to the actual
  container UID once confirmed via `kubectl -n seafile exec deploy/seafile-server -- id`.

---

## 7. Restoring From a Dump

Same scratch-database-then-verify discipline as Vaultwarden's Postgres
restore procedure — never load a dump directly into the live database:

    # 1. Copy the dump out of the backup PVC (via a throwaway pod, or
    #    from wherever Velero has archived it in MinIO)

    # 2. Load into a scratch MariaDB instance (not the live one) and
    #    verify the expected databases (ccnet_db, seafile_db, seahub_db,
    #    or whatever this version actually created) are present and
    #    queryable

    # 3. Only once confirmed: restore into the live mariadb pod's
    #    volume during a maintenance window, with seafile-server and
    #    seahub scaled to zero first
