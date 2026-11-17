# 00 - Environment baseline

Date: 2026-11-17

## Hosts

| Role | Hostname | Orchestration | Purpose |
|------|----------|---------------|---------|
| Legacy (source) | server | Docker + Compose, k3s also installed | Runs the Compose version of the app |
| Target | learning | MicroK8s v1.35 (containerd) | Receives the migrated app |

## Hardware / OS (both hosts)
- Ubuntu 24.04 LTS
- 7.7 GiB RAM each
- Source host: 99 GB disk, 51 GB free

## Software (source host)
- Docker 24.0.7
- Docker Compose v2.40.3 (plugin, `docker compose`)
- Legacy docker-compose v1.29.2 is also installed (EOL), not used in this project

## Software (target host)
- MicroK8s with Calico, Traefik, MetalLB, cert-manager, Longhorn CRDs, Argo CD CRDs
- Container runtime: containerd

## Port situation on the source host
| Port | Used by |
|------|---------|
| 80, 443 | Existing unrelated Caddy container |
| 8080, 8081, 8443 | Other existing containers |
| 6443 | k3s API |
| 22 | sshd |

## Findings
- The source host is shared with other workloads, not a clean slate.
- Default voting-app host ports (8080, 8081) conflict, so they are remapped in `docker-compose.yml` only. The Dockerfiles and container ports are unchanged.
- Compose v1 is end-of-life, so this project uses Compose v2.

## Decisions
- Vote UI on host port 18080, result UI on host port 18081.
- The voting app gets its own isolated Compose network and does not touch the existing Caddy.
