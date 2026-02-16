# Medium Level Interview Questions

Intermediate-level Kubernetes interview questions and answers.

---

## 1. How does DNS resolution work inside a pod? And what do you check when a service isn't reachable by name?

Every pod gets `/etc/resolv.conf` injected automatically, pointing to CoreDNS (usually `10.96.0.10`). It has search domains like `default.svc.cluster.local`, so when you do `curl my-service`, K8s expands it to `my-service.default.svc.cluster.local` and resolves it via CoreDNS.

**The thing most people miss — `ndots:5`.** If the hostname has fewer than 5 dots, K8s treats it as a relative name and appends search domains first. So even `api.stripe.com` (2 dots) triggers 4 extra DNS lookups before trying the actual domain. In a high-throughput service making 1000 requests/sec, that's 4000 unnecessary DNS queries/sec. We fixed this by adding a trailing dot (`api.stripe.com.`) which tells the resolver "this is absolute, don't append anything."

**When a service isn't reachable by name, my checklist:**

1. `kubectl exec -it <pod> -- nslookup kubernetes.default` — if this fails, CoreDNS itself is broken, not your service
2. `kubectl get pods -n kube-system -l k8s-app=kube-dns` — are CoreDNS pods running? I've seen them go OOMKilled on clusters with 500+ services
3. `kubectl get endpoints <svc>` — if empty, the service selector doesn't match any pod labels. This is the #1 cause. I once spent 30 minutes debugging DNS when the real issue was a typo: selector said `app: myapp` but pods had `app: my-app`
4. Cross-namespace check — calling `my-service` from a different namespace won't work. You need `my-service.other-namespace.svc.cluster.local`
5. Check `dnsPolicy` on the pod — if set to `Default`, the pod uses the node's DNS instead of CoreDNS. Cluster-internal names won't resolve at all
6. Check NetworkPolicies — a deny-all egress policy silently blocks DNS (port 53/UDP). The pod just hangs on any DNS lookup with no error message. This one burned us in production twice

**Real scenario:** We had intermittent DNS failures affecting ~5% of queries. After two days of debugging, root cause was `conntrack` table exhaustion. DNS uses UDP, conntrack table was full, kernel silently dropped new UDP packets. Fix: increased `nf_conntrack_max` and deployed `NodeLocal DNSCache` to reduce load on CoreDNS.

---

## 2. Kubernetes pods are in Running state, but traffic is not reaching the app. List the exact components you validate.

I go layer by layer, outside-in. The trick is to **narrow down whether the issue is above the Service layer or below it.**

**Quick shortcut first:** `kubectl exec <another-pod> -- curl -s <service-name>:<port>`. If this works → issue is above the Service (ingress, LB, external DNS). If this fails → issue is at the Service/pod level.

**Then I validate each layer:**

1. **Readiness probe** — `kubectl get pods`. Is the pod `Ready` (1/1) or just `Running` (0/1)? If readiness probe fails, K8s removes the pod from Service endpoints. The pod is alive but receives zero traffic. I've seen this when a developer added a readiness probe hitting `/healthz` but the app exposed `/health` — one character difference, all traffic dropped.

2. **Service endpoints** — `kubectl get endpoints <svc>`. If empty, selector doesn't match pod labels. This is the most common cause. Run `kubectl describe svc <svc>` and compare the `Selector` field with `kubectl get pods --show-labels`.

3. **Port mapping** — Service `targetPort` must match the port your app listens on. Example: Service has `targetPort: 8080` but app listens on `3000`. Traffic hits the pod but nothing is listening on that port → connection refused. I verify with `kubectl exec <pod> -- netstat -tlnp` to see what's actually listening.

4. **Container binding address** — If the app binds to `127.0.0.1`, it only accepts connections from inside the container itself. Traffic from the Service (which comes from the pod network) is rejected. The app must bind to `0.0.0.0`. We caught this once with a Go service that defaulted to localhost.

5. **NetworkPolicy** — `kubectl get networkpolicy -n <namespace>`. A deny-all ingress policy blocks all incoming traffic to pods. The pod is healthy, the service selector matches, but the network layer drops every packet silently.

6. **Ingress/LB** — For external traffic, `kubectl describe ingress <name>`. Check if the backend service name and port are correct. Check TLS — if TLS is misconfigured, the ingress controller returns 503 before traffic even reaches the pod.

7. **kube-proxy** — Rare, but if kube-proxy is down on a node, iptables/ipvs rules aren't created for Services. Pods on that node can't reach any Service by ClusterIP. Check `kubectl get pods -n kube-system -l k8s-app=kube-proxy`.

---

## 3. Explain how to build a Docker image and deploy it to Kubernetes using CI/CD.

Three pieces: **Dockerfile**, **Kubernetes manifest**, and **CI/CD pipeline**. I'll walk through each with what I'd actually use in production.

**Dockerfile:**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```
Key points: Alpine for smaller image, `npm ci` instead of `npm install` for deterministic builds, `USER node` for non-root execution.

**Kubernetes Deployment + Service:**
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
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 3000
```

**Jenkins pipeline:**
```groovy
pipeline {
  stages {
    stage('Test')    { steps { sh 'npm test' } }
    stage('Scan')    { steps { sh 'trivy image --exit-code 1 my-app:${BUILD_NUMBER}' } }
    stage('Build')   { steps { sh 'docker build -t myrepo/my-app:${BUILD_NUMBER} .' } }
    stage('Push')    { steps { sh 'docker push myrepo/my-app:${BUILD_NUMBER}' } }
    stage('Deploy')  { steps { sh 'kubectl set image deployment/my-app my-app=myrepo/my-app:${BUILD_NUMBER}' } }
  }
}
```

**What I'd do differently in production:** Instead of `kubectl set image`, I'd use ArgoCD (GitOps). The pipeline updates the image tag in Git, and ArgoCD syncs it to the cluster. This gives us an audit trail, rollback via `git revert`, and no direct cluster access from CI.

**Real scenario:** We had a pipeline that pushed to ECR and deployed directly. One night, a developer merged a broken image. By the time we noticed, the old image was already deregistered from ECR. We couldn't rollback because the previous image tag was gone. After that, we switched to immutable tags and GitOps — the image is always traceable and rollback is always possible.

---

## 4. What security best practices have you implemented in Kubernetes?

> **Also asked as:** "How does Kubernetes handle security and access control?"

I'll share what we actually run in production, not just theory:

- **Non-root containers** — Every Dockerfile uses `USER nonroot`. SecurityContext enforces `runAsNonRoot: true` and `allowPrivilegeEscalation: false`. Why it matters: A container running as root can modify the node's filesystem via volume mounts. We caught a third-party image running as root during a security audit — it had write access to the host's `/var/log`.

- **Read-only root filesystem** — `readOnlyRootFilesystem: true` in securityContext. The app writes temp files to a mounted `emptyDir` instead. This prevents attackers from writing malicious binaries into the container. When we enabled this, 3 services broke because they were writing to `/tmp` inside the container — we fixed them by mounting `emptyDir` at `/tmp`.

- **Network Policies** — Default deny-all in every namespace. Then explicitly whitelist pod-to-pod communication. Example: the API pod can talk to the database pod, but the frontend pod cannot. Without this, any compromised pod can scan the entire cluster network.

- **Secrets management** — External Secrets Operator syncing from AWS Secrets Manager. Secrets never exist in Git or plain manifests. When a secret rotates in AWS, ESO updates the K8s Secret automatically. Before this, we had database passwords hardcoded in ConfigMaps — one leaked via a kubectl get command in a shared terminal session.

- **Image policies** — Kyverno enforces: images must come from our private ECR only, no `latest` tag allowed, no images without a vulnerability scan report. This stops someone from deploying `docker.io/random-image` to production.

- **RBAC** — Developers get read-only access to prod (`get`, `list`, `watch`). Deployments only happen through CI/CD service accounts with scoped permissions. No one has `cluster-admin` except break-glass accounts.

- **Pod Security Admission** — `restricted` profile enforced on all prod namespaces. We started with `warn` mode for 2 weeks to identify violations, then switched to `enforce`.

---

## 5. Where and how do you store secrets securely in Kubernetes?

**Never** rely on plain K8s Secrets alone — they're base64 encoded, not encrypted. Anyone with `kubectl get secret -o yaml` access can read them.

**What we use in production:** External Secrets Operator (ESO) with AWS Secrets Manager.

The secret lives in AWS. ESO watches for `ExternalSecret` resources and syncs them into K8s Secrets automatically:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-creds
  data:
    - secretKey: password
      remoteRef:
        key: prod/db-password
```

**Why not Vault directly?** We evaluated it. Vault is powerful but adds operational overhead — you need to manage the Vault cluster, unsealing, HA, backups. For our scale, AWS Secrets Manager + ESO gives us automatic rotation, IAM-based access control, and audit logging without managing another stateful service.

**Other practices we follow:**
- **etcd encryption at rest** — enabled via EncryptionConfiguration so K8s Secrets are encrypted in etcd, not just base64
- **Mount secrets as files, not env vars** — env vars leak into `kubectl describe pod` output, process listings (`/proc/1/environ`), and crash dumps. Files are safer.
- **RBAC on secrets** — only the ServiceAccount of the specific pod can read its own secrets. Not every SA in the namespace.

**Real scenario that convinced us:** Before ESO, a developer accidentally ran `kubectl get secret db-creds -o yaml` during a screen-share demo. The base64-encoded password was visible on screen for 10 seconds before anyone noticed. We rotated the password immediately, but that incident pushed us to external secrets management and stricter RBAC.

---

## 6. What's the real difference between CPU bottleneck vs I/O bottleneck, and how do you identify each in Kubernetes?

They feel similar (app is slow) but the root cause is completely different, and the fix is opposite.

**CPU bottleneck** — the app has work to do but can't get enough CPU cycles.

How to identify:
- `kubectl top pods` shows CPU usage near or at the limit
- `container_cpu_cfs_throttled_periods_total` is high in Prometheus — the kernel is actively pausing your process
- Response times increase **linearly** with load — more requests = proportionally slower
- CPU usage graph looks like a flat line at the limit (it's capped)

Real scenario: Our Go API service had a 500m CPU limit. During peak traffic, it hit the limit and CFS throttling kicked in. Every request took 3x longer because the kernel paused the process for 60% of each 100ms scheduling period. The app didn't crash, didn't error — just got slower. We removed the CPU limit (kept request at 500m for scheduling) and latency returned to normal. Node-level CPU was only at 40% — there was plenty of headroom, but the pod limit was the constraint.

Fix: Increase CPU limits, remove CPU limits entirely (use only requests), or scale horizontally.

**I/O bottleneck** — the app is waiting for disk reads/writes or network responses. CPU is idle because the thread is blocked waiting, not computing.

How to identify:
- `kubectl top pods` shows **low** CPU usage — this is the key difference. CPU is idle because the app is waiting.
- `node_disk_io_time_seconds_total` is high — disk is busy
- `iowait` in node metrics (`node_cpu_seconds_total{mode="iowait"}`) is elevated — CPUs are idle waiting for I/O
- Response times are **inconsistent** — some requests fast, some very slow (depends on disk queue depth)
- Network I/O: distributed traces show long spans on database calls or external API calls

Real scenario: A team ran PostgreSQL on a gp2 EBS volume (default 100 IOPS for small volumes). During peak hours, read queries queued up — some taking 500ms that normally took 5ms. CPU on the database pod was at 10%. The team kept increasing CPU limits, which did nothing. The bottleneck was disk, not CPU. Fix: migrated to gp3 with 3000 baseline IOPS — query latency dropped by 90%.

**The quick test:** If CPU is near the limit → CPU bottleneck. If CPU is low but the app is slow → I/O bottleneck (disk, network, DNS, or connection pool wait). Always check `iowait` and distributed traces before assuming it's a CPU problem.

---

## 7. Why do systems fail even when CPU and memory look fine?

This question tests whether you understand that CPU and memory are just two of many failure dimensions. Here are the ones that catch people off guard:

**1. CPU throttling (hidden CPU issue)** — Average CPU looks fine (30%), but CFS throttling is at 70%. The kernel pauses your app during burst periods. Prometheus shows healthy averages, but p99 latency is 5x higher. This is the most common "ghost" issue in Kubernetes. Your dashboard says 30% CPU, but the pod is being throttled 70% of the time.

**2. Connection pool exhaustion** — Your app has 10 database connections. 50 concurrent requests arrive. 40 threads are blocked waiting for a free connection. CPU is idle (threads are waiting, not computing), memory is stable, but the app is frozen. We had a microservice that worked fine at 100 req/sec but fell apart at 300 req/sec — the HikariCP pool was maxed out. The fix was increasing pool size from 10 to 30 and adding connection pool wait-time metrics to our dashboards.

**3. File descriptor limits** — Every network connection, open file, and socket uses a file descriptor. Default container limit is often 1024. A service handling 2000 concurrent WebSocket connections hits this limit and starts refusing new connections with `too many open files`. CPU and memory are fine. Fix: set `ulimits` in the container spec or increase via security context.

**4. DNS failures** — CoreDNS is overwhelmed or conntrack table is full. Every outbound request hangs for 5 seconds waiting for DNS resolution that never completes. The app eventually times out. No CPU issue, no memory issue — just DNS packets being silently dropped. We had this in production — 5% of requests randomly took 5+ seconds. Root cause: conntrack table at capacity, UDP DNS packets dropped.

**5. Network policies blocking traffic** — A new NetworkPolicy was applied that accidentally blocked egress to a dependency. The app starts, health checks pass (they check localhost), but every request to the database times out silently. CPU idle, memory stable, app appears healthy.

**6. Disk pressure** — The node's disk is 95% full. Kubelet starts evicting pods, container logs can't be written, image pulls fail. The pod might still show `Running` for a while but can't function properly. `kubectl describe node` shows `DiskPressure: True`.

**7. External dependency failure** — Your service is fast, but it calls a payment API that's returning 500s. Your service retries 3 times (silently), each with a 2-second timeout. One request now takes 6 seconds. CPU and memory of your pod are fine — the bottleneck is entirely outside your cluster.

**Key insight for interviews:** CPU and memory are just the visible layer. Real production failures come from connection limits, DNS, disk I/O, network policies, and external dependencies. A senior engineer checks all of these, not just the top two.

---

## 8. Why do deployments succeed but users still see errors?

The deployment succeeded means K8s is happy — pods are running, the rollout completed. But K8s doesn't validate whether your app actually works correctly end-to-end. Here's why users still see errors:

**1. Readiness probe is too shallow** — The probe checks `/health` which returns 200 as soon as the HTTP server starts. But the app hasn't loaded configs, warmed caches, or established database connections yet. K8s marks the pod as Ready, traffic arrives, app fails. Real scenario: Our Spring Boot service returned 200 on `/actuator/health` after 5 seconds, but the full initialization (DB connection pool, cache warm-up, config loading) took 40 seconds. For 35 seconds, every request to new pods returned 500.

**2. Lifecycle race condition** — Old pods terminate before the ingress controller removes them from the upstream list. For a few seconds, NGINX sends traffic to a dead pod IP → 502. This is the most common cause of deploy-related errors. Fix: add a `preStop` hook with `sleep 5-10` seconds so the old pod stays alive while endpoints and ingress update.

**3. Breaking API changes** — During a rolling update, both v1 and v2 serve traffic simultaneously. If v2 changes the response format or database schema, v1 and v2 responses are inconsistent. Users on v1 see old behavior, users on v2 see new behavior, and downstream services parsing the response might break on either. Fix: make changes backward-compatible. Deploy schema changes separately from code changes.

**4. Config/secret mismatch** — The new image expects a new environment variable (`FEATURE_FLAG_X`) that wasn't added to the ConfigMap. The app starts, health check passes (it doesn't check for this specific env var), but the feature that depends on it throws null pointer errors. K8s doesn't validate that your ConfigMap has every env var your app needs.

**5. Missing endpoints window** — If `maxUnavailable` is too high and readiness probe has high `initialDelaySeconds`, there's a window where old pods are terminated but new pods aren't ready yet. The Service has fewer (or zero) endpoints → errors. Fix: set `maxUnavailable: 0` for critical services, meaning K8s always keeps the current replicas running until new ones are confirmed ready.

**6. Resource limits too tight** — New version uses slightly more memory than the old one (new feature, new dependency). The limit is the same. Under production load, pods hit OOMKill, restart, hit OOMKill again. The deployment "succeeded" (pods were created), but they're constantly dying under load. `kubectl describe pod` shows `Last State: OOMKilled` but the pod might show Running at any given moment because it restarts fast.

**Bottom line:** A successful deployment only means "K8s created the pods." It doesn't mean the app works. That's why you need: proper readiness probes, preStop hooks, smoke tests after deploy, and alerting on error rates — not just deployment status.

---

## 9. What's the difference between metrics, logs, and traces, and when do you start with each?

> **Also asked as:** "Describe Grafana and ELK/PLG stack to someone who doesn't know them" · "What monitoring and logging tools do you use?"

These are the three pillars of observability. Each answers a different question:

**Metrics** — "What is happening right now, numerically?" Numbers over time. CPU usage, request rate, error rate, latency percentiles. Metrics are cheap to store, fast to query, and great for dashboards and alerts. You can look at a Grafana dashboard and instantly see "error rate spiked to 5% at 3:42 PM."

**Logs** — "What exactly happened, in detail?" Text records of events. `ERROR: connection refused to postgres:5432`. Logs are expensive to store (especially at scale), slow to search, but contain the richest detail. You use logs to find the exact error message, stack trace, or sequence of events.

**Traces** — "What was the journey of a single request across services?" A distributed trace shows: user hit the API gateway (5ms) → API called the auth service (15ms) → auth called the database (200ms) → total: 220ms. Traces show you exactly where time is spent across your microservices.

**When to start with each:**

**Start with metrics when:** "Something is wrong, but I don't know what." Metrics give you the big picture fast. I look at the RED dashboard: Request rate dropped? Error rate spiked? Duration increased? This tells me which service is affected and when it started. Real scenario: Alert fires — "order-service error rate > 5%." I check Grafana, see error rate spiked at 14:32, correlate with a deployment at 14:30. Now I know where to look.

**Start with logs when:** "I know which service is broken, I need the error message." After metrics tell me the order-service is failing, I check logs: `kubectl logs -n prod -l app=order-service --since=10m`. I find: `ERROR: SQLSTATE[42P01] table "orders_v2" does not exist`. Now I have the exact root cause — a database migration didn't run.

**Start with traces when:** "The request is slow, but I don't know which service or which call is the bottleneck." Metrics show latency is high, but the order-service itself only takes 10ms of CPU time. Where's the other 500ms going? I open Jaeger, find a slow trace, and see: order-service (10ms) → inventory-service (15ms) → payment-gateway (480ms). The external payment gateway is slow. Without tracing, I'd be guessing which service to investigate.

**Real-world example of using all three together:**

Customer reports: "Checkout is slow." (no other detail)
1. **Metrics first** — Grafana shows checkout endpoint p99 latency jumped from 200ms to 2.5s at 10 AM
2. **Traces second** — I filter Jaeger for slow checkout traces. I see the `payment-service → stripe-api` span is taking 2 seconds. All other spans are normal.
3. **Logs third** — I check payment-service logs and find: `WARN: Stripe API rate limit exceeded, retrying in 1s`. We were hitting Stripe's rate limit, and the retry logic was adding 2 seconds to every rate-limited request.

Fix: Implemented request queuing with exponential backoff and increased our Stripe rate limit tier.

**The tooling we use:**
- Metrics: Prometheus + Grafana (alerts via Alertmanager → PagerDuty)
- Logs: Loki (integrated with Grafana, so metrics and logs are in one place)
- Traces: Tempo (also integrated with Grafana — click from a metric panel to the matching trace)

Having all three in Grafana means I can go from "what happened" (metrics) → "where did it happen" (traces) → "why did it happen" (logs) without switching tools.

---

## 10. If a pod is down, how do you troubleshoot it?

"Pod is down" can mean several things — Pending, CrashLoopBackOff, Error, ImagePullBackOff, or Terminating. Each has a different root cause. My first step is always: **what state is the pod actually in?**

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

**If `Pending`** — the pod can't be scheduled.
- Check the `Events` section in `describe`: `Insufficient cpu` or `Insufficient memory` → no node has enough resources. Either scale the cluster or reduce resource requests.
- `0/5 nodes are available: 5 node(s) had taint {dedicated: gpu}` → the pod doesn't tolerate the node taint. Add a toleration or use a different node pool.
- PVC is `Pending` → StorageClass can't provision the volume, or the volume is in a different AZ than the node.
- Real scenario: A pod was stuck in Pending for 20 minutes. All nodes had enough CPU but the pod had a `nodeSelector` requiring `disktype: ssd`. No node had that label. Someone had relabeled the nodes during maintenance and forgot to re-add custom labels.

**If `ImagePullBackOff`** — K8s can't pull the container image.
- Wrong image tag (typo, tag doesn't exist in registry)
- Private registry but no `imagePullSecret` configured on the pod/ServiceAccount
- Registry is down or unreachable from the node
- `kubectl describe pod` shows the exact pull error. I once spent 15 minutes debugging this until I noticed the image was `myrepo/app:v1.2` but the actual tag was `myrepo/app:v1.2.0` — one missing `.0`.

**If `CrashLoopBackOff`** — container starts and immediately crashes.
- `kubectl logs <pod> --previous` — shows the last crash's logs
- `kubectl describe pod` → check `Last State` → `Reason: OOMKilled` (memory limit too low) or `Exit Code: 1` (app error) or `Exit Code: 127` (command not found)
- Most common: OOMKilled, missing env vars, or can't connect to a dependency at startup

**If `Running` but not `Ready` (0/1)** — readiness probe is failing.
- Check the probe path: is it `/health` or `/healthz`? Does it match what the app exposes?
- Check `targetPort` — probe might check port 8080 but app listens on 3000
- `kubectl describe pod` shows `Readiness probe failed: connection refused` or `HTTP probe failed with status code: 404`

**If `Terminating` and stuck** — pod won't die.
- Usually a finalizer is blocking deletion, or the app doesn't handle SIGTERM gracefully
- `kubectl get pod <pod> -o yaml | grep finalizer` — if a finalizer is present, the controller that owns it might be down
- Force delete (last resort): `kubectl delete pod <pod> --grace-period=0 --force`

**My debugging flow in 30 seconds:**
```
kubectl get pods → What state?
kubectl describe pod → What events? What's blocking?
kubectl logs --previous → What error killed it?
```
These three commands solve 90% of pod issues.

---

## 11. How do you manage Kubernetes YAML manifests across environments?

Raw YAML files copied across staging/prod is a disaster waiting to happen. Here's how we manage it:

**Helm charts** — We template our manifests. The chart has a `values.yaml` for defaults, then environment-specific overrides:

```
my-app/
├── Chart.yaml
├── values.yaml           # defaults
├── values-staging.yaml   # staging overrides
├── values-prod.yaml      # prod overrides
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

In `templates/deployment.yaml`:
```yaml
replicas: {{ .Values.replicas }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
resources:
  requests:
    cpu: {{ .Values.resources.requests.cpu }}
```

`values-staging.yaml` has `replicas: 2, cpu: 100m`. `values-prod.yaml` has `replicas: 5, cpu: 500m`. Same template, different configs.

**ArgoCD (GitOps)** — Each environment has its own ArgoCD Application pointing to the same chart but different values file:
```yaml
# ArgoCD Application for prod
source:
  repoURL: https://github.com/our-org/k8s-manifests
  path: charts/my-app
  helm:
    valueFiles:
      - values-prod.yaml
```

Changes go through Git PRs → code review → merge → ArgoCD auto-syncs. No one runs `kubectl apply` manually.

**Why not Kustomize?** We evaluated it. Kustomize works well for simple overlays (change replicas, add labels), but Helm is better when you need conditionals (`if ingress.enabled`), loops (generate multiple containers from a list), and a package ecosystem (community charts for Prometheus, NGINX, etc.). We use Kustomize only for quick patches on third-party Helm charts.

**Real scenario that convinced us:** Before Helm + ArgoCD, each team had their own folder of YAML files per environment. A developer updated the staging Deployment but forgot to update prod. Prod was running an image 3 versions behind staging. Nobody noticed for 2 weeks until a bug fix that was "already deployed" turned out to be only in staging. With Helm, the image tag is in one place per environment — `values-prod.yaml`. ArgoCD shows you if any environment is out of sync.

---

## 12. Where and how do you store cluster information securely?

"Cluster information" includes: kubeconfig files, cluster endpoints, CA certificates, ServiceAccount tokens, and infrastructure details. Each needs different handling.

**Kubeconfig files** — Never stored in Git, never shared via Slack/email. We use AWS SSO + `aws eks update-kubeconfig` to generate short-lived kubeconfig on the fly. Each engineer authenticates via SSO, gets temporary credentials, and the kubeconfig is generated locally with a 12-hour token expiry. If someone's laptop is stolen, the kubeconfig expires automatically.

For CI/CD: the pipeline uses an IAM role (not a static kubeconfig) to authenticate to EKS. No long-lived credentials stored anywhere.

**Cluster metadata and configs** — Stored in Terraform state (encrypted S3 bucket with versioning + DynamoDB for state locking). Cluster endpoint, VPC ID, node group configs — all managed as IaC. If someone asks "what's the prod cluster version?" — the answer is in `terraform.tfstate`, not someone's memory.

**Secrets (DB passwords, API keys)** — External Secrets Operator syncing from AWS Secrets Manager. Never in Git, never in ConfigMaps, never in plain K8s Secrets without encryption at rest.

**ServiceAccount tokens** — Since K8s 1.24, default SA tokens are no longer auto-mounted. We create bound tokens with expiry for any pod that needs API access:
```yaml
automountServiceAccountToken: false  # default for all pods
```
Only pods that explicitly need K8s API access get a ServiceAccount with scoped RBAC permissions.

**Real scenario:** Early in our K8s journey, someone committed a kubeconfig file to a private Git repo. The repo was later made public during an open-source migration. The kubeconfig had cluster-admin access. We discovered it during a security audit — the credentials were still valid 6 months later. After that incident, we:
- Rotated all cluster credentials
- Switched to SSO-based short-lived auth
- Added `git-secrets` pre-commit hooks to scan for credentials before push
- Set `automountServiceAccountToken: false` cluster-wide

**Rule of thumb:** If it grants access to the cluster, it should be short-lived, scoped, and never stored in Git. Treat kubeconfig like a password, not a config file.

---

## 13. How will you deploy an application using Argo CD?

ArgoCD follows the GitOps model: **Git is the single source of truth**. You define the desired state in a Git repo, ArgoCD watches it, and syncs the cluster to match.

**Step-by-step — how we do it:**

1. **Manifests live in Git** — We have a separate repo (`k8s-manifests`) with Helm charts or plain YAML for every service, organized by environment:
```
k8s-manifests/
├── apps/
│   └── order-service/
│       ├── Chart.yaml
│       ├── values-staging.yaml
│       └── values-prod.yaml
```

2. **Create an ArgoCD Application** — This tells ArgoCD what to watch and where to deploy:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/our-org/k8s-manifests
    path: apps/order-service
    targetRevision: main
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true      # Revert manual kubectl changes
```

3. **Deploy flow**: Developer merges code → CI pipeline builds image, pushes to ECR, updates `values-prod.yaml` with new image tag → ArgoCD detects the Git change → syncs the new manifests to the cluster → new pods roll out.

**Why we chose ArgoCD over `kubectl apply` in CI:**

Real scenario: A developer ran `kubectl apply` manually on a Friday night with untested changes. The service broke. There was no record of what changed — `kubectl apply` doesn't create audit trails. With ArgoCD:
- Every change is a Git commit — full audit trail
- Rollback = `git revert` → ArgoCD auto-syncs the previous state
- `selfHeal: true` reverts any manual `kubectl edit` — nobody can make untracked changes
- We see the diff before syncing in the ArgoCD UI

**Key features we use:**
- **Auto-sync** — ArgoCD polls Git every 3 minutes (or uses webhooks for instant sync)
- **Sync waves** — deploy ConfigMaps first, then Deployments, then Ingress. Ordered deployment.
- **Health checks** — ArgoCD waits for pods to be Running+Ready before marking the sync as successful. If pods fail, it marks the sync as `Degraded`.

---

## 14. What is a PDB (PodDisruptionBudget) in Kubernetes?

A PDB tells Kubernetes: "During voluntary disruptions, don't take down more than X pods at once." It protects your application's availability during node drains, cluster upgrades, and autoscaler scale-downs.

**What are voluntary disruptions?**
- `kubectl drain` (before node maintenance)
- Cluster autoscaler removing an underutilized node
- Cluster upgrades rolling node pools
- Spot instance reclamation

**What are NOT voluntary disruptions (PDB won't help):**
- Node crash, hardware failure
- OOMKilled, kernel panic
- Pod crash (CrashLoopBackOff)

**Example:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
spec:
  minAvailable: 2          # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: order-service
```

Or alternatively:
```yaml
spec:
  maxUnavailable: 1        # At most 1 pod can be down at a time
```

**How it works in practice:** You have 4 pods and `minAvailable: 2`. When you `kubectl drain node-1`, K8s checks: "If I evict the pod on node-1, will there still be 2 pods?" If yes → eviction proceeds. If evicting would drop below 2 → K8s waits until another pod is healthy elsewhere before proceeding.

**Real scenario where PDB saved us:**

We ran a 3-replica Elasticsearch cluster without PDB. During a cluster upgrade, K8s drained 2 nodes simultaneously. Two ES pods were evicted at the same time — the cluster lost quorum and went read-only. Data wasn't lost, but writes were blocked for 15 minutes until pods rescheduled.

After that, we added:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: elasticsearch-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: elasticsearch
```

Now during upgrades, K8s drains one ES pod at a time, waits for it to rejoin the cluster as healthy, then proceeds to the next. Zero downtime.

**Common mistakes:**
- `minAvailable` equals replica count (e.g., 3 pods, `minAvailable: 3`) → nothing can ever be evicted, node drains hang forever. The cluster upgrade gets stuck.
- No PDB at all on stateful services → autoscaler or upgrades can evict multiple pods simultaneously, causing data loss or outage.
- Using PDB on single-replica deployments with `minAvailable: 1` → the node can never be drained. Use `maxUnavailable: 1` instead if you have only 1 replica.

**Rule of thumb:** Every production service with 2+ replicas should have a PDB. Set `maxUnavailable: 1` for most services. For critical stateful services (databases, message brokers), use `minAvailable: N-1` where N is the quorum requirement.

---

## 15. How do you integrate ArgoCD with Jenkins?

They handle different parts of the pipeline. **Jenkins = CI** (build, test, scan). **ArgoCD = CD** (deploy to K8s). They connect through Git.

**The flow:**

```
Developer pushes code
    → Jenkins triggers (webhook)
    → Jenkins: run tests, build image, push to ECR, scan with Trivy
    → Jenkins: update image tag in the Git manifests repo
    → ArgoCD: detects Git change, syncs new manifests to cluster
    → ArgoCD: monitors rollout health
```

**Jenkins pipeline (CI part):**
```groovy
pipeline {
  stages {
    stage('Test')  { steps { sh 'npm test' } }
    stage('Build') { steps { sh 'docker build -t $ECR_REPO:$BUILD_NUMBER .' } }
    stage('Scan')  { steps { sh 'trivy image --exit-code 1 $ECR_REPO:$BUILD_NUMBER' } }
    stage('Push')  { steps { sh 'docker push $ECR_REPO:$BUILD_NUMBER' } }
    stage('Update Manifest') {
      steps {
        // Clone the manifests repo, update the image tag, push
        sh '''
          git clone https://github.com/our-org/k8s-manifests.git
          cd k8s-manifests
          sed -i "s|image:.*|image: $ECR_REPO:$BUILD_NUMBER|" apps/my-app/values-prod.yaml
          git add . && git commit -m "Update my-app to build $BUILD_NUMBER"
          git push
        '''
      }
    }
  }
}
```

**ArgoCD (CD part):** Already configured to watch the `k8s-manifests` repo. When Jenkins pushes the updated image tag, ArgoCD detects the change (via webhook or polling) and syncs it to the cluster automatically.

**Why this separation matters:**

- **Jenkins never touches the cluster directly** — no kubeconfig in Jenkins, no `kubectl` commands. Jenkins only writes to Git. This is more secure — if Jenkins is compromised, the attacker can't run arbitrary commands on the cluster.
- **ArgoCD owns the cluster state** — it ensures the cluster matches Git, and reverts any manual changes. If someone `kubectl edit`s a deployment, ArgoCD reverts it within minutes.
- **Rollback is Git-native** — `git revert` the Jenkins commit → ArgoCD syncs the old image tag → previous version deploys. No need to figure out what was running before.

**Real scenario:** Before this integration, Jenkins deployed directly to K8s via `kubectl set image`. One day, Jenkins credentials were rotated but the pipeline wasn't updated — deploys silently failed for 3 days. Nobody noticed because the old pods kept running. With the Git-based approach, even if Jenkins breaks, the last known good state is always in Git, and ArgoCD keeps the cluster in sync.

**Alternative to sed:** For Helm values, we use `yq` instead of `sed` to avoid regex issues:
```bash
yq -i '.image.tag = "'$BUILD_NUMBER'"' apps/my-app/values-prod.yaml
```

---

## 16. EKS vs self-managed Kubernetes — when would you choose each?

**EKS (or any managed K8s)** — 90% of the time, this is the right answer. AWS manages the control plane (API server, etcd, scheduler, controller-manager). You don't patch it, back it up, or worry about HA. Upgrades are a button click. You focus on workloads, not infrastructure.

**Choose EKS when:**
- Team is small (< 5 DevOps engineers) — you don't have the bandwidth to run a control plane
- Need to move fast — EKS is production-ready day one. Self-managed takes weeks to harden
- Compliance requires managed services — SOC2/HIPAA auditors are more comfortable with managed services that have built-in encryption, logging, and access controls
- Want native AWS integrations — ALB Ingress Controller, IAM Roles for Service Accounts (IRSA), EBS CSI driver, CloudWatch Container Insights — all work out of the box

**Choose self-managed (kubeadm/k3s/RKE) when:**
- **On-prem/air-gapped** — no cloud provider, or strict data residency that prohibits cloud. Banks, defense contractors, healthcare with strict regulations.
- **Cost at massive scale** — EKS charges $0.10/hr per cluster ($73/month). If you're running 50+ clusters, that's $3,650/month just for control planes. At that scale, self-managed can be cheaper — but factor in the engineering cost.
- **Need control plane customization** — custom admission controllers, specific etcd tuning, non-standard API server flags. EKS doesn't let you modify control plane configuration.
- **Edge/IoT** — k3s on ARM devices, lightweight clusters on edge nodes where EKS doesn't make sense.

**Real scenario:** We started with self-managed K8s (kubeadm on EC2). Two engineers spent 30% of their time on etcd backups, certificate rotation, control plane upgrades, and etcd defragmentation. We migrated to EKS and those engineers now build platform tooling instead. The $73/month per cluster is nothing compared to engineering time.

**Trade-off summary:** EKS = you trade control and money for operational simplicity. Self-managed = you trade engineering time for full control. Unless you have a specific reason, start with managed.

---

## 17. What breaks if your readinessProbe is misconfigured?

A misconfigured readinessProbe is one of the sneakiest issues — the pod is running, the app might be healthy, but **traffic routing is broken**.

**If the probe is too aggressive (low timeout, high frequency):**
The probe fails intermittently during GC pauses, high load, or slow DB queries. K8s removes the pod from the Service endpoint. Traffic shifts to other pods, overloading them, causing their probes to fail too. Cascading failure — all pods marked unready, **zero pods receiving traffic**, service is down even though every pod is running.

**If the probe checks the wrong thing:**
We had a readinessProbe hitting `/health` which only checked if the web server was up. But the app also needed a database connection. Pod was "ready" but couldn't serve requests — returning 500s on every database query. Users saw errors but K8s thought the pod was healthy. Fix: changed probe to `/ready` which verified DB connection pool and cache warmup.

**If the probe is too slow (high `initialDelaySeconds`):**
Pod starts and is ready to serve in 5 seconds, but `initialDelaySeconds: 60` means K8s won't check for a full minute. During deployments, new pods sit idle for 60 seconds while old pods handle all traffic. Rolling update takes 10 minutes instead of 2.

**If the probe is missing entirely:**
K8s considers the pod ready the moment the container starts. But if the app takes 30 seconds to initialize (load configs, warm caches, establish connections), those first 30 seconds of traffic hit an unready app. Users see errors during every deployment.

**Best practice:**
```yaml
readinessProbe:
  httpGet:
    path: /ready    # Checks actual dependencies, not just process alive
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```

**Real scenario:** After a deployment, 5% of requests were failing for 30 seconds. Readiness probe was hitting `/` (homepage) with `initialDelaySeconds: 0`. The container started instantly but the Spring Boot app took 25 seconds to initialize. K8s sent traffic immediately. Fix: added `initialDelaySeconds: 30` and changed the probe to `/actuator/health/readiness` which returns 200 only after all dependencies are connected.

---

## 18. Pods are stuck in Pending — how do taints and tolerations cause this?

`Pending` means the scheduler can't find a node to place the pod. Taints and tolerations are one of the most common reasons.

**How it works:** A **taint** on a node says "don't schedule here unless you tolerate me." A **toleration** on a pod says "I can handle that taint." If a pod doesn't have the right toleration, the scheduler skips that node.

**Common scenarios that cause stuck pods:**

**1. GPU/specialized nodes:** You taint GPU nodes with `gpu=true:NoSchedule` so regular pods don't waste GPU resources. But if your ML pod doesn't have the matching toleration, it can't schedule on GPU nodes and no other nodes have GPUs → Pending forever.

**2. Spot/preemptible nodes:** Tainted with `kubernetes.azure.com/scalesetpriority=spot:NoSchedule`. If your Deployment doesn't tolerate this, and all on-demand nodes are full, pods go Pending even though spot nodes have capacity.

**3. Node upgrades:** During a rolling node upgrade, old nodes get tainted `ToBeDeletedByClusterAutoscaler:NoSchedule`. If all healthy nodes are at capacity, new pods can't schedule anywhere.

**How to debug:**
```bash
# Check why the pod is pending
kubectl describe pod <pod-name>
# Look at Events: "0/5 nodes are available: 5 node(s) had untolerated taint"

# Check taints on all nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check tolerations on the pod
kubectl get pod <pod-name> -o jsonpath='{.spec.tolerations}'
```

**Fix:** Either add the matching toleration to the pod spec, or remove the taint from the node if it's no longer needed.

**Real scenario:** After enabling a new node pool on AKS, all pods from one team went Pending. The new node pool had a taint `team=data:NoSchedule` (intended for data team workloads), but the old node pool was decommissioned. The team's Deployments didn't have tolerations because they never needed them before. Fix: added tolerations to their Deployments and `nodeAffinity` to prefer the new pool. Lesson: always check taints when changing node pools.

---

## 19. What are the pre-requisites to upgrade a Kubernetes cluster?

Upgrading K8s is not `apt upgrade`. It can break workloads, APIs, and integrations if you skip preparation. Here's the checklist I follow before every upgrade.

**1. Check version skew policy**
K8s only supports upgrading **one minor version at a time** (1.27 → 1.28, not 1.27 → 1.29). The control plane must be upgraded before worker nodes. Kubelet can be up to 2 minor versions behind the API server, but don't rely on that — upgrade nodes promptly.

```bash
# Check current versions
kubectl version --short
kubectl get nodes -o wide   # Shows kubelet version per node
```

**2. Read the changelog for deprecated/removed APIs**
This is where most upgrades break. K8s removes APIs in major upgrades. For example, 1.25 removed PodSecurityPolicy, 1.22 removed `extensions/v1beta1` Ingress.

```bash
# Find resources using deprecated APIs
kubectl api-resources --api-group=extensions
# Or use a tool like pluto
pluto detect-all-in-cluster
```

We once upgraded from 1.24 to 1.25 without checking — all our PodSecurityPolicies stopped working. Pods that required privilege escalation started getting blocked. Rolled back within 15 minutes, but it caused a 15-minute outage for 3 teams.

**3. Backup etcd**
On self-managed clusters, snapshot etcd before upgrading. If the upgrade corrupts etcd, you lose everything.
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-pre-upgrade.db
```
On managed clusters (EKS/AKS/GKE), the provider handles etcd — but still backup your manifests via `velero backup create pre-upgrade-backup`.

**4. Check PodDisruptionBudgets**
During node upgrades, pods get drained. If PDBs are too restrictive (`minAvailable: 100%`), the drain hangs and the upgrade stalls.

```bash
kubectl get pdb -A
# Look for PDBs that would block draining
```

**5. Verify addon compatibility**
Ingress controllers, CNI plugins (Calico, Cilium), CSI drivers, metrics-server, cert-manager — each has a compatibility matrix with K8s versions. Upgrading K8s without upgrading addons can break networking, storage, or monitoring.

**6. Test in staging first**
Upgrade staging, run your full test suite, check that all workloads schedule correctly, probes pass, and traffic flows. Only then proceed to production.

**7. Drain nodes gracefully**
```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```
This evicts pods respecting PDBs and graceful termination periods.

**Real upgrade workflow:** Backup → upgrade control plane → verify API server healthy → upgrade nodes one by one (drain, upgrade kubelet, uncordon) → verify all pods running → upgrade addons → run smoke tests.

---

## 20. What monitoring tools do you use in Kubernetes? What alerts do you set up, and what are common pod management errors?

**Tools — the observability stack:**

- **Prometheus + Grafana** — Metrics collection and visualization. Prometheus scrapes `/metrics` endpoints from pods, nodes, and K8s components. Grafana dashboards for real-time visibility.
- **Loki** — Log aggregation. Like Prometheus but for logs. Collects logs from all pods via Promtail/Fluentbit. Query with LogQL in Grafana.
- **Alertmanager** — Routes Prometheus alerts to Slack, PagerDuty, email. Handles deduplication, silencing, and escalation.
- **Kube-state-metrics** — Exposes K8s object state as Prometheus metrics (pod status, deployment replicas, PVC status).

**Critical alerts we set up:**

```yaml
# Pod CrashLoopBackOff for > 5 minutes
- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
  for: 5m

# Pod stuck in Pending for > 10 minutes
- alert: PodPending
  expr: kube_pod_status_phase{phase="Pending"} == 1
  for: 10m

# Node NotReady
- alert: NodeNotReady
  expr: kube_node_status_condition{condition="Ready",status="true"} == 0
  for: 2m

# High memory usage (> 90% of limit)
- alert: ContainerMemoryHigh
  expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
  for: 5m

# OOMKilled detection
- alert: OOMKilled
  expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} > 0

# Deployment replica mismatch (desired ≠ available)
- alert: DeploymentReplicaMismatch
  expr: kube_deployment_spec_replicas != kube_deployment_status_available_replicas
  for: 10m
```

**Common pod management errors I've faced:**

**1. OOMKilled** — Most common. App exceeds memory limit. Usually a memory leak or undersized limit. Fix: analyze actual usage with `kubectl top`, increase limit or fix the leak.

**2. CrashLoopBackOff** — Container keeps crashing. Check `kubectl logs --previous` for the error. Usually missing env vars, wrong config, or a dependency that's unreachable at startup.

**3. ImagePullBackOff** — Can't pull the container image. Wrong tag, expired registry credentials, or network issue to the registry.

**4. Evicted pods** — Node under resource pressure (disk, memory). Kubelet evicts pods to protect the node. Check `kubectl describe node` for `DiskPressure` or `MemoryPressure`.

**5. Stuck terminating** — Pod won't delete. Usually a stuck finalizer or a container that ignores SIGTERM. Fix: check finalizers (`kubectl get pod -o jsonpath='{.metadata.finalizers}'`), or force delete as last resort.

**Real scenario:** We had no alerting on `Evicted` pods. A node's disk filled up from container logs (no log rotation). Kubelet started evicting pods silently — they rescheduled to other nodes, so the service stayed up, but 3 nodes lost half their pods. We didn't notice for 2 days. After that, we added alerts on `kube_pod_status_reason{reason="Evicted"} > 0` and disk usage alerts at 80% on every node.

---

## 21. How do you determine resource limits for your Kubernetes cluster?

Never guess. Start with data, then iterate.

**Step 1: Deploy without limits, observe actual usage**

In staging, deploy your app with only `requests` (no `limits`) and run realistic traffic for a few days. Monitor actual consumption:
```bash
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

For historical data, use Prometheus queries:
```promql
# P95 CPU usage over 7 days
quantile_over_time(0.95, rate(container_cpu_usage_seconds_total{pod=~"my-app.*"}[5m])[7d:])

# Peak memory usage over 7 days
max_over_time(container_memory_working_set_bytes{pod=~"my-app.*"}[7d])
```

**Step 2: Use VPA recommendations**

Deploy VPA in `Off` mode — it observes and recommends, but doesn't change anything:
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"
```
After a few days: `kubectl describe vpa my-app-vpa` → shows recommended CPU and memory based on actual usage patterns.

**Step 3: Set requests and limits based on data**

- **Requests** = average usage + 20% buffer. This is what the scheduler uses for placement — it guarantees this much resource.
- **Limits** = peak usage + 30% buffer. This is the ceiling — container gets OOMKilled or CPU-throttled beyond this.

```yaml
resources:
  requests:
    cpu: 250m       # Average is 200m
    memory: 256Mi   # Average is 200Mi
  limits:
    cpu: 500m       # Peak is 400m
    memory: 512Mi   # Peak is 400Mi
```

**Common mistakes:**

**Over-provisioning** — Copy-pasting `cpu: 2, memory: 4Gi` from a template when the app uses 100m CPU and 200Mi memory. We found a team doing this across 40 microservices — they were wasting 12 nodes worth of capacity. Fixing requests alone freed up $3,000/month.

**Setting CPU limits too tight** — CPU limits cause **throttling** via Linux CFS. A pod using 300m CPU with a 250m limit gets throttled even if the node has spare CPU. This causes latency spikes that don't show up in CPU usage metrics. Many teams now set CPU requests but **no CPU limits** — let pods burst when needed, rely on requests for fair scheduling.

**Setting memory requests ≠ limits** — If `requests < limits`, the pod gets QoS class `Burstable` and can be evicted under memory pressure. For critical services, set `requests = limits` to get `Guaranteed` QoS — these are the last pods evicted.

**Real scenario:** A Java service had `limits.memory: 1Gi` but the JVM was configured with `-Xmx768m`. The JVM heap never exceeded 768m, but total process memory (heap + metaspace + threads + native) hit 1.05Gi during traffic spikes → OOMKilled. We checked with `kubectl top` and it showed 980Mi peak. Set the limit to `1.5Gi` (2x heap as a rule of thumb for JVM). No more OOMKills.

---

## 22. What is a Kubernetes Operator?

An Operator is a **custom controller that extends Kubernetes** to manage complex, stateful applications automatically. It encodes human operational knowledge into software.

**The problem it solves:** K8s knows how to manage Deployments and StatefulSets, but it doesn't know how to manage a PostgreSQL cluster, an Elasticsearch cluster, or a Kafka broker. It can't handle database failovers, backup schedules, or schema migrations. An Operator teaches K8s how.

**How it works:**
1. You define a **Custom Resource Definition (CRD)** — a new API type (e.g., `kind: PostgresCluster`)
2. The Operator runs as a Deployment in the cluster, watching for these custom resources
3. When you create a `PostgresCluster` object, the Operator creates the StatefulSets, Services, PVCs, ConfigMaps, and handles replication, failover, and backups automatically

```yaml
# Instead of manually creating 15 K8s objects for PostgreSQL:
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: my-database
spec:
  postgresVersion: 15
  instances:
    - replicas: 3
      dataVolumeClaimSpec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi
  backups:
    pgbackrest:
      repos:
        - name: repo1
          s3:
            bucket: my-backups
            region: us-east-1
```

One YAML file → the Operator creates a 3-node PostgreSQL cluster with streaming replication, automatic failover, and S3 backups.

**Common Operators in production:**
- **Prometheus Operator** — manages Prometheus, Alertmanager, and ServiceMonitor resources
- **cert-manager** — automates TLS certificate issuance and renewal
- **Strimzi** — manages Kafka clusters
- **CloudNativePG / CrunchyData** — manages PostgreSQL

**Real scenario:** Before using the Prometheus Operator, adding monitoring for a new service meant editing Prometheus config files, restarting Prometheus, and hoping the config was valid. With the Operator, teams create a `ServiceMonitor` resource in their namespace — the Operator automatically updates Prometheus config. No manual edits, no restarts, no config mistakes.

---

## 23. What is Kubernetes Helm?

Helm is a **package manager for Kubernetes** — like `apt` for Ubuntu or `npm` for Node.js. It lets you template, package, and deploy K8s applications as reusable **charts**.

**The problem it solves:** A typical microservice needs 5-10 YAML files (Deployment, Service, ConfigMap, Ingress, HPA, PDB, ServiceAccount, etc.). Managing these manually across dev, staging, and production with different values is painful and error-prone.

**How it works:**
```
my-app-chart/
├── Chart.yaml          # Chart metadata (name, version)
├── values.yaml         # Default configuration values
├── values-prod.yaml    # Production overrides
├── templates/
│   ├── deployment.yaml # Templated K8s manifests
│   ├── service.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
```

**Templated manifest example:**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicas }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          resources:
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}
```

**Deploy with different values per environment:**
```bash
# Staging
helm install my-app ./my-app-chart -f values-staging.yaml

# Production
helm install my-app ./my-app-chart -f values-prod.yaml

# Upgrade
helm upgrade my-app ./my-app-chart -f values-prod.yaml --set image.tag=v1.2.4

# Rollback
helm rollback my-app 1   # Reverts to revision 1
```

**Why Helm matters:**
- **Reusability** — one chart deploys the same app across 5 environments with different configs
- **Versioning** — Helm tracks release history. `helm history my-app` shows every deployment
- **Ecosystem** — thousands of community charts. `helm install prometheus prometheus-community/kube-prometheus-stack` gives you a full monitoring stack in one command
- **Rollback** — `helm rollback` reverts all resources to a previous state, not just the Deployment

**Real scenario:** We had 30 microservices, each with hand-written YAML files. Every time we added a standard label, updated resource limits template, or added a new sidecar, we had to edit 30 sets of files. After creating a shared Helm chart, all 30 services use the same templates with different `values.yaml`. Adding a new standard label is now a one-line change in the chart, applied everywhere on next deploy.

---
