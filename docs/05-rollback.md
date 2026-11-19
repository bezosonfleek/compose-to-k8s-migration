# 05 - Rollback

Date: 2026-11-19
Status: drafted. Test result: scenario A passed. With the k8s side scaled to zero, the source `vote` started, returned 200 and the source database still held a=1, b=2. Scenario B (copying newer target votes back) was not tested.

## Principle
The Compose host is never modified beyond stopping `vote`. Its data volume (`src_db-data`) stays intact. Never run `docker compose down -v` on the source until the migration is signed off.

## Triggers
- Row counts differ after restore
- New votes do not reach the result page
- Target pods crash-loop or cannot be scheduled
- Any error that cannot be fixed within the agreed window

## A. Rollback before any new votes reached the target
1. Scale the target front end down:
   ```bash
   kubectl -n voting scale deploy vote result worker --replicas=0
   ```
2. Restart the source front end:
   ```bash
   cd ~/voting-app/src && docker compose start vote
   ```
3. Point users back at the old URL and confirm the vote page answers.

## B. Rollback after new votes were written on the target
Copy the newer data back first so no votes are lost.
1. Freeze target writes (step A.1, vote and worker).
2. Dump the target and restore into the source database:
   ```bash
   microk8s kubectl -n voting exec db-0 -- pg_dump -U postgres --clean --if-exists postgres \
    | ssh <source> "cd ~/voting-app/src && docker compose exec -T db psql -v ON_ERROR_STOP=1 -U postgres postgres"
   ```
3. Start the source front end and compare counts.

## Verify the rollback
- Source vote page answers.
- Source counts equal the last known target counts.

## Retention
Suggested: keep the source stack and volume for 7 days after cutover (agree the window with the client), then remove deliberately.
