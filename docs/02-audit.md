# 02 - Audit

Date: 2026-11-19
Inputs: `legacy-compose/docker-compose.yml`, upstream commit `63e9150`, Kompose 1.38.0 output (`k8s-kompose-raw/`).

## Service inventory
| Service | Source | Container ports | Depends on | Stateful | Kubernetes object |
|---------|--------|-----------------|------------|----------|-------------------|
| vote | build `./vote` (dev target) | 80 | redis (healthy) | no | Deployment + Service |
| result | build `./result`, nodemon | 80, 9229 (debugger) | db (healthy) | no | Deployment + Service |
| worker | build `./worker` | none | redis, db | no | Deployment, no Service (outbound only) |
| redis | redis:alpine | 6379 | none | no (transient vote queue) | Deployment + Service |
| db | postgres:15-alpine | 5432 | none | **yes** | StatefulSet + Service + PVC |
| seed | build `./seed-data`, profile `seed` | none | vote | n/a | not migrated (test-data helper) |

## Volumes
| Compose | Type | Purpose | Decision |
|---------|------|---------|----------|
| `db-data` | named volume | Postgres data | `volumeClaimTemplates`, 2Gi, `microk8s-hostpath` |
| `./vote`, `./result` | bind mount | dev live-reload of source code | dropped; code ships in the image |
| `./healthchecks` (redis, db) | bind mount | healthcheck scripts | dropped; `pg_isready` and `redis-cli ping` probes |

## Configuration and secrets
- Compose sets `POSTGRES_USER` / `POSTGRES_PASSWORD` inline (`postgres` / `postgres`).
- `worker/Program.cs` and `result/server.js` **hardcode** `postgres:postgres@db`. They do not read env vars, so the DB credentials cannot be rotated without rebuilding those images.
- `vote` only talks to Redis and never touches Postgres.
- The hostnames `db` and `redis` are a contract with the app code, so the Kubernetes Services must keep those names.
- Moved into a Secret (`db-credentials`) with `envFrom`. Still demo values.

## Networking
- Compose: `front-tier` and `back-tier`. Published host ports 18080 and 18081, debugger on `127.0.0.1:9229`.
- Kubernetes: one flat pod network, so tier separation is not enforced yet (NetworkPolicy is a later phase; Calico is available).
- Exposure: Traefik Ingress by hostname (nip.io for now, no TLS yet). Debugger dropped.

## Startup ordering
`depends_on: condition: service_healthy` has no Kubernetes equivalent. Ordering relies on the apps retrying (`worker` and `result` log "Waiting for db") plus readiness probes.

## Dev vs production
The Compose file is a development setup (nodemon, `--inspect`, dev build target, source bind mounts). The migration targets the published production images instead.

## Conversion tooling
Kompose translated syntax but not intent. Full list in `kompose-fixes.md`. Biggest misses: no Services for `db` and `redis`, Postgres as a Deployment, PVCs created from dev bind mounts, local image names, dev command and debugger port kept.

## Risks and findings
1. Hardcoded DB credentials in upstream app images.
2. Docker Hub images may not match the pinned upstream commit. CI should build our own later.
3. Target node was at about 96% requested CPU, so pods stayed Pending until requests were lowered.
4. Postgres 15 to 17 needs dump/restore or pg_upgrade. Kept out of this migration on purpose.
5. No TLS yet.
6. Voter identity is a client-side cookie (`voter_id`). Clearing cookies, a second browser or `curl` without a cookie each create a new row, so there is no protection against repeat voting. Same-cookie votes are updated, not duplicated (verified on the migrated stack).
