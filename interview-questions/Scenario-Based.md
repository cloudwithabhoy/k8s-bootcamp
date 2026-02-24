# Scenario-Based Questions

Situational Kubernetes questions that test real-world problem-solving skills.

---

## 1. Post-deploy, latency spikes for 30% of users. No errors. No logs. What now?

First — 30% is suspicious. If you have 10 pods and 3 are slow, that's your 30%. So I start there.

**Triage steps:**

1. **Pod distribution** — `kubectl get pods -o wide`. Did new pods land on a hot node? If 3 out of 10 pods are on the same overloaded node, that explains 30%. I compare with `kubectl top nodes`. Real example: After a deploy, 3 new pods landed on a node that was also running a memory-intensive batch job. CPU steal was at 40%. We added `podAntiAffinity` to spread pods across nodes.

2. **CPU throttling** — The #1 silent killer. CPU limits cause kernel-level CFS throttling — the app slows down but produces no errors, no logs. The kernel literally pauses your process mid-request. Check `container_cpu_cfs_throttled_periods_total` in Prometheus. We had a Go service with a 200m CPU limit. Under load, 60% of its CPU periods were throttled. Response times went from 50ms to 800ms. No errors anywhere. Fix: removed CPU limits entirely (kept only requests) and latency dropped immediately.

3. **Cold starts** — During rolling update, new pods serve traffic immediately after readiness probe passes. But the probe might pass before the app is truly warm — JIT not optimized, connection pools empty, caches cold. Check `kubectl get pods --sort-by=.metadata.creationTimestamp`. If slow requests correlate with new pods, that's your answer. Fix: implement readiness probes that check warmth, not just liveness. Example: our Java service's readiness endpoint verifies the connection pool has minimum 5 active connections before returning 200.

4. **Stale endpoints/iptables** — During rollout, `kube-proxy` iptables rules might still point to terminating pods. Traffic hits a dying pod, hangs for a few seconds, then retries. Looks like latency, not errors. Check `kubectl get endpoints` — are pod IPs listed that are in `Terminating` state? Also check if `terminationGracePeriodSeconds` is too long. Fix: add a `preStop` hook with `sleep 5` so the pod stays alive while endpoints update.

5. **DNS latency** — If the app makes outbound calls, `ndots:5` causes 4-5 extra DNS lookups per request. Under load, CoreDNS gets overwhelmed. We verified this with `kubectl exec <pod> -- time nslookup api.stripe.com` — it took 800ms instead of 5ms. Fix: deployed NodeLocal DNSCache and set `ndots:2`.

6. **Ingress controller** — NGINX might not have updated its upstream list yet. It sends traffic to old pod IPs → connection timeout → latency spike. Check ingress controller logs for `upstream_response_time` values.

7. **Distributed tracing** — If all above is clean, filter Jaeger/Tempo traces by p99 latency in the deploy window. It shows exactly which span is slow — app code, database query, external API, or network.

**Most common root causes in my experience:** CPU throttling, cold starts, and uneven pod distribution. In 80% of cases, it's one of these three.

---

## 2. Tell me about the last outage you debugged in Kubernetes.

**Incident:** Friday evening. Payment service — 100% of requests returning 503. All pods `Running` and `Ready` (1/1). Customer checkout was completely down. PagerDuty alert: "payment-service: 0% success rate."

**How I debugged it:**

First — `kubectl get pods -n payments`. All pods running, no restarts, no CrashLoopBackOff. So the app itself is alive. The problem is in the path between users and the app.

Next — `kubectl get endpoints payment-service -n payments`. Endpoints populated, pods registered. So the payment Service is fine. The issue is deeper.

We use Istio, so I checked the sidecar: `kubectl logs <pod> -c istio-proxy -n payments`. Found: `upstream connect error or disconnect/reset before headers. reset reason: connection failure`. The proxy couldn't reach an upstream dependency. The payment service calls PostgreSQL.

Checked — `kubectl get endpoints postgres-service -n databases`. **Empty.** Zero endpoints. A service with no backends = every request gets 503.

But the postgres pod was running. So why no endpoints? I compared `kubectl describe svc postgres-service` (selector: `app: postgres`) with `kubectl get pods -n databases --show-labels` (label: `app: postgresql`). Someone had "standardized" the label name in the Deployment manifest without updating the Service selector. The Service couldn't find any matching pods.

**Fix:** `kubectl patch svc postgres-service -n databases -p '{"spec":{"selector":{"app":"postgresql"}}}'`. Endpoints repopulated in 2 seconds. Traffic recovered within 30 seconds.

**What we did after (postmortem):**
- Implemented ArgoCD — no more manual `kubectl apply` in production. Every change goes through Git PR review.
- Added a Kyverno policy that validates every Service selector matches at least one running pod
- Alert if any service has 0 endpoints for more than 60 seconds
- Standardized on `app.kubernetes.io/name` label across all teams — no more `app: postgres` vs `app: postgresql` confusion

**Why this story works in interviews:** It shows structured debugging (not random guessing), understanding of K8s internals (empty endpoints = selector mismatch), ownership (stayed through resolution), and prevention (systemic fixes, not band-aids). The worst answer to this question is "I haven't had an outage."

---

## 3. A deployment succeeded, but users get 502 errors. What do you check first, and in what order?

> **Also asked as:** "During high traffic, your app shows intermittent 502 errors through Ingress — how do you debug?"

502 means the gateway/proxy received an invalid response from upstream. The deploy succeeded, K8s is happy, but users aren't.

**My order of checks:**

1. **Readiness probe** — `kubectl get pods`. Are pods `Ready` (1/1) or just `Running` (0/1)? If the readiness probe path is wrong (app exposes `/health`, probe checks `/healthz`), pods never become ready, never get endpoints, and the ingress has no backend to route to. Real example: A developer changed the health endpoint from `/health` to `/api/health` but didn't update the probe. All pods Running, zero Ready, all traffic 502.

2. **Endpoints** — `kubectl get endpoints <svc>`. If new pods aren't in the endpoints list yet but old pods already terminated, there's a window with zero backends → 502. This happens when readiness probes have a long `initialDelaySeconds` and old pods die before new ones are ready.

3. **Ingress controller logs** — This is the most common 502 source. During rollout, NGINX might still have the old pod IPs in its upstream list. Those pods are terminating → NGINX sends a request → gets no response → returns 502. Look for `upstream prematurely closed connection while reading response header` in NGINX logs.

4. **preStop hook** — If old pods shut down instantly, there's a race: the pod is terminating but NGINX hasn't removed it from upstreams yet. Requests to the dead pod fail. Fix:
    ```yaml
    lifecycle:
      preStop:
        exec:
          command: ["sleep", "10"]
    ```
    This keeps the old pod alive for 10 seconds while the ingress updates its upstream list. This one fix eliminated 90% of our deploy-related 502s.

5. **Resource limits** — New pods might be getting OOMKilled immediately. The pod restarts fast enough to show `Running`, but during request processing it crashes → 502. Check `kubectl describe pod <pod>` for `Last State: Terminated, Reason: OOMKilled`.

6. **App startup time vs probe timing** — If `initialDelaySeconds` is too short, the probe passes before the app is fully initialized. Traffic arrives, app returns 500/connection reset, ingress translates it to 502. We had a Spring Boot app that took 45 seconds to start. The readiness probe was set to `initialDelaySeconds: 10`. For 35 seconds after each pod start, it received traffic it couldn't handle.

**Pattern to remember:** 502s that appear exactly during rollouts and disappear 2-3 minutes later = lifecycle coordination issue. Fix with preStop hooks, proper readiness probes, and `maxUnavailable: 0`.

---

## 4. CPU is low, memory is stable, but latency is high. What metrics do you look at next and why?

Low CPU and stable memory is misleading — they're not the only bottlenecks. When the CPU is idle, the app is **waiting on something**, not running fast.

1. **CPU throttling** — Even with low average CPU, the pod can be throttled. CFS quota limits burst usage per scheduling period (100ms). Your app might need a burst of 500m CPU for 20ms to handle a request, but the limit is 200m. The kernel pauses the process, request takes 80ms instead of 20ms. Check `container_cpu_cfs_throttled_periods_total` in Prometheus. Real scenario: Our Node.js service showed 30% average CPU but 70% throttled periods. Response time was 4x higher than expected. We removed the CPU limit (kept request at 200m) and p99 latency dropped from 800ms to 150ms.

2. **Network latency to dependencies** — The app might execute in 5ms but the database query takes 200ms. CPU is idle during the network wait. Check distributed traces or `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` broken down by endpoint. We found one endpoint calling an external API with a 2-second timeout — it was slow 10% of the time, pulling the average latency up.

3. **Disk I/O** — If the app reads/writes to a PersistentVolume, slow disk = slow app. Cloud disks (especially gp2/gp3 with low provisioned IOPS) cause I/O queueing. The CPU sits idle while the app waits for disk. Check `node_disk_io_time_seconds_total`. A team ran PostgreSQL on a 100 IOPS gp2 volume — reads were queueing 200ms during peak hours.

4. **Connection pool exhaustion** — App has 10 database connections in the pool but 50 concurrent requests. 40 requests wait for a free connection. CPU is idle, memory is stable, but latency is high. Check app-level metrics for pool wait time. HikariCP (Java), pgBouncer (Postgres) — they all expose pool saturation metrics.

5. **DNS resolution time** — With `ndots:5`, every outbound request triggers up to 5 DNS lookups before the real one. Under load, CoreDNS gets overwhelmed or conntrack table fills up. Check CoreDNS latency or `coredns_dns_request_duration_seconds_bucket`.

6. **Garbage collection** — JVM/Go/Node.js GC pauses cause latency spikes while CPU averages look normal. A 200ms GC pause every 10 seconds means 2% of requests are slow. Check GC pause metrics. We had a Java service with 2Gi heap doing full GC every 30 seconds — 500ms pause each time.

**Key insight for interviews:** "Low CPU + high latency" = the app is blocked on I/O, network, locks, or connection pools. Always look at **what the thread is waiting on**, not what it's computing.

---

## 5. A service works in staging but fails in production. How do you narrow the difference without guessing?

> **Also asked as:** "One job working fine in dev environment and not working in production — how will you troubleshoot?"

I systematically diff the two environments. No guessing, no "maybe it's this."

1. **Diff the manifests** — `kubectl get deploy <name> -n staging -o yaml` vs prod. Look for: different image tags, env vars, resource limits, ConfigMaps mounted. We once spent 2 hours debugging a prod failure. The root cause: staging had `LOG_LEVEL=debug` which happened to initialize a module that prod didn't (because `LOG_LEVEL=warn` skipped it). One env var difference changed the app's behavior completely.

2. **Diff ConfigMaps and Secrets** — Staging might point to a mock payment gateway that always returns success. Prod points to the real one that requires authentication. `kubectl get cm <name> -n staging -o yaml` vs prod.

3. **Diff NetworkPolicies** — Staging often has zero NetworkPolicies. Prod has deny-all with specific allow rules. A new microservice was added as a dependency but nobody updated the NetworkPolicy to allow traffic to it. The app worked in staging (no policies) and failed in prod (blocked by default-deny). No error in app logs — the connection just timed out silently.

4. **Diff RBAC** — The ServiceAccount in staging might have `cluster-admin` (bad practice but common). Prod has scoped permissions. If the app calls the K8s API (e.g., to discover services), it works in staging and gets 403 in prod.

5. **Diff infrastructure** — Node instance types, K8s version, CNI version, ingress controller version. Staging on K8s 1.29, prod on 1.28? A deprecated API that works in 1.28 might behave differently. Different CNI plugin versions can cause subtle networking differences.

6. **Diff traffic patterns** — Staging gets 10 req/sec, prod gets 10,000. The app works fine at low load but hits connection pool limits, thread exhaustion, or external API rate limits at production scale. This one is impossible to catch without load testing staging with production-like traffic.

**My go-to command before any investigation:** `kubectl get events -n prod --sort-by=.lastTimestamp`. Events tell you what K8s is complaining about that the app might not log — failed mounts, image pull errors, scheduling failures, failed probes.

---

## 6. Logs look fine, metrics look fine, but customers report errors. What signal do you trust next?

Your observability has a blind spot. The user is always right. Here's how I find what's missing:

1. **Check per-pod, not aggregate metrics** — If 10 pods serve traffic and 1 is broken, aggregate metrics show 90% healthy. But the 10% of users hitting that pod get 100% errors. Real scenario: We had 8 healthy pods and 1 pod with a corrupted local cache. Aggregate error rate was 0.5% — below our alert threshold. But for the users hitting that pod, every request failed. We found it by checking `per-pod error rates` in Grafana instead of the service-level dashboard.

2. **Client-side errors (499s)** — If the client closes the connection before the server responds (slow response, timeout), the server might not log anything. But the user got an error. Check ingress controller access logs for 499 status codes. We found 200+ 499s per minute during peak — clients timing out after 30 seconds. Our server-side metrics showed zero errors.

3. **CDN/WAF/Load Balancer** — Errors might happen before traffic reaches your cluster. The CDN returns a cached error page, the WAF blocks requests from certain regions, the cloud LB health check fails and stops routing. None of this shows in K8s logs. Check your cloud provider's LB metrics and access logs.

4. **Downstream dependency failures** — Your service returns 200, but it's calling a downstream API that returns degraded data. The service retries silently or returns a partial response. Users see wrong data, not errors. Check dependency health and distributed traces end-to-end.

5. **DNS/network-level issues** — Users in a specific region can't resolve your domain, or their ISP has routing issues. This is completely invisible to your cluster. We had users in India unable to reach our service for 2 hours — turned out their ISP had a BGP routing issue. Our metrics were 100% green because only affected users couldn't even connect.

6. **Stale cache** — After a deploy, the app serves correct data but a CDN or in-app cache serves stale/wrong data. No errors, correct latency, but users see old information. Check cache TTLs and invalidation logic after deployments.

**The lesson I always share:** "Metrics look fine" means your metrics don't cover the right things. After every incident like this, I add the missing signal to our observability stack — per-pod error rates, 499 tracking, dependency health checks. Over time, your blind spots shrink.

---

## 7. A rollout caused partial outage. How do you decide between rollback, roll-forward, or hotfix?

Three factors: **severity**, **root cause clarity**, and **time to fix**.

**Rollback** — when:
- Outage is severe (payments failing, data loss, >30% users affected)
- You don't understand the root cause yet
- Previous version was stable
- `kubectl rollout undo deployment/<name>` — takes 30-60 seconds
- Real scenario: We deployed a new version that caused a database connection leak. Connections exhausted in 15 minutes, then 100% errors. We didn't know the root cause yet. Rollback first. Investigate after. Every minute spent debugging while users are down is a minute of lost revenue. We rolled back in 45 seconds, then spent 3 hours finding the leak in a dev environment.

**Roll-forward** — when:
- Root cause is clear AND it's a config issue, not code
- You can fix it in under 10 minutes
- Rolling back would cause its own problems (DB migration already ran, downstream services already adapted to new API)
- Real scenario: New deployment had a wrong Redis endpoint in the ConfigMap — pointed to staging Redis instead of prod. Fix: update ConfigMap, restart pods. Took 3 minutes. Faster than rollback, and rollback wouldn't help anyway because the old version had the old Redis endpoint which was being decommissioned.

**Hotfix** — when:
- Root cause is a code bug (not config)
- Fix is a 1-2 line change you're confident about
- Rollback isn't safe (breaking DB migration, new dependency versions)
- **Critical rule:** The hotfix goes through the same CI/CD pipeline. No `kubectl edit` in production. If your pipeline takes 30 minutes, choose rollback and hotfix after.
- Real scenario: A null pointer in a new API endpoint. Only that endpoint was broken, rest of the app was fine. 2-line fix, PR merged, pipeline ran in 8 minutes. We chose hotfix over rollback because rolling back would have removed 3 other features that were working fine and already being used by customers.

**My decision tree:**
```
Is it critical (payments, data, >30% users)? → ROLLBACK NOW, investigate later
Can I identify root cause in 2 minutes?
  → Yes, and it's config → ROLL-FORWARD
  → Yes, and it's code, fix is small, rollback isn't safe → HOTFIX
  → No → ROLLBACK
```

**Non-negotiable:** Whatever you choose, communicate immediately. Tell the team on Slack, update the status page, start a timeline in the incident channel. The worst outcome isn't picking the wrong option — it's 5 engineers spending 20 minutes debating while customers suffer.

---

## 8. Can you explain your hands-on experience with Kubernetes?

*(Frame your answer around problems you solved, not tools you used. Adapt with your own details.)*

I've managed production Kubernetes clusters on EKS for 3+ years. Let me walk you through what I actually do, not just the tool list.

**Cluster provisioning:** I set up EKS clusters using Terraform — VPC with public/private subnets, managed node groups with launch templates, IAM OIDC for pod-level IAM roles, and add-ons (CoreDNS, kube-proxy, VPC CNI). We run separate clusters for staging and production. Why separate clusters? We learned the hard way — a staging load test once caused node pressure that affected production pods when they shared a cluster.

**Deployments:** Everything goes through ArgoCD (GitOps). Developers merge to main, CI builds and pushes the image, updates the Helm values file, and ArgoCD syncs it to the cluster. Why we switched to GitOps: a developer once ran `kubectl apply` manually on a Friday night with an untested manifest. It broke the service. There was no record of what changed. With ArgoCD, every change is a Git commit — we can `git revert` to rollback, and we have a full audit trail.

**Scaling:** HPA on CPU + custom metrics (request rate from Prometheus adapter) for stateless services. Karpenter for node-level autoscaling — it replaced our Cluster Autoscaler and reduced node provisioning time from 3-4 minutes to under 60 seconds. Cost savings: ~30% on compute because Karpenter picks the cheapest instance type that fits pending pod requirements, including spot instances with automatic fallback to on-demand.

**Observability:** Prometheus + Grafana (metrics), Loki (logs), Tempo (distributed tracing). Dashboards follow RED method per service — Request rate, Error rate, Duration. Alerts go to PagerDuty. The specific alert that has saved us the most: "Service X has 0 endpoints for >60 seconds" — this catches selector mismatches, failed rollouts, and misconfigured probes before users notice.

**Security:** Pod Security Admission in restricted mode, NetworkPolicies with default deny (using Cilium), External Secrets Operator syncing from AWS Secrets Manager, Trivy scanning in CI, Kyverno for policy enforcement. We started with `warn` mode on PSA, identified 47 violations across 12 namespaces, fixed them over 2 sprints, then switched to `enforce`.

**Real problems I've solved:** Debugged intermittent 502s during rollouts (fixed with preStop hooks), resolved DNS failures caused by conntrack exhaustion (deployed NodeLocal DNSCache), handled data loss from autoscaler removing nodes with local storage (moved to EBS + PDB), fixed HPA not scaling because metrics-server certificate expired after cluster upgrade.

**Key tip:** In interviews, don't list tools like a resume. Pick 2-3 real problems and explain the debugging process, the root cause, and the fix. That's what separates someone who used K8s from someone who understands it.

---

## 9. If a pod loses connectivity, how do you find and fix the issue?

> **Also asked as:** "One pod not reachable to other pod in Kubernetes"

"Pod lost connectivity" could mean: pod can't reach other pods, can't reach Services, can't reach external internet, or external users can't reach the pod. I narrow it down immediately.

**First question: connectivity to what?**

```bash
# Can the pod reach another pod by IP?
kubectl exec <pod> -- curl -s <other-pod-ip>:8080

# Can it reach a Service by name?
kubectl exec <pod> -- curl -s my-service:80

# Can it reach external internet?
kubectl exec <pod> -- curl -s https://google.com
```

These three commands tell me exactly which layer is broken.

**If pod-to-pod by IP fails:**

The issue is at the CNI (Container Network Interface) level. The overlay network between nodes is broken.

- Check if the CNI pods are running: `kubectl get pods -n kube-system -l k8s-app=calico-node` (or aws-node for VPC CNI, cilium for Cilium)
- On AWS EKS with VPC CNI: check if the node ran out of ENIs (Elastic Network Interfaces). Each node type has a limit on how many IPs it can assign to pods. If you're running `t3.medium`, you get ~17 pod IPs. Pod #18 gets no IP and can't communicate. `kubectl describe node <node>` → check the `Allocatable` section for `pods` count.
- Real scenario: A new node was added to the cluster but the VPC CNI plugin couldn't assign pod IPs — the subnet was exhausted (only /24, all 254 IPs used). Pods on that node had IPs but couldn't route to pods on other nodes. Fix: added a secondary CIDR to the VPC and configured the CNI to use the new subnet.

**If pod-to-Service by name fails but pod-to-pod by IP works:**

Either DNS is broken or the Service has no endpoints.

- `kubectl exec <pod> -- nslookup my-service` — if this fails, CoreDNS is the problem (see DNS troubleshooting in medium Q1)
- `kubectl get endpoints my-service` — if empty, selector doesn't match pod labels
- If DNS resolves but connection times out — check if `kube-proxy` is running on the node: `kubectl get pods -n kube-system -l k8s-app=kube-proxy`. Broken kube-proxy = no iptables rules for Services.

**If pod-to-external internet fails:**

- Check NetworkPolicy — is there a deny-all egress policy? `kubectl get networkpolicy -n <namespace>`. This is the most common cause. A team deployed a "security hardening" manifest that included deny-all egress. All pods lost internet access. App couldn't reach S3, couldn't call external APIs, couldn't resolve external DNS.
- Check if the node has internet access — is it in a private subnet without a NAT Gateway? `kubectl exec <pod> -- curl -s https://1.1.1.1` — if even IP-based access fails, it's a VPC/routing issue, not DNS.
- Check if the pod is using a proxy — some clusters route external traffic through an HTTP proxy. Check if `HTTP_PROXY` env var is set and correct.

**If external users can't reach the pod:**

Work outside-in:
1. Is the LoadBalancer/Ingress healthy? Check cloud provider LB health checks.
2. Is the Ingress routing to the correct Service? `kubectl describe ingress <name>`
3. Does the Service have endpoints? `kubectl get endpoints <svc>`
4. Is the pod Ready? `kubectl get pods` — if 0/1 Ready, readiness probe is failing → pod is removed from endpoints → no traffic.

**Real production scenario:**

We had a partial connectivity loss — pods in namespace `orders` could reach `payments` but not `inventory`. All three namespaces had NetworkPolicies. The `inventory` namespace had a deny-all ingress policy, and the allow rule was specific: `from: [{namespaceSelector: {matchLabels: {team: platform}}}]`. The `orders` namespace had the label `team: orders`, not `team: platform`. Traffic was silently dropped. No error in app logs — just connection timeout after 30 seconds.

Fix: Updated the NetworkPolicy to allow ingress from namespaces with `team: orders` as well. Added a label convention document and a Kyverno policy that validates NetworkPolicies reference existing namespace labels.

**Debugging shortcut:** When connectivity is lost and you're not sure where, `kubectl exec <pod> -- traceroute <destination>` or `kubectl exec <pod> -- nc -zv <service> <port>` quickly tells you if the packet is even leaving the pod, reaching the node, or dying at a firewall/policy.

---

## 10. You get a 2 AM alert for high latency across multiple services. What's your first move?

The key word here is "multiple services." That means it's probably not an application bug — it's something shared: infrastructure, a common dependency, or the network layer.

**My first 60 seconds:**

1. **Check if there was a recent deployment** — `kubectl get events -A --sort-by=.lastTimestamp | head -30`. If someone deployed at 1:55 AM and latency spiked at 1:56 AM, that's your starting point. We once had a 2 AM spike because a CronJob triggered a database migration that locked a critical table. Every service waiting on that table went slow simultaneously.

2. **Check node health** — `kubectl top nodes`. If one or more nodes are at 95%+ CPU or memory, all pods on those nodes will be slow. `kubectl get nodes` — any NotReady? A single node going NotReady causes pod rescheduling, and the pods landing on remaining nodes can overload them → latency cascades.

3. **Check shared dependencies** — Is the database slow? Is Redis responding? Is the message queue backed up? Multiple services being slow simultaneously almost always points to a shared backend. `kubectl exec <any-pod> -- curl -w "%{time_total}" -s -o /dev/null http://postgres-service:5432` — even a rough connection test tells you if the DB is reachable and responsive.

4. **Check CoreDNS** — If DNS is slow, every service that makes any network call (database, other services, external APIs) slows down. `kubectl top pods -n kube-system -l k8s-app=kube-dns`. If CoreDNS pods are at memory/CPU limits, every DNS lookup queues. This causes latency across all services simultaneously — exactly the pattern described.

**Real 2 AM scenario I handled:** PagerDuty alert: "p99 latency > 2s on 5 services." I checked nodes — one `m5.2xlarge` had been terminated by AWS (spot interruption). 12 pods rescheduled to the remaining 3 nodes. Those nodes went from 60% to 90% CPU. Every pod on those nodes was getting CPU-throttled. Fix: Karpenter provisioned a new node in 45 seconds, pods rebalanced, latency normalized. Time to resolve: 3 minutes.

**What I tell the interviewer:** Don't start debugging individual services at 2 AM. Start at the infrastructure layer — nodes, DNS, shared databases. If it's affecting multiple services, the cause is almost always shared, not per-service.

---

## 11. Error rates are spiking, but CPU and memory look normal. What could be wrong?

CPU and memory being fine means the pods are healthy — the issue is in what the pods are trying to do, not in the pods themselves.

**What I check, in order:**

1. **Downstream dependency failure** — Your service is up, but the database or external API it calls is returning errors. `kubectl logs <pod> | grep -i "error\|timeout\|refused"`. Look for `ECONNREFUSED`, `ETIMEDOUT`, `5xx from upstream`. Real scenario: Error rate spiked to 30% on our order service. CPU was 15%, memory stable. Logs showed `connection refused to redis:6379`. Redis pod had been evicted by kubelet due to node memory pressure, but Redis wasn't part of the order service's monitoring dashboard. Nobody noticed Redis was down until the order service started failing.

2. **Database saturation** — Connection pool exhausted, query timeouts, deadlocks. The app has CPU headroom but threads are blocked waiting for the database. Check `kubectl logs` for `connection pool exhausted` or `lock wait timeout`. If you have database metrics in Prometheus: `pg_stat_activity_count` shows active connections, `pg_locks_count` shows lock contention.

3. **External API failures** — If your service calls Stripe, Twilio, or any third-party API, their outage = your errors. CPU looks fine because the app is just waiting for a response that never comes (or comes as 500). Check `kubectl logs` for HTTP status codes from external calls.

4. **Readiness probe flapping** — Pod passes readiness, gets traffic, then fails readiness under load, gets removed from endpoints, recovers, gets traffic again → cycle. Error rate spikes in bursts. `kubectl describe pod <pod>` — check events for `Readiness probe failed` entries.

5. **ConfigMap or Secret change** — Someone updated a ConfigMap (wrong DB endpoint, bad API key) but didn't restart the pods. Some pods picked up the new config (if mounted as volume — auto-updates in ~60s), others didn't. Inconsistent behavior = partial error rate. `kubectl get configmap <cm> -o yaml` — check if it was recently modified.

6. **Certificate expiry** — TLS certs between services or to external endpoints expired. Connections fail with TLS handshake errors. CPU and memory are irrelevant — the connection is rejected before any processing happens.

**Key insight:** "Error rate spiking with normal CPU/memory" means the pods are healthy but can't complete their work. Always look at what's happening **outside** the pod — dependencies, network, configs, certificates.

---

## 12. A critical service is flapping — up and down repeatedly. How do you stabilise it?

Flapping means the service starts, serves some traffic, crashes, restarts, serves traffic briefly, crashes again. The restart count in `kubectl get pods` is climbing steadily.

**Immediate stabilization (first 5 minutes):**

1. **Check what's killing it** — `kubectl describe pod <pod>` → `Last State`. If `OOMKilled` → the pod is running out of memory under load. Immediate fix: increase memory limits temporarily.

2. **Check liveness probe** — This is the most common cause of flapping that people miss. The liveness probe has a tight timeout (e.g., 3 seconds). Under load, the app can't respond in time, probe fails, kubelet kills the container. The container restarts, is healthy for a bit (low load), then gets traffic, gets slow, probe fails again → flapping cycle.

    Real scenario: Our Java service had a liveness probe with `timeoutSeconds: 3`. During garbage collection pauses (200ms, usually fine), the app couldn't respond. Under heavy load, GC pauses hit 4 seconds → liveness probe failed → kubelet killed the pod. 8 pods were flapping in rotation. Fix: increased `timeoutSeconds` to 10 and `failureThreshold` to 5. The flapping stopped immediately.

    Quick fix: `kubectl edit deployment <name>` — increase liveness probe `timeoutSeconds` and `failureThreshold`. This gives the app more breathing room while you investigate the root cause.

3. **Scale up before investigating** — If the service is critical, add replicas immediately: `kubectl scale deployment <name> --replicas=10`. More pods = each pod handles less traffic = less likely to hit the failure condition. This buys you time to investigate without user impact.

4. **Check logs between crashes** — `kubectl logs <pod> --previous` shows what happened right before the crash. Look for the pattern: what's the last log line before death? Is it always the same endpoint? Same error?

5. **Rollback if it started after a deploy** — `kubectl rollout history deployment/<name>` → `kubectl rollout undo deployment/<name>`. If the flapping started right after a deployment, rollback first, investigate later.

**Root cause investigation (after stabilizing):**

- If `OOMKilled` → the app has a memory leak. It works initially, memory grows over time, hits the limit, gets killed, restarts with clean memory, cycle repeats. Check memory usage over time in Grafana — if it's a steady climb, it's a leak.
- If liveness probe → the app is slow under load, not dead. Liveness probes should only check "is the process stuck" — never check dependencies or slow endpoints.
- If exit code 137 without OOMKilled → the pod was preempted or evicted. Check node-level events: `kubectl get events --field-selector involvedObject.kind=Node`.

**Key takeaway for interviews:** Stabilize first (scale up, relax probes, rollback), then investigate. Never spend 30 minutes debugging a flapping service while users are experiencing repeated failures.

---

## 13. Alerts are firing, but users aren't reporting issues. What do you do?

This is about alert hygiene — distinguishing real problems from noise. If you ignore it, you risk missing a real issue hidden behind noisy alerts. If you react to every one, you burn out.

**My approach:**

1. **Verify the alert signal** — Is the metric actually measuring something user-facing? An alert on `node_cpu_usage > 80%` doesn't mean users are affected. Nodes can run at 80% CPU and serve requests perfectly fine. But an alert on `http_error_rate > 5%` directly correlates to user experience. Check if the alert's metric maps to user impact.

2. **Check synthetic monitoring / real user metrics** — Do you have a synthetic monitor that mimics actual user flows (login → browse → checkout)? If synthetic tests pass and real users aren't complaining, the alert is likely measuring a non-user-facing signal. We had alerts firing on `pod_restart_count > 3` for a background worker that intentionally restarts after processing each batch. Technically correct alert, zero user impact.

3. **Check if it's a threshold problem** — `kubectl top pods` shows 82% CPU, alert threshold is 80%. Is 82% actually a problem? Maybe the pod is fine at 82% and the threshold was set arbitrarily. We audited our alerts and found 40% had thresholds copied from a blog post, not based on actual SLIs. We recalibrated: removed CPU-percentage alerts, replaced with latency-based alerts that directly reflect user experience.

4. **Check if the alert is stale** — The alert might reference a service that was decommissioned, a namespace that was renamed, or a metric that no longer exists. Prometheus returns "no data" which some alert rules interpret as a violation.

**What I do after:**

- If the alert is genuinely noisy → tune the threshold, add a `for: 5m` duration to prevent momentary spikes from firing, or delete the alert if it has no user-impact correlation.
- If the alert found a real but non-urgent issue → convert it to a lower severity (warning, not page). Create a ticket. Fix during business hours.
- If users aren't complaining **yet** but the signal is real → it might be a slow-building problem (memory leak, disk filling up). Acknowledge it and monitor.

**Real scenario:** We got paged at 3 AM for "CoreDNS memory usage > 70%." Users were fine. DNS was fine. But the trend showed memory growing 2% per hour. If we'd ignored it, CoreDNS would have OOMKilled at ~7 AM during peak traffic, causing a real outage. We restarted CoreDNS pods during the low-traffic window. The alert was noisy in timing but caught a real issue. We changed it to a warning at 70% and a page at 85% — giving us time to react during business hours.

**Key insight:** The goal isn't zero alerts — it's zero **irrelevant** alerts. Every alert should answer: "If this fires, what action should I take?" If the answer is "nothing," delete the alert.

---

## 14. A bad deployment is causing widespread failures. How do you respond?

**First 30 seconds — confirm and rollback:**

```bash
# Confirm the deployment is the cause
kubectl rollout history deployment/<name> -n prod
kubectl get events -n prod --sort-by=.lastTimestamp | head -20

# Rollback immediately
kubectl rollout undo deployment/<name> -n prod
```

Don't debate. Don't investigate root cause while users are down. Rollback takes 30-60 seconds. If the previous version was stable, roll back first and investigate from a position of stability.

**Next 2 minutes — verify recovery:**

```bash
kubectl get pods -n prod -w         # Watch pods come up
kubectl get endpoints <svc> -n prod  # Verify endpoints are populated
```

Check the error rate dashboard. Is it dropping? If it's not dropping after rollback, the deployment might not be the only cause — check downstream dependencies.

**Communication (within first 5 minutes):**

Post in the incident channel:
> "Widespread failures detected in [service]. Caused by deployment [version] at [time]. Rollback initiated. Monitoring recovery. Next update in 10 minutes."

Don't speculate on root cause yet. Just state: what happened, what you did, when you'll update again.

**Real scenario:** We deployed a new version of our authentication service. Within 3 minutes, every service that depended on auth started returning 401s. Error rate went from 0.1% to 45%. The auth service pods were Running and Ready — but the new code had a bug where token validation used a new signing key that wasn't synced to the other services yet.

Rollback: `kubectl rollout undo deployment/auth-service -n prod`. 40 seconds later, old version was serving. Error rate dropped to 0.1% within 2 minutes.

**Post-incident:** The developer had tested the signing key change in staging (where all services ran the same version simultaneously). In prod, the rolling update meant auth-service v2 was validating tokens while other services still used v1 signing keys. Fix: implemented the key change as a two-phase deploy — first deploy the new key to all services (they accept both old and new), then switch auth to use the new key.

**What separates a good answer:** Don't just say "I'd rollback." Explain: how you confirm the cause (events, timing correlation), the exact rollback command, how you verify recovery (endpoints, error rates), and how you communicate. That's the full incident response loop.

---

## 15. One region is failing while others are healthy. What's your approach?

**Immediate action — shift traffic:**

Before debugging, move users away from the broken region. If you're behind a global load balancer (AWS Route 53, Azure Traffic Manager, Cloudflare):

```bash
# If using Route 53 health checks, it should failover automatically
# If not, manually shift traffic:
aws route53 change-resource-record-sets --hosted-zone-id <id> \
  --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"api.example.com","Type":"A","SetIdentifier":"us-east-1","Weight":0,...}}]}'
```

Users in the failing region get routed to a healthy region. Latency is slightly higher (cross-region), but they get a working service instead of errors. This buys you time to investigate without user impact.

**Then investigate — what's different in this region?**

1. **Node health** — `kubectl get nodes --context=us-east-1-cluster`. Are nodes NotReady? A regional AZ outage can take down nodes in that AZ.

2. **Regional dependency** — The service might connect to a region-specific database, cache, or queue. If the us-east-1 RDS instance is down but the service in eu-west-1 connects to a different RDS, only us-east-1 fails. `kubectl logs <pod> --context=us-east-1-cluster` — look for connection errors to regional resources.

3. **Cloud provider issue** — Check the cloud provider's status page (status.aws.amazon.com, status.azure.com). Regional outages affect specific services (EC2, EBS, RDS) in specific AZs. During the AWS us-east-1 outage in 2023, EBS volumes in one AZ became unavailable — pods using PVCs in that AZ went Pending, but pods in other AZs were fine.

4. **Config drift** — Was a change deployed to only one region? If you deploy region-by-region (canary by region), the failing region might have a new version that others don't. `kubectl get deployment <name> -o jsonpath='{.spec.template.spec.containers[0].image}' --context=us-east-1-cluster` — compare the image tag across regions.

5. **Certificate or secret expiry** — Certificates or secrets might be region-specific. If the TLS cert in us-east-1 expired but eu-west-1 has a different cert, only one region fails.

**Real scenario:** Our us-east-1 cluster started returning 503s while eu-west-1 was 100% healthy. Same code, same config. Root cause: the NAT Gateway in us-east-1's VPC hit the connection tracking limit (hard limit on concurrent connections). All pods lost outbound connectivity — couldn't reach RDS, couldn't reach external APIs. Traffic shifted to eu-west-1 in 2 minutes via Route 53. We fixed the NAT Gateway by splitting egress across multiple NAT Gateways (one per AZ). Took 30 minutes to fix, but users experienced less than 3 minutes of degradation because of the traffic shift.

**Key insight:** Multi-region is useless without automatic or fast manual failover. If you run multi-region but can't shift traffic in under 5 minutes, you're paying for redundancy you can't use.

---

## 16. You fixed the incident, but it keeps recurring. What's next?

If it recurs, you fixed the symptom, not the root cause. Here's how I handle it:

**1. Run a proper post-incident review (within 48 hours)**

Not a blame session. A structured analysis:
- **Timeline** — What happened, minute by minute? When was it detected? When was it resolved?
- **Root cause** — Not "the pod crashed" but "why did the pod crash?" Keep asking why. Pod crashed → OOMKilled → memory limit too low → memory leak in new dependency → dependency wasn't load-tested before merge.
- **Contributing factors** — No alert for memory growth, no load testing in CI, memory limits copied from a template without measuring actual usage.

**2. Distinguish between immediate cause and systemic cause**

The immediate cause might be "Redis connection pool exhausted." The systemic cause is "we have no connection pool monitoring, no load testing, and no alerts on pool saturation." Fixing only the immediate cause means it'll recur with a different pool, different service, same pattern.

Real scenario: Our notification service crashed 3 times in 2 weeks. Each time, the on-call engineer fixed it differently — restart, scale up, increase timeout. On the 4th occurrence, I ran a proper review. Root cause: the service called an external SMTP server synchronously. When the SMTP server was slow (every few days), requests queued up, memory grew, OOMKilled. Each "fix" was a band-aid. Real fix: made SMTP calls async via a message queue. The service hasn't crashed since.

**3. Create concrete action items with owners and deadlines**

Not "improve monitoring" — that's too vague. Specific items:
- "Add alert for notification-service memory usage > 80% — Owner: Abhoy — Due: Friday"
- "Add connection pool saturation metric to Grafana dashboard — Owner: DevTeam — Due: next sprint"
- "Make SMTP calls async via SQS — Owner: Backend team — Due: Sprint 24"
- "Add load test for notification service in CI pipeline — Owner: Platform team — Due: Sprint 25"

**4. Verify the fix actually works**

Don't just deploy the fix and hope. Reproduce the failure condition in staging:
- If it was OOM → run a load test that generates the same traffic pattern
- If it was connection exhaustion → simulate the dependency being slow
- If it was a race condition → run concurrent requests at the same volume

**5. Update runbooks**

If the on-call engineer at 2 AM faces this again (before the permanent fix is deployed), they should have a clear runbook: "If notification-service OOMKills, increase memory to 1Gi and restart. Root cause is tracked in JIRA-1234."

**Key insight for interviews:** The answer isn't "write a postmortem." It's demonstrating that you understand the difference between symptoms and root causes, that you create specific action items (not vague promises), and that you verify the fix works under the same conditions that caused the failure. Recurring incidents are a sign of an immature incident response process — fixing that process is more valuable than fixing any single incident.

---

## 17. Management asks for a quick incident summary during the outage. What do you share?

This is a communication question disguised as a technical one. The interviewer wants to know if you can be clear under pressure without speculating.

**My template (30-second update):**

> **Impact:** [What users are experiencing] — "Checkout is failing for approximately 30% of users in the US region."
>
> **Cause:** [What you know, not what you guess] — "We've identified a failing database connection in the payment service. Investigating root cause."
>
> **Action:** [What you're doing right now] — "We've rolled back the last deployment and are monitoring recovery."
>
> **ETA:** [When you'll update next] — "Next update in 15 minutes."

**What NOT to do:**

- Don't speculate — "I think it might be a DNS issue" → If you're wrong, you lose credibility. Say "investigating" until you're sure.
- Don't share technical jargon — Management doesn't care about CrashLoopBackOff or iptables rules. They care about: Are users affected? When will it be fixed?
- Don't over-promise — "It'll be fixed in 5 minutes" → If it takes 30, you've failed twice (the outage + the broken promise). Say "next update in 15 minutes."
- Don't go silent — No update is worse than a "still investigating" update. Even if nothing changed, communicate that.

**Real scenario:** During a payment outage, our VP of Engineering joined the incident channel and asked "What's happening?" I responded:

> "Payment processing is down for all users since 14:32. Root cause: database service endpoint misconfiguration after a label change. Rollback was applied at 14:40 and we're seeing traffic recover. Monitoring for 15 minutes to confirm full recovery. Next update at 15:00."

Three sentences. Impact, cause, action. No speculation. He replied "Thanks" and didn't ask again until my next update.

**Key insight:** During an outage, your job is two things: fix the problem AND keep stakeholders informed. Most engineers focus only on the first one. The second one is just as important — if management doesn't hear from you, they start making their own assumptions, asking more questions, and pulling you away from the actual fix. A 30-second update every 15 minutes keeps them calm and keeps you focused.

---

## 18. War room scenario: Service is completely down, but no alerts fired, logs are green, and dashboards show healthy. How do you find the problem?

This is the hardest type of incident — everything looks fine internally but users can't reach the service. The monitoring blind spot.

**First 2 minutes — confirm the problem is real:**
- Check from outside the cluster: `curl -v https://myapp.example.com` from your laptop or an external monitoring tool (Pingdom, Uptime Robot). If it fails externally but works from inside the cluster (`kubectl exec -it <pod> -- curl localhost:8080`), the problem is between the user and the pod — not the pod itself.
- Check from a different network/region — could be a DNS issue, CDN issue, or regional outage.

**Minutes 2-5 — trace the traffic path from user to pod:**

Users hit: **DNS → CDN/WAF → Load Balancer → Ingress Controller → Service → Pod**

Any of these can break without your pod-level metrics knowing.

**1. DNS:** `dig myapp.example.com` — Is it resolving? Is it pointing to the right IP? We once had a Route53 record expire because someone deleted the hosted zone during a Terraform cleanup. The app was perfectly healthy. DNS just stopped resolving. No alerts because our health checks used the IP directly, not the domain.

**2. CDN/WAF:** If using CloudFront or Azure Front Door, check if the CDN is returning cached errors or if the WAF is blocking traffic. WAF rule updates can silently block legitimate traffic. Check WAF logs — they're separate from your app logs.

**3. Load Balancer:**
```bash
# Check target group health on AWS
aws elbv2 describe-target-health --target-group-arn <arn>
```
If targets are `unhealthy`, the LB stops forwarding traffic even though the pods are running. Common cause: security group change blocked the LB health check port, or the health check path returns 404 after a deployment.

**4. Ingress Controller:**
```bash
kubectl logs -n ingress-nginx <ingress-controller-pod> --tail=100
kubectl get ingress -A
```
Check if the Ingress resource still exists and points to the right Service. We had a case where a Helm upgrade accidentally removed an Ingress resource — the app was running, the Service existed, but no Ingress route was pointing to it. External traffic got a 404 from nginx.

**5. Service → Pod:**
```bash
kubectl get endpoints <service-name>
```
If endpoints list is empty, the Service selector doesn't match any pod labels. This happens when someone changes pod labels in a Deployment but forgets to update the Service selector. Pods are running, healthy, and logs are clean — but zero traffic reaches them.

**Why alerts didn't fire:**
- Health checks are pod-level (readiness/liveness probes check the container, not the full traffic path)
- Metrics are application-level (Prometheus scrapes `/metrics` directly from the pod, bypassing the LB/Ingress)
- Logs are generated by the app (if traffic never reaches the app, there are no error logs — the app is idle)
- Internal synthetic checks hit the Service directly (not the external URL)

**The fix for the monitoring gap:**
After this type of incident, add **external black-box monitoring** — a probe that hits the full public URL every 30 seconds from outside the cluster. If this fails, you know the user-facing path is broken, regardless of what internal metrics say. We use Prometheus Blackbox Exporter hitting the external URL, plus Pingdom as an independent external check.

**Real scenario:** Production was "down" for 22 minutes. All dashboards green. No alerts. The root cause: a new NetworkPolicy was applied to the `ingress-nginx` namespace during a security hardening sprint. It blocked the ingress controller's egress to application pods. The ingress controller was healthy (its own health check passed), pods were healthy (readiness probes passed via kubelet, not via ingress), but the actual user traffic path was broken. We only found it because a customer called. After that, we added end-to-end synthetic tests that simulate real user requests through the full external path, alerting within 60 seconds of failure.

---

## 19. A container works perfectly locally but fails in Kubernetes. How do you debug it?

This is one of the most common frustrations. "It works on my machine" but crashes or misbehaves in the cluster. The gap is almost always in one of six areas: environment, networking, permissions, resources, secrets, or config.

**Step 1: Get the exact failure**

```bash
kubectl describe pod <pod-name>     # Events, exit code, probe results
kubectl logs <pod-name> --previous  # Last crash logs
```

Look at `Exit Code` in `Last State`:
- `Exit Code: 1` — app error, check logs
- `Exit Code: 137` — OOMKilled or SIGKILL
- `Exit Code: 127` — command not found (wrong ENTRYPOINT or binary missing)
- `Exit Code: 126` — permission denied on the executable

**Step 2: Check the six common gaps**

**1. Environment variables — missing or wrong values**

Locally you have a `.env` file or shell exports. In K8s, env vars come from ConfigMaps, Secrets, or the pod spec. If any are missing, the app crashes on startup.

```bash
# Check what env vars the container actually sees
kubectl exec <pod> -- env | sort

# Common miss: DATABASE_URL locally is "localhost:5432"
# In K8s it must be "postgres-service:5432"
```

**2. Non-root user — permission denied**

Locally you run as root or your own user. In K8s with `runAsNonRoot: true`, the process may not have write access to files it expects.

```bash
kubectl exec <pod> -- whoami          # What user is the process running as?
kubectl exec <pod> -- ls -la /app     # Does the user own the files?
```

Fix in Dockerfile:

```dockerfile
COPY --chown=appuser:appuser scenario-based /app
USER appuser
```

**3. Resource limits — CPU throttling or OOMKill**

Locally you have no limits. In K8s, tight limits that don't exist on your machine will kill the container. A JVM that works fine with 2GB RAM locally will OOMKill with a 256Mi limit.

```bash
kubectl describe pod <pod> | grep -A5 "Limits\|Requests"
kubectl top pod <pod>
```

**4. Networking — service names vs localhost**

Locally: `localhost:5432` for Postgres. In K8s: `postgres-service:5432`. If the app has `localhost` hardcoded, it won't find the database in the cluster.

```bash
kubectl exec <pod> -- nc -zv postgres-service 5432
kubectl exec <pod> -- nslookup postgres-service
```

**5. File paths and mounted volumes**

In K8s, a ConfigMap or Secret mounted at `/app/config/` replaces the entire directory — wiping out any files baked into the image at that path. Locally, no mount exists so the image's files are used.

```bash
kubectl exec <pod> -- ls /app/config/         # Are the expected files there?
kubectl exec <pod> -- cat /app/config/app.yaml # Is the content correct?
```

**6. Image architecture mismatch**

You build locally on Apple M1/M2 (ARM). Your K8s nodes are x86. The container starts locally but fails in the cluster with `exec format error`.

```bash
docker inspect <image> | grep Architecture

# Build explicitly for the cluster's architecture
docker buildx build --platform linux/amd64 -t my-app:v1 .
```

**Step 3: Run the same image interactively inside the cluster**

If the above checks don't reveal the issue, run a debug pod using the exact same image in the cluster:

```bash
kubectl run debug \
  --image=my-app:v1 \
  --env="DATABASE_URL=postgres-service:5432" \
  --command -- sleep 3600

kubectl exec -it debug -- sh
# Manually run the entrypoint and watch what fails
```

This puts you inside the exact cluster environment — same network, same DNS, same RBAC — without the restart cycle interfering with your investigation.

**Real scenario:** A Node.js service worked perfectly in local Docker but crashed in K8s every time with `ENOENT: no such file or directory, open '/app/config/settings.json'`. The file existed in the Docker image. The problem: the Helm chart mounted a ConfigMap at `/app/config/` — which replaced the entire directory, wiping out `settings.json` that was baked into the image. Locally, no ConfigMap mount existed so the image's file was used fine.

Fix: changed the ConfigMap mount from a directory mount to a single file mount at `/app/config/env-settings.json`. The image's `settings.json` was preserved and the ConfigMap values were available separately. We found it by running `kubectl exec <pod> -- ls /app/config/` and seeing only one file instead of three. Took 90 minutes to debug — would have taken 5 minutes with this checklist.

---

## 20. An entire Kubernetes region goes down. How do you failover workloads?

A full region failure means every node, control plane, and managed service in that region is unavailable. The only options are pre-built: you can't build a multi-region failover during an outage. Everything here has to be set up in advance.

**Prerequisites (build these before the outage, not during):**

**1. Multi-cluster setup with ArgoCD**

Run ArgoCD in a region that's not your primary:

```yaml
# ArgoCD ApplicationSet — deploys to both clusters simultaneously
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: production
  template:
    spec:
      project: default
      source:
        repoURL: https://github.com/org/infra
        path: charts/my-app
      destination:
        server: '{{server}}'
        namespace: my-app
```

This keeps `us-east-1` and `eu-west-1` clusters in sync — same version, same config, always.

**2. Route 53 health-check-based DNS failover**

```hcl
# Primary record — us-east-1 load balancer
resource "aws_route53_record" "primary" {
  zone_id = var.zone_id
  name    = "api.company.com"
  type    = "A"

  alias {
    name    = aws_lb.us_east.dns_name
    zone_id = aws_lb.us_east.zone_id
  }

  set_identifier = "primary"
  failover_routing_policy { type = "PRIMARY" }

  health_check_id = aws_route53_health_check.primary.id
}

# Secondary record — eu-west-1 load balancer
resource "aws_route53_record" "secondary" {
  zone_id = var.zone_id
  name    = "api.company.com"
  type    = "A"

  alias {
    name    = aws_lb.eu_west.dns_name
    zone_id = aws_lb.eu_west.zone_id
  }

  set_identifier = "secondary"
  failover_routing_policy { type = "SECONDARY" }
}

resource "aws_route53_health_check" "primary" {
  fqdn              = aws_lb.us_east.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10
}
```

When the health check fails 3 times (30 seconds), Route 53 automatically routes all traffic to the secondary. No manual action.

**3. Stateless workloads — failover is automatic**

If your app is stateless (no local disk, reads config from env/ConfigMap, connects to external DB), failover is transparent — Route 53 switches DNS and the secondary cluster already has the same pods running.

**4. Stateful workloads — requires data replication**

Databases need cross-region replication before a failover is possible:

| Database | Replication mechanism |
|---|---|
| RDS | Multi-region read replica → promote to primary |
| Aurora | Global Database — failover in ~1 minute |
| DynamoDB | Global Tables — active-active, no promotion needed |
| Redis | ElastiCache Global Datastore |
| Self-managed | Velero for PVC snapshots + cross-region restore |

**Failover runbook (when region goes down):**

```bash
# 1. Confirm region is down (not a localized issue)
aws ec2 describe-availability-zones --region us-east-1

# 2. Route 53 should have already auto-failed over if health checks are configured
# Verify DNS is pointing to secondary:
dig api.company.com

# 3. If DNS hasn't auto-failed — manual override:
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123 \
  --change-batch file://manual-failover.json

# 4. Promote read replica (RDS)
aws rds promote-read-replica --db-instance-identifier my-db-eu-west-replica

# 5. Scale secondary cluster if it was running at reduced capacity
kubectl scale deployment my-app --replicas=20 -n my-app
# or trigger Karpenter scale-up via load

# 6. Communicate status to stakeholders
```

**Active-passive vs active-active:**

| Mode | RTO | RPO | Cost | Best for |
|---|---|---|---|---|
| Active-passive | 1–5 min | Minutes | ~100% extra infra | Most production apps |
| Active-active | Seconds | Near-zero | ~200% extra infra | Payment, auth, zero-tolerance |
| Backup-restore | Hours | Hours | Minimal | Non-critical apps |

**Real scenario:** Our primary EKS cluster in `us-east-1` went down during an AWS networking incident affecting the entire region (not us alone — about 40 services were impacted). Route 53 health checks detected the failure in 30 seconds. DNS automatically shifted to `eu-west-1`. The stateless API services were up in the secondary within 90 seconds of the region failure. The database took 4 minutes because we had to manually promote the RDS read replica (Aurora Global Database would have done this in ~60s automatically). Total user-facing downtime: 4.5 minutes. Post-incident: we migrated the database to Aurora Global to bring that 4 minutes down to under 90 seconds.

---

## 21. Your cluster is hit by a massive traffic surge, and HPA cannot scale pods fast enough. How do you handle it?

HPA has inherent lag: it scrapes metrics every 15 seconds, evaluates every 30–60 seconds, then waits for new pods to start and pass readiness. During a sudden surge, that delay can mean 2–5 minutes of degraded service. You need strategies that work both faster than HPA and alongside it.

**Why HPA is slow:**

```
Traffic spike → metrics server scrapes (15s lag)
             → HPA evaluates (30s default period)
             → scheduler places new pods (~5s)
             → kubelet pulls image (30s–2min if not cached)
             → readiness probe passes (10s–60s depending on app)

Total: 1–4 minutes minimum before new pods serve traffic
```

**Fix 1: Pre-warm with scheduled scaling (KEDA CronJob scaler)**

If you know traffic patterns (e.g., flash sale at 2 PM), scale proactively:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-app-scaledobject
spec:
  scaleTargetRef:
    name: my-app
  minReplicaCount: 5
  maxReplicaCount: 50
  triggers:
    - type: cron
      metadata:
        timezone: Asia/Kolkata
        start: "55 13 * * *"    # scale up 5 min before 2 PM
        end: "30 15 * * *"      # scale down at 3:30 PM
        desiredReplicas: "30"
    - type: prometheus             # also scale on actual load
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_per_second
        threshold: "500"
        query: sum(rate(http_requests_total[1m]))
```

**Fix 2: Tune HPA behavior for faster scale-up**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0    # no stabilization — scale up immediately
      policies:
        - type: Percent
          value: 100                   # double replicas every 15 seconds
          periodSeconds: 15
        - type: Pods
          value: 10                    # or add 10 pods at once
          periodSeconds: 15
      selectPolicy: Max                # use whichever adds more pods
    scaleDown:
      stabilizationWindowSeconds: 300  # slow scale-down to avoid thrashing
```

**Fix 3: Pre-pull images on all nodes**

New pods spend most startup time pulling images. Use a DaemonSet to pre-cache:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: image-prepuller
spec:
  selector:
    matchLabels:
      app: image-prepuller
  template:
    spec:
      initContainers:
        - name: prepull
          image: my-app:v1.2.0    # same image your deployment uses
          command: ["sh", "-c", "echo image pulled"]
      containers:
        - name: pause
          image: k8s.gcr.io/pause:3.9
```

Once the DaemonSet runs, the image is cached on every node. New pods start in seconds, not minutes.

**Fix 4: Use Karpenter for faster node provisioning**

Cluster Autoscaler takes 3–5 minutes to provision a new node. Karpenter takes 30–60 seconds:

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: burst-pool
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]   # spot for burst capacity
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.xlarge", "m5.2xlarge", "m6i.xlarge"]
  limits:
    cpu: "200"
  disruption:
    consolidationPolicy: WhenUnderutilized
```

**Fix 5: Circuit breaker + graceful degradation in app**

When pods are overloaded before new ones are ready, protect existing pods from being crushed:

```yaml
# Ingress rate limiting — protect backend during surge
nginx.ingress.kubernetes.io/limit-rps: "200"
nginx.ingress.kubernetes.io/limit-connections: "50"
```

Return cached responses or a "we're busy, try again" response rather than queuing indefinitely.

**Fix 6: Increase `minReplicas` permanently**

The cheapest fix: don't start from 1 replica. Start from a baseline that handles 2x normal load:

```yaml
spec:
  minReplicas: 10    # never below 10, even at 3 AM
  maxReplicas: 50
```

**Decision table:**

| Strategy | Helps with | Latency to effect |
|---|---|---|
| Scheduled pre-scaling | Known traffic events | Immediate (pre-done) |
| HPA behavior tuning | Faster reaction to metrics | 15–30 seconds |
| Image pre-pulling | Pod startup time | Immediate |
| Karpenter | Node provisioning speed | 30–60 seconds |
| Higher `minReplicas` | Always-on baseline capacity | Immediate |
| Rate limiting | Protect pods during surge | Immediate |

**Real scenario:** A product launch drove 10x normal traffic in 3 minutes. HPA was configured (CPU target 70%) but pods needed 90 seconds each to start (large image, slow readiness probe). By the time 8 new pods were ready, the original 4 pods had been throttled to the point of 30s response times. Fix applied for next launch: set `minReplicas: 15`, tuned HPA `scaleUp.stabilizationWindowSeconds: 0` and `Percent: 100` policy, pre-cached the image via DaemonSet, and pre-scaled to 30 replicas 10 minutes before the launch using a KEDA cron trigger. Next product launch: latency stayed flat throughout the traffic surge.

---
