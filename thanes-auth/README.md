# thanes-auth — Kubernetes manifests for authentication-v2

Manifests for the SMtrack auth service (tenant/department version), source repository
`smtrack-authentication`. Argo CD syncs this directory through `kustomization.yaml`.

| File | Synced by Argo CD | What |
|---|---|---|
| `kustomization.yaml` | — | The list Argo CD applies |
| `configmap.yaml` | yes | `authentication-v2-config`: non-secret settings |
| `deploy-v2.yaml` | yes | Deployment (2 replicas), Service and PodDisruptionBudget `authentication-v2` |
| `purge-cronjob.yaml` | yes | Weekly purge of expired soft-deleted records; starts in `--dry-run` |
| `import-legacy-job.yaml` | **no** | One-off import from the old database, applied by hand at cutover |
| `secrets.example.yaml` | **no** | Template of the `authentication-v2-secrets` Secret |
| `secrets/` | **no** | Real Secret manifests; ignored by Git (`.gitignore`), exist only on the operator's machine |
| `argocd/application.yaml` | **no** | The Argo CD Application, applied once |

The auth repository's CI (`.github/workflows/main.yaml`) rewrites the `image:` lines of
`deploy-v2.yaml`, `purge-cronjob.yaml` and `import-legacy-job.yaml` after each push to its `main`.

## First install

```bash
# 1. Secrets: fill in the CHANGE-ME values in secrets/ (never committed), then
kubectl apply -f secrets/authentication-v2-secrets.yaml
kubectl apply -f secrets/authentication-legacy-db.yaml   # only for the cutover import

# 2. Fill in the TODOs in configmap.yaml (TRUSTED_PROXIES, REDIS_HOST, S3_*, IMAGE_BASE_URL) and commit

# 3. Hand the directory to Argo CD
kubectl apply -n argocd -f argocd/application.yaml
kubectl -n smtrack rollout status deploy/authentication-v2

# 4. On a database that did not come from the import: create the first SUPER user once
kubectl -n smtrack run auth-bootstrap --rm -it --restart=Never \
  --image=siamatic/smtrack-auth-service:3.0.0 \
  --overrides='{"spec":{"containers":[{"name":"auth-bootstrap","image":"siamatic/smtrack-auth-service:3.0.0","stdin":true,"tty":true,"command":["./bootstrap","--username","<name>"],"envFrom":[{"secretRef":{"name":"authentication-v2-secrets"}},{"configMapRef":{"name":"authentication-v2-config"}}]}]}}'
```

Cutover, import and rollback: `docs/ops/cutover-runbook.md` in the auth repository.
At cutover, enable RabbitMQ by uncommenting the `RABBITMQ` block in `deploy-v2.yaml`.

Check locally before pushing: `kubectl kustomize .`
