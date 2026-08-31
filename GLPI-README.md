# GLPI — README

Minimum steps to deploy GLPI + MariaDB on K3s.

---

## 1. Prerequisites — Secret Creation

GLPI and MariaDB share one Secret, `glpi-secrets`, in the `glpi`
namespace. It must exist **before** the ArgoCD Application first syncs —
ArgoCD creates the namespace (`CreateNamespace=true`) but will NOT create
this secret. Both pods will crash-loop until it's present.

The secret carries two key sets that must agree on the same values — one
set for the MariaDB container's own bootstrap, one for GLPI's connection
to it:

    openssl rand -hex 12   # -> root password
    openssl rand -hex 12   # -> glpi db user password

    kubectl create namespace glpi

    kubectl create secret generic glpi-secrets \
      --namespace glpi \
      --from-literal=MYSQL_ROOT_PASSWORD='<PLACEHOLDER_ROOT_PW>' \
      --from-literal=MYSQL_DATABASE=glpidb \
      --from-literal=MYSQL_USER=glpi \
      --from-literal=MYSQL_PASSWORD='<PLACEHOLDER_GLPI_PW>' \
      --from-literal=GLPI_DB_NAME=glpidb \
      --from-literal=GLPI_DB_USER=glpi \
      --from-literal=GLPI_DB_PASSWORD='<PLACEHOLDER_GLPI_PW>'

`MYSQL_PASSWORD` and `GLPI_DB_PASSWORD` must be the identical value —
one is read by the MariaDB container to create the user, the other by
GLPI to connect as that user.

Save these in Ansible Vault the same way `vault_k3s_cluster_token` is
stored, so they're recoverable and not just sitting in shell history.

---

## 2. Deploying

GLPI is deployed as an ArgoCD Application (`gitops/apps/glpi.yaml`).
Trigger the sync once the namespace and secret above exist.

Auto-install runs automatically on first boot since all five
`GLPI_DB_*` variables are present — no web installation wizard needed.

---

## 3. Verification Commands

Confirm pod health

    kubectl -n glpi get pods

Confirm GLPI actually connected and auto-installed

    kubectl -n glpi logs deploy/glpi | grep -i -E "install|database"

Confirm MariaDB is healthy

    kubectl -n glpi exec -it mariadb-0 -- healthcheck.sh --connect

---

## 4. First-Run: Log In and Change Default Credentials

Default GLPI accounts after auto-install: `glpi`/`glpi` (super-admin),
`tech`/`tech`, `normal`/`normal`, `post-only`/`postonly`. Log in as
`glpi`/`glpi` immediately and:

1. Change the `glpi` account password.
2. Disable or delete the other three default demo accounts.
3. Set your own timezone and locale under Setup > General.

---

## 5. Post-Deploy: SMTP for Ticket Notifications

Not env-var driven — configured entirely through the web UI:

1. Setup > Notifications > enable "Send notifications".
2. Administration > Notification templates, or the "Send by" setting,
   to point at your SMTP relay (Setup > General > Notifications, "Send
   emails using" > SMTP), with your relay's host/port/credentials.
3. `mariadb-isolation`/`glpi-isolation` NetworkPolicy egress currently
   allows any destination on 587/465/25 — once you know the actual relay
   IP (SMTP provider or a self-hosted relay), narrow the `ipBlock` in
   `networkpolicy.yaml` to just that address.

---

## 6. Timezone Support (Optional)

If you want GLPI's timezone-aware date fields to work correctly:

    kubectl -n glpi exec -it mariadb-0 -- \
      mysql -u root -p"$(kubectl -n glpi get secret glpi-secrets -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d)" \
      -e "GRANT SELECT ON mysql.time_zone_name TO 'glpi'@'%'; FLUSH PRIVILEGES;"

    kubectl -n glpi exec -it deploy/glpi -- \
      /var/www/glpi/bin/console database:enable_timezones

---

## 7. Gotchas to Watch Out For

- **DNS**: `glpi.pearsalls.fr` won't resolve until Unbound split-horizon
  DNS is live on OPNsense. Until then, add a manual `/etc/hosts` entry
  pointing at a K3s node IP, or use
  `kubectl -n glpi port-forward svc/glpi 8080:80`.
- **Image tag**: `glpi/glpi:10.0` is pinned as the current stable line —
  verify against https://hub.docker.com/r/glpi/glpi/tags before applying
  if it's been a while; GLPI 11 is still beta as of this writing and
  shouldn't be used for anything you actually rely on.
- **`strategy: Recreate`**: same reasoning as Vaultwarden — `glpi-data`
  is a single RWO Ceph RBD volume, a rolling update would try to mount it
  twice and fail to schedule the new pod.
- **Marketplace plugins**: the default `/var/glpi` volume does NOT cover
  `/var/www/glpi/marketplace` on the 10.0.x image line. If you install
  plugins from the in-app marketplace, uncomment the second volumeMount
  in `glpi-deploy.yaml` first, or plugin data won't survive a pod
  restart.
- **Backup strategy**: a nightly `mysqldump` CronJob runs at 00:30,
  writing a gzipped SQL dump to a dedicated PVC (`mariadb-backup`), 7-day
  local retention. Velero's existing daily backup (01:00) sweeps that PVC
  up automatically. This is a logical (SQL-statement) backup, safe for a
  live database in a way Kopia's fs-backup alone would not have been —
  same reasoning as Vaultwarden's Postgres backup.
