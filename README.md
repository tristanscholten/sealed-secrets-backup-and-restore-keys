# sealed-secrets-backup-and-restore-keys

Tiny Kubernetes backup for Bitnami Sealed Secrets controller private keys.

Deploy it once. It finds every Secret with the standard Sealed Secrets key label, groups backups by controller namespace, and writes them to at least one configured backup target.

Supported backup targets:

- node-local hostPath on one or more labeled nodes
- S3-compatible bucket endpoint
- both at the same time

If neither target is configured, the CronJob fails loudly.

## Node-local backup layout

Every labeled node gets controller-specific local copies:

```text
/var/lib/sealed-secrets-backup/<controller-namespace>/sealed-secrets-keys.yaml
/var/lib/sealed-secrets-backup/<controller-namespace>/sealed-secrets-keys-<timestamp>.yaml
/var/lib/sealed-secrets-backup/<controller-namespace>/sealed-secrets-keys.latest.txt
```

This prevents multiple Sealed Secrets controllers from overwriting each other's hostPath backup files.

## S3 backup layout

If `S3_BUCKET` is configured, backups are also uploaded to:

```text
s3://<bucket>/<prefix>/<controller-namespace>/<target>/sealed-secrets-keys.yaml
s3://<bucket>/<prefix>/<controller-namespace>/<target>/sealed-secrets-keys-<timestamp>.yaml
s3://<bucket>/<prefix>/<controller-namespace>/<target>/sealed-secrets-keys.latest.txt
```

`<target>` is the sanitized node name when node-local backups are enabled. If S3 is the only configured target, `<target>` is `cluster`.

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

- The CronJob checks configured targets.
- If one or more nodes have `sealed-secrets-backup=true`, it creates one worker Job pinned to each labeled node.
- If no nodes are labeled but `S3_BUCKET` is set, it creates one unpinned S3-only worker Job.
- Each worker lists all namespaces that contain Sealed Secrets key Secrets.
- Each controller namespace gets its own hostPath subdirectory and S3 prefix.
- If no labeled nodes and no `S3_BUCKET` exist, the CronJob fails without writing anything.

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

## Deploy S3-only backups

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

Then deploy normally:

```bash
kubectl apply -k .
```

No node label is required when `S3_BUCKET` is configured.

## Deploy both node-local and S3 backups

Configure `S3_BUCKET`, then also label one or more nodes. Each labeled node writes a hostPath backup and uploads its own S3 copy under that node name.

## Optional single-namespace mode

Default behavior is cluster-wide discovery of all Sealed Secrets key namespaces. That is the simplest and safest mode for clusters with multiple Sealed Secrets controllers.

If you deliberately want to back up only one controller namespace, add this env var to the CronJob:

```yaml
- name: SEALED_SECRETS_NAMESPACE
  value: kube-system
```

Most users should not set it.

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

Download either the latest backup or a timestamped backup from one prefix.

S3-only example:

```bash
aws --endpoint-url https://minio.example.com \
  s3 cp \
  s3://my-sealed-secrets-backups/sealed-secrets-backup/prod/<controller-namespace>/cluster/sealed-secrets-keys.yaml \
  sealed-secrets-keys.yaml
```

Node + S3 example:

```bash
aws --endpoint-url https://minio.example.com \
  s3 cp \
  s3://my-sealed-secrets-backups/sealed-secrets-backup/prod/<controller-namespace>/<node>/sealed-secrets-keys.yaml \
  sealed-secrets-keys.yaml
```

For AWS S3, omit `--endpoint-url`.

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
