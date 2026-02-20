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

> **Also asked as:** "How do you debug a container crash?" · "Your container keeps crashing — what do you check?"

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
