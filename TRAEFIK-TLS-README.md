# Traefik — TLS via ACME DNS-01 (Cloudflare) — README

Enables Traefik to issue real, publicly-trusted certificates for internal-
only hosts (e.g. `vault.pearsalls.fr`) that are never publicly reachable
and never get a Cloudflare Tunnel entry.

---

1. Prerequisites — Cloudflare API Token

Create a **scoped** token — not your Global API Key, and not the existing
Tunnel token (different purpose, different blast radius if leaked):

1. Cloudflare dashboard → My Profile → API Tokens → Create Token
2. Use the "Edit zone DNS" template
3. Restrict "Zone Resources" to the specific zone: `pearsalls.fr` only
4. Create, copy the token (shown once)

Create the Secret manually — same pattern as `velero-minio-credentials`
and `vaultwarden-secrets`, not committed to Git:

    kubectl create secret generic traefik-cloudflare-dns-token \
      --namespace kube-system \
      --from-literal=CF_DNS_API_TOKEN='<PLACEHOLDER_TOKEN>'

---

2. Validate on Staging Before Going Live

Let's Encrypt's production CA has a hard rate limit — 50 certificates per
registered domain per week. If DNS-01 propagation, token scope, or the
resolver config is wrong, you can burn through retries fast while
debugging and lock yourself out of new certs for that domain for days.

Before syncing for real:

1. Uncomment the `caserver` staging line in `gitops/apps/traefik.yaml`
2. Sync, confirm the whole flow completes — check:

       kubectl -n kube-system logs deploy/traefik | grep -i acme

   Look for a successful certificate obtained message referencing the
   staging CA, not an error.
3. Once confirmed, comment the staging line back out (defaults to
   production automatically) and re-sync.

A staging cert will NOT be browser-trusted — that's expected, it's only
proving the DNS-01 mechanics work end-to-end before spending a real
attempt.

---

3. Essential Verification Commands

Confirm the resolver initialized without error

    kubectl -n kube-system logs deploy/traefik | grep -i certresolver

Confirm a cert was actually issued for a given host

    kubectl -n kube-system logs deploy/traefik | grep -i "vault.pearsalls.fr"

Confirm acme.json persisted (survives pod restarts)

    kubectl -n kube-system exec deploy/traefik -- cat /data/acme.json | head -20

---

4. Gotchas to Watch Out For

- Wrong Cloudflare Token Type: `CF_DNS_API_TOKEN` (scoped token) and the
  older `CF_API_EMAIL` + `CF_API_KEY` (Global Key) pair are NOT
  interchangeable in the underlying `lego` ACME library Traefik uses —
  only set `CF_DNS_API_TOKEN`, don't mix both.
- No Public DNS Record Needed: DNS-01 only requires the ability to create
  a `_acme-challenge.<host>` TXT record — it does not require an A/CNAME
  record to exist. Don't create one; leaving it absent is intentional and
  keeps the hostname unresolvable outside your local /etc/hosts override.
- acme.json Persistence: if `persistence.enabled` ever gets accidentally
  set to `false` or the PVC is lost, every cert gets re-requested on next
  restart — fine occasionally, a problem if it happens repeatedly against
  production and trips the rate limit.
- Certificate Transparency Logs Are Public: once a production cert is
  issued, `vault.pearsalls.fr` becomes visible on crt.sh and similar CT
  log search tools. Not a security hole — nothing is reachable without
  Tailscale/WireGuard — but expect automated recon scanners to notice the
  hostname exists.
- Ceph RBD Doesn't Honor fsGroup: fresh RBD PVCs come back `root:root`
  owned regardless of the pod's `fsGroup` — Traefik's non-root container
  (UID 65532) can't write `acme.json` without the `volume-permissions`
  init container fixing ownership first. Symptom: `permission denied`
  opening `/data/acme.json`, and the file may not even exist yet.
  Diagnostic: `kubectl get csidriver rbd.csi.ceph.com -o yaml` — check
  `fsGroupPolicy`; if it's not `File`, this will bite any future
  RBD-backed workload that runs as non-root, not just Traefik.
- One Resolver, Many Hosts: `certResolver: cloudflare` is reusable as-is
  for every future internal `*.pearsalls.fr` app (Grafana, Dozzle,
  BenToPDF) — just reference it in each new IngressRoute's `tls` block,
  no further Traefik-side changes needed.
