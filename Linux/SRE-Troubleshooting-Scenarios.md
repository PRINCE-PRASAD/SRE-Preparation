# SRE Troubleshooting Scenarios — Answers

A structured answer for each scenario: **what they're testing**, **the investigation flow**, and **what to say out loud**. The goal in these interviews isn't reciting commands — it's showing a calm, structured thought process under pressure.

---

## 1. Production Server is Slow (2 AM page)

**What they're testing**: Do you have a structured method, or do you panic and randomly poke at things?

**Flow** — go top-down, broad to narrow:
1. `uptime` → load average (1/5/15 min trend — climbing or already peaked?).
2. `top`/`htop` → what's consuming CPU/memory right now.
3. `free -h` → memory pressure, swap usage.
4. `df -h` / `df -i` → disk/inode space.
5. `iostat -x 1` → disk I/O saturation (`%util`, `await`).
6. `ss -tulnp` / `sar -n DEV` → network/connection saturation.
7. Application/service logs (`journalctl -u`, app logs) → the actual error surfacing at the top.
8. Check **recent changes**: deployments, config changes, traffic spike, cron jobs — "slow" at 2 AM is very often a **cron job** (backup, log rotation, batch job) colliding with normal load.

**Say this out loud**: "I'd start broad — CPU, memory, disk, network — before diving into app logs, because I want to rule out infrastructure-level causes before assuming it's a code bug. I'd also immediately check what changed recently — deploys, cron jobs, traffic pattern — since '2 AM' is a strong hint toward a scheduled job."

---

## 2. CPU Suddenly Hits 100%

**Flow**:
1. `top` sorted by `%CPU`, or `ps -eo pid,ppid,cmd,%cpu --sort=-%cpu | head`.
2. Identify: one runaway process, or many processes collectively?
3. `mpstat -P ALL 1` → is it spread across all cores or pegged on one (single-threaded bug/infinite loop)?

**If it's a Java process**:
- Get the PID, then take a **thread dump**: `jstack <pid>` (or `kill -3 <pid>` which dumps to stdout/log).
- Cross-reference high-CPU **threads** (via `top -H -p <pid>` to get thread IDs, convert to hex, `printf '%x\n' <tid>`) against the thread dump to find exactly which thread/stack trace is spinning — classic signature of a stuck loop, deadlock-adjacent busy-wait, or runaway GC.
- Check GC activity too — `jstat -gcutil <pid> 1000` — heavy/constant GC (not classic 100% CPU from app code but from garbage collection) is a very common Java "high CPU" root cause.

**If multiple processes are high**:
- Could be legitimate load (traffic spike — check request rate/metrics) vs. something systemic (e.g., a retry storm across multiple workers after a downstream dependency started failing, or a fork-bomb-like bug spawning excess children — check with `pstree`).

**Reduce impact without rebooting**:
- `renice` the offending process down in priority to protect other workloads sharing the box.
- If it's one bad request/connection causing it, can sometimes kill just that connection/request rather than the whole process.
- Scale out horizontally (add capacity) as an immediate mitigation while root-causing, rather than restarting blind (restarting destroys the evidence you need for RCA — kill it *after* capturing thread dumps/logs, not before).

---

## 3. Memory Keeps Increasing (3-day climb, eventual crash)

**Investigate**:
- `free -h` over time / Grafana memory graphs → confirm it's a steady climb, not sawtooth (sawtooth = normal GC cycling; steady climb with no drops = leak).
- `ps -eo pid,cmd,%mem,rss --sort=-rss | head` → identify the specific process.
- Track that process's RSS over time (`pidstat -r -p <pid> 60` running over hours) — a leak shows RSS climbing without ever coming back down, even after GC/idle periods.

**Is it a memory leak?**
- Key signal: memory grows **even under steady/idle load**, and never plateaus or drops after garbage collection (for GC'd languages) — that rules out "it's just legitimately caching more data" and points to objects/connections/handles never being released (unclosed DB connections, growing in-memory caches with no eviction, event listeners not being detached, etc).

**OOM Killer**: kernel mechanism that kills a process when the system is critically low on memory and can't reclaim enough — picks a victim based on `oom_score` (memory footprint weighted against `oom_score_adj`).

**Where to check OOM logs**: `dmesg | grep -i "killed process"` or `journalctl -k | grep -i oom`, also `/var/log/messages` or `/var/log/syslog` depending on distro. On ECS/Fargate specifically: task stopped reason will show `OutOfMemoryError: Container killed due to memory usage` in the ECS console/CloudWatch logs — you've hit this pattern before with container OOM kills, so you can speak to it directly from experience.

---

## 4. Disk is 100% Full (`/` partition, users can't log in)

**Why login fails**: many systems need to write to `/` for session files, PAM logs, or `/tmp` during login — a fully-full root partition can block new SSH sessions entirely even though the box is technically "up."

**Investigate**:
```bash
df -h                      # confirm which mount is full
du -sh /* 2>/dev/null | sort -rh   # walk down to find the heavy directory
du -sh /var/log/* | sort -rh       # logs are the most common culprit
```

**Safely free space**:
- Truncate (don't delete!) an actively-written log if you need immediate relief without breaking a process's open file handle: `truncate -s 0 /var/log/bigfile.log` (safer than `rm` when a process is still writing to it).
- Clear old rotated logs (`*.gz`, `*.1`, `*.old`) once you've confirmed they're not needed.
- Clear package manager cache (`apt clean` / `yum clean all`), old kernels no longer in use.

**Deleted files still consuming disk**:
- Classic gotcha: `df -h` shows full, `du -sh` doesn't add up. A process still has a deleted file **open** (file is unlinked from the directory but its inode/data blocks aren't freed until the last file descriptor referencing it is closed).
- Find it: `lsof +L1` (lists open files with a link count of 1 or less — i.e., deleted-but-open) or `lsof | grep deleted`.
- Fix: restart the offending process (releases the file handle, kernel reclaims the space) — or if you can't restart yet, some daemons support a signal to reopen log files (e.g. `logrotate`'s `copytruncate`, or `kill -HUP` on daemons that support live log reopening) as a less disruptive option.

---

## 5. High Load Average (15) but CPU Only 20%

**Why**: Load average counts processes in the **run queue** (waiting for CPU, state R) **plus** processes in **uninterruptible sleep** (state D — usually blocked on disk I/O). A pile of D-state processes waiting on slow/stuck disk I/O inflates load average heavily without touching CPU utilization at all.

**What load average includes**: run-queue + D-state (I/O-wait) processes, averaged over 1/5/15 minutes.

**Troubleshoot**:
1. `ps aux | awk '$8=="D"'` (or check the `STAT` column in `top`) → find the D-state processes.
2. `iostat -x 1` → check `%util` and `await` on the disk(s) those processes touch — very likely near 100% util or high latency.
3. Could also be NFS-related (a hung/slow NFS mount causes exactly this pattern) or a failing/degraded physical disk (`dmesg` for I/O errors, `smartctl` for disk health).
4. **Say this out loud**: "High load with low CPU almost always points me to disk I/O wait, not CPU — the numbers alone are the tell, before I even run a single command."

---

## 6. Server Cannot Reach Database (PostgreSQL)

**Structured order (cheapest/fastest checks first)**:
1. **Service status on the DB side**: `systemctl status postgresql`, is it even running/listening — `ss -tulnp | grep 5432`.
2. **Network reachability**: `ping` (may be blocked by security groups — not conclusive if it fails), better: `nc -zv db-host 5432` or `telnet db-host 5432` to test the actual TCP port.
3. **DNS**: does the app's DB hostname resolve? `dig db-host` / `getent hosts db-host` — especially relevant if the DB endpoint is an RDS DNS name that could have changed (failover, endpoint rotation).
4. **Firewall / Security Groups / NACLs** (AWS-specific): is the app's security group allowed inbound on the DB's security group rules? NACLs are stateless — check both directions.
5. **App-level**: correct connection string, credentials not expired/rotated, connection pool exhausted (check `pg_stat_activity` for max connections hit).
6. **Say this out loud**: "I work outward from the database itself — is it up and listening — before I look at network layers, since there's no point debugging security groups if Postgres itself isn't even running."

---

## 7. Website Returns 502 Bad Gateway

**Meaning**: NGINX (acting as reverse proxy) got an invalid/no response from the **upstream** (backend app server).

**Possible reasons**:
- Backend process crashed/not running.
- Backend is running but not listening on the port NGINX is configured to proxy to (misconfig, wrong port after a deploy).
- Backend is overwhelmed (out of workers/threads) and refusing new connections abruptly.
- Unix socket permission issue if NGINX proxies via a socket rather than TCP.

**Logs to check first**:
1. `/var/log/nginx/error.log` — will show something like `connect() failed (111: Connection refused) while connecting to upstream` — tells you immediately if NGINX can't even reach the backend.
2. Backend application logs — did it crash, or is it slow to accept connections.

**Isolate NGINX vs backend**:
- `curl` the backend **directly** (bypassing NGINX), e.g. `curl -v http://127.0.0.1:8080/health` if that's the app's actual bind address/port. If direct curl works but NGINX proxying doesn't → NGINX config issue (wrong upstream address/port, or NGINX can't reach it due to a local firewall/SELinux context). If direct curl also fails → it's the backend, not NGINX.

---

## 8. Website Returns 504 Gateway Timeout (backend reports healthy)

**Why 504 can happen even with a "healthy" backend**: 504 means NGINX **got no response within its configured timeout** — the backend can be "healthy" in the sense of being up and passing a shallow health check, while still being too slow to answer this *particular* request in time (slow DB query, downstream API call hanging, thread pool exhausted, GC pause).

**Timeout settings that matter** (NGINX side): `proxy_connect_timeout`, `proxy_read_timeout`, `proxy_send_timeout` — if the backend is legitimately slow for some endpoints (e.g. a report-generation endpoint) but the global timeout is short, you'll see 504s only on those specific paths.

**Metrics to check**:
- Backend request latency (p50/p95/p99) — is it *this* endpoint's latency spiking, or across the board?
- Downstream dependency latency (DB, external API calls) — often the real root cause hiding behind a "healthy" app.
- Thread/connection pool saturation on the backend — requests queueing before they're even processed.

---

## 9. Users Report DNS Failure (some can access, some can't)

This split (some users affected, others not) is the key clue — it's rarely "DNS is broken" globally, it's usually **caching or resolver-specific**.

- **DNS propagation**: if a record recently changed, users hitting resolvers that haven't refreshed their cache (based on TTL) will still get the old answer — a low TTL beforehand minimizes this pain during changes.
- **Resolver issue**: different users may use different DNS resolvers (ISP resolver, corporate DNS, public resolver like 8.8.8.8) — one resolver may have a stale/cached bad entry while another has already picked up the change.
- **Local cache**: individual machines/browsers can cache DNS results (OS-level resolver cache, or the browser's own cache) — a user's own machine can be stuck on an old IP even after everything upstream is fixed.
- **Route53** (if applicable): check the actual record — health-check-based routing, weighted routing, or a failover record could be routing different users differently by design; also check Route53 health check status if using failover routing.
- **Corporate DNS**: some users behind a corporate network may go through an internal DNS forwarder/proxy with its own cache or filtering rules, independent of public DNS.

**Diagnosis approach**: `dig +trace domain` to see the full resolution path, compare `dig @8.8.8.8 domain` (public) vs `dig @<corporate-resolver> domain` (internal) to isolate where the divergence is.

---

## 10. SSH Suddenly Stops Working

**Structured check, cheapest first** (this is a classic "elimination" question):
1. **EC2 health** (AWS console): is the instance status check passing? (`2/2 checks passed` — instance reachability + system reachability).
2. **Security Groups**: is port 22 allowed inbound from your IP/bastion?
3. **NACL**: stateless — check both inbound *and* outbound rules on the subnet.
4. **sshd status**: if you have any other access (Session Manager, console output, serial console) — `systemctl status sshd`, did it crash or fail to restart after a config change.
5. **Disk full**: a completely full `/` can prevent sshd from writing PAM/session logs, causing SSH to hang or refuse — ties back to scenario 4.
6. **CPU**: pegged at 100% can make sshd unresponsive to new connections even if it's technically running.
7. **Network**: routing table changes, a broken VPN/bastion path, or an ACL change elsewhere in the path.
8. **Bastion**: if access is via a bastion host, is the bastion itself reachable/healthy — don't assume the target server is the problem until you've confirmed the bastion path is fine.

**AWS-specific fallback if genuinely locked out**: **EC2 Instance Connect** or **Session Manager** (SSM) can often get a shell even when SSH itself is broken, since Session Manager doesn't rely on port 22 or security group inbound rules.

---

## 11. Kubernetes Pod Keeps Restarting (CrashLoopBackOff)

**First five commands**:
```bash
kubectl get pods                          # confirm status & restart count
kubectl describe pod <pod>                # events, resource limits, recent state
kubectl logs <pod>                        # current/last logs
kubectl logs <pod> --previous             # logs from the PREVIOUS crashed instance — critical, since current logs may be empty right after a fresh restart
kubectl get events --sort-by='.lastTimestamp'   # cluster-wide recent events
```

**What to look for**:
- **Exit code** (from `describe pod`, under "Last State"): `137` = SIGKILL, very often OOMKilled (check `describe` output for `Reason: OOMKilled` explicitly) or a liveness probe killing it; `1` = generic app error; `0` = clean exit, but if it keeps happening the app is exiting normally when it shouldn't (e.g. finishing its main function immediately instead of blocking).
- **Liveness probe** misconfigured too aggressively (timeout too short, wrong path) → Kubernetes kills a genuinely-healthy-but-slow-starting pod repeatedly.
- **Readiness probe** failing just marks it out of service (no traffic) — doesn't restart it, so this is a different symptom (traffic issues, not crash loop).
- Resource limits too low → OOMKilled on legitimate memory usage that exceeds the configured limit.

---

## 12. One Pod Healthy, Another Isn't (same Deployment)

Since it's the *same* deployment (same image, same spec) but one replica fails, the difference is almost always **node-specific or scheduling-specific**, not code:

- **Node issue**: is the failing pod scheduled on a node with resource pressure, disk pressure, or a hardware/kernel issue? `kubectl describe node <node>` for conditions.
- **Image**: unlikely if truly identical deployment — but check if it's mid-rollout (different pods briefly running different image versions during a rolling update).
- **Secrets/ConfigMap**: if there was a recent update to a mounted Secret/ConfigMap, pods created *before* vs *after* the update can end up with different mounted content until they're recycled — check pod age vs Secret/ConfigMap last-modified time.
- **Resource limits**: that specific node might be under more memory/CPU pressure from other pods co-located there, pushing this one over its limit while an identical pod on a quieter node is fine.

---

## 13. Application Works on One Server, Not Another (identical code)

"Identical code" doesn't mean identical **environment** — that's the whole point of this question:

- Environment variables / config files that differ (not actually identical setup, just identical *app* code).
- OS package / library version drift (different patch level, different OpenSSL version, etc — `diff <(rpm -qa | sort) <(ssh other rpm -qa | sort)` or equivalent).
- Different underlying hardware/kernel version → subtle syscall or performance behavior differences.
- Filesystem state differences — disk full, permissions, SELinux/AppArmor context differing between the two.
- Time/clock skew (breaks SSL handshakes, token-based auth, or scheduled jobs) — check `timedatectl`/`chronyc tracking` on both.
- Network path differences — one server might have a stale DNS cache, different route to a dependency, or is behind a different security group.

**Approach**: diff everything you can between the two — package versions, env vars, config files, and system time — rather than re-reading the app code, since the code is confirmed identical.

---

## 14. One ECS Task Keeps Restarting (desired count 4, one flapping)

**Where to check**:
1. **ECS Service Events** (console or `aws ecs describe-services`) — shows high-level reasons like failed health checks, task stopped, deployment circuit breaker tripped.
2. **Stopped task details** (`aws ecs describe-tasks --tasks <task-arn>`) → `stoppedReason` field is gold — e.g. `Essential container in task exited`, `OutOfMemoryError`, or a failed health check.
3. **Container logs** in CloudWatch Logs (via the log group configured in the task definition) — the actual application-level error.
4. **Health checks**: if the task definition or ALB target group health check is too strict/short, ECS may kill a slow-starting-but-otherwise-fine container repeatedly.
5. **Exit codes**: same interpretation as the Kubernetes case — `137` often memory-related, non-zero app-specific codes point to app logic.

Given your direct GitHub Actions/ECS pipeline experience, you already have a natural example here: task definition registration/deployment issues where a bad revision or missing env var caused exactly this kind of flapping.

---

## 15. Docker Container Exits Immediately (`docker run my-app` exits in seconds)

**Why**: Docker containers stay alive only as long as their **PID 1 process** (the `ENTRYPOINT`/`CMD`) keeps running in the **foreground**. If that process exits — intentionally or not — the container exits immediately, regardless of what else might be "running" inside.

**Common causes**:
- `ENTRYPOINT`/`CMD` runs a one-shot command instead of a long-running foreground process (e.g., a setup script that finishes and exits, instead of finally `exec`-ing into the actual server process).
- The main process daemonizes/forks into the background — Docker sees the foreground process (the launcher) exit and kills the container, even though a child process is technically still alive in the background.
- App crashes immediately on startup due to a missing config/env var/dependency — check `docker logs <container>` (works even after exit, as long as you don't `--rm` before checking) for the actual stack trace.
- Missing `CMD`/`ENTRYPOINT` entirely, or a shell script that has `exit 0` or similar at the end instead of continuing to run the app.

**Debug approach**: `docker logs <container_id>`, or run interactively without detaching: `docker run -it my-app sh` to manually execute the intended start command and see the real error, or override the entrypoint to poke around: `docker run -it --entrypoint sh my-app`.

---

## 16. GitHub Actions Deployment Suddenly Fails (worked yesterday)

Since nothing in *your* pipeline config necessarily changed, investigate in this order:

1. **Diff the actual failing step's logs** — GitHub Actions gives a full log; find the exact command/step that changed from green to red.
2. **Check for upstream dependency changes**: did a base Docker image get updated (`node:20` moving to a new patch that breaks something), did an npm/pip package release a breaking version overnight (if versions aren't pinned)?
3. **Check for expired/rotated credentials**: OIDC role trust policy changes, expired secrets, rotated AWS credentials, expired GitHub tokens for private registries.
4. **Runner-level changes**: GitHub's hosted runner images update periodically — a runner image bump can silently change available tool versions.
5. **Any infra-side change**: did an IAM policy get tightened, a security group change, an ECR repo policy change, that would only bite the pipeline on its next run?
6. **Say this out loud**: "Since the pipeline definition itself likely didn't change, I'd assume something in its *environment* changed — a base image, a dependency version, or credentials — and confirm by diffing the exact failing step's output against yesterday's successful run."

---

## 17. Production Deployment Causes HTTP 500s Immediately After Deploy

**Immediate priority**: mitigate first, root-cause second — a live 500-error incident isn't the time to debug patiently.

1. **Rollback** — fastest path to recovery in almost all cases; roll back the deployment/task definition/image tag to the last known-good version.
2. **Then investigate** (post-mitigation, or in parallel if you have capacity):
   - **Logs**: application stack trace — usually points directly at the change (new code path throwing, missing dependency).
   - **Database migration**: did a migration run that the new code depends on, but didn't actually apply cleanly (failed halfway, or is still running as an async job while the new code already expects the new schema)?
   - **Secrets/env vars**: did the new deployment expect a new environment variable/secret that wasn't actually provisioned in this environment?
3. **Say this out loud**: "First priority is stopping the bleeding — roll back — then I investigate with the pressure off, rather than trying to root-cause live in production while users are getting 500s."

---

## 18. One EC2 Instance Unhealthy Behind an ALB (3 healthy, 1 unhealthy)

1. **Health check config**: what path/port/protocol is the ALB checking, and does the unhealthy instance actually respond correctly on that exact path (curl it directly, bypassing the ALB, from another instance or via SSM).
2. **Security Groups**: does the ALB's security group have access to the target's security group on the health-check port specifically (a common miss: app port allowed, but health-check-specific port on a different listener isn't).
3. **Port**: is the app actually listening on the port the target group expects on *that* instance (could differ if it was provisioned slightly differently, or the app crashed/restarted on a different port after a config drift).
4. **NGINX/app-level**: is NGINX (or the app directly) actually up and returning a 200 on the health endpoint on that specific instance, or is it silently failing there due to a local issue (disk full, one bad deploy that only landed on this instance, etc).

---

## 19. SSL Certificate Suddenly Stops Working (`NET::ERR_CERT_DATE_INVALID`)

That specific error is squarely about **expiration** or **date validity**, not a wrong-cert or chain issue (those show different errors) — narrows it fast:

- **Expired**: check actual cert expiry: `openssl s_client -connect host:443 -servername host 2>/dev/null | openssl x509 -noout -dates`.
- **ACM** (if using AWS Certificate Manager): ACM auto-renews certs **only if DNS validation records are still correctly in place** — if someone removed/changed the validation CNAME record (common after a DNS cleanup), renewal silently fails and the cert expires on schedule without anyone noticing until it's too late.
- **Wrong certificate served**: possible if multiple certs are attached to a listener and SNI routing serves the wrong one — but this typically gives a hostname-mismatch error, not a date error, so less likely given the specific message here.
- **CloudFront/ALB**: confirm which layer is actually terminating TLS and serving the cert users see — CloudFront can have its own cert independent of the origin/ALB, and it's easy to fix the wrong layer while the actual expired cert lives elsewhere in the chain.

---

## 20. Disk Usage Keeps Growing Every Night (60% → 70% → 80% → 95%)

The steady, **time-correlated** growth (specifically at night) is the biggest clue — points at a **scheduled job**, not organic app growth:

- **Cron jobs**: check `/etc/cron.d/`, `crontab -l` for every user, and `systemctl list-timers` for systemd timers — a nightly backup/report/log-archival job is the single most common cause.
- **Logs**: is log rotation actually configured and working (`logrotate` config, and confirm it's actually running via `/var/lib/logrotate/status`)? A verbose app that logs heavily overnight (batch jobs, nightly reports) combined with broken/missing rotation is extremely common.
- **Backups**: local backup jobs that dump to disk before uploading elsewhere, and don't clean up after themselves (or the upload step silently fails, leaving the local copy behind every night).
- **Containers**: Docker's own log driver (`json-file` by default) can grow unbounded per-container if no `max-size`/`max-file` log rotation is configured at the daemon level.
- **Temporary files**: a nightly batch job generating temp/intermediate files it doesn't clean up on completion (or only cleans up on success, leaving orphaned files after a partial failure).

**Approach**: `du -sh /var/log /var/backups /tmp | sort -rh` right after one of these nightly jumps, correlate the timestamp of newly large files/directories against `crontab`/timer schedules.

---

## 21. Network Latency Suddenly Increases (P95: 40ms → 700ms)

A P95 jump this large usually isn't "the network" in the literal packet-loss sense — investigate in order of likelihood:

- **Database**: slow queries, lock contention, connection pool exhaustion — often the actual root cause even when the symptom looks like "network latency" from the outside.
- **GC (garbage collection)**: for GC'd runtimes, a long stop-the-world pause can look exactly like a latency spike from the client's perspective — check GC logs/metrics for pause duration correlating with the latency spike.
- **CPU**: saturated CPU causes request queueing before processing even starts — check `top`/CPU metrics for the same time window.
- **Disk**: if the request path touches disk (logging synchronously, writing to a slow volume), disk I/O wait can bleed into request latency.
- **External API**: if the request depends on a third-party/downstream API call, that dependency's latency directly shows up as your own — check downstream service dashboards/timeouts.
- **Actual network**: genuine network issues (packet loss, route change, MTU mismatch) are real but comparatively rarer as the root cause of a *sudden* app-level P95 spike — confirm/rule out with `mtr`/`ping` stats and interface error counters (`ip -s link`) before spending too much time here.

**Approach**: correlate the latency spike timestamp against every other metric (CPU, GC, DB, downstream calls) — the overlap tells you the root cause faster than guessing.

---

## 22. Logs Stop Appearing in Grafana (app is healthy, Loki/Promtail stack)

Since the app itself is healthy, the break is somewhere in the **shipping pipeline**, not app logic:

- **Promtail**: is it actually running and scraping the expected log files/paths? `systemctl status promtail`, check its own logs for scrape errors.
- **Permissions**: did a log file's ownership/permissions change (e.g., after a deploy that changed the running user) such that Promtail can no longer read it?
- **Network**: can Promtail actually reach the Loki endpoint — connectivity, DNS, or a recently-changed firewall/security-group rule blocking it?
- **Labels**: did a label change on the Promtail/Loki config side (or an app-side change to log format) cause queries in Grafana to no longer match because they're filtering on labels that no longer apply to the new log stream?
- **Loki itself**: check Loki's own health/ingestion metrics — it could be silently dropping logs due to rate limits or storage backend issues, not a Promtail-side problem at all.

---

## 23. Prometheus Stops Scraping Targets ("Target Down")

1. **Exporter**: is the exporter process actually running and listening on its metrics port on the target host? `curl localhost:<port>/metrics` directly on the target.
2. **Firewall**: can Prometheus's host actually reach that port on the target — security group/firewall change blocking it?
3. **Service Discovery**: if using dynamic SD (Kubernetes SD, EC2 SD, Consul, etc.), did the discovery mechanism stop finding the target — e.g., a label/tag the SD config relies on was removed, or the target's IP changed and SD hasn't refreshed.
4. **TLS**: if scraping is configured over HTTPS, a cert issue (expired, hostname mismatch) on the exporter side will cause scrape failures that show clearly in Prometheus's own scrape error message.
5. **Metrics endpoint**: confirm the actual path is still correct (`/metrics` vs a custom path) — an app update that changed the exposed path without updating the scrape config causes exactly this.

**Fastest first step**: check the actual error message in Prometheus's own "Targets" page (Status → Targets) — it almost always tells you directly (connection refused, context deadline exceeded, x509 error, etc.) rather than needing to guess.

---

## 24. RDS CPU is 95% (Database Slow)

1. **Slow query**: enable/check the **slow query log** (or `pg_stat_statements` for Postgres) to find the specific query(ies) consuming the most cumulative time.
2. **Connection count**: check current connections vs `max_connections` — too many idle-but-open connections (connection pool misconfiguration on the app side) can itself cause overhead even without heavy query load.
3. **Locks**: check for lock contention/blocking queries (`pg_locks` joined against `pg_stat_activity` for Postgres) — one long-running transaction holding a lock can cascade into many blocked queries, which *looks* like generic "high CPU" from the outside.
4. **Missing indexes**: a query doing a full table scan where an index would let it use an index scan — look at the query plan (`EXPLAIN ANALYZE`) for the worst offenders from step 1.
5. **Storage**: for some RDS engines/storage types, low burst credits (gp2 volumes) or IOPS exhaustion can manifest as CPU-adjacent slowness — check the storage/IOPS CloudWatch metrics alongside CPU.

---

## 25. Complete Production Outage (CEO calls) — First 30 Minutes

This tests **incident command**, not technical depth — structure and calm communication matter more than the exact command you run.

1. **Acknowledge and declare the incident** — get it formally tracked (incident channel/tool), so effort isn't duplicated and there's a clear record.
2. **Assess customer impact** — which services/regions/user segments are actually affected; don't assume "everything" without checking.
3. **Check monitoring dashboards and recent alerts** — let existing tooling tell you what fired first, rather than starting from a blank slate.
4. **Review recent deployments/infra changes** — the single highest-probability root cause in most real incidents is "something changed recently."
5. **Identify the failing component** — load balancer, app tier, database, DNS — narrow systematically rather than guessing.
6. **Mitigate fast**: rollback, failover, or scale — bias toward the fastest safe path to recovery over the "correct" root-cause fix; you can root-cause after the fire's out.
7. **Communicate regularly** — status updates to stakeholders on a fixed cadence (even "no update yet, still investigating" is better than silence) — this is very often what's actually graded in these interviews, since technical people forget this under pressure.
8. **After recovery**: RCA (root cause analysis) and concrete preventive action items — the incident isn't "done" when the outage ends, it's done when there's a documented cause and a fix/guardrail to prevent recurrence.

**Say this out loud**: "My first move isn't a command — it's declaring the incident and getting visibility, because an uncoordinated fire-fight where three people are quietly making changes at once is its own risk. Only after that do I start narrowing down the failing component."

---

## Priority List (if short on prep time)

Focus on these — they cover both the highest interview frequency and the broadest transferable troubleshooting muscle:

1. Server is slow (Q1) — the master framework everything else builds on.
2. High CPU (Q2) / High memory & OOM (Q3) / Disk full (Q4) / High load average (Q5) — the core Linux resource quartet.
3. NGINX 502/504 (Q7, Q8) — extremely common in real environments.
4. SSH access failure (Q10) — pure elimination-based troubleshooting, tests structure more than trivia.
5. Kubernetes CrashLoopBackOff (Q11) / ECS task restart (Q14) — container-orchestration equivalents of the same underlying logic.
6. Production deployment failure (Q17) — tests "mitigate first, root-cause second" instinct.
7. Complete production outage (Q25) — tests incident command, almost always asked in some form for a senior-leaning SRE role.

---

**One meta-tip across all of these**: interviewers are grading your *narration* as much as your answer. Say your reasoning out loud step by step ("I'd check X first because Y, which would tell me whether to go down path A or B next") rather than jumping straight to a conclusion — that's what actually distinguishes a strong SRE answer from a list of memorized commands.
