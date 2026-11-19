# Compose to Kubernetes Migration

Documenting a real migration of a multi-service app from Docker Compose
to Kubernetes, from a clean environment through cutover and rollback.

**Status:** in progress. Compose baseline done, manifests being deployed to MicroK8s.

## App
Based on [dockersamples/example-voting-app](https://github.com/dockersamples/example-voting-app) (Apache-2.0).

## Layout
- `legacy-compose/` Compose file (the "before")
- `k8s-kompose-raw/` unedited Kompose 1.38.0 output
- `k8s/` hand-corrected manifests, applied with `kubectl apply -k k8s/`
- `docs/` write-ups, one per phase

## Docs
- [00 - Environment](docs/00-environment.md)
- [01 - Compose Setup](docs/01-compose-setup.md)

Cluster state is inspected with [Lens](https://k8slens.dev/).
