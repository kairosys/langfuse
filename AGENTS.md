# LangFuse k8s overlay

Local workspace is a configuration-only overlay: three Kubernetes manifests (one service/ingress per file). There is no source code, package manager config, or build tooling in this repository; do not expect npm/yarn/test/README artifacts. The `.gitignore` excludes `k8s/*-secret.yaml` and `data/`.

## Cluster layout

| Resource / image | Namespace | Notes |
|---|---|---|
| Service + Deployment (app) `langfuse/langfuse:3`, host `langfuse.localhost` | `furseal` | Ingress class `nginx`, path `/` prefix-trail. Host is a `.localhost` name — resolve it via `/etc/hosts` or an ingress controller; it is not routable on its own. |
| Worker Deployment `langfuse/langfuse-worker:3` | `furseal` | Uses the same Secret; resource requests are small (256Mi/100m) but limits generous (1Gi/1CPU). |

## Environment / connectivity assumptions

All runtime secrets and service URLs live in `.gitignore`-d `k8s/langfuse-secret.yaml`, so they do **not** appear inline in the non-secret manifests. When editing or validating, keep these names present in that Secret:

- `DATABASE_URL` (PostgreSQL), `CLICKHOUSE_*` (`http://` host on `8123`; native `clickhouse://host:9000` for migrations — do not confuse port 8123 vs 9000), `REDIS_HOST`/`REDIS_PORT`/`REDIS_AUTH`, `SALT`, `ENCRYPTION_KEY`, `NEXTAUTH_SECRET`, and S3 block (`LANGFUSE_S3_EVENT_UPLOAD_*`).
- Workers connect to Redis (BullMQ) and S3 by service names like `redis` / `rustfs`; these are sibling services in the same namespace, not defined here.

## Applying changes locally

```bash
# validate only this overlay's manifests against k8s schemas (no cluster needed for schema check)
kubectl apply --dry-run=client -f .   # from repo root; catches Secret/namespace name drift

# apply to a real/trial furseal namespace
kubectl apply -n furseal -f .
```

Image tags are pinned to `:3` (floating-major-tag style, not SHAs). If stability is required beyond overlay edits, override by editing each container `image:` and re-applying rather than relying on upstream tag mutability.

## Operational gotchas

- Removing rows from `.gitignore`-d `langfuse-secret.yaml` silently breaks the app at runtime (the Secret simply becomes empty); there is no compile-time check in-repo to catch it.
- Changing the Ingress host requires updating DNS/hosts entries locally since the cluster does not own a public domain here.