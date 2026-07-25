# Linux SRE Interview Roadmap — Answers

---

## Module 1: Linux Fundamentals

### Linux boot process

**Quick summary:** BIOS/UEFI → GRUB → Kernel + initramfs → mount real root → systemd (PID 1) → targets/services.

#### 1. BIOS/UEFI
- **BIOS** (legacy): runs POST (Power-On Self-Test), checks hardware, reads the **MBR** (first 512 bytes of the boot disk) — MBR holds tiny bootstrap code pointing to the actual bootloader.
- **UEFI** (modern): more capable, reads bootloader info from the **EFI System Partition** (a FAT32 partition, `/boot/efi` on Linux), supports larger disks (MBR is capped at 2TB), and enforces **Secure Boot** (cryptographically verifies the bootloader/kernel before running).
- **Interview trap**: know the partition table difference — MBR (legacy) vs **GPT** (GUID Partition Table, required for UEFI, supports >2TB disks and more partitions). `lsblk -o NAME,PTTYPE` or `parted -l` shows which one a disk uses.

#### 2. Bootloader (GRUB2)
- Reads its config from `/boot/grub2/grub.cfg` (or `/boot/grub/grub.cfg` on Debian/Ubuntu) — usually auto-generated from `/etc/default/grub` + scripts in `/etc/grub.d/`, via `grub2-mkconfig` / `update-grub`.
- Presents the boot menu (kernel version selection, recovery mode), then loads the selected **kernel image** + **initramfs image** into memory and hands off control.
- **Real-world scenario interviewers like**: "System won't boot after a kernel update — how do you recover?" → Boot into the GRUB menu, select the previous working kernel entry (GRUB usually keeps old kernels around), then investigate/fix the new kernel or its initramfs before making it default again.

#### 3. Kernel initialization
- The kernel image is compressed (`vmlinuz`) — first step is self-decompression into memory.
- Kernel initializes core subsystems: CPU, memory management, and **loads essential drivers** — but at this point it doesn't yet have access to the real root filesystem (which might be on LVM, RAID, encrypted, or need a driver not compiled into the kernel itself).
- That's the whole reason step 4 (initramfs) exists.

#### 4. initramfs (initial RAM filesystem)
- A small, temporary root filesystem loaded entirely into RAM, containing just enough — kernel modules, tools like `lvm`, `mdadm`, `cryptsetup` — to **find, assemble, and mount the real root filesystem**.
- Once the real root is mountable, the kernel does a `pivot_root` (switches the root filesystem from the initramfs to the real disk) and hands off execution.
- **Why this matters practically**: this is the layer where LVM, RAID, or disk-encryption startup failures show up. If you resize/add a disk with LVM and forget to regenerate the initramfs (`dracut` on RHEL/Fedora, `update-initramfs` on Debian/Ubuntu), the new config may not be picked up at boot and the system can fail to find its root volume.
- **Common interview question**: "What's the difference between initrd and initramfs?" → `initrd` (older) was an actual disk image mounted as a block device; `initramfs` (current) is a cpio archive unpacked directly into a RAM-based tmpfs — faster, more flexible, no fixed size limit.

#### 5. init process (PID 1) — systemd
- Once the real root is mounted, the kernel executes `/sbin/init` (usually a symlink to `systemd` on modern distros) as **PID 1**.
- PID 1 is special: it's the ancestor of every other process, and it's responsible for reaping orphaned processes (re-parented to it) so they don't become permanent zombies.
- If PID 1 crashes, the kernel panics — there's nothing above it to restart it.

#### 6. systemd targets
- Targets are systemd's replacement for old SysV **runlevels**. Examples: `rescue.target` (single-user/recovery), `multi-user.target` (normal, non-graphical, most servers run this), `graphical.target` (adds a display manager on top of multi-user).
- Services declare which target they belong to via `WantedBy=` in their unit file, and systemd starts everything in a target **in parallel where dependencies allow** (a major performance improvement over SysV init's strictly sequential startup) — ordering is only enforced where `Before=`/`After=` is explicitly declared.
- Check current target: `systemctl get-default`. Switch (e.g., troubleshooting without a GUI eating resources): `systemctl set-default multi-user.target` then reboot, or `systemctl isolate multi-user.target` to switch live.

#### Follow-up questions interviewers tend to layer on
- **"What happens if `/etc/fstab` has a bad entry?"** → Boot can hang waiting to mount it, or drop to an emergency shell (systemd has a timeout, then offers emergency mode) — fixable by editing fstab from that emergency shell or a rescue boot.
- **"How would you debug a slow boot?"** → `systemd-analyze` (total boot time), `systemd-analyze blame` (which units took longest), `systemd-analyze critical-chain` (the dependency chain that determined total boot time).
- **"What's the very first user-space process, and how do you prove it?"** → `ps -p 1 -o comm=` → should print `systemd` (or `init` on older systems).

**Interview line**: "BIOS → GRUB → Kernel + initramfs → mount real root → systemd (PID 1) → targets/services."

### Kernel space vs User space
- **Kernel space**: privileged mode, direct hardware access, runs the kernel, drivers, core scheduling/memory management. A crash here can crash the whole system.
- **User space**: where applications run, restricted access — must go through **system calls** to request kernel services (file I/O, network, memory allocation).
- This separation is enforced by CPU privilege rings (ring 0 = kernel, ring 3 = user on x86).

### Process lifecycle
`fork()` → creates child process (copy of parent) → `exec()` → replaces child's memory image with a new program → process runs → `exit()` → parent calls `wait()`/`waitpid()` to reap the exit status → process removed from process table.

If parent doesn't call `wait()`, child becomes a **zombie**. If parent dies before child, child becomes an **orphan**, adopted by PID 1.

### Process states
| State | Meaning |
|---|---|
| R (Running) | On CPU or ready to run |
| S (Sleeping, interruptible) | Waiting for an event (e.g., I/O, signal) — can be woken by a signal |
| D (Uninterruptible sleep) | Usually waiting on disk I/O — **cannot** be killed or interrupted by signals; heavy D-state processes = disk bottleneck |
| T (Stopped) | Suspended (e.g., via `Ctrl+Z` or `SIGSTOP`) |
| Z (Zombie) | Finished execution, waiting for parent to reap exit status |

### Threads vs Processes
- **Process**: independent memory space, own PID, isolated — heavier to create/switch.
- **Thread**: shares the same memory space/address space and file descriptors as its parent process, lighter weight, faster context switch, but a crash in one thread can corrupt the whole process.
- On Linux, both are created via `clone()` — threads just share more resources (memory, file descriptor table) than a full process fork.

### Signals
Software interrupts sent to a process to notify it of an event.
- `SIGTERM (15)` — polite request to terminate; process **can** catch/ignore/cleanup.
- `SIGKILL (9)` — immediate termination; **cannot** be caught, blocked, or ignored — kernel removes it directly.
- `SIGHUP (1)` — often used to tell a daemon to reload config.
- `SIGINT (2)` — Ctrl+C.
- `SIGSTOP`/`SIGCONT` — pause/resume.
- Best practice in production: always send `SIGTERM` first, give the app a grace period to clean up (close DB connections, finish in-flight requests), then `SIGKILL` if it doesn't exit.

### System calls
The interface user-space programs use to request kernel services — e.g. `read()`, `write()`, `open()`, `fork()`, `execve()`, `socket()`. You can trace them live with `strace -p <pid>`.

### Daemons
Background processes with no controlling terminal, typically started at boot, running continuously to provide a service (e.g. `sshd`, `nginx`, `cron`). Traditionally they double-fork to detach from the terminal; under systemd, this is managed declaratively via unit files.

### systemd
Modern init system (PID 1) and service manager. Key concepts:
- **Unit files** (`.service`, `.socket`, `.timer`, `.mount`) define how to start/stop/manage a resource.
- **Targets** group units (replace old runlevels), e.g. `multi-user.target`.
- Dependency management via `Requires=`, `After=`, `Wants=`.
- Commands: `systemctl start/stop/restart/status/enable/disable`, `journalctl -u <unit>`.

---

## Module 2: Process Management

### How do you find a high CPU process?
`top` or `htop` (sorted by `%CPU` by default) — press `P` in top to sort by CPU explicitly.
For a snapshot without a live UI: `ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head`.
For per-core breakdown: `mpstat -P ALL 1`.
To see which threads within a process are hot: `top -H -p <pid>` or `pidstat -t -p <pid> 1`.

### How do you kill a process gracefully?
`kill -15 <pid>` (or just `kill <pid>`, since SIGTERM is default) — allows the process to catch the signal, flush buffers, close connections, then exit cleanly. Only escalate to `kill -9 <pid>` if it doesn't respond after a reasonable grace period (this is exactly how Kubernetes/ECS do container shutdown — SIGTERM, wait `terminationGracePeriodSeconds`, then SIGKILL).

### SIGTERM vs SIGKILL
- **SIGTERM (15)**: catchable, ignorable, allows graceful shutdown/cleanup.
- **SIGKILL (9)**: not catchable, not ignorable, immediate — kernel just deallocates the process. Risk: no cleanup, can leave locks/files/DB transactions in a bad state.

### Zombie process
Child has finished executing (exited), but its exit status hasn't been read/reaped by the parent (`wait()` not called) — the process table still holds a slim entry (PID, exit code) until reaped. **You cannot kill a zombie** — there's no live process to signal. Fix: signal or fix the **parent** to reap it (or if the parent is unresponsive, kill the parent so init/systemd adopts and reaps the zombie).

### Orphan process
Parent exited/died before the child did. The child is re-parented to PID 1 (init/systemd), which will properly reap it when it eventually exits. Orphans are not inherently a problem — they're just parentless, unlike zombies which are dead-but-unreaped.

### Nice vs Renice
- **`nice`**: set the initial scheduling priority when starting a process. Range -20 (highest priority) to +19 (lowest). `nice -n 10 command`.
- **`renice`**: change the priority of an already-running process. `renice -n 5 -p <pid>`.
- Lower nice value = higher priority (gets more CPU time relative to others). Only root can set negative (higher-priority) values.

### Load Average
Represents the average number of processes in the **run queue** (state R) plus **uninterruptible sleep** (state D) over 1, 5, and 15 minutes (`uptime` or `top` header).
- Load average of 4 on a 4-core box = fully utilized, no queueing.
- Load average of 8 on a 4-core box = processes are waiting for CPU — bottleneck.
- **Important gotcha**: high load doesn't always mean CPU-bound — a lot of D-state (disk I/O wait) processes will also spike load average even if CPU itself is idle. Always cross-check with `vmstat`/`iostat` before concluding "CPU problem."

---

## Module 3: Memory

### Virtual Memory
Abstraction the kernel gives every process: each process sees its own full, contiguous address space, regardless of physical RAM layout. The MMU (memory management unit) + page tables translate virtual addresses to physical ones. Enables isolation between processes and lets the OS use swap transparently.

### Physical Memory
The actual RAM installed. Managed by the kernel in fixed-size **pages** (typically 4KB), allocated to processes on demand.

### Swap
Disk space used as an overflow for RAM when physical memory is under pressure — inactive pages get written to swap to free up RAM for active processes. Swapping is **slow** (disk vs RAM latency), so heavy swap usage is a red flag for memory pressure. Check with `free -h` (`Swap` row) or `vmstat` (`si`/`so` columns for swap-in/swap-out rates).

### OOM Killer
When the system is critically low on memory and can't reclaim enough, the kernel invokes the **Out-Of-Memory killer**, which picks a process to kill based on an `oom_score` (considers memory usage, priority, and `oom_score_adj`) to free memory and keep the system alive. Evidence of an OOM kill: check `dmesg` or `/var/log/messages` for `Out of memory: Killed process...`. You can protect critical processes by adjusting `/proc/<pid>/oom_score_adj` (lower = less likely to be killed; `-1000` = never killed).

### Page Cache
The kernel caches recently-read/written file data in unused RAM so future access is fast (avoids re-reading from disk). This is why `free -h` shows low "free" memory on a healthy system — most of it is reclaimable page cache, not actually "used" in a problematic sense.

### Buffers vs Cache
- **Buffers**: cache for raw block-device I/O metadata (e.g., filesystem metadata).
- **Cache (page cache)**: cache for file contents read from disk.
- Both are reclaimable under memory pressure — the kernel will evict them before invoking OOM killer. In `free -h`, the `available` column already accounts for this reclaimability — that's the number to actually trust, not `free`.

### Memory Leak
A process continuously allocates memory without releasing it, causing RSS (resident memory) to grow unbounded over time until it's OOM-killed or exhausts the system. Diagnose by watching a process's memory over time (`ps -o rss -p <pid>` repeatedly, or `pidstat -r`), or with language-specific tools (heap dumps, `valgrind`, Node's `--inspect` + heap snapshot, etc.).

---

## Module 4: Filesystem

### Inode
A data structure storing metadata about a file — permissions, owner, size, timestamps, and pointers to the data blocks on disk. The filename itself is **not** stored in the inode; it lives in the directory entry, which maps a name to an inode number. Every filesystem has a **fixed number of inodes** set at creation — you can run out of inodes (many tiny files) even with disk space free. Check with `df -i`.

### Hard Link vs Soft Link (Symlink)
- **Hard link**: another directory entry pointing to the **same inode**. Deleting the original file doesn't remove the data as long as one hard link remains (inode's link count > 0). Cannot cross filesystems, cannot link directories.
- **Soft link (symlink)**: a separate file containing a *path* to the target. Can cross filesystems, can link directories, but breaks if the target is moved/deleted ("dangling symlink").

### Mount points
Directories where a separate filesystem/partition/device is attached into the overall directory tree (e.g., `/mnt/data`, `/home` as a separate partition). View current mounts with `mount` or `findmnt`; persistent config lives in `/etc/fstab`.

### ext4 vs xfs
| | ext4 | xfs |
|---|---|---|
| Max filesystem/file size | Smaller (still huge — 1 EiB/16 TiB practical) | Larger, better at very large scale |
| Shrinking | Supported | **Cannot shrink** an XFS filesystem once created |
| Performance | Good general purpose | Better for large files, high parallel I/O (common on RHEL/CentOS default) |
| Maturity/tooling | Very mature, widely default (Debian/Ubuntu) | Default on RHEL/CentOS 7+ |

### LVM (Logical Volume Manager)
Adds a flexible layer between physical disks and filesystems: **Physical Volumes (PV)** → grouped into **Volume Groups (VG)** → carved into **Logical Volumes (LV)**, which get filesystems. Lets you resize volumes, add disks, and take snapshots without unmounting or repartitioning. Commands: `pvcreate`, `vgcreate`, `lvcreate`, `lvextend` + `resize2fs`/`xfs_growfs`.

### Disk full troubleshooting
1. `df -h` — confirm which mount is full.
2. `du -sh /* 2>/dev/null | sort -rh` — find what's consuming it (walk down directories).
3. **Classic gotcha**: `df` shows 100% full but `du` doesn't add up to that much → a process is holding a file open that's already been deleted (common with log files rotated incorrectly). Find it with `lsof +L1` or `lsof | grep deleted` — the space isn't freed until the process closes the file descriptor or is restarted.
4. Also check `df -i` in case it's actually inode exhaustion, not space, giving a "no space left on device" error despite `df -h` showing free space.

### File permissions
`rwx` for owner/group/other, represented numerically (e.g. `755` = rwxr-xr-x) or symbolically. `chmod`, `chown`, `chgrp`. Special bits:
- **SUID**: file runs with the **owner's** privileges regardless of who executes it (e.g. `passwd` runs as root so any user can update `/etc/shadow`).
- **SGID**: on a file, runs with the group's privileges; on a directory, new files inherit the directory's group.
- **Sticky bit**: on a directory (e.g. `/tmp`), only the file's owner (or root) can delete/rename it, even if others have write permission on the directory.

### ACL (Access Control Lists)
Extends beyond the basic owner/group/other model — lets you grant specific permissions to specific additional users/groups on a file or directory. `getfacl file`, `setfacl -m u:username:rwx file`. Useful when standard Unix permissions aren't granular enough (e.g., three different teams needing different access levels to the same directory).

---

## Module 5: Performance

### CPU bottleneck
Symptoms: high `%CPU` in `top`, high load average with CPU actually busy (not D-state), high `us`/`sy` in `vmstat`. Tools: `top`/`htop`, `mpstat -P ALL` (per-core), `pidstat -u`.

### Memory bottleneck
Symptoms: high swap usage/swap-in-out activity, low `available` memory, OOM kills in `dmesg`. Tools: `free -h`, `vmstat` (`si`/`so`), `pidstat -r`, `/proc/meminfo`.

### Disk bottleneck
Symptoms: high `%util` and high `await` (latency) in `iostat -x`, lots of processes in D state. Tools: `iostat -x 1`, `iotop` (per-process disk usage), `sar -d`.

### Network bottleneck
Symptoms: high retransmits, dropped packets, high latency. Tools: `ss -s` (socket summary), `sar -n DEV`, `iftop`/`nload` for throughput, `tcpdump` for packet-level inspection, `ping`/`mtr` for latency/loss.

### Key performance commands
| Command | Purpose |
|---|---|
| `top`/`htop` | Live overview: CPU, memory, per-process usage |
| `vmstat 1` | CPU, memory, swap, I/O summary over intervals |
| `iostat -x 1` | Per-disk I/O: throughput, `%util`, `await` (latency) |
| `iotop` | Per-process disk I/O (like `top` for disk) |
| `sar` | Historical performance data (needs `sysstat` collecting) — CPU, memory, disk, network over time |
| `free -h` | Memory summary: used/free/buff-cache/available/swap |
| `mpstat -P ALL` | Per-core CPU utilization breakdown |
| `pidstat` | Per-process CPU/memory/disk/thread stats over time |

---

## Module 6: Networking

### Key commands
| Command | Purpose |
|---|---|
| `ss -tulnp` | Modern replacement for `netstat` — list listening/established sockets, ports, owning process |
| `netstat -tulnp` | Legacy equivalent (deprecated on many distros, but still asked about) |
| `ip addr`/`ip route`/`ip link` | Modern replacement for `ifconfig`/`route` — show/manage interfaces, IPs, routes |
| `ping` | ICMP reachability + round-trip latency |
| `dig`/`host` | DNS lookups (`dig` gives more detail: query time, authority, TTL) |
| `tcpdump -i eth0 port 443` | Packet capture/filtering for deep network debugging |
| `traceroute`/`mtr` | Trace the network path/hops to a destination; `mtr` combines ping + traceroute continuously |
| `curl -v` | Test HTTP(S) endpoints, see headers/handshake/status codes |
| `nc` (netcat) | Swiss-army knife — test if a port is open/reachable (`nc -zv host port`), simple data transfer |
| `telnet host port` | Older way to test raw TCP port connectivity (no encryption, just useful for a basic connect test) |

### TCP vs UDP
- **TCP**: connection-oriented, reliable (retransmits, ordering, ACKs), higher overhead — used for HTTP, databases, SSH.
- **UDP**: connectionless, no delivery guarantee, lower overhead — used for DNS queries, streaming, DHCP, health-check style traffic where speed > reliability.

### DNS
Resolves hostnames to IPs. Resolution order on a Linux host: check `/etc/hosts` first (unless `nsswitch.conf` says otherwise), then query nameservers listed in `/etc/resolv.conf`. `/etc/nsswitch.conf` controls the overall order of lookup sources (files, dns, etc).

### Routing
Determines which interface/gateway a packet should be sent through based on destination IP, governed by the routing table (`ip route show`). Default route (`0.0.0.0/0`) is where non-local traffic goes if no more specific route matches.

### NAT (Network Address Translation)
Translates private IPs to a public IP (or vice versa) so multiple internal hosts can share a single external IP — foundational for how VPCs/private subnets reach the internet (e.g. via a NAT Gateway in AWS).

### Firewall / iptables / nftables
- **iptables**: traditional Linux firewall, rule-based packet filtering organized into chains (`INPUT`, `OUTPUT`, `FORWARD`) and tables (`filter`, `nat`, `mangle`).
- **nftables**: modern replacement for iptables — unified syntax, better performance, is gradually replacing iptables as the default on newer distros, though iptables commands are often still available as a compatibility shim.
- Both let you allow/deny/redirect traffic based on IP, port, protocol, state.

### TCP connection states worth knowing cold
- `LISTEN` — socket waiting for incoming connections.
- `ESTABLISHED` — active connection.
- `TIME_WAIT` — connection closed, waiting to ensure delayed packets are handled — normal in moderate amounts, but a *huge* pile-up can exhaust ephemeral ports under high connection churn.
- `CLOSE_WAIT` — remote side closed but **local application hasn't closed its end** yet — a big pile-up of these usually indicates an application bug (not calling `close()` on sockets properly), a classic real-world troubleshooting question.

---

## Module 7: Logs

### Key commands
| Command | Purpose |
|---|---|
| `journalctl -u <service>` | Logs for a specific systemd unit |
| `journalctl -f` | Follow logs live (like `tail -f` for the journal) |
| `journalctl --since "1 hour ago"` | Time-bounded log query |
| `dmesg` | Kernel ring buffer — hardware, driver, OOM-kill, filesystem errors |
| `tail -f /var/log/app.log` | Follow a plain log file live |
| `grep -i error file.log` | Filter/search logs |
| `less file.log` | Paginated viewing of large log files without loading it all into memory |
| `cat file.log` | Dump full file content (fine for small files) |

**Interview tip**: know that `dmesg` is where you find OOM kills, disk I/O errors, and hardware/driver issues — many candidates only think to check application logs and miss kernel-level evidence.

---

## Module 7b: Shell Scripting / Text Processing

SRE interviews frequently include a live or take-home task here — usually "parse this log file and give me X."

### grep
Pattern search. `grep -i error app.log` (case-insensitive), `grep -c` (count matches), `grep -v` (invert match — exclude lines), `grep -E` (extended regex), `grep -A3 -B3` (show 3 lines after/before a match for context).

### awk
Field-based text processing — best for column-oriented data.
- Example: **count 5xx errors per minute from a log** — if your log has a timestamp in field 1 and status code in field 9:
```bash
awk '$9 ~ /^5[0-9]{2}$/ {print substr($1,1,16)}' access.log | sort | uniq -c
```
- `awk -F','` to set a custom field delimiter (e.g., CSV).
- `{print $1, $3}` prints specific columns; `NR` = current line number; `NF` = number of fields in the line.

### sed
Stream editor — best for find/replace and line-based transformations.
- `sed 's/foo/bar/g' file` — replace all occurrences of `foo` with `bar`.
- `sed -n '10,20p' file` — print only lines 10–20.
- `sed -i` — edit the file in place (use with caution, always test without `-i` first).

### Exit codes
Every command returns an exit code (`$?`) — `0` = success, non-zero = failure (the specific number is command-defined, e.g. `1` = general error, `127` = command not found, `130` = terminated by Ctrl+C/SIGINT). Critical for scripting: always check exit codes on commands that can fail silently (migrations, deploy steps, health checks) rather than assuming success just because the script kept running.

### `set -euo pipefail`
The standard defensive header for production bash scripts:
- `set -e` — exit immediately if any command fails (non-zero exit), instead of silently continuing.
- `set -u` — treat use of an **undefined variable** as an error and exit, instead of silently substituting an empty string (this is exactly the class of bug that causes things like unexpanded/blank variables silently corrupting a command).
- `set -o pipefail` — in a pipeline (`cmd1 | cmd2`), the pipeline's exit code reflects the **first** failing command, not just the last one. Without this, a failure in `cmd1` can be silently masked if `cmd2` succeeds.

**Interview line, and you can speak to this from real experience**: "I hit a real-world case where a `$` character inside a `DATABASE_URL` env var got mangled by unintended shell expansion — that's exactly the kind of bug `set -u` and careful quoting (`"$VAR"` instead of `$VAR`) are meant to catch early instead of failing silently deep in a pipeline. The fix in that case was to percent-encode the `$` (`%24`) so it wasn't a live shell metacharacter at all — but the broader lesson was to always treat env vars as untrusted/unescaped input in bash and quote defensively."

---

## Module 8: Troubleshooting (Scenario Walkthroughs)

### "Server is slow"
1. `uptime` → check load average.
2. `top`/`htop` → is it CPU-bound, memory-bound, or lots of D-state (disk-bound) processes?
3. `free -h` / `vmstat` → memory pressure, swapping?
4. `iostat -x 1` → disk `%util`/`await` high?
5. `ss -tulnp` / `sar -n DEV` → network saturation, retransmits?
6. Application/service logs (`journalctl -u`, app logs) for the actual error surfacing at the top of that stack.

### "High CPU"
`top` sorted by CPU → identify PID → `top -H -p <pid>` to see which thread → if it's your own app, `strace -p <pid>` or a profiler to see what syscalls/functions it's spinning on. Check if it's a legitimate load spike vs. a runaway loop/bug (e.g., retry storm, infinite loop after a bad deploy).

### "High Memory"
`free -h` for overall pressure → `ps -eo pid,cmd,%mem --sort=-%mem` to find the top consumer → check `dmesg` for OOM kill history → if a specific process's RSS keeps climbing over time with no plateau, that's a memory leak, not just normal usage.

### "Disk Full"
`df -h` → `du -sh /* | sort -rh` to locate the offender → check for deleted-but-open files via `lsof +L1` if `du` doesn't match `df` → also check `df -i` for inode exhaustion.

### "Application Down"
`systemctl status <service>` → `journalctl -u <service> -n 100` for the exit reason/error → check if the process crashed (OOM? unhandled exception?) vs never started (bad config, port conflict, missing dependency) → check port with `ss -tulnp | grep <port>` to see if it's even listening.

### "Port not listening"
`ss -tulnp | grep <port>` — if nothing shows, the app didn't bind (check its logs/config, or if another process already grabbed the port — "address already in use" in logs). If it *is* listening but unreachable externally, it's likely a firewall/security-group/iptables rule, or the app bound to `127.0.0.1` instead of `0.0.0.0`.

### "DNS failure"
`dig <domain>` / `nslookup <domain>` against the configured resolver → check `/etc/resolv.conf` for the right nameservers → try a known-good public resolver (`dig @8.8.8.8 domain`) to isolate whether it's the resolver itself, the network path, or the domain's authoritative records.

### "SSL issue"
`curl -v https://host` to see the handshake and where it fails (cert expired? hostname mismatch? chain incomplete?). `openssl s_client -connect host:443 -servername host` gives deep detail on the certificate chain and expiry. Common root causes: expired cert, missing intermediate cert in the chain, clock skew on the server (breaks cert validation), or client not sending SNI.

### "NFS mount issue"
Check `mount | grep nfs` and `showmount -e <server>` (list exports from the server side). Common failures: server-side export permissions, firewall blocking NFS ports, stale file handle (server restarted/export changed while client still had it mounted — usually needs a remount), or a hung mount blocking the shell (NFS mounts without the `soft`/`intr` option can hang indefinitely on network issues).

---

## Module 9: Security

### SSH
Encrypted remote access. Key-based auth preferred over passwords (`~/.ssh/authorized_keys`). Config lives in `/etc/ssh/sshd_config` — hardening basics: disable root login (`PermitRootLogin no`), disable password auth (`PasswordAuthentication no`), change default port if desired (minor obscurity benefit only, not real security).

### sudo
Lets a permitted user run commands as another user (typically root) without sharing the root password directly. Configured in `/etc/sudoers` (edit only via `visudo` to catch syntax errors before they lock you out). Supports fine-grained rules — specific users/groups allowed specific commands only.

### PAM (Pluggable Authentication Modules)
A modular framework that abstracts authentication logic (password checks, account lockout policies, MFA, etc.) so applications (`sshd`, `sudo`, `login`) don't need to implement auth logic themselves — configured via files in `/etc/pam.d/`.

### SELinux
Mandatory Access Control (MAC) system (RHEL/CentOS/Fedora) — enforces policy-based restrictions *beyond* standard Unix permissions, even for root. Processes and files get security **contexts/labels**; policy dictates what a labeled process can do to a labeled resource, regardless of file permission bits. Modes: `enforcing`, `permissive` (logs violations but doesn't block — useful for debugging), `disabled`. `getenforce`/`setenforce`, check denials via `ausearch`/`audit.log`.

### AppArmor
Similar goal to SELinux (Mandatory Access Control) but path-based rather than label-based, and generally considered simpler to configure — default on Ubuntu/Debian. Profiles per-application define what files/capabilities that app can access.

### File permissions / SUID / SGID / Sticky bit
(Covered in Module 4 above — same concepts, security-relevant here because misconfigured SUID binaries are a classic privilege-escalation vector interviewers probe: "why is an SUID root binary dangerous if it has a shell-escape or unsafe environment variable handling?")

---

## Quick Prep Tips
- For every topic, be ready to **connect it to a real incident** you've handled — you already have strong, authentic material from your CI/CD and infra work (OOM kills on ECS, `$` shell expansion corruption, CloudTrail exposure via env blocks, D-state disk waits during Prisma migrations). Interviewers weight "have you actually debugged this" far higher than textbook definitions.
- Practice narrating the **troubleshooting flow** (Module 8) out loud without looking anything up — that's the part that actually gets tested live, not isolated definitions.
- Know the difference between "legacy" and "modern" tool pairs (`netstat`→`ss`, `ifconfig`→`ip`) — interviewers sometimes ask this directly to gauge how current your knowledge is.
