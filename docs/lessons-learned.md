# Lessons learned

- **Long builds over flaky SSH die with the session.** Use `tmux` and `ServerAliveInterval`. Layer caching made the rerun cheap.
- **Distro and snap packages of fast-moving tools are years behind.** The Kompose snap (1.21) could not parse a modern Compose file. Use the upstream release binary. Bash then cached the old path, so `hash -r` was needed.
- **Kompose translates syntax, not intent.** It made Postgres a Deployment, turned dev bind mounts into empty PVCs, left no Services for backends, and kept dev commands and ports. Keep the raw output in git and log every fix.
- **Bind mounts do not translate to claims.** A bind mount of source code is a dev trick; code belongs in the image. Named volumes become PVCs.
- **Compose healthcheck maps to a readiness probe, not liveness.** A liveness probe on a slow-starting database can kill it mid-init.
- **`PGDATA` needs a subdirectory on Kubernetes volumes** because initdb refuses a non-empty directory (`lost+found`).
- **Scheduling uses requests, not usage.** An almost idle node was 96% requested. Check `describe node` before sizing.
- **A StatefulSet will not replace a pod that never became Ready.** A Pending pod keeps its old spec until deleted.
- **`--dry-run=client` does not catch unknown fields.** Use server-side validation or a schema checker such as kubeconform.
- **Aliases do not exist in non-interactive SSH.** Scripts need the real binary (`microk8s kubectl`).
- **Services can all listen on port 80.** Each pod and Service has its own IP. Only the host's published port was ever a conflict.
- **Upstream apps hardcode credentials,** so a different DB password would break them. Check before rotating.
- **Replicas on a database do not give replication.** Three Postgres pods would be three unrelated databases.
- **Check what the app treats as identity before migrating state.** Votes are keyed on a cookie, so test "same user changes vote" and "new user votes" separately. Both behaved identically on Kubernetes.
- **Scaling to zero and back changes pod names.** Anything that records the hostname (the vote page footer) will show a new value.
