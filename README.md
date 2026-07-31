# sealed-secrets-backup-and-restore-keys

Tiny Kubernetes backup for Bitnami Sealed Secrets controller private keys.

It backs up the controller TLS key Secrets from `kube-system` to one labeled node:

```text
/var/lib/sealed-secrets-backup/sealed-secrets-keys.yaml
```

Context7 source used: `/bitnami/sealed-secrets`. Bitnami docs say backup keys with:

```bash
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > main.key
```

and restore with:

```bash
kubectl apply -f main.key
kubectl delete pod -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

## Deploy

Pick exactly one node to hold the backup:

```bash
kubectl label node <node-name> sealed-secrets-backup=true
```

Deploy:

```bash
kubectl apply -k .
```

Run a backup now:

```bash
kubectl -n sealed-secrets-backup create job --from=cronjob/sealed-secrets-key-backup sealed-secrets-key-backup-manual
kubectl -n sealed-secrets-backup wait --for=condition=complete job/sealed-secrets-key-backup-manual --timeout=120s
```

If no node has `sealed-secrets-backup=true`, `sealed-secrets-backup-label-check` exits with an error and goes `CrashLoopBackOff`. The backup CronJob is also pinned to that label.

## Recover after cluster loss

On the backup node, copy this file to your admin machine:

```text
/var/lib/sealed-secrets-backup/sealed-secrets-keys.yaml
```

After rebuilding the cluster, restore before expecting GitOps SealedSecrets to decrypt:

```bash
kubectl apply -f sealed-secrets-keys.yaml
kubectl -n kube-system rollout restart deploy/sealed-secrets-controller
```

Then sync GitOps again.

## Defaults

- Backup schedule: daily at `03:17 UTC`
- Backup namespace: `sealed-secrets-backup`
- Sealed Secrets namespace: `kube-system`
- Secret selector: `sealedsecrets.bitnami.com/sealed-secrets-key`
- Node label: `sealed-secrets-backup=true`
- Host path: `/var/lib/sealed-secrets-backup`
