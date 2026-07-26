# Kubernetes Interview Questions & Answers for SRE

A practical, no-fluff answer bank for SRE / DevOps interviews. Answers are written the way you'd actually **say them out loud** — short enough to not ramble, deep enough to show you've run this in production.

**How to use this:**
- Read the short answer first. That's your 20–30 second spoken answer.
- The `→ Going deeper` lines are what an interviewer follows up with. Have them ready.
- Commands are included where you'd realistically be asked "and how would you check that?"

**Interview tip that matters more than memorisation:** for every troubleshooting question, answer as a *sequence* — observe, narrow, confirm, fix, prevent. Interviewers are grading your method, not your trivia.

---

## Table of Contents

1. [Kubernetes Basics](#kubernetes-basics)
2. [Pods](#pods)
3. [Deployments & ReplicaSets](#deployments--replicasets)
4. [Services & Networking](#services--networking)
5. [Storage](#storage)
6. [Scheduling](#scheduling)
7. [Configuration](#configuration)
8. [Health Checks](#health-checks)
9. [Autoscaling](#autoscaling)
10. [Security](#security)
11. [Monitoring & Logging](#monitoring--logging)
12. [Troubleshooting](#troubleshooting)
13. [Production SRE Scenarios](#production-sre-scenarios)
14. [Amazon EKS](#amazon-eks)

---

## Kubernetes Basics

### What is Kubernetes?

Kubernetes is a container orchestrator. You describe the *desired state* of your application in YAML — "I want 3 replicas of this image, reachable on port 8080, with this much memory" — and Kubernetes continuously works to make reality match that description.

The key idea is the **reconciliation loop**: controllers constantly compare actual state to desired state and act on the difference. That's why a Pod comes back after you delete it, and why Kubernetes is described as declarative rather than imperative.

### Why do we need Kubernetes?

Because running containers on a single Docker host stops working the moment you care about uptime. Without an orchestrator you're manually handling: what happens when a container dies, what happens when a whole server dies, how traffic finds healthy instances, how you roll out a new version without dropping requests, and how you scale on load.

Kubernetes gives you all of that as built-in behaviour instead of custom scripts. In SRE terms: it removes a large category of toil and makes recovery automatic rather than pager-driven.

### What problems does Kubernetes solve?

- **Self-healing** — restarts crashed containers, replaces Pods on dead nodes.
- **Scheduling / bin-packing** — decides which node has room, so you're not placing workloads by hand.
- **Service discovery & load balancing** — stable DNS names and virtual IPs in front of ephemeral Pods.
- **Progressive rollouts & rollbacks** — replace versions incrementally, revert on failure.
- **Config and secret decoupling** — same image across environments, different config injected.
- **Autoscaling** — Pods and nodes scale with demand.
- **A common API** — one control surface across cloud, on-prem, and local.

### Explain the Kubernetes architecture.

Two planes:

**Control plane (the brain)** — API Server, etcd, Scheduler, Controller Manager, and the cloud controller manager. It decides *what should happen* and stores the truth.

**Data plane / worker nodes (the muscle)** — kubelet, kube-proxy, and a container runtime on each node. It makes things *actually happen* and reports back.

The flow of a typical request: `kubectl apply` → API Server authenticates, validates, writes to etcd → Scheduler notices a Pod with no node and assigns one → the kubelet on that node sees the assignment, pulls the image, starts containers via the runtime → kubelet reports status back to the API Server.

Nothing talks to etcd except the API Server. Everything else watches the API Server. That single detail explains most of Kubernetes' failure behaviour.

### What are the components of the control plane?

| Component | Job |
|---|---|
| **kube-apiserver** | The only front door. Auth, validation, admission, and the sole writer to etcd. |
| **etcd** | Distributed key-value store holding all cluster state. |
| **kube-scheduler** | Assigns unscheduled Pods to nodes. |
| **kube-controller-manager** | Runs the reconciliation loops (deployment, replicaset, node, endpoint, job…). |
| **cloud-controller-manager** | Talks to the cloud provider — load balancers, volumes, node lifecycle. |

### What are the components of a worker node?

- **kubelet** — the node agent. Takes Pod specs from the API Server, tells the runtime to run them, reports status and health probe results.
- **kube-proxy** — programs iptables/IPVS rules so Service virtual IPs route to backing Pods. (Some CNIs like Cilium replace it with eBPF.)
- **Container runtime** — containerd or CRI-O; actually pulls images and runs containers.

Plus, in practice: a CNI plugin for Pod networking and often a log/metrics agent as a DaemonSet.

### What is etcd?

A consistent, distributed key-value store — the single source of truth for the cluster. Every object you create lives there.

Three things to know for an interview:
1. It uses **Raft consensus**, so you run an odd number of members (3 or 5) and it tolerates `(n-1)/2` failures.
2. It's **latency-sensitive** — put it on fast disks. Slow etcd disk I/O is a classic cause of a sluggish or flapping cluster.
3. **Back it up.** Losing etcd without a snapshot means losing the cluster's state. On EKS, AWS manages and backs up etcd for you — which is a fair chunk of why people choose EKS.

### What is the role of the API Server?

It's the hub. It exposes the REST API, handles authentication and authorisation (RBAC), runs admission controllers and webhooks, validates objects, and persists them to etcd. It also serves the **watch** streams that every other component depends on.

Practically: if the API Server is down, running Pods keep serving traffic, but nothing can change — no deploys, no scaling, no self-healing of failed Pods.

### What is the role of the Scheduler?

It watches for Pods with no `nodeName` and picks a node for each. Two phases:

1. **Filtering** — eliminate nodes that can't work: not enough CPU/memory, taints not tolerated, node selectors unmatched, required volume unavailable, no free ports.
2. **Scoring** — rank the survivors (spread across nodes, image already cached locally, affinity preferences) and bind the Pod to the winner.

The Scheduler only *decides*. The kubelet is what actually starts the Pod.

### What is the role of the Controller Manager?

A single binary running many independent control loops. Each one watches a resource type and drives actual state toward desired state:

- **Deployment controller** → manages ReplicaSets
- **ReplicaSet controller** → keeps N Pods alive
- **Node controller** → marks nodes NotReady, evicts Pods after the toleration timeout
- **Endpoints / EndpointSlice controller** → keeps Service backends in sync with healthy Pods
- **Job, CronJob, PV, ServiceAccount controllers** → and more

This is where "self-healing" physically happens.

### What is kubelet?

The agent on every node, and the only component that manages containers directly. It:

- Registers the node with the API Server
- Watches for Pods assigned to its node
- Instructs the container runtime (via CRI) to pull images and start/stop containers
- Runs liveness/readiness/startup probes and acts on the results
- Mounts volumes, injects ConfigMaps/Secrets
- Reports node and Pod status, and triggers eviction under resource pressure

Note: kubelet manages *Pods*, not Deployments. It has no idea what a Deployment is.

### What is kube-proxy?

It implements the Service abstraction on each node. It watches Services and EndpointSlices and writes iptables (or IPVS) rules so that traffic sent to a Service's ClusterIP gets DNAT'd to one of the healthy Pod IPs.

Worth saying in an interview: kube-proxy is **not a proxy sitting in the data path** in iptables mode — it's a rules-writer. The kernel does the forwarding. That's why it's fast and why Service load balancing is roughly random per-connection, not smart or latency-aware.

### What is a Container Runtime?

The software that actually runs containers — pulls images, sets up namespaces and cgroups, starts processes. Kubernetes talks to it through the **CRI (Container Runtime Interface)**.

Common ones: **containerd** (the default nearly everywhere, including EKS AMIs), **CRI-O**, and low-level runtimes like `runc` underneath. Dockershim was removed in Kubernetes 1.24, so "Kubernetes dropped Docker" means it dropped the *shim* — images built with Docker still run fine, because they're OCI images.

---

## Pods

### What is a Pod?

The smallest deployable unit — one or more containers that share a network namespace (same IP and port space, can reach each other on `localhost`), share storage volumes, and are always scheduled together onto one node.

Pods are **ephemeral and disposable by design**. You don't repair a Pod; you replace it. That's why you almost never create bare Pods in production — you let a Deployment or StatefulSet own them.

### Can a Pod contain multiple containers?

Yes, and there are three legitimate patterns:

- **Sidecar** — a helper alongside the main app: log shipper, metrics exporter, service mesh proxy, OTel collector.
- **Adapter** — reshapes the app's output into a standard format.
- **Ambassador** — proxies outbound connections, e.g. a local DB connection pooler.

Plus **init containers**, which run to completion *before* the app containers start — used for migrations, waiting on dependencies, or fetching config.

The rule of thumb: containers belong in the same Pod only if they must scale together and share a lifecycle. Two things you'd want to scale independently belong in separate Pods.

### What are Pod lifecycle phases?

| Phase | Meaning |
|---|---|
| **Pending** | Accepted, but not running yet — waiting to be scheduled or pulling images. |
| **Running** | Bound to a node, at least one container running. |
| **Succeeded** | All containers exited 0 and won't restart. |
| **Failed** | All containers terminated, at least one with a non-zero exit. |
| **Unknown** | Node's status can't be reached (usually a node or network problem). |

Careful in interviews: `CrashLoopBackOff` and `ImagePullBackOff` are **container states**, not Pod phases. The Pod is still `Pending` or `Running`.

### What is CrashLoopBackOff?

The container starts, exits, and kubelet restarts it — repeatedly. Kubernetes adds exponential backoff between attempts (10s, 20s, 40s… capped at 5 minutes) so it stops hammering the node. The name describes the *backoff*, not the root cause.

Common causes: application crashes at startup, missing env var or config, bad DB credentials, a failing dependency, an OOMKill, wrong command/entrypoint, or a liveness probe that's too aggressive.

First move — read the logs of the previous, dead container:

```bash
kubectl logs <pod> --previous
kubectl describe pod <pod>   # check Last State, Exit Code, Reason
```

Exit code 137 = OOMKilled or SIGKILL. Exit code 1 = app error. Check both.

### What is ImagePullBackOff?

kubelet can't pull the image, so it backs off and retries. `ErrImagePull` is the immediate failure; `ImagePullBackOff` is the retry state.

Causes, in the order I'd check them:

1. Typo in the image name or tag — most common by far.
2. Private registry with no or wrong `imagePullSecrets`.
3. In AWS: the node's IAM role lacks ECR pull permissions (`ecr:GetAuthorizationToken`, `BatchGetImage`, `GetDownloadUrlForLayer`).
4. No network route to the registry — private subnet with no NAT gateway and no ECR VPC endpoint.
5. Registry rate limiting (classic Docker Hub anonymous pull limit).

`kubectl describe pod` puts the actual registry error in the Events — read it rather than guessing.

### Why does a Pod remain in Pending state?

Pending means it hasn't been successfully scheduled or hasn't started. Run `kubectl describe pod` and read the Events — the Scheduler tells you exactly why. Typical reasons:

- **Insufficient CPU/memory** — no node can satisfy the requests. Note it's *requests*, not usage.
- **Taints not tolerated** by the Pod.
- **nodeSelector / affinity** matches no node.
- **PVC unbound** — no matching PV, or the StorageClass can't provision.
- **Pod anti-affinity or topology spread** can't be satisfied with the current node count.
- **Cluster Autoscaler can't add nodes** — at max size, or the requested shape doesn't fit any node group.
- **Node ports already in use** on every candidate node.

### How do you debug a failed Pod?

My standard sequence:

```bash
kubectl get pod <pod> -o wide           # phase, restarts, node, IP
kubectl describe pod <pod>              # Events + container states + exit codes
kubectl logs <pod> -c <container>       # current logs
kubectl logs <pod> --previous           # logs from the crashed instance
kubectl get events --sort-by=.lastTimestamp -n <ns>
kubectl exec -it <pod> -- sh            # if it stays up long enough
```

Then reason about *where* it broke: never scheduled → Scheduler/capacity problem. Scheduled but never started → image or volume or config problem. Started then died → application or resource limit problem. That triage split narrows it fast.

If the container has no shell (distroless), use an ephemeral debug container:

```bash
kubectl debug -it <pod> --image=busybox --target=<container>
```

### How do you view Pod logs?

```bash
kubectl logs <pod>                          # single-container Pod
kubectl logs <pod> -c <container>           # multi-container
kubectl logs -f <pod>                       # follow
kubectl logs <pod> --previous               # crashed instance
kubectl logs <pod> --since=15m --tail=200
kubectl logs -l app=api --all-containers    # all Pods matching a label
```

Important caveat to mention: `kubectl logs` reads files on the node, so logs die with the node and are rotated away. Production needs shipping to a real backend — CloudWatch Logs, Loki, or an ELK/OpenSearch stack — via a DaemonSet agent like Fluent Bit.

### How do you exec into a running Pod?

```bash
kubectl exec -it <pod> -- /bin/sh
kubectl exec -it <pod> -c <container> -- /bin/bash
kubectl exec <pod> -- env                  # one-off command, no TTY
```

Useful follow-up to volunteer: on hardened or distroless images there's no shell, so you use `kubectl debug` to attach an ephemeral container that shares the Pod's namespaces. And in a tightly controlled production environment, `exec` is often restricted by RBAC and audited — the expectation is that you debug from telemetry, not by shelling in.

### What happens when a Pod is deleted?

1. The Pod is marked for deletion with a `deletionTimestamp` and enters `Terminating`.
2. It's **removed from Service endpoints** immediately, so it stops receiving new traffic.
3. The `preStop` hook runs, if defined.
4. `SIGTERM` goes to PID 1 in each container.
5. Kubernetes waits up to `terminationGracePeriodSeconds` (default 30).
6. Anything still alive gets `SIGKILL`.
7. Volumes are unmounted and the Pod object is removed from etcd.

Two production points worth raising: endpoint removal and SIGTERM are **concurrent, not ordered**, so a `preStop: sleep 5` is a common trick to let in-flight and just-routed requests finish. And your app must actually handle SIGTERM by draining — if it ignores it, every deploy drops connections.

---

## Deployments & ReplicaSets

### What is a Deployment?

A controller for stateless workloads that manages ReplicaSets for you. You declare the Pod template and replica count; the Deployment handles versioned rollouts, rollbacks, and scaling.

The chain is: **Deployment → ReplicaSet → Pods**. Changing the Pod template creates a *new* ReplicaSet and scales the old one down — which is exactly what makes rollback possible, because the old ReplicaSet is kept (up to `revisionHistoryLimit`, default 10).

### What is a ReplicaSet?

The controller that guarantees a specified number of identical Pods are running. It watches Pods matching its label selector; too few, it creates more, too many, it deletes some.

You rarely create one directly — Deployments own them. Its predecessor was ReplicationController; ReplicaSet adds set-based selectors (`in`, `notin`, `exists`).

### Deployment vs ReplicaSet?

ReplicaSet answers "are N Pods alive?" Deployment answers "how do I safely move from version A to version B?"

ReplicaSet has no concept of versions, rollouts, or history. If you edit a ReplicaSet's Pod template, existing Pods aren't touched — only newly created ones use it. Deployment adds the rollout strategy, revision history, pause/resume, and `kubectl rollout undo`. In production you use Deployments; ReplicaSets are an implementation detail.

### How do rolling updates work?

With `strategy: RollingUpdate` (the default), the Deployment creates a new ReplicaSet and shifts replicas over gradually:

1. Scale new RS up by `maxSurge`.
2. Wait for the new Pods to become **Ready** (this is why readiness probes matter enormously).
3. Scale the old RS down.
4. Repeat until the new RS is at full count and the old is at zero.

Availability during the rollout is bounded by `maxUnavailable`. The alternative strategy is `Recreate`, which kills everything first — acceptable for a batch job or a single-writer app that can't run two versions, not for a web service.

Useful commands:

```bash
kubectl rollout status deployment/api
kubectl rollout history deployment/api
kubectl rollout pause deployment/api     # canary-ish: pause partway
```

### What is a rolling rollback?

Rolling back is just a rolling update in reverse — Kubernetes scales the *previous* ReplicaSet back up and the current one down, using the same `maxSurge`/`maxUnavailable` rules. No downtime, same mechanics.

```bash
kubectl rollout undo deployment/api
kubectl rollout undo deployment/api --to-revision=4
```

Two gotchas to mention: rollback restores the **Pod template only** — it does *not* revert a ConfigMap, a Secret, or a database migration. If your bad release included a destructive migration, `rollout undo` won't save you. That's why backward-compatible (expand/contract) migrations are the practice.

### What are maxSurge and maxUnavailable?

Both are counts or percentages controlling rollout pace:

- **maxSurge** (default 25%) — how many Pods above `replicas` may exist temporarily. Higher = faster rollout, more peak resource usage.
- **maxUnavailable** (default 25%) — how many Pods may be unavailable at once. `0` = never dip below full capacity.

For a zero-downtime service: `maxUnavailable: 0, maxSurge: 1` (or 25%) — always add before removing. You need spare cluster capacity for that. Setting both to 0 is invalid; nothing could ever progress.

### How do you scale a Deployment?

```bash
kubectl scale deployment/api --replicas=6
kubectl autoscale deployment/api --min=3 --max=20 --cpu-percent=70
```

Or declaratively by editing `replicas` in Git and letting your CD tool apply it — which is the right answer for a production question.

Important caveat: if an HPA manages the Deployment, don't also manage `replicas` in Git — they'll fight each other and flap. Either remove `replicas` from the manifest or have your GitOps tool ignore that field.

### What happens if a Deployment fails during rollout?

The rollout **stalls rather than breaking the running service**. Because new Pods never become Ready, old Pods are never scaled down — so users keep hitting the healthy old version. This is readiness probes earning their keep.

After `progressDeadlineSeconds` (default 600) the Deployment gets a condition `Progressing=False, reason=ProgressDeadlineExceeded`. Kubernetes does **not** auto-rollback — you have to act:

```bash
kubectl rollout status deployment/api      # will report the failure
kubectl describe pod <new-pod>             # find the real cause
kubectl rollout undo deployment/api
```

In a CI/CD context, the right pattern is: `kubectl rollout status --timeout=5m` in the pipeline, and on non-zero exit, run `rollout undo` and fail the job. Otherwise you get a green pipeline over a broken deploy.

---

## Services & Networking

### What is a Kubernetes Service?

A stable network identity in front of a set of Pods. It gets a virtual IP (ClusterIP) and a DNS name that never change, and it load-balances to whichever Pods currently match its label selector and are Ready.

The Service selects Pods by label; the EndpointSlice controller maintains the actual list of backend IPs. If a Pod isn't Ready, it isn't in that list.

### Why do we need Services?

Because Pod IPs are ephemeral. Every restart, rescale, or reschedule gives you new IPs — you cannot hardcode them, and you can't wait for a DNS TTL to catch up. A Service decouples "who I want to talk to" from "where it currently lives."

It also gives you built-in load balancing and health-aware routing for free: only Ready Pods receive traffic.

### What are the different Service types?

- **ClusterIP** (default) — internal-only virtual IP. For service-to-service traffic.
- **NodePort** — opens the same high port (30000–32767) on every node, forwarding to the Service. Mostly for testing or as a building block under a load balancer.
- **LoadBalancer** — asks the cloud provider for an external LB (an ELB/NLB on AWS) pointing at the Service.
- **ExternalName** — no proxying at all, just a CNAME to an external DNS name. Handy for pointing at RDS or a third-party API by a cluster-internal name.
- **Headless** (`clusterIP: None`) — no virtual IP; DNS returns the individual Pod IPs. Used by StatefulSets and by clients that do their own load balancing (e.g. gRPC, Kafka).

### ClusterIP vs NodePort vs LoadBalancer?

They're layered, not alternatives — NodePort includes ClusterIP behaviour, and LoadBalancer includes both.

| | Reachable from | Typical use |
|---|---|---|
| ClusterIP | Inside cluster only | Internal microservice calls |
| NodePort | Any node IP + high port | Dev, on-prem, or behind an external LB |
| LoadBalancer | Public/internal cloud LB | Exposing a service directly |

The production caveat: one `LoadBalancer` Service = one cloud load balancer = one bill. Ten services means ten LBs. That's precisely why you put an Ingress in front instead.

### What is an Ingress?

An HTTP/HTTPS routing rule set — one entry point that fans out to many Services based on hostname and path, with TLS termination.

```
api.example.com/users  → users-svc
api.example.com/orders → orders-svc
```

An Ingress object is just configuration. It does nothing without an **Ingress Controller** running in the cluster to read it and implement the routing.

### Ingress vs LoadBalancer?

A LoadBalancer Service is **L4** — it forwards TCP/UDP to one Service, and knows nothing about hostnames or paths. An Ingress is **L7** — one load balancer, many services, routed by host/path, with TLS, redirects, and rewrites.

Cost and operations are the real differentiator: 20 LoadBalancer Services means 20 cloud LBs to pay for and monitor; 20 Ingress rules can share one. Ingress is HTTP-only though — for raw TCP (a database, a game server), you still use a LoadBalancer Service. And for anything more advanced, the Gateway API is the modern successor to Ingress.

### What is an Ingress Controller?

The component that watches Ingress objects and actually configures a proxy or cloud load balancer to match. Common ones:

- **ingress-nginx** — runs NGINX in-cluster, usually behind one LoadBalancer Service.
- **AWS Load Balancer Controller** — provisions a real ALB per Ingress (or shares one via `alb.ingress.kubernetes.io/group.name`).
- **Traefik, HAProxy, Istio Gateway, Contour**.

No controller installed → your Ingress sits there with no address and nothing works. That's a frequent "Ingress not routing" cause.

### What is CoreDNS?

The cluster's DNS server, running as a Deployment in `kube-system`. Every Pod's `/etc/resolv.conf` points at the `kube-dns` Service IP, and CoreDNS resolves Service names to ClusterIPs (and Pod names for headless Services). Non-cluster names are forwarded upstream.

It's a shared dependency, so it's a blast-radius component: when CoreDNS is unhealthy or throttled, *everything* looks broken in confusing ways. Worth monitoring its latency and error rate explicitly, and worth knowing about NodeLocal DNSCache for high-QPS clusters.

### How does service discovery work?

Primarily via DNS. Every Service gets a record in the form:

```
<service>.<namespace>.svc.cluster.local
```

Within the same namespace, `http://payments:8080` resolves, because the Pod's resolv.conf has a `search` path of `<ns>.svc.cluster.local svc.cluster.local cluster.local`. Cross-namespace, you use `payments.prod`.

Resolution chain: app → CoreDNS returns the ClusterIP → the Pod sends traffic to that VIP → kube-proxy's iptables/IPVS rules DNAT it to a Ready Pod IP. Kubernetes also injects Service env vars into Pods, but that's legacy and order-dependent — DNS is the answer to give.

### What is CNI?

**Container Network Interface** — the plugin standard Kubernetes uses to give Pods network connectivity. Kubernetes itself doesn't implement Pod networking; the CNI plugin does.

The model it must satisfy: every Pod gets its own IP, all Pods can reach all Pods without NAT, and nodes can reach Pods. Plugins differ in *how*: **AWS VPC CNI** assigns real VPC IPs directly to Pods (great for AWS-native networking and security groups, but subnet IP exhaustion is a real planning concern); **Calico** and **Cilium** use overlays or eBPF and add richer network policy. Cilium can also replace kube-proxy entirely.

### What are Network Policies?

Firewall rules for Pods, expressed as Kubernetes objects. They select Pods by label and define allowed ingress/egress by Pod selector, namespace selector, or CIDR.

Two things that trip people up:

1. Pod networking is **allow-all by default**. Policies only take effect once one selects a Pod — and then that Pod becomes default-deny for the directions specified.
2. **Your CNI must support them.** The AWS VPC CNI needs Calico or the built-in EKS network policy support enabled; otherwise your NetworkPolicy objects are accepted and silently ignored, which is a nasty false sense of security.

The standard production baseline is a default-deny-all policy per namespace, then explicit allows.

### How do Pods communicate across nodes?

Flat routing, no NAT — a Pod on node A addresses a Pod on node B by its Pod IP directly. How that's implemented depends on the CNI:

- **Native routing** (AWS VPC CNI): Pod IPs are real VPC IPs, so the VPC route table handles it. No encapsulation, minimal overhead.
- **Overlay** (VXLAN/Geneve, e.g. Calico VXLAN, Flannel): packets are encapsulated between nodes and decapsulated on arrival.
- **BGP** (Calico native): nodes advertise Pod CIDRs to each other or to the physical fabric.

When cross-node Pod traffic breaks, the usual suspects are security groups / firewall rules between nodes, MTU mismatch with an overlay (works for small packets, hangs on large ones — very characteristic symptom), or a crashed CNI DaemonSet on one node.

---

## Storage

### What is a Persistent Volume (PV)?

A cluster-level object representing a real piece of storage — an EBS volume, an EFS filesystem, an NFS share. Its lifecycle is independent of any Pod, which is the whole point: the Pod can die and the data survives.

Key fields: `capacity`, `accessModes`, `storageClassName`, `persistentVolumeReclaimPolicy` (Retain / Delete), and the driver-specific source. In modern clusters you rarely create PVs by hand — a CSI driver creates them dynamically.

### What is a Persistent Volume Claim (PVC)?

A request for storage, made by a workload: "I need 20Gi, ReadWriteOnce, from the gp3 StorageClass." Kubernetes binds it to a suitable PV — either an existing one, or one provisioned on demand.

The separation matters: PVC is namespaced and belongs to the app team; PV is cluster-scoped and belongs to the platform. The Pod mounts the PVC and never needs to know whether it's EBS, EFS, or NFS underneath.

### What is a StorageClass?

A named storage "profile" that defines *how* to provision volumes: which CSI provisioner, what parameters (volume type, IOPS, throughput, encryption, KMS key), the `reclaimPolicy`, `allowVolumeExpansion`, and the `volumeBindingMode`.

```yaml
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

`volumeBindingMode: WaitForFirstConsumer` is the detail worth mentioning — it delays provisioning until a Pod is scheduled, so the EBS volume is created in the *same AZ* as the node. With `Immediate`, you can end up with a volume in AZ-a and the only available node in AZ-b, and the Pod never starts.

### What is dynamic provisioning?

Creating the underlying storage automatically when a PVC is created, instead of an admin pre-creating PVs. The PVC names a StorageClass, the CSI driver calls the cloud API, an EBS volume appears, a PV is created and bound — all without human involvement.

Very much an SRE win: it's the difference between a self-service platform and a ticket queue. The trade-off to be aware of is that `reclaimPolicy: Delete` means deleting a PVC destroys real data — so stateful production workloads often use `Retain` plus snapshots.

### How do StatefulSets use storage?

Via `volumeClaimTemplates`. Instead of all replicas sharing one PVC, the StatefulSet creates a **separate PVC per Pod**, named deterministically:

```
data-postgres-0, data-postgres-1, data-postgres-2
```

Because Pod identity is stable (`postgres-0` is always `postgres-0`), a restarted or rescheduled Pod **reattaches to its own volume**. That's what makes databases and other stateful systems viable. Two follow-ups worth knowing:

- With EBS (ReadWriteOnce, AZ-bound), `postgres-0` can only be rescheduled onto a node in the same AZ as its volume.
- Deleting the StatefulSet does **not** delete the PVCs, by design. Cleanup is deliberate and manual (or via `persistentVolumeClaimRetentionPolicy`).

---

## Scheduling

### How does the Scheduler decide where to place Pods?

Two phases, as above: **filter** then **score**.

Filtering removes nodes that are outright impossible — insufficient allocatable CPU/memory against the Pod's *requests*, untolerated taints, unmatched `nodeSelector`/`nodeAffinity`, unsatisfiable anti-affinity, a volume that can't attach in that AZ, a host port already taken.

Scoring ranks what's left: least-allocated vs most-allocated (bin-packing), image locality (image already on the node), affinity preferences, topology spread balance. Highest score wins, the Pod is bound, and kubelet takes over.

If nothing survives filtering, the Pod stays `Pending` with a `FailedScheduling` event naming the reason — and if configured, Cluster Autoscaler reads exactly that signal to add a node.

### What is nodeSelector?

The simplest node constraint: a hard requirement that a node carry specific labels.

```yaml
nodeSelector:
  node.kubernetes.io/instance-type: m5.large
```

All listed labels must match — it's AND-only, no operators, no preferences. If nothing matches, the Pod stays Pending forever. Fine for simple cases; use node affinity when you need anything richer.

### What is Node Affinity?

nodeSelector with expressions and soft options. Two flavours:

- **requiredDuringSchedulingIgnoredDuringExecution** — hard rule, same effect as nodeSelector but with operators (`In`, `NotIn`, `Exists`, `Gt`, `Lt`).
- **preferredDuringSchedulingIgnoredDuringExecution** — a weighted preference; if it can't be met, the Pod still schedules.

`IgnoredDuringExecution` is the part interviewers probe: the rule is only evaluated **at scheduling time**. Relabel the node afterwards and the running Pod is not evicted.

Real use: "prefer spot instances, fall back to on-demand" is a preferred affinity with weights.

### What is Pod Affinity?

Schedule a Pod *near* other Pods, defined by their labels and a `topologyKey`.

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels: { app: redis }
    topologyKey: kubernetes.io/hostname
```

"Put me on the same node as a Pod labelled `app=redis`." Used for latency-sensitive co-location — app next to its cache. `topologyKey` sets the granularity: `kubernetes.io/hostname` = same node, `topology.kubernetes.io/zone` = same AZ.

Worth noting it's expensive to evaluate at scale; required pod affinity in a large cluster slows scheduling noticeably.

### What is Pod Anti-Affinity?

The opposite, and far more common in production: keep replicas *apart* so a single failure doesn't take them all.

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels: { app: api }
    topologyKey: topology.kubernetes.io/zone
```

That spreads replicas across AZs. Use `required` and you get a hard guarantee — but if you have 3 zones and ask for 4 replicas, the 4th stays Pending. `preferred` is usually the safer production choice.

Modern alternative worth mentioning: **`topologySpreadConstraints`**, which expresses "spread evenly with a maximum skew of N" more precisely and cheaply than anti-affinity.

### What are taints?

A property on a **node** that repels Pods: "don't schedule here unless you explicitly accept this."

```bash
kubectl taint nodes node1 gpu=true:NoSchedule
```

Three effects:
- `NoSchedule` — no new Pods without a matching toleration.
- `PreferNoSchedule` — soft version.
- `NoExecute` — also **evicts** Pods already running that don't tolerate it.

Kubernetes uses taints internally: `node.kubernetes.io/not-ready`, `unreachable`, `disk-pressure`, `memory-pressure`. That's the mechanism behind Pods being evicted from a failing node.

### What are tolerations?

The Pod-side counterpart — permission to be scheduled onto a tainted node.

```yaml
tolerations:
- key: gpu
  operator: Equal
  value: "true"
  effect: NoSchedule
```

The important nuance: a toleration doesn't *attract* a Pod, it only removes the barrier. To dedicate nodes to a workload you need **taint + toleration** (keep others off) **plus node affinity or nodeSelector** (pull yours on). People frequently give only half the answer.

A toleration with no `key` and `operator: Exists` tolerates everything — that's how DaemonSets like log agents run on every node, including tainted ones.

### What is cordon?

Mark a node unschedulable without touching what's running on it.

```bash
kubectl cordon node1
```

Existing Pods stay and keep serving; no *new* Pods land there. It's the first step of maintenance, and also a good containment move when you suspect a node is misbehaving — stop the bleeding, then investigate at your own pace.

### What is drain?

Cordon **plus** evict everything so the node can be safely taken out.

```bash
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

Drain uses the **Eviction API**, which respects **PodDisruptionBudgets** — so it will block rather than violate your availability guarantee. That's a feature, not a bug: it's what stops a routine node rotation from taking your service down. The flags exist because DaemonSet Pods can't be evicted meaningfully, and emptyDir data loss must be acknowledged explicitly.

If drain hangs, check the PDB and whether replacement Pods can actually schedule elsewhere.

### What is uncordon?

Marks the node schedulable again after maintenance.

```bash
kubectl uncordon node1
```

Note that uncordon doesn't *move anything back* — existing Pods stay where they are, and the node only fills up as new Pods get created. If you need rebalancing, that's a separate concern (Descheduler, or just rolling the Deployment).

---

## Configuration

### What is a ConfigMap?

A key-value object for non-sensitive configuration — feature flags, log levels, hostnames, whole config files. Consumed as environment variables, command-line args, or mounted files.

The point is decoupling: one image, many environments. Same container runs in QA and prod with different ConfigMaps.

Practical facts worth knowing: it's limited to ~1MiB (it lives in etcd), it is **not encrypted or protected**, and updates propagate to *mounted volumes* automatically (with a delay) but **never** to env vars — env vars are set at container start, so you must restart Pods:

```bash
kubectl rollout restart deployment/api
```

### What is a Secret?

Same shape as a ConfigMap, intended for sensitive data — passwords, tokens, TLS keys, registry credentials. Values are **base64-encoded, which is encoding, not encryption**. Say that explicitly in an interview; it's a common trap.

Real protection comes from what you add around it: encryption at rest for etcd (KMS provider), RBAC restricting who can read Secrets in which namespaces, disabling automatic ServiceAccount token mounting where unneeded, and preferring an external store.

### ConfigMap vs Secret?

Functionally almost identical; the difference is intent and the handling that intent unlocks.

| | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive config | Credentials, keys, certs |
| Storage | Plaintext in etcd | Base64; encrypted at rest *if configured* |
| Volume mounts | On disk | Mounted as `tmpfs` (memory, not disk) |
| Logging/audit | Values often shown | Values redacted by kubectl/API in places |
| Typical RBAC | Loose | Tightly restricted |

The honest summary: a Secret is a ConfigMap with better defaults and a promise that you'll treat it carefully. The mechanism doesn't enforce security for you.

### How do you inject environment variables into a Pod?

Four ways, in ascending order of how much you'd like them:

```yaml
env:
- name: LOG_LEVEL          # 1. literal
  value: "debug"

- name: DB_HOST            # 2. single key from a ConfigMap
  valueFrom:
    configMapKeyRef: { name: app-config, key: db_host }

- name: DB_PASSWORD        # 3. single key from a Secret
  valueFrom:
    secretKeyRef: { name: db-creds, key: password }

envFrom:                   # 4. every key in one go
- configMapRef: { name: app-config }
```

Plus the **downward API** for Pod metadata, which is handy for logging and tracing:

```yaml
- name: POD_NAME
  valueFrom:
    fieldRef: { fieldPath: metadata.name }
```

Two things I'd volunteer: env-var changes require a Pod restart, and secrets in env vars are visible via `kubectl describe pod`, in crash dumps, and to anything reading `/proc/<pid>/environ` — mounted files or a CSI secrets driver is the safer pattern.

---

## Health Checks

### What is a Liveness Probe?

"Is this container still functioning, or is it wedged?" If it fails `failureThreshold` times in a row, **kubelet kills and restarts the container**.

It's for unrecoverable states a process can't detect itself — a deadlock, an event loop stuck, a thread pool exhausted. It should check *only* the process itself.

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3
```

The single biggest mistake: making the liveness endpoint check the database or a downstream API. Then a slow database restarts every Pod simultaneously — you've converted a dependency slowdown into a full self-inflicted outage. Keep dependencies out of liveness.

### What is a Readiness Probe?

"Should this Pod receive traffic right now?" On failure, the Pod is **removed from Service endpoints** but is **not restarted**.

This is the one that most affects user-visible behaviour: it gates rolling updates (new Pods must be Ready before old ones go away) and it lets a Pod shed traffic while it warms a cache, finishes a long start-up, or briefly loses a dependency — then rejoin automatically.

Readiness *can* legitimately check critical dependencies, since the consequence is traffic removal, not a restart. Do that cautiously though: if every replica marks itself not-ready at once, the Service has zero endpoints and you're fully down.

### What is a Startup Probe?

A probe that runs *before* liveness and readiness begin, for slow-starting applications — big JVMs, apps loading models, legacy services with a five-minute boot.

```yaml
startupProbe:
  httpGet: { path: /healthz, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10     # allows up to 300s to start
```

While it's failing, liveness is suspended, so a slow start can't cause a restart loop. Once it passes once, it never runs again and the other probes take over.

Before startup probes existed, people abused `initialDelaySeconds` on liveness — which forces you to choose between tolerating slow starts and detecting hangs quickly. A startup probe lets you have both: generous startup window, aggressive liveness afterwards.

### Liveness vs Readiness Probe?

| | Liveness | Readiness |
|---|---|---|
| Question | Is it broken? | Can it serve? |
| Action on failure | Restart container | Remove from Service endpoints |
| Recovers on its own? | No — needs a restart | Yes — rejoins when it passes |
| Should check dependencies? | **No** | Cautiously, yes |
| Affects rollouts? | No | **Yes** — gates the rollout |

The one-liner: liveness fixes a *stuck* Pod, readiness protects *users* from a Pod that isn't ready yet. Get readiness wrong and you serve 502s during every deploy; get liveness wrong and you cause restart storms.

---

## Autoscaling

### What is Horizontal Pod Autoscaler (HPA)?

Scales the **number of Pods** in a Deployment/StatefulSet based on observed metrics — CPU, memory, or custom/external metrics via the metrics APIs.

```bash
kubectl autoscale deployment api --min=3 --max=20 --cpu-percent=70
```

Requirements to state: **metrics-server** must be installed, and the Pods must have **resource requests set** — CPU utilisation is calculated as a percentage of *requests*, so with no requests, HPA has no denominator and does nothing.

### What is Vertical Pod Autoscaler (VPA)?

Adjusts the **requests and limits** of individual Pods rather than the replica count — it observes actual usage and recommends or applies right-sized values.

Three modes: `Off` (recommendations only), `Initial` (set at Pod creation), and `Auto` (evict and recreate with new values). The catch, and the reason many teams run it in `Off` mode: applying changes historically required **restarting the Pod**, since resources were immutable. In-place Pod resizing has been arriving in recent Kubernetes versions, which improves this.

Also: don't run VPA and HPA on the same metric for the same workload — they'll fight. VPA on memory + HPA on CPU is a workable combination.

### What is Cluster Autoscaler?

Scales the **number of nodes**. It watches for Pods stuck `Pending` due to insufficient resources and adds nodes to the appropriate node group; it removes nodes that have been underutilised for a period and whose Pods can be relocated.

Two crucial points:

1. It's driven by **Pod requests**, not actual usage. A cluster at 10% real CPU can still trigger scale-up if requests are inflated.
2. Scale-down is blocked by Pods it can't safely move — Pods with local storage, restrictive PDBs, bare Pods with no controller, or `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`.

On AWS, **Karpenter** is now the common alternative: it provisions right-sized instances directly rather than scaling fixed node groups, which is faster and usually cheaper.

The three autoscalers together: HPA = more Pods, VPA = bigger Pods, CA/Karpenter = more nodes.

### How does HPA work?

A control loop, every 15 seconds by default:

1. Fetch current metrics for the target's Pods from the metrics API.
2. Compute `desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)`.
3. Clamp to `minReplicas`/`maxReplicas`, apply the scaling policies, and update the target's `replicas`.

Example: 4 Pods at 90% CPU with a 60% target → `ceil(4 × 90/60)` = 6 Pods.

Stability features to mention: a **stabilisation window** (default 300s for scale-down, 0 for scale-up) prevents flapping, and `behavior` policies let you cap how fast it moves in each direction. Scale-up is deliberately eager, scale-down deliberately cautious.

Its limitation, which is a good thing to raise proactively: HPA is *reactive* and bounded below by the metrics pipeline — roughly a minute of delay before it even sees a spike. For sudden traffic bursts you need headroom, over-provisioning "pause" Pods, or predictive/scheduled scaling — autoscaling alone won't save you.

---

## Security

### What is RBAC?

**Role-Based Access Control** — the authorisation layer that decides which identity may perform which verb on which resource. Four objects:

- **Role** — permissions within one namespace
- **ClusterRole** — permissions cluster-wide (or for cluster-scoped resources)
- **RoleBinding** — grants a Role (or ClusterRole) to a subject *in a namespace*
- **ClusterRoleBinding** — grants a ClusterRole cluster-wide

RBAC is **purely additive** — there are no deny rules. Effective permissions are the union of every binding a subject has, so you grant narrowly rather than trying to subtract later.

Best command to know:

```bash
kubectl auth can-i create pods --as=system:serviceaccount:prod:api -n prod
```

### What is a Service Account?

The identity a **Pod** uses to talk to the Kubernetes API — as opposed to user accounts, which are for humans and live outside the cluster.

Every namespace has a `default` ServiceAccount, and Pods use it unless told otherwise. Modern clusters mount a short-lived, audience-bound projected token rather than a permanent Secret.

Two things I'd always say here:

1. The `default` SA should have **no permissions**, and Pods that don't call the API should set `automountServiceAccountToken: false` — an unused mounted token is free lateral movement for an attacker.
2. On EKS, the ServiceAccount is also how you get **AWS** credentials, via IRSA or EKS Pod Identity — annotate the SA with an IAM role and the Pod assumes it. That's how you avoid baking access keys into images.

### Role vs ClusterRole?

Same permission syntax; different scope.

- **Role** is namespaced — it can only grant access to resources in its own namespace.
- **ClusterRole** is cluster-scoped and can additionally cover cluster-scoped resources (nodes, PVs, namespaces, CRDs) and non-resource URLs like `/healthz`.

The useful pattern: define a ClusterRole once as a reusable permission template (say, `app-reader`), then bind it with a **RoleBinding** in each namespace so it applies only there. One definition, per-namespace scope.

### RoleBinding vs ClusterRoleBinding?

The binding determines the *effective* scope, which is where people get caught out.

| Binding | Role referenced | Result |
|---|---|---|
| RoleBinding | Role | Permissions in that namespace |
| RoleBinding | ClusterRole | ClusterRole's permissions, **limited to that namespace** |
| ClusterRoleBinding | ClusterRole | Permissions across **all** namespaces |
| ClusterRoleBinding | Role | **Not allowed** |

Row 2 is the one interviewers like. Row 3 is where accidents happen: binding `cluster-admin` with a ClusterRoleBinding because someone was debugging a permissions error is one of the most common real-world Kubernetes security findings.

### How do you secure Secrets?

Layered, because base64 is not security:

1. **Encrypt etcd at rest** with a KMS provider. On EKS, enable envelope encryption with a customer-managed KMS key.
2. **Restrict RBAC** — very few subjects should have `get`/`list` on Secrets, and `list` on Secrets in a namespace is effectively "read all of them."
3. **Use an external secrets manager** — AWS Secrets Manager or Parameter Store via **External Secrets Operator** or the **Secrets Store CSI driver**. Secrets get mounted at runtime and can rotate without redeploying.
4. **Mount as files, not env vars** — env vars leak into `describe pod`, logs, crash reports, and child processes.
5. **Never commit Secrets to Git** — if you must do GitOps, use SOPS or Sealed Secrets. And run `gitleaks`/`trufflehog` in CI to catch mistakes.
6. **Don't leak them in logs or audit trails** — for example, passing a database URL in an ECS/K8s API call's plaintext environment block writes it into CloudTrail; referencing a secret by ARN doesn't.
7. **Rotate, and scope tightly** — least privilege on the credential itself, so a leak is bounded.

### How do you restrict Pod communication?

The primary tool is **NetworkPolicies**, with a default-deny baseline per namespace and explicit allows on top:

```yaml
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

Then allow only what's needed — plus DNS egress to CoreDNS, which everyone forgets and which breaks everything when omitted.

Beyond that: namespace separation as a boundary, a **service mesh** (Istio/Linkerd) for mTLS and identity-based L7 authorisation, and on AWS, **security groups for Pods** with the VPC CNI when you need to gate access to RDS or other AWS resources.

And the recurring caveat: confirm your CNI actually enforces NetworkPolicy. On EKS with the VPC CNI you must enable network policy support (or run Calico) — otherwise the objects are accepted and ignored.

---

## Monitoring & Logging

### How do you monitor Kubernetes?

The standard stack, and what each piece is for:

- **kube-state-metrics** — object state from the API (replicas desired vs available, Pod phase, restarts, PVC status, cronjob failures).
- **node-exporter** — node-level OS metrics (CPU, memory, disk, network, filesystem).
- **cAdvisor** (inside kubelet) — per-container CPU/memory/network usage.
- **Prometheus** — scrapes and stores; **Alertmanager** routes alerts.
- **Grafana** — dashboards.
- **metrics-server** — lightweight, feeds `kubectl top` and HPA. Not a monitoring system.

Managed equivalents: CloudWatch Container Insights or Amazon Managed Prometheus + Managed Grafana on AWS.

The framing that impresses: monitor in two layers. **Cluster health** (are nodes and control plane fine?) and **service health** (SLI-based — latency, error rate, saturation, traffic). Alert on the second, use the first for diagnosis. Nobody should be paged because CPU hit 80%; they should be paged because the error budget is burning.

### Which metrics are most important?

**Service-level (page on these):**
- Request rate, error rate, latency percentiles (p50/p95/p99) — the SLIs
- Saturation and error-budget burn rate

**Workload-level:**
- Pod restart count and `CrashLoopBackOff` count
- Deployment replicas available vs desired
- OOMKill events
- Pods Pending for longer than a threshold
- HPA at max replicas (you've run out of runway)
- Container CPU throttling (`container_cpu_cfs_throttled_seconds_total`) — the quiet latency killer people miss

**Node/cluster-level:**
- Node `Ready` status, disk/memory/PID pressure
- Allocatable vs requested CPU/memory (capacity headroom)
- Node disk and inode usage
- API Server request latency and error rate; etcd fsync duration and DB size
- CoreDNS latency and error rate
- PV/PVC usage nearing capacity, and certificate expiry dates

### How do you collect logs from Pods?

Containers write to stdout/stderr; the runtime writes that to files on the node; an agent ships it onward. So:

1. Apps log to **stdout/stderr** in **structured JSON** — no writing to files inside the container.
2. A **DaemonSet agent** (Fluent Bit, Fluentd, Vector, or the CloudWatch agent) reads `/var/log/containers/*.log` on each node.
3. It enriches with Kubernetes metadata (namespace, pod, container, labels) and ships to CloudWatch Logs, Loki, OpenSearch, or a SaaS backend.
4. Set node-level log rotation, and retention/lifecycle policies at the backend — log cost gets out of hand fast.

Say why the sidecar-per-Pod pattern is discouraged: it multiplies resource overhead. And mention correlation — inject trace IDs so logs, metrics, and traces line up. That's what actually shortens MTTR.

### How do you monitor cluster health?

Quick checks:

```bash
kubectl get nodes
kubectl get --raw='/readyz?verbose'
kubectl get componentstatuses          # deprecated but still seen
kubectl top nodes
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get events -A --sort-by=.lastTimestamp
```

Continuously, I'd watch four groups:

1. **Control plane** — API Server latency/errors/availability, etcd fsync latency and DB size, scheduler and controller-manager loop health. On EKS, expose control-plane logs to CloudWatch since you can't SSH to it.
2. **Nodes** — Ready condition, the pressure conditions, kubelet health, capacity headroom.
3. **Core add-ons** — CoreDNS, CNI, kube-proxy, CSI drivers, ingress controller. These are shared dependencies with huge blast radius.
4. **Capacity and cost trends** — requested vs allocatable, so you see the wall before you hit it.

Plus a **synthetic probe** that deploys a trivial Pod and curls a Service end-to-end. It catches the compound failures that individual component metrics miss.

---

## Troubleshooting

> For all of these, the shape of a good answer is: **what I observe → what that narrows it to → how I confirm → fix → prevent recurrence.** Jumping straight to a fix reads as guessing.

### A Pod is stuck in Pending. How do you troubleshoot?

```bash
kubectl describe pod <pod>            # read the FailedScheduling event first
```

That event names the reason almost every time. Then, by reason:

- *Insufficient cpu/memory* → `kubectl describe node` and compare Allocatable vs Allocated requests. Either the requests are too big, or the cluster needs a node.
- *had taint that the pod didn't tolerate* → add a toleration, or you're targeting the wrong nodes.
- *node(s) didn't match node selector/affinity* → check node labels: `kubectl get nodes --show-labels`.
- *pod has unbound immediate PersistentVolumeClaims* → jump to the PVC troubleshooting below.
- *didn't match pod anti-affinity rules* → too few nodes/zones for the replica count.
- **No events at all** → the Scheduler itself may be down, or the Pod has a `nodeName` set, or a webhook is stalling admission.

Also check Cluster Autoscaler logs — if it *tried* and failed to add a node, it says why (at max size, no node group fits the shape, insufficient EC2 capacity in the AZ).

### A Pod is in CrashLoopBackOff. What do you check?

```bash
kubectl logs <pod> --previous          # the actual error, 80% of the time
kubectl describe pod <pod>             # Last State: exit code + Reason
```

Then interpret:

- **Exit 137 / Reason: OOMKilled** → memory limit too low, or a leak. Compare limit against real usage before raising it blindly.
- **Exit 1 or 2 with an app stack trace** → config/dependency issue: missing env var, bad DB URL, unreachable downstream at boot.
- **Exit 127** → command not found; wrong entrypoint or wrong base image.
- **Exit 0 in a Deployment** → the process finished. It's not a long-running server, or the entrypoint is wrong.
- **Empty logs and a fast restart** → often a liveness probe killing it before it can start. Check `initialDelaySeconds`, and add a startup probe.
- **Permission errors** → non-root `securityContext` against a volume or path the user can't write.

Escape hatch for a Pod that dies too fast to inspect: temporarily override the command with `sleep 3600`, then exec in and run the real entrypoint by hand.

Prevention: right-size requests/limits from observed usage, fail fast with a clear error message, add a startup probe for slow boots.

### A Pod cannot pull an image. How do you debug it?

Read the exact registry error — don't guess:

```bash
kubectl describe pod <pod> | tail -20
```

Then work outward:

1. **Name/tag** — is the tag real? `aws ecr describe-images --repository-name <repo>`. Typos and a tag your CI never actually pushed are the top two causes.
2. **Auth** — private registry: does `imagePullSecrets` exist in *this namespace* and is it correct? Secrets are namespaced, so a working Pod in another namespace proves nothing.
3. **IAM (ECR)** — the node role (or Pod Identity role) needs `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`. Also check the ECR **repository policy** for cross-account pulls.
4. **Network** — private subnet: is there a NAT gateway, or ECR + S3 VPC endpoints? ECR image *layers* come from S3, so a missing S3 gateway endpoint fails the pull even when ECR itself is reachable. Very common gotcha.
5. **Rate limits / region** — Docker Hub anonymous limits; wrong ECR region in the image URI.

Confirm from the node if you have access: `crictl pull <image>`.

Prevention: immutable digest or SHA-based tags (never `latest`), pull-through cache or VPC endpoints, and a CI step that verifies the pushed tag exists before deploying.

### A Service has no endpoints. What could be wrong?

```bash
kubectl get endpointslices -l kubernetes.io/service-name=<svc>
kubectl describe svc <svc>
```

Empty endpoints has a short list of causes:

1. **Selector doesn't match Pod labels** — the number one cause. Compare `kubectl get svc <svc> -o yaml` selector against `kubectl get pods --show-labels`. A single character difference is enough.
2. **No Pods are Ready** — Pods exist but readiness is failing, so they're excluded by design. `kubectl get pods` and look at the READY column.
3. **Port mismatch** — the Service's `targetPort` doesn't match the container's actual listening port, or a named port doesn't exist in the Pod spec.
4. **Wrong namespace** — Service and Pods must be in the same namespace for a selector to match.
5. **No selector at all** — valid for manually managed Endpoints or `ExternalName`, but then you must maintain the EndpointSlice yourself.

Prevention: this is exactly what a smoke test after deploy catches — assert endpoints are non-empty before declaring success.

### An Ingress is not routing traffic. How do you troubleshoot?

Work down the path, outside in:

```bash
kubectl describe ingress <ing>          # address assigned? events? errors?
kubectl logs -n <ns> deploy/<ingress-controller>
```

1. **Is a controller running?** No controller → no ADDRESS on the Ingress and nothing happens.
2. **Is `ingressClassName` set correctly?** With multiple controllers, an unclaimed Ingress is silently ignored.
3. **DNS** — does the hostname resolve to the LB? `dig`/`nslookup`.
4. **Does the LB exist and are its targets healthy?** In AWS, check ALB target group health — unhealthy targets usually mean the health check path or port is wrong.
5. **Backend Service** — does it have endpoints (previous question)? Is the port right? Test from inside: `kubectl run tmp --rm -it --image=curlimages/curl -- curl http://<svc>.<ns>:<port>`.
6. **Rules** — host and path exactly right? `pathType: Prefix` vs `Exact` catches people, as do rewrite annotations.
7. **TLS** — does the referenced secret exist in the *Ingress's* namespace, or the ACM cert ARN annotation match the region?
8. **Security groups / NetworkPolicy** — can the LB reach the nodes/Pods on the target port?

The fastest way to bisect: curl the Service from inside the cluster. Works internally → problem is Ingress/LB/DNS. Fails internally → problem is the Service or the app.

### A Node is NotReady. What are your steps?

```bash
kubectl describe node <node>        # which condition? what's the message?
```

**First, contain:** `kubectl cordon <node>` so nothing new lands there while you work.

Then diagnose by the condition:

- **`MemoryPressure` / `DiskPressure` / `PIDPressure`** → resource exhaustion. Disk pressure is usually image/log accumulation: check `df -h`, inodes with `df -i`, and run `crictl rmi --prune`.
- **`Ready: Unknown`** → the kubelet has stopped reporting. Either kubelet is dead or the network to the API Server is broken. On the node: `systemctl status kubelet`, `journalctl -u kubelet -n 200`.
- **Node genuinely gone** (EC2 stopped/terminated/hardware failure) → confirm in the AWS console or `aws ec2 describe-instance-status`.
- **CNI plugin crashed** → node can't set up Pod networking, so it reports NotReady. Check the CNI DaemonSet Pods.
- **Clock skew or expired kubelet certificate** → check certificate rotation / CSR approval.

Then decide: fix it in place, or drain and replace. In a cloud/immutable-infrastructure setup, **replacing is usually faster and safer** than debugging a sick node — `kubectl drain` then terminate the instance and let the ASG/Karpenter build a fresh one. Save the diagnostics first if you need a root cause.

Prevention: multiple AZs, PDBs so evictions are safe, node auto-repair, alerting on node conditions, and enough headroom that losing a node doesn't leave Pods Pending.

### A Deployment rollout is stuck. What do you check?

```bash
kubectl rollout status deployment/<name>
kubectl describe deployment <name>       # look for ProgressDeadlineExceeded
kubectl get pods -l app=<name>           # which new Pods aren't Ready?
kubectl describe pod <new-pod>
```

The rollout is stuck because new Pods aren't becoming Ready, so drill into *why the new Pod* is unhealthy:

- Pending → capacity/scheduling (see above). Common with `maxSurge` needing room you don't have.
- ImagePullBackOff → bad tag; frequently a CI pipeline that tagged something different from what the manifest references.
- CrashLoopBackOff → the new version is broken, or its new config/env var is missing.
- Running but not Ready → readiness probe failing. Check the path, the port, the timeout, and whether a dependency it probes is actually down.
- Also check: `maxUnavailable: 0` with no spare capacity (deadlock), a PDB blocking the old Pods' eviction, a mutating webhook failing admission on the new Pods, or a quota rejecting new Pods.

The reassuring thing to say: because old Pods aren't removed until new ones are Ready, **users are still being served the old version**. So you're not in an outage — you're in a failed deploy, which buys you time to either fix forward or `kubectl rollout undo`.

Prevention: `rollout status --timeout` in the pipeline with automatic rollback on failure, plus a PDB and realistic probes.

### Pods cannot communicate with each other. How do you investigate?

Establish the shape of the failure first — is it *all* Pods or some, *cross-node* or same-node, *DNS* or IP-level? That distinction eliminates most causes immediately.

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
# then, from inside:
ping <pod-ip>            # raw L3 reachability
curl <pod-ip>:<port>     # app-level
nslookup <svc>.<ns>      # DNS
curl <svc>.<ns>:<port>   # via the Service
```

Interpretation:

- **Pod IP works, Service name doesn't** → DNS or Service/endpoints problem, not networking.
- **Service resolves but connection refused** → app not listening on that port, or wrong `targetPort`.
- **Same-node works, cross-node doesn't** → CNI, node security groups, or route table. Check the CNI DaemonSet on both nodes and the SG rules between nodes.
- **Small requests work, large ones hang** → classic **MTU mismatch** with an overlay network. Very distinctive symptom, worth naming.
- **Nothing works between specific namespaces/apps** → a NetworkPolicy. `kubectl get netpol -A` and check whether a default-deny is in place without the allow you need (especially DNS egress).
- **Intermittent** → some Pods Ready and some not, so half the endpoints are bad; or a single unhealthy node.

Also check the Pod has an IP at all — no IP means CNI IPAM exhaustion, which on the AWS VPC CNI means subnet IP exhaustion or hitting the per-instance ENI/IP limit.

### DNS resolution is failing inside the cluster. What do you check?

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl run dnsutils --rm -it --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- sh
# nslookup kubernetes.default
```

Checklist:

1. **Are CoreDNS Pods Running and Ready?** If they're crashlooping or evicted, everything downstream is broken.
2. **Is the `kube-dns` Service reachable and does it have endpoints?**
3. **Pod's `/etc/resolv.conf`** — correct nameserver IP, and correct `search` domains? A wrong `dnsPolicy` (e.g. `Default` instead of `ClusterFirst`) breaks cluster names.
4. **Internal vs external** — `kubernetes.default` fails → CoreDNS or connectivity. Internal works but `google.com` fails → upstream forwarding, the CoreDNS `forward` config, or NAT/egress.
5. **NetworkPolicy blocking egress to port 53** — extremely common after introducing default-deny.
6. **CoreDNS resource limits / throttling** — under load it gets CPU-throttled and you see intermittent timeouts rather than clean failures.
7. **`conntrack` table exhaustion** on nodes, and the old UDP conntrack race — symptom is ~5-second DNS delays. Mitigations: NodeLocal DNSCache, `ndots:2`, or using TCP.

Prevention: scale CoreDNS with cluster size (or run cluster-proportional autoscaling), monitor its latency and error rate, and add NodeLocal DNSCache for high-QPS clusters.

### A Pod cannot connect to a database. How do you troubleshoot?

Layer by layer, and identify the failure *mode* first — DNS failure, timeout, connection refused, and auth error each point somewhere different.

```bash
kubectl exec -it <pod> -- nslookup <db-host>
kubectl exec -it <pod> -- nc -zv <db-host> 5432
kubectl exec -it <pod> -- env | grep -i db      # is the config even right?
```

- **DNS fails** → RDS endpoint typo, or cluster DNS can't forward externally, or you need to be inside the VPC/private hosted zone.
- **Timeout (hangs)** → almost always a **network/firewall** issue: RDS security group doesn't allow the node/Pod security group on 5432, wrong subnet/route, missing NAT, or an egress NetworkPolicy.
- **Connection refused** → reachable but nothing listening: wrong port, or the DB is down.
- **Auth error** → credentials. Check the Secret is present, mounted, and *current* — a rotated password with a stale Secret is a classic. Watch for special characters in a connection URL that get mangled by shell expansion or need percent-encoding (`$` and `@` are frequent offenders).
- **Works sometimes** → **connection pool exhaustion**. Every Pod opens its own pool, so scaling replicas multiplies connections and blows past `max_connections`. Fix with a pooler (PgBouncer/RDS Proxy) and sane per-Pod pool sizes.
- **SSL error** → the DB requires TLS and the client isn't configured, or the CA bundle isn't in the image.

On AWS specifically, check whether the Pod's identity has what it needs — with IAM database authentication, the IRSA/Pod Identity role must have `rds-db:connect`.

### A PVC is stuck in Pending. What could be the issue?

```bash
kubectl describe pvc <pvc>              # events state the reason
kubectl get storageclass
kubectl logs -n kube-system deploy/ebs-csi-controller   # provisioner logs
```

Causes:

1. **No StorageClass / no default StorageClass** — the PVC omitted `storageClassName` and no default exists, so nothing provisions it. Very common on fresh EKS clusters, because **EKS has no default StorageClass until you install the EBS CSI driver** and mark one default.
2. **CSI driver not installed or unhealthy** — no provisioner to answer the claim.
3. **IAM permissions** — the EBS CSI controller's role can't call `ec2:CreateVolume`/`AttachVolume`. Check the controller logs; the AWS error is right there.
4. **AZ mismatch** — with `volumeBindingMode: Immediate`, the volume was created in an AZ with no schedulable node. Use `WaitForFirstConsumer`.
5. **No matching PV** for static provisioning — capacity, accessMode, or selector mismatch. Note that `ReadWriteMany` on EBS is impossible; you need EFS.
6. **Quota or cloud limits** — EBS volume limits, or a ResourceQuota on storage in the namespace.
7. **Stuck on a previous attachment** — the volume is still attached to another node, or a `volumeattachment` is hung.

Prevention: set a default StorageClass, use `WaitForFirstConsumer`, and alert on PVCs Pending beyond a few minutes.

### A Pod keeps restarting every few minutes. How do you debug it?

The *periodicity* is the clue — a regular interval suggests something is killing it on a schedule rather than a startup failure.

```bash
kubectl describe pod <pod>       # Last State, Reason, Exit Code, restart count
kubectl logs <pod> --previous
kubectl top pod <pod>            # memory trending up?
```

Ranked causes:

1. **Slow memory leak → OOMKilled.** Exit 137, `Reason: OOMKilled`. Memory climbs steadily and hits the limit at a predictable interval. Watch `container_memory_working_set_bytes` and confirm the sawtooth pattern.
2. **Liveness probe failing intermittently.** Look for `Unhealthy` events in `describe`. Often the probe times out under load rather than the app being broken — `timeoutSeconds: 1` against an endpoint that occasionally takes 1.2s. Loosen the probe or fix the endpoint; don't just increase `failureThreshold` and hope.
3. **Application crashing on a specific input** — a poison-pill message, a periodic cron-style task, a nightly job.
4. **Node-level eviction** — recurring memory/disk pressure on the node evicting the Pod. Check node conditions, not just the Pod.
5. **Dependency flapping** — the app exits when it loses its database or queue.

Distinguishing test: if it's OOM, the interval scales with traffic; if it's a probe, `describe` shows Unhealthy events; if it's eviction, the Pod name changes rather than the restart count incrementing.

### How do you investigate high CPU usage in a cluster?

Go top-down: cluster → node → Pod → process.

```bash
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl describe node <node> | grep -A5 "Allocated resources"
```

Then separate the two questions people conflate:

- **Real CPU exhaustion** — a workload genuinely burning cores. Find it with `kubectl top`, then profile inside it. Check whether it's a code regression (correlate with the last deploy), a traffic increase (correlate with request rate), or a runaway loop.
- **CPU throttling despite low usage** — the sneaky one. A container with a CPU *limit* gets throttled by CFS quota even when the node is idle, causing latency spikes with unremarkable-looking utilisation. Check `container_cpu_cfs_throttled_seconds_total` and `..._throttled_periods_total`. This is why many teams set CPU requests but **no CPU limits**.

Also: over-committed requests can make the *scheduler* refuse work while nodes look idle — that's a requests problem, not a CPU problem. And check that HPA isn't stuck at max.

Remediation ladder: scale out (HPA), right-size requests/limits, remove CPU limits if throttling is the issue, add nodes, then fix the application.

### How do you investigate high memory usage in a cluster?

```bash
kubectl top pods -A --sort-by=memory
kubectl get events -A | grep -i oom
kubectl describe node <node> | grep -A5 "Allocated resources"
```

Key differences from CPU, worth stating explicitly: **memory is incompressible.** You can throttle CPU, but exceeding a memory limit gets you killed — no gradual degradation. So:

- **Exceeding the container limit** → OOMKilled, exit 137, container restarts. Bounded blast radius.
- **Node running out of memory** → kubelet evicts Pods by QoS class (BestEffort first, then Burstable over requests, Guaranteed last). If it happens faster than kubelet can react, the kernel OOM killer fires and can take out something important. Wider blast radius.
- **Leak vs legitimate growth** → graph `container_memory_working_set_bytes` over days. Sawtooth with restarts = leak. Step change after a deploy = regression. Correlated with traffic = under-provisioned.
- Watch for JVM/Node.js heap settings that ignore the container limit — a JVM without `MaxRAMPercentage` sizing itself from the *host's* memory is a very common OOM cause.

Remediation: right-size requests and limits from real p99 usage, set requests == limits (Guaranteed QoS) for critical Pods, reserve node capacity with `--kube-reserved`/`--system-reserved`, and fix the leak. Also note `container_memory_working_set_bytes` is the metric the OOM killer effectively cares about — not RSS, and page cache can mislead you.

### How do you troubleshoot disk pressure on a node?

```bash
kubectl describe node <node> | grep -i pressure
# on the node:
df -h ; df -i          # inodes matter as much as bytes
du -sh /var/lib/containerd /var/log/*
```

`DiskPressure` makes the node stop accepting new Pods and start evicting existing ones, and kubelet begins garbage-collecting images. Usual culprits:

1. **Unused container images** accumulating — the most common. `crictl rmi --prune`, and tune kubelet's `imageGCHighThresholdPercent`.
2. **Container logs not rotating** — a chatty app filling `/var/log/containers`. Set `containerLogMaxSize`/`containerLogMaxFiles`.
3. **An app writing inside the container or to `emptyDir`** — `emptyDir` consumes node disk. Bound it with `sizeLimit`, or use a PVC.
4. **Inode exhaustion** — plenty of free bytes but millions of tiny files. `df -i` is the only way you'll see it.
5. **Undersized root volume** for the workload's image sizes.

Immediate action: cordon, clean up, verify the condition clears, then uncordon. If it recurs, fix the cause rather than the symptom.

Prevention: alert on disk *and inode* usage with enough lead time, right-size node volumes, enforce log rotation and `emptyDir` limits, use smaller images (multi-stage builds, distroless), and prefer replacing nodes periodically over letting them accumulate cruft.

### What would you do if the API Server is unreachable?

First, the reassuring fact to lead with: **running Pods keep serving traffic.** kube-proxy rules and running containers don't depend on the API Server. What you lose is *change* — no deploys, no scaling, no self-healing, no `kubectl`.

Triage:

```bash
kubectl get --raw='/readyz?verbose'
curl -k https://<api-endpoint>/livez
```

- **Is it me or the cluster?** Check from another network/machine. Expired kubeconfig credentials, a rotated token, wrong context, or an IP allowlist blocking you look identical to an outage but aren't.
- **On EKS**, check the AWS Health Dashboard and the cluster status, plus whether the endpoint is private-only and you're outside the VPC/VPN. Also check security groups and the API server's public access CIDRs.
- **Self-managed**: is etcd healthy (the most common real cause)? Are the API server Pods/processes running on the control-plane nodes? `journalctl -u kube-apiserver`. Check certificate expiry — an expired control-plane cert is a classic annual outage.
- **Overload**: too many requests, an expensive `list` on a huge resource set, or a runaway controller hot-looping. Look at etcd latency and API request duration; consider priority-and-fairness settings.
- **A failing admission webhook** with `failurePolicy: Fail` can make the API server appear broken for writes — good detail to mention.

Prevention: HA control plane across AZs (or a managed control plane), monitor cert expiry, alert on API latency and etcd health, and don't let a webhook be a single point of failure.

### What happens if etcd becomes unavailable?

The cluster becomes **read-mostly and frozen**:

- Existing Pods keep running and serving traffic. kube-proxy and CNI state stay as they are.
- The API Server can't write, so nothing changes: no create/update/delete, no rollouts, no scaling, no new Pod scheduling.
- **Self-healing stops.** A Pod that crashes may still be restarted by the local kubelet, but a Pod on a *failed node* will not be recreated elsewhere, because that requires writes.
- Controllers can't reconcile; leader election fails, so controller-manager and scheduler drop out.
- If quorum is *lost* but members survive, etcd goes read-only; if the data is destroyed with no backup, the cluster is effectively gone.

So the honest summary is: **not an immediate outage, but a loss of all resilience.** You're one node failure away from a real outage, with no ability to respond. That's why it's a sev-1 even though customers may not notice yet.

### How do you recover from an etcd failure?

Depends on the failure mode:

**Lost one member of three (quorum intact):** the least dramatic case. Remove the failed member and add a fresh one; it syncs from the leader.

```bash
etcdctl member list
etcdctl member remove <id>
etcdctl member add <name> --peer-urls=https://<ip>:2380
```

**Lost quorum (2 of 3 down):** you must restore from a snapshot, or bring the failed members back if their data is intact. Don't improvise here — verify the newest good snapshot first.

**Full restore from snapshot:**

```bash
# take snapshots routinely:
etcdctl snapshot save /backup/etcd-$(date +%F).db
etcdctl snapshot status /backup/etcd-2026-07-26.db     # verify BEFORE you need it

# restore:
etcdctl snapshot restore /backup/etcd-2026-07-26.db \
  --data-dir=/var/lib/etcd-restored
# point the etcd manifest at the new data dir, restart etcd, then the API server
```

Practical points that matter more than the commands: stop the API servers first so nothing writes during restore; restore to **all** members consistently; expect to lose everything created after the snapshot (so snapshot frequency defines your RPO); and afterwards reconcile drift — the cluster's view of the world may be behind reality, and resources created since the snapshot will vanish.

**And the answer that scores best:** on EKS this is not your problem — AWS manages and backs up etcd. Which is exactly why I'd choose a managed control plane for production unless there's a strong reason not to. Separately, keep all manifests in Git so the cluster is rebuildable from source regardless of etcd — that's the real disaster recovery story.

### How do you troubleshoot kubelet failures?

kubelet dying means the node goes `NotReady` and stops reporting, even though containers may keep running for a while.

```bash
systemctl status kubelet
journalctl -u kubelet -n 300 --no-pager
crictl ps ; crictl info          # is the runtime healthy underneath?
```

Common causes, in order:

1. **Certificate problems** — expired kubelet client cert, or CSR rotation not being approved. Very common in long-lived self-managed clusters.
2. **Bad configuration** — a typo in `/var/lib/kubelet/config.yaml` or the systemd unit after a change. The logs say so plainly.
3. **Container runtime down** — kubelet can't talk to containerd via the CRI socket. Check `systemctl status containerd`.
4. **Resource starvation on the node** — kubelet itself starved of CPU/memory by workloads. This is what `--kube-reserved`/`--system-reserved` exist to prevent.
5. **Disk full** — kubelet can't write state.
6. **Network/API connectivity** — can't reach the API Server: security group, DNS, or endpoint change. On EKS, check the bootstrap script and cluster endpoint/CA in the node config.
7. **Version skew** — kubelet more than the supported number of minor versions off the control plane.

Response: cordon and drain the node, fix or replace it. In an immutable-infrastructure setup, replace — terminate the instance and let the ASG/Karpenter provide a clean one. Capture logs first if you need a root cause.

### How do you investigate network latency between Pods?

Quantify before theorising — "latency" from an app team often turns out to be application time, not network time.

```bash
kubectl exec -it <pod> -- sh
# ping <pod-ip>            # RTT and jitter, baseline L3
# curl -w "@curl-format" -o /dev/null -s http://<svc>:<port>
#   → shows dns / connect / tls / ttfb / total separately
# mtr <target>             # per-hop loss and latency
```

That `curl -w` breakdown is the highest-value move: it tells you immediately whether time is going to **DNS**, **TCP connect**, **TLS handshake**, or **server processing**. Each points somewhere different.

Then consider, roughly in order of how often they're the answer:

- **DNS**, not the network — slow or timing-out lookups (the 5-second conntrack race is a signature). Fix with NodeLocal DNSCache or `ndots` tuning.
- **CPU throttling** on the client or server container — looks exactly like network latency. Check throttling metrics.
- **Cross-AZ traffic** — single-digit-ms extra per hop, plus data transfer cost. Use topology-aware routing or `topologySpreadConstraints` to keep chatty services in-zone.
- **Overlay encapsulation overhead**, and **MTU mismatch** causing fragmentation or retransmits.
- **conntrack table saturation** on the node → dropped packets and retries.
- **Node saturation** — NIC bandwidth limits on smaller instance types, or a noisy neighbour Pod.
- **Service mesh sidecar** — an extra proxy hop each way; check the proxy's own metrics and resources.
- **Connection churn** — no keep-alive, so every request pays TCP + TLS setup. Often the real culprit.

The right long-term answer is **distributed tracing**: it attributes latency per hop across services, which turns this from guesswork into a lookup.

---

## Production SRE Scenarios

### How would you perform a zero-downtime deployment?

Zero downtime isn't a single setting — it's a set of preconditions that all have to hold:

1. **Multiple replicas** across nodes/AZs. One replica cannot be zero-downtime, full stop.
2. **`maxUnavailable: 0`, `maxSurge: 1`** (or a percentage) — add capacity before removing any.
3. **Accurate readiness probe** — this is what stops traffic reaching a Pod that isn't ready, and what gates the rollout.
4. **Graceful shutdown** — handle SIGTERM: stop accepting new connections, drain in-flight requests, close pools, exit. Set `terminationGracePeriodSeconds` above your longest request.
5. **`preStop` sleep of a few seconds** — endpoint removal and SIGTERM happen concurrently, and load balancers take time to notice. This closes the race that causes those few 502s during every deploy.
6. **PodDisruptionBudget** so voluntary disruptions (node drains) can't take too many replicas at once.
7. **Backward-compatible changes** — expand/contract database migrations, additive API changes. Old and new versions run simultaneously during the rollout, so both must work against the same schema.
8. **Automated verification** — `kubectl rollout status --timeout=5m`, then smoke tests, with automatic rollback on failure.

For higher-risk releases, layer on canary or blue/green with Argo Rollouts or Flagger so a bad version is exposed to 5% of traffic instead of 100%.

The point I'd emphasise: most "Kubernetes dropped requests during deploy" incidents are #4 and #5, not the rollout strategy.

### How would you upgrade a Kubernetes cluster?

Order matters: **control plane first, then node groups, then add-ons** — and never skip minor versions.

1. **Read the release notes and the deprecated API guide.** Removed APIs are the #1 upgrade breakage. Scan the cluster with `kubectl-convert`, `pluto`, or `kubent` for manifests using APIs removed in the target version.
2. **Check compatibility** of every add-on and controller: CNI, CSI drivers, ingress controller, cert-manager, metrics-server, autoscaler, service mesh. One incompatible controller can break the cluster.
3. **Practise in a lower environment** — QA/staging first, same version path.
4. **Confirm backups** — etcd snapshot (self-managed) and manifests in Git.
5. **Upgrade the control plane** (one minor version). On EKS: `eksctl upgrade cluster` or the console/API; AWS handles it in a rolling fashion.
6. **Upgrade add-ons** to versions matching the new control plane.
7. **Upgrade nodes** — replace, don't patch in place (next question).
8. **Verify** — nodes Ready, all workloads healthy, run smoke tests, watch error rates and latency for a while afterwards.

Respect **version skew**: kubelet may be up to 3 minor versions behind the API server (in recent releases), never ahead. So control plane first, always.

### How would you upgrade worker nodes without downtime?

Rolling replacement, one node (or a small batch) at a time. The safe loop:

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=600s
# terminate the instance; let the ASG / Karpenter bring up a new one on the new AMI
kubectl get nodes            # confirm the replacement is Ready
# repeat
```

What makes it actually safe rather than just sequential:

- **PodDisruptionBudgets** — drain uses the Eviction API and respects PDBs, so it refuses to break your availability guarantee. If drain blocks, that's the PDB doing its job; investigate rather than force.
- **Spare capacity or surge nodes** — the evicted Pods need somewhere to go. Add the new node *before* draining the old one where possible (surge upgrade).
- **Anti-affinity / topology spread** so replicas weren't all on that node to begin with.
- **Graceful shutdown handling**, same as deployments — otherwise every drain drops connections.
- Do it **during low traffic** and one AZ at a time for large clusters.

Managed options do this for you: **EKS managed node groups** support a rolling update that respects PDBs, and **Karpenter drift/consolidation** replaces nodes automatically as AMIs change. Prefer these over hand-rolled scripts — less toil, fewer mistakes.

Immutable infrastructure is the principle: never `apt upgrade` a node in place. Build a new AMI, roll new nodes, delete old ones.

### How would you handle a node failure?

**Automatically, mostly** — and the interview answer should highlight that Kubernetes already does most of it:

1. kubelet stops reporting; after ~40s the node controller marks the node `NotReady`.
2. After the toleration period (default 300s), `NoExecute` taints cause the Pods to be evicted.
3. Controllers create replacement Pods; the Scheduler places them on healthy nodes.
4. Endpoints are updated so traffic stops going to the dead Pods.
5. The ASG (or Karpenter) notices the unhealthy instance and replaces it.

My job as an SRE is to make sure that path *works*, and to shorten it:

- **Multiple replicas across AZs** with anti-affinity/topology spread — otherwise recovery means a cold start, not a failover.
- **Enough headroom** that replacement Pods can actually schedule; otherwise they sit Pending and you're down.
- **PDBs** so this doesn't compound with an in-progress drain.
- **StatefulSets are the hard case** — an EBS volume is AZ-bound, so `postgres-0` can only come back in the same AZ. Plan for that explicitly.
- **Tune the eviction timeout** if 5 minutes of degraded capacity is too long for the service.

And then triage the node itself: cordon (already effectively done), gather diagnostics if you need root cause, and let it be replaced rather than repaired.

### How do you recover from an accidental Deployment deletion?

**If you practise GitOps: re-sync from Git.** That's the whole answer, and it's the answer interviewers want. The cluster is not the source of truth; the repo is.

```bash
kubectl apply -f deployments/api.yaml
# or: argocd app sync api  /  flux reconcile kustomization api
```

If it isn't in Git, in rough order of desperation:

1. **Cluster backup** — Velero can restore individual namespaces or resources.
2. **Audit logs** — the API server audit log contains the full object in the delete request's `requestObject`/`responseObject`. On EKS, that's in CloudWatch Logs. Slow, but it works.
3. **Rebuild from a running twin** — if the same app runs in another environment, `kubectl get deploy api -o yaml` there and adapt it.
4. **Rebuild from scratch** using the image tag from your registry and CI history.

Then close the hole, which is the more important half of the answer:

- **RBAC**: developers shouldn't have `delete` on Deployments in prod. Use a CI/CD service account for changes.
- **GitOps with self-heal** — an accidental deletion gets reverted automatically within seconds.
- **Velero backups** on a schedule, with restores actually tested.
- Consider a **validating webhook** or Kyverno policy blocking deletion of resources with a protection label.
- Audit logging enabled and alerting on unexpected deletes in production namespaces.

### How do you perform disaster recovery for Kubernetes?

Start by defining **RTO and RPO** — everything else follows from those numbers, and saying so is what separates a senior answer from a list of tools.

Then split state into three buckets, because each needs a different strategy:

1. **Cluster configuration (manifests)** → Git. This should be fully reproducible: `terraform apply` for the cluster, then Argo/Flux applies the workloads. If you can rebuild a cluster from source in under an hour, most DR scenarios stop being scary.
2. **Cluster object state** → Velero, scheduled, including PV snapshots. Useful for namespace-level restores and for state that isn't in Git (some CRDs, cert-manager certs, sealed secrets).
3. **Application data** → the actual hard part. RDS automated backups plus point-in-time recovery, EBS snapshots via the CSI snapshotter, S3 versioning and cross-region replication. Kubernetes DR does not cover your database; that's a separate plan.

Architecture choices by RTO: **multi-AZ** as a baseline (survives a zone loss automatically), **warm standby cluster in a second region** with data replication for low RTO, or **cold rebuild from IaC** if hours are acceptable and cost matters more.

And the part everyone skips: **test the restore.** A backup you've never restored is a hypothesis. Run a game day, restore into a scratch cluster, time it, and document the runbook. Also back up the things people forget: secrets, DNS records, TLS certificates, and the IaC state file itself.

### How do you back up etcd?

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# always verify
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-2026-07-26.db --write-out=table
```

Operational practice around it:

- **Schedule it** (CronJob or systemd timer) at a frequency that matches your RPO — snapshot interval *is* your RPO.
- **Ship it off-node** to S3 with versioning, lifecycle rules, and encryption. A backup on the failing machine is not a backup.
- **Verify every snapshot** with `snapshot status`, and periodically do a real restore into a scratch environment.
- **Also back up the PKI directory** (`/etc/kubernetes/pki`) — restoring etcd without the original CA certs is painful.
- **Encrypt at rest** — the snapshot contains every Secret in the cluster in whatever form etcd holds them. Treat it as a crown-jewel artifact with tight access control.

For EKS: you don't and can't — AWS manages etcd, including backups. Your equivalents are Git for manifests and Velero for cluster state, which is the honest answer to give.

### How do you reduce toil in Kubernetes operations?

Toil is manual, repetitive, automatable work that scales with the system and adds no enduring value. The approach:

1. **Measure it first.** Categorise your tickets and interrupts for a couple of weeks. You can't reduce what you haven't quantified, and you'll usually find two or three sources dominate.
2. **Then, roughly in order of leverage:**
   - **GitOps** — Argo CD/Flux. Removes manual `kubectl apply`, gives an audit trail, and auto-corrects drift.
   - **IaC for everything** — Terraform for clusters and cloud resources; no console changes.
   - **Autoscaling** — HPA + Karpenter/Cluster Autoscaler removes manual capacity work entirely.
   - **Self-service platform** — Helm charts or templates so app teams ship without a platform ticket. This is usually the single biggest toil reducer, because it removes *you* from the path.
   - **Operators/controllers** for recurring stateful operations.
   - **Policy as code** — OPA/Gatekeeper or Kyverno enforces standards automatically instead of you reviewing YAML.
   - **Better alerts** — alert on symptoms and SLO burn, not causes. Delete alerts that never lead to action; every noisy alert is toil.
   - **Runbooks, then automate the runbook** — if a runbook is followed identically every time, it should be a script or a controller.
   - **Automated certificate and secret rotation** (cert-manager, External Secrets).
3. **Set a target** — the SRE convention is under 50% of time on toil — and review it. Otherwise it silently creeps back.

### How do you secure a production Kubernetes cluster?

Layered, and I'd organise it as: cluster → workload → supply chain → runtime.

**Cluster / control plane**
- Private API endpoint or a tight CIDR allowlist; access via VPN or bastion.
- RBAC least privilege; no `cluster-admin` handed out casually; short-lived credentials via OIDC/SSO rather than static kubeconfigs.
- Audit logging enabled and shipped somewhere you actually query.
- etcd encryption at rest with a customer-managed KMS key.
- Patch and upgrade on a schedule — running an EOL version is itself the vulnerability.

**Workload**
- Pod Security Standards (`restricted`) enforced at the namespace level: no privileged containers, no host namespaces, drop capabilities.
- `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, seccomp `RuntimeDefault`.
- Resource requests and limits on everything — a missing limit is an availability risk.
- NetworkPolicies with default-deny; segment namespaces.
- Dedicated ServiceAccounts, `automountServiceAccountToken: false` where unused; IRSA/Pod Identity instead of long-lived AWS keys.
- Secrets from an external manager, mounted as files.

**Supply chain**
- Image scanning in CI (Trivy/Grype) with a build-failing severity threshold, plus registry-side continuous scanning.
- Minimal base images — distroless or Alpine, multi-stage builds. Less surface, fewer CVEs.
- Immutable digest-pinned tags; signing and verification (Cosign) with an admission policy that rejects unsigned images.
- SAST and secret scanning (Semgrep, Gitleaks) and IaC scanning (Checkov/tfsec) in the pipeline.
- Admission policy as code (Kyverno/Gatekeeper) so standards are enforced, not requested.

**Runtime**
- Runtime threat detection (Falco or GuardDuty for EKS).
- Continuous posture checks (Prowler, kube-bench against the CIS benchmark).
- Alert on anomalies: unexpected `exec` into prod Pods, privilege escalation, new outbound destinations.

The framing that lands well: shift left where it's cheap (CI scanning, policy as code), but assume something gets through, so have runtime detection and a blast radius plan too.

### How do you optimize Kubernetes costs?

The dominant cost driver in almost every cluster is **unused requested capacity** — people request 2 CPUs and use 200m, and you pay for the request. So:

1. **Right-size requests and limits from real data.** VPA in recommendation mode, or Goldilocks/Kubecost reports. This alone routinely cuts 30–50%.
2. **Get visibility first** — Kubecost or OpenCost for per-namespace/team/workload showback. Untracked cost never improves; showback changes behaviour on its own.
3. **Cheaper compute**: Spot instances for stateless and fault-tolerant workloads (with PDBs, spread across instance types, and interruption handling), Savings Plans or Reserved Instances for the steady baseline, and Graviton/ARM where your images support it — typically 20%+ cheaper per unit of performance.
4. **Better bin-packing** — Karpenter consolidation picks right-sized instances and defragments nodes continuously, instead of fixed node groups half-empty.
5. **Scale to demand** — HPA, plus scaling non-prod to zero outside working hours. Dev and QA clusters running 24/7 are pure waste.
6. **Cut the hidden line items**, which people always forget:
   - **Cross-AZ data transfer** — use topology-aware routing to keep traffic in-zone.
   - **One ALB/NLB per Service** — consolidate behind a shared Ingress or ALB group.
   - **Log and metric volume** — retention policies and sampling; observability bills can rival compute.
   - **Orphaned EBS volumes and snapshots** from deleted PVCs; unattached EIPs; old ECR images (lifecycle policies).
   - **NAT Gateway data processing** — VPC endpoints for S3/ECR both cut cost and improve reliability.
7. **Clean up abandoned namespaces and workloads** — every long-lived cluster has some.

The SRE caveat worth voicing: cost optimisation has a reliability trade-off. Trimming all headroom means autoscaling can't absorb a spike. Optimise to a target of *utilisation with headroom*, not maximum utilisation.

### How do you troubleshoot intermittent application failures in Kubernetes?

Intermittent means "some subset of requests or some subset of Pods" — so the first job is to find the pattern, not the fix.

**Narrow the axis:**
- Is it **specific Pods**? Compare error rates per Pod (`pod` label in your metrics). One bad replica among five gives you exactly 20% errors — a very recognisable signature.
- Is it **specific nodes**? A single node with a failing CNI, disk pressure, or a noisy neighbour.
- Is it **specific AZs**? Cross-AZ latency or a zonal issue.
- Is it **time-correlated**? Deploys, cron jobs, batch loads, HPA scaling events, node rotations, certificate renewals.
- Is it **load-correlated**? Then it's saturation somewhere: connection pool, thread pool, CPU throttling, or conntrack.

**Then the usual suspects for intermittent-specifically:**
- **Rollout races** — requests hitting terminating Pods because there's no `preStop` delay or SIGTERM handling. Classic "errors only during deploys."
- **Readiness flapping** — Pods entering and leaving endpoints. `kubectl get events` shows the Unhealthy churn.
- **OOMKills** on a subset of Pods under uneven load.
- **CPU throttling** causing timeouts that look random.
- **DNS timeouts** — the 5s conntrack race gives you sporadic failures with a suspiciously round latency.
- **Connection pool exhaustion** at the database as replicas scale.
- **Idle-timeout mismatches** — LB idle timeout shorter than the app's keep-alive means occasional resets.
- **One stale endpoint** pointing at a dead Pod.

**Tooling that actually solves these:** distributed tracing to find which hop fails, structured logs with request IDs so you can follow one failing request end to end, and per-Pod metric breakdowns rather than aggregates. Aggregate dashboards hide intermittent failures by definition — averaging is how you lose a 2% error rate.

### How would you migrate workloads from one cluster to another?

The overall shape: stand up the new cluster, run both in parallel, shift traffic gradually, keep the ability to roll back, then decommission.

1. **Build the new cluster from IaC**, matching or upgrading versions, with all add-ons and policies in place. Verify it independently before any traffic touches it.
2. **Inventory what has to move**: workloads, ConfigMaps/Secrets, PVs, CRDs, DNS records, TLS certs, IAM roles/IRSA trust policies, ingress and load balancers, external integrations, and CI/CD targets. The non-obvious items are usually what bite.
3. **Deploy workloads via GitOps** — point Argo/Flux at the new cluster and let it build the same state. If you can't do this, that's a signal your manifests aren't the source of truth, and that's the real problem to fix first.
4. **Handle data deliberately** — this is the crux:
   - Stateless: trivial, just redeploy.
   - Databases: prefer *external* managed data stores (RDS) that both clusters can reach, so migration doesn't involve moving data at all. If the DB is in-cluster, use replication and a planned cutover, or restore from snapshot with a write freeze.
   - PVs: Velero with snapshots, or a sync tool. Cross-region means copying snapshots.
5. **Shift traffic gradually** — weighted DNS (Route 53) or a global load balancer, 5% → 25% → 50% → 100%, watching error rates and latency at each step. Keep TTLs low *before* you start.
6. **Validate at each stage** with smoke tests and real SLI comparison between clusters.
7. **Keep the old cluster warm** until you're confident — that's your rollback.
8. **Decommission** and clean up cloud resources, IAM, and DNS.

The two things I'd flag as most likely to go wrong: DNS TTLs not lowered in advance (so rollback takes hours), and stateful data — everything else is comparatively mechanical.

### How do you manage secrets in production?

The target state: **secrets live in a dedicated secret manager, are pulled at runtime, and never touch Git or an image.**

Concretely, on AWS:

- **AWS Secrets Manager / SSM Parameter Store** as the store of record, with automatic rotation where the service supports it.
- **External Secrets Operator** to sync into Kubernetes Secrets, or the **Secrets Store CSI driver** to mount directly into Pods without creating a K8s Secret at all. The CSI approach is stronger — nothing lands in etcd.
- **IRSA / EKS Pod Identity** so the Pod's ServiceAccount authorises access. No static AWS keys, per-workload least privilege on individual secret ARNs.
- **Mount as files, not env vars** — env vars leak into `describe pod`, logs, crash dumps, and child processes.
- **etcd encryption at rest** with a customer-managed KMS key as defence in depth.
- **Tight RBAC** on Secrets, and remember `list` on Secrets in a namespace means read-everything.
- **No secrets in Git.** If GitOps requires something in the repo, use SOPS or Sealed Secrets (encrypted at rest in Git), and run `gitleaks`/`trufflehog` in CI to catch accidents.
- **No secrets in images or build args** — build args persist in image history.
- **Rotation with a graceful path**: prefer dual-credential rotation so a rotation doesn't require a synchronised restart. Note that mounted CSI secrets can refresh without a restart; env vars cannot.
- **Audit everything** — CloudTrail on secret access, and alert on unusual read patterns.

Also worth naming: be careful how secrets travel through your *pipeline*. Passing a database URL in a plaintext environment block of an API call can write it into CloudTrail; referencing it by ARN doesn't. Same secret, very different exposure.

### How do you handle certificate expiration in Kubernetes?

Three separate certificate populations, and conflating them is a common mistake.

**1. Application / ingress TLS certs** — automate completely with **cert-manager**, which issues from Let's Encrypt or a private CA and renews well before expiry. On AWS, ACM certificates on an ALB renew automatically as long as DNS validation records stay in place. Monitor either way with a blackbox exporter checking `probe_ssl_earliest_cert_expiry`, and alert at 30 and 14 days.

**2. Cluster PKI (API server, kubelet, etcd, front-proxy)** — the one that causes surprise outages, because these are typically **one-year** certs in kubeadm clusters and nobody has a calendar entry.

```bash
kubeadm certs check-expiration
kubeadm certs renew all      # then restart control plane components
```

Enable kubelet certificate rotation (`rotateCertificates: true`) with automatic CSR approval. Note that upgrading a cluster renews these as a side effect, which is why clusters that upgrade regularly rarely hit this — and clusters left alone for 12 months do.

**3. Webhook and service-mesh certs** — admission webhook CA bundles and mesh mTLS certs. An expired webhook cert with `failurePolicy: Fail` blocks all matching API writes, which presents as "the cluster is broken." Manage these with cert-manager too, and consider `failurePolicy: Ignore` for non-security-critical webhooks.

The prevention story is the answer: **automate renewal, monitor expiry independently of the renewal mechanism** (so you catch automation that silently stopped working), and don't rely on a human remembering. Also keep an inventory — including certs outside the cluster, like the ones on your load balancers and in your CI system.

### What best practices do you follow for running production workloads on Kubernetes?

I'd group these by what they protect:

**Availability**
- Minimum 2–3 replicas, spread across nodes and AZs via `topologySpreadConstraints`.
- PodDisruptionBudgets on everything that matters.
- Accurate readiness probes; liveness probes that check only the process.
- Graceful shutdown: SIGTERM handling, `preStop` delay, `terminationGracePeriodSeconds` above the longest request.
- `maxUnavailable: 0` for rollouts, with automated rollback on failure.

**Resource discipline**
- Requests and limits on every container, right-sized from real usage.
- Requests == limits (Guaranteed QoS) for latency-critical workloads; consider omitting CPU limits to avoid throttling.
- Capacity headroom so autoscaling and node failure have somewhere to go.

**Deployment hygiene**
- Everything in Git; GitOps for delivery; no `kubectl apply` from laptops in prod.
- Immutable, digest- or SHA-pinned image tags — never `latest`.
- Same artifact promoted through environments, config injected per environment.
- Backward-compatible database migrations, decoupled from the deploy.

**Security**
- Pod Security Standards `restricted`, non-root, read-only root filesystem, dropped capabilities.
- NetworkPolicy default-deny; per-workload ServiceAccounts with IRSA/Pod Identity.
- External secret management, images scanned and signed, policy enforced at admission.

**Observability**
- SLIs and SLOs defined per service, with alerts on symptoms and error-budget burn — not on CPU.
- Structured JSON logs to stdout, shipped centrally; metrics; distributed tracing.
- Dashboards and runbooks linked from the alert itself.

**Operations**
- Stay on a supported version; upgrade on a schedule.
- Namespaces with ResourceQuotas and LimitRanges per team.
- Managed control plane and managed add-ons where available — less to operate, fewer failure modes.
- Blameless postmortems, and feed every incident back into automation.

If I had to compress it: **declare everything in Git, right-size resources, probe honestly, shut down gracefully, and alert on user-visible symptoms.** Most Kubernetes incidents I've seen trace back to one of those five being neglected.

---

## Amazon EKS

### What is Amazon EKS?

AWS's managed Kubernetes service. AWS runs the control plane — API server, etcd, scheduler, controller manager — across multiple AZs, handles its patching, scaling, and backups, and gives you an endpoint. You bring the worker capacity (managed node groups, self-managed nodes, Fargate, or Karpenter-provisioned instances) and your workloads.

It's upstream-conformant Kubernetes, so manifests are portable. What's specific to EKS is the *integration* layer: IAM for authentication and workload identity, VPC CNI for native Pod networking, ALB/NLB controllers, EBS/EFS CSI drivers, and CloudWatch for logs and metrics.

### EKS vs self-managed Kubernetes?

| | EKS | Self-managed |
|---|---|---|
| Control plane | AWS operates, multi-AZ HA, backed up | You operate, you back up etcd |
| Upgrades | You trigger; AWS performs | Fully yours, including etcd and PKI |
| Cost | Per-cluster hourly fee + nodes | Nodes only, but engineer time is the real cost |
| Customisation | Limited flags/feature gates | Full control of every component |
| AWS integration | Native (IAM, VPC, ELB, CSI) | You wire it up |
| Certs, etcd, PKI | Not your problem | Definitely your problem |

My take for an interview: choose EKS unless you have a concrete requirement it can't meet — unusual control-plane flags, an unsupported version, air-gapped or on-prem, or genuine scale economics. The operational burden you avoid (etcd backups, control-plane HA, cert rotation, API server tuning) is exactly the work that causes 3am pages and adds no product value.

### How do managed node groups work?

A managed node group is an EC2 Auto Scaling group that EKS provisions and lifecycle-manages for you, using EKS-optimised AMIs. You specify instance types, size bounds, subnets, and a node IAM role; EKS handles bootstrapping (joining the cluster), tagging for autoscaling discovery, and health checks.

The valuable part operationally is the **managed upgrade**: when you update the node group's Kubernetes version or AMI, EKS performs a rolling replacement that **cordons, drains respecting PodDisruptionBudgets**, launches replacements, and only then terminates old nodes. That's the workflow you'd otherwise script yourself and get subtly wrong.

Practical notes: use **launch templates** for custom AMIs, user data, or volume settings; managed node groups support **Spot** with capacity-optimized allocation and automatic interruption draining; and one node group per distinct shape or purpose (GPU, ARM, spot vs on-demand) with labels and taints to steer workloads. Compared to Karpenter, node groups are simpler and more predictable but less efficient at bin-packing — many teams now use Karpenter for the bulk of capacity and a small managed group for system add-ons.

### What is Fargate in EKS?

Serverless compute for Pods — no EC2 instances to manage. You define a **Fargate profile** (namespace and optional label selectors); matching Pods get scheduled onto AWS-managed capacity, and you pay per-Pod for the vCPU and memory it requests, per second.

The trade-offs, which is what interviewers are actually asking about:

- **One Pod per "node"** — full isolation, no noisy neighbours, but no bin-packing benefit, and resources are rounded up to fixed CPU/memory combinations.
- **No DaemonSets** — so log and metric collection must be sidecars, which changes your observability setup.
- **No privileged containers, no host networking, no host paths.**
- **EFS only** for persistent volumes; no EBS.
- **Slower Pod start** (tens of seconds) since capacity is provisioned per Pod.
- Generally **more expensive per unit of compute** than well-packed EC2, though cheaper than badly-packed EC2 or an idle node group.

Good fits: bursty or infrequent workloads, batch jobs, small clusters where managing nodes isn't worth it, and untrusted or multi-tenant workloads that benefit from hard isolation. Poor fits: high-density steady-state services, anything needing DaemonSets or EBS, latency-sensitive scale-out.

### What is IAM Roles for Service Accounts (IRSA)?

The mechanism that gives a **Pod** its own AWS IAM role, instead of every Pod inheriting the node's role.

How it works: the cluster has an **OIDC identity provider**. You annotate a ServiceAccount with a role ARN; the Pod gets a projected, short-lived OIDC token mounted at a known path; the AWS SDK calls `sts:AssumeRoleWithWebIdentity` with that token; STS validates it against the cluster's OIDC provider and the role's trust policy, and returns temporary credentials.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-reader-role
```

Why it matters: without IRSA, you either bake access keys into images (bad) or grant the *node* role every permission any Pod on it might need — which means every Pod on that node inherits all of them. IRSA gives per-workload least privilege with automatically-rotating short-lived credentials.

Worth mentioning the successor: **EKS Pod Identity** does the same thing with simpler setup — an association via the EKS API instead of managing OIDC trust policies per role, and no reliance on the OIDC provider URL. Easier to manage across many clusters. IRSA is still widely used and still the thing you'll be asked about.

### How does the AWS Load Balancer Controller work?

It's a controller running in the cluster that watches Kubernetes objects and provisions real AWS load balancers to match:

- An **Ingress** → provisions an **ALB** (L7), with rules mapping hosts/paths to target groups.
- A **Service of type LoadBalancer** with the right annotations → provisions an **NLB** (L4).

Target modes matter:

- **`ip` mode** (recommended, and required for Fargate) — registers **Pod IPs** directly in the target group. Traffic goes ALB → Pod, skipping the node and kube-proxy hop. Fewer hops, better health checks, no second load-balancing step.
- **`instance` mode** — registers nodes on the Service's NodePort; kube-proxy then forwards to a Pod, possibly on another node.

Requirements: IRSA or Pod Identity giving it permission to manage ELBs, security groups, and target groups; and correct subnet tags (`kubernetes.io/role/elb` for public, `internal-elb` for private) so it knows where to place the LB.

Useful annotations to know: `alb.ingress.kubernetes.io/scheme` (internet-facing vs internal), `certificate-arn` for ACM TLS, `target-type: ip`, `healthcheck-path`, and `group.name` — which lets **multiple Ingresses share one ALB**, a meaningful cost saving.

### How do you expose an application in EKS?

Depends on the traffic type, and the cost conversation is part of the answer:

1. **Internal only** → `ClusterIP`. Nothing external needed.
2. **HTTP/HTTPS externally** → **Ingress + ALB** via the AWS Load Balancer Controller. TLS from ACM, path/host routing, WAF attachable. Use `group.name` to share one ALB across many Ingresses instead of paying for one per app.
3. **Raw TCP/UDP, or extreme throughput** → `Service type=LoadBalancer` with NLB annotations. Static IPs, preserves source IP, very high performance.
4. **Global, with CDN and edge caching** → **CloudFront** in front of the ALB, with WAF at the edge.
5. **gRPC or long-lived connections** → ALB supports gRPC and HTTP/2; NLB works too, but note that L4 load balancing plus long-lived connections gives you poor distribution — you may need client-side balancing or a headless Service.
6. **Internal-only but cross-VPC/account** → internal ALB/NLB plus **VPC Lattice** or PrivateLink.

Whichever you pick, the same fundamentals apply: readiness probes so only healthy Pods are targeted, `target-type: ip` to cut a hop, health check path matching your actual health endpoint, and DNS via Route 53 (ExternalDNS can manage records automatically from Ingress annotations).

### How do EBS and EFS integrate with EKS?

Both via **CSI drivers**, which on modern EKS you install as **EKS add-ons** — and importantly, **neither is installed by default**, which is why a fresh cluster leaves PVCs Pending.

**EBS CSI driver** — block storage:
- `ReadWriteOnce` only: one node at a time. Not shareable across Pods on different nodes.
- **AZ-bound.** A volume in `us-east-1a` can only attach to a node in `us-east-1a`. This is the single most important EBS-on-Kubernetes fact, and it's why you use `volumeBindingMode: WaitForFirstConsumer` and why StatefulSet Pods are effectively pinned to an AZ.
- Use `gp3` (better price/performance than gp2, and IOPS/throughput configurable independently of size), with `encrypted: true` and a KMS key.
- Supports volume expansion and snapshots (via the external snapshotter, which is what Velero uses for PV backups).
- Needs IRSA/Pod Identity permissions for `ec2:CreateVolume`, `AttachVolume`, `CreateSnapshot`, etc.

**EFS CSI driver** — NFS-based shared filesystem:
- `ReadWriteMany` — many Pods across many nodes and AZs can mount it simultaneously. This is the answer whenever "shared storage" or "RWX" comes up.
- Not AZ-bound, so it doesn't pin Pods, which makes it much friendlier for rescheduling.
- Higher latency and higher cost per GB than EBS; poor fit for databases or anything IOPS-sensitive.
- Uses **access points** for per-workload directory isolation and POSIX permissions.
- Security groups must allow NFS (2049) from the nodes — the classic reason an EFS mount hangs.

Rule of thumb: **EBS for single-writer, performance-sensitive state (databases). EFS for shared read/write across Pods (uploads, shared config, legacy apps expecting a shared filesystem). S3 for anything that can be object storage** — which is usually the cheapest and most scalable answer if the app can be adapted.

### How do you upgrade an EKS cluster?

The order is control plane → add-ons → nodes, one minor version at a time.

1. **Pre-flight checks**
   - Read the EKS release notes and the Kubernetes deprecation guide for the target version.
   - Scan for removed APIs (`kubent`, `pluto`) — EKS will also surface **upgrade insights** in the console/API flagging deprecated API usage it has observed. Use them.
   - Verify add-on compatibility: VPC CNI, CoreDNS, kube-proxy, EBS/EFS CSI, ALB controller, cert-manager, Karpenter, metrics-server.
   - Confirm **subnet IP capacity** — upgrades create new nodes alongside old ones, and a nearly-full subnet will stall the whole thing. Underrated failure mode.
   - Check PDBs are sane: a PDB that can never be satisfied will block node draining indefinitely.
   - Practise the whole thing in a lower environment first.

2. **Upgrade the control plane** — console, `eksctl upgrade cluster`, or Terraform. AWS does it in place with no downtime for running workloads; takes 10–30 minutes.

3. **Upgrade managed add-ons** to versions matching the new control plane. Do this after the control plane, before the nodes.

4. **Upgrade nodes** — bump the managed node group version (rolling, PDB-respecting) or, with Karpenter, update the AMI/nodepool and let drift-based replacement roll them. Do it in batches; watch application health between batches.

5. **Verify** — nodes Ready, all Deployments at desired replicas, smoke tests pass, error rate and latency unchanged. Keep watching for a day, not five minutes.

Two things to be explicit about: **you cannot downgrade an EKS control plane.** Rollback means restoring from IaC into a new cluster, so the pre-flight work is the actual safety mechanism. And EKS auto-upgrades clusters approaching end of standard support — so a cluster you neglect will eventually be upgraded *for* you, on AWS's schedule rather than yours.

### What are the common issues you've faced in EKS and how did you resolve them?

The ones that come up most, and what they taught me:

**1. Pod IP exhaustion with the VPC CNI.** Pods get real VPC IPs, so a `/24` subnet runs out fast and new Pods sit Pending with no obvious "out of memory" signal. Fixes: larger or additional subnets, **prefix delegation** (`ENABLE_PREFIX_DELEGATION=true`) to assign /28 prefixes instead of individual IPs, or custom networking with a secondary CIDR. The lesson: **plan Pod CIDR capacity before the cluster exists**, because retrofitting is painful.

**2. Image pulls failing in private subnets.** The pull times out even though ECR permissions look right — because ECR layers come from **S3**, so you need an S3 gateway endpoint (or NAT) in addition to the ECR interface endpoints. Adding the S3 endpoint fixed it and cut NAT data-processing cost as a bonus.

**3. PVCs stuck Pending on a fresh cluster.** No EBS CSI driver and no default StorageClass — EKS doesn't ship one. Install the add-on with proper IRSA permissions and mark a `gp3` class default.

**4. AZ-bound EBS blocking rescheduling.** A StatefulSet Pod wouldn't come back after a node failure because the only spare capacity was in another AZ. `WaitForFirstConsumer` prevents the provisioning half; the structural fix is node capacity in every AZ, or EFS/managed data stores where the workload allows.

**5. IRSA not taking effect.** Pod still using the node role. Causes, in order of frequency: the ServiceAccount annotation was wrong or on the wrong SA, the Pod wasn't restarted after the annotation, the role's trust policy had the wrong OIDC provider or `sub` condition, or an old AWS SDK that doesn't support web identity. `aws sts get-caller-identity` from inside the Pod settles it immediately.

**6. NetworkPolicies silently ignored.** Written, applied, accepted — and doing nothing, because VPC CNI network policy support wasn't enabled. A false sense of security is worse than no policy, so I now verify enforcement with an actual connectivity test rather than trusting that the object exists.

**7. Load balancer not provisioning.** Missing subnet tags (`kubernetes.io/role/elb`), or the ALB controller lacking IAM permissions. The controller logs say exactly which API call was denied — always read them before theorising.

**8. Cluster Autoscaler / Karpenter not scaling.** At max size, no node group matching the Pod's shape (e.g. a GPU or large-memory request), the ASG missing discovery tags, or EC2 insufficient-capacity errors in one AZ. The autoscaler logs are explicit; the mistake is not reading them.

**9. CoreDNS throttled under load** causing intermittent timeouts across the whole cluster. Scaled it up, set proper requests, and added NodeLocal DNSCache. The general lesson: **shared add-ons have cluster-wide blast radius**, so they get resourced and monitored like production services, not treated as background infrastructure.

**10. Cross-AZ data transfer cost creeping up** because Services load-balanced randomly across zones. Topology-aware routing kept traffic in-zone and cut both cost and latency.

---

*Good luck. If you can explain the reasoning behind an answer rather than reciting it, you're already ahead of most candidates.*
