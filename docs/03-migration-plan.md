# 03 - Migration plan

Date: 2026-11-19

## Strategy
Parallel run with dump/restore. The new stack is deployed and tested next to the old one, data is moved with `pg_dump | psql`, users are pointed at the new URL, and the Compose host stays intact as the rollback.

- Source: Compose host (Docker Compose v2, Postgres 15)
- Target: MicroK8s host (Traefik ingress, hostpath storage)
- Out of scope: Postgres 15 to 17 upgrade, TLS and real DNS, CI/CD, NetworkPolicies (separate milestones)

## Phases
| # | Phase | Status |
|---|-------|--------|
| 1 | Compose baseline | done |
| 2 | Audit and Kompose conversion | done |
| 3 | Deploy to target, verify target DB is empty | done |
| 4 | Data move and verification | done |
| 5 | Cutover rehearsal (stop writes, final sync, switch) | done (timing estimated) |
| 6 | Rollback test | done (scenario A) |

## Cutover runbook
1. Confirm target pods are Running and the Ingress answers.
2. Record source vote counts.
3. Scale the target worker to 0 so nothing writes during the restore.
4. Stop `vote` on the source (freezes writes).
5. Final dump from source, restore into `db-0`.
6. Compare counts source vs target.
7. Scale the target worker to 1; cast a test vote and confirm it appears on the result page.
8. Announce the new URL. Keep the source stopped but intact.

## Risks
| Risk | Mitigation |
|------|------------|
| Votes written to source after the dump | Stop `vote` on source before the final dump |
| Restore fails halfway | `--clean --if-exists` makes it repeatable; source is untouched |
| Target scheduling fails on a full node | Small CPU requests; check `describe node` before cutover |
| App can't reach DB after move | Credentials are hardcoded in the images, so keep `postgres` / `postgres` |
| Prebuilt images differ from source code | Note as finding; build own images in CI later |
| Need to undo | See `05-rollback.md` |

## Downtime estimate
Writes are frozen from step 4 to step 7. For this lab dataset (3 rows) the expected window is under a minute and dominated by manual typing. For a client database, dump and restore time grows with size, and a large database should use logical replication instead.

## Success criteria
- Row counts match between source and target.
- A new vote on the new URL appears on the new result page.
- All pods Running with no restarts.
- Rollback procedure tested.
