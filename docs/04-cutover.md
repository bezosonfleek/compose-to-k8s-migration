# 04 - Cutover record

Date: 2026-11-19
Source: Compose host. Target: MicroK8s host, namespace `voting`.

## 1. Deploy
```bash
kubectl apply -k k8s/
```
Pods Running: redis, vote, result, worker, `db-0`. See "Problems" for what it took.

## 2. Prove the target database was empty
After `db-0` started, the worker had already created the `votes` table.
```
\dt                       -> public | votes | table | postgres
select count(*) from votes -> 0
```

## 3. Hold the worker
```bash
kubectl -n voting scale deploy worker --replicas=0
```

## 4. Source baseline
```
 vote | count
------+-------
 a    |     1
 b    |     2
```

## 5. Restore
```bash
ssh <source> "cd ~/voting-app/src && docker compose exec -T db pg_dump -U postgres --clean --if-exists postgres" \
 | ssh <target> "microk8s kubectl -n voting exec -i db-0 -- psql -v ON_ERROR_STOP=1 -U postgres postgres"
```
Output ended with `DROP TABLE`, `CREATE TABLE`, `COPY 3`. For a three-row database the transfer takes about a second; the wall-clock figure was dominated by typing the SSH password, so it is not a meaningful benchmark.

## 6. Verify
```
select count(*) from votes -> 3
```
Split on target: a=1, b=2 (result page shows Cats 33.0%, Dogs 67.0%, 3 votes)

## 7. Resume
```bash
kubectl -n voting scale deploy worker --replicas=1
```
- Result page shows source counts: yes (3 votes, Cats 33.0% / Dogs 67.0%)
- New vote on the vote page appears on the result page: yes (see Verification below)

## Downtime
Estimated, not measured: about 1 second per command (scale down, restore, scale up) for a 3-row database. A scripted cutover takes a few seconds and a manual one under a minute. Real figures grow with database size.

## Problems and fixes
| Problem | Cause | Fix |
|---------|-------|-----|
| `strict decoding error: unknown field "...containers[0].exec"` | Mis-indented `exec` under `readinessProbe` in the redis Deployment | Fix indentation. Client dry-run did not catch it; the real apply did |
| `db-0` and `result` Pending, "Insufficient cpu" | Node at ~96% requested CPU | Lowered CPU requests (25m / 10m) |
| `db-0` stayed Pending with old 100m request after the fix | A StatefulSet will not replace a pod that never became Ready | `kubectl delete pod db-0` so it was recreated with the new spec (PVC untouched) |
| `kubectl: command not found` over SSH | `kubectl` is a shell alias on the target, and aliases are not loaded by non-interactive SSH | Call `microk8s kubectl` explicitly |

## Verification on Kubernetes
Cast through `http://vote.<ip>.nip.io`, read on `http://result.<ip>.nip.io`.

| Step | Result page | Rows in `votes` |
|------|-------------|-----------------|
| Restored data plus new votes | 5 votes: Cats 20%, Dogs 80% | 5 |
| Same browser changes Dogs to Cats | 5 votes: Cats 40%, Dogs 60% | 5 (updated, not added) |
| Cookie deleted, vote again | not captured | 6 (new voter id, new row) |

The vote page footer showed the serving pod name (`vote-8b966d98b-jr2gm`), which is the pod hostname that Compose showed as a container ID.

## State after the test
The target database now holds more votes than the source (6 vs 3). A rollback after this point must use scenario B in `05-rollback.md`, or the extra votes are lost.

## Try it yourself
Each vote is tied to a `voter_id` cookie, so anyone following along can cast more votes in any of these ways:

1. **Change your vote:** click the other option in the same browser. The total stays the same and your row is updated.
2. **Delete the cookie:** clear the site's cookies (browser dev tools, Application tab, Cookies, delete `voter_id`), reload and vote. This counts as a new voter.
3. **Incognito window or a different browser:** starts with no cookie, so it also counts as a new voter.
4. **curl:** `curl` sends no cookie, so every call is a new voter. Use `-d vote=a` for Cats or `-d vote=b` for Dogs:
   ````bash
   curl -s -o /dev/null -d vote=a http://vote.<ip>.nip.io/
   for i in $(seq 5); do curl -s -o /dev/null -d vote=b http://vote.<ip>.nip.io/; done
   ```
4. **curl:** `curl` sends no cookie, so every call is a new voter. Use `-d vote=a` for Cats or `-d vote=b` for Dogs:
   ````bash
   curl -s -o /dev/null -d vote=a http://vote.<ip>.nip.io/
   for i in $(seq 5); do curl -s -o /dev/null -d vote=b http://vote.<ip>.nip.io/; done
   ```

To watch the effect on the database:
```bash
kubectl -n voting exec db-0 -- psql -U postgres -c 'select count(*) from votes'
```
Run it before and after each vote. The count rises for options 2, 3 and 4, and stays flat for option 1.
