# Velero — README

This document covers the minimum operational steps required to deploy, verify, and restore backups using Velero on your K3s cluster.

---

1. Prerequisites — External Storage Setup

Create MinIO Bucket and User
1. Create Bucket: Create a dedicated bucket named k3s-state.
2. Create User: Create a user named velero and generate a secret key.
3. Attach Policy: Apply a read/write/delete policy restricted to k3s-state/*.

MinIO client commands
mc alias set homelab-minio http://10.0.10.15:9000 <ROOT_USER> <ROOT_PASSWORD>
mc mb homelab-minio/k3s-state
mc admin user add homelab-minio velero <PLACEHOLDER_VELERO_SECRET>
mc admin policy create homelab-minio velero-rw ./velero-policy.json
mc admin policy attach homelab-minio velero-rw --user velero

Create the Kubernetes Secret
Velero expects an AWS credentials-file format inside its secret. Run this before syncing the ArgoCD application.

cat <<EOF> /tmp/velero-credentials
[default]
aws_access_key_id=<PLACEHOLDER_VELERO_ACCESS_KEY>
aws_secret_access_key=<PLACEHOLDER_VELERO_SECRET_KEY>
EOF

kubectl create namespace velero

kubectl create secret generic velero-minio-credentials \
  --namespace velero \
  --from-file=cloud=/tmp/velero-credentials

rm /tmp/velero-credentials

---

2. Deploying

Velero is deployed as an ArgoCD Application (gitops/apps/velero.yaml). Trigger the ArgoCD sync once the velero namespace and credentials secret are present.

---

3. Essential Verification Commands


Confirm Pod Health
kubectl -n velero get pods

Verify Bucket Connectivity
kubectl -n velero get backupstoragelocation
(Status should show Available.)

Trigger a Manual Smoke Test
velero backup create test-backup-$(date +%s) --wait
velero backup describe test-backup-<timestamp>
(Confirm the status is Completed, not PartiallyFailed.)

Verify S3 Upload
mc ls homelab-minio/k3s-state/backups/

Test Restore Functionality
velero restore create --from-backup test-backup-<timestamp>
velero restore describe <restore-name>

---

4. Gotchas to Watch Out For

- Path-Style Routing: Set s3ForcePathStyle: true in Helm values for MinIO compatibility.
- HTTP Protocol Selection: If MinIO uses plain HTTP, ensure s3Url explicitly starts with http://.
- Secret Key Name: The --from-file= key name used when creating the secret (cloud) must match the key: cloud Helm value.
- No CSI Snapshotting: Local-path volumes are backed up by the Kopia-based node-agent running as a privileged DaemonSet.
- Unavailable Storage Status: A BackupStorageLocation stuck in Unavailable is usually a credential/secret configuration error.
- Verify True Deletions: Backup TTL policies prune Velero manifests but may not always delete S3 objects; audit bucket size periodically.
