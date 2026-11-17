# 01 - Compose setup (legacy side)

Date: 2026-11-17
Host: source host (Docker + Compose v2.40.3)

## Goal
Run the voting app with Docker Compose as the "before" state for the migration.

## Source
- Upstream: https://github.com/dockersamples/example-voting-app (Apache-2.0)
- Upstream commit: `63e9150ca17af4ed05880d4245e486481f73fcb4`
- Our file: `legacy-compose/docker-compose.yml`

## Stack
| Service | Image / build | Role | Host port |
|---------|---------------|------|-----------|
| vote | built from `./vote` (Python/Flask) | voting UI | 18080 |
| result | built from `./result` (Node) | live results UI | 18081 |
| worker | built from `./worker` (.NET) | moves votes Redis -> Postgres | none |
| redis | redis:alpine | vote queue | none |
| db | postgres:15-alpine | persistent storage | none |

Networks: `front-tier`, `back-tier`. Volume: `db-data` (Postgres data).

## Changes from upstream
Only the host side of the published ports changed. Container ports, Dockerfiles and app code are untouched.

```diff
@@ -19,7 +19,7 @@ services:  (vote)
-      - "8080:80"
+      - "18080:80"
@@ -34,7 +34,7 @@ services:  (result)
-      - "8081:80"
+      - "18081:80"
       - "127.0.0.1:9229:9229"
```

Reason: other containers on the host already hold 80, 443, 8080, 8081 and 8443.

## Steps
```bash
git clone https://github.com/dockersamples/example-voting-app.git src
cd src
# edit ports in docker-compose.yml
docker compose up -d --build
docker compose ps
```

## Verification
- `curl -sI http://localhost:18080` returned `HTTP/1.1 200 OK` (Werkzeug/Python 3.11).
- Also reachable via the host's public IP.
- All five containers up; redis and db report healthy.
- Worker log: FILL
- Vote cast on :18080 appeared on :18081: FILL yes/no

## Findings (feed into the audit)
1. **This is a dev setup.** `result` runs `nodemon --inspect` and publishes a debugger on `127.0.0.1:9229`; `vote` uses the `dev` build target with `watchdog`. A production migration should build the production targets and drop the debugger.
2. **Build-based compose file.** It uses `build:` contexts, so it only works next to the upstream source. `docker-compose.images.yml` (prebuilt
