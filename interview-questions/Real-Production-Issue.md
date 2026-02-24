# Real Production Issues

Real-world Kubernetes production issues, debugging steps, and solutions.

---

## 1. What happens if a node with local storage gets autoscaled down?

**The data is gone. Permanently.** No migration, no backup, no warning to the application.

`emptyDir`, `hostPath`, and `local` PersistentVolumes are all tied to the node's lifecycle. When the cluster autoscaler terminates that node, the disk is destroyed with the VM.

**The worst scenario I've seen:** A team ran Elasticsearch on StatefulSets with `local` PVs for NVMe performance. During a weekend low-traffic window, the autoscaler removed a node. The PV's `nodeAffinity` now pointed to a node that didn't exist. The StatefulSet tried to reschedule the pod but it was stuck in `Pending` forever — waiting for a specific node that would never come back. The ES cluster lost a shard. When a second node was removed, they lost data permanently because the replication factor was only 2.

**The sneaky part about emptyDir:** The cluster autoscaler considers `local` PVs non-evictable by default, but `emptyDir` pods **will get evicted** unless you annotate them. A team used `emptyDir` to stage files in a data pipeline. Autoscaler removed the node at 2 AM, pod restarted fresh on another node, and 6 hours of unprocessed files vanished. No errors in logs — the pod just started processing new data. They didn't notice until downstream reports showed missing records the next morning.

**How to protect:**
- Use network-attached storage (EBS, GCE PD) for anything you can't afford to lose
- PodDisruptionBudgets for stateful workloads — autoscaler respects PDBs
- `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotation on critical pods using emptyDir
- `cluster-autoscaler.kubernetes.io/scale-down-disabled=true` on nodes running stateful workloads
- Separate stateful and stateless workloads into different node pools — disable autoscaling on the stateful pool
- If using local storage for performance, ensure replication at the app level (Kafka RF=3, ES replicas) with `podAntiAffinity` to spread replicas across nodes

**Mental model:** Local storage = tied to the machine. Machine gone = data gone. Always design as if any node can disappear at any moment.

---

## 2. A node suddenly goes NotReady in Kubernetes. What are the first commands you run and why?

**Step 1:** `kubectl describe node <node-name>` — Go straight to the `Conditions` section:
- `MemoryPressure: True` → node OOM'd, kubelet started evicting pods
- `DiskPressure: True` → disk full — usually container logs or images filling `/var/lib/docker`
- `PIDPressure: True` → too many processes, possible fork bomb or process leak
- `Ready: False` → kubelet stopped heartbeating. Either kubelet crashed, node is down, or network partition

This immediately narrows the cause. In 70% of cases, the `Conditions` section tells you exactly what happened.

**Step 2:** `kubectl get events --field-selector involvedObject.name=<node-name>` — Shows the timeline: when the node became unreachable, when kubelet stopped posting status, when pods got evicted.

**Step 3:** SSH into the node and check:
- `systemctl status kubelet` + `journalctl -u kubelet --since "10 min ago"` — is kubelet running? What killed it?
- `df -h` — disk full? I've seen `/var/lib/containerd` grow to 200GB from unrotated container logs
- `dmesg | grep -i oom` — did the OOM killer kill kubelet itself?
- `free -m` — current memory state

**Real scenarios I've debugged:**

**Scenario 1 — Certificate expiry:** Self-managed cluster (kubeadm). Kubelet's client certificate expired. Kubelet couldn't authenticate to the API server, heartbeat stopped, node went NotReady. Fix: `kubeadm certs renew all` + restart kubelet. After that, we set up cert-manager for auto-rotation and alerting 30 days before expiry.

**Scenario 2 — Disk full from logs:** A Java app was printing stack traces in a loop — 50GB of logs in 4 hours. No log rotation configured on the container runtime. `/var/lib/containerd` hit 100%, kubelet couldn't function. Fix: configured containerd log rotation (`max-size: 100m`, `max-file: 3`), added disk pressure alerts at 80%.

**Scenario 3 — Network partition on AWS:** The node was alive (SSH worked via session manager) but couldn't reach the API server. Security group was modified by a Terraform run that accidentally removed the rule allowing node-to-control-plane communication. Kubelet kept running locally but API server marked the node NotReady after 5 minutes (`node-monitor-grace-period`).

**Immediate action:** If the node can't be recovered quickly and it runs critical workloads: `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`. This moves all workloads to healthy nodes. Don't wait for the node to self-heal if users are impacted.

---

## 3. Cost spiked 40% overnight. How do you identify the exact root cause?

Don't guess. Follow the money systematically — compute, then storage, then networking.

**Step 1: Node count** — `kubectl get nodes`. Did new nodes appear overnight? If yes, check autoscaler logs: `kubectl logs -n kube-system -l app=cluster-autoscaler`. Look for `ScaleUp` events and what triggered them. Real scenario: A CronJob was configured to run every minute instead of every hour (typo: `*/1` instead of `0 */1`). Each run created a pod requesting 2 CPUs. After an hour, 60 pods were pending, autoscaler added 8 new `m5.2xlarge` nodes.

**Step 2: What created the demand** — `kubectl get events --sort-by=.lastTimestamp | grep TriggeredScaleUp`. Trace back to which pods caused the scale-up. Usually it's: runaway HPA, misconfigured CronJob, or a new deployment with oversized resource requests.

**Step 3: Runaway HPA or Jobs** — `kubectl get hpa -A`. Did an HPA scale to `maxReplicas` and stay there? `kubectl get jobs -A` — are there failed jobs accumulating? We once had a Job with `backoffLimit: 6` and `restartPolicy: OnFailure` that kept failing due to a bad config. K8s kept retrying, and each retry created a new pod. 500 failed pods in 8 hours, each requesting resources.

**Step 4: Resource over-provisioning** — Compare actual usage vs requests: `kubectl top pods -A --sort-by=cpu`. If pods request 4 CPUs but use 0.2, they're wasting 95% of node capacity. This forces unnecessary scale-up. We identified a team that copy-pasted resource requests from a template: every microservice requested 2 CPUs and 4Gi memory, but most used 100m CPU and 200Mi. Fixing this freed up 12 nodes.

**Step 5: Orphaned resources** — `kubectl get pvc -A` (unused PVCs, especially gp3/io1 volumes), `kubectl get svc -A --field-selector spec.type=LoadBalancer` (each AWS NLB/ALB costs $18-25/month). We found 7 LoadBalancers from services that were deleted months ago but the LB wasn't cleaned up — that's $150/month sitting there doing nothing.

**After the incident:** We set up Kubecost for real-time cost visibility by namespace/team and added alerts for cost anomalies (>20% increase triggers PagerDuty).

---

## 4. A pod is continuously restarting and entering CrashLoopBackOff. How would you troubleshoot it?

> **Also asked as:** "How do you debug a container crash?" · "Your container keeps crashing — what do you check?" · "Pod stuck in CrashLoopBackOff error — how will you troubleshoot?" · "A pod is stuck in CrashLoopBackOff. Logs show failure during initialization — how do you troubleshoot?"

CrashLoopBackOff means: container starts → crashes → K8s restarts it → crashes again → backoff delay increases (10s, 20s, 40s, up to 5 minutes). It's not an error itself — it's K8s telling you "I keep trying but this thing keeps dying."

**Step 1:** `kubectl describe pod <pod>` — Jump to `Last State` and `State` sections:
- `Reason: OOMKilled` → container used more memory than its limit. This is the most common cause. Increase `resources.limits.memory`. Real example: A Node.js app worked fine with 256Mi in staging. In production, with higher concurrency, it used 300Mi and kept getting OOMKilled. We increased to 512Mi.
- `Exit Code: 1` → app crashed due to an unhandled error. Check logs.
- `Exit Code: 137` → killed by SIGKILL. Either OOM (kernel killed it before K8s could) or the pod was preempted.
- `Exit Code: 127` → "command not found." Wrong `CMD` or `ENTRYPOINT` in Dockerfile. We had this when a dev used a multi-stage build but forgot to copy the binary to the final stage.

**Step 2:** `kubectl logs <pod> --previous` — This shows logs from the **last crashed** container, not the current one. This is where you find the actual error: `ECONNREFUSED` (can't reach database), `FileNotFoundException` (missing config file), `permission denied` (running as non-root but writing to root-owned directory).

**Step 3:** If logs are empty (container dies before logging):
- Debug interactively: `kubectl run debug --image=<same-image> --command -- sleep 3600`, then `kubectl exec -it debug -- sh`. Run the entrypoint manually and see what happens.
- Check if init containers are failing: `kubectl describe pod <pod>` — look at `Init Containers` section. If an init container can't complete (waiting for a dependency, wrong permissions), the main container never starts.

**Step 4:** Dependency issues — the app starts but immediately fails because it can't connect to a database, Redis, or config server at boot time. The app exits, K8s restarts it, same failure. Fix: add retry logic in the app's startup, or use init containers to wait for dependencies:
```yaml
initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres-service 5432; do sleep 2; done']
```

**Most common causes in my experience:** OOMKilled (65% of cases), missing env vars/ConfigMaps (20%), wrong command/entrypoint (10%), dependency unreachable (5%).

---

## 5. How do you detect and fix OOMKilled pods in production?

OOMKilled means the container used more memory than its `resources.limits.memory`. The Linux kernel's OOM killer terminates the process — no graceful shutdown, no SIGTERM, just instant death.

**Detection:**
```bash
# Quick check — shows last termination reason
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].lastState.terminated.reason}{"\n"}{end}'

# Detailed check on a specific pod
kubectl describe pod <pod-name>
# Look for: Last State: Terminated - Reason: OOMKilled, Exit Code: 137
```

Set up alerts — don't rely on manual checks:
```promql
# Prometheus alert for OOMKilled
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} > 0
```

**Root cause analysis — don't just increase the limit blindly:**

**1. Check actual memory usage vs limit:**
```bash
kubectl top pod <pod-name>
# Compare with the limit in the pod spec
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].resources.limits.memory}'
```

**2. Memory leak vs genuine need:**
- If memory grows steadily over hours/days and then OOMs → **memory leak**. Increasing the limit just delays the crash. Fix the app.
- If memory spikes during specific operations (batch job, report generation, image processing) → **genuine peak demand**. Increase the limit.

**3. JVM-specific trap:** Java apps with `-Xmx512m` in a container with `limits.memory: 512Mi` will OOM. The JVM needs memory beyond heap — thread stacks, metaspace, native memory, GC overhead. Rule of thumb: set container limit to **1.5x the heap size**. `-Xmx512m` → `limits.memory: 768Mi`.

**Real scenario:** A Python data processing service OOMKilled every day at 3 AM during a scheduled batch job. Memory limit was 1Gi. We checked `kubectl top` during the job — it peaked at 980Mi. The batch job loaded a 500MB CSV into a pandas DataFrame, which doubled in memory during processing. Fix wasn't increasing the limit — it was switching to chunked processing (`pd.read_csv(chunksize=10000)`). Memory usage dropped to 200Mi during the batch job. The real fix is always understanding *why* the memory is high, not just throwing more at it.

**Prevention:**
- Always set memory limits (without limits, a leaking pod consumes the entire node's memory and causes evictions of other pods)
- Set requests = limits for critical services (guarantees QoS class `Guaranteed` — last to be evicted)
- Use VPA in `Off` mode to get recommendations without auto-restarts

---

## 6. Image pull is failing — how do you triage `ImagePullBackOff`?

> **Also asked as:** "A Pod is stuck in ImagePullBackOff. How do you troubleshoot?"

`ImagePullBackOff` means K8s tried to pull the container image and failed. It backs off exponentially (10s, 20s, 40s... up to 5 minutes) and keeps retrying.

**Step 1: Get the exact error**
```bash
kubectl describe pod <pod-name>
# Look at Events section — the error message tells you everything
```

**Common errors and fixes:**

**`ErrImagePull: unauthorized`** — Authentication issue. The node can't authenticate to the container registry.
```bash
# Check if imagePullSecrets is set on the pod/ServiceAccount
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets}'

# Verify the secret exists and has valid credentials
kubectl get secret <secret-name> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```
On EKS with ECR: check the node's IAM role has `ecr:GetDownloadUrlForLayer` permission. On AKS with ACR: verify `az aks check-acr --name myCluster --resource-group myRG --acr myregistry`.

**`manifest unknown` or `not found`** — The image tag doesn't exist.
```bash
# Verify the image exists in the registry
docker manifest inspect <image>:<tag>
# Or for ECR:
aws ecr describe-images --repository-name my-app --image-ids imageTag=v1.2.3
```
Common cause: CI built and pushed `v1.2.3` to the staging ECR but not the production ECR. Or someone used `latest` tag and it was overwritten.

**`context deadline exceeded` or `i/o timeout`** — Network issue. The node can't reach the registry.
- Check if the node has internet access (for Docker Hub/public registries)
- Check VPC security groups — outbound HTTPS (443) to the registry must be allowed
- If using a VPC endpoint for ECR, verify the endpoint is correctly configured
- Check if a proxy is required but not configured on the container runtime

**`ImagePullBackOff` only on new nodes** — Usually a missing `imagePullSecret`. The secret exists in the namespace, but the new node's kubelet doesn't have registry credentials. If using ECR, the ECR credential helper token expires every 12 hours — if node provisioning is slow, the cached token might be stale.

**Real scenario:** After migrating from Docker Hub to a private ECR, one team's pods kept hitting `ImagePullBackOff`. The image existed in ECR, credentials were valid. After 45 minutes of debugging, we found the Dockerfile referenced a base image from Docker Hub (`FROM python:3.11`). The build succeeded locally (Docker Hub login cached), but the K8s node couldn't pull the base layer because we'd blocked Docker Hub at the network level. Fix: changed the base image to our ECR mirror of `python:3.11`.

**Prevention:** Use `AlwaysPullPolicy` only when needed (it adds latency to every pod start). For production, use immutable tags (never `:latest`) and `IfNotPresent` pull policy.

---

## 7. HPA caused a cost explosion in staging — what went wrong and how do you prevent it?

This happens more often than people admit. HPA is designed to scale up aggressively — that's its job. But without guardrails, it can burn through your cloud budget.

**What typically goes wrong:**

**1. maxReplicas set too high (or not set at all):**
A developer set `maxReplicas: 100` on a staging service "just in case." A load test ran over the weekend, HPA scaled to 100 pods, each requesting 2 CPUs and 4Gi memory. The cluster autoscaler added 25 new nodes to accommodate them. Nobody noticed until Monday when the AWS bill alert fired — $2,000 in compute over 2 days for a staging environment.

**2. Custom metrics misfiring:**
HPA scaling on a Prometheus metric (queue depth). The metric source had a bug — reported queue depth as 10,000 when the actual queue was empty. HPA scaled to `maxReplicas` and stayed there. Cost ran for a week before someone noticed.

**3. Scale-down too slow:**
Default `scaleDown.stabilizationWindowSeconds` is 300 seconds (5 minutes). But if traffic spikes every 4 minutes (health checks, synthetic monitoring), HPA never scales down — the stabilization window keeps resetting. Pods stay at peak count 24/7.

**How to prevent:**

**Set sensible maxReplicas per environment:**
```yaml
# staging values
maxReplicas: 10   # Not 100

# production values
maxReplicas: 50
```

**Resource quotas on staging namespaces:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    pods: "50"
```
Even if HPA wants 100 pods, the ResourceQuota caps it at 50. This is the safety net.

**Cluster autoscaler limits:**
```bash
# Set max nodes on the staging node group
--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled...
# ASG max size = 10 nodes for staging
```

**Cost alerts:**
Set up billing alerts at 50%, 80%, and 100% of the staging budget. We use AWS Budgets with SNS → Slack integration. If staging spend exceeds $500/day, the on-call gets paged.

**Real scenario:** Our staging HPA was configured identically to production (`maxReplicas: 50`). A QA engineer ran a load test with 10x production traffic to "stress test." HPA scaled to 50 replicas, cluster autoscaler added 12 nodes. Staging cost that day: $800 (normally $50/day). After that, we: (1) set staging `maxReplicas: 10`, (2) added ResourceQuotas, (3) set the staging ASG max to 5 nodes, (4) added a daily cost alert. The load test would now be capped and the engineer would know immediately when they hit the ceiling.

---

## 8. Cluster nodes are under memory pressure. What actions do you take?

When nodes hit memory pressure, kubelet starts evicting pods to protect the node itself — first `BestEffort` pods (no requests/limits), then `Burstable` (requests < limits), then `Guaranteed` (requests = limits). If you don't act fast, the eviction cascade can take down healthy workloads.

**Step 1: Confirm memory pressure and identify the scope**

```bash
# Check node conditions
kubectl get nodes
# Look for: MemoryPressure = True

kubectl describe node <node-name>
# Check Conditions section:
# MemoryPressure   True    kubelet has insufficient memory available

# See which pods are already evicted
kubectl get pods -A | grep Evicted

# Current memory usage per node
kubectl top nodes
```

**Step 2: Find what's consuming the memory**

```bash
# Which pods are using the most memory on that node
kubectl top pods -A --sort-by=memory | head -20

# Which pod is eating memory — check if it's growing (leak) or just large
# Get pod's node assignment
kubectl get pods -A -o wide | grep <node-name>
```

**Step 3: Immediate actions — stop the bleeding**

**Option A: Drain the node** — if one node is under pressure, move its pods elsewhere:
```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
This evicts pods gracefully (respecting PDBs and terminationGracePeriod) and reschedules them. Don't use this if all nodes are under pressure — you'll just move the problem.

**Option B: Kill the memory hog** — if a specific pod is clearly leaking:
```bash
kubectl delete pod <pod-name> -n <namespace>
# Deployment controller recreates it — fresh start with empty memory
```

**Option C: Emergency memory — free up system memory on the node** (last resort):
```bash
# SSH to the node
sync && echo 3 > /proc/sys/vm/drop_caches   # Drops filesystem page cache
```
This frees cached memory that the kernel holds but doesn't actively need. Temporary relief, not a fix.

**Step 4: Root cause — why did memory pressure occur?**

**1. Memory leak in application:**
Memory grows steadily over hours/days → hits node capacity → pressure. Check container-level memory trend in Grafana:
```promql
container_memory_working_set_bytes{pod=~"my-app.*"}
```
If the graph is a steady upward slope with no plateau, it's a leak. Fix: find the leak, set `limits.memory` to trigger OOMKill before it takes down the node.

**2. Missing memory limits:**
Pod has no `limits.memory` — it can consume all node memory without bound. K8s won't kill it until the node is in distress. Fix: always set memory limits. Enforce it with a LimitRange:
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        memory: 512Mi     # Applied if no limit is specified
      defaultRequest:
        memory: 256Mi
      max:
        memory: 2Gi       # Hard ceiling per container
```

**3. Over-scheduling — too many pods per node:**
Scheduler places pods based on `requests`, not actual usage. If actual usage > requests, nodes get oversubscribed. A node with 8Gi memory can have 10 pods each requesting 512Mi (total requests: 5Gi) but actually using 900Mi each (total actual: 9Gi) → memory pressure.

Fix: set requests to match actual p95 usage (use VPA recommendations), not a low number to squeeze more pods onto each node.

**4. JVM / container runtime overhead:**
Java apps with `-Xmx2g` use 2Gi heap + metaspace + GC overhead + thread stacks. Total process memory is typically 1.5-2x heap. Set limits accordingly: `-Xmx2g` → `limits.memory: 3Gi`.

**Step 5: Prevention — alerts before pressure hits**

```yaml
# Prometheus alert — fire at 80% to give time to react before evictions start
- alert: NodeMemoryHigh
  expr: |
    (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
    / node_memory_MemTotal_bytes > 0.80
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Node {{ $labels.node }} memory > 80%"

# Alert on evicted pods immediately
- alert: PodEvicted
  expr: kube_pod_status_reason{reason="Evicted"} > 0
  for: 0m
  labels:
    severity: critical
```

**Real scenario:** At 2 AM, 6 pods were evicted across 3 nodes — all from the same namespace. `kubectl top nodes` showed 3 nodes at 95%+ memory. `kubectl top pods` showed one pod using 6Gi with no limit set — it was a data processing job that loaded an entire dataset into memory. It had been running fine for months but the dataset grew past the node's available memory. No alert fired because we had no node memory alert, only pod-level alerts.

Immediate fix: deleted the job pod, memory freed, evicted pods rescheduled. Permanent fix: added memory limits to the job (`limits.memory: 4Gi`), switched the job to stream data in chunks instead of loading it all at once (peak memory dropped from 6Gi to 400Mi), and added node memory alerts at 75% and 85%.

---

## 9. Pod is not scheduled due to resource constraints. How do you handle it?

> **Also asked as:** "Helm deployment fails due to insufficient cluster resources — what's your approach?"

A pod stuck in `Pending` due to resource constraints means the scheduler looked at every node and couldn't find one that can satisfy the pod's requests. The pod exists as an API object but no node has accepted it. It will stay `Pending` indefinitely — K8s will not schedule it until something changes.

**Step 1: Confirm the cause**

```bash
kubectl describe pod <pod-name>
# Jump to Events section at the bottom — the scheduler always leaves a reason:
# "0/5 nodes are available: 5 Insufficient cpu, 3 Insufficient memory"
# "0/5 nodes are available: 5 node(s) had untolerated taint"
# "0/5 nodes are available: 5 node(s) didn't match Pod's node affinity"
```

The Events section tells you exactly which constraint failed and on how many nodes.

**Step 2: See actual available capacity**

```bash
# Current resource usage per node
kubectl top nodes

# Allocatable vs requested per node (what the scheduler sees)
kubectl describe nodes | grep -A5 "Allocatable\|Allocated resources"
```

The scheduler uses `requests`, not actual usage. A node can be at 20% actual CPU but 95% requested — the scheduler sees it as nearly full and won't place new pods there.

**Step 3: Identify the specific constraint and fix it**

**Cause 1 — Cluster is genuinely full (most common)**

All nodes are at or near their allocatable capacity. No room for the new pod.

Fixes:
```bash
# Option A: Scale the node group (if cluster autoscaler is configured)
# The autoscaler should do this automatically — check its logs:
kubectl logs -n kube-system -l app=cluster-autoscaler | tail -50
# Look for: "Scale-up: setting group size to X"

# Option B: Manually increase the node group on EKS
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --scaling-config minSize=3,maxSize=15,desiredSize=8

# Option C: Add a new node manually (self-managed)
# Provision a new node and join it to the cluster
```

**Cause 2 — Pod requests are too high**

The pod requests more CPU or memory than any single node can provide. Even on an empty cluster, no node is big enough.

```bash
# Check what the pod is requesting
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources.requests}'

# Check the largest allocatable on any node
kubectl get nodes -o custom-columns=\
"NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory"
```

If the pod requests `cpu: 8` and your largest node only has `7` allocatable CPU, it will never schedule. Fix: reduce the resource request to match actual need (use `kubectl top` from a previous run to check real usage).

**Cause 3 — ResourceQuota exhausted in the namespace**

The namespace has a `ResourceQuota` and the existing pods have already consumed the full allocation.

```bash
kubectl describe resourcequota -n <namespace>
# Shows: Used vs Hard limit
# Example:
# requests.cpu    3900m / 4000m   ← only 100m left, new pod needs 500m → blocked
```

Fix: either increase the quota (requires cluster admin), or reduce requests on existing pods, or delete unused pods in the namespace.

```yaml
# Increase quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: payments
spec:
  hard:
    requests.cpu: "8"         # increased from 4
    requests.memory: 16Gi     # increased from 8Gi
    pods: "40"
```

**Cause 4 — LimitRange sets a minimum higher than the pod's request**

A `LimitRange` in the namespace enforces a minimum request. If the pod specifies less (or nothing), K8s injects the minimum — which may be too large to fit.

```bash
kubectl describe limitrange -n <namespace>
# Shows min/max/default for containers
```

Fix: either update the pod to meet the minimum, or adjust the LimitRange.

**Cause 5 — Node selector or affinity with no matching nodes**

```bash
kubectl describe pod <pod> | grep -A5 "Node-Selectors\|Affinity"
# "Node-Selectors: disk=ssd"

kubectl get nodes --show-labels | grep disk=ssd
# No output — no node has this label
```

Fix: label the correct node, or remove/update the nodeSelector if it was applied by mistake.

```bash
kubectl label node <node-name> disk=ssd
```

**Cause 6 — Taint on all nodes with no matching toleration**

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
# All nodes show taint: gpu=true:NoSchedule

# Pod has no toleration for it
kubectl get pod <pod> -o jsonpath='{.spec.tolerations}'
# empty
```

Fix: add the toleration to the pod spec, or remove the taint from the node if it was applied incorrectly.

**Prevention — alert before users notice:**

```yaml
# Prometheus alert — pod stuck Pending for more than 5 minutes
- alert: PodPendingTooLong
  expr: kube_pod_status_phase{phase="Pending"} == 1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Pod {{ $labels.pod }} has been Pending for > 5 minutes"
```

**Real scenario:** After a new microservice was onboarded, its pods sat `Pending` for 20 minutes. `kubectl describe pod` showed: `0/6 nodes are available: 6 Insufficient memory`. The new service had copy-pasted resource requests from a heavy batch job — `requests.memory: 4Gi` per pod, 3 replicas = 12Gi total. Our nodes were `m5.xlarge` with 16Gi total and ~12Gi allocatable (4Gi used by system + existing pods). No single node had 4Gi free.

The service actually needed 400Mi based on load testing. We corrected the requests to `memory: 512Mi`, and all 3 pods scheduled immediately on existing nodes — no new nodes needed. The root cause was copy-pasted resource requests with no measurement behind them.

---

## 10. Pod is in CrashLoopBackOff but logs are empty. Where do you start?

Empty logs mean the container is dying before it can write anything to stdout. Standard log debugging is useless — you need to work around the container's own output.

**Step 1: Get the exit code — it tells you the category of failure**

```bash
kubectl describe pod <pod-name>
# Look at Last State section:
# Last State: Terminated
#   Reason:    Error
#   Exit Code: 127     ← command not found
#   Exit Code: 137     ← OOMKilled or SIGKILL
#   Exit Code: 126     ← permission denied on binary
#   Exit Code: 1       ← generic app error (before any logging started)
#   Exit Code: 139     ← segmentation fault
```

Exit code alone narrows the cause significantly before you do anything else.

**Step 2: Check events and init containers**

```bash
kubectl describe pod <pod-name>
# Events section — K8s-level failures that happen before the app starts:
# - OOMKilled before first log line
# - Failed to mount volume
# - Failed to pull image layer
# - Init container failed

# Init containers have their own logs — people forget this
kubectl logs <pod-name> -c <init-container-name>
kubectl logs <pod-name> -c <init-container-name> --previous
```

**Step 3: Run the same image interactively**

```bash
# Run the exact same image but override the entrypoint with sleep
kubectl run debug-pod \
  --image=<same-image-as-failing-pod> \
  --env="VAR1=value1" \
  --command -- sleep 3600

kubectl exec -it debug-pod -- sh

# Now manually run the entrypoint inside the shell
# Watch what error appears — it will be visible in your terminal
/app/start.sh
# → bash: /app/start.sh: Permission denied   (exit code 126)
# → /app/start.sh: line 5: db-migrate: command not found  (exit code 127)
```

This bypasses the restart cycle and gives you interactive access to the exact failure.

**Step 4: Check for OOMKill before first log line**

If `Exit Code: 137` and logs are empty — the container was killed by the OOM killer before the app printed anything. This happens when the JVM or runtime itself exceeds the limit during startup.

```bash
# Check if memory limit is too tight for the runtime to even start
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].resources.limits.memory}'

# Check node-level OOM events
kubectl get events --field-selector reason=OOMKilling
dmesg | grep -i "oom\|killed process"   # On the node via SSH
```

Fix: increase `limits.memory`. For JVMs, startup memory (loading classes, JIT compilation) peaks before the app is ready to log.

**Step 5: Check volume mount failures**

```bash
kubectl describe pod <pod-name>
# Events:
# MountVolume.SetUp failed for volume "config" : configmap "app-config" not found
# MountVolume.SetUp failed for volume "certs" : secret "tls-certs" not found
```

If a required volume can't be mounted, the container never starts — zero logs. The Events section catches this.

**Common causes by exit code:**

| Exit Code | Likely cause | First check |
|---|---|---|
| 137 | OOMKill or SIGKILL | Memory limit, `dmesg` on node |
| 127 | Binary not found | Wrong CMD/ENTRYPOINT, missing binary in image |
| 126 | Permission denied on binary | `chmod +x` missing, non-root user |
| 139 | Segfault | Corrupt binary, architecture mismatch (ARM vs x86) |
| 1 | Generic error before logging | Run interactively, check volume mounts |

**Real scenario:** A new service entered CrashLoopBackOff on first deploy. Exit code 1, zero logs. The `kubectl run debug` approach revealed it immediately: the entrypoint script tried to source `/etc/app/env.sh` which was mounted from a ConfigMap — but the ConfigMap key was named `env.sh` and the mount was at `/etc/app/` but the file had Windows line endings (`\r\n`). Bash couldn't parse the script. The error was `$'\r': command not found` — which only appeared when running interactively. This would have been invisible in normal CrashLoopBackOff debugging.

---

## 11. Cluster Autoscaler is not scaling up despite Pending pods. What do you check first?

Pending pods that don't trigger a scale-up is a deceptive problem — the autoscaler is running but silently deciding not to act. The reason is almost always one of five causes.

**Step 1: Check autoscaler logs directly**

```bash
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100 | grep -i "scale\|pending\|cannot\|skip"
```

The autoscaler logs its decision for every pending pod. It will tell you exactly why it isn't scaling.

**Step 2: The five causes and how to identify each**

**Cause 1 — Pod has a `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotation AND is blocking a node from scale-down, not scale-up.** *(Not your issue here, but common confusion.)*

**Cause 2 — No node group can satisfy the pod's requirements**

The autoscaler only adds nodes from existing node groups. If your node group uses `m5.large` (2 CPU, 8GB RAM) but the pod requests 4 CPU, no new `m5.large` node will ever satisfy it. The autoscaler won't create a different instance type.

```bash
# What is the pod requesting?
kubectl describe pod <pending-pod> | grep -A5 "Requests"

# What instance types are in your node groups?
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name general \
  --query 'nodegroup.instanceTypes'
```

Fix: add a node group with a larger instance type, or reduce the pod's resource requests.

**Cause 3 — Node group is already at maxSize**

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names <asg-name> \
  --query 'AutoScalingGroups[0].{Max:MaxSize,Desired:DesiredCapacity}'

# Autoscaler log will show:
# "Skipping scale-up: node group <name> is at max size (10)"
```

Fix: increase `maxSize` on the ASG or in the autoscaler config.

**Cause 4 — Pod has constraints no node in the group satisfies**

NodeSelector, affinity rules, or tolerations that don't match any node in any node group:

```bash
kubectl describe pod <pending-pod>
# Node-Selectors: gpu=true
# But no node group has nodes with this label

# Autoscaler log:
# "Pod <pod> can't be scheduled on any node group"
```

Fix: add a node group that matches the pod's constraints, or fix the pod's constraints.

**Cause 5 — Autoscaler expander is set to `least-waste` and decides no expansion is needed**

The autoscaler calculates bin-packing efficiency. If it determines that the pending pod could theoretically fit on an existing node after other pods are shuffled — it won't scale up even if practically no node can accept the pod right now.

```bash
# Check expander setting
kubectl get deployment cluster-autoscaler -n kube-system -o yaml | grep expander
# --expander=least-waste   ← might cause this

# Switch to priority or random expander for more predictable behavior
--expander=priority
```

**Cause 6 — Scale-up cooldown is active**

After a recent scale-up, autoscaler waits before scaling again (default `--scale-down-delay-after-add=10m`). There is also a scale-up stabilization. Check if a scale-up happened recently.

```bash
kubectl logs -n kube-system -l app=cluster-autoscaler | grep "ScaleUp"
# "Scale-up: group was scaled up recently, cooldown period not expired"
```

**Verify once cause is fixed:**

```bash
# Watch autoscaler decisions in real time
kubectl logs -n kube-system -l app=cluster-autoscaler -f | grep -i scale

# Watch new node appear
kubectl get nodes -w
```

**Real scenario:** After enabling a new namespace with GPU-bound ML workloads, all pods sat `Pending`. The autoscaler was running. Logs showed: `"No node group found for pod — all node groups have unsatisfied requirements"`. The GPU pods had `nodeSelector: hardware=gpu` but the GPU node group in the autoscaler config had nodes labeled `accelerator=nvidia`. The autoscaler correctly concluded that no node group could satisfy the pods — so it didn't scale. Fix: standardized the label to `hardware=gpu` across both the node group launch template and the pods. Autoscaler immediately triggered a scale-up and 2 GPU nodes were provisioned within 3 minutes.

---

## 12. Your Ingress controller crashes repeatedly under heavy load. How do you stabilize it?

Ingress controller crashes under load usually mean one of three things: resource exhaustion (OOMKilled), connection backlog overflow, or misconfigured worker settings. Treat the controller like a production workload — it needs resource limits, HPA, PDB, and tuning.

**Step 1: Identify the crash cause**

```bash
kubectl get pods -n ingress-nginx
# NAME                                       READY   STATUS      RESTARTS
# ingress-nginx-controller-xxx               0/1     OOMKilled   14

kubectl describe pod -n ingress-nginx ingress-nginx-controller-xxx
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

**Step 2: Increase resource limits (most common fix)**

Default Helm install of ingress-nginx gives the controller very conservative limits. Under production load, 256Mi is never enough.

```yaml
# values.yaml for ingress-nginx
controller:
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: "2"
      memory: 2Gi
```

**Step 3: Scale horizontally with HPA**

One ingress controller pod is a single point of failure. Run at least 2, scale with HPA:

```yaml
controller:
  replicaCount: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80
```

**Step 4: Add PodDisruptionBudget**

Prevents all replicas from being evicted simultaneously during node maintenance:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ingress-nginx-pdb
  namespace: ingress-nginx
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
```

**Step 5: Tune NGINX worker settings**

Under high concurrency, default worker settings saturate quickly:

```yaml
# ConfigMap for ingress-nginx
data:
  worker-processes: "auto"           # one worker per CPU core
  max-worker-connections: "65536"    # per worker connection limit
  keep-alive: "75"                   # keepalive timeout (seconds)
  keep-alive-requests: "1000"        # max requests per keepalive connection
  upstream-keepalive-connections: "200"
  proxy-connect-timeout: "10"
  proxy-send-timeout: "60"
  proxy-read-timeout: "60"
```

**Step 6: Enable rate limiting to protect under surge**

```yaml
# Ingress annotation
nginx.ingress.kubernetes.io/limit-rps: "100"
nginx.ingress.kubernetes.io/limit-connections: "20"
```

**Step 7: Spread across nodes with anti-affinity**

```yaml
controller:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
          topologyKey: kubernetes.io/hostname
```

This guarantees no two controller pods land on the same node — one node crash can't take out the entire ingress layer.

**Root cause comparison:**

| Symptom | Root cause | Fix |
|---|---|---|
| OOMKilled | Memory limit too low | Increase `limits.memory` |
| High CPU throttle | CPU limit too low | Increase `limits.cpu` |
| 502/504 under load | Too few workers or replicas | HPA + worker tuning |
| All pods go down at once | No PDB, no anti-affinity | Add PDB + podAntiAffinity |
| Spikes overwhelm upstream | No rate limiting | Add `limit-rps` annotations |

**Real scenario:** Our ingress-nginx controller was handling 8,000 RPS normally but crashed every time a flash sale hit 25,000 RPS. OOMKilled every time — memory limit was 256Mi. We increased memory to 2Gi, enabled HPA (2→8 replicas), added podAntiAffinity across 3 AZs, and set `worker-processes: auto`. The next flash sale hit 31,000 RPS — controller scaled to 6 replicas, zero crashes, p99 latency stayed under 180ms.

---

## 13. All pods in one namespace suddenly fail readiness checks. What do you do?

When all pods in a namespace fail readiness simultaneously, the cause is almost never inside the pods — it's something shared across the namespace: a dependency, a config change, or a network/DNS issue. Don't restart pods first. Investigate the shared layer.

**Step 1: Confirm the scope**

```bash
kubectl get pods -n my-namespace
# All 0/1 READY but STATUS is Running — readiness probe failing, not crashing

kubectl describe pod <any-pod> -n my-namespace
# Events:
# Readiness probe failed: HTTP probe failed with statuscode: 503
# or: dial tcp: i/o timeout
# or: Get "http://10.0.0.5:8080/health": context deadline exceeded
```

**Step 2: Check what changed recently**

```bash
# Recent events in the namespace
kubectl get events -n my-namespace --sort-by='.lastTimestamp' | tail -30

# Recent ConfigMap or Secret changes
kubectl get configmaps -n my-namespace -o yaml | grep -A5 "last-applied"

# Recent deployments
kubectl rollout history deployment -n my-namespace
```

**Step 3: Test the readiness probe endpoint manually**

```bash
# Exec into one of the failing pods
kubectl exec -n my-namespace <pod> -- curl -v http://localhost:8080/health

# If probe is hitting another service:
kubectl exec -n my-namespace <pod> -- curl -v http://downstream-service:8080/health
```

**Step 4: Check downstream dependencies**

Most "all pods fail readiness" incidents are caused by a dependency going down — database, cache, external API:

```bash
# Is the database service reachable from within the namespace?
kubectl exec -n my-namespace <pod> -- nc -zv postgres-service 5432
# or
kubectl exec -n my-namespace <pod> -- curl http://redis-service:6379

# Check if the dependency itself is healthy
kubectl get pods -n database-namespace
```

**Step 5: Check DNS resolution**

A CoreDNS issue or NetworkPolicy change can break service discovery:

```bash
kubectl exec -n my-namespace <pod> -- nslookup downstream-service
kubectl exec -n my-namespace <pod> -- nslookup downstream-service.other-namespace.svc.cluster.local

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Step 6: Check NetworkPolicy changes**

If a NetworkPolicy was recently added/changed, it might be blocking the readiness probe path or the backend connection:

```bash
kubectl get networkpolicy -n my-namespace
kubectl describe networkpolicy -n my-namespace
```

**Step 7: Check if a ConfigMap/Secret changed**

```bash
# Did a config change that broke the health endpoint?
kubectl rollout undo deployment/<app> -n my-namespace
# See if readiness recovers — if yes, the last config change was the cause
```

**Common root causes and fixes:**

| Root cause | Signal | Fix |
|---|---|---|
| Downstream DB/cache down | Probe fails with connection refused | Fix dependency, add circuit breaker |
| Bad ConfigMap/Secret update | All pods affected after deploy | `kubectl rollout undo` |
| NetworkPolicy too restrictive | DNS lookup fails, connection timeout | Audit and fix NetworkPolicy |
| Readiness probe path changed | 404 on health endpoint | Sync probe path with new app endpoint |
| CoreDNS overloaded | DNS timeouts | Scale CoreDNS, check ndots setting |
| Quota exhausted | New pods can't start | Check `kubectl describe resourcequota -n my-namespace` |

**Real scenario:** All 12 pods in our `payments` namespace failed readiness at 2:47 AM. No deployment had happened. First check: `kubectl get events` showed nothing unusual. Second check: exec into a pod and curl the health endpoint — got `503 Service Unavailable`. The health endpoint called Redis for a cache check. Third check: `kubectl get pods -n caching` — Redis was in `Pending` (node had been drained for maintenance and the PVC couldn't bind on the new node). Payments pods were healthy; Redis was the root cause. Fix: resolved the PVC binding issue, Redis came up, all 12 pods passed readiness within 90 seconds. No restart needed.

---
