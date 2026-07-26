# SRE Troubleshooting Questions — Simple English Answers

25 common interview problems. Each one has three parts:

- **Why they ask this** — what the interviewer wants to see
- **What to check** — the steps, in order
- **What to say** — a short answer you can practise out loud

**The most important rule:** the interviewer wants to hear *how you think*. Do not just say the answer. Say your reason too.

Bad answer: "I would check the disk."
Good answer: "First I check the disk. If the disk is full, that explains the problem. If it is fine, I move to the next thing."

That is the whole difference. Say the step, then say why.

There is a **word list at the end** of this file. If a word is new to you, look there.

---

## 1. The server is slow. You get a page at 2 AM.

**Why they ask this**
They want to see if you have a plan. Many people panic and check random things. Do not do that. Start wide, then go narrow.

**What to check**

Start with the four basic resources. CPU, memory, disk, network.

```bash
uptime            # How busy is the server? Is it getting worse or better?
top               # Which program is using the most CPU and memory?
free -h           # Is memory full? Is it using swap?
df -h             # Is the disk full?
df -i             # Is the disk out of inodes? (small files use these up)
iostat -x 1       # Is the disk too busy? Look at %util and await
ss -tulnp         # How many network connections are open?
```

Then read the logs:

```bash
journalctl -u <service-name>     # System logs for one service
```

Then ask the most important question: **what changed?**

- Was there a new deploy?
- Did someone change a setting?
- Did traffic go up?
- Is a **cron job** running? (a job on a timer)

**A useful clue:** the problem started at 2 AM. That is a strong hint. Backups, log cleanup, and report jobs usually run at night. A night job plus normal traffic can make the server slow.

**What to say**

> "I start wide, with CPU, memory, disk and network. I want to remove the simple causes first, before I say it is a code problem. I also check what changed recently. Because the time is 2 AM, I would check the cron jobs early. Night jobs are a common cause."

---

## 2. CPU suddenly goes to 100%.

**Why they ask this**
They want to see if you can find *which* program is the problem, not just say "CPU is high."

**What to check**

Find the program first:

```bash
top                                                    # sort by CPU
ps -eo pid,ppid,cmd,%cpu --sort=-%cpu | head           # top CPU users
mpstat -P ALL 1                                        # check each CPU core
```

Then ask two questions:

1. **Is it one program, or many programs?**
2. **Is it using all CPU cores, or only one?**

Only one core at 100% usually means a loop in the code. A loop is code that repeats forever and never stops.

**If it is a Java program**

Java has special tools. This is a common follow-up question.

```bash
top -H -p <pid>          # show the threads inside the program
printf '%x\n' <tid>      # change the thread ID to hex (needed for next step)
jstack <pid>             # print all threads and what they are doing
jstat -gcutil <pid> 1000 # check garbage collection
```

Take the thread ID from `top -H`, change it to hex, then find it in the `jstack` output. Now you know the exact line of code that is stuck.

Also check **garbage collection** (GC). GC is how Java cleans up unused memory. If GC runs all the time, CPU goes to 100%. The code is not the problem here — the memory settings are. This is very common.

**If many programs are high**

Then it may not be a bug. Check these:

- **Real traffic increase.** Check your request count. More users means more CPU. This is normal.
- **Retry storm.** A service below you started failing. Now all your programs keep retrying it. The retries use the CPU.
- **Too many child processes.** Use `pstree` to see if one program is creating many copies of itself.

**How to reduce the problem without a restart**

- `renice <pid>` — lower the priority of the bad program. Other programs get CPU again.
- Add more servers (scale out). This helps users now, while you keep looking.
- Kill only the bad request or connection, not the whole program.

**Important:** do not restart first. A restart deletes the evidence. You will never find the real cause. Take the thread dump and the logs **first**, then restart.

**What to say**

> "First I find which program is using the CPU. Then I check if it is one CPU core or all of them. One core usually means a loop in the code. If it is Java, I take a thread dump with jstack and match the busy thread to the code. I also check garbage collection, because heavy GC often looks like high CPU. And I save the logs and thread dump before any restart, so I do not lose the evidence."

---

## 3. Memory keeps going up for 3 days, then the app crashes.

**Why they ask this**
They want to know if you can tell a **memory leak** from normal memory use. These look similar but are different.

**What to check**

Look at the shape of the graph over time. The shape tells you the answer.

```bash
free -h                              # current memory
ps -eo pid,cmd,%mem,rss --sort=-rss | head    # which program uses the most
pidstat -r -p <pid> 60               # watch one program over time
```

Two shapes to know:

| Shape on the graph | What it means |
|---|---|
| Up, down, up, down (like teeth) | **Normal.** Memory is cleaned up. |
| Up, up, up, never down | **A leak.** Memory is never released. |

**Is it a memory leak?**

The clear sign: **memory grows even when the app is quiet.** If nobody is using the app and memory still goes up, it is a leak. Normal caching stops growing when traffic stops. A leak does not.

Common causes of leaks:

- Database connections that are opened but never closed
- A cache with no size limit and no expiry time
- Event listeners that are added but never removed
- Lists or maps in memory that only grow

**What is the OOM Killer?**

OOM means "out of memory". The OOM Killer is part of the Linux kernel. When the server is almost out of memory, the kernel picks one program and kills it. This protects the whole server.

The kernel picks the program with the highest `oom_score`. Programs using more memory get a higher score.

**Where to find OOM logs**

```bash
dmesg | grep -i "killed process"
journalctl -k | grep -i oom
```

Or check `/var/log/messages` or `/var/log/syslog`. It depends on your Linux version.

**On AWS ECS or Fargate**, look at the stopped task reason. You will see:

```
OutOfMemoryError: Container killed due to memory usage
```

You have seen this before in your own ECS work, so you can talk about it from real experience. That is much stronger than theory.

**Short-term fix:** restart the app on a schedule, or give it more memory. This buys you time. It is **not** the real fix. The real fix is finding the leak in the code.

**What to say**

> "First I look at the memory graph over time. A saw shape means normal. A straight line up means a leak. The clearest test is whether memory grows when the app is idle. If yes, it is a leak. Then I check the OOM logs in dmesg to confirm the kernel killed it. A bigger memory limit helps for now, but the real fix is in the code."

---

## 4. The disk is 100% full. Users cannot log in.

**Why they ask this**
This has one classic trick in it (deleted files). They want to see if you know it.

**Why login stops working**

When you log in, the system needs to write small files. It writes session files, and login logs. If the disk is completely full, it cannot write them. So login fails or hangs — even though the server is still running.

**What to check**

```bash
df -h                                # which disk is full?
du -sh /* 2>/dev/null | sort -rh     # which folder is big?
du -sh /var/log/* | sort -rh         # logs are the most common cause
```

Work down step by step. Find the big folder, then look inside it.

**How to free space safely**

For a log file that a program is still writing to, **do not delete it.** Empty it instead:

```bash
truncate -s 0 /var/log/bigfile.log
```

Why? If you delete a file that a program has open, the space is **not** freed. Emptying the file is safe and works right away.

Other safe things to clean:

- Old rotated logs: files ending in `.gz`, `.1`, `.old`
- Package cache: `apt clean` or `yum clean all`
- Old Linux kernels you are not using

**The classic trick: deleted files still use space**

Here is the situation. `df -h` says the disk is full. But `du -sh` says the files are small. The numbers do not match. Why?

Someone deleted a big file. But a program still has that file **open**. Linux only frees the space when the last program closes the file. Until then, the space is still used but you cannot see the file.

Find these files:

```bash
lsof +L1              # shows deleted files that are still open
lsof | grep deleted
```

Fix it:

- Restart the program that has the file open. The space comes back at once.
- Or, if you cannot restart it, send it a signal to reopen its log file: `kill -HUP <pid>`. Not all programs support this.

**What to say**

> "I use df to find the full disk, then du to find the big folder. Logs are usually the cause. For a live log file I use truncate, not rm, because deleting a file that a program has open does not free the space. If df and du do not agree, I check lsof +L1 for deleted files that are still open."

---

## 5. Load average is 15, but CPU is only 20%.

**Why they ask this**
This is a knowledge question. The answer tells them right away if you understand Linux. You should be able to answer it without running any command.

**Why this happens**

Load average counts two kinds of processes:

1. Processes **waiting for CPU** (state R)
2. Processes **stuck waiting for the disk** (state D)

State D is the important part. "D" means uninterruptible sleep. The process is blocked, waiting for the disk to answer. It uses **no CPU** while it waits.

So: many processes stuck on a slow disk makes load average very high, but CPU stays low.

**High load + low CPU = a disk problem, almost always.**

**What to check**

```bash
ps aux | awk '$8=="D"'    # find the stuck processes
top                        # check the STAT column for D
iostat -x 1                # check %util and await on the disk
```

If `%util` is near 100% or `await` is high, you found it. The disk is too slow or too busy.

Two other causes to know:

- **NFS.** A network file system that is slow or hanging creates exactly this pattern. Very common.
- **A failing disk.** Check `dmesg` for I/O errors, and `smartctl` for disk health.

**What to say**

> "High load with low CPU points to disk wait, not CPU. Load average counts processes in state D, and those are waiting on disk. They use no CPU. So I look at iostat for disk util and await, and I check for a slow NFS mount or a failing disk. I can say this before running any command — the two numbers together are the clue."

---

## 6. The server cannot reach the database (PostgreSQL).

**Why they ask this**
They want to see if you check things in a smart order. Start close, then move outward.

**What to check**

Check the database first. Then the network. Then the app. Do not start with the hard things.

**Step 1. Is the database even running?**

```bash
systemctl status postgresql
ss -tulnp | grep 5432        # is it listening on port 5432?
```

There is no point checking firewalls if the database is stopped.

**Step 2. Can you reach the port?**

```bash
nc -zv db-host 5432          # best test — tests the real port
telnet db-host 5432          # also works
```

Do not trust `ping`. Ping often gets blocked by firewall rules. A failed ping does **not** mean the server is down.

**Step 3. Does the name resolve?**

```bash
dig db-host
getent hosts db-host
```

This matters a lot for AWS RDS. The endpoint is a DNS name. If RDS failed over to another server, the name now points somewhere new. If your app cached the old address, it is talking to nothing.

**Step 4. Firewall rules (on AWS)**

- **Security group:** does the database's security group allow the app's security group on port 5432?
- **NACL:** this is a subnet firewall. It is **stateless**, so you must allow traffic in *and* out. This catches many people.

**Step 5. The app side**

- Is the connection string correct?
- Did the password change (rotate) recently?
- Is the connection pool full? Check with:

```sql
SELECT count(*) FROM pg_stat_activity;
```

Compare that number to `max_connections`. If they are equal, the pool is full. New connections will fail.

**What to say**

> "I work outward from the database. First, is Postgres running and listening. Then can I reach the port with nc, because ping is often blocked and tells me nothing. Then DNS, which matters for RDS because the endpoint changes on failover. Then security groups, and NACLs both directions because NACLs are stateless. Last, the app config and the connection pool."

---

## 7. The website returns 502 Bad Gateway.

**Why they ask this**
They want you to know what 502 actually means. Many people guess.

**What 502 means**

NGINX sits in front of your app. It receives the user's request and passes it to the app behind it. The app behind is called the **upstream**.

502 means: **NGINX could not get a valid answer from the app.**

So the problem is almost always the app, not NGINX.

**Possible causes**

- The app crashed and is not running
- The app is running, but on a different port than NGINX expects (often after a deploy)
- The app is overloaded and refusing new connections
- If NGINX connects through a Unix socket, the socket permissions may be wrong

**Which logs to read first**

```bash
tail -100 /var/log/nginx/error.log
```

Look for a line like this:

```
connect() failed (111: Connection refused) while connecting to upstream
```

"Connection refused" tells you a lot. It means nothing is listening on that port. The app is down, or on the wrong port.

Then read the app's own logs. Did it crash? Why?

**How to test if the problem is NGINX or the app**

Skip NGINX. Talk to the app directly:

```bash
curl -v http://127.0.0.1:8080/health
```

Use the real port your app listens on.

| Result | What it means |
|---|---|
| Direct curl works, but NGINX gives 502 | The problem is NGINX config (wrong port or address) |
| Direct curl also fails | The problem is the app |

This one test splits the problem in half. It is the fastest thing you can do.

**What to say**

> "502 means NGINX did not get a valid answer from the app behind it. So I check the app first. I read the NGINX error log, because it usually says connection refused, which means nothing is listening on that port. Then I curl the app directly, skipping NGINX. If the direct curl works, it is an NGINX config problem. If it fails too, it is the app."

---

## 8. The website returns 504 Gateway Timeout, but the app says it is healthy.

**Why they ask this**
They want to see if you understand that "healthy" and "fast" are not the same thing.

**Why 504 happens even when the app is healthy**

502 means "no answer". 504 means "the answer was too slow."

The app is running. Its health check passes. But the health check is usually very simple — it just returns "OK". It does not test the slow part.

So the app can be healthy and still be too slow for a real request.

Common reasons the real request is slow:

- A slow database query
- A call to another API that is hanging
- The thread pool is full, so requests wait in a queue before they even start
- A long garbage collection pause

**The NGINX timeout settings**

```nginx
proxy_connect_timeout    # time to open the connection
proxy_read_timeout       # time to wait for the answer  ← usually this one
proxy_send_timeout       # time to send the request
```

`proxy_read_timeout` is the one that causes most 504s. The default is 60 seconds.

**A useful clue:** are you getting 504 on **all** pages, or only on **one** page?

Only one page usually means that page is genuinely slow. For example, a report page that takes 90 seconds, with a 60 second timeout. The fix is either to make the page faster, or to make it run in the background.

**What to check**

- Latency **per endpoint**, at p95 and p99. Not the average. The average hides slow requests.
- Latency of every downstream call (database, other APIs). The slow part is usually here.
- Thread pool and connection pool usage. If they are full, requests queue up and time out.

**What to say**

> "504 means the answer was too slow, not missing. A health check is usually very simple, so the app can pass it and still be slow on real requests. I check latency for each endpoint separately, not the average. Then I check the downstream calls, because a slow database or API is usually the real cause. And I check if it is all pages or only one — one page means that page is slow, not the whole app."

---

## 9. Users report DNS failure. Some users can access the site, others cannot.

**Why they ask this**
The important clue is in the question: **some** users, not all users. They want to see if you notice that.

**What that clue means**

If DNS were truly broken, **nobody** could reach the site. Some users working means DNS is not broken. Something is different between the users.

The difference is almost always **caching** or **which DNS server they use**.

**Possible causes**

**1. DNS propagation.** Someone changed the DNS record. DNS servers cache the old answer for a set time. That time is the **TTL** (time to live). Until the TTL expires, some servers still give the old answer.

Tip: lower the TTL *before* you plan a change. Then the change spreads quickly. You cannot lower it after — the old TTL is already cached.

**2. Different DNS servers.** Users do not all use the same DNS server. Some use their internet provider. Some use their company DNS. Some use Google (8.8.8.8). One of these may have a stale or wrong answer while others are correct.

**3. Local cache on the user's own computer.** The operating system caches DNS. The browser caches DNS too. One user's laptop can be stuck on the old address, even after everything else is fixed.

**4. Route 53 routing (on AWS).** This may not be a bug at all. If you use weighted routing, failover routing, or geolocation routing, then **different users get different answers by design**. Check your Route 53 health checks — if one is failing, traffic is being sent away on purpose.

**5. Company DNS.** Users inside an office often go through an internal DNS forwarder. It has its own cache and its own filter rules. It works separately from public DNS.

**How to check**

Compare two DNS servers. This finds the difference fast:

```bash
dig @8.8.8.8 example.com              # public DNS
dig @<company-dns-ip> example.com     # company DNS
dig +trace example.com                # show the full path of the lookup
```

If the two answers are different, you found where the problem is.

**What to say**

> "The clue is that only some users are affected. If DNS were really broken, nobody could reach the site. So it is caching or a specific resolver. I compare dig against a public resolver and against the company resolver. If the answers differ, I know where the stale record is. I also check Route 53, because with weighted or failover routing, different answers can be correct by design."

---

## 10. SSH suddenly stops working.

**Why they ask this**
This has no single answer. They want to watch you remove causes one by one, in a smart order.

**What to check, cheapest first**

**1. Is the server alive?** Check the EC2 status checks in the AWS console. You want to see `2/2 checks passed`. If they fail, the problem is the server or the hardware, not SSH.

**2. Security group.** Is port 22 open, from your IP address?

**3. NACL.** The subnet firewall. It is stateless, so check inbound **and** outbound rules.

**4. Is sshd running?** You need another way in for this (Session Manager, or the serial console):

```bash
systemctl status sshd
```

Did it crash? Did someone edit the config and restart it with a mistake?

**5. Is the disk full?** A full `/` disk stops SSH from writing session files. Login hangs or fails. This connects back to question 4.

**6. Is CPU at 100%?** A very busy server may be too slow to accept new SSH connections, even though sshd is running.

**7. Network path.** Did a route table change? Is the VPN working? Did someone change an ACL somewhere in the middle?

**8. The bastion host.** If you connect through a bastion (a jump server), check the bastion first. The problem may not be the target server at all. Do not assume.

**If you are truly locked out (AWS)**

Two ways in that do **not** use SSH:

- **EC2 Instance Connect**
- **Session Manager (SSM)**

These do not need port 22 and do not need a security group rule. They use an agent on the server. This is a very good answer to give, because it shows you plan for being locked out.

**What to say**

> "I remove causes one by one, starting with the cheapest checks. Is the instance healthy, is port 22 open in the security group, then the NACL in both directions. If I can get in another way, I check sshd itself, and also disk and CPU, because a full disk stops SSH login. If I connect through a bastion, I check the bastion first. And if I am fully locked out, Session Manager works without port 22."

---

## 11. A Kubernetes Pod keeps restarting (CrashLoopBackOff).

**Why they ask this**
This is the most common Kubernetes problem in real life. They want to see if you know the right commands and how to read the output.

**First, what does the name mean?**

- **Crash** — the container starts and then stops
- **Loop** — Kubernetes starts it again, and it stops again
- **BackOff** — Kubernetes waits longer between each try (10s, 20s, 40s, up to 5 minutes) so it does not overload the server

The name describes the *waiting*. It does not tell you the cause. You must find that yourself.

**The first five commands**

```bash
kubectl get pods                                  # status and restart count
kubectl describe pod <pod>                        # events, exit code, limits
kubectl logs <pod>                                # current logs
kubectl logs <pod> --previous                     # logs from the CRASHED container
kubectl get events --sort-by='.lastTimestamp'     # recent cluster events
```

**The most important one is `--previous`.**

Why? The container just restarted. The current logs are empty or almost empty. The error you need is in the container that **already died**. `--previous` shows you those logs. Many beginners miss this and then say "there are no logs".

**How to read the exit code**

Run `kubectl describe pod` and look under `Last State`. You will see an exit code.

| Exit code | What it usually means |
|---|---|
| **137** | The container was killed. Usually **OOMKilled** (out of memory). Also check the `Reason` field. |
| **1** | A general app error. Read the logs for the real message. |
| **127** | Command not found. Wrong entrypoint, or wrong base image. |
| **0** | The app finished normally. But a web server should never finish. Wrong command, or it is not a long-running app. |

**Other common causes**

**Liveness probe too strict.** A liveness probe checks if the app is alive. If it fails, Kubernetes **kills and restarts** the container. If your app takes 40 seconds to start, and the probe starts checking after 10 seconds, Kubernetes kills a perfectly healthy app again and again. Fix: use a **startup probe**, or increase `initialDelaySeconds`.

**Memory limit too low.** The app uses more memory than the limit allows. Kubernetes kills it. Exit code 137. Compare the real usage to the limit before you raise it.

**Readiness probe is different — know this difference.** A readiness probe failing does **not** restart the Pod. It only removes the Pod from the load balancer, so it gets no traffic. So a readiness problem looks like "no traffic", not "restarting". This is a common interview follow-up.

**What to say**

> "First I run kubectl logs with --previous, because the container just restarted and the current logs are empty. The error is in the container that already died. Then I check describe pod for the exit code. 137 usually means out of memory. Exit 1 means an app error. I also check the liveness probe, because a probe that is too strict will kill a healthy app that starts slowly."

---

## 12. One Pod is healthy, another Pod is not. Same Deployment.

**Why they ask this**
This is a logic question. If both Pods have the same image and the same settings, then the code cannot be the difference. So what is?

**The key thought**

Same Deployment means: same image, same config, same command. So the difference must be **outside** the Pod. That means the **node**, or **timing**.

Say this reasoning out loud. It is the whole point of the question.

**What to check**

**1. The node.** This is the most likely cause. Check the node the bad Pod is on:

```bash
kubectl get pod <pod> -o wide            # which node is it on?
kubectl describe node <node>             # check the conditions
```

Look for `MemoryPressure`, `DiskPressure`, or `NotReady`. Also check if other Pods on that node are using a lot of memory. Your Pod may be fine, but the node is full.

**2. Timing during a rollout.** If a deploy is happening right now, some Pods are the old version and some are the new one. They look like the same Deployment but they are not the same image yet.

```bash
kubectl get pods -o wide     # compare the ages
kubectl rollout status deployment/<name>
```

**3. ConfigMap or Secret changed.** This one is subtle and worth knowing. If someone updated a ConfigMap or Secret:

- Pods created **after** the change get the new value
- Pods created **before** the change may still have the old value

So two Pods in the same Deployment can have different config. Compare the Pod age with the time the ConfigMap was changed.

**4. Resource limits plus a busy node.** Your Pod has a memory limit. On a quiet node it stays under the limit. On a busy node, with other Pods using memory, the same Pod may get pushed over the limit and killed. Same Pod, different result, because of the neighbours.

**What to say**

> "Same Deployment means the same image and the same config. So the difference is not the code. It is the node, or the timing. I check which node the bad Pod is on, and look at the node conditions for memory or disk pressure. I also check if a rollout is running, and whether a ConfigMap changed recently, because Pods created before and after a change can hold different values."

---

## 13. The app works on one server but not on another. The code is identical.

**Why they ask this**
The question says the code is the same. So do not read the code. That is the trap. The answer is the **environment** around the code.

**The key sentence to say**

> "Identical code does not mean identical environment."

**What to check — compare the two servers**

Do not read the code. Compare the two servers, item by item.

**1. Environment variables and config files.** The most common cause by far.

```bash
env                        # on server A
diff <(env) <(ssh serverB env)
```

Someone set up server B by hand, or a deploy missed one variable.

**2. Package and library versions.**

```bash
diff <(rpm -qa | sort) <(ssh serverB 'rpm -qa | sort')
# or for Ubuntu/Debian:
diff <(dpkg -l | sort) <(ssh serverB 'dpkg -l | sort')
```

A different version of a library (like OpenSSL) can break the app on one server only.

**3. The clock.** This is a good answer to give, because most people forget it.

```bash
timedatectl
chronyc tracking
```

If the clock is wrong on one server:
- SSL/TLS handshakes fail (the certificate looks expired or not yet valid)
- Tokens fail (JWT tokens have a valid time window)
- Scheduled jobs run at the wrong time

**4. Filesystem and permissions.**

- Is the disk full on one server? (`df -h`)
- Are file permissions or the owner different?
- Is SELinux or AppArmor blocking something on one server only?

**5. The network path.** One server may:
- Have an old DNS answer in its cache
- Take a different route to the database
- Be in a different subnet with a different security group

**What to say**

> "Identical code does not mean identical environment. So I do not read the code. I compare the two servers directly. Environment variables and config files first, because that is the most common difference. Then package versions, then the system clock, because clock skew breaks TLS and tokens. Then permissions and the network path. I diff everything I can rather than guessing."

---

## 14. One ECS task keeps restarting. Desired count is 4. One keeps flapping.

**Why they ask this**
This is the AWS version of the Kubernetes CrashLoopBackOff question. Same thinking, different commands.

**Where to look, in order**

**1. ECS Service Events.** Start here. It gives the high-level reason in plain words.

```bash
aws ecs describe-services --cluster <cluster> --services <service>
```

Look at the `events` list. You will see things like "failed ELB health checks" or "deployment circuit breaker".

**2. The stopped task details.** This is the most useful step. Find the `stoppedReason` field:

```bash
aws ecs describe-tasks --cluster <cluster> --tasks <task-arn>
```

`stoppedReason` usually tells you the answer directly. Common values:

| stoppedReason | Meaning |
|---|---|
| `Essential container in task exited` | The app crashed. Read the container logs. |
| `OutOfMemoryError: Container killed...` | Not enough memory. Raise the limit or fix the leak. |
| `Task failed ELB health checks` | The load balancer thinks it is unhealthy. |
| `CannotPullContainerError` | Bad image name, or no permission to pull from ECR. |

**3. Container logs in CloudWatch.** The task definition says which log group to use. The real application error is here.

**4. Health check settings.** If the health check is too strict, ECS keeps killing a container that is actually fine but starts slowly. Check the grace period (`healthCheckGracePeriodSeconds`). If your app needs 60 seconds to start and the grace period is 30, ECS will kill it every time.

**5. Exit codes.** Same meaning as Kubernetes. 137 usually means memory. Other non-zero codes mean an app error.

**Your own experience**

You have worked on ECS task definition registration and deployments in your pipelines. You have seen a bad revision or a missing environment variable cause exactly this. Use that as your example. A real story is always stronger than theory.

**What to say**

> "I start with the ECS service events, because they give the reason in plain language. Then I look at describe-tasks and read the stoppedReason field, which usually tells me directly if it was memory, a crash, or a failed health check. Then the container logs in CloudWatch for the real error. I also check the health check grace period, because a slow-starting app gets killed if the grace period is too short."

---

## 15. A Docker container exits immediately. `docker run my-app` stops in seconds.

**Why they ask this**
They want to know if you understand the one rule that controls container lifetime.

**The one rule**

A container lives **only as long as its main process runs in the foreground.**

The main process is the `ENTRYPOINT` or `CMD`. It is process number 1 inside the container (called PID 1).

When PID 1 stops, the container stops. Immediately. It does not matter what else is happening inside.

**Common causes**

**1. The command finishes and exits.** For example, the entrypoint is a setup script. The script does its work, reaches the end, and exits. The container stops.

Fix: at the end of the script, use `exec` to start the real app:

```bash
#!/bin/sh
# do setup work here
exec node server.js      # exec replaces the script with the app
```

`exec` matters. Without it, the app runs as a child process, and the script is still PID 1.

**2. The app goes into the background.** Some apps "daemonize" — they start, then move themselves to the background, then the original process exits. Docker sees PID 1 exit and stops the container, even though a child process is still alive.

Fix: run the app in foreground mode. Most apps have a flag for this, like `-g 'daemon off;'` for NGINX.

**3. The app crashes at startup.** A missing environment variable, a missing config file, or a bad connection string. Check the logs:

```bash
docker logs <container-id>
```

This still works after the container stops. But **not** if you used `--rm`, because that deletes the container.

**4. No `CMD` or `ENTRYPOINT` at all**, or the script ends with `exit 0`.

**How to debug it**

Two useful commands:

```bash
docker logs <container-id>              # see the error
docker run -it --entrypoint sh my-app   # get a shell instead of the app
```

The second one is very useful. It skips the entrypoint and gives you a shell inside the image. Now you can run the real start command by hand and see the actual error message.

**What to say**

> "A container only lives as long as PID 1 runs in the foreground. So the question is always: why did PID 1 exit? Usually it is a script that finished instead of starting the app, or an app that moved itself to the background. I check docker logs first, then I run the container with --entrypoint sh and start the app by hand, so I can see the real error."

---

## 16. A GitHub Actions deploy suddenly fails. It worked yesterday.

**Why they ask this**
Your pipeline file did not change. So something *around* it changed. They want to see if you think that way.

**The key sentence**

> "If my config did not change, then something in the environment changed."

**What to check, in order**

**1. Find the exact step that failed.** Read the full log. Compare it to yesterday's successful run, side by side. Find the first line that is different. Do not guess before you do this.

**2. Base image changed.** If your Dockerfile says `FROM node:20`, that tag moves. It pointed to one version yesterday and a new version today. A new patch version can break your build.

Fix: pin the version, or better, pin the digest (`node:20.11.0` or `node@sha256:...`).

**3. A dependency released a new version.** If your `package.json` says `^1.2.0`, npm can install `1.3.0` today. If that release has a bug, your build breaks.

Fix: commit your lock file (`package-lock.json`, `pnpm-lock.yaml`) and use `npm ci` instead of `npm install`.

**4. Credentials expired or changed.**

- Did an OIDC role trust policy change?
- Did a GitHub secret expire?
- Did someone rotate AWS credentials?
- Did a token for a private registry expire?

You have hit exactly this class of problem before, with expired GitHub OAuth tokens breaking a CodeBuild source connection. That is a good real example to use.

**5. The runner image changed.** GitHub updates its hosted runners regularly. A runner update can change the version of a tool you rely on, without you doing anything.

**6. Something changed on the AWS side.** An IAM policy got tightened. A security group changed. An ECR repository policy changed. The pipeline only fails the next time it runs, which can be a day later. This makes it look sudden when it is not.

**What to say**

> "My pipeline file did not change, so I assume something in its environment changed. First I diff the failing step against yesterday's successful log to find the exact line. Then I check the usual suspects: a moving base image tag, an unpinned dependency, expired credentials, or a runner image update. I have seen expired tokens break a build exactly like this, so I check credentials early."

---

## 17. A production deploy causes HTTP 500 errors immediately.

**Why they ask this**
This is a values question, not a technical one. They want to know your **instinct**. Do you debug, or do you fix?

**The correct instinct: roll back first.**

Users are seeing errors right now. Every minute matters. Do not debug in production while users are affected.

**The order**

**1. Roll back.** Go back to the last version you know worked. Roll back the deployment, the task definition, or the image tag.

```bash
kubectl rollout undo deployment/api          # Kubernetes
# ECS: update the service to the previous task definition revision
```

Even faster, if you have it: turn off the feature flag. That takes seconds, not minutes.

**2. Check that it worked.** Watch the error rate go down. Do not assume. Confirm.

**3. Save the evidence.** Before everything is cleaned up:
- Save the logs
- Keep one bad instance out of the load balancer, if you can
- Note the exact build number and time

If you skip this, you cannot find the cause later.

**4. Now investigate, with no pressure.**

- **Logs.** The application stack trace usually points straight at the new code.
- **Database migration.** Did a migration run? Did it finish? Did it fail halfway? Does the new code expect a column that does not exist yet? This is a very common cause.
- **Missing environment variable.** Does the new version need a new secret or variable that nobody added to this environment?

**When would you NOT roll back?**

A good interviewer may ask this. Real answers:

- The release included a database migration you **cannot** undo
- Rolling back brings back a worse bug, or a security hole
- The change was a security fix

Outside those cases, roll back.

**What to say**

> "First priority is stopping the damage, so I roll back. If we have a feature flag for it, I turn that off instead, because it is faster. Then I confirm the error rate is dropping. I save the logs and one bad instance for evidence. Only then do I investigate, with the pressure off. I would not roll back only if the release included a migration I cannot undo."

---

## 18. One EC2 instance is unhealthy behind an ALB. Three are healthy.

**Why they ask this**
Three out of four work. So the load balancer config is probably fine. The difference is on that one instance.

**What to check**

**1. What is the health check actually checking?** Find the path, port, and protocol in the target group settings. Then test that exact path yourself, from another instance:

```bash
curl -v http://<bad-instance-ip>:<port>/health
```

Test the **exact** path the ALB uses. Not `/`. If the ALB checks `/health` and your app only serves `/api/health`, it will always fail.

**2. Security groups.** Can the ALB reach the instance on the health check port?

A common mistake: the app port is allowed, but the health check uses a **different** port on a different listener, and that port is not allowed.

**3. Is the app listening on the right port on that instance?** Check on the instance:

```bash
ss -tulnp | grep <port>
```

Maybe the app restarted on a different port. Maybe the config drifted on this one server.

**4. Is the app actually healthy on that instance?** The instance may have its own local problem:

- Disk full
- Out of memory
- A bad deploy landed only on this one instance
- NGINX stopped on this instance

**A quick way to decide**

| Test result | What it means |
|---|---|
| Direct curl works, ALB says unhealthy | Network or config problem (security group, wrong port, wrong path) |
| Direct curl fails too | The app on that instance is really broken |

**What to say**

> "Three instances are healthy, so the target group config is probably fine. The difference is on that one instance. I curl the exact health check path directly, from inside the network. If that works, it is a network or port problem, so I check the security group on the health check port. If the curl also fails, the app on that instance is broken, and I check disk, memory, and whether the app is listening on the right port."

---

## 19. An SSL certificate suddenly stops working. Error: `NET::ERR_CERT_DATE_INVALID`.

**Why they ask this**
The error message is very specific. They want to see if you read it carefully.

**Read the error name**

`CERT_DATE_INVALID` says **DATE**. So the problem is the date, not the name and not the chain.

This means one of two things:
1. The certificate **expired**
2. The certificate is not valid **yet** (rare), or the client's clock is wrong

Different problems give different errors. A wrong hostname gives `ERR_CERT_COMMON_NAME_INVALID`. A missing chain gives `ERR_CERT_AUTHORITY_INVALID`. So this error narrows things quickly.

**Check the real expiry date**

```bash
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -dates
```

You will see `notBefore` and `notAfter`. Compare `notAfter` to today.

**If you use AWS ACM**

ACM renews certificates automatically. But only if the **DNS validation record is still there.**

Here is how this breaks in real life:

1. ACM adds a CNAME record to your DNS for validation
2. Months later, someone cleans up "unused" DNS records and deletes it
3. Nothing breaks that day. The certificate is still valid.
4. Renewal time comes. ACM tries to renew. It fails silently.
5. The certificate expires. Now the site is down.

So the real cause is a DNS record that was deleted weeks ago. Check that the validation CNAME still exists.

**Which layer is actually serving the certificate?**

This catches people. You may have certificates in several places:

- CloudFront (at the edge)
- The ALB
- The web server on the instance

Users see the certificate from the **first** layer that terminates TLS. If CloudFront is in front, the CloudFront certificate is the one that matters. You can fix the ALB certificate and nothing changes, because the expired one is on CloudFront.

Check which layer you actually reached:

```bash
curl -vI https://example.com 2>&1 | grep -i -A3 'server certificate'
```

**The real fix is not renewing the certificate**

Renewing fixes today. It does not stop it happening again.

The real fixes are:
1. **Automate renewal** (ACM, or cert-manager)
2. **Monitor the expiry date separately** from the renewal system

Point 2 matters. Do not just monitor the renewal job. Check the **actual certificate that users receive**, and alert 30 days and 14 days before it expires. Because the failure mode is a renewal job that thinks it succeeded.

**What to say**

> "The error says DATE, so it is expiry, not a name or chain problem. I check the real expiry with openssl s_client. If we use ACM, I check that the DNS validation record still exists, because ACM renewal fails silently if someone deleted it. I also check which layer serves the certificate, because CloudFront and the ALB can have different ones. And the real fix is monitoring the served certificate expiry, not just renewing it once."

---

## 20. Disk usage grows every night. 60% → 70% → 80% → 95%.

**Why they ask this**
The pattern is the answer. Growth **every night** at the same time is not random. It is a scheduled job.

**The key thought**

Say this first: **"Growth at a regular time means a scheduled job, not normal app growth."**

Normal app growth is slow and steady all day. Growth in steps, every night, means something runs on a timer.

**What to check**

**1. Scheduled jobs.** Look in all the places jobs can hide:

```bash
crontab -l                    # jobs for the current user
sudo crontab -l -u <user>     # jobs for other users — easy to miss
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
systemctl list-timers         # systemd timers, the modern way
```

The most common cause is a nightly backup, report, or archive job.

**2. Log rotation.** Is it configured, and is it actually working?

```bash
cat /etc/logrotate.conf
ls /etc/logrotate.d/
cat /var/lib/logrotate/status    # when did it last run?
```

A very common combination: an app that logs a lot during a nightly batch job, plus log rotation that is broken or missing. Logs grow every night and nothing cleans them.

**3. Local backups.** A job dumps a backup to local disk, then uploads it. Two ways this fills the disk:

- It does not delete the local copy after uploading
- The upload fails silently, so the local copy stays behind every single night

**4. Docker logs.** Docker's default log driver (`json-file`) has **no size limit**. A container that logs a lot will grow forever.

Fix it in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "100m", "max-file": "3" }
}
```

**5. Temporary files.** A batch job creates temp files and does not clean them up. Or it only cleans up when it **succeeds** — so every failed run leaves files behind forever.

**How to find it fast**

Check right after one of the nightly jumps:

```bash
du -sh /var/log /var/backups /tmp /var/lib/docker | sort -rh
find / -type f -mtime -1 -size +100M 2>/dev/null    # big files made in the last day
```

The second command is very useful. It finds big new files. Then compare the file time to your cron schedule. That gives you the job.

**What to say**

> "Growth at a regular time means a scheduled job, not normal growth. So I check crontab for all users, and also systemd timers, because people forget those. Then I check whether log rotation is actually running. Then I look for a backup job that does not clean up its local copy, and Docker logs, because the default json-file driver has no size limit. To find it fast, I use find with mtime to list big new files and match the time to the cron schedule."

---

## 21. Latency suddenly increases. p95 goes from 40ms to 700ms.

**Why they ask this**
The word "latency" makes people say "network". Usually it is not the network. They want to see if you check the likely causes first.

**First, what is p95?**

p95 means: 95% of requests were faster than this number. 5% were slower.

Use p95 and p99, not the average. The average hides problems. If 99 requests take 10ms and one takes 5 seconds, the average is 60ms — which looks fine, but one user had a terrible experience.

**What to check, most likely first**

**1. The database.** The most common cause. Check:
- Slow queries (slow query log, or `pg_stat_statements`)
- Lock contention (one long transaction blocking many others)
- Connection pool full (requests wait for a free connection)

**2. Garbage collection (GC).** For Java, Go, Node, and similar. GC is automatic memory cleanup. Some GC pauses stop the whole program for a moment. From the outside this looks exactly like slow network. Check GC pause times for the same minute as the spike.

**3. CPU saturation.** If CPU is full, requests wait in a queue before they even start being processed. The work itself is fast; the waiting is slow.

Also check **CPU throttling** if you run in containers. A container with a CPU limit gets slowed down by the kernel, even when the server has free CPU. Very easy to miss.

**4. Disk.** If the request path writes to disk (for example, synchronous logging), a slow disk adds directly to request time.

**5. A downstream API.** If your service calls another service, their slowness becomes your slowness. Check the latency of every call you make.

**6. The real network.** Yes, check it — but check it **last**. Real network problems (packet loss, a route change, MTU mismatch) do happen, but they are less common than the five causes above.

```bash
mtr <host>              # latency and packet loss per hop
ip -s link              # interface errors and drops
```

**The best technique: correlate the time**

This is the most useful skill here. Take the exact minute the latency jumped. Then look at every other graph at that same minute:

- Did CPU jump at 14:32? → CPU problem
- Did GC pause time jump at 14:32? → GC problem
- Did database latency jump at 14:32? → database problem
- Did a deploy happen at 14:32? → the deploy caused it

The graph that moves at the same time is your answer. This is much faster than guessing.

**What to say**

> "A p95 jump like this is usually not the network, even though it sounds like it. I check the database first, because slow queries or a full connection pool is the most common cause. Then GC pauses and CPU, including CPU throttling if we are in containers. Then downstream API latency. The fastest method is to take the exact minute of the spike and compare every other graph at that minute. Whatever moved at the same time is the cause."

---

## 22. Logs stop appearing in Grafana. The app is healthy. (Loki and Promtail)

**Why they ask this**
The app is fine. So the problem is in the pipeline that **moves** the logs, not in the app.

**First, understand the path**

Logs travel like this:

```
App writes log file  →  Promtail reads the file  →  Promtail sends to Loki  →  Grafana reads from Loki
```

Four steps. The break is in one of them. Check each step in order.

**What to check**

**1. Is Promtail running?**

```bash
systemctl status promtail
journalctl -u promtail -n 100      # read Promtail's own logs
```

Promtail's own logs usually say what is wrong. Read them before guessing.

**2. Can Promtail read the file?** This is a common cause. A deploy changed the user the app runs as. Now the log file has a different owner, and Promtail cannot read it.

```bash
ls -la /var/log/myapp/           # who owns the file?
ps aux | grep promtail           # who does Promtail run as?
```

**3. Is the file path still correct?** Did the app change where it writes logs? Promtail is looking at the old path and finding nothing.

**4. Can Promtail reach Loki?** Test the network:

```bash
curl -v http://loki:3100/ready
```

Check DNS, and check whether a firewall or security group rule changed recently.

**5. Did the labels change?** This is a subtle one worth knowing.

Labels are tags on the logs, like `app=api` or `env=prod`. Your Grafana query filters on labels. If the label names changed on the Promtail side, the logs are arriving fine — but your query no longer matches them.

Test this in Grafana: query with **no filter at all**, or use the label browser to see which labels exist now. If logs appear without the filter, the labels changed.

**6. Is Loki itself the problem?** Loki may be dropping logs, not Promtail.

- Loki has **rate limits**. If you send too much, it rejects logs. Check Loki's logs for "rate limit" or "429".
- Loki's storage backend (S3, or disk) may be full or unreachable.

**The order that saves time**

Test it from the end backwards, or check the simplest thing first:

1. Query Grafana with no filter → if logs appear, it is a **label** problem
2. Check Promtail is running and read its logs → they usually say the answer
3. Check file permissions → very common
4. Check network to Loki
5. Check Loki for rate limits

**What to say**

> "The app is healthy, so the break is in the log pipeline, not the app. The path is app file, then Promtail, then Loki, then Grafana. I check each step. First I query Grafana with no filter, because if logs appear then the labels changed and nothing is actually broken. Then I read Promtail's own logs, which usually say the problem. File permissions are a common cause after a deploy changes the app user. Then network to Loki, and finally Loki rate limits."

---

## 23. Prometheus stops scraping a target. It shows "Target Down".

**Why they ask this**
There is one step that answers this almost immediately. They want to see if you know it.

**Start here — it usually gives you the answer**

Open Prometheus. Go to **Status → Targets**. Find your target. Prometheus shows the **exact error message**.

Do not guess before you read this. The error tells you the answer:

| Error message | What it means | What to check |
|---|---|---|
| `connection refused` | Nothing is listening on that port | Is the exporter running? |
| `context deadline exceeded` | Too slow to answer (timeout) | Is the target overloaded? Is the timeout too short? |
| `no such host` | DNS failed | Did the hostname change? |
| `x509: certificate has expired` | TLS certificate problem | Renew the exporter's certificate |
| `i/o timeout` | Network is blocked | Firewall or security group |
| `404 Not Found` | Wrong path | Is it `/metrics` or a custom path? |

**Then check, based on the error**

**1. Is the exporter running?** Test it directly on the target machine:

```bash
curl localhost:9100/metrics      # example: node_exporter
```

If this fails on the machine itself, the exporter is the problem. No network involved.

**2. Can Prometheus reach the port?** Test from the Prometheus machine:

```bash
nc -zv <target-ip> 9100
```

If the local curl works but this fails, it is a firewall or security group.

**3. Service discovery.** If you use dynamic discovery (Kubernetes, EC2, Consul), Prometheus finds targets by labels or tags. Two things break this:

- Someone removed the tag or label that the discovery config looks for
- The target's IP changed and discovery has not refreshed yet

Check your `prometheus.yml` discovery rules against the target's current tags.

**4. Did the metrics path change?** An app update can move `/metrics` to a new path. Prometheus still looks at the old one and gets a 404.

**What to say**

> "The first thing I do is open Status and Targets in Prometheus and read the actual error message, because it almost always tells me the answer directly. Connection refused means the exporter is not running. A timeout means it is too slow. An x509 error means a certificate. Then I curl the metrics endpoint on the target itself. If that works but Prometheus cannot reach it, it is a firewall. If we use service discovery, I also check whether a tag or label was removed."

---

## 24. RDS CPU is at 95%. The database is slow.

**Why they ask this**
They want you to find the *cause* of the CPU use, not just say "CPU is high, make the instance bigger".

**What to check**

**1. Find the expensive query.** This is the first and most important step. Usually one or two queries cause most of the load.

For PostgreSQL:

```sql
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Order by **total** time, not average time. A query taking 5ms but running 100,000 times per minute uses more CPU than a query taking 2 seconds once. Beginners often miss this.

For MySQL, use the slow query log.

**2. Check the connection count.**

```sql
SELECT count(*) FROM pg_stat_activity;
SHOW max_connections;
```

Too many connections costs CPU by itself, even when they are idle. This happens when the app scales out — every app instance opens its own pool, so 10 instances × 20 connections = 200 connections.

Fix: use a connection pooler like **PgBouncer** or **RDS Proxy**.

**3. Check for locks.** One long transaction can block many queries. From outside, this looks like general high CPU, but the real cause is one blocking query.

```sql
SELECT blocked.pid AS blocked_pid,
       blocking.pid AS blocking_pid,
       blocked.query AS blocked_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

Also look for sessions stuck in `idle in transaction`. Those hold locks and do nothing.

**4. Check for a missing index.** Take the worst query from step 1 and look at its plan:

```sql
EXPLAIN ANALYZE SELECT ...;
```

If you see `Seq Scan` on a large table, an index is probably missing. A sequential scan reads every row. That burns CPU.

**5. Check storage, not just CPU.** This one is easy to miss.

- **gp2 volumes** have **burst credits**. When the credits run out, disk speed drops a lot. Latency gets worse and CPU can look high because queries take longer. Check the `BurstBalance` metric in CloudWatch.
- If you have hit your provisioned IOPS limit, you get the same effect.

So always check the storage metrics next to CPU. A "CPU problem" is sometimes a disk problem.

**Quick fixes vs real fixes**

| Quick fix (now) | Real fix (later) |
|---|---|
| Kill the worst long-running query | Add the missing index |
| Send reads to a read replica | Fix the N+1 query in the app |
| Add a connection pooler | Reduce the number of connections |
| Make the instance bigger | Optimise the query |

Making the instance bigger works, but it hides the problem and costs more every month. Mention it as a temporary step, not the answer.

**What to say**

> "First I find the expensive query with pg_stat_statements, and I order by total time, not average, because a fast query running very often can use more CPU than a slow one running once. Then I check connections, because too many connections costs CPU by itself. Then locks, then the query plan for a missing index. I also check the storage metrics, because on gp2 an empty burst balance makes everything slow and it looks like a CPU problem. Making the instance bigger is a temporary step, not the fix."

---

## 25. Complete production outage. The CEO is calling. What do you do in the first 30 minutes?

**Why they ask this**
This is **not** a technical question. They are testing whether you can stay organised and communicate when there is pressure. Most technical people forget the communication part. That is exactly what is being graded.

**The most important point**

Your first action is **not** a command. Your first action is to **declare the incident**.

Why? Because in a real outage, three people quietly making changes at the same time is dangerous. Someone restarts a service while someone else is taking a diagnostic dump. Someone rolls back while someone else rolls forward. You need one place where everyone can see what is happening.

**The first 30 minutes, in order**

**1. Declare the incident.** Open an incident channel. Create a ticket. Make it official. Say clearly who is coordinating.

**2. Assign a coordinator.** One person coordinates. They do **not** debug. They delegate, they decide, and they keep the picture clear. If you are the only person, you do both — but know which hat you are wearing.

**3. Check the real impact.** Do not assume "everything is down". Check:
- Which services?
- Which regions?
- All users, or some users?

You may find that 80% still works. That changes the plan and it changes what you tell people.

**4. Look at your dashboards and alerts.** Which alert fired **first**? That is your best clue. Let your tools tell you where to look instead of starting from nothing.

**5. Ask what changed.** This is the highest-value question in any outage. Check:
- Recent deploys
- Config changes
- Infrastructure changes
- Certificate expiry
- A scheduled job that just ran

Most incidents follow a change.

**6. Mitigate. Do not root-cause yet.** Pick the fastest safe way to restore service:
- Roll back
- Fail over
- Scale up
- Turn off a feature flag

You do **not** need to understand the cause to fix the symptom. Understanding comes later.

**7. Communicate on a fixed schedule.** Say this at the start: "I will post an update every 15 minutes."

Then post every 15 minutes, **even if there is no news**. "Still investigating, no change, next update at 14:45" is a real update. Silence makes people panic and they will interrupt you constantly.

Also separate facts from guesses:
- Fact: "Error rate is 40% on checkout."
- Guess: "We think it may be the cache."

Label them differently. If your guess is wrong and you said it like a fact, you lose trust.

**8. After recovery, write the postmortem.** The incident is not finished when the errors stop. It is finished when there is a documented cause and a fix that prevents it happening again.

**About the CEO calling**

Handle it directly: give the impact, the current status, and the update time. Then get back to work.

> "About 60% of users cannot check out. We rolled back the 14:20 deploy two minutes ago and error rate is falling. Next update at 14:45."

That is short, honest, and it stops the next five phone calls.

**What to say**

> "My first move is not a command. It is declaring the incident, so we have one place to coordinate and people are not making changes at the same time without knowing. Then I check the real impact instead of assuming everything is down, look at which alert fired first, and ask what changed recently. Then I mitigate with the fastest safe option, usually a rollback. I do not need the root cause to restore service. And I set an update schedule of 15 minutes so stakeholders are informed and I am not interrupted every two minutes."

---

## If you have little time: study these first

These seven cover the most common questions and teach you the thinking you can reuse everywhere.

| Priority | Question | Why |
|---|---|---|
| 1 | **Q1 — Server is slow** | The base method. Everything else uses it. |
| 2 | **Q2, Q3, Q4, Q5 — CPU, memory, disk, load** | The four Linux basics. Asked constantly. |
| 3 | **Q7, Q8 — 502 and 504** | Very common in real work. Easy to explain well. |
| 4 | **Q10 — SSH not working** | Pure step-by-step elimination. Tests method, not memory. |
| 5 | **Q11, Q14 — CrashLoopBackOff and ECS restart** | Same logic, two platforms. You have real experience here. |
| 6 | **Q17 — Deploy causes 500s** | Tests the "fix first, debug later" instinct. |
| 7 | **Q25 — Full outage** | Almost always asked for senior roles. Tests communication. |

---

## Word list

Terms used in this file, explained simply.

**Latency** — how long one request takes.

**p95 / p99** — p95 means 95% of requests were faster than this. Use these instead of the average, because the average hides slow requests.

**Root cause** — the real reason something broke, not just the first symptom you see.

**Mitigate** — stop the damage now, even if you do not know why it happened yet. (Roll back, fail over, scale up.)

**Rollback** — go back to the previous version that worked.

**Failover** — switch to a backup server automatically when the main one fails.

**Load average** — how many processes are waiting. It counts processes waiting for CPU **and** processes waiting for disk.

**State D (uninterruptible sleep)** — a process stuck waiting for the disk. It uses no CPU while waiting.

**PID** — process ID, the number Linux gives each running program.

**PID 1** — the first process. Inside a container, if PID 1 stops, the container stops.

**Upstream / downstream** — upstream is the service behind your proxy (the app). Downstream is a service your app calls. People use these words in different directions, so it is fine to say "the app behind NGINX" to be clear.

**Reverse proxy** — a server in front of your app that receives requests and passes them on. NGINX is often used this way.

**Health check** — an automatic test that asks "are you working?" If it fails, traffic stops going to that server.

**Liveness probe** — a health check that **restarts** the container when it fails.

**Readiness probe** — a health check that **removes traffic** when it fails. It does not restart anything.

**Startup probe** — a health check for slow-starting apps. It gives the app time to start before the other probes begin.

**Security group** — a firewall on an AWS resource. It is **stateful**: if you allow traffic in, the reply is automatically allowed out.

**NACL** — a firewall on an AWS subnet. It is **stateless**: you must allow both directions yourself.

**OOM (out of memory)** — no memory left. The **OOM Killer** is the part of Linux that kills a program to save the server.

**OOMKilled** — the container was killed for using too much memory. Exit code 137.

**GC (garbage collection)** — automatic memory cleanup in languages like Java, Go, and Node. Some GC pauses stop the program for a short time, which looks like slow network.

**Exit code** — the number a program returns when it stops. 0 means success. 137 usually means killed for memory. 1 means a general error.

**Cron job** — a task that runs automatically on a schedule.

**Inode** — a small record Linux uses for each file. You can run out of inodes while still having free space, if you have millions of tiny files. Check with `df -i`.

**TTL (time to live)** — how long a DNS answer is cached before it is looked up again.

**Connection pool** — a set of ready-made database connections that an app reuses. If it is full, new requests wait.

**Throttling** — being slowed down on purpose, usually because you hit a limit. Container CPU throttling makes an app slow even when the server has free CPU.

**Burst credits** — on AWS gp2 disks and T-family instances, you get saved-up performance you can use in short bursts. When credits run out, performance drops sharply.

**Node** — one server in a Kubernetes cluster.

**Pod** — the smallest unit in Kubernetes. One or more containers that run together.

**Target group** — on AWS, the list of servers an ALB sends traffic to, with their health status.

**Postmortem** — a written review after an incident. It explains what happened and what will be fixed. **Blameless** means it looks for system problems, not for a person to blame.

---

## One last tip

Interviewers grade **how you explain**, as much as the answer itself.

Practise saying the "What to say" paragraphs out loud. Not to memorise the words — to get comfortable with the shape:

> "First I check **X**, because that tells me **Y**. If Y is true, I go to A. If not, I go to B."

That sentence pattern works for every question in this file. Use it and you will sound organised, even for a problem you have never seen before.
