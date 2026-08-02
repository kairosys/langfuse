# LangFuse k8s overlay

Configuration-only Kubernetes manifests for deploying **LangFuse v3** (observability platform) into a Kubernetes cluster. There is no application source code, package manager config, or build tooling in this repository — it contains only `k8s/` manifests plus the `.gitignore` and this `README.md`. Runtime secrets are excluded from version control (see [.gitignore](./.gitignore)).

## Overview

This overlay deploys the LangFuse v3 stack into a Kubernetes cluster on the `default` namespace, integrating with three external backing services:

- **PostgreSQL** — metadata store (organizations, projects, prompts).
- **ClickHouse** — analytical query engine for traces and events. Two endpoints are used: an HTTP endpoint (`CLICKHOUSE_URL`, port 8123) for runtime traffic, and a native TCP migration endpoint (`CLICKHOUSE_MIGRATION_URL`, port 9000) for `langfuse-worker` migrations.
- **Redis** — BullMQ queue backing the worker and async job processing.

Service discovery between components uses in-cluster DNS names (`postgres`, `clickhouse`, `redis`), so these services must run as siblings on the same namespace or be routed appropriately. The ingress exposes LangFuse at host `langfuse.localhost` via an Nginx Ingress controller using a prefix-trailing path `/`. Local resolution is required since `.localhost` names are not routable by public DNS (add to `/etc/hosts` or configure your local ingress).

The app image (`langfuse/langfuse:3`) serves the web UI and the OTLP ingestion endpoint, while `langfuse/langfuse-worker:3` consumes jobs from Redis. Images use floating major tags — for reproducible deploys consider pinning to a digest.

## Kubernetes Architecture

| Resource | Kind | Defined in | Namespace | Notes |
|---|---|---|---|---|
| langfuse-app | Deployment (`langfuse` + Service) | `deployment.yaml` (app container, replicas=1, port 3000; requests 512Mi/250m — limits 2Gi/1CPU) plus an inline ClusterIP Service on port 3000 and a prefix-trailing Ingress (`/` to service:3000 at `langfuse.localhost`, ingress class `nginx`) | `default` | Exposes the web UI and OTLP ingestion. Auto-postgres migration is enabled via env `LANGFUSE_AUTO_POSTGRES_MIGRATION_DISABLED=false`. Telemetry disabled with `TELEMETRY_ENABLED=false`. Pulls all runtime secrets from Secret named `langfuse-secret`. |
| langfuse-worker | Deployment (`worker`) | `deployment.yaml` (separate document — worker container, replicas=1; image `langfuse/langfuse-worker:3`, same secret consumer) plus resource requests 256Mi/100m with generous limits (1Gi memory / 1CPU cpu) and no exposed service | `default` | Runs BullMQ workers that consume from Redis. Has no Service of its own; reached only via internal job queue. Uses the same Secret named `langfuse-secret`. |
| langfuse-config | Secret | `secret.yaml` (gitignored via `.gitignore`) | `default` | Single Opaque secret consumed by both Deployments through `envFrom.secretRef`. Stores Postgres, ClickHouse URLs/credentials, Redis credentials, S3 event-upload config, and auth secrets. No ConfigMap is defined. |

No StatefulSet, PersistentVolumeClaim, or headless Service are present in these manifests — storage lifecycle for PostgreSQL, ClickHouse, and Redis is assumed to be provisioned externally. For local clusters you can bind-mount emptyDir volumes or run them as `Deployment` replicas with init containers; the app relies on `DATABASE_URL` and `CLICKHOUSE_*` variables alone (no PVC paths are referenced by these manifests).

## Network Ports & Protocols

| Resource | Port | Protocol | Purpose | Path / Detail |
|---|---|---|---|---|
| langfuse-app container (`app`) | 3000 | TCP | HTTP UI + API, incl. OTLP ingestion `/api/public/otel/v1/traces` — use `http://langfuse.localhost/api/` (no separate gRPC port exposed here) | served via Service `langfuse`, containerPort 3000 → servicePort 3000; Ingress routes host `/` to it |
| langfuse-worker container (`app:worker`) | none | TCP | BullMQ queue consumption only — not directly reachable from outside the pod network. Communication is outbound HTTP/SQL (Redis) and SQL (Postgres/ClickHouse). No Service is attached, so there are no service ports for the worker here. |

Only one inbound endpoint is exposed through the Ingress: `http://langfuse.localhost/` mapped to prefix `/`. The OTLP ingestion path is reached at `/api/public/otel/v1/traces` over HTTP on port 3000 behind that same ingress/host. If you need a distinct ClickHouse native TCP listener (e.g., for remote migrations), map `CLICKHOUSE_MIGRATION_URL=clickhouse://host:9000` so the migration client targets port 9000 while runtime HTTP keeps using `http://host:8123`.

## Configuration & Secret Reference

The following keys must be present in the gitignored Secret `k8s/langfuse-secret.yaml` (kind: Secret, name: langfuse-config). The app container disables telemetry and enables auto-postgres migration through env entries that are not part of the secret.

| Kind | Variable / Port key | Where set | Description |
|---|---|---|---|
| Secret key (`langfuse-config`) | `DATABASE_URL` (port 5432) | `k8s/langfuse-secret.yaml` → PostgreSQL connection string, e.g. `postgresql://postgres@postgres:5432/langfuse` — consumed by both app and worker via `envFrom.secretRef.name=langfuse-config`. |
| Secret key (`langfuse-config`) | `CLICKHOUSE_URL` (port 8123) | `k8s/langfuse-secret.yaml` → ClickHouse HTTP endpoint, e.g. `http://clickhouse:8123`; runtime query engine for traces/events. |
| Secret key (`langfuse-config`) | `CLICKHOUSE_USER` / `CLICKHOUSE_PASSWORD` | `k8s/langfuse-secret.yaml` → optional when ClickHouse auth is enabled; otherwise omit to avoid breaking an unauthenticated local instance. |
| Secret key (`langfuse-config`) | `CLICKHOUSE_CLUSTER_ENABLED=false` | `k8s/langfuse-secret.yaml` → disables clustered DDL for single-node local setups — set true in production clusters only if you provision a ClickHouse Keeper topology to match. |
| Secret key (`langfuse-config`) | `CLICKHOUSE_MIGRATION_URL` (port 9000) | `k8s/langfuse-secret.yaml` → native TCP migration URL such as `clickhouse://clickhouse:9000`; used only by the worker during schema bootstrap. Do not confuse with port 8123; migrations fail if they hit the HTTP listener instead of 9000, so keep both URLs aligned to the same server on different ports. |
| Secret key (`langfuse-config`) | `REDIS_HOST` / `REDIS_PORT=6379` / `REDIS_AUTH` | `k8s/langfuse-secret.yaml` → BullMQ queue backing for the worker process; must be reachable from within the cluster (typically a sibling service named redis on 6379). |
| Secret key (`langfuse-config`) | S3 event upload block: `LANGFUSE_S3_EVENT_UPLOAD_BUCKET`, `..._ENDPOINT` (http://rustfs:9000), `..._ACCESS_KEY_ID/SECRET_ACCESS_KEY`, `..._REGION=us-east-1`, `..._FORCE_PATH_STYLE=true` | `k8s/langfuse-secret.yaml` → configures where exported traces/events are spilled to an S3-compatible store (RustFS by convention); all keys must be set together or ingestion of large payloads may silently fail. |
| Secret key (`langfuse-config`) | `NEXTAUTH_SECRET`, then app auth: `SALT`, `ENCRYPTION_KEY` and `NEXTAUTH_URL=http://langfuse.localhost` | `k8s/langfuse-secret.yaml` → cryptographic secrets (JWT/ NextAuth signing), data-at-rest encryption salt/key for PII (prompt/variable encryption). These are high-value targets; treat like credentials, never log here. NEXTAUTH_URL overrides the host used in redirect URIs and must match your real ingress hostname to avoid auth callback loops. |
| Container env (`k8s/langfuse-deployment.yaml`) | `TELEMETRY_ENABLED=false` (port: n/a) | defined inline on the app container — turns off anonymized usage telemetry so no traffic leaves the cluster during local runs; overrides do NOT live in the secret, editing only the deployment will toggle it. |
| Container env (`k8s/langfuse-deployment.yaml`) | `LANGFUSE_AUTO_POSTGRES_MIGRATION_DISABLED=false` (port n/a) | defined inline on app container — when false triggers a one-shot Postgres schema migration at startup; useful for dev but verify `DATABASE_URL` in the secret points to an empty database first, otherwise migrations can fail mid-flight and leave the app unhealthy. |