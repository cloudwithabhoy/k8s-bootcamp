# Basic Interview Questions

Beginner-level Kubernetes interview questions and answers.

---

## 1. What is Blue/Green deployment?

You run two identical environments — **Blue** (current live) and **Green** (new version). You deploy the new version to Green, test it fully, then switch all traffic from Blue to Green in one shot.

In Kubernetes, the simplest way is having two Deployments with a shared Service. You just flip the Service selector:

```yaml
# Switch traffic to green
kubectl patch svc my-app -p '{"spec":{"selector":{"version":"green"}}}'
```

**Real-world example:** Say you're running an e-commerce app. v1 (Blue) is live serving customers. You deploy v2 (Green) with a new checkout flow. QA tests Green internally using a separate test Service. Once verified, you switch the main Service selector to Green. If customers start reporting payment failures — you switch back to Blue in one command. Recovery takes seconds, not minutes.

**Advantage:** Instant rollback — just flip the selector back.
**Disadvantage:** Runs double the resources during the switch window. For a 20-pod deployment, you need capacity for 40 pods temporarily.

**When to use:** High-risk releases where instant rollback matters more than resource cost — like payment services, auth services, or anything customer-facing where downtime = revenue loss.

---

## 2. How does a Rolling Update work?

> **Also asked as:** "How do you do a rolling update in Kubernetes?" · "Explain rolling deployment strategy" · "Explain the concept of a rolling update" · "How can you perform rolling updates with zero downtime?"

It's the default deployment strategy in Kubernetes. Instead of replacing all pods at once, it gradually replaces old pods with new ones — so the app stays available throughout.

Controlled by two fields:
- `maxSurge` — how many extra pods can exist during the update (default 25%)
- `maxUnavailable` — how many pods can be down during the update (default 25%)

**Example with real numbers:** You have a Deployment with 4 replicas. With `maxSurge: 1` and `maxUnavailable: 1`:
- K8s creates 1 new pod (now 5 total, 4 old + 1 new)
- Waits for the new pod to pass readiness probe
- Kills 1 old pod (now 4 total, 3 old + 1 new)
- Repeats until all 4 pods are the new version

**Real-world scenario:** We use this for our API services. We run 10 replicas behind a Service. During rollout, users never notice because at least 7-8 pods are always healthy and serving traffic. We set `maxUnavailable: 0` for critical services — meaning K8s won't kill any old pod until a new one is fully ready.

**Advantage:** Zero downtime, no extra environment needed.
**Gotcha:** During rollout, both v1 and v2 serve traffic simultaneously. If v2 changes the API response format or database schema, you'll get inconsistent behavior. Your app must be backward-compatible.

---

## 3. How does a Canary Deployment work?

You deploy the new version alongside the old but only route a **small percentage of traffic** (say 5-10%) to it. You watch error rates and latency. If it's healthy, gradually increase traffic. If something breaks, you only affected 5% of users — not everyone.

**How to do it in Kubernetes:**

Simplest way — two Deployments with the same label:
- `app-v1`: 9 replicas (label: `app: myapp`)
- `app-v2`: 1 replica (label: `app: myapp`)
- Service selector: `app: myapp` → routes ~10% traffic to v2

For finer control (like routing 2% traffic or targeting specific users), use **Istio VirtualService**:
```yaml
- route:
    - destination:
        host: my-app
        subset: v1
      weight: 95
    - destination:
        host: my-app
        subset: v2
      weight: 5
```

**Real-world scenario:** We deployed a new recommendation engine. Instead of rolling it out to all users, we canary'd it to 5% first. Within 10 minutes, we noticed the p99 latency was 3x higher than v1 — the new ML model was too heavy for our pod resource limits. We killed the canary, bumped the CPU limits, re-tested, and gradually rolled it out over 2 days. Without canary, all users would've had a bad experience.

**Advantage:** Limits blast radius. You catch issues with real production traffic but minimal impact.
**Disadvantage:** Requires solid monitoring — if you can't detect problems in 5% of traffic, canary is useless.

---

## 4. What is an Ingress Controller in Kubernetes?

A Service exposes your app inside the cluster. But how do external users reach it via `https://myapp.example.com`? That's where **Ingress** comes in.

**Ingress** is the K8s resource that defines routing rules — which hostname goes to which service, path-based routing, TLS termination. But Ingress YAML alone does nothing. You need an **Ingress Controller** — an actual reverse proxy (NGINX, Traefik, ALB) running as a pod that reads Ingress resources and configures itself accordingly.

Think of it like this:
- **Ingress** = a set of traffic rules written in YAML
- **Ingress Controller** = the NGINX/Traefik pod that actually enforces those rules

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

**Real-world scenario:** We run a microservices platform. A single NGINX Ingress Controller handles all external traffic. It routes `api.example.com` to the API service, `app.example.com` to the frontend, and `admin.example.com` to the admin panel. All through one LoadBalancer IP. Without Ingress, we'd need a separate LoadBalancer for each service — at $18/month each on AWS, that adds up fast with 15+ services.

**Common choices:**
- **NGINX Ingress Controller** — most widely used, battle-tested, huge community
- **Traefik** — lightweight, auto-discovers services, great for smaller setups
- **AWS ALB Ingress Controller** — if you want native AWS integration with WAF, Cognito auth, etc.

---

## 5. What actually happens when a Kubernetes Pod restarts?

When people say "the pod restarted," what actually happened is the **container inside the pod restarted** — the pod object itself doesn't restart. This distinction matters.

**The sequence:**

1. The container crashes (app error, OOM, failed liveness probe)
2. Kubelet detects the container is no longer running
3. Kubelet checks the pod's `restartPolicy` — `Always` (default for Deployments), `OnFailure`, or `Never`
4. If policy allows, kubelet restarts the container **on the same node, in the same pod**, with the same IP
5. The restart count increments — you see this in `kubectl get pods` as `RESTARTS: 3`
6. If the container keeps crashing, the backoff delay increases: 10s → 20s → 40s → 80s → ... up to 5 minutes. That's `CrashLoopBackOff`

**What stays the same:**
- Pod IP doesn't change — same pod, same network namespace
- Mounted volumes (PVC, ConfigMap, Secret) remain — data in PVCs survives
- The pod stays on the same node

**What resets:**
- Container filesystem — anything written to the container's writable layer is gone. Only data in mounted volumes survives
- Process state — the app starts fresh, no in-memory data from the previous run
- `emptyDir` volumes — they survive container restarts (they're tied to the pod, not the container), but they're lost if the **pod** is deleted/rescheduled

**Real scenario where this matters:** We had a caching service that built an in-memory cache on startup by reading from a database. Every restart took 2 minutes to rebuild the cache. During that window, every request hit the database directly — causing a load spike. The app didn't crash during this, but response times were 10x higher. The fix: we moved the cache to Redis so restarts didn't cause cold-cache thundering herd.

**Common confusion:** `kubectl delete pod <pod>` is NOT a restart — it deletes the pod entirely. The Deployment controller creates a new pod (new IP, potentially different node). A restart is the kubelet restarting the container inside the existing pod. If you want to restart all pods in a Deployment cleanly: `kubectl rollout restart deployment/<name>`.

---

## 6. Difference between readiness and liveness probes, and how each can break production.

> **Also asked as:** "What is the difference between liveness and readiness probes?"

Both are health checks, but they serve opposite purposes:

**Liveness probe** — "Is this container alive?" If it fails, kubelet **kills and restarts** the container. Use this to recover from deadlocks, infinite loops, or stuck states where the app is running but can't serve requests.

**Readiness probe** — "Is this container ready to receive traffic?" If it fails, the pod is **removed from Service endpoints**. The container keeps running — it's just not receiving new requests. Use this for startup warmup, dependency checks, or temporary overload.

**How liveness probes break production:**

Scenario: You set a liveness probe on `/health` with a 3-second timeout. Your app calls a database on `/health`. During a database slowdown (not outage — just slow), the health check takes 4 seconds → liveness probe fails → kubelet kills the container → all pods restart → all pods try to reconnect to the already-slow database simultaneously → database gets even more overwhelmed → pods keep restarting → **cascading failure**.

This is called the **death spiral**. The liveness probe, which is supposed to keep your app healthy, killed your entire service.

**Rule:** Liveness probes should check the **process itself** (is the app stuck?), not external dependencies. Never call a database, Redis, or external API in a liveness probe. If the dependency is down, restarting your app won't fix it.

```yaml
# GOOD — checks if the app process is responsive
livenessProbe:
  httpGet:
    path: /livez    # Returns 200 if the HTTP server is not deadlocked
    port: 3000
  timeoutSeconds: 5
  failureThreshold: 3

# BAD — checks a database connection in liveness
livenessProbe:
  httpGet:
    path: /health   # This endpoint queries PostgreSQL
    port: 3000
```

**How readiness probes break production:**

Scenario: Readiness probe has `initialDelaySeconds: 5` but the app takes 30 seconds to start. The probe passes after 5 seconds (the HTTP server is up, but the app hasn't loaded configs/caches yet). Traffic arrives → app returns 500s for 25 seconds until it's fully initialized.

Opposite scenario: Readiness probe is too strict — it checks cache warmth, connection pool, AND external API reachability. If any dependency has a blip, ALL pods become not-ready simultaneously → zero endpoints → 100% traffic drop. Your app is fine, but nobody can reach it.

**Best practice:**
- **Liveness**: check only the process (is the HTTP server responding?)
- **Readiness**: check if the app can serve requests (configs loaded, critical connections established)
- **Startup probe**: for slow-starting apps (JVM, ML models). Gives the app time to start before liveness kicks in. Without this, liveness kills slow-starting containers before they're ready.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 3000
  failureThreshold: 30
  periodSeconds: 10    # Gives the app 300 seconds (5 min) to start
```

---

## 7. What's the difference between horizontal and vertical scaling, and when should you avoid autoscaling?

> **Also asked as:** "How does Kubernetes handle container scaling?"

**Horizontal scaling (HPA)** — Add more pods. You go from 3 replicas to 10. Each pod stays the same size. Works for stateless apps where more instances = more capacity.

**Vertical scaling (VPA)** — Make existing pods bigger. Instead of 3 pods with 256Mi memory, you resize them to 1Gi each. Fewer pods, bigger resources. VPA in K8s requires **restarting the pod** to apply new limits.

**Real-world analogy:** Horizontal = adding more cashiers at a supermarket. Vertical = replacing a cashier with a faster one. Horizontal is usually better because if one cashier goes down, others keep working.

**When to avoid autoscaling entirely:**

1. **Databases and stateful services** — PostgreSQL, MySQL, single-instance Redis. You can't just add replicas and expect them to share load. Data consistency, replication lag, and connection management make horizontal scaling complex. We run our PostgreSQL on a fixed, right-sized instance. No autoscaling.

2. **Predictable, steady workloads** — If your app handles 100 req/sec 24/7 with no variance, autoscaling adds complexity (metrics-server, HPA configs, testing scale-up/down behavior) for zero benefit. We have internal tools that serve 50 users during business hours — 3 fixed replicas, no HPA.

3. **Cost-sensitive environments** — HPA can scale up aggressively during traffic spikes. If you set `maxReplicas: 50` and a bot hits your API, you're paying for 50 pods until the cooldown kicks in. One team's HPA scaled to 40 replicas during a DDoS, adding $800/day in compute before anyone noticed. If cost predictability matters, use fixed replicas with a sensible buffer.

4. **Slow-starting apps** — If your Java app takes 4 minutes to start, HPA pods won't be ready before the traffic spike ends. You're paying for pods that never served a single request. Better to over-provision with 2-3 extra replicas permanently.

5. **VPA with zero-downtime requirements** — VPA restarts pods to apply new limits. For a payments service processing transactions, a random restart is unacceptable. We use VPA in `Off` mode (recommendation only) and apply changes during maintenance windows.

**When autoscaling shines:** Stateless API services with variable traffic — e-commerce during sales events, SaaS platforms with business-hour peaks. Our main API uses HPA (min: 5, max: 30) on CPU + request rate. During normal hours: 5-8 pods. During flash sales: 25-30 pods. Scale-down happens automatically over 10 minutes after the spike.

---

## 8. Why are StatefulSets useful, and how do they differ from Deployments?

> **Also asked as:** "What is a StatefulSet in Kubernetes?" · "Explain the difference between a Deployment and a StatefulSet" · "Kubernetes deployment vs StatefulSets"

**Deployments** are for stateless apps — your API servers, frontends, workers. Every pod is identical and interchangeable. If pod-3 dies, a new pod is created with a random name (`app-7xk2f`) and can be scheduled anywhere. Pods don't have a fixed identity.

**StatefulSets** are for stateful apps — databases, message brokers, distributed systems like Kafka, Elasticsearch, ZooKeeper. Pods need stable identity, predictable ordering, and persistent storage.

**Key differences:**

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random (`app-7xk2f`) | Ordered (`app-0`, `app-1`, `app-2`) |
| Startup order | All at once (parallel) | Sequential (app-0 first, then app-1) |
| Storage | Shared or no persistence | Each pod gets its own PVC that sticks to it |
| Network identity | Random pod IP | Stable hostname via headless Service (`app-0.my-service`) |
| Scaling down | Kills any pod | Kills in reverse order (app-2 first, then app-1) |

**Why stable identity matters — real scenario:**

We run a 3-node Kafka cluster. Each broker has a unique ID (broker-0, broker-1, broker-2) and stores data on its own persistent volume. If broker-1 crashes, it **must** come back as broker-1 with the same volume — not as a random new pod with an empty disk. If Kafka lost its data on restart, we'd lose messages. StatefulSet guarantees this: `kafka-1` always gets PVC `data-kafka-1`, always gets hostname `kafka-1.kafka-headless`, always starts after `kafka-0`.

With a Deployment, K8s would create a new pod with a random name, no data, and no way for other brokers to recognize it. The Kafka cluster would be broken.

**When to use StatefulSets:**
- Databases (PostgreSQL, MySQL, MongoDB)
- Message brokers (Kafka, RabbitMQ)
- Distributed caches (Redis Cluster, Elasticsearch)
- Any app that says "I need to know who I am and keep my data between restarts"

**When NOT to use StatefulSets:**
- Stateless API services — Deployments are simpler and scale faster
- Apps that store everything in external storage (S3, RDS) — they don't need local persistent identity
- We once saw a team using StatefulSets for a REST API because they "wanted stable pod names." That's not a valid reason — it added unnecessary complexity (ordered rollouts are slower, scaling is sequential). Use Deployments for anything that doesn't need persistent identity.

---

## 9. Difference between Persistent Volume (PV) and Persistent Volume Claim (PVC).

> **Also asked as:** "How does Kubernetes handle storage for applications?"

Think of it like renting an apartment:
- **PV (Persistent Volume)** = the actual apartment (the physical storage). It exists in the cluster, created by an admin or dynamically provisioned. It has a size, access mode, and storage backend (EBS, NFS, GCE PD).
- **PVC (Persistent Volume Claim)** = the rental application (the request for storage). A pod says "I need 10Gi of ReadWriteOnce storage" without caring which specific PV provides it.

K8s matches PVCs to PVs automatically (binding). The pod references the PVC, and K8s attaches the correct PV.

```yaml
# PV — the actual storage (admin creates this, or StorageClass provisions it dynamically)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    volumeID: vol-0abc123
    fsType: ext4

# PVC — the pod's request for storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

# Pod references the PVC, not the PV directly
spec:
  containers:
    - name: app
      volumeMounts:
        - mountPath: /data
          name: my-storage
  volumes:
    - name: my-storage
      persistentVolumeClaim:
        claimName: my-pvc
```

**Why this separation exists:**

It decouples the "I need storage" (developer concern) from "here's how storage is provisioned" (infra/admin concern). A developer writes a PVC in their manifest saying "I need 10Gi." They don't care if it's EBS, NFS, or Ceph. The cluster admin configures a **StorageClass** that defines how PVs are dynamically created.

**Real scenario:** We use dynamic provisioning with a StorageClass in EKS:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Retain
```

When a PVC requests `storageClassName: fast`, EKS automatically creates a gp3 EBS volume with 3000 IOPS, attaches it to the node, and binds it to the PVC. No manual PV creation needed.

**Common gotcha:** If `reclaimPolicy` is `Delete` (the default), deleting the PVC **deletes the underlying volume and all data**. We set it to `Retain` for databases — so even if someone accidentally deletes the PVC, the EBS volume survives and can be re-attached. This saved us once when a developer deleted a StatefulSet without realizing it would cascade-delete the PVCs.

---

## 10. Difference between ConfigMap and Secret

> **Also asked as:** "What is the role of a Kubernetes ConfigMap?"

Both store configuration data as key-value pairs. The difference is **what** they store and **how** they handle it.

**ConfigMap** — stores non-sensitive configuration: feature flags, database hostnames, log levels, app settings. Stored as plain text in etcd. Anyone with read access can see the values.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  LOG_LEVEL: "info"
  FEATURE_NEW_UI: "true"
```

**Secret** — stores sensitive data: passwords, API keys, TLS certificates, tokens. Base64-encoded in etcd (NOT encrypted by default — just encoded). With encryption at rest enabled, Secrets are actually encrypted in etcd.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=   # base64 of "password123"
```

**Key differences:**

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| Data type | Non-sensitive config | Sensitive credentials |
| Storage | Plain text in etcd | Base64-encoded (encrypted if configured) |
| Size limit | 1 MB | 1 MB |
| `kubectl get` output | Shows values directly | Hides values (shows `Opaque`) |
| RBAC | Usually broader access | Should be tightly restricted |

**How I use them in production:**

- ConfigMap for: database hostname, feature flags, log levels, app port, external service URLs
- Secret for: database passwords, API keys, TLS certs, OAuth tokens

**Real scenario that taught us the difference matters:** A developer stored a database password in a ConfigMap instead of a Secret. During a debugging session, someone ran `kubectl get configmap app-config -o yaml` on a shared screen. The password was visible in plain text. If it had been a Secret, `kubectl get secret` only shows the key names, not values (you have to explicitly decode it). After that, we added a Kyverno policy that blocks creating ConfigMaps with keys named `password`, `secret`, `token`, or `key`.

**Important:** Secrets are NOT secure by default. Base64 is encoding, not encryption. You must enable **etcd encryption at rest** and use **External Secrets Operator** or **Sealed Secrets** for real security. Think of K8s Secrets as "slightly hidden ConfigMaps" unless you add encryption.

---

## 11. How to check logs of a Pod or application?

**Basic log check:**
```bash
# Current pod logs
kubectl logs <pod-name>

# Logs from a specific container (multi-container pod)
kubectl logs <pod-name> -c <container-name>

# Logs from the previous crashed container (critical for CrashLoopBackOff)
kubectl logs <pod-name> --previous

# Follow logs in real-time (like tail -f)
kubectl logs <pod-name> -f

# Last 100 lines only
kubectl logs <pod-name> --tail=100

# Logs from the last 30 minutes
kubectl logs <pod-name> --since=30m
```

**Logs from multiple pods at once:**
```bash
# All pods with a specific label
kubectl logs -l app=my-service --all-containers=true

# Across all namespaces (useful during incidents)
kubectl logs -l app=my-service -n prod --since=5m
```

**Real scenario — when `kubectl logs` isn't enough:**

We had a pod that crashed so fast it had zero log output. `kubectl logs --previous` returned nothing. The container died before it could write anything. In this case:

1. Checked events: `kubectl describe pod <pod>` — the `Events` section showed `OOMKilled` before the app could even start logging
2. For init container failures: `kubectl logs <pod> -c <init-container-name>` — people forget init containers have separate logs
3. For node-level issues: `kubectl get events --field-selector involvedObject.name=<pod>` — shows scheduling failures, image pull errors, mount failures that the pod itself can't log

**In production, we don't rely on `kubectl logs` for day-to-day:**

We ship all container logs to **Loki** using Promtail (runs as a DaemonSet on every node). Developers query logs in Grafana with LogQL:

```
{namespace="prod", app="order-service"} |= "error" | json | status_code >= 500
```

Why: `kubectl logs` only shows logs from running/recently-crashed pods. If a pod was deleted or rescheduled, those logs are gone forever. Loki retains logs for 30 days regardless of pod lifecycle.

**Pro tip:** If an app writes logs to a file instead of stdout, `kubectl logs` shows nothing. You need to either exec into the pod (`kubectl exec <pod> -- cat /var/log/app.log`) or add a sidecar container that tails the log file to stdout. This is a common gotcha with legacy Java apps that log to files.

---

## 12. Explain DaemonSet, StatefulSet, and Deployment — when to use each?

> **Also asked as:** "What is a DaemonSet in Kubernetes?"

All three manage pods, but for very different use cases:

**Deployment** — for stateless apps. Your API servers, frontends, microservices. Every pod is identical and replaceable. If a pod dies, a new one is created with a random name, potentially on a different node. You don't care which pod handles which request.

**StatefulSet** — for stateful apps. Databases (PostgreSQL, MongoDB), message brokers (Kafka), distributed systems (Elasticsearch). Pods get stable names (`app-0`, `app-1`), start in order, and each gets its own persistent volume that sticks to it across restarts. (Covered in detail in Q8 above.)

**DaemonSet** — runs exactly **one pod on every node** (or a subset of nodes). Used for node-level infrastructure: log collectors, monitoring agents, network plugins.

| Aspect | Deployment | StatefulSet | DaemonSet |
|--------|-----------|-------------|-----------|
| Use case | Stateless apps | Stateful apps | Per-node agents |
| Pod count | You choose replicas | You choose replicas | One per node (automatic) |
| Pod names | Random (`app-x7k2f`) | Ordered (`app-0`, `app-1`) | One per node (`agent-nodeA`) |
| Scaling | Horizontal (add replicas) | Sequential (one at a time) | Automatic (add a node = add a pod) |
| Storage | Shared or none | Per-pod PVC | Usually hostPath or emptyDir |

**DaemonSet examples from our cluster:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
spec:
  selector:
    matchLabels:
      app: promtail
  template:
    spec:
      containers:
        - name: promtail
          image: grafana/promtail:latest
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

This runs Promtail on every node — it reads container logs from `/var/log` on each node and ships them to Loki. When we add a new node to the cluster, a Promtail pod is automatically scheduled on it. No manual action needed.

**What we run as DaemonSets in production:**
- **Promtail** — log shipping (every node needs to ship its logs)
- **Datadog/Node Exporter** — node-level metrics (CPU, memory, disk per node)
- **aws-node (VPC CNI)** — networking plugin (every node needs pod networking)
- **kube-proxy** — service networking (every node needs iptables/ipvs rules)
- **Falco** — security monitoring (every node needs syscall monitoring)

**Real scenario:** We once deployed a monitoring agent as a Deployment with 3 replicas instead of a DaemonSet. On a 10-node cluster, only 3 nodes had the agent. The other 7 nodes had zero monitoring — we missed a disk-full incident on an unmonitored node. Switching to a DaemonSet ensured every node is monitored automatically, including new nodes added by the autoscaler.

**When to use nodeSelector with DaemonSet:** If you only want the DaemonSet on specific nodes (e.g., GPU monitoring only on GPU nodes):
```yaml
spec:
  template:
    spec:
      nodeSelector:
        hardware: gpu
```

---

## 13. Explain Kubernetes cluster architecture

> **Also asked as:** "What are the main components of Kubernetes?"

A K8s cluster has two parts: the **Control Plane** (brain) and **Worker Nodes** (muscle).

**Control Plane components:**

- **API Server (`kube-apiserver`)** — the front door. Every `kubectl` command, every controller, every node talks to the cluster through the API server. It validates requests, authenticates users, and persists state to etcd. If the API server is down, you can't manage the cluster (but running workloads keep running).

- **etcd** — the database. Stores all cluster state: deployments, pods, services, configmaps, secrets — everything. It's a distributed key-value store. If etcd is lost and there's no backup, the cluster state is gone. That's why we take etcd snapshots every hour in self-managed clusters.

- **Scheduler (`kube-scheduler`)** — decides which node runs which pod. When a new pod is created, the scheduler checks node resources, taints/tolerations, affinity rules, and picks the best node. It doesn't start the pod — it just assigns it to a node.

- **Controller Manager (`kube-controller-manager`)** — runs reconciliation loops. The Deployment controller creates ReplicaSets. The ReplicaSet controller creates pods. The Node controller marks nodes NotReady. Each controller watches desired state vs actual state and takes action to close the gap.

**Worker Node components:**

- **Kubelet** — the agent on every node. It watches for pods assigned to its node, pulls images, starts containers via the container runtime (containerd), runs health probes, and reports status back to the API server. If kubelet dies, the node goes NotReady.

- **kube-proxy** — handles networking for Services. It creates iptables/ipvs rules so that when a pod calls `my-service:80`, the traffic is routed to one of the service's backend pods.

- **Container Runtime (containerd)** — actually runs containers. Docker was removed as a runtime in K8s 1.24. Most clusters now use containerd directly.

**How a request flows — real example:**

You run `kubectl apply -f deployment.yaml`:
1. `kubectl` sends the request to the **API server**
2. API server validates it, stores it in **etcd**
3. **Deployment controller** notices a new Deployment → creates a ReplicaSet
4. **ReplicaSet controller** notices the RS → creates pod objects
5. **Scheduler** sees unscheduled pods → assigns them to nodes
6. **Kubelet** on each assigned node → pulls the image, starts the container
7. **kube-proxy** → updates iptables so the Service can route traffic to the new pods

**Real scenario where understanding architecture helped:** Our cluster was "slow" — `kubectl` commands took 10-15 seconds. Pods were running fine but any API call was laggy. Root cause: etcd disk was a gp2 volume with 100 IOPS. etcd does a LOT of disk writes (every state change). Upgrading to gp3 with 3000 IOPS brought API latency from 10s to <1s. Understanding that etcd is the bottleneck for API server performance pointed us directly to the fix.

---

## 14. What are the different Kubernetes deployment strategies?

> **Also asked as:** "Explain the deployment strategy used in your organisation" · "How do you ensure zero-downtime deployments?" · "How would you ensure zero-downtime deployment during a critical update?"

Four main strategies, each with different risk profiles:

**1. Rolling Update** (default)
- Gradually replaces old pods with new ones
- Controlled by `maxSurge` and `maxUnavailable`
- Zero downtime, both versions serve traffic during rollout
- Best for: most stateless services
- Risk: if v2 is broken, some users are affected before you notice
- (Covered in detail in Q2)

**2. Blue/Green**
- Two full environments running simultaneously
- Switch all traffic at once by changing Service selector
- Instant rollback — switch the selector back
- Best for: high-risk releases (payments, auth)
- Risk: double resource cost during switch
- (Covered in detail in Q1)

**3. Canary**
- Route small % of traffic (5-10%) to new version
- Monitor error rates/latency, gradually increase traffic
- Best for: large-scale services where you need to validate with real traffic
- Risk: requires solid monitoring to detect issues in small traffic %
- (Covered in detail in Q3)

**4. Recreate**
- Kill ALL old pods first, then create ALL new pods
- Has downtime — there's a window where zero pods are running
- Best for: development environments, or apps that can't run two versions simultaneously (e.g., database schema changes that break old version)
- Risk: guaranteed downtime

```yaml
spec:
  strategy:
    type: Recreate    # All old pods die before new ones start
```

**Which one do I use in production?**

- **Rolling Update** for 90% of services — APIs, frontends, workers. It's the default for a reason.
- **Canary** for critical, high-traffic services — we use Argo Rollouts with Istio for automated canary analysis. It monitors error rate and latency and auto-promotes or auto-rolls-back.
- **Blue/Green** for database-dependent releases where we need instant rollback capability.
- **Recreate** only in dev/staging when testing migration scripts.

**Real scenario:** We initially used Rolling Update for everything. Then we deployed a new version of the recommendation service that had a subtle bug — it returned wrong recommendations for 15% of users. Rolling update completed successfully, all pods healthy, no errors. We didn't catch it for 2 hours because the error wasn't technical (no 5xx, no crashes). After that, we switched to Canary for the recommendation service with a Prometheus query checking recommendation accuracy metrics. Now we catch these "silent regressions" during the canary phase.

---

## 15. What's the difference between `kubectl exec`, `kubectl logs`, and `kubectl describe`? When do you use each?

They serve completely different purposes — think of them as three different levels of investigation.

**`kubectl logs <pod>`** — Shows stdout/stderr from the container. This is your **first stop** for application errors. Use `--previous` to see logs from a crashed container, `-f` to stream live logs, and `-c <container>` for multi-container pods.

When to use: App returning 500 errors, pod restarting, need to see application-level output.

**`kubectl describe pod <pod>`** — Shows the **full lifecycle** of the pod: scheduling decisions, events, conditions, resource limits, image pull status, probe results. This is Kubernetes-level information, not application-level.

When to use: Pod stuck in `Pending` (scheduling issue), `ImagePullBackOff` (wrong image/registry), container not starting, probe failures. The `Events` section at the bottom is gold — it tells you exactly what K8s tried to do and what failed.

**`kubectl exec -it <pod> -- <command>`** — Opens a shell **inside** the running container. You can run commands, check files, test network connectivity, inspect environment variables.

When to use: Need to debug networking (`curl`, `nslookup`), check if config files are mounted correctly (`cat /etc/config/app.yaml`), verify environment variables (`env | grep DB`), or test connectivity to other services (`nc -z postgres-service 5432`).

**Real debugging flow:** Pod returning errors → `kubectl logs` first (see the app error) → `kubectl describe` (check events, probe failures, resource limits) → `kubectl exec` (dig deeper — is the config file there? Can the pod reach the database?).

**Real scenario:** A pod was running but returning empty responses. `kubectl logs` showed no errors. `kubectl describe` showed everything healthy. `kubectl exec` into the pod and checked the mounted ConfigMap — it was empty. Someone had updated the ConfigMap but the pod was using a cached version. After restarting the pod, it picked up the new config.

---

## 16. What are manifest files in Kubernetes?

Manifest files are YAML (or JSON) files that declare the **desired state** of your Kubernetes resources. You write what you want, and Kubernetes makes it happen.

Every K8s object — Deployment, Service, ConfigMap, Ingress, PV — is defined as a manifest. The structure always has four top-level fields:

```yaml
apiVersion: apps/v1        # Which API group and version
kind: Deployment            # What type of resource
metadata:                   # Name, namespace, labels, annotations
  name: my-app
  namespace: production
  labels:
    app: my-app
spec:                       # The actual desired state (different for each kind)
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v1.2.3
          ports:
            - containerPort: 8080
```

**How you apply them:**
```bash
kubectl apply -f deployment.yaml          # Apply a single file
kubectl apply -f ./k8s/                   # Apply all files in a directory
kubectl apply -f https://raw.git...       # Apply from a URL
```

**Why manifests matter:**
- **Declarative** — you describe the end state, not the steps to get there. K8s figures out the diff and applies changes.
- **Version controlled** — manifests live in Git. You can track who changed what, when, and why. Roll back with `git revert`.
- **Reproducible** — same manifest produces the same result in dev, staging, and production (with Helm/Kustomize for environment-specific values).

**Real scenario:** A team was creating resources with `kubectl create` and `kubectl run` commands (imperative approach). When a node failed, they couldn't recreate the exact same setup — nobody remembered the exact flags used. After switching to manifest files stored in Git with ArgoCD syncing, every resource was reproducible. Node failure → ArgoCD reapplied everything automatically.

**In production we manage manifests with:** Helm charts for templating (values per environment) + Kustomize for overlays + ArgoCD for GitOps deployment. Raw YAML files are only used for one-off resources or quick testing.

---

## 17. What is Kubernetes?

Kubernetes (K8s) is an open-source **container orchestration platform**. It automates deploying, scaling, and managing containerized applications.

**Without Kubernetes:** You have 50 containers running across 10 servers. A server crashes — you manually figure out which containers were on it, manually restart them on another server, manually update the load balancer. At 3 AM. Every time.

**With Kubernetes:** You declare "I want 5 replicas of my app." K8s places them across nodes, monitors their health, and if a node crashes, automatically reschedules the pods elsewhere. You sleep through the night.

**What it actually does:**
- **Scheduling** — decides which node runs which container based on available resources
- **Self-healing** — restarts crashed containers, replaces pods on failed nodes, kills pods that don't respond to health checks
- **Scaling** — HPA adds/removes pods based on CPU, memory, or custom metrics
- **Service discovery** — pods find each other by name, not IP address (via CoreDNS)
- **Rolling updates** — deploy new versions with zero downtime
- **Secret/config management** — inject configuration and credentials without baking them into images

**Real scenario:** Before K8s, our team deployed 12 microservices using bash scripts and docker-compose on EC2 instances. A deployment took 45 minutes of manual SSH, and rollbacks meant re-running the scripts in reverse. After migrating to K8s, deployments are a `git push` (ArgoCD handles the rest), rollbacks are `kubectl rollout undo`, and self-healing means we don't get paged for single-container crashes anymore.

**One-liner for interviews:** "Kubernetes is a platform that manages the lifecycle of containers at scale — it handles placement, health, scaling, networking, and configuration so engineers focus on the application, not the infrastructure."

---

## 18. What is a Pod in Kubernetes?

A Pod is the **smallest deployable unit** in Kubernetes. It's not a container — it's a wrapper around one or more containers that share the same network and storage.

**Key characteristics:**
- Every pod gets its own **IP address**. Containers within the same pod communicate via `localhost`.
- Containers in a pod share the same **network namespace** (same IP, same port space) and can share **volumes**.
- A pod is **ephemeral** — it's not meant to be long-lived. If it dies, K8s creates a new one (with a new IP).

**Single-container pod (99% of cases):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: my-app
      image: my-app:v1.0
      ports:
        - containerPort: 8080
```

**Multi-container pod (sidecar pattern):**
```yaml
spec:
  containers:
    - name: my-app
      image: my-app:v1.0
    - name: log-shipper
      image: fluentbit:latest   # Ships logs to a central system
```
The sidecar (log-shipper) shares the filesystem with the main container and has access to its logs.

**Why not just run containers directly?** Pods provide a shared context — co-located containers that need to communicate tightly (main app + log shipper, main app + envoy proxy) run in the same pod. They start together, stop together, and share the same lifecycle.

**Real scenario:** We run Istio service mesh. Every application pod has an Envoy sidecar proxy injected automatically. The app container handles business logic, the sidecar handles mTLS, retries, and traffic routing. Both are in the same pod — the app doesn't even know the sidecar exists.

**Important:** You almost never create Pods directly. You use Deployments, StatefulSets, or DaemonSets — which manage pods for you.

---

## 19. What is a ReplicaSet?

A ReplicaSet ensures that a **specified number of identical pods are running** at all times. If a pod crashes, the ReplicaSet creates a new one. If there are too many, it deletes extras.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v1.0
```

**How it works:** The ReplicaSet controller watches the cluster and counts pods matching its label selector. Too few → create new pods. Too many → delete extras (newest first, preferring unready pods).

**Important:** You almost never create ReplicaSets directly. **Deployments manage ReplicaSets for you.** When you create a Deployment, it creates a ReplicaSet behind the scenes. When you update the Deployment (new image tag), it creates a *new* ReplicaSet and gradually shifts pods from the old one to the new one — that's how rolling updates work.

**When to know about ReplicaSets:** During debugging. If a deployment is stuck, `kubectl get rs` shows you the ReplicaSets. You might see the old RS with 3 pods and the new RS with 0 — meaning the new pods can't schedule (resource shortage, image pull error, etc.).

**Real scenario:** After a bad deployment, `kubectl get rs` showed two ReplicaSets — old one with 2 pods, new one with 1 pod stuck in `ImagePullBackOff`. The Deployment was stuck mid-rollout. Understanding that Deployments manage ReplicaSets helped us quickly identify the issue (`kubectl describe rs <new-rs>` showed the image tag was wrong).

---

## 20. What is a Deployment?

A Deployment is the **most common way to run stateless applications** in Kubernetes. It manages ReplicaSets, which manage Pods — giving you declarative updates, rollbacks, and scaling.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v1.2.3
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
```

**What a Deployment gives you:**
- **Rolling updates** — change the image tag, K8s gradually replaces old pods with new ones. Zero downtime.
- **Rollback** — `kubectl rollout undo deployment/my-app` instantly reverts to the previous version.
- **Scaling** — `kubectl scale deployment/my-app --replicas=5` or let HPA handle it automatically.
- **Self-healing** — if a pod crashes or a node fails, the Deployment ensures the desired replica count is maintained.

**Deployment vs Pod vs ReplicaSet:**
- **Pod** = a single running instance
- **ReplicaSet** = ensures N pods are running
- **Deployment** = manages ReplicaSets + rolling updates + rollback history

**Real scenario:** We deploy all our stateless microservices (APIs, frontends, background workers) as Deployments. A typical production Deployment has: 3 replicas, resource requests/limits, readiness probes, pod anti-affinity (spread across AZs), and a PodDisruptionBudget. The Deployment controller handles updates — we just push a new image tag via ArgoCD and the rolling update happens automatically.

---

## 21. What is a Kubernetes Namespace?

A Namespace is a **virtual cluster** within a Kubernetes cluster. It provides isolation — separating resources, access, and configuration by team, environment, or project.

```bash
kubectl get namespaces
# default        — where resources go if you don't specify a namespace
# kube-system    — K8s system components (CoreDNS, metrics-server, kube-proxy)
# kube-public    — publicly readable, rarely used
# kube-node-lease — node heartbeat tracking
```

**What namespaces give you:**
- **Resource isolation** — ResourceQuotas limit CPU/memory per namespace. Team A can't consume all cluster resources.
- **Access control** — RBAC scopes permissions per namespace. Developers get `edit` in their namespace, `view` in others.
- **Network isolation** — NetworkPolicies can restrict traffic between namespaces.
- **Organization** — `kubectl get pods -n payments` shows only payments team pods, not 500 pods from the entire cluster.

**How we use them in production:**
```
production/
├── payments        # Payment service team
├── orders          # Orders team
├── frontend        # Frontend team
├── monitoring      # Prometheus, Grafana, Alertmanager
├── ingress-nginx   # Ingress controllers
└── argocd          # GitOps tooling
```

Each team namespace has: ResourceQuota, LimitRange (default resource requests), NetworkPolicy (default deny), and RBAC RoleBindings.

**Real scenario:** Without namespaces, a developer accidentally ran `kubectl delete deployment --all` and wiped every deployment in the cluster. After that, we enforced namespace-per-team with RBAC — developers only have `edit` access in their own namespace. The same mistake now only affects their own services.

---

## 22. How does Kubernetes handle service discovery and load balancing?

Kubernetes has **built-in service discovery** — pods find each other by name, not IP. This is critical because pod IPs change every time a pod restarts.

**Service** is the answer. A Service provides a stable DNS name and IP that routes to a set of pods matching a label selector.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    app: my-app      # Routes to all pods with this label
  ports:
    - port: 80        # Service port
      targetPort: 8080 # Container port
  type: ClusterIP     # Default — internal only
```

**How service discovery works:**
1. Pod A wants to reach Pod B's service → queries `my-app.production.svc.cluster.local`
2. **CoreDNS** resolves this to the Service's ClusterIP (e.g., `10.96.0.15`)
3. **kube-proxy** (running on every node) intercepts traffic to that ClusterIP and forwards it to one of the healthy backend pods using iptables/IPVS rules
4. Traffic reaches the pod — load balanced across all ready pods

**Short DNS names work within the same namespace:**
```bash
# From a pod in 'production' namespace:
curl http://my-app:80         # Same namespace — works
curl http://my-app.production  # Explicit namespace — works
curl http://my-app.production.svc.cluster.local  # FQDN — works
```

**Load balancing:** kube-proxy distributes traffic using **round-robin** (iptables mode) or **least-connection** (IPVS mode) across all pods that pass their readiness probe.

**Service types:**
- `ClusterIP` (default) — internal only, reachable within the cluster
- `NodePort` — exposes on every node's IP at a static port (30000-32767)
- `LoadBalancer` — provisions a cloud load balancer (AWS NLB/ALB, Azure LB)

**Real scenario:** A pod was restarting frequently, and dependent services kept getting connection errors. The pod's IP changed on every restart, but the Service abstracted that away — once the new pod passed its readiness probe, the Service automatically routed traffic to it. Without Services, every dependent app would need to track pod IPs manually.

---

## 23. How can you expose a Kubernetes deployment to the outside world?

There are three common approaches, each for different use cases:

**1. Service type `LoadBalancer` — simplest**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```
Creates a cloud load balancer (AWS NLB/ALB, Azure LB) with a public IP. Direct, simple, but **each service gets its own LB** — at $18-25/month each, this gets expensive with many services.

**2. Ingress + Ingress Controller — recommended for HTTP**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```
One LoadBalancer (the Ingress Controller — nginx, Traefik, or ALB) handles routing for **all** services based on hostname/path. 50 services behind 1 LB instead of 50 separate LBs.

**3. NodePort — for testing/on-prem**
Exposes the service on a static port (30000-32767) on every node's IP. Access via `<node-ip>:30XXX`. Not ideal for production — no SSL termination, no hostname routing.

**What we use in production:** Ingress Controller (nginx-ingress) behind a single AWS NLB, with cert-manager for automatic TLS certificates. All services share one LB. External DNS automatically creates Route53 records when new Ingress resources are created.

**Real scenario:** A team created 12 services, each with `type: LoadBalancer`. Monthly LB cost: $216 for just the load balancers. We consolidated to a single Ingress Controller — one LB, all 12 services routed via host-based rules. Monthly cost: $18. Same functionality, 92% cost reduction.

---

## 24. What are Kubernetes labels and selectors?

**Labels** are key-value pairs attached to K8s objects (pods, services, nodes). **Selectors** are queries that filter objects by their labels. This is how Kubernetes connects things together.

```yaml
# Pod with labels
metadata:
  labels:
    app: my-app
    env: production
    team: payments
    version: v1.2.3
```

**Why they matter — everything in K8s uses labels:**

**Services** use selectors to find pods:
```yaml
# Service routes traffic to pods with app=my-app
spec:
  selector:
    app: my-app
```

**Deployments** use selectors to manage pods:
```yaml
spec:
  selector:
    matchLabels:
      app: my-app
```

**NetworkPolicies** use selectors to apply rules:
```yaml
spec:
  podSelector:
    matchLabels:
      app: my-app
```

**Node selectors / affinity** use labels to place pods on specific nodes:
```yaml
spec:
  nodeSelector:
    disk: ssd           # Only schedule on nodes labeled disk=ssd
```

**Useful `kubectl` commands with labels:**
```bash
kubectl get pods -l app=my-app                    # Filter by label
kubectl get pods -l 'env in (production,staging)'  # Multiple values
kubectl get pods -l app=my-app,team=payments       # AND condition
kubectl label node node-1 disk=ssd                 # Add label to node
```

**Real scenario:** A Service was returning 404s even though the pods were running fine. `kubectl get endpoints my-app-service` showed zero endpoints. The problem: the pod had `app: my-app-v2` but the Service selector was `app: my-app`. Labels didn't match → Service couldn't find the pods → no traffic routed. Changed the pod labels to match and traffic flowed immediately. This is the #1 cause of "my Service isn't working" issues.

---

## 25. What are Taints and Tolerations in Kubernetes?

**Taints** are applied to nodes — they repel pods from being scheduled there. **Tolerations** are applied to pods — they allow a pod to be scheduled on a tainted node. Together, they give you control over which pods run on which nodes.

Think of it like this:
- **Taint** = a "No entry" sign on a node
- **Toleration** = a special pass the pod carries that lets it through

**Applying a taint to a node:**
```bash
kubectl taint nodes node-1 gpu=true:NoSchedule
```

This means: "Don't schedule any pod on node-1 unless it explicitly tolerates `gpu=true`."

Three taint effects:
- `NoSchedule` — new pods without the toleration won't be scheduled here
- `PreferNoSchedule` — K8s will try to avoid scheduling here, but not strictly
- `NoExecute` — new pods won't schedule AND existing pods without the toleration are evicted

**Adding a toleration to a pod:**
```yaml
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: ml-trainer
      image: ml-trainer:v1.0
```

This pod can now be scheduled on nodes tainted with `gpu=true:NoSchedule`.

**Common real-world uses:**

| Use Case | Taint on Node | Toleration on Pod |
|---|---|---|
| GPU nodes for ML workloads | `gpu=true:NoSchedule` | ML training pods only |
| Spot/preemptible nodes | `spot=true:NoSchedule` | Batch jobs, workers |
| Dedicated nodes per team | `team=payments:NoSchedule` | Payments pods only |
| Node under maintenance | `node.kubernetes.io/unschedulable:NoSchedule` | Added automatically by K8s |

**Taints vs Node Affinity — what's the difference:**
- **Taints/Tolerations** = used to *repel* pods from nodes (push model — node pushes pods away)
- **Node Affinity** = used to *attract* pods to specific nodes (pull model — pod chooses node)

In practice, use both together: taint GPU nodes to keep regular pods off, and add node affinity to ML pods to ensure they land on GPU nodes specifically.

```yaml
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: gpu
                operator: In
                values:
                  - "true"
```

**Real scenario:** We had a 10-node cluster where 2 nodes had NVMe SSDs for high-IOPS database workloads. Without taints, the scheduler freely placed regular API pods on those nodes, consuming resources meant for the database. We tainted the SSD nodes with `disk=nvme:NoSchedule` and added the toleration only to the database StatefulSets. After that, the SSD nodes were exclusively used for databases and API pods no longer competed for that capacity. Query latency dropped by 40% because the database pods were no longer sharing nodes with CPU-heavy services.

**Removing a taint:**
```bash
kubectl taint nodes node-1 gpu=true:NoSchedule-   # The trailing dash removes it
```

---

## 26. What Load Balancer do you use in Kubernetes and why?

In Kubernetes, "Load Balancer" can mean two different things — the **Service type** and the **Ingress Controller**. In production, we use both, but for different purposes.

**Service type `LoadBalancer` — for raw TCP/UDP or when you need a dedicated external IP:**

When you set `type: LoadBalancer` on a Service, the cloud provider provisions a load balancer (AWS NLB, Azure LB, GCP Load Balancer) with a public IP that routes directly to your pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

**Problem:** Every Service gets its own cloud load balancer. At $18-25/month each on AWS, 20 services = $400-500/month just on load balancers.

**Ingress Controller — for HTTP/HTTPS traffic (recommended for most services):**

One cloud load balancer sits in front of an **Ingress Controller** (NGINX, Traefik, AWS ALB). The controller reads Ingress rules and routes traffic based on hostname and path — all through a single LB.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

**Which one we use and why:**

| Scenario | Choice | Reason |
|---|---|---|
| HTTP/HTTPS microservices | NGINX Ingress Controller | One LB for all services, host/path routing, TLS termination |
| Internal services | ClusterIP | No external exposure needed |
| Non-HTTP (gRPC, TCP, databases) | Service `LoadBalancer` or NodePort | Ingress only handles HTTP |
| AWS-native auth (Cognito, WAF) | AWS ALB Ingress Controller | Native integration with AWS services |

**Our production setup:**

We run **NGINX Ingress Controller** behind a single AWS NLB. All HTTP/HTTPS traffic enters through one NLB → NGINX routes to the correct service by hostname. TLS is terminated at NGINX using certs managed by cert-manager (auto-renewed from Let's Encrypt).

```
User → Route53 → AWS NLB → NGINX Ingress Controller → Service → Pod
```

For internal service-to-service communication, everything uses `ClusterIP` — no load balancer involved, just K8s DNS + kube-proxy round-robin.

**Real scenario:** When we first set up the cluster, each of our 12 microservices had `type: LoadBalancer`. Monthly bill for load balancers alone: $216. We switched to a single NGINX Ingress Controller with host-based routing rules. All 12 services went behind one NLB. Monthly LB cost dropped to $18 — a 92% reduction with zero change in functionality. The switch took one afternoon: create the Ingress resources, update DNS records in Route53, delete the old LoadBalancer services.

**When NOT to use an Ingress Controller:**
- If the service needs raw TCP (e.g., a custom protocol, a database exposed externally) — use `LoadBalancer` directly
- If you're on a bare-metal cluster without a cloud provider — use MetalLB to get LoadBalancer functionality, or use NodePort + an external load balancer

---
