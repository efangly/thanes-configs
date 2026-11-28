# smtrack-v2 — Kubernetes manifests for the SMtrack v2 backend

IoT telemetry/log ingestion and notification backend (NestJS). Argo CD syncs this
directory through `kustomization.yaml`. TimescaleDB, Redis, RabbitMQ, MQTT and S3/MinIO
are external: nothing here installs them.

| File | Synced by Argo CD | What |
|---|---|---|
| `kustomization.yaml` | — | The list Argo CD applies, and the image tags |
| `configmap.yaml` | yes | `smtrack-v2-backend-config`: non-secret settings |
| `migrate-job.yaml` | yes | `prisma migrate deploy`, a Sync hook that runs before the Deployment |
| `deploy.yaml` | yes | Deployment (1 replica) and Service `smtrack-v2-backend` (http 3000, grpc 50051) |
| `archive-export-cronjob.yaml` | yes | Monthly archive and chunk drop; starts in `--dry-run` |
| `archive-cleanup-cronjob.yaml` | yes | Hourly cleanup of restored data |
| `legacy-reconcile-cronjob.yaml` | yes | Daily legacy sync check; starts suspended |
| `ingress.yaml` | yes | `https://api.siamatic.work/log`; TLS Secret `smtrack-v2-backend-tls` |
| `archive-backfill-job.yaml` | **no** | One-off archive backfill / verify, applied by hand |
| `legacy-backfill-job.yaml` | **no** | One-off metadata import from the old system, applied by hand |
| `secrets.example.yaml` | **no** | Template of the two Secrets |
| `secrets/` | **no** | Real Secret manifests; ignored by Git, exist only on the operator's machine |
| `argocd/application.yaml` | **no** | The Argo CD Application, applied once |

## Before the first sync

1. **Database:** PostgreSQL with TimescaleDB. The role in `DATABASE_URL` must be able to run
   the migrations, including `CREATE EXTENSION timescaledb` (or create the extension first).
2. **Buckets:** create the `*_S3_BUCKET` buckets; the app does not create them.
3. **RabbitMQ:** check that no old service consumes the queue names in `configmap.yaml`.
   Two consumers on one queue split its messages.
4. **ConfigMap:** resolve the `TODO`s in `configmap.yaml` and commit.
5. **ingress-nginx and issuer:** `thanes-auth-issuer` comes from `../thanes-auth`.
6. **CI** (backend repository, Settings → Secrets and variables → Actions):

   | Kind | Name | Value |
   |---|---|---|
   | variable | `GITOPS_REPOSITORY` | `efangly/thanes-configs` |
   | secret | `GITOPS_TOKEN` | fine-grained PAT for that repository, Contents: read and write |
   | secret | `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` | allowed to push `siamatic/smtrack-backend` and `-migrate` |
   | secret | `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` | notifications (optional) |

   Commit this directory **before** the CI deploy job runs, and let one CI run succeed so
   the images exist. If the Docker Hub repositories are private and the nodes have no
   credentials, add an `imagePullSecrets` entry to every workload.

## First install

```bash
# 1. Secrets: fill in secrets/ (never committed), then
kubectl apply -f secrets/smtrack-v2-backend-secrets.yaml
kubectl apply -f secrets/smtrack-v2-backend-legacy-db.yaml   # only for legacy sync

# 2. Hand the directory to Argo CD (after it is committed and pushed)
kubectl apply -n argocd -f argocd/application.yaml
kubectl -n smtrack rollout status deploy/smtrack-v2-backend
```

## Checks

```bash
kubectl -n smtrack get pods,jobs,cronjobs -l app.kubernetes.io/name=smtrack-v2-backend
kubectl -n smtrack logs job/smtrack-v2-backend-migrate
curl -fsS https://api.siamatic.work/log/health
curl -sS https://api.siamatic.work/log/health/archive    # 503 = archive overdue
kubectl -n smtrack get certificate smtrack-v2-backend-tls
kubectl -n smtrack port-forward svc/smtrack-v2-backend 50051   # Partner API (gRPC)
```

A Secret change needs `kubectl -n smtrack rollout restart deploy/smtrack-v2-backend`.

Check locally before pushing: `kubectl kustomize .`

## Archive cut-over

The migrations still set a 6-month `add_retention_policy`. Follow "Cut-over" in
`docs/archive-operations.md` of the backend repository, using `archive-backfill-job.yaml`
(backfill `--dry-run` → backfill → verify). Remove `--dry-run` from
`archive-export-cronjob.yaml` once the dry-run log looks right. While it stays, nothing
is exported by the CronJob, and the retention policy can drop unarchived data.

## Legacy sync

See `docs/legacy-sync-contract.md` in the backend repository. Queue setup →
`legacy-backfill-job.yaml` → set `LEGACY_SYNC_ENABLED: "true"` → unsuspend
`smtrack-v2-backend-legacy-reconcile`.

## Limits

- One replica only, until SSE fan-out moves out of the process.
- `OTEL_EXPORTER_OTLP_ENDPOINT` is empty: no traces or metrics, so the archive alerts do
  not fire. Watch for failed Jobs until a collector is set.
- The Partner API is plaintext and reachable only inside the cluster.
