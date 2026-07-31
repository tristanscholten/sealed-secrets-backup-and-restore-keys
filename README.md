# sealed-secrets-backup-and-restore-keys

Tiny Kubernetes backup for Bitnami Sealed Secrets controller private keys.

Deploy it once. It finds every Secret with the standard Sealed Secrets key label, groups backups by controller namespace, and writes them to **one or more** labeled nodes.

Every labeled node gets controller-specific local copies:

```text
/var/lib/sealed-secrets-backup/<controller-namespace>/sealed-secrets-keys.yaml
/var/lib/sealed-secrets-backup/<controller-namespace>/sealed-secrets-keys-<timestamp>.yaml
/var/lib/sealed-secrets-backup/<controller-namespace>/sealed-secrets-keys.latest.txt
```

This prevents multiple Sealed Secrets controllers from overwriting each other's hostPath backup files.

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

## How it works

- `sealed-secrets-backup-label-check` verifies at least one node has the configured label.
- The CronJob lists all labeled nodes.
- For every labeled node, the CronJob creates a short-lived worker Job pinned to that exact node.
- Each worker lists all namespaces that contain Sealed Secrets key Secrets.
- Each controller namespace gets its own hostPath subdirectory and S3 prefix.
- If `S3_BUCKET` is configured, each worker also uploads `latest`, timestamped backup, and metadata files to S3.

No private key material is printed by the scripts. Logs only include counts, paths, node names, namespaces, and upload destinations.

## Deploy node-local backups

Label one or more nodes to hold backups:

```bash
kubectl label node <node-name-1> sealed-secrets-backup=true
kubectl label node <node-name-2> sealed-secrets-backup=true
```

Deploy:

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

## Optional single-namespace mode

Default behavior is cluster-wide discovery of all Sealed Secrets key namespaces. That is the simplest and safest mode for clusters with multiple Sealed Secrets controllers.

If you deliberately want to back up only one controller namespace, add this env var to the CronJob:

```yaml
- name: SEALED_SECRETS_NAMESPACE
  value: kube-system
```

Most users should not set it.

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
s3://<bucket>/<prefix>/<controller-namespace>/<node>/sealed-secrets-keys.yaml
s3://<bucket>/<prefix>/<controller-namespace>/<node>/sealed-secrets-keys-<timestamp>.yaml
s3://<bucket>/<prefix>/<controller-namespace>/<node>/sealed-secrets-keys.latest.txt
```

Each labeled node uploads its own copy under the sanitized controller namespace and node name. The contents should be identical for a healthy cluster, but per-node paths make it obvious which nodes successfully wrote/uploaded a backup.

## Restore from node-local backup

On any backup node, copy this file to your admin machine:

```text
/var/lib/sealed-secrets-backup/<controller-namespace>/sealed-secrets-keys.yaml
```

After rebuilding the cluster, restore before expecting GitOps SealedSecrets for that controller to decrypt:

```bash
kubectl apply -f sealed-secrets-keys.yaml
kubectl -n <controller-namespace> rollout restart deploy/sealed-secrets-controller
```

Then sync GitOps again.

## Restore from S3-compatible storage

Download either the latest backup or a timestamped backup from one node prefix:

```bash
aws --endpoint-url https://minio.example.com \
  s3 cp \
  s3://my-sealed-secrets-backups/sealed-secrets-backup/prod/<controller-namespace>/<node>/sealed-secrets-keys.yaml \
  sealed-secrets-keys.yaml
```

For AWS S3, omit `--endpoint-url`:

```bash
aws s3 cp \
  s3://my-sealed-secrets-backups/sealed-secrets-backup/prod/<controller-namespace>/<node>/sealed-secrets-keys.yaml \
  sealed-secrets-keys.yaml
```

Apply it to the rebuilt cluster:

```bash
kubectl apply -f sealed-secrets-keys.yaml
kubectl -n <controller-namespace> rollout restart deploy/sealed-secrets-controller
```

Validate by applying or syncing a known-good `SealedSecret` and confirming the controller creates the target Secret.

## Defaults

- Backup schedule: daily at `03:17 UTC`
- Backup namespace: `sealed-secrets-backup`
- Controller discovery: all namespaces with `sealedsecrets.bitnami.com/sealed-secrets-key` Secrets
- Secret selector: `sealedsecrets.bitnami.com/sealed-secrets-key`
- Node label: `sealed-secrets-backup=true`
- Host path on every labeled node: `/var/lib/sealed-secrets-backup/<controller-namespace>`
- S3 upload: disabled until `S3_BUCKET` is set
