# SRE Fundamentals, SLOs, Observability & Incident Management — Interview Answer Bank

Companion to the Kubernetes/EKS answer bank. Same format: answers written the way you'd **say them**, with the follow-up depth interviewers actually probe.

**How to use this:**
- The first paragraph of each answer is your spoken answer. Everything after is follow-up ammunition.
- **Boxed numbers matter.** Availability arithmetic, burn rates, and headroom math get asked directly and candidates fumble them. Learn those cold.
- §17 has STAR skeletons — fill in your own numbers *before* the interview, not during.

**The meta-skill being graded:** can you explain *why* a practice exists, not just what it's called? "Error budgets convert a political argument into a pre-agreed policy" beats "an error budget is one minus the SLO" every time.

---

## Table of Contents

1. [SRE Fundamentals](#1-sre-fundamentals)
2. [SLI, SLO, SLA & Error Budgets](#2-sli-slo-sla--error-budgets)
3. [Monitoring & Observability](#3-monitoring--observability)
4. [Incident Management](#4-incident-management)
5. [Postmortem & RCA](#5-postmortem--rca)
6. [Toil & Automation](#6-toil--automation)
7. [Availability & Reliability](#7-availability--reliability)
8. [Production Scenarios](#8-production-scenarios)
9. [Deployment & Release Engineering](#9-deployment--release-engineering)
10. [Capacity Planning](#10-capacity-planning)
11. [Performance Engineering](#11-performance-engineering)
12. [Kubernetes & Cloud (SRE Lens)](#12-kubernetes--cloud-sre-lens)
13. [Security & Reliability](#13-security--reliability)
14. [Networking & Distributed Systems](#14-networking--distributed-systems)
15. [System Design](#15-system-design)
16. [On-Call & Operational Excellence](#16-on-call--operational-excellence)
17. [Behavioral — STAR Skeletons](#17-behavioral--star-skeletons)
18. [Senior / Advanced](#18-senior--advanced)
19. [Questions to Ask Them](#19-questions-to-ask-them)

---

## 1. SRE Fundamentals

### What is SRE?

A discipline that applies software engineering practice to operations problems — treating reliability as an engineering problem with measurements, targets, and automation, rather than manual firefighting. It originated at Google and Ben Treynor's framing is still the crispest: "SRE is what happens when you ask a software engineer to design an operations function."

The concrete difference from traditional ops: reliability gets a **number** (the SLO), a **budget** for failure, and an explicit **policy** for what happens when the budget runs out. That turns reliability from a matter of opinion into a matter of data.

### SRE vs DevOps?

DevOps is a cultural philosophy — break down dev/ops silos, shared ownership, automate delivery, measure everything. It tells you *what* to aim for but not *how*.

SRE is a specific, prescriptive implementation of it. Where DevOps says "reduce organisational silos," SRE says "share on-call and use error budgets as the shared contract." Where DevOps says "measure everything," SRE says "define SLIs, set SLOs, alert on burn rate." Where DevOps says "accept failure as normal," SRE says "here's the blameless postmortem template."

The one-liner: **DevOps is the interface, SRE is one implementation of it.**

### What are the core responsibilities of an SRE?

- Define, measure, and report on SLIs/SLOs; own the error budget policy.
- Build and maintain observability — metrics, logs, traces, dashboards, alerts.
- Automate toil out of existence; build self-service tooling for dev teams.
- On-call incident response, and postmortems that produce real fixes.
- Capacity planning and performance engineering.
- Production readiness reviews — gate what gets to run in production and on what terms.
- Partner with dev teams on reliability trade-offs, using the error budget as the negotiating instrument.

### Why do organisations need SREs?

Because feature velocity wins by default. Shipping features is visible and rewarded; preventing an outage that never happens is invisible. Without a discipline that owns reliability with its own metrics and its own budget, reliability loses every prioritisation argument until an incident forces the issue — and then you overcorrect.

SRE formalises the counterweight so the trade-off is made deliberately and in advance, not emotionally and after the fact.

### What are the key principles of SRE?

1. **Embrace risk** — 100% reliability is the wrong target. It's unachievable, and chasing it costs more than the marginal reliability is worth to users.
2. **SLOs and error budgets** as the shared contract between dev and ops.
3. **Eliminate toil** — cap it (Google's convention is under 50% of time) and automate the rest.
4. **Monitor meaningfully** — alert on symptoms users feel, not on causes.
5. **Automate, with guardrails.**
6. **Release engineering as a discipline** — repeatable, gradual, reversible.
7. **Simplicity** — the underrated one. Complexity is the source of most unreliability, and SREs are the team that pushes back on it.
8. **Blameless postmortems** — fix systems, not people.

### What makes a service reliable?

It does what users need, when they need it, within acceptable latency and correctness bounds — measured against an agreed SLO rather than a vibe. "Stable" isn't a property; a 99.9% availability SLO measured on the checkout path is.

Worth adding: reliability is **user-perceived**, so it's measured as close to the user as you can get. A service can be 100% healthy by internal metrics while every user sees timeouts because the CDN or DNS layer is broken.

### How do you measure reliability?

SLIs tracked against SLO targets over a defined window, with an error budget quantifying the allowance.

Concretely, an SLI is expressed as a ratio of good events to valid events:

```
availability SLI = successful requests / total valid requests
latency SLI      = requests served under 300ms / total valid requests
```

Note that latency SLIs are usually expressed as a **ratio, not a percentile** — "99% of requests under 300ms" rather than "p99 latency is 300ms." Same information, but the ratio form composes into an error budget and the percentile form doesn't.

### What is "keeping the lights on"?

The baseline operational work required just to keep existing systems running — patching, alert response, manual runbook execution, capacity firefighting. It's a synonym for toil at the organisational level.

The SRE goal isn't to do it well; it's to shrink it, so engineering time goes to work that compounds. If your KTLO load grows linearly with the number of services, you don't have an SRE practice, you have an ops team with a new title.

### What are the responsibilities of an on-call SRE?

Acknowledge within the agreed SLA, assess severity, **mitigate before diagnosing**, escalate early rather than heroically, document actions in real time, and hand off cleanly with written context.

The instinct that matters most: restoring service and understanding the cause are two different jobs, and the first one comes first. Rolling back an unclear failure at 3am is correct; debugging it live while users are down is not.

---

## 2. SLI, SLO, SLA & Error Budgets

### SLI, SLO, SLA — what's the difference?

- **SLI (Indicator)** — the measurement. "99.94% of requests succeeded over the last 30 days."
- **SLO (Objective)** — your internal target for that measurement. "99.9% of requests succeed over a rolling 30 days." This is what your team holds itself to and what drives engineering priority.
- **SLA (Agreement)** — the external contractual promise, with financial or legal consequences. "99.5% monthly uptime or you get service credits."

The chain: SLI is measured → SLO is the internal bar → SLA is a looser external promise built on top.

**Why the SLA is looser** is the follow-up: you want the internal alarm to go off well before you owe anyone money. If SLO and SLA are the same number, you find out you've breached your contract at the same moment you find out you have a problem. The gap is deliberate margin.

Also worth noting: SLAs are usually measured more coarsely (monthly calendar windows, sometimes uptime rather than request success) precisely because they need to be legally unambiguous.

### **How much downtime does each availability target allow?**

Learn this table. It gets asked directly, and the answer to "should we target four nines?" is usually contained in it.

| SLO | Per month (30d) | Per quarter | Per year |
|---|---|---|---|
| 99% | 7h 12m | 21h 36m | 3d 15h |
| 99.5% | 3h 36m | 10h 48m | 1d 20h |
| 99.9% ("three nines") | **43m 12s** | 2h 10m | **8h 46m** |
| 99.95% | 21m 36m | 1h 5m | 4h 23m |
| 99.99% ("four nines") | **4m 19s** | 13m | **52m 36s** |
| 99.999% ("five nines") | 26s | 1m 18s | 5m 15s |

The judgement this unlocks: **99.99% means your total annual budget is under an hour**, which is less time than a single careless deploy or one node replacement gone wrong. If you can't detect and mitigate in under ten minutes, four nines is aspirational marketing, not an achievable SLO. That reasoning is what interviewers are looking for.

Also: your SLO can't meaningfully exceed the reliability of your dependencies. Building a 99.99% service on a 99.9% database means the database alone can consume 10x your budget.

### How do you define a good SLO?

1. **Start from a critical user journey**, not a service. "User can complete checkout," not "payments-api is up."
2. **Measure as close to the user as possible** — at the load balancer or client, not deep inside the service where you'll miss whole classes of failure.
3. **Set the target from observed reality plus intent**, not aspiration. Measure current performance for a few weeks first; an SLO you're already violating daily teaches the team to ignore SLOs.
4. **Leave room to ship.** The target must permit a normal rate of change. 100% leaves zero budget for any deploy.
5. **Keep it small** — a handful of SLOs per service. Twenty SLOs means nobody looks at any of them.
6. **Agree the error budget policy at the same time.** An SLO with no consequence attached is a dashboard, not a contract.

### How do you choose the right SLIs?

Pick what users would complain about, then find the closest measurable proxy. The standard menu:

- **Availability** — success rate of requests.
- **Latency** — proportion served under a threshold.
- **Correctness / quality** — right answer, not just a 200.
- **Freshness** — for pipelines and caches: how stale is the data users see.
- **Throughput/coverage** — for batch: proportion of records processed.

Measurement point matters enormously. Server-side metrics miss DNS failures, TLS problems, CDN errors, and client-side breakage — everything between your load balancer and the user's eyeballs. Where feasible, corroborate with RUM or synthetic checks from outside your network.

### Request-based vs windowed (time-based) SLIs?

Two ways to compute the same idea, and they give different answers:

- **Request-based**: `good requests / valid requests`. Precise, weights by traffic, but a low-traffic outage barely registers.
- **Windowed / time-based**: divide the period into minutes, mark each minute good or bad, then `good minutes / total minutes`. Every bad minute counts equally regardless of traffic — which is why SLAs and status pages use it.

The trap: under a windowed SLI, a 2am outage affecting 12 users costs the same budget as a 2pm outage affecting 120,000. Under a request-based SLI, the opposite distortion. Mature setups often run both.

### Rolling vs calendar windows?

- **Rolling 30 days** — always current, no artificial reset, smoother. Best for engineering decisions.
- **Calendar month** — aligns with business reporting and SLAs. But it creates a perverse incentive: burn the budget on the 28th and you get a clean slate on the 1st, which encourages exactly the wrong end-of-month behaviour.

Standard practice: rolling windows for internal SLOs, calendar for the contractual SLA.

### What is an error budget?

`1 − SLO`, over the window. It's the quantified amount of unreliability you're permitted before reliability work takes priority over features.

**Worked example, because this gets asked:**

```
SLO:            99.9% success over 30 days
Traffic:        100,000,000 requests / 30 days
Error budget:   0.1% × 100M = 100,000 failed requests

A 20-minute total outage at average traffic (~2,300 rps)
  = 2,300 × 1,200s ≈ 2.76M failures
  = 27x your entire monthly budget, in 20 minutes.
```

That calculation is the most persuasive artefact in an SRE's toolkit. It reframes "we should probably improve failover" as "one bad twenty-minute incident costs us two years of budget."

### Why do error budgets matter?

They convert an unwinnable argument into arithmetic. Without one, "is this change too risky?" is settled by whoever is more senior or more insistent, every single time, forever.

With one, both sides agree the policy *in advance*, while calm: budget remaining → ship freely, SRE doesn't get to veto. Budget exhausted → feature work pauses for reliability, and dev doesn't get to override. Neither side is exercising authority in the moment; they're both following a rule they wrote together.

The subtle point that lands well in interviews: an error budget is as much a **licence to take risk** as a brake. It tells developers exactly how much they're allowed to break, which is more freedom than a vague "don't cause incidents."

### How do dev and SRE teams actually use error budgets?

As a gate on release velocity, plus a prioritisation signal:

- Budget healthy → normal velocity, experiments allowed, risky migrations scheduled.
- Budget burning fast → tighten canary analysis, slow rollouts, require extra review.
- Budget exhausted → freeze non-critical releases, redirect to reliability work until it recovers.

Also useful in reverse: consistently *unspent* budget means the SLO is too loose or you're over-investing in reliability relative to what users need. That's a signal to either tighten the target or move engineers to features.

### What do you do when the error budget is exhausted?

Follow the pre-agreed policy — that's the whole point, it isn't a judgement call made under stress:

1. Freeze non-critical releases (reliability fixes and security patches still ship).
2. Redirect engineering to the top contributors to the burn, identified from incident data.
3. Communicate the freeze and its exit criteria to stakeholders clearly.
4. Run a review: was this one bad incident, or a trend? A single outage that ate the budget is a different problem from steady erosion.
5. Exit when the rolling window recovers — and if it never does, the SLO or the architecture is wrong, not the team.

The escape valve worth mentioning: some incidents are legitimately excludable (a dependency's regional outage that you'd documented as out of scope), but the exclusion rules must be written *before* the incident, or the budget becomes meaningless.

### Can one application have multiple SLIs?

Yes, and it should. Typically availability + latency at minimum, and often per **critical user journey** rather than per service.

The reason to split by journey: a single blended SLI hides asymmetric failure. If login is 5% of traffic and completely broken while browsing is 95% and fine, your aggregate SLI reads 95% — which looks like mild degradation, not "nobody can log in." Separate SLOs on separate journeys catch that.

### How do you monitor SLO compliance?

A dashboard showing the SLI against target, budget consumed, budget remaining, and **burn rate** — how fast you're spending relative to the window. Burn rate is the actionable number: 1x means you'll exactly exhaust the budget at the end of the window; 10x means you'll exhaust it in a tenth of the time.

Alert on burn rate, not on the SLO being breached — by the time it's breached, it's too late to be useful.

### **How does burn-rate alerting work?**

This is a strong differentiator, because most candidates only know static thresholds. The multi-window, multi-burn-rate approach from the SRE Workbook, for a 30-day window:

| Burn rate | Long window | Short window | Budget consumed | Action |
|---|---|---|---|---|
| **14.4x** | 1 hour | 5 min | 2% | **Page** |
| **6x** | 6 hours | 30 min | 5% | **Page** |
| **3x** | 1 day | 2 hours | 10% | Ticket |
| **1x** | 3 days | 6 hours | 10% | Ticket |

Two things to explain:

- **Why two windows per rule?** The long window detects the burn; the short window (1/12th the length) confirms it's *still happening*. Without it, a resolved 2am spike keeps paging you at 8am. The alert clears fast once the burn stops.
- **Why several rates?** A fast burn is an outage and needs a human now. A slow burn is a chronic bug that needs a ticket, not a page. One static threshold can't distinguish those, which is exactly why static thresholds generate so much noise.

The practical payoff: this is *the* answer to "how would you reduce alert noise?" — you've replaced dozens of cause-based threshold alerts with four symptom-based rules tied directly to user impact.

### Which metrics should NOT be used as SLIs?

Anything disconnected from user experience:

- **CPU, memory, disk utilisation** — a node at 95% CPU serving every request under 100ms has no user-facing problem. These are *diagnostic* signals, not objectives.
- **Pod restart counts, queue depth, replica counts** — internal states. Useful for debugging, terrible as targets.
- **Averages of anything** — mean latency hides the tail where the unhappy users live.
- **Raw uptime of a process** — a running process serving 500s is "up."

The general test: if the metric can go bad while users are perfectly happy, or stay good while users are furious, it's not an SLI. Alert on symptoms, diagnose with causes.

---

## 3. Monitoring & Observability

### Monitoring vs observability?

**Monitoring** watches known signals against known thresholds and tells you when a predefined condition is met. It answers questions you knew to ask: "is error rate above 1%?"

**Observability** is the property of being able to infer a system's internal state from its outputs well enough to answer questions you *didn't* anticipate: "why are requests from one tenant, on one API version, slow only when they hit the EU region?"

The relationship: monitoring is what you build on top of an observable system. You can have beautiful dashboards and poor observability — every dashboard answers a question someone already thought of. The test of observability is how you fare against a novel failure mode with no pre-built dashboard.

The practical difference in tooling: monitoring favours pre-aggregated metrics; observability favours high-cardinality, high-dimensionality data you can slice arbitrarily after the fact.

### What are the three pillars of observability?

- **Metrics** — aggregated numeric time series. Cheap, fast, ideal for alerting and trends. Weak at explaining a specific request.
- **Logs** — discrete event records. Rich detail, expensive at volume, hard to aggregate.
- **Traces** — the path of one request across services, with timing per span. The only signal that shows you *where* latency lives in a distributed call chain.

**Worth knowing the pushback**, since observability-mature teams probe it: "three pillars" is now considered a limiting frame. The critique is that three separate silos force you to manually correlate across three tools, which is exactly the work you wanted automated. The alternatives offered are **wide structured events** (one richly-dimensioned event per request from which metrics and traces can be derived), and treating **profiles** as a fourth signal. In practice, **OpenTelemetry** is the answer that matters: one vendor-neutral instrumentation standard emitting all signals with shared context, so correlation is automatic rather than manual.

Also mention **exemplars** — linking a metric data point directly to a trace ID that exemplifies it. That's the bridge that makes "p99 is bad" immediately actionable.

### **Why are averages dangerous, and how should you use percentiles?**

Missing from most candidates' answers and asked frequently.

An average hides the tail. If 99 requests take 10ms and one takes 5 seconds, the mean is 60ms — which looks great, and one user just had an awful experience. Averages describe a distribution only if it's normal, and latency distributions never are: they're long-tailed and often multi-modal (cache hit vs cache miss).

So use percentiles — and know what they mean at your scale:

- **p50** — the typical experience.
- **p95 / p99** — the unhappy tail. At 10,000 rps, p99 means **100 requests per second are worse than this**.
- **p99.9** — matters for anything fanning out: if one page makes 20 backend calls, roughly 2% of page loads hit a p99.9 backend response. Tail latency at the component level becomes typical latency at the page level.

Three technical points that mark you as having done this for real:

1. **You cannot average percentiles.** Averaging the p99 of ten servers does not give you the fleet p99. You need histograms and quantile estimation over the merged distribution — `histogram_quantile()` over summed buckets in Prometheus, not `avg(p99)`.
2. **Percentiles from pre-aggregated per-instance summaries are wrong** for the same reason. Prefer histograms.
3. **Coordinated omission** — load generators that wait for a response before issuing the next request systematically under-report the tail, because during a stall they simply stop measuring. Gil Tene's point; it means a lot of benchmark numbers are optimistic fiction.

### What metrics would you monitor for a web application?

- **Rate** — requests/sec, by endpoint and status class.
- **Errors** — 5xx rate, 4xx rate separately (4xx often means a client/contract problem, not yours), and business-level failures that return 200.
- **Duration** — p50/p95/p99, per endpoint. Aggregate latency across all endpoints is nearly useless.
- **Saturation** — CPU, memory, connection pool utilisation, thread pool queue depth, GC time.
- **Dependencies** — latency and error rate per downstream call, plus circuit breaker state.
- **Business signals** — checkout completion, signup completion, payment success. These catch failures that every technical metric misses: a deploy that returns 200 with an empty cart is invisible to your error rate.

### What Kubernetes metrics matter?

Covered in depth in the Kubernetes answer bank, but the SRE-lens summary: **Pod restarts and OOMKills, Pods Pending beyond a threshold, replicas available vs desired, container CPU throttling, node conditions, API server latency, etcd fsync latency, CoreDNS latency.**

The framing to add: alert on almost none of these. They're diagnostic. You page on the service SLO burning; you then look at this list to find out why.

### What database metrics matter?

Query latency (percentiles, and per query pattern), connections in use vs `max_connections`, replication lag, lock waits and deadlocks, cache/buffer hit ratio, slow query count, disk IOPS and queue depth against the provisioned limit, transaction rollback rate, and table/index bloat for Postgres.

The two that cause the most surprise outages: **connection exhaustion** (because every app replica has its own pool, so scaling out multiplies connections) and **hitting the provisioned IOPS ceiling**, where latency degrades non-linearly with no CPU signal at all.

### What AWS infrastructure metrics matter?

Per service, the ones that actually predict incidents:

- **EC2** — CPU, `StatusCheckFailed`, network throughput against the instance's baseline, and **CPU credit balance** on burstable T-family instances (a silently exhausted credit balance looks exactly like an application performance regression).
- **RDS** — CPU, `DatabaseConnections` vs max, `FreeableMemory`, `ReadLatency`/`WriteLatency`, `ReplicaLag`, burst balance and IOPS against provisioned.
- **ALB** — `HTTPCode_ELB_5XX` vs `HTTPCode_Target_5XX` (crucial distinction: the LB failing vs your app failing), `TargetResponseTime`, `UnHealthyHostCount`, `RejectedConnectionCount`, and `SurgeQueueLength`/`ActiveConnectionCount`.
- **ECS/Fargate** — running vs desired task count, CPU/memory utilisation, task stopped reasons.
- **SQS** — `ApproximateAgeOfOldestMessage` (the best single queue health signal), queue depth, DLQ depth.
- **Lambda** — throttles, concurrent executions vs limit, duration against timeout, error rate.
- **Cross-cutting** — NAT Gateway `ErrorPortAllocation`, service quota utilisation, and CloudWatch billing/anomaly alarms.

### What is RED monitoring?

**Rate, Errors, Duration** — Tom Wilkie's method, aimed at **request-driven services**. For every service: how many requests, how many failed, how long did they take. Three metrics per service, uniform across the fleet, which makes it easy to standardise dashboards.

### What is USE monitoring?

**Utilisation, Saturation, Errors** — Brendan Gregg's method, aimed at **resources** rather than services. For every resource (CPU, disk, network, memory bus): how busy is it, how much queued work is waiting, and is it throwing errors.

The distinction from RED worth stating: utilisation tells you how full something is, **saturation tells you what's waiting** — and saturation is the leading indicator. A disk at 100% utilisation with no queue is fine; at 80% with a growing queue you're about to have a latency incident.

### What are the golden signals?

Google's four: **latency, traffic, errors, saturation.** The minimum viable signal set for any service.

How the three methods relate: RED is golden signals minus saturation, specialised for services. USE is resource-focused. Golden signals span both. If asked which to use — RED (or golden signals) for services, USE for the infrastructure underneath, and don't over-think it; the point is uniform coverage, not method purity.

### **What is metric cardinality and why does it matter?**

The number of unique time series, which is the product of all label value combinations. A metric with 3 endpoints × 5 status codes × 10 regions = 150 series. Add `user_id` as a label with a million users and you have a hundred and fifty million.

Why it's the single most important practical observability constraint:

- Cost and memory scale with cardinality, not query volume. Cardinality explosions are the standard way teams take down their own Prometheus.
- The usual culprits are unbounded label values: user IDs, request IDs, trace IDs, email addresses, full URL paths with IDs embedded, error messages as labels.

The rules: keep label values from a **bounded** set, template dynamic path segments (`/users/{id}`, never `/users/12345`), put high-cardinality identifiers in **logs and traces** where they belong, use exemplars to bridge metric → trace, and enforce limits at the collector.

This also happens to be the answer to "how would you control observability cost?", which is an increasingly common question because observability bills now rival compute.

### How do you reduce alert fatigue?

1. **Alert on symptoms, not causes.** One SLO burn-rate alert replaces twenty threshold alerts on CPU, memory, queue depth, and replica counts.
2. **Use burn-rate alerting** (see §2) so severity maps to actual budget impact.
3. **Every page must be actionable.** If the runbook's first step is "check whether this matters," delete the alert or make it a ticket.
4. **Split page vs ticket vs dashboard-only.** Most alerts belong in the second two categories, and most teams page on all three.
5. **Group and deduplicate** so one root cause produces one notification.
6. **Add inhibition rules** — don't page on twelve dependent services when the shared database is already paging.
7. **Review the data.** Track alerts per shift and the proportion that led to action. Delete anything with a low action rate; a monthly alert review is one of the highest-leverage rituals a team can adopt.
8. **Tune for flapping** with `for:` durations, so a 30-second blip doesn't wake anyone.

### What makes an alert actionable?

There is a specific human action, needed now, that the alert makes clear. Concretely, an actionable alert has: a symptom-based condition tied to user impact, a severity that determines the response, a linked runbook, a linked dashboard, and enough context in the payload to start work without hunting.

The falsification test: **if the standard response is "acknowledge and see if it clears," it is not actionable.** That's a metric, not an alert.

### How do you prioritise alerts?

By user impact and urgency, not arrival order. A P1 revenue-affecting outage outranks a P3 degradation regardless of which fired first — and if you're handling both, the correct move is usually to escalate for a second responder rather than split attention, then say out loud which one you've deprioritised and why.

The structural version of this answer: severity should be *encoded in the alert definition* so this prioritisation isn't improvised at 3am by a sleepy human.

### What are noisy alerts, and why do they matter so much?

Alerts that fire often without corresponding to actionable problems. They matter more than they look because they cause two compounding harms:

1. **Burnout** — the direct cost, and the reason people leave on-call rotations.
2. **Desensitisation** — the dangerous cost. Teams learn to reflexively acknowledge and ignore, and then they ignore the one page that mattered. Every real incident postmortem that contains "the alert did fire, but we'd been seeing it all week" is this failure mode.

Treat noisy alerts as production bugs with a real cost, not as an annoyance to tolerate.

### What is alert correlation?

Grouping and deduplicating alerts that share a root cause into a single incident. One node failing shouldn't produce twenty Pod alerts, five Service alerts, and a latency alert as twenty-six separate pages.

Mechanisms: grouping by shared labels (Alertmanager `group_by`), inhibition rules (suppress dependents when the cause is already firing), dependency-aware suppression using a service graph, and time-window clustering.

The deeper fix, though, is upstream: if you're alerting on symptoms at the user-journey level rather than on every component, you generate far fewer alerts to correlate in the first place.

### What is synthetic monitoring?

Scripted transactions run on a schedule from outside your system — uptime checks, login flows, checkout flows, API probes from multiple geographic locations.

Its unique value: it works when there's **no traffic** (so you catch a broken deploy at 4am before users arrive), it tests the **full path** including DNS, TLS, CDN, and network, and it gives you a consistent baseline unaffected by traffic mix. It's also the only way to monitor a critical path that's rarely exercised — the disaster-recovery login, the annual report export.

Limits: it only tests what you scripted, from where you put the probes, and synthetic checks routinely become the thing nobody maintains.

### What is Real User Monitoring (RUM)?

Instrumentation in real users' browsers or apps reporting actual experienced performance — page load, Core Web Vitals, JS errors, API latency as the client saw it, segmented by device, browser, geography, and network.

RUM catches what nothing else can: client-side JS errors, slow performance on cheap Android devices over 3G, a CDN edge misbehaving in one region, third-party script blocking render. Server-side metrics can look perfect while RUM shows a broken experience.

The pairing to state clearly: **RUM tells you what users actually experienced; synthetic tells you whether the system works when nobody's looking.** They answer different questions and mature setups run both.

### How do you monitor a microservices architecture?

The core problem is that no single service's metrics explain a user-visible failure, because a request crosses fifteen of them. So:

1. **Distributed tracing with propagated context** — the non-negotiable foundation. Without it, cross-service latency is guesswork.
2. **Uniform RED metrics per service**, on identical dashboards, so you can compare and so a new service is observable on day one.
3. **Correlation IDs threaded through every log line**, so you can reconstruct one request's full journey.
4. **A service dependency map** — ideally generated from trace data rather than maintained by hand, since hand-maintained ones are always wrong.
5. **SLOs on user journeys, not services** — because "which of these forty services is at fault" is a diagnostic question, and "can users check out" is the alerting question.
6. **Per-dependency client-side metrics** — service A's view of service B's latency differs from B's own view, and the gap is network, queueing, or serialisation.
7. **Standardised instrumentation** — OpenTelemetry with agreed semantic conventions and consistent label names. Otherwise every team's telemetry is individually fine and collectively useless.

The senior framing: in a microservices architecture, observability is a **platform capability**, not each team's individual responsibility. If it's optional, cross-team incidents become undebuggable.

---

## 4. Incident Management

### What is an incident?

An unplanned event that degrades service (or credibly threatens to) and requires a response outside normal operations.

Two things worth adding: an incident is defined by **impact, not cause** — the same failed deploy is a P1 or a non-event depending on what users experienced. And declaring an incident is cheap while declaring one late is expensive, so a healthy culture over-declares and downgrades freely.

### What is incident severity, and how do you classify?

A classification of impact and urgency (P1–P4 or SEV1–SEV4) that determines response urgency, who gets paged, escalation path, and communication cadence.

Classify on:

- **Scope** — how many users, which users (all vs one tenant), how much revenue.
- **Severity of degradation** — total unavailability vs slow vs cosmetic.
- **Workaround** — does one exist and can users find it?
- **Trend** — stable, or getting worse? A small but rapidly growing impact often warrants a higher severity than a larger static one.
- **Data integrity or security involvement** — usually escalates automatically, because those are unrecoverable in a way that downtime isn't.

The practical advice: define these in a table in advance with concrete thresholds, because severity assessed by vibes during an incident is inconsistent and always debated afterwards.

### What happens during a P1?

1. Immediate page to on-call; the alert should page without a human deciding to.
2. Incident formally declared — dedicated channel, tracking ticket, timestamped.
3. **Incident Commander assigned** (may be the responder initially, handed off if it grows).
4. Impact assessed and communicated, including a status page update if customers are affected.
5. **Mitigation begins immediately, in parallel with diagnosis** — rollback, failover, scale, feature-flag off. Not after root cause.
6. Additional responders pulled in by role, not by "who's around."
7. Stakeholder updates at a fixed cadence, stated up front.
8. Resolution confirmed by the SLI recovering, not by the alert clearing.
9. Postmortem scheduled before the incident channel closes.

### Walk me through your incident response process.

**Detect** (alert, or a customer report — and if it's usually the latter, that's the actual finding) → **Declare** and open a channel → **Assess** impact and set severity → **Assemble** responders by role → **Mitigate** to stop the bleeding → **Communicate** on a cadence → **Verify** resolution against the SLI → **Postmortem** with tracked action items.

The two things I'd emphasise, because they're where responses actually go wrong:

- **Mitigate before you understand.** The temptation to find the root cause first is strong and wrong while users are affected. Roll back, fail over, shed load — restore service, then investigate with the pressure off.
- **Separate coordinating from fixing.** Once more than two people are involved, whoever is deepest in the debugging cannot also be running comms and delegation. That's what the IC role exists for.

### **What incident roles exist beyond Incident Commander?**

Derived from the fire service's Incident Command System; Google's IMAG uses a similar split. Most candidates only know IC, so knowing the full set stands out:

- **Incident Commander** — owns the response, not the fix. Delegates, decides, maintains the shared picture, calls severity and resolution. Deliberately keeps hands off the keyboard.
- **Operations / Ops Lead** — owns the technical work; the only role touching production. Directs the hands-on responders.
- **Communications Lead** — stakeholder and customer updates, status page, exec liaison. Exists so the IC isn't interrupted every four minutes.
- **Planning Lead** — tracks longer-horizon needs: shift handover for a long incident, follow-up tasks, filing the postmortem, tracking what's been tried.
- **Scribe** — timestamped log of actions, decisions, and observations. Underrated: without a scribe, the postmortem timeline gets reconstructed from Slack scrollback and half of it is wrong.

For small incidents one person holds all roles — that's fine and normal. The value is knowing which hat you're wearing, and recognising when the incident has outgrown one person. The single most common failure in a growing incident is the IC also trying to debug.

### How do you communicate during an outage?

- **Fixed cadence, stated up front**: "I'll post an update every 15 minutes." Predictability stops the stream of individual "any news?" pings that destroy focus.
- **Post even with nothing new.** "Still investigating, no change, next update at 14:45" is a real update and prevents escalating anxiety.
- **Single source of truth** — one incident channel, one status page. Not four DM threads.
- **Separate confirmed from suspected.** "Error rate is 40% on checkout" is a fact; "we think it's the cache" is a hypothesis. Labelling them differently protects your credibility when the hypothesis is wrong.
- **Audience-appropriate**: engineers get technical detail; execs and customers want impact, scope, and ETA-or-honest-lack-of-one.
- **Never promise a resolution time you don't have.** "Next update in 15 minutes" instead of "fixed in an hour."
- **Close the loop** with a clear all-clear and a commitment to a follow-up.

### How do you prioritise multiple simultaneous incidents?

By business impact and blast radius. Total outage outranks partial degradation; revenue path outranks internal tooling; growing impact outranks static impact.

Practically: **escalate for more responders rather than splitting attention.** Two half-handled P1s is a worse outcome than one properly handled and one explicitly deferred. And say the prioritisation out loud — "we're treating checkout first, search is a known secondary, no one is actively on it" — because silence lets stakeholders assume both are covered.

Also check whether they're actually one incident: two P1s at the same moment are correlated far more often than not.

### What does an Incident Commander do?

Coordinates the response; does not fix the problem. Specifically: assembles and delegates to responders, maintains the shared mental model of what's known, decides on risky mitigations, owns severity and resolution calls, ensures communication happens, and protects responders from interruption.

Two properties worth naming: the IC role is about **authority during the incident regardless of org seniority** — a mid-level engineer can direct a director, and a healthy culture supports that. And **the IC deliberately stays off the keyboard**; the moment they start debugging, nobody is coordinating and the response degrades.

### Runbook vs playbook?

- **Runbook** — specific, prescriptive steps for a *known* problem, meant to be followed literally. "How to fail over the primary database." Ideally automatable — a runbook followed identically every time should become a script.
- **Playbook** — broader guidance for a *class* of situations, more decision framework than procedure. "How to respond to a suspected data breach."

The useful heuristic: if you can write it as a numbered list with no judgement calls, it's a runbook and it's a candidate for automation. If it needs judgement, it's a playbook and the value is in the decision tree.

### What should a runbook contain?

Symptoms and how to confirm you're in the right runbook; **impact and severity guidance**; diagnostic steps with the actual commands; mitigation steps; **rollback procedure**; verification that it worked; escalation contacts with names and paths; links to dashboards and related runbooks; and a last-reviewed date.

The two most commonly missing pieces are the **rollback path** and the **verification step** — plenty of runbooks tell you what to do and nothing about how to know it worked or how to undo it. And a runbook nobody has executed is untested code: the review date exists so you can spot the ones that have quietly rotted.

### What is an escalation policy?

Predefined rules for who is paged next and after how long, if the first responder doesn't acknowledge or can't resolve. Typically: primary → 5 minutes unacknowledged → secondary → 10 minutes → team lead → manager.

It exists so an incident can't stall on one unreachable person, and — equally important — so **escalating is normal and blameless**. If escalation is culturally read as failure, people sit on incidents too long trying to be a hero. The policy should also cover severity-based escalation (a P1 pages the secondary immediately rather than waiting) and cross-team paths for shared dependencies.

### How do you handle an incident on your on-call shift?

Acknowledge fast. Assess severity. **Mitigate before diagnosing.** Escalate early if it's beyond your access or knowledge. Document as you go, in the channel, with timestamps. Verify with the SLI, not the alert. Hand off in writing if the shift ends first.

The judgement to demonstrate: knowing when you're out of depth and escalating at that moment rather than 40 minutes later. That's a strength signal, not a weakness one — and interviewers are specifically listening for whether you'd sit on something.

### What if you don't know the root cause?

Mitigate anyway. There's almost always a generic lever that doesn't require understanding: **roll back** the last deploy, **fail over** to the standby, **scale up**, **feature-flag off** the new path, **shed load** to protect the core, **restart** the affected component.

Then be honest in updates: "We don't yet know the cause. We've rolled back the 14:20 deploy and error rate is recovering. Next update in 15 minutes." That's a strong update. A confident wrong guess is worse than acknowledged uncertainty, because people make decisions on it.

The framing that reads as senior: **understanding the cause is a postmortem deliverable, not an incident deliverable.** During the incident, the only goal is restoring service. Preserve diagnostics (logs, a heap dump, one unhealthy instance kept out of rotation) so the investigation is still possible afterwards.

### What if management keeps asking for updates during an outage?

Set the cadence proactively and early: "I'll post an update every 15 minutes in this channel, including if there's nothing new." That converts an unbounded stream of individual requests into one predictable obligation.

If it continues, that's what a **Communications Lead** is for — delegate stakeholder management so responders keep focus. And in a longer incident, don't be defensive about it: leadership asking repeatedly is usually a signal they need something specific (customer impact numbers, whether to notify a key account) rather than idle curiosity, so asking "what decision are you trying to make?" often resolves it faster than another status update.

---

## 5. Postmortem & RCA

### What is a blameless postmortem, and why blameless?

A post-incident review focused entirely on systemic causes and improvements, explicitly not on identifying who erred.

The reasoning is practical, not sentimental: **blame destroys the data you need.** If admitting "I ran the command against prod by mistake" gets someone punished, the next person minimises, omits, or reframes — and you lose the honest timeline that is the entire input to the investigation. You end up with a comfortable, useless postmortem.

The second argument is that the individual is almost never the interesting cause. If one person's mistake could take down production, the real finding is that **the system permitted it** — no confirmation prompt, no staging gate, no RBAC boundary, no canary. Fixing the person changes nothing; fixing the system prevents the next twenty people from doing it.

Worth knowing the nuance if pushed: some practitioners prefer **"blame-aware"** over "blameless," arguing that pretending human factors don't exist is its own distortion. The distinction is between *examining* human decisions to understand why they made sense at the time, versus *punishing* them. "Why was this the reasonable action given what they knew?" is the question that produces findings.

### What is Root Cause Analysis?

The process of tracing an incident past its immediate trigger to the underlying conditions that made it possible.

I'd add a caveat that reads well in senior interviews: the singular "**root** cause" is somewhat misleading for complex systems. Serious incidents almost always require **several** conditions to coincide — a bug, plus a missing alert, plus a saturated dependency, plus a runbook that was out of date. Fixating on one "root" cause leads to a single narrow fix and a repeat incident through a slightly different path. Modern practice looks for the set of contributing conditions and picks the highest-leverage ones to address.

### How do you perform an RCA?

1. **Build the timeline first**, from evidence — logs, metrics, deploy history, chat transcripts — with timestamps. Not from memory; memory reorders events and invents causality.
2. **Establish the trigger** — the proximate change or event.
3. **Ask "why" repeatedly** (5 Whys or similar) past the trigger: not just "the deploy broke it" but why the deploy passed CI, why no canary caught it, why detection took 20 minutes, why the rollback took 15 more.
4. **Ask the detection and response questions separately** from the cause questions. "Why did it break" and "why did it take us 35 minutes to notice" are two independent findings, and the second is often more valuable.
5. **Look for the conditions, not just the trigger** — what made this failure possible, and what made it *worse* than it needed to be.
6. **Check for counterfactuals you're smuggling in.** "They should have noticed the graph" is hindsight; at the time, there were forty graphs and no alert.
7. **Identify corrective actions** targeting the systemic gaps, with owners and dates.

### What sections should a post-incident report have?

Summary and customer impact (in plain language, with numbers — duration, requests affected, revenue if known); detection method and time; **timeline** with timestamps; contributing causes; **what went well** (genuinely — this is how you learn which investments paid off); what went poorly; where you got lucky (the near-miss that didn't happen this time is a free warning); and **action items with named owners, priorities, and deadlines**.

Two things that separate a good report from a filed one: an explicit **detection and response analysis** (MTTD and MTTR broken down by phase — detect, engage, diagnose, mitigate — because that tells you where to invest), and a **"lessons for other teams"** note so the learning escapes the owning team.

### How do you prevent recurring incidents?

The uncomfortable answer: **actually complete the action items.** The dominant reason incidents recur is that the postmortem was written, filed, and the items never prioritised against feature work. So:

- Action items go in the normal backlog with owners and dates, and are reviewed like any other commitment. A P1 postmortem item that's open after 30 days should escalate.
- Reserve capacity for reliability work in every sprint, so items don't have to win a fresh argument each time.
- **Review across postmortems**, quarterly, looking for themes. Five unrelated incidents that all involved a missing timeout is a much more actionable finding than any one of them individually. This meta-analysis is where the real leverage is, and almost nobody does it.
- Track a **repeat incident rate** as an explicit metric. If it isn't going down, the process is theatre.
- Convert findings into **guardrails rather than knowledge** — a lint rule, an admission policy, a required canary — because guardrails outlast the people who remember the incident.

### Root cause vs contributing factors?

The root (or primary) cause is the condition without which the incident wouldn't have happened. Contributing factors made it more likely, more severe, longer-lasting, or harder to detect — but wouldn't have caused it alone.

Example: root cause was an unhandled null in a new code path. Contributing factors: no test covered that path; no alert existed for that error class so detection took 18 minutes; the runbook's rollback steps were stale so mitigation took 12 more; and the deploy went to 100% of traffic with no canary.

The insight to voice: **the contributing factors are usually where the better fixes are.** You can't prevent all future bugs, but you can absolutely add canary analysis and fix the runbook — and those changes reduce the impact of every future bug, not just this one. Optimising for reduced blast radius beats optimising for zero defects.

### How do you identify good corrective actions?

They target systemic gaps, are specific, have an owner, and are verifiable. "Add an alert on 5xx rate for the checkout endpoint at a 2% threshold, owned by Priya, by 15 August" is an action item. "Improve monitoring" is a sentiment.

Anti-patterns to name explicitly: "be more careful," "add documentation" as the only item, "add more training," and anything phrased as a behaviour change without a mechanism. If the fix depends on a human remembering, it will fail; if it's enforced by tooling, it won't.

Prioritise by expected impact: prevention beats detection beats mitigation-speed — but detection and mitigation improvements are usually cheaper and apply across many future incidents, so they often win on ROI.

### Who should attend a postmortem?

The responders, the owners of the affected systems, whoever made the relevant changes, and engineering leadership for anything significant. Optionally: someone from an unrelated team, because outside questions surface assumptions insiders have stopped noticing.

Practical facilitation notes worth mentioning: someone other than the primary responder should facilitate (so they can participate rather than defend), the timeline should be circulated in advance so the meeting is spent on analysis rather than reconstruction, and the atmosphere is set by whoever is most senior in the room. If a director says "who broke it," the blameless culture is over regardless of what the template says.

---

## 6. Toil & Automation

### What is toil?

Google's definition, and it's worth giving precisely because the specificity is the point: work that is **manual, repetitive, automatable, tactical, devoid of enduring value, and scales linearly with service growth.**

The last property is the critical one. Work that scales linearly means doubling your services doubles your operational load, which caps how much you can run and eventually consumes the team entirely. Non-toil work — building a platform, writing an operator — makes the *next* hundred services cheaper.

Two clarifications interviewers like: toil is not the same as low-status work (some toil is technically demanding), and it isn't the same as overhead (meetings and reviews aren't toil, they're just not engineering).

### What are examples of toil?

- Manually restarting a service that crashes predictably.
- Manually applying the same deployment checklist each release.
- Rotating credentials by hand on a schedule.
- Resizing disks as they fill, one ticket at a time.
- Manually granting access for routine requests.
- Copy-pasting metrics into a weekly report.
- Responding to the same alert with the same three commands, repeatedly.
- Manually creating a namespace/pipeline/dashboard whenever a new service is onboarded.

The last one is the highest-value target, because it's the request that arrives most often as your organisation grows.

### How do you identify toil?

**Measure it.** You can't reduce what you haven't quantified, and estimates are consistently wrong. Practical approaches: tag tickets and interrupts by category for two to four weeks; log time in a toil bucket; count alert-driven manual interventions. The result almost always shows two or three sources dominating, which makes prioritisation easy.

The quick heuristic for a single task: **would a script do this identically every time?** If yes, it's toil. If it needs real judgement, it isn't.

Also track the ratio: Google's convention caps toil at **50% of an SRE's time**, with the rest on engineering. It's less about the exact number than having *a* number that you review, because toil expands to fill available time otherwise.

### How do you reduce toil?

In order of leverage:

1. **Eliminate the cause.** The best fix is that the work stops existing — fix the crash rather than automating the restart. Automating around a bad design is a trap: you've made a permanent commitment to maintaining the workaround.
2. **Self-service tooling.** Removing yourself from the request path scales best. A template, a Helm chart, a portal — anything that lets a dev team do it without a ticket.
3. **Automate the runbook.** Any procedure followed identically becomes a script, then a job, then a controller.
4. **Declarative infrastructure and GitOps** — removes a whole category of manual application and drift correction.
5. **Autoscaling** — removes manual capacity work entirely.
6. **Better alerting** — every non-actionable page is pure toil.
7. **Push work upstream** into the platform, so it's solved once rather than per team.

And the organisational half: reserve explicit capacity for toil-reduction work. If it always competes with features at equal priority, it always loses, and you'll be doing the same manual work in two years.

### Why does automation matter in SRE?

Three reasons, and the third is the one people miss:

1. **Frees engineering time** from linear-scaling work to work that compounds.
2. **Reduces human error** in repetitive procedures — and removes the variance, so outcomes are consistent and reviewable.
3. **Makes operations auditable and reversible.** An automated process has a definition you can read, review, version, test, and roll back. A manual procedure lives in someone's head and differs slightly every time — including in ways nobody notices until it matters.

The important non-goal: automation is not about removing humans. It's about moving humans from executing procedures to designing and improving them.

### What should you automate first?

Highest frequency × highest risk of human error, weighted by blast radius. In practice that ordering is usually:

1. **Deployment and rollback** — done constantly, high blast radius when done wrong, and fast rollback improves MTTR on every future incident.
2. **The remediation for your noisiest recurring alert.**
3. **Provisioning and onboarding** — the request that scales with organisational growth.
4. **Access management and credential rotation** — repetitive, security-relevant, easy to get wrong.
5. **Backups and, crucially, restore verification.**

Note that #1 and #5 both improve incident outcomes rather than just saving time, which is why they outrank things that are technically more repetitive.

### How do you decide whether to automate something?

Compare the total cost of the manual work against the cost of building *and maintaining* the automation. The naive version is `frequency × time per run × expected lifetime`, but the factors people forget are:

- **Maintenance cost** — automation is code, and code rots. Under-maintained automation that silently fails is worse than a manual process.
- **Risk-weighted cost of error** — a rare task with catastrophic failure modes may be worth automating even at low frequency (DR failover, for instance).
- **Whether the process is stable.** Automating a process that's about to change is wasted work; sometimes the right answer is "document it now, automate it when it settles."
- **Blast radius if the automation itself misbehaves.**

The honest answer to give: sometimes "not yet" is correct, and saying so demonstrates judgement. A quarterly ten-minute task with no failure risk doesn't justify a week of engineering plus indefinite maintenance.

### Can automation introduce new risk?

Yes, and this is a favourite follow-up because it separates people who've operated automation from people who've only advocated it.

Automation fails faster and wider than a human would. A remediation loop that "fixes" something into a worse state can destroy a fleet in the time a human would have taken to check one instance. Real failure modes:

- **Remediation loops** — restarting a service repeatedly, masking a real failure until something bigger breaks.
- **Amplification** — a bug applied uniformly to 500 nodes instead of one.
- **Masking** — auto-healing hides a chronic problem, so nobody investigates until it fails in a way automation can't handle.
- **Skill atrophy** — nobody remembers how to do it manually when the automation is the thing that's broken.
- **Silent failure** — automation stops working and nobody notices, because success was invisible. This is how expired-certificate outages happen despite auto-renewal.
- **Automation as the incident** — the automated deploy that pushed the bad config everywhere in 90 seconds.

### How do you safely automate production operations?

1. **Idempotent and reversible** by design; every action needs a defined undo.
2. **Dry-run / plan mode** first, and test in a lower environment against realistic conditions.
3. **Canary the automation itself** — one instance, then one AZ, then the fleet. Same discipline as a code rollout.
4. **Rate limits and circuit breakers** — cap actions per minute; halt entirely if the failure rate spikes or if the action count exceeds a sane bound. A remediation that has fired eleven times in five minutes should stop and page a human.
5. **Kill switch** that is easy to find and doesn't require the automation to be functioning.
6. **Observable** — log every action and decision, emit metrics, and **alert when the automation stops running**, not just when it errors.
7. **Blast radius limits** — never let one run touch everything; require an explicit override for cross-AZ or cross-region scope.
8. **Human in the loop for irreversible actions** — deleting data, terminating a primary, anything with no undo.
9. **Version-controlled and reviewed** like any production code, because it is.

The principle: automation should fail **safe and loud**, not silent and wide.

### What is self-healing infrastructure?

Systems that detect and remediate known failure classes without human intervention — restarting a crashed container, replacing a failed instance via an ASG health check, autoscaling on load, failing over a database, evicting Pods from an unhealthy node.

The benefit is real: it drives MTTR toward zero for well-understood failures and removes a whole tier of pages.

The risk is equally real and worth raising unprompted: **self-healing masks chronic problems.** A Pod that OOMs and restarts every 40 minutes forever looks healthy on a dashboard, and nobody investigates the leak until it gets bad enough to outrun the restarts. Guardrails: alert on *remediation frequency* even when remediation succeeds, cap retry attempts before escalating to a human, and treat repeated auto-healing as a bug to be fixed rather than a feature working correctly.

---

## 7. Availability & Reliability

### What is high availability?

Designing a system to remain operational for a very high proportion of time, primarily through redundancy, health-checked automatic failover, and the elimination of single points of failure.

The concrete practice: identify every SPOF (single instance, single AZ, single database primary, single load balancer, one person with the credentials), then decide for each whether you're accepting or eliminating it. HA isn't a product you buy; it's the absence of the things that take you down.

### What is fault tolerance?

The system's ability to continue operating **correctly** when a component fails.

Note the correction to a common framing: fault tolerance isn't "stronger than availability" — they aren't on the same axis. Availability is a *measurement* of uptime; fault tolerance is a *design property* about behaviour under failure. A system can be highly available and not fault-tolerant (it stays up, but returns wrong answers during a partition), or fault-tolerant and unavailable (it correctly refuses to serve rather than risk incorrect data — which is what a quorum-based system does when it loses quorum, deliberately).

That trade-off is essentially the CAP theorem in practical clothing, and saying so is a good sign.

### What is redundancy?

Duplicate components so no single failure is fatal — multiple instances, multiple AZs, replicated data, multiple network paths.

Two refinements worth adding. **N+1 vs N+2**: N+1 survives one failure, N+2 survives two (or one failure *during* planned maintenance, which is the more common real scenario). And redundancy only helps if the copies fail **independently** — three instances on one host, or three AZs behind one NAT gateway, or three replicas sharing one config source, are not really redundant. Correlated failure is what turns a redundant design into an outage, and it's usually a shared dependency nobody drew on the diagram.

### What is resiliency?

The broader ability to absorb disturbances and recover gracefully — encompassing fault tolerance, but also adaptation to conditions you didn't anticipate.

The distinction I'd draw: fault tolerance handles the failures you designed for; resiliency is about degrading sensibly under the ones you didn't. Timeouts, circuit breakers, backpressure, load shedding, and graceful degradation are resiliency patterns — none of them require knowing in advance what will fail, only that *something* will.

### What is graceful degradation?

Continuing to provide reduced but useful functionality when a component fails, instead of failing entirely.

Concrete examples: serve stale cached data when the live source is down; drop personalised recommendations and show a generic list; disable non-essential features under load while protecting checkout; queue writes for later instead of rejecting them; degrade image quality rather than timing out.

The design question it forces, which is the interesting part: **which features are essential?** You have to rank them in advance, because you can't make that judgement mid-incident. That ranking is also what makes load shedding possible.

### What is failover?

Automatically switching to a standby or redundant component when the primary fails — a database promoting a replica, DNS shifting to another region, a load balancer removing an unhealthy target.

The things that actually determine whether failover works when you need it:

- **Detection time** — usually the dominant term in your total recovery time, not the switch itself.
- **Automatic vs manual** — manual failover during a real crisis is slow and error-prone, but automatic failover risks flapping and split-brain, so it needs quorum or fencing.
- **Data loss on failover** — with asynchronous replication you lose whatever hadn't replicated. That's your RPO.
- **Failback** — the return path is frequently untested and frequently the source of a second incident.
- **Whether it's ever been tested.** Untested failover is a hypothesis. Regularly exercising it is the only thing that makes it real.

### What is disaster recovery?

The plan and mechanisms for restoring service after a major catastrophic event — region loss, large-scale data corruption, ransomware, accidental mass deletion.

The distinction from failover: failover handles expected, component-level failures automatically and often invisibly. DR handles rare, large-scale events, usually involves a deliberate human decision, and is measured in RTO/RPO rather than seconds. And the DR strategy tiers by cost: backup-and-restore (cheapest, slowest), pilot light, warm standby, active-active (fastest, most expensive).

Add the line that matters: **an untested DR plan is a hypothesis.** Most DR failures are discovered during the actual disaster.

### What is business continuity?

The organisation-wide plan for continuing critical business operations through a major disruption — of which IT disaster recovery is one component. It also covers people (who's authorised to decide when key staff are unreachable), facilities, communications, vendor dependencies, and regulatory obligations.

The SRE-relevant point: your technical DR plan can be flawless and still fail if nobody knows who declares a disaster, the runbook is in the wiki that's hosted in the failed region, or the on-call rotation assumes an office that's inaccessible.

### What is MTTR?

Mean Time To **Repair/Resolve** — average time from incident start to resolution.

Worth being precise, because the acronym is overloaded: MTTR is variously Repair, Resolve, Recovery, or Respond, and people use it inconsistently. State which you mean. It's also worth decomposing, because that's where the actionable information is:

```
MTTD  detect      — incident start → someone/something notices
MTTA  acknowledge — notice → responder engaged
      diagnose    — engaged → cause understood well enough to act
      mitigate    — action taken → service restored
```

**The senior point:** MTTR as a *mean* is a weak metric, and knowing why is a genuine differentiator. Incident durations are long-tailed — a handful of multi-hour incidents dominate the average, so MTTR moves mostly with outliers and is nearly uncorrelated with typical performance. The VOID report made this case directly. If pushed, say you'd track the **distribution** (median and p90 duration, plus incident count by severity) rather than a single mean, and be sceptical of MTTR as a target since it's trivially gamed by reclassifying incidents.

### What is MTTD?

Mean Time To **Detect** — incident start to detection.

Usually the largest reducible component of total downtime, and the cheapest to improve: better SLO-based alerting, synthetic monitoring for low-traffic paths, and burn-rate alerts that catch problems before they're outages. Also the most diagnostic metric you have about your monitoring: if a meaningful share of your incidents are detected by *customers* rather than alerts, that's the finding, and it outranks whatever the postmortem's stated root cause was.

### What is MTBF?

Mean Time **Between Failures** — how often failures occur at all.

Be careful with the definition, since it's commonly stated loosely. Classically, for a repairable system, `MTBF = MTTF + MTTR` — failure-start to failure-start, including the repair time. "Time from one incident ending to the next beginning" is closer to **MTTF** (mean time to failure), i.e. mean uptime. The distinction rarely matters operationally but flagging it correctly signals rigour.

The framing that matters more: MTBF is about **prevention**, MTTR is about **recovery**, and SRE deliberately invests more in the latter. You cannot drive MTBF to infinity — failure is inevitable in a system you're actively changing — but you can drive MTTR very low. That's why fast rollback, feature flags, and good observability are higher-leverage investments than trying to never break anything.

### How do you improve MTTR?

Attack it phase by phase, since the decomposition tells you where the time actually goes:

- **Detection** — SLO burn-rate alerts, synthetic checks, alert on symptoms. Usually the biggest single win.
- **Acknowledgement** — sane escalation policies, reliable paging, no single point of human failure.
- **Diagnosis** — this is where observability pays for itself: distributed tracing, correlation IDs, dashboards linked from the alert, and deploy markers on every graph so "what changed" takes seconds.
- **Mitigation** — **fast, tested rollback is the highest-leverage investment here.** Feature flags let you disable a code path in seconds without a deploy. Automated rollback on failed canary analysis removes the human entirely.
- **Across all phases** — practised incident response (game days, so process is reflex rather than improvisation), current runbooks, and a clear IC structure so coordination isn't invented under pressure.

The single best answer if you only get one sentence: **make rollback fast, safe, and boring**, because it converts "diagnose then fix" into "revert now, diagnose later" for a large share of incidents.

### RTO and RPO?

- **RTO (Recovery Time Objective)** — maximum acceptable time to restore service. Drives your architecture: an RTO of minutes requires warm standby or active-active; an RTO of a day permits backup-and-restore.
- **RPO (Recovery Point Objective)** — maximum acceptable **data loss**, measured in time. An RPO of 15 minutes means losing up to 15 minutes of data is tolerable, which sets your replication and snapshot frequency.

The practical framing: **RTO drives compute architecture, RPO drives data architecture.** RPO of zero requires synchronous replication, which costs you write latency and, under partition, availability — a direct CAP trade-off, and a good place to show you understand that reliability decisions have prices.

And define them *before* designing anything, per service. Not every service needs the same numbers, and pretending they do is how DR budgets get wasted on internal admin tools.

---

## 8. Production Scenarios

Answer these as a *method*, not a list of guesses. The frame that works for nearly all of them: **what changed → which golden signal moved → narrow by dimension → mitigate → confirm → prevent.** "What changed recently" resolves a large share of production incidents and should be your first question every time.

### The application is slow. Where do you start?

Quantify first: slow for whom, since when, on which endpoints, and by how much? "Slow" from a report is not the same as a latency graph.

Then: **check what changed** (deploys, config, feature flags, traffic mix, a dependency's release) — then walk the request path with a `curl -w` style breakdown or a trace to find *where* the time goes. DNS, connect, TLS, server processing, and downstream calls are four completely different problems.

Then localise: all users or a subset? All endpoints or one? All instances or one? Started at a specific timestamp (points to a change) or grew gradually (points to saturation or a leak)?

### CPU is low but users report slowness. What's happening?

Low CPU rules out compute saturation, which is informative. The remaining candidates:

- **I/O wait** — processes blocked on disk or network, not CPU. Check `iowait`, D-state processes, disk queue depth. A saturated EBS volume shows near-zero CPU.
- **Downstream latency** — you're waiting on a database or API. Your service is idle *because* it's blocked.
- **Lock contention** — threads serialised on a mutex or a database row lock. Concurrency collapses, CPU stays low.
- **Connection or thread pool exhaustion** — requests queue *before* processing, so they never consume CPU. Very common and very invisible: the pool is the bottleneck, not the machine.
- **GC pauses** — stop-the-world pauses show as latency spikes with low average CPU.
- **CPU throttling** (containers) — a CFS quota throttles you while node-level CPU looks fine. Check throttled seconds, not utilisation.
- **Network** — packet loss and retransmits, or hitting an instance's bandwidth allowance.

The general lesson to state: **saturation is not always CPU.** Queue depth and wait time are better saturation signals than utilisation, which is exactly the point of the USE method.

### Memory keeps increasing daily. What do you do?

Graph it over a week or two first — the shape tells you the class of problem. Steady linear growth to a ceiling then a restart, repeating, is a leak. A step change after a deploy is a regression. Growth tracking traffic is under-provisioning.

Then: check for unbounded growth in the usual places — caches with no eviction policy or TTL, connection or session objects never closed, unbounded in-memory queues, accumulating timers or listeners. Use the runtime's own tooling (heap dumps, JVM/Go/Node profilers) rather than guessing from RSS.

Also check for confounders: RSS includes page cache in some measurements, so "growing memory" is occasionally just the kernel caching files, which is fine. `container_memory_working_set_bytes` is the number that matters in containers. And check whether the runtime is sized against the *container* limit or the *host's* memory — a JVM defaulting to the host's RAM inside a small container OOMs reliably.

Short-term mitigation is a scheduled restart or a raised limit; that buys time, it isn't the fix.

### Disk hits 100%. What now?

Find the growth, then reclaim safely:

```bash
df -h ; df -i                        # bytes AND inodes
du -sh /* 2>/dev/null | sort -h
lsof +L1                             # deleted-but-open files
```

Common causes: logs not rotating; a growing database or WAL; container images and layers accumulating; core dumps; temp files never cleaned; backups written locally.

Two gotchas worth naming: **inode exhaustion** shows plenty of free space while writes fail — only `df -i` reveals it. And **deleted-but-still-open files** hold space until the holding process is restarted, so deleting a huge log that a process still has open frees nothing; `lsof +L1` finds them and truncating (`: > file`) is safer than deleting.

Prevention: log rotation, retention policies, monitoring with enough lead time to act (alert at 75%, not 95%), and separate volumes so a runaway log can't fill the same disk as the database.

### The database is suddenly slow. How do you diagnose?

Check in this order, since it roughly tracks frequency:

1. **What changed** — a deploy introducing a new query, a data volume crossing a threshold that flipped a query plan, or an index dropped.
2. **Slow query log / `pg_stat_statements`** — find the query actually consuming time. Usually one query pattern dominates.
3. **Locks and blocking** — long transactions holding locks, or an idle-in-transaction session blocking everything behind it.
4. **Connections** — at or near `max_connections`? Queueing for a connection presents as universal slowness with an idle-looking database.
5. **Resource ceilings** — CPU, but especially **IOPS and burst balance** against what's provisioned. Hitting the IOPS limit degrades latency sharply with no CPU signal, which misleads people badly.
6. **Cache hit ratio** — a drop means you're reading from disk; often a data-growth or memory-pressure symptom.
7. **Replication lag** — if reads go to replicas, lag makes reads slow *and* stale.
8. **Plan regression** — stale statistics after a bulk load can flip an index scan to a sequential scan. `ANALYZE`.

Mitigation levers before root cause: kill the offending long-running query, add a connection pooler, fail reads over to a replica, or roll back the deploy that introduced the query.

### An availability zone goes down. What happens?

Ideally, nothing that requires you — this is precisely what multi-AZ architecture is for. Multi-AZ RDS promotes the standby, ASGs replace instances in surviving AZs, and the load balancer stops routing to unhealthy targets.

The SRE job is therefore to **verify the design worked**, not to engineer a fix live: is the database failed over, are there enough healthy instances in the remaining AZs, are Pods rescheduling successfully, is the error rate recovering?

The failure mode to watch for and mention: **capacity**. Losing one of three AZs shifts 33% of load onto the remaining two. If you were running at 80% utilisation, you now need 120% and you're down anyway — the redundancy was on paper only. (See the headroom math in §10.) Second failure mode: AZ-bound resources. An EBS volume in the failed AZ can't attach elsewhere, so StatefulSet Pods won't recover — that's a real design constraint, not a bug.

### A deployment causes 500s. What's your response?

**Roll back first.** Don't diagnose a customer-facing regression in production while it's ongoing.

```
1. Confirm correlation with the deploy (timestamps, deploy marker on the error graph)
2. Roll back — or flip the feature flag, which is faster
3. Verify error rate recovers
4. Preserve evidence: logs, one bad instance out of rotation, the failing build
5. Then diagnose, off the clock
```

The follow-up question is usually "when *wouldn't* you roll back?" Legitimate answers: when the release included a non-reversible database migration (which is why expand/contract migrations matter); when rolling back reintroduces a worse bug; when the change is a security fix. Otherwise, roll back.

Prevention: canary deploys with automated analysis, `rollout status --timeout` with automatic rollback in the pipeline, and feature flags so release is decoupled from deploy.

### How do you debug intermittent failures?

Find the pattern before you form a theory — intermittent means "correlated with something you haven't identified yet."

Slice by dimension: **specific instances or Pods** (one bad replica in five gives exactly 20% errors — a recognisable signature), **specific nodes or AZs**, **specific request types or tenants**, **time-correlated** (deploys, cron jobs, batch loads, scaling events, certificate renewals), or **load-correlated** (which means saturation somewhere).

The critical tooling point: **aggregate dashboards hide intermittent failures by construction.** A 2% error rate averages into invisibility. You need per-instance and per-dimension breakdowns, high-cardinality querying, tracing to find which hop fails, and structured logs with request IDs so you can follow a single failing request end to end.

Usual suspects for intermittent specifically: one unhealthy backend still in rotation; connection pool exhaustion under load; DNS timeouts (the ~5s conntrack race has a distinctive round-number signature); idle-timeout mismatch between LB and application causing periodic resets; retries masking a partial failure; and requests hitting terminating Pods during rollouts.

### One microservice keeps timing out. How do you investigate?

Distinguish *its* problem from *its dependencies'* problem — that's the whole question.

Trace a slow request and look at the span breakdown: if time is spent in its own code, look at its CPU, GC, locks, and thread pool. If time is spent waiting on a downstream call, the timeout is a symptom and you've found the wrong service.

Then check the classics for the service itself: saturated connection/thread pool, CPU throttling, undersized replicas, and whether its **timeout configuration is coherent** — a caller with a 1s timeout in front of a service that takes 1.2s at p99 produces exactly this. Timeouts should decrease as you go deeper into the call chain; if an inner call's timeout exceeds the outer one, the outer gives up while work continues, which wastes capacity and confuses everyone.

The resiliency fix, regardless of cause: timeouts, retries with jitter and a retry budget, and a **circuit breaker** so a struggling dependency isn't hammered and the caller fails fast instead of hanging.

### An external API's latency increases. What do you do?

Protect yourself; you can't fix their service.

- **Timeouts** — aggressive and explicit. The default in most HTTP clients is "wait forever," which is how one slow dependency exhausts your thread pool and takes you down.
- **Circuit breaker** — after a failure threshold, stop calling for a cooldown and fail fast.
- **Bulkhead** — a dedicated connection/thread pool per dependency, so this one can't starve the rest of your service.
- **Graceful degradation** — serve cached or stale data, omit the feature, or queue the work for later.
- **Retries with exponential backoff and jitter**, and a retry budget so you don't amplify their overload.
- **Make it async** if the response isn't needed synchronously — the strongest structural fix.
- **Alert on dependency SLIs separately**, and communicate with the vendor with data.

The point to make: a downstream dependency's latency becoming *your* outage is a design failure on your side, not just theirs.

### The cache hit ratio drops. What's going on?

Candidates, in rough order:

- **Deploy changed the cache key** — a version or serialisation change invalidates everything at once. Check what shipped.
- **Mass invalidation or a flush** — deliberate or accidental.
- **Eviction under memory pressure** — the working set outgrew the cache, so it's thrashing. Check evictions and memory usage, not just hit ratio.
- **TTLs too short** relative to access patterns.
- **Traffic shift** — new content, a crawler, or a long-tail access pattern that simply isn't cacheable.
- **A cache node lost** — a shard's worth of keys gone; with poor hashing, more than its share.

The reason it matters urgently: a cache miss storm becomes a **thundering herd** against the origin, and the origin is sized for cached traffic. That's a classic cascading failure. Mitigations: request coalescing / single-flight so a thousand concurrent misses for one key become one origin call, staggered TTLs with jitter to avoid synchronised expiry, stale-while-revalidate, and pre-warming after a flush.

### DNS resolution fails intermittently. What do you check?

Resolver health and reachability first, then narrow: is it internal names, external names, or both? Specific resolvers or all? Specific nodes?

Check: resolver CPU/throttling and error rate; whether egress to port 53 is being blocked (a newly-added default-deny network policy is a classic); TTL and caching behaviour; upstream forwarding config; and conntrack table saturation on the nodes.

The signature to know: **~5-second delays that then succeed** point to the UDP conntrack race condition, not a broken resolver. Mitigations are NodeLocal DNSCache, reducing `ndots`, or forcing TCP.

The framing worth adding: DNS is a shared dependency with cluster-wide blast radius, which is why it should be resourced and monitored like a production service rather than treated as background plumbing.

### SSL certificates expire unexpectedly. What's the real problem?

The certificate is the symptom; the **process** is the incident. Renewing it manually resolves the outage and fixes nothing.

Diagnose why automation didn't fire: an ACM DNS validation record deleted during a DNS cleanup; cert-manager unable to complete an ACME challenge because an ingress rule changed; a renewal job that has been failing silently for weeks; a certificate managed by a person who left; or a cert outside the automated system entirely (on a load balancer, in a CI system, on an internal mTLS endpoint).

The fixes are both halves: automate renewal (cert-manager, ACM), and **monitor expiry independently of the renewal mechanism** — a blackbox exporter checking actual served certificate expiry, alerting at 30 and 14 days. Monitoring the renewal job isn't enough, because the failure mode is the job succeeding against the wrong thing. And maintain an inventory, because the cert that expires is always the one nobody knew about.

### A third-party dependency is unavailable. How do you handle it?

Fail fast rather than hang (circuit breaker), degrade the dependent feature gracefully, and communicate clearly if the impact is customer-visible.

The preparation matters more than the response: know in advance, per dependency, whether it's on the critical path, whether you can operate without it, and for how long. That inventory is what lets you answer "can we still take orders if the payment provider is down?" with something other than a shrug.

Also decide **fail-open vs fail-closed** deliberately, per dependency: an authorisation service should fail closed (deny) because failing open is a security hole; a recommendation service should fail open (serve generic content) because failing closed costs you a page for no reason. Getting this backwards is a real and common design error.

### The monitoring system goes down during an incident. What now?

Fall back to direct observation — SSH or `kubectl` to the affected systems, curl health endpoints directly, read logs at source, check the cloud provider's own console metrics (CloudWatch keeps working when your Prometheus doesn't).

The finding to state, though, is architectural: **monitoring must not share failure domains with what it monitors.** If Prometheus runs in the cluster it watches, a cluster-wide failure blinds you exactly when you need sight most. Mitigations: run monitoring in a separate cluster/account/region, use a managed backend (CloudWatch, Managed Prometheus, a SaaS), keep an out-of-band alerting path that doesn't depend on your primary stack, use external synthetic checks from a third-party vantage point, and **monitor the monitoring** with a dead man's switch — an alert that fires if it *stops* receiving a heartbeat.

That last one is the specific answer to "how would you know your monitoring is down?" and it's the piece most setups lack.

### Traffic increases 10x suddenly. What happens and what do you do?

Autoscaling helps but is **too slow for a genuine spike** — metrics have a resolution floor, then a scaling decision, then instance boot and warm-up. You're looking at minutes, and the spike arrives in seconds. So the answer has three layers:

1. **Absorb** with existing headroom — which only works if you kept some. This is the argument for capacity buffer over maximum utilisation.
2. **Shed and throttle** — rate limit per client, and load shed by priority: protect checkout, drop recommendations, disable analytics writes. Shedding cheaply and early beats collapsing under load, because a saturated system serves *nobody*, not just the excess.
3. **Scale**, with pre-warming where possible (pre-provisioned concurrency, over-provisioning "pause" Pods, warmed LB capacity for known events).

Also protect the layers behind you: a 10x front-end spike becomes a 10x database spike, and databases don't scale in seconds. Connection pooling, caching, and queueing writes are what stop the spike propagating.

And check whether the traffic is legitimate — a 10x jump is as likely to be a bot, a crawler, or a retry storm from your own client as it is real users. Retry storms in particular are self-inflicted and get worse as you scale up.

### A region becomes unavailable. What do you do?

If you're multi-region: execute the failover — DNS or global load balancer shifts traffic, the standby region takes over, data has been replicated. Then verify capacity in the surviving region can carry full load, and confirm data consistency at the failover point (with async replication you've lost your RPO's worth of writes and need to know what).

If you're single-region: this is a DR event, and your RTO is whatever your restore process actually takes. Be honest about that in an interview — plenty of production systems are single-region by deliberate cost decision, and the mature answer is "we accepted this risk explicitly, here's the documented RTO," not pretending otherwise.

The point interviewers are usually probing: **has this ever been tested?** Regional failover that's been architected but never exercised typically fails on the small things — a hardcoded endpoint, an unreplicated secret, a certificate that only exists in the primary region, a Terraform state file in the failed region, IAM roles that don't exist in the DR account.

### What causes cascading failures, and how do you prevent them?

The mechanism: one component slows or fails → callers retry and hold connections open → shared resources (thread pools, connection pools, memory) exhaust in the *callers* → they now fail too → their callers retry → the failure propagates upward and outward, often to services with no direct relationship to the original fault.

The accelerants: **no timeouts** (threads block indefinitely), **retries without backoff or budget** (amplifying load onto something already struggling), **no circuit breakers** (continuing to call a dead dependency), **shared resource pools** (one dependency's slowness starves all of them), and **synchronised retries** (no jitter, so clients retry in waves).

Prevention is a set of patterns, not a single fix: timeouts everywhere with deadline propagation; retries with exponential backoff, jitter, and a **retry budget** capping total retry traffic; circuit breakers; **bulkheads** isolating pools per dependency; load shedding by priority; backpressure so overload is signalled upstream rather than queued; and blast-radius limits like cells or shuffle sharding.

The one-sentence version: **cascading failures are a design property, not bad luck.** A system with proper timeouts, circuit breakers, and bulkheads contains a component failure; a system without them converts any component failure into a total outage.

---

## 9. Deployment & Release Engineering

### What is blue-green deployment?

Two identical production environments. Blue serves live traffic; green gets the new version and is validated; then traffic switches wholesale from blue to green. Blue is kept intact as the rollback path.

Strengths: rollback is a traffic switch, so it's near-instant, and you can validate the new environment fully before any user touches it. Costs: double the infrastructure during the cutover, and **database schema is the hard part** — both environments usually share one database, so migrations must be compatible with both versions. The cutover is also all-or-nothing, meaning 100% of users meet the new version simultaneously; a bug that only appears under real production traffic hits everyone at once.

### What is canary deployment?

Route a small fraction of traffic (1%, then 5%, 25%, 50%, 100%) to the new version, monitoring at each stage, and abort if metrics degrade.

The core advantage: you get **real production traffic feedback with a bounded blast radius**. A bug reaches 1% of users, not all of them. What makes it work in practice is **automated canary analysis** — statistically comparing the canary's error rate and latency against the baseline and rolling back automatically, rather than a human eyeballing a dashboard.

Caveats worth raising: you need enough traffic for 1% to be statistically meaningful (a low-traffic service can't canary usefully), both versions run simultaneously so schema and API compatibility are required, and you need per-version metric labelling or you can't tell the canary's numbers from the baseline's.

### What is rolling deployment?

Replace instances with the new version incrementally — a few at a time — without a parallel environment. This is Kubernetes' default (`maxSurge`/`maxUnavailable`).

No extra infrastructure, and it's the simplest to operate. Trade-offs: rollback means another rolling operation, so it's slower than blue-green's switch; you're in a mixed-version state during the rollout; and there's no traffic-percentage control — you're shifting by instance count, not by request routing. It's the sensible default for most services and doesn't need justifying.

### What is A/B deployment?

Mechanically similar to canary — traffic split across versions — but the *purpose* differs, and that's the whole answer. Canary asks "is this release safe?" and lasts minutes to hours. A/B asks "which variant produces better business outcomes?" and runs for days or weeks with statistically designed cohorts, deliberate segmentation, and a hypothesis about conversion or engagement.

Practical implication: A/B is typically implemented with **feature flags** rather than deployment infrastructure, because you want cohort assignment to be sticky per user and independent of which instance served the request.

### How do you achieve zero-downtime deployments?

The mechanics (rolling/blue-green/canary) are necessary but not sufficient. The full set:

1. **Multiple replicas.** One replica cannot be zero-downtime.
2. **Health checks gating the traffic shift** — new instances receive traffic only once genuinely ready, not merely started.
3. **Connection draining** — remove from the load balancer, let in-flight requests finish, *then* terminate. This includes a short delay to cover the LB's own propagation lag, which is the source of most "we still see a few 502s on every deploy."
4. **Graceful shutdown in the application** — handle SIGTERM: stop accepting new work, finish in flight, close pools, exit. An app that ignores SIGTERM drops connections no matter how good your orchestration is.
5. **Backward-compatible database migrations** — expand/contract: add the new column, deploy code writing to both, backfill, deploy code reading the new one, only then drop the old. Never a destructive migration in the same release as the code that needs it.
6. **Backward-compatible APIs and message formats**, since both versions run at once.
7. **Automated verification and rollback** in the pipeline.

The part candidates usually miss is #3 and #4 — most residual deploy errors are connection-handling, not orchestration.

### Which deployment strategy do you prefer?

Frame it as a judgement, because it is one: **canary with automated analysis** is the strongest general-purpose default for a high-traffic service — real production feedback, bounded blast radius, cheaper than full blue-green.

But I'd choose by context: **rolling** for low-traffic internal services where canary statistics are meaningless and the complexity isn't earned; **blue-green** when you need to validate a whole environment atomically or when the change spans components that must move together; **canary plus feature flags** for anything genuinely risky, because flags decouple "code is deployed" from "behaviour is enabled" and give you a sub-second off switch.

The reasoning matters more than the choice. An interviewer asking this is testing whether you have a defensible position, not whether you picked their favourite.

### When do you roll back?

As soon as the release is confirmed to be causing user-facing harm — error rate up, latency regressed, canary analysis failed, business metric dropped. **Bias hard toward rolling back rather than fixing forward under pressure**, because a hot fix written during an incident is written by a stressed person with no review and frequently makes things worse.

Define the triggers in advance so the decision isn't a debate: "canary error rate exceeds baseline by X for Y minutes → automatic rollback." A human deciding whether to roll back mid-incident will hesitate; a threshold won't.

When *not* to roll back: irreversible migrations, rollback reintroducing a worse bug, or a security fix. Those are the exceptions, and each is a reason to have avoided that situation in the first place.

### What metrics indicate deployment success?

Compared against a pre-deployment baseline, not against absolute thresholds:

- **Error rate** — overall and per endpoint, 5xx and business-level failures.
- **Latency percentiles** — p50 and p99; p99 often degrades while p50 looks fine.
- **Traffic** — a *drop* can mean clients are failing before they reach you.
- **Saturation** — a version that uses 40% more memory will OOM under peak later.
- **Business metrics** — conversion, checkout completion, signup. These catch the deploy that returns 200s while doing nothing useful, which no technical metric detects.
- **Error budget burn rate** during and after.

"It didn't crash" is not a success metric, and neither is "the pipeline was green."

### How do you validate a production deployment?

Automated canary analysis against the baseline; smoke tests on critical user journeys; synthetic checks against real endpoints; and a **bake time** at each traffic percentage before advancing — long enough for slow-burn issues (memory growth, connection leaks, cache degradation) to appear, since those don't show in the first ninety seconds.

Then a defined post-deploy watch window with the error budget in view. And the release isn't "done" at 100% traffic — plenty of regressions surface at the next traffic peak, hours later.

### What is progressive delivery?

The umbrella term for releasing changes gradually and reversibly, with automated decision-making. It covers canary, blue-green, feature flags, traffic shifting, ring-based rollouts (internal users → beta → general), and automated rollback.

The unifying idea: **exposure is a dial, not a switch**, and advancing the dial is gated on data rather than on someone's confidence. Tools: Argo Rollouts, Flagger, LaunchDarkly, or a service mesh for traffic splitting.

### How do feature flags improve reliability?

By **decoupling deploy from release.** Code ships dark, disabled. You then enable it for internal users, then 1%, then everyone — and disable it in seconds if anything looks wrong, with no rebuild, no pipeline, no rollback.

The reliability gains are concrete: mitigation time drops from a deploy cycle (minutes at best) to a config change (seconds); you can kill a specific misbehaving feature without reverting unrelated changes in the same release; and you get a **kill switch for load shedding** — disable expensive non-essential features during a traffic spike to protect the core path.

The cost, which you should volunteer: flags are **technical debt with a half-life.** Every flag doubles the number of code paths, and untested combinations become their own failure mode. You need an expiry process — flags get owners and removal dates, and stale flags are cleaned up. A codebase with three hundred permanent flags is less reliable, not more.

---

## 10. Capacity Planning

### What is capacity planning?

Forecasting resource needs ahead of demand so you scale proactively rather than discovering the ceiling in production. It covers compute, memory, storage, network, database connections, and — often forgotten — **service quotas and limits** (API rate limits, IP address space, connection limits).

Modern framing: autoscaling handles *reactive* capacity within a range; capacity planning sets that range, ensures the underlying resources exist, and covers everything that can't autoscale — databases, quotas, subnet IPs, licence counts.

### Why does capacity planning matter?

Because both failure modes are expensive and asymmetric. Under-provisioning causes outages and degraded experience, which costs revenue and trust. Over-provisioning costs money continuously and invisibly. Capacity planning is explicitly the search for the right point between them — with a deliberate bias toward the cheaper mistake.

The reason it can't be replaced by autoscaling alone: autoscaling is bounded below by its reaction time (metrics resolution, scaling decision, instance boot, warm-up) and above by limits you set and quotas you have. A spike arriving faster than that loop needs pre-existing headroom.

### How do you estimate future capacity needs?

1. **Trend analysis** on historical data — organic growth, plus seasonality (weekly and annual). Extrapolate, but check the shape: linear and exponential growth give very different answers a year out.
2. **Known events** — marketing campaigns, product launches, seasonal peaks, a large customer onboarding, a regulatory deadline. Talk to the business; this information exists and engineers routinely don't ask for it.
3. **Load testing to find the actual ceiling** — not "can we handle 2x?" but "at what load does latency degrade and what breaks first?" You want the identity of the first bottleneck, because that's what you'll fix.
4. **Per-unit resource cost** — measure resource consumption per request or per user, so you can convert a business forecast into an infrastructure number.
5. **Headroom for failure and deploys** — see the math below.
6. **Lead times** — some capacity isn't instant: reserved instance purchases, quota increase requests, database instance resizing with a maintenance window, subnet expansion. Plan against the longest lead time, not the shortest.

### **How much headroom do you actually need?**

This is the concrete answer to "why can't we run at 90% utilisation," and having the arithmetic is unusual and persuasive.

**Redundancy headroom** — if load is spread across N failure domains and you must survive losing one, each domain needs capacity for `1/(N-1)` of total load:

| AZs | Load per AZ (normal) | After losing one | Max steady-state utilisation |
|---|---|---|---|
| 2 | 50% | 100% | **50%** |
| 3 | 33% | 50% | **~67%** |
| 4 | 25% | 33% | **75%** |

So a three-AZ deployment intended to survive an AZ loss **cannot run above ~67% utilisation**, and that's before accounting for anything else. This is why "we're multi-AZ" and "we're at 85% CPU" are mutually exclusive claims.

**Then add:**
- **Deploy headroom** — rolling updates need spare capacity for surge Pods, or `maxUnavailable: 0` is impossible.
- **Autoscaling reaction headroom** — enough buffer to absorb load for the duration of the scaling loop.
- **Queueing headroom** — the important one, below.

**Why utilisation and latency aren't linear:** in a simple queueing model, wait time scales with `1/(1−ρ)` where ρ is utilisation:

| Utilisation | Relative queueing delay |
|---|---|
| 50% | 2x |
| 70% | 3.3x |
| 80% | 5x |
| 90% | **10x** |
| 95% | **20x** |

Which is the answer to "we have spare CPU, why is it slow?" — the last 20% of utilisation costs you far more latency than the first 80%. Target 60–70% steady-state for latency-sensitive services, not because you're wasteful, but because the tail latency past that point is unacceptable.

### **What is Little's Law and why does it matter?**

`L = λW` — the average number of items in a system equals arrival rate times average time in the system.

The practical use is **sizing pools**:

```
500 requests/sec × 0.2s average latency = 100 requests in flight
→ you need ~100 concurrent workers / connections, plus headroom
```

And read in reverse, it explains a failure mode: if latency doubles because a dependency slowed down, in-flight requests double at the same arrival rate. Your pool of 100 is now insufficient at 200, requests queue, latency rises further, and more requests pile in. That feedback loop is how a modest downstream slowdown becomes a total outage — and it's why timeouts and bounded queues matter more than large pools.

Good answer to give when asked how you'd size a connection pool or thread pool, since most people guess.

### When do you scale vertically?

- The workload doesn't distribute — a single-writer relational database being the canonical case.
- You're not near the instance family's ceiling and a bigger instance is simpler than re-architecting.
- The bottleneck is per-instance resources: memory for a large in-memory dataset, single-thread performance, or local IOPS.
- Licensing is per-instance, making fewer bigger nodes cheaper.
- Short term, as a stopgap while you build horizontal capability.

Limits to acknowledge: there's always a ceiling; scaling up usually needs a restart (so downtime, or failover); cost grows super-linearly at the top of the range; and a big single instance is still a single point of failure — you've bought capacity, not availability.

### When do you scale horizontally?

- The workload is stateless or shardable.
- **You want redundancy anyway** — this is the key point, since horizontal scaling gives you availability and capacity from the same investment.
- You're near the vertical ceiling.
- Load is variable and you want to scale elastically with demand, in both directions.
- You want to avoid a single point of failure and reduce blast radius.

Costs: statelessness must be designed for (sessions, sticky routing, local caches); coordination overhead grows; more instances means more connections to shared dependencies, which frequently moves the bottleneck to the database; and per-instance efficiency drops (each replica has its own baseline overhead and cold caches).

The default for cloud-native services is horizontal, and the interesting engineering is usually in making a workload horizontally scalable rather than in the scaling itself.

### Which metrics drive scaling decisions?

- **Utilisation trends** — CPU and memory, over weeks not minutes.
- **Request rate** and its growth trajectory.
- **Latency degradation curve against load** — the point where p99 starts to bend is your real capacity, well before 100% utilisation.
- **Saturation signals** — queue depth, connection pool utilisation, thread pool wait time. These lead; utilisation lags.
- **Queue age** for async systems (`ApproximateAgeOfOldestMessage` is often the single best signal you have).
- **Error budget burn** — if you're burning budget at normal load, you're already under-provisioned regardless of what CPU says.
- **Business-forecast-derived load**, for anything beyond a few weeks out.

The nuance worth stating: **scale on saturation, plan on trends.** Autoscaling should react to a saturation signal (queue depth, concurrency) rather than CPU where possible, because CPU is a lagging and often misleading proxy.

### How do you calculate infrastructure requirements?

```
1. Measure resource cost per unit of work
   (e.g. 1 vCPU per 200 rps; 400MB RSS per instance at steady state)
2. Take forecast peak load — peak, not average, with seasonality
3. Divide → base capacity required
4. Multiply for redundancy headroom  (×1.5 for 3-AZ survivability)
5. Add deploy surge headroom          (+ maxSurge worth of capacity)
6. Add queueing headroom              (target ~65–70% utilisation, not 100%)
7. Check the constraints that don't scale:
   database connections, service quotas, subnet IPs, licence counts
8. Validate with a load test rather than trusting the arithmetic
```

Step 7 is where real capacity plans fail. Compute is easy to add; you discover the subnet has 12 free IPs, or RDS is at `max_connections`, or you've hit an account-level EC2 quota, at the worst possible moment. Step 8 matters because per-unit costs are rarely linear — efficiency changes with concurrency and cache behaviour.

### What are the risks of over-provisioning?

Direct cost, continuously and quietly. Secondary effects: masked inefficiency (a memory leak or an O(n²) query hides comfortably in spare capacity for a long time), reduced pressure to optimise, and worse bin-packing economics.

But it's the **safer failure mode**, and saying so is the balanced answer. Over-provisioning costs money; under-provisioning costs users. Given uncertainty, err toward capacity — then use utilisation data to trim deliberately rather than guessing tight from the start.

### What are the risks of under-provisioning?

Degraded latency, errors, and outages under real load — direct user impact and error budget burn. And it's worse than it looks in three ways:

- **Non-linear degradation.** Performance doesn't decline gently as you approach capacity; it falls off a cliff (see the queueing table above). You get very little warning.
- **It removes your resilience.** A system at 95% utilisation has no capacity to absorb an AZ failure, a retry storm, or a rolling deploy — so a routine event becomes an incident.
- **It's the precondition for cascading failure.** Saturated systems can't absorb the extra load from retries, so a small perturbation propagates.

The framing: under-provisioning doesn't just risk an outage at peak, it converts every other minor failure into an outage.

---

## 11. Performance Engineering

### Latency vs throughput?

**Latency** is the time for one operation to complete. **Throughput** is the rate of operations over time.

They're related but not interchangeable, and the trade-off is the interesting part: **batching increases throughput and increases latency** (waiting to fill a batch); adding concurrency increases throughput until a resource saturates, after which latency rises while throughput plateaus. Optimising for one often costs the other, so you need to know which your users actually care about — a payment API cares about latency, a nightly ETL cares about throughput, and treating them the same is a design error.

### What is concurrency, and how does it differ from parallelism?

**Concurrency** is multiple operations making progress over overlapping time periods — a structural property of how work is organised. **Parallelism** is multiple operations executing literally simultaneously on separate execution units.

Rob Pike's line is the one to use: *concurrency is about dealing with a lot of things at once; parallelism is about doing a lot of things at once.* A single-core machine can be highly concurrent (an event loop handling 10,000 connections) with zero parallelism. Node.js is the standard example — concurrent, not parallel, for application code.

Why it matters operationally: an I/O-bound service benefits from concurrency (more in-flight requests while waiting), while a CPU-bound service needs parallelism (more cores) and gets nothing from more threads. Misdiagnosing which one you have leads to adding threads to a CPU-bound service and making it slower through context-switching overhead.

### What is scalability?

The ability to handle increased load by adding resources, ideally with capacity growing roughly proportionally to resources added.

The honest version acknowledges that it never is proportional — Amdahl's Law says the serial fraction bounds your speedup, and the Universal Scalability Law adds that beyond some point **coordination overhead makes additional resources actively harmful** (throughput decreases as you add nodes). That's not theoretical: it's what's happening when adding replicas increases latency because they all contend on the same database.

So the practical question is never "is this scalable" but "what's the bottleneck, and what's the coordination cost of adding capacity?"

### What is backpressure?

A mechanism where a saturated component signals upstream producers to slow down, rather than silently queueing unboundedly or dropping work uncontrolled.

Why it matters: without backpressure, an overloaded consumer accumulates an unbounded queue. Latency climbs toward infinity, memory grows until OOM, and — the worst part — the work sitting in the queue is often already useless, because the client timed out long ago. You're spending capacity on requests nobody is waiting for.

Implementations: TCP flow control (the classic), bounded queues that block or reject when full, gRPC/HTTP2 flow control, reactive streams' request(n) protocol, and consumer-driven pull models like Kafka. The related practices are **bounded queues everywhere** (an unbounded queue is a latent outage) and **dropping stale work** — if a request has exceeded its deadline, discard it rather than processing it.

### Backpressure vs load shedding vs rate limiting?

Worth distinguishing clearly, since the original list conflates them:

- **Rate limiting** — enforcing a per-client quota, applied regardless of your health. Purpose: fairness and abuse prevention. "You get 1000 requests/minute."
- **Load shedding** — *you* rejecting work because *you* are saturated, ideally by priority. Purpose: self-preservation. "I'm at capacity; I'll drop analytics writes and keep serving checkout."
- **Backpressure** — signalling upstream to slow down rather than rejecting outright. Purpose: flow control through the whole pipeline.

The key insight about load shedding: **rejecting requests fast is better than accepting all of them slowly.** A saturated system that tries to serve everything serves *nobody* successfully — every request times out. Shedding 30% cleanly means 70% succeed. That's counterintuitive and worth stating explicitly, because the instinct is always to try to serve everything.

And shed by priority, which requires having ranked your traffic in advance: health checks and critical paths first, batch and analytics first to go.

### What causes latency spikes?

- **GC pauses** — stop-the-world collection; classic p99 spike with normal p50.
- **CPU throttling** — CFS quota in containers; looks like network latency and isn't.
- **Lock contention** — serialisation on a mutex or database row.
- **Cold caches** — after a deploy, restart, or cache flush; every request goes to origin.
- **Downstream slowness** — including a dependency's own GC or deploy.
- **Connection/thread pool exhaustion** — queueing before processing begins.
- **Noisy neighbours** — shared tenancy, or a batch job on the same node.
- **DNS timeouts** — distinctive round-number delays.
- **Disk or IOPS saturation** — including exhausted burst credits, which is invisible in CPU.
- **TLS handshakes** — no keep-alive means every request pays setup cost.
- **Retry storms** amplifying load onto something already struggling.
- **Scheduled work** — cron jobs, backups, log rotation, certificate renewal, index rebuilds.

The diagnostic move: **correlate the spike with a timestamp pattern.** Perfectly periodic points to scheduled work or GC; correlated with deploys points to cold caches; correlated with load points to saturation; random points to noisy neighbours or an unhealthy instance.

### How do you identify performance bottlenecks?

**Measure, don't guess** — and the reason to say that first is that intuition about performance is reliably wrong, including for experienced engineers.

1. **Distributed tracing** to find *where* time goes across the request path. This is the highest-value first step in a distributed system; it turns a search into a lookup.
2. **The `curl -w` breakdown** for a single request — DNS, connect, TLS, TTFB, transfer as separate numbers.
3. **Profiling** — CPU profiles and flame graphs for compute-bound code; heap profiles for memory; continuous profiling in production if available, since dev-environment profiles often mislead.
4. **The USE method per resource** — utilisation, saturation, errors on CPU, memory, disk, network. Systematic rather than intuitive.
5. **Database-side analysis** — slow query logs, `EXPLAIN ANALYZE`, `pg_stat_statements`. A large share of "application performance problems" are one query.
6. **Then verify.** Change one thing, measure again. Optimisations that "should" help frequently don't, and occasionally hurt.

And the discipline point: **fix the actual bottleneck, in order.** Optimising anything other than the current constraint produces no measurable improvement, which is the most common way performance work gets wasted.

### How do you benchmark a service?

- **Realistic traffic mix**, not a single endpoint hammered. Real workloads have a distribution of request types, payload sizes, and cache-hit ratios, and the mix changes the answer.
- **Ramp load gradually** and record the full curve of latency and error rate against load, rather than a single pass/fail at one level. What you want is the **knee** — the load at which p99 starts to bend — because that's your real capacity.
- **Report percentiles, not averages**, and report error rate alongside. A "fast" result with 5% errors is meaningless.
- **Run long enough** to expose warm-up effects, GC behaviour, connection leaks, and memory growth. Sixty-second benchmarks measure the best case.
- **Avoid coordinated omission** — use an open-model load generator that maintains the target rate regardless of response times, or a tool that corrects for it. Closed-loop generators that wait for each response systematically understate the tail, because during a stall they stop issuing requests and simply don't record the delay.
- **Test in a production-like environment**, with production-like data volumes. Query plans change with table size.
- **Test failure modes too** — behaviour when a dependency is slow or down is more useful than behaviour on the happy path.

### How do you optimise API performance?

Roughly in order of typical impact:

1. **Fix the database.** Missing indexes, N+1 query patterns, and unbounded result sets account for a large share of slow APIs. Look here before anything clever.
2. **Cache** at the right layer — CDN for static, application cache for computed results, database query cache — with correct invalidation and jittered TTLs.
3. **Reduce round trips.** Parallelise independent downstream calls instead of sequencing them; batch where possible. Ten sequential 20ms calls is 200ms that could be 25ms.
4. **Make non-essential work async** — push it to a queue and return. The single biggest structural win available in most APIs.
5. **Connection pooling and keep-alive** — avoid paying TCP and TLS setup per request.
6. **Reduce payload size** — compression, field selection, pagination. Matters most on mobile networks.
7. **Set coherent timeouts with deadline propagation**, so a slow dependency can't consume the whole request budget.
8. **Only then micro-optimise code.** Serialisation costs and allocation patterns are real but almost never the top item.

The meta-point: **profile first.** Every one of these is wasted effort if applied to the wrong layer, and most API latency budgets are dominated by one or two items on this list rather than spread evenly.

---

## 12. Kubernetes & Cloud (SRE Lens)

*(Deep coverage in the Kubernetes/EKS answer bank; this is the reliability-framing layer.)*

### What role does Kubernetes play in SRE?

It provides primitives for several core reliability concerns as platform behaviour rather than bespoke scripts: self-healing, declarative desired state, rolling deployments with health gating, horizontal autoscaling, and service discovery with health-aware routing.

The honest caveat to add, because it distinguishes an operator from an enthusiast: **Kubernetes moves the reliability problem, it doesn't remove it.** It adds significant operational complexity of its own — a control plane, a network layer, a storage layer, and a substantial set of ways to misconfigure things. It reliably solves "a process died"; it doesn't solve "the change we shipped was wrong," which is what causes most incidents.

### How do you ensure Kubernetes cluster reliability?

- **Spread across AZs** — nodes and Pods, with `topologySpreadConstraints`.
- **Resource requests and limits everywhere** — a missing limit is a noisy-neighbour and eviction risk; a wrong request is a scheduling and cost problem.
- **PodDisruptionBudgets** so voluntary disruptions (drains, upgrades) can't take too many replicas.
- **Honest probes** — readiness gating traffic, liveness checking only the process.
- **Graceful shutdown** with `preStop` and SIGTERM handling.
- **Capacity headroom** so failures and rollouts have somewhere to go.
- **Managed control plane** where possible; managed add-ons.
- **Treat shared add-ons as production services** — CoreDNS, CNI, ingress controller, CSI drivers all have cluster-wide blast radius and should be resourced, monitored, and alerted accordingly.
- **Tested backup/restore** — etcd snapshots for self-managed, Velero plus GitOps for workload state.
- **Stay on a supported version** and upgrade on a schedule.

### How do you design a highly available Kubernetes cluster?

Control plane across at least three AZs (or managed); workers across at least three AZs with enough capacity to lose one (see the 67% headroom math in §10); replicas spread with topology constraints; multi-AZ or managed data stores; PDBs on everything; and no single-instance dependencies in the critical path.

The two things people forget: **AZ-bound EBS volumes** mean StatefulSet Pods can only return to their original AZ, so stateful HA needs explicit design rather than the same pattern as stateless. And **shared single points of failure** — one NAT gateway, one ingress controller replica, one CoreDNS Pod — quietly undo the AZ spread you paid for.

### How do you handle an entire node failure?

Mostly automatic: kubelet stops heartbeating, the node controller marks it NotReady after ~40s, `NoExecute` taints evict its Pods after the toleration period (default 300s), controllers recreate them, the scheduler places them elsewhere, and the ASG replaces the instance.

The SRE questions are whether the *preconditions* hold: is there capacity on the remaining nodes (or Pods sit Pending and you're down), were replicas spread so this isn't a cold start, do PDBs prevent this compounding with an in-progress drain, and can AZ-bound volumes actually reattach. Optionally, tune the eviction timeout if five minutes of degraded capacity is too long for the service.

### How do you upgrade a cluster safely?

Control plane → add-ons → nodes, one minor version at a time, never skipping. Pre-flight: check for removed APIs, verify add-on compatibility, confirm subnet IP capacity (upgrades add nodes before removing them), sanity-check PDBs, and rehearse in a lower environment. Drain gracefully respecting PDBs rather than force-terminating. Verify against application SLIs, not just node readiness.

### How do you avoid noisy neighbour problems?

Requests and limits on every container so the scheduler and kubelet can enforce fair sharing; namespace `ResourceQuota` and `LimitRange` so a single team can't consume the cluster; separate node pools for workloads with incompatible profiles (batch vs latency-sensitive); and Guaranteed QoS (requests == limits) for anything latency-critical, since it's evicted last.

Worth noting the CPU-limit subtlety: CPU limits *prevent* noisy neighbours but *cause* throttling, which is its own latency problem. Many teams set CPU requests without limits and rely on requests for fair sharing.

### How do you ensure workload isolation?

Namespaces as the organisational boundary; NetworkPolicies with default-deny for network isolation; RBAC scoped per namespace; separate ServiceAccounts with IRSA/Pod Identity for cloud permissions; taints and tolerations plus node affinity for dedicated node pools; and Pod Security Standards to prevent privilege escalation.

The honest limit: **namespaces are not a security boundary against a determined attacker.** For genuinely hostile multi-tenancy — untrusted user code — you need separate clusters, or at minimum a sandboxed runtime (gVisor, Firecracker) plus strict policy. Saying this unprompted is a good signal.

### What's specific about monitoring EKS?

Same Kubernetes fundamentals, plus: **enable control-plane logging to CloudWatch** (you can't SSH to the control plane, so those logs are your only window), monitor node group health and autoscaler/Karpenter decisions, and watch the EKS-specific failure classes — **IRSA/OIDC auth failures**, **VPC CNI IP exhaustion** (which presents as Pending Pods with no obvious resource shortage), security group misconfiguration, and ALB target group health.

Also monitor **service quotas** as first-class signals: ENIs per instance, IPs per subnet, ELBs per region, EC2 instance limits. Quota exhaustion is a common EKS incident cause and looks nothing like a resource problem.

---

## 13. Security & Reliability

### What is the shared responsibility model?

The cloud provider secures the infrastructure *of* the cloud — physical facilities, hardware, hypervisor, and the internals of managed services. The customer secures what they build *in* the cloud — IAM configuration, network configuration, data, application code, patching of anything they run.

The line moves with the service tier, which is the part worth articulating: on EC2 you patch the OS; on RDS AWS patches the engine but you own users, encryption settings, and network exposure; on Lambda you own only code and permissions. Misjudging where the line sits for a given service is the source of a lot of real breaches — "we assumed the provider handled that" is the postmortem sentence.

### How do you manage secrets securely?

Dedicated secret manager (AWS Secrets Manager, SSM Parameter Store, Vault) as the store of record; runtime injection rather than baked into images or Git; least-privilege access to individual secret ARNs; encryption at rest with a customer-managed key; automatic rotation where supported; mounted as **files rather than environment variables** (env vars leak into process listings, crash dumps, and logs); and audit logging on access.

Never in Git — and if GitOps demands it, use SOPS or Sealed Secrets so what's committed is ciphertext, with `gitleaks` in CI to catch the mistakes. Never in image layers or build args, since both persist in image history.

### Why does least privilege matter?

It bounds the blast radius of a compromise. The realistic threat isn't usually an attacker breaking your crypto; it's a leaked token, an SSRF, a compromised dependency, or a stolen laptop. Whether that becomes a minor incident or a total breach is determined almost entirely by what the compromised identity was permitted to do.

The reliability angle, which is the connection interviewers are often fishing for: least privilege also prevents **accidents**. A CI role that can't delete production resources can't accidentally delete them, and a developer role scoped to staging can't `kubectl delete` the wrong context. A meaningful share of outages are authorisation failures in the sense that someone had permissions they didn't need.

### How do you audit production access?

Centralised, immutable, tamper-evident logging of who did what and when — CloudTrail with log file validation, Kubernetes audit logs, database audit logs — shipped to an account the operators being audited can't modify. Then: alerting on high-risk actions (root usage, IAM policy changes, security group modification, `exec` into production Pods, mass deletions), periodic access reviews to remove accumulated permissions, and just-in-time elevated access with expiry rather than standing admin.

The two things that make audit real rather than theatrical: someone actually reviews it, and the logs are outside the blast radius of the thing being audited.

### How do you rotate secrets without downtime?

**Dual-credential (overlapping validity) rotation** is the pattern: create the new credential, ensure both old and new are simultaneously valid, roll consumers over to the new one, verify nothing is using the old one, then revoke it. Attempting an atomic swap requires perfect synchronisation across every consumer and doesn't survive contact with reality.

Supporting practices: a secrets manager with built-in rotation (Secrets Manager's dual-user strategy for RDS does exactly this), consumers that **re-read credentials at runtime** rather than caching them at startup (this is the piece applications usually get wrong — a mounted CSI secret can refresh in place; an env var cannot), and monitoring old-credential usage so you know when it's safe to revoke.

### What is defence in depth?

Multiple independent controls, so no single failure is fully compromising: network segmentation, then identity and authorisation, then encryption, then runtime detection, then audit. An attacker past the perimeter still faces authorisation; past authorisation, the data is encrypted; and throughout, detection is watching.

The design principle underneath it is **assume breach.** Perimeter-only security fails completely the moment the perimeter is crossed, and it always eventually is.

### How does security contribute to reliability?

They're the same discipline viewed from two angles, and the strongest version of this answer is that **a security incident *is* a reliability incident**: ransomware is downtime, a DDoS is a saturation event, a breach forces an emergency shutdown, and data exfiltration destroys trust more thoroughly than any outage.

They also share almost all their practices. Least privilege limits blast radius for accidents and attacks alike. Audit logs serve forensics and postmortems. Patching closes vulnerabilities and fixes reliability bugs. Immutable infrastructure prevents drift and persistence. IaC gives reviewability for both.

The tension to acknowledge honestly: they sometimes conflict. Mandatory MFA on break-glass access slows incident response; fail-closed authorisation trades availability for safety; aggressive patching risks regressions. Those are real trade-offs and the answer is to make them deliberately per service, not to pretend the tension doesn't exist.

---

## 14. Networking & Distributed Systems

### What happens when a request times out?

The caller stops waiting and treats it as a failure — but the crucial point is that **the server may still be processing it, and may still complete it successfully.** The timeout is a client-side decision; it doesn't cancel server-side work unless you've propagated cancellation explicitly.

The consequences: retrying a non-idempotent operation can duplicate it (the classic double-charge); the client and server now disagree about what happened; and the abandoned work still consumes server capacity, which is why a service under stress with clients retrying gets worse rather than better.

Mitigations: **idempotency keys** so a retry is safely deduplicated; **deadline propagation** so downstream services know the request is already abandoned and can stop; and **cancellation** (context cancellation, HTTP client disconnect detection) so abandoned work is dropped.

### What is retry logic, and when should you not retry?

Re-attempting a failed request, bounded by an attempt cap and a backoff schedule.

**Don't retry when:**
- **The operation isn't idempotent** and you have no idempotency key. Retrying a charge, a send, or an insert can duplicate it.
- **The failure isn't transient.** A 400, 401, 403, 404, or 422 will fail identically forever. Retry 5xx, 429, and connection-level failures; not 4xx (except 429).
- **The deadline has already passed** — the client has given up, so the retry is pure waste.
- **A circuit breaker is open** — that's the entire point of the breaker.
- **The downstream is overloaded** — retries make it worse. This is the important one: retries are the mechanism by which a partial degradation becomes a total outage.

### **What is retry amplification?**

The failure mode nobody mentions and interviewers love:

```
Gateway retries 3x → Service A retries 3x → Service B retries 3x
= up to 27 requests hit the database for one user request
```

At 1,000 rps of user traffic, a database hiccup means 27,000 rps of retry traffic arriving at the thing that's already struggling. The retries *are* the outage.

Defences: **retry at one layer only** (usually the outermost, or the one closest to the failure — but pick one deliberately); **retry budgets** capping retries to a small percentage of total request volume, so retry traffic can never dominate; deadline propagation so exhausted requests aren't retried; and circuit breakers to stop retrying a dependency that's clearly down.

### What is exponential backoff, and why jitter?

Increasing the delay between attempts exponentially — 1s, 2s, 4s, 8s — rather than retrying immediately or at a fixed interval. It gives the struggling dependency room to recover instead of being hammered on a fixed cadence.

**Jitter** — randomising the delay — matters just as much, and here's why: without it, all clients that failed at the same moment retry at the same moment, producing a synchronised thundering herd that repeats at every backoff interval. The system oscillates between idle and overwhelmed and never recovers. Jitter spreads the retries out.

Worth knowing the variants (from AWS's write-up on the topic): **full jitter** (`random(0, backoff)`) is generally the best default; **equal jitter** (`backoff/2 + random(0, backoff/2)`) keeps a minimum delay; **decorrelated jitter** performs better in some measured scenarios. Naming full jitter as your default shows you've read past the basics.

### What is the circuit breaker pattern?

A client-side state machine that stops calling a failing dependency:

- **Closed** — normal; requests pass through, failures are counted.
- **Open** — after a failure threshold, requests fail immediately without a call, for a cooldown period.
- **Half-open** — after cooldown, allow a limited number of trial requests. Success closes the circuit; failure re-opens it.

Two benefits, and both matter: it **fails fast** (the client isn't holding threads waiting on timeouts, so its own resources aren't exhausted), and it **gives the dependency room to recover** rather than pinning it under load.

Tuning notes: thresholds should be rate-based rather than count-based (5 failures means something different at 10 rps than at 10,000 rps), the cooldown must be long enough for real recovery, and you need a **fallback path** for when the circuit is open — cached data, a degraded response, or a clear error. A circuit breaker with no fallback just converts slow failures into fast ones, which is an improvement but not a solution.

### What is bulkheading?

Isolating resources per dependency or per workload class, so one failure can't exhaust shared resources and take down everything. Named after ship compartments that contain flooding.

The concrete failure it prevents: your service has one thread pool of 200. Dependency A slows to 10 seconds. Requests needing A occupy all 200 threads. Now requests that don't touch A at all can't be served either — you're fully down because of one degraded dependency. With a dedicated 40-thread pool for A, A's requests fail and everything else continues.

Implementations: separate connection and thread pools per dependency, separate node pools or clusters per workload class, separate queues per priority, and — at the largest scale — **cell-based architecture**, where the entire stack is replicated into independent cells and each customer is served by one cell, so a bad deploy or a poison-pill request affects one cell rather than the fleet.

### What is service discovery?

The mechanism by which services locate each other dynamically, since instance addresses change constantly in containerised and autoscaled environments.

Approaches: **DNS-based** (Kubernetes Services, Route 53 — simple and universal, but caching and TTLs make it slow to converge); **registry-based** (Consul, Eureka — services register and clients query, with health checking); **load-balancer-based** (the LB is the stable address); and **mesh-based** (a sidecar handles discovery and load balancing transparently).

Operational concerns worth mentioning: **health checking** (a discovered endpoint must be verified healthy, or you're distributing traffic to dead instances), **propagation delay** during scale events, and **DNS caching in clients** — JVM applications caching DNS indefinitely is a well-known source of traffic sent to terminated instances long after they're gone.

### What is eventual consistency?

A model where, absent new writes, all replicas converge to the same value — but may return stale or differing data in the interim.

The trade-off is the CAP/PACELC one: you get higher availability and lower latency, at the cost of reading stale data. Whether that's acceptable is a product question, not a technical one — a stale follower count is fine, a stale bank balance is not.

Practical middle grounds worth naming, since "eventual vs strong" is a false binary: **read-your-writes** consistency (a user always sees their own changes, even if others see them later) is often exactly what's needed and much cheaper than global strong consistency. Also **monotonic reads** (never see data go backwards) and **bounded staleness** (stale by at most N seconds). Mechanisms include sticky routing to the primary for a period after a write, or version tokens.

### What is split-brain?

A network partition causes two subsets of nodes to each believe they're the sole leader, so both accept writes. When the partition heals you have conflicting state and no safe automatic resolution — which is why split-brain is usually worse than downtime.

Prevention: **quorum** — require a majority to elect a leader, so a minority partition cannot form one (this is why you run odd numbers of etcd or ZooKeeper nodes). Plus **fencing** — the classic STONITH, or a fencing token that the storage layer checks, so a deposed leader's writes are rejected even if it hasn't noticed it was deposed. Leases with expiry rather than indefinite leadership.

The design lesson to state: in a partition you must choose availability or consistency. Systems that try to have both quietly end up with split-brain, and the failure is silent until reconciliation.

### What are distributed transactions?

Transactions spanning multiple independent services or databases that need all-or-nothing semantics.

- **Two-phase commit** — a coordinator asks all participants to prepare, then commits if all agree. Gives strong consistency, but it's **blocking**: if the coordinator fails after prepare, participants hold locks indefinitely. Poor availability, and it doesn't scale.
- **Saga pattern** — a sequence of local transactions, each with a **compensating action** to undo it. If step four fails, run the compensations for three, two, one. Available and scalable, but only eventually consistent, and compensations are genuinely hard (you can't un-send an email; you send an apology).
- **Outbox pattern** — write the business change and the outgoing event in one local transaction, then publish the event asynchronously. Solves the common "we updated the database but the message never sent" dual-write problem, and it's the pattern to name if asked something practical.

The senior framing: the best answer is usually **avoid needing one.** Distributed transactions are a symptom of service boundaries drawn in the wrong place. If two services must always commit together, they may be one service — or the operation should be modelled as an async workflow with explicit intermediate states, which is what a Saga really is.

### What causes cascading failures?

*(Also covered in §8 — the short version.)* One component's degradation causes callers to retry and hold resources; shared pools exhaust in the callers; they fail too; their callers retry; the failure propagates outward, often to services with no direct dependency on the original fault.

The accelerants are all absences: no timeouts, no retry budgets, no jitter, no circuit breakers, no bulkheads, no load shedding, and no capacity headroom to absorb the extra load. **A cascading failure is a design property, not bad luck** — the same component failure in a system with those patterns is contained to that component.

Worth adding one non-obvious accelerant: **recovery can trigger a second cascade.** When the dependency comes back, every queued client retries simultaneously, and the thundering herd knocks it over again. Jittered backoff and gradual ramp-up on recovery are what prevent that, and it's the detail that shows you've actually watched this happen.

---

## 15. System Design

The graded skill is your **structured approach**, not a specific architecture. Never start drawing boxes — start with requirements. Interviewers fail candidates for jumping to a solution far more often than for picking the wrong database.

### The framework

1. **Clarify requirements** (spend real time here — 5 of 45 minutes)
   - Functional: what must it do? Which user journeys matter?
   - Scale: requests/sec, data volume, growth rate, read/write ratio, peak vs average.
   - **Reliability targets: RTO, RPO, and a proposed SLO.** This is the SRE-specific move, and most candidates skip it. Ask for it, or state your assumption.
   - Constraints: budget, team size, compliance, existing stack, latency requirements.
2. **Do back-of-envelope math** — derive storage, bandwidth, and instance counts from the numbers. Show the arithmetic.
3. **Sketch the high-level architecture** — components and data flow, happy path first.
4. **Then apply the reliability lens, deliberately:**
   - Where are the single points of failure? Name each and say whether you're eliminating or accepting it.
   - What's the failure mode of every component, and what's the blast radius?
   - Where are the timeouts, retries, circuit breakers, and bulkheads?
   - How does it degrade gracefully rather than fail totally?
5. **Address data** — consistency model, durability, replication, backup and restore, and migration path.
6. **Address observability** — what are the SLIs, what would you alert on, what would you need to debug this at 3am.
7. **State trade-offs explicitly.** "I'm choosing X, accepting Y, because Z." This is the single highest-scoring behaviour in a design interview.
8. **Identify what you'd build first** — an MVP and an evolution path, rather than presenting a finished five-year architecture.

### Worked example: design a highly available web application

Because "multi-AZ, load balancer, ASG" is the answer everyone gives, here's what the same question looks like answered properly.

**Requirements I'd establish:** 5,000 rps peak, 500 rps average, read-heavy (90/10), 99.95% availability SLO (~22 min/month), p99 under 300ms, RPO 5 minutes, RTO 15 minutes, single region acceptable initially.

**Architecture:**

```
Route 53 (health-checked)
   ↓
CloudFront (static assets, TLS termination at edge)
   ↓
ALB (multi-AZ, deletion protection)
   ↓
App tier: 3 AZs, ASG or EKS, stateless
   ↓
ElastiCache (Redis, multi-AZ, replica) — read cache
   ↓
RDS Postgres, Multi-AZ, + 2 read replicas
   ↓
S3 (versioned, lifecycle) for objects; SQS for async work
```

**The reliability reasoning — this is the part being graded:**

- **Capacity:** 5,000 rps peak across 3 AZs must survive losing one → each AZ sized for 2,500 rps → provision 7,500 rps of capacity, i.e. **150% of peak**, targeting ~65% steady-state utilisation. That's the §10 math applied, and stating it makes the design credible.
- **Stateless app tier** so instances are disposable and horizontal scaling is trivial. Sessions in Redis or a signed cookie, never in instance memory.
- **SPOFs, named:** the RDS primary (mitigated by Multi-AZ automatic failover, ~60–120s — which must fit inside the 22-minute monthly budget, and note that a failover consumes a meaningful chunk of it); the ALB (AWS-managed, multi-AZ); Route 53 (global, health-checked); the single region (accepted, documented, with a DR plan to meet the 15-minute RTO if that assumption changes).
- **Degradation:** if Redis fails, serve from the database at higher latency rather than erroring — cache is an optimisation, not a dependency. If a read replica fails, route to the primary. If the write path fails, serve read-only rather than a blank error page.
- **Cascading failure protection:** timeouts on every database and cache call, connection pooling with **PgBouncer or RDS Proxy** (because the app tier autoscaling would otherwise exhaust `max_connections` — a very common real failure), circuit breakers on downstream calls, and load shedding by endpoint priority.
- **Data:** RDS automated backups plus PITR gives RPO well inside 5 minutes; S3 versioning against accidental deletion; cross-region snapshot copies if the DR posture tightens later.
- **Deployment:** rolling or canary, `maxUnavailable: 0`, automated rollback on canary failure, expand/contract migrations so rollback is always safe.
- **Observability:** SLIs are request success rate and p99 latency measured at the ALB (closest to the user I can get cheaply), plus RUM for the real client experience. Burn-rate alerting on the SLO. Dashboards: golden signals per tier, database saturation, cache hit ratio, connection pool utilisation.
- **What I'd build first:** single-AZ with Multi-AZ RDS and a working deploy pipeline, then add AZs and read replicas as traffic justifies. Presenting the end state as the starting point is a red flag, not a strength.

**Trade-offs I'd state out loud:** single region trades DR capability for roughly half the cost and much less operational complexity — appropriate at this SLO, and I'd revisit it if the target moved to 99.99%. Read replicas introduce replication lag, so anything requiring read-your-writes goes to the primary. Caching introduces staleness, bounded by TTL.

### Anchors for the other common prompts

- **Monitoring for a payment platform** — tightest SLOs in the organisation on the money-movement path specifically, and a *separate, looser* SLO for peripheral features like transaction history (don't blend them). Immutable audit-grade logging for compliance. Continuous synthetic transactions against the real payment path, because it's the one path you cannot afford to discover is broken from a customer report. Reconciliation monitoring — alert on financial totals not matching, which catches correctness failures that return 200s. Idempotency keys throughout, since a duplicate charge is worse than a failed one.
- **Alerting for a banking application** — severity tiers mapped to regulatory and compliance impact, not just user impact. Compliance-mandated response times drive the escalation policy. Deliberately asymmetric on fraud and security signals: accept false positives to avoid false negatives, because the costs aren't symmetric. Separate alerting paths for availability vs correctness vs security. Audit trail on the alerts themselves.
- **Logging pipeline for millions of requests/sec** — never block the request path on logging (async, buffered, drop-on-full with a counter so you know you dropped). Buffer through Kafka or Kinesis to absorb bursts and decouple producers from the storage backend. Tiered storage: hot in OpenSearch/Loki for days, warm in S3, queried by Athena for older data. **Sampling** — keep all errors, sample successes at 1–10%, with tail-based sampling for traces so you retain the slow and failed ones. Structured JSON with a defined schema and trace IDs. And cost as a first-class design constraint, since at this volume logging can cost more than the service it observes.
- **Disaster recovery strategy** — RTO/RPO first, per service, because they determine everything downstream. Then pick the tier that meets them: backup-and-restore, pilot light, warm standby, or active-active, each roughly an order of magnitude more expensive and faster than the last. Automate the failover trigger. Back up the non-obvious things: secrets, DNS, certificates, IaC state, IAM roles in the DR account. **Then test it on a schedule** — an untested DR plan is a hypothesis, and most DR failures are discovered during the actual disaster.
- **Multi-region architecture** — active-passive vs active-active driven by consistency requirements and cost tolerance. Global load balancing (Route 53 latency or failover routing, or CloudFront/Global Accelerator). The hard part is always data: synchronous replication costs write latency and creates a partition problem; asynchronous means a non-zero RPO and possible conflicts. Decide the conflict resolution strategy explicitly. Watch for hidden single-region dependencies — a global service in one region, a certificate, a Terraform state bucket.
- **Scalable notification system** — decouple trigger from delivery with a queue. Per-channel workers (email/SMS/push) scaling independently, since providers have wildly different throughput and failure characteristics. Retry with backoff, plus a DLQ for permanent failures. Rate limiting per provider so you don't get throttled or blacklisted. **Idempotency and deduplication** so a retry doesn't double-send. User preference and quiet-hours checks. Priority queues so a password reset isn't stuck behind a marketing blast — that separation is the detail interviewers look for.
- **Reliable API gateway** — it's the SPOF for everything behind it, so its own HA comes first: multi-AZ, multi-instance, no shared state in memory. Then: rate limiting and quota enforcement per client, circuit breaking per backend, timeouts with deadline propagation, response caching, centralised auth (with the auth service's failure mode decided deliberately — fail closed), request/response size limits, and graceful degradation when a backend is down. Plus per-route observability, since "the gateway is slow" needs to resolve to which backend.

---

## 16. On-Call & Operational Excellence

### What does being on-call involve?

Being the designated first responder for a defined shift, expected to acknowledge within an agreed time (commonly 5 minutes for a page) and to have the access, tooling, and context to act.

Worth adding what makes it *sustainable*, since that's usually the real question: a rotation large enough that shifts aren't punishing (6+ people is a common floor), compensation or time-off recognition, an explicit expectation that you're not also delivering project work during a heavy shift, and a hard rule that unsustainable alert volume is a bug to fix rather than a burden to absorb.

### How do you prepare for an on-call shift?

- Read the **handoff notes** from the previous shift — ongoing issues, known-flaky alerts, in-flight changes.
- Review **recent changes**: deploys, migrations, config, infrastructure. Most incidents follow a change, so knowing what shipped this week is the highest-value context you can have.
- **Verify your access works** — VPN, cloud console, `kubectl` contexts, dashboards, paging app, escalation contacts. Discovering an expired credential mid-incident is a self-inflicted delay, and it happens constantly.
- Check upcoming scheduled events — planned migrations, marketing pushes, certificate renewals, partner maintenance.
- Skim the runbooks for the services you're covering.
- Practical: charged phone, paging notifications not muted, known laptop and connectivity, and a plan for the hours you'll be away from a desk.

### How do you handle alert fatigue?

Treat it as a production defect with a cost, not a personality test. Concretely: track alerts per shift and the proportion that led to action; take the worst offenders to a regular alert review; delete or downgrade anything non-actionable; move causes to dashboards and page only on symptoms; adopt burn-rate alerting; add grouping and inhibition.

And be willing to say it out loud: **if on-call is unsustainable, that's a finding about the system, not a resilience problem in the engineer.** The organisational fix is making alert quality someone's explicit responsibility with time allocated to it.

### You get multiple simultaneous P1s. What do you do?

Triage by business impact and blast radius, **escalate for a second responder immediately** rather than trying to hold both, and state clearly which one you're prioritising and which is deprioritised so nobody assumes both are covered.

Then check whether they're actually the same incident — simultaneous P1s are correlated far more often than not, and finding the shared cause resolves both. If they're genuinely independent, an IC should be assigned for each once responders are available, because one person cannot coordinate two incidents.

### How do you improve on-call quality of life?

- **Cut alert noise** — the single biggest lever, and everything else is secondary to it.
- **Good runbooks** so response doesn't depend on the one person who "just knows." That dependency is both a QoL problem and an availability risk.
- **Fair rotation** — big enough rotation, reasonable shift lengths, follow-the-sun for global teams, no back-to-back shifts, and respect for time zones.
- **Fix the top pain sources** — track which services page most and treat that as a reliability backlog with real priority.
- **Reduce hero dependency** — spread knowledge deliberately through shadowing and rotation.
- **Explicit expectations** — clear response SLAs, clear escalation permission, and no expectation of project delivery during a heavy shift.
- **Post-shift review** — a short handoff meeting to surface recurring pain, and a norm of filing tickets for anything that made the shift worse.
- **Compensation and recovery time** for out-of-hours work.
- **Onboarding**: shadow shifts before primary, and a game day so the first real page isn't the first time.

### How do you hand over an unresolved incident?

A written summary covering: current impact and severity; timeline of what's happened; what's been tried and what the result was; **current hypotheses and what's been ruled out** (this is the part that saves the most time — otherwise the next person re-tests the same theories); the active mitigation and whether it's holding; next planned steps; who's been informed and what's been promised; and links to the channel, dashboards, and tickets.

Then **walk them through it live** if at all possible. A written handoff plus five minutes of conversation is enormously better than either alone, because the incoming responder's questions surface the context you didn't know to write down.

### What tools do you rely on during an incident?

Metrics and dashboards (Grafana, CloudWatch); log search; distributed tracing; the alerting and paging platform; a dedicated incident channel plus an incident management tool for timeline and roles; **deploy and change history** (frequently the fastest path to an answer — "what changed at 14:20"); direct access via `kubectl`, SSH, and the cloud console; runbooks; and a status page for external comms.

The judgement to add: **know which of these to reach for first.** The highest-value opening move is almost always "what changed recently," followed by "which golden signal moved and for whom." Opening seven dashboards and reading them in parallel is how a response loses fifteen minutes.

---

## 17. Behavioral — STAR Skeletons

Graded on **structure and specificity.** Vague answers are the dominant failure mode, and the fix is preparation, not eloquence.

**Prepare 5 stories, not 15 answers.** A well-chosen incident story can answer "most challenging incident," "time you debugged something hard," "time you were under pressure," "biggest mistake," and "time you improved reliability" — with different emphasis each time. Trying to prepare a distinct story per question phrasing is wasted effort.

**Use this template, and fill in the bracketed parts before the interview:**

```
SITUATION  (2 sentences — context and stakes, no more)
TASK       (1 sentence — what was specifically yours to do)
ACTION     (the bulk — what YOU did, decisions and reasoning, in sequence)
RESULT     (numbers, and what changed permanently as a result)
```

**Two rules that matter more than the template:**

1. **Numbers, or it didn't happen.** "Reduced deployment failures" is nothing. "Cut failed deploys from roughly one in four to near zero over two months, and cut rollback time from ~20 minutes of manual steps to a single automated command" is a story. Reconstruct real numbers now — approximate is fine, and saying "roughly" is honest.
2. **Say "I," not "we."** Interviewers cannot grade a team. Describe your specific decisions, including the ones you got wrong first.

### The five stories to build

**1. The hard debugging story** — a non-obvious root cause found methodically.

*Your best material:* the CI/CD pipeline chain where GitHub Actions' secret-masking silently blanked the AWS account ID in cross-job outputs, so ARNs arrived malformed with no error message. The reason it's a strong story is the *shape*: the failure had no useful error, the obvious hypotheses were all wrong, and the actual cause was a security feature behaving as designed. Emphasise the method — how you isolated it, what you eliminated, how you confirmed. Then the systemic fix (pass bare revision integers and reconstruct ARNs locally), which is better than a workaround because it removes the class of problem. The `$`-in-`DATABASE_URL` shell corruption is a good second beat if they want more.

**2. The reliability improvement story** — something measurably safer afterwards.

*Your material:* gating Prisma migrations as a discrete, exit-code-checked ECS RunTask so a failed migration blocks the deploy instead of producing a half-migrated production database. Frame it as risk removed, not work done: before, a failed migration could leave the deploy in an inconsistent state; after, it fails loudly at a known point. Add the security-scan gating (making Trivy and ZAP actually block rather than report) as evidence you look for the same class of gap repeatedly. **Get a number here** — deploys per week, failures caught, whatever you can honestly reconstruct.

**3. The automation / toil story.**

*Your material:* the multi-repo GitHub Actions status dashboard, or the pipelines themselves replacing manual deploy steps. The stronger framing for the dashboard is the *problem*: checking pipeline status across several client organisations meant opening N browser tabs, repeatedly, daily. That's textbook toil — repetitive, no judgement, scaling with the number of clients. Then say what it cost and what it saved. Bonus credit for mentioning you thought about the security posture (short-expiry fine-grained PAT, deployment protection), because that shows you don't ship internal tools carelessly.

**4. The failure / mistake story** — and this one needs the most preparation.

Pick something real with genuine consequence. Own it plainly in one sentence with no hedging, then spend the rest on the systemic fix. Interviewers are testing exactly one thing: whether you deflect. Any of the CodeBuild/ECS permission or buildspec issues work if you frame the learning properly — but the strongest version is one where **you** caused or missed something, not one where a tool was confusing.

The structure that works: "I did X. The impact was Y. The immediate fix was Z. But the real problem was that nothing would have caught it, so I added [the guardrail]." That last clause is the entire point of the question.

**5. The disagreement / trade-off story.**

*Your material:* two good options. Flagging `desiredCount = 1` as a downtime risk for staging and production and recommending 2 is a clean reliability-vs-cost argument. Scoping Trivy to library vulnerabilities rather than OS packages is a better story in some ways, because you argued for *less* scanning — which demonstrates you optimise for signal rather than for looking thorough. Explain your reasoning, acknowledge the counter-argument honestly, and say what was decided, including if it went against you. "I was overruled and here's why that was reasonable" is a strong answer; interviewers are testing whether you can disagree without being difficult.

### Mapping stories to question phrasings

| If they ask… | Use | Emphasise |
|---|---|---|
| Most challenging incident | Story 1 | Method under uncertainty |
| Time you reduced downtime | Story 2 | Risk removed, numbers |
| Automated a manual process | Story 3 | Toil identified and quantified |
| Failed deployment, what you learned | Story 4 | The guardrail you added |
| Disagreed with developers | Story 5 | Reasoning, and the outcome |
| Biggest production mistake | Story 4 | Ownership in the first sentence |
| Couldn't resolve immediately | Story 1 | Mitigated first, diagnosed after |
| Balancing features vs reliability | Story 5 | Explicit trade-off, not dogma |
| Improved observability | Story 3 | Who else benefited |
| Stayed calm under pressure | Story 1 or 4 | Show behaviour, don't assert calm |
| Mentored someone | Runbook/doc you wrote | Knowledge that outlived you |

### Two answers worth rehearsing specifically

**"Tell me about an outage you couldn't immediately resolve."** The correct answer describes mitigating *without* understanding — rolling back, failing over, scaling — and diagnosing afterwards. Candidates often apologise for not root-causing live, which inverts the grading: mitigate-first *is* the right instinct, so present it as a decision, not a shortcoming.

**"How do you stay calm under pressure?"** Never answer by asserting that you do. Describe the mechanism: "I work from a checklist so I'm not improvising, I state what I know versus what I'm guessing, I set an update cadence so I'm not interrupted, and I escalate early rather than trying to be the hero." Behaviour is evidence; temperament claims aren't.

---

## 18. Senior / Advanced

### What SLO would you set for a payment gateway?

Very tight — but the more important move is to **decompose by criticality** rather than quote a single number, because that's the actual senior judgement.

The money-movement path (authorisation, capture) warrants something like 99.99% availability with a strict latency bound, since failures there have direct revenue and often regulatory consequences. Peripheral features — transaction history, receipt downloads, reporting — get materially looser targets. Blending them into one SLO means the tight requirement is either unachievable or the loose one is over-engineered.

Then the reliability requirements that matter more than the availability number, and mentioning these is what distinguishes the answer:

- **Correctness is a separate SLI, and it outranks availability.** A duplicate or lost charge is far worse than a declined request. Reconciliation monitoring and idempotency keys are non-negotiable.
- **You cannot exceed your dependencies.** If the card network or acquiring bank offers 99.95%, your 99.99% target is arithmetic fiction. Either the SLO accounts for the dependency, excludes it explicitly, or you build multi-provider failover.
- **Latency bounds matter commercially** — checkout conversion measurably degrades with latency, so the latency SLO has a revenue justification, not just an engineering one.
- **Four nines means under an hour of budget per year**, so it requires sub-minute detection and automated mitigation. Committing to it without that capability is committing to missing it.

### How would you design a global monitoring platform?

**Federated, regional collection aggregating to a global view.** A single global collector is both a bottleneck and a single point of failure, and it means a cross-region network problem blinds you everywhere at once.

- **Regional collection and regional storage** for local queries and fast local alerting.
- **Region-local alerting** that works independently, so a partition doesn't silence alerts for a region that's still serving users.
- **Global aggregation** for cross-region views — long-term downsampled data in a central store (Thanos, Mimir, Managed Prometheus with cross-region query).
- **Separate failure domains from what's monitored** — different account, different region, ideally a managed backend. Monitoring must not die with the thing it watches.
- **Dead man's switch** — alert on the *absence* of a heartbeat, so a silent collector is detected. Most setups lack this and it's the specific answer to "how would you know monitoring is down?"
- **Cardinality governance** — enforced limits per team, or one team's bad label takes down shared infrastructure. At global scale this is the dominant operational problem, not query performance.
- **Standardised instrumentation** — OpenTelemetry with enforced semantic conventions, or cross-team correlation is impossible.
- **Tiered retention** — high resolution briefly, downsampled long-term, because cost scales with cardinality × retention.

### How would you reduce MTTR across an organisation?

First, **measure the phases** — detect, acknowledge, diagnose, mitigate — because the distribution tells you where the time actually is, and it's usually not where people assume. In most organisations detection and diagnosis dominate, and mitigation is fast once the cause is known.

Then, in rough order of leverage:

1. **Reduce detection time** — SLO burn-rate alerting, synthetics on low-traffic critical paths, and eliminating customer-reported-first incidents.
2. **Invest in fast, safe rollback and feature flags.** This is the highest-leverage single change: it converts "diagnose then fix" into "revert now, diagnose later" for a large fraction of incidents.
3. **Standardise incident process** — one tool, defined roles, consistent severity definitions. Cross-team incidents fail on coordination, not technical difficulty.
4. **Improve diagnosis tooling** — distributed tracing, deploy markers on every dashboard, correlation IDs, dashboards linked directly from alerts.
5. **Practise** — game days and chaos exercises, so the process is reflex. Untested process invented under pressure is slow.
6. **Runbooks with tested rollback steps**, and automation of the repeated ones.
7. **Feed postmortem action items back**, and review across incidents for themes.

The caveat to voice, since it demonstrates you think about metrics critically: **be careful making MTTR a target.** It's a long-tailed distribution, so the mean is dominated by outliers, and it's trivially gamed by reclassifying or closing incidents early. Track the distribution and the count, and treat MTTR as a diagnostic rather than a KPI.

### How would you design a multi-region failover strategy?

1. **RTO and RPO first, per service** — they determine everything, and they differ by service. Don't build one strategy for everything.
2. **Choose the posture:** active-active (lowest RTO, highest complexity, hardest consistency), active-passive warm standby (minutes RTO, moderate cost), pilot light, or backup-and-restore (hours RTO, cheapest). Match to the numbers, not to ambition.
3. **Solve data first** — it's the actual hard part. Synchronous replication gives RPO zero at the cost of write latency and a partition dilemma; asynchronous gives you a non-zero RPO and possible conflicts, which need an explicit resolution strategy. Global databases (Aurora Global, DynamoDB Global Tables) shift the problem but don't remove it.
4. **Automate the trigger** — manual failover during a real crisis is slow, and the person who knows how is often unavailable. But include a manual override, because automatic cross-region failover on a false positive is its own outage.
5. **Traffic steering** — Route 53 health-checked failover or latency routing, or Global Accelerator. Lower TTLs *before* you need them; a 300-second TTL means five minutes of RTO you can't recover.
6. **Capacity in the standby region** — a warm standby sized for 30% of load is not a failover target. This is a common and expensive gap.
7. **Find the hidden single-region dependencies** — a global service pinned to one region, certificates, Terraform state, IAM roles absent from the DR account, secrets not replicated, container images in one ECR region. These are what actually break real failovers.
8. **Plan failback** — the return path is less tested and frequently causes a second incident.
9. **Test on a schedule**, ideally with real traffic. Untested failover is the single most common reason DR plans fail when needed.

### How do you do chaos engineering safely?

It's an experiment with a hypothesis, not vandalism. The discipline:

1. **Start with a hypothesis** — "if one Redis node fails, latency rises under 10% and no requests error." A test with no prediction teaches you nothing.
2. **Establish a steady-state baseline** you can measure against.
3. **Start absurdly small** — one instance, non-critical service, off-peak, in staging first.
4. **Define abort criteria in advance**, and have a fast, tested stop button that doesn't depend on the thing you're breaking.
5. **Blast radius limits** — never let an experiment affect more than a defined fraction of traffic.
6. **Announce it.** An unannounced experiment that causes a real incident destroys the programme's political capital permanently, and the on-call engineer's trust with it.
7. **Expand only as confidence grows** — staging → production off-peak → production peak; single instance → AZ → region.
8. **Fix what you find, then re-run** to verify the fix. An experiment that finds a weakness nobody fixes is a report, not engineering.

The prerequisites people skip: you need **good observability first** (you can't measure the effect otherwise), you need to have fixed the failures you already know about, and you need organisational buy-in including from leadership. Running chaos experiments on a system with unaddressed known weaknesses and no tracing is just causing outages with extra steps.

### Can you measure engineering productivity with SRE metrics?

The DORA four are the standard, and they're the right answer because each is outcome-based:

1. **Deployment frequency** — how often you ship to production.
2. **Lead time for changes** — commit to running in production.
3. **Change failure rate** — proportion of deploys causing degradation.
4. **Time to restore service** — recovery duration.

The finding worth citing: the DORA research shows speed and stability are **positively correlated**, not traded off. High performers deploy more frequently *and* have lower change failure rates, because small frequent changes are easier to verify and to revert. That's the argument that wins the "we need to slow down to be more reliable" debate.

Caveats to raise, because the question is partly about metric literacy:

- These measure **delivery capability**, not individual productivity. Applying them per engineer is a known anti-pattern that produces gaming and no insight.
- Every one is gameable — deployment frequency by splitting commits, change failure rate by reclassifying incidents.
- DORA has since added **reliability** as a fifth metric, which is where SLO attainment fits.
- Use them as a trend for the team's own diagnosis, never as a cross-team ranking.

### How would you improve the reliability of a legacy monolith?

Incrementally, and **observability first** — because you cannot safely change what you can't measure, and legacy systems are typically opaque. Start by adding metrics, structured logging, and tracing at the boundaries, then define SLOs from what you actually observe rather than what you'd like.

Then, in order:

1. **Identify the highest-risk components** from incident history — where do outages actually come from? That data usually contradicts intuition and points at two or three modules.
2. **Add resilience at the boundaries** — timeouts, retries with backoff, circuit breakers around external calls. Often the cheapest large win, since legacy code frequently has no timeouts at all.
3. **Make deployment safer before making the code better** — a reliable pipeline with fast rollback improves every future change. Frequently the single highest-ROI intervention.
4. **Add tests around the risky paths**, so change becomes possible at all.
5. **Fix the operational basics** — resource limits, health checks, graceful shutdown, log hygiene.
6. **Then consider extraction** — strangler fig for specific high-risk or high-change components, routing traffic through a facade and moving one capability at a time.

And say the unpopular thing clearly: **resist the big-bang rewrite.** Rewrites of systems nobody fully understands routinely take multiples of their estimate and ship with the original's bugs plus new ones, while the legacy system continues to need maintenance throughout. Incremental improvement is less satisfying and much more likely to work.

### How would you migrate a critical service to Kubernetes with zero downtime?

Run both in parallel and shift traffic gradually — canary the *migration*, not just the deploy.

1. **Understand the current system properly** first: dependencies, state, configuration, cron jobs, filesystem assumptions, network assumptions. The surprises live here.
2. **Make it container-ready** — externalise state and sessions, configuration via environment, logs to stdout, handle SIGTERM. Do this on the existing platform first, so it's a separate, verifiable change.
3. **Deploy to Kubernetes alongside the existing system**, both live, both connected to the same data stores.
4. **Shadow traffic** if feasible — mirror real requests to the new stack without serving its responses. Highest-confidence validation available and it's invisible to users.
5. **Shift gradually** with a weighted load balancer or DNS: 1% → 5% → 25% → 50% → 100%, comparing SLIs at each step against the old stack rather than against an absolute threshold.
6. **Keep the old stack warm and ready** for a full traffic revert until you're confident — that's your rollback, and it should stay viable for days, not minutes.
7. **Then decommission**, and clean up DNS, IAM, and infrastructure.

The two things most likely to break, worth naming preemptively: **DNS TTLs not lowered in advance**, which makes rollback take hours instead of seconds; and **stateful assumptions** — local disk, in-memory sessions, a cron job that must run exactly once and now runs in three Pods.

### How would you build an org-wide observability strategy?

Treat observability as a **platform product** with users, not as each team's individual problem. If it's optional, you get twelve incompatible stacks and cross-team incidents become undebuggable.

- **Standardise instrumentation** — OpenTelemetry, with enforced semantic conventions and a shared set of required labels (service, environment, version, team). Provide it as a library or sidecar so adopting it is easier than not.
- **Consistent naming and labelling**, enforced at the collector. Without this, cross-service queries are impossible.
- **Trace context propagation everywhere** — the highest-value single investment, and it only works if adoption is near-universal.
- **Shared dashboards and templates** so a new service is observable on day one and every service's golden signals look the same. Familiarity matters at 3am.
- **SLOs as a first-class platform capability** — a standard way to define them, dashboard them, and generate burn-rate alerts, rather than each team hand-rolling it.
- **Cost governance** — cardinality limits, retention tiers, sampling defaults, and per-team cost visibility. Observability spend grows faster than infrastructure spend if nobody owns it.
- **Make the paved road genuinely easier** than the alternative. Mandates without ergonomics produce compliance theatre; a good default that saves teams work produces actual adoption.
- **Measure adoption and usefulness** — which services have SLOs, which have tracing, and whether MTTD is actually falling.

### How do you convince developers to adopt SRE practices?

Lead with what they get, not what they must do — and error budgets are the strongest instrument available because they genuinely favour developers most of the time.

The trade to offer: **budget remaining means ship freely, and reliability doesn't get to veto you.** That's more autonomy than most developers have under a regime of vague ops caution and case-by-case negotiation. In exchange, when the data says reliability is suffering, priority shifts — automatically, by pre-agreed rule, not by argument.

Then the practical tactics:

- **Start where the pain is.** Find the team being paged most and help them, visibly. Demonstrated results recruit better than any presentation.
- **Make the reliable path the easy path** — templates, pipelines, and defaults that include probes, limits, dashboards, and alerts. Adoption should be less work than not adopting.
- **Show the data.** "This service pages someone 14 times a week and 12 of those are one alert" is more persuasive than a principle.
- **Share on-call.** Developers who carry a pager for their own service build reliability into it without being asked. This is the single most effective cultural change available, and it's also the hardest to negotiate.
- **Don't be a gatekeeper.** SRE as an approval body creates adversaries and gets routed around. SRE as a capability provider gets invited in.
- **Avoid dogma.** Not every service needs four nines or a full SLO framework. Insisting otherwise costs credibility on the services where it genuinely matters.

### How do you balance reliability, cost, and feature velocity?

There's no universally correct point, and claiming there is would be the wrong answer. The position: it should be an **explicit, revisited, per-service decision tied to business context** — and error budgets exist precisely to make that trade objective rather than political.

Concretely:

- **Reliability requirements come from the business**, not from engineering preference. A payment path and an internal admin tool warrant genuinely different investment, and the cost of an outage in each is knowable.
- **Reliability has diminishing returns** — each additional nine costs roughly an order of magnitude more, and the point where the marginal reliability is worth less to users than the marginal cost is a real point that can be identified.
- **Don't over-deliver either.** Consistently unspent error budget means you're spending on reliability users don't perceive; that's engineering capacity that should go to features, and saying so builds enormous credibility with product.
- **The DORA finding matters here** — speed and stability aren't opposed at the practice level. Small frequent changes with good automation give you both, so a lot of the apparent trade-off dissolves with better delivery engineering.
- **Make the cost visible.** Per-service cost attribution turns "we need more capacity" into a decision someone owns.
- **Revisit on a cadence.** The right balance for a pre-revenue product and the same product with enterprise customers are different, and SLOs should move deliberately when that changes.

### How do you define service ownership across teams?

**Singular, explicit ownership per service**, documented, with on-call responsibility following ownership directly. Shared ownership reliably means nobody owns reliability, because the incentive to fix a chronic problem is diluted while the incentive to ship is not.

Mechanisms: a **service catalogue** recording owner, on-call rotation, SLOs, runbooks, dependencies, and tier/criticality. Ownership including production responsibility, not just code authorship — the team that ships it carries the pager for it. Explicit handling of shared infrastructure (a platform team owning it as *their* service, with SLOs offered to internal consumers, which is the model that works). A defined process for transferring ownership, including a readiness bar so nobody inherits an unmaintainable service.

And the orphan problem, which every organisation has: an explicit process for services with no owner, because they exist, they page people, and they're the ones that cause incidents nobody knows how to resolve.

---

## 19. Questions to Ask Them

Asking well signals seniority more efficiently than almost any answer, because these questions reveal you know what a functioning SRE practice looks like. Pick three or four; don't interrogate.

**On the actual work**
- How is on-call structured — rotation size, shift length, and typical pages per shift?
- What proportion of an SRE's time here goes to toil versus engineering? Is it measured?
- Do you have SLOs in production? Are they used to make prioritisation decisions, or mainly reported?
- Who carries the pager for a service — the SRE team, or the team that built it?

**On reliability culture**
- What was your last significant incident, and what changed as a result?
- Are postmortem action items tracked to completion? What proportion get done?
- When the error budget is exhausted, what actually happens?

**On the team**
- Is SRE embedded with product teams, or a central function? How does that work in practice?
- What's the biggest reliability problem you'd want the person in this role to take on first?
- What does the path from "developer asks for infrastructure" to "it exists" look like today?

**Diagnostic answers to listen for:** no SLOs, or SLOs nobody references. Postmortems written but action items never completed. Pages measured in dozens per shift. An SRE team functioning as a ticket queue for developer requests. "We don't really have toil" — which usually means it isn't measured.

---

*Two documents, one method: know why the practice exists, answer with a sequence rather than a fact, and put a number on it wherever you can.*
