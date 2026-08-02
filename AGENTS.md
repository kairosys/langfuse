# LangFuse k8s overlay

Configuration-only overlay: 2 YAML files defining 4 Kubernetes resources plus 1 gitignored Secret.
No source code, package manager, build tooling, or tests. Do not expect npm/yarn/README artifacts.

## Files

| File | Resources |
|---|---|
| `k8s/langfuse-deployment.yaml` | Service (ClusterIP, port 3000) + Deployment (`langfuse/langfuse:3`) + Ingress (nginx, `langfuse.localhost`) |
| `k8s/langfuse-worker-deployment.yaml` | Deployment (`langfuse/langfuse-worker:3`) |
| `k8s/langfuse-secret.yaml` | Secret (gitignored, contains all env vars) |

Both deployments reference the same `langfuse-secret` via `envFrom.secretRef`.

## Namespace

All resources go into `default`.

## Applying changes

```bash
# schema validation only (no cluster needed)
kubectl apply --dry-run=client -f k8s/

# apply to cluster
kubectl apply -f k8s/
```

## Secret keys (in gitignored `k8s/langfuse-secret.yaml`)

Do not remove existing keys — the app will fail silently at runtime if they're missing:

- `DATABASE_URL` (PostgreSQL)
- `CLICKHOUSE_*` — HTTP host on port `8123`, native migrations on `clickhouse://host:9000` (don't confuse the two)
- `REDIS_HOST`, `REDIS_PORT`, `REDIS_AUTH`
- `SALT`, `ENCRYPTION_KEY`, `NEXTAUTH_SECRET`, `NEXTAUTH_URL` (must match the ingress host, else auth redirect loops)
- `LANGFUSE_S3_EVENT_UPLOAD_*` (S3 block)

Worker connects to Redis (BullMQ) and S3 by sibling service names (`redis` / `rustfs`) in the same namespace.

## Gotchas

- `langfuse.localhost` ingress host is not publicly routable — resolve via `/etc/hosts` or ingress controller.
- Image tags are `:3` (floating major). Override `image:` for stability; don't rely on upstream tag immutability.
- App deployment has `LANGFUSE_AUTO_POSTGRES_MIGRATION_DISABLED: "false"` — it runs migrations on startup. Ensure PostgreSQL is healthy before rolling updates, or migrations can fail mid-flight.
- `TELEMETRY_ENABLED: "false"` is set inline (not in the Secret).
- README.md prose calls the Secret `langfuse-config`, but the actual manifest name is `langfuse-secret` — trust the YAML.