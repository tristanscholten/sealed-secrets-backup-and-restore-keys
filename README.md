# sealed-secrets-backup-and-restore-keys

Tiny Kubernetes backup for Bitnami Sealed Secrets controller private keys.

**Namespace scope is intentional:** this backs up only Sealed Secrets key Secrets in the namespace where this backup is deployed. If you run multiple Sealed Secrets controllers in different namespaces, deploy one copy of this backup per controller namespace. Do not use one global backup deployment for all controllers.

It backs up the controller TLS key Secrets from the configured controller namespace to **one or more** labeled nodes. Every node with the configured backup label gets its own local copy:

```text
/var/lib/sealed-secrets-backup/sealed-secrets-keys.yaml
/var/lib/sealed-secrets-backup/sealed-secrets-keys-<timestamp>.yaml
/var/lib/sealed-secrets-backup/sealed-secrets-keys.latest.txt
```

Optionally, each node backup can also be uploaded to an S3-compatible bucket endpoint.

Context7 source used: `/bitnami/sealed-secrets`. Bitnami docs say backup keys with:

```bash
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > main.key
```

and restore with:

```bash
kubectl apply -f main.key
kubectl delete pod -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

## Namespace targeting

The backup reads Secrets only from its own deployment namespace by default:

```yaml
namespace: sealed-secrets-backup
```

That value lives in `kustomization.yaml`. Change it to the namespace that contains the Sealed Secrets controller you want to protect, for example:

```yaml
namespace: sealed-secrets-prod
```

For multiple controllers, deploy separate copies with different `namespace` values:

```text
controller in sealed-secrets-prod  -> backup deployed in sealed-secrets-prod
controller in sealed-secrets-dev   -> backup deployed in sealed-secrets-dev
```

This prevents a backup for one controller namespace from accidentally collecting another controller's key material.

## How it works

- `sealed-secrets-backup-label-check` verifies at least one node has the configured label.
- The CronJob lists all labeled nodes.
- For every labeled node, the CronJob creates a short-lived worker Job pinned to that exact node.
- Each worker reads only Sealed Secrets key Secrets from the backup namespace and writes the same backup to that node's host path.
- If `S3_BUCKET` is configured, the worker also uploads `latest`, timestamped backup, and metadata files to S3.

No private key material is printed by the scripts. Logs only include counts, paths, node names, and upload destinations.

## Deploy node-local backups

Label one or more nodes to hold backups:

```bash
kubectl label node <node-name-1> sealed-secrets-backup=true
kubectl label node <node-name-2> sealed-secrets-backup=true
```

Set `namespace:` in `kustomization.yaml` to the namespace of the Sealed Secrets controller you want to back up, then deploy:

```bash
kubectl apply -k .
```

Run a backup now:

```bash
kubectl -n sealed-secrets-backup create job \
  --from=cronjob/sealed-secrets-key-backup \
  sealed-secrets-key-backup-manual
kubectl -n sealed-secrets-backup wait \
  --for=condition=complete \
  job/sealed-secrets-key-backup-manual \
  --timeout=300s
```

If no node has `sealed-secrets-backup=true`, `sealed-secrets-backup-label-check` exits with an error and goes `CrashLoopBackOff`. The backup CronJob also fails loudly.

## Optional S3-compatible backup endpoint

The container image includes `aws`, so AWS S3, MinIO, Ceph RGW, Garage, Cloudflare R2, and other S3-compatible endpoints can be used.

Create credentials as a Secret. Use your real endpoint credentials; do not commit them:

```bash
kubectl -n sealed-secrets-backup create secret generic sealed-secrets-key-backup-s3 \
  --from-literal=AWS_ACCESS_KEY_ID='<access-key>' \
  --from-literal=AWS_SECRET_ACCESS_KEY='<secret-key>' \
  --from-literal=AWS_DEFAULT_REGION='us-east-1'
```

Configure the CronJob environment in `cronjob.yaml`:

```yaml
- name: S3_BUCKET
  value: my-sealed-secrets-backups
- name: S3_PREFIX
  value: sealed-secrets-backup/prod
- name: S3_ENDPOINT_URL
  value: https://minio.example.com
```

For normal AWS S3, leave `S3_ENDPOINT_URL` empty and set `AWS_DEFAULT_REGION` in the Secret.

S3 object layout:

```text
s3://<bucket>/<prefix>/<node>/sealed-secrets-keys.yaml
s3://<bucket>/<prefix>/<node>/sealed-secrets-keys-<timestamp>.yaml
s3://<bucket>/<prefix>/<node>/sealed-secrets-keys.latest.txt
```

Each labeled node uploads its own copy under its sanitized node name. The contents should be identical for a healthy cluster, but per-node paths make it obvious which nodes successfully wrote/uploaded a backup.

## Restore from node-local backup

On any backup node, copy this file to your admin machine:

```text
/var/lib/sealed-secrets-backup/sealed-secrets-keys.yaml
```

After rebuilding the cluster, restore before expecting GitOps SealedSecrets to decrypt:

```bash
kubectl apply -f sealed-secrets-keys.yaml
kubectl -n kube-system rollout restart deploy/sealed-secrets-controller
```

Then sync GitOps again.

## Restore from S3-compatible storage

Download either the latest backup or a timestamped backup from one node prefix:

```bash
aws --endpoint-url https://minio.example.com \
  s3 cp \
  s3://my-sealed-secrets-backups/sealed-secrets-backup/prod/<node>/sealed-secrets-keys.yaml \
  sealed-secrets-keys.yaml
```

For AWS S3, omit `--endpoint-url`:

```bash
aws s3 cp \
  s3://my-sealed-secrets-backups/sealed-secrets-backup/prod/<node>/sealed-secrets-keys.yaml \
  sealed-secrets-keys.yaml
```

Apply it to the rebuilt cluster:

```bash
kubectl apply -f sealed-secrets-keys.yaml
kubectl -n kube-system rollout restart deploy/sealed-secrets-controller
```

Validate by applying or syncing a known-good `SealedSecret` and confirming the controller creates the target Secret.

## Defaults

- Backup schedule: daily at `03:17 UTC`
- Backup/controller namespace: `sealed-secrets-backup` by default; change `namespace:` in `kustomization.yaml` per Sealed Secrets controller namespace
- Secret selector: `sealedsecrets.bitnami.com/sealed-secrets-key`
- Node label: `sealed-secrets-backup=true`
- Host path on every labeled node: `/var/lib/sealed-secrets-backup`
- S3 upload: disabled until `S3_BUCKET` is set
