# Hard Level Interview Questions

Advanced-level Kubernetes interview questions and answers.

---

## 1. Walk me through what the controller manager does during a Deployment. No rollout status — reconciliation logic.

Three controllers work in a chain: **Deployment Controller → ReplicaSet Controller → Scheduler + Kubelet**. Each runs an independent reconciliation loop — compare desired vs actual, take action on the diff.

**Deployment Controller** watches Deployment objects via informers (not polling — it gets events pushed). When you `kubectl apply` a Deployment with a changed pod template, it computes a new template hash, creates a new ReplicaSet, and starts scaling the old RS down. It never touches pods directly — it only manages ReplicaSets.

**ReplicaSet Controller** counts pods matching its selector. Too few → creates pod objects (just API objects, not containers yet). Too many → deletes extras (preferring unready pods first, then newest).

**Scheduler** picks up pods with no `nodeName`, runs filtering (can this node run it? checks taints, affinity, resources) and scoring (which node is best?), then binds the pod to a node.

**Kubelet** on the assigned node pulls the image via CRI, sets up networking via CNI, mounts volumes, starts health probes, and reports status back.

**Why this matters practically:** I once had a rollout stuck at 50%. The Deployment controller had created the new ReplicaSet, but the Scheduler couldn't place the new pods — all nodes were at capacity. The Deployment controller didn't know or care why. It just saw "desired ≠ actual" and waited. Once Karpenter provisioned a new node, the Scheduler placed the pods, kubelet started them, and the Deployment controller continued the rollout. No coordination between controllers — each one independently converged. That's the beauty of reconciliation.

**Key insight for interviews:** If the interviewer asks "what if the controller-manager crashes mid-rollout?" — the answer is nothing breaks. When it restarts, each controller re-evaluates desired vs actual and picks up exactly where it left off. No state is lost because all state is in etcd.

---

## 2. How do you enforce runtime security in Kubernetes?

It's layered. Anyone who says "we use OPA" and stops there is covering one layer.

**Admission layer** — Pod Security Admission (replaced PSP in v1.25) with three profiles: `privileged`, `baseline`, `restricted`. We enforce `restricted` on all prod namespaces. For custom rules, we use **Kyverno** — block `latest` tag, require resource limits, restrict image registries. Example: A developer once tried to deploy a container from Docker Hub directly in production. Kyverno blocked it instantly with a clear message: "images must come from 123456789.dkr.ecr.us-east-1.amazonaws.com."

**Container layer** — Seccomp with `RuntimeDefault` profile restricts syscalls. AppArmor restricts file/network access. We caught a compromised container trying to mount the host filesystem — AppArmor blocked it before any damage.

**Network layer** — Default deny-all NetworkPolicy in every namespace, then explicit allow rules. Without this, a compromised pod in the `frontend` namespace can directly access the `database` namespace. We use Cilium which also gives us DNS-aware policies — we can say "this pod can only reach `api.stripe.com`, nothing else external."

**Detection layer** — Falco monitors syscalls in real-time. It alerted us when someone `kubectl exec`'d into a production pod and ran `curl` to download a binary. The exec was from a compromised CI service account. Falco caught it in 3 seconds. Without it, we wouldn't have known until the damage was done.

**Image layer** — Trivy scans every image in CI. Critical CVEs block the pipeline. We also run continuous scanning on running images — a CVE discovered after deployment still gets flagged.

**Real scenario:** We passed a SOC2 audit because we could demonstrate all five layers. The auditor specifically asked "what happens if someone deploys an untrusted image?" We showed the Kyverno deny log. "What if a container is compromised?" We showed Falco alerts. Defense in depth isn't just a concept — auditors check for it.

---

## 3. HPA vs VPA vs Karpenter — when would you NOT use each?

**Don't use HPA when:**
- App isn't horizontally scalable — databases, single-instance Redis, Kafka Connect workers with partition assignments. Adding replicas doesn't distribute load automatically.
- Startup time is 3-5 minutes (heavy JVM apps, ML model loading). By the time the new pod is ready, the traffic spike is over. We had a Java service that took 4 minutes to start. HPA kept adding pods during spikes, but they were never ready in time. We solved it by over-provisioning 2 extra replicas permanently — cheaper than wasted scaling.
- Already using VPA on CPU/memory — they fight each other. HPA says "add pods," VPA says "make pods bigger." Use both only if HPA scales on a custom metric (requests/sec) and VPA handles CPU/memory.

**Don't use VPA when:**
- App can scale horizontally — HPA is faster, no restarts, no downtime. VPA is the fallback, not the default.
- Can't tolerate pod restarts — VPA **restarts pods** to apply new limits. For a payments service with strict availability requirements, a random restart is unacceptable. We use VPA in `Off` mode — it recommends, we apply during maintenance windows.
- Running JVM with fixed heap (`-Xmx4g`) — VPA increases the memory limit, but the JVM ignores the extra memory. You're paying for resources the app will never use.

**Don't use Karpenter when:**
- Not on AWS — limited provider support. GCP → use GKE Autopilot. Azure → AKS with node auto-provisioning.
- Workload is stable and predictable — if you run the same 50 pods 24/7, Karpenter adds complexity for zero benefit. Fixed node groups are simpler.
- Compliance requires specific instance types — Karpenter dynamically picks instances. If your PCI audit demands "only m5.xlarge," you'll spend more time constraining Karpenter than benefiting from it.

**Bonus — simulating HPA in staging:** We use `k6` to generate realistic traffic patterns, watch with `kubectl get hpa -w`, and specifically test scale-down behavior. Default cooldown is 5 minutes — we once had an issue where HPA scaled up to 20 pods during load test, then took 15 minutes to scale down because the stabilization window kept resetting.

---

## 4. HPA is not scaling even though traffic increased. Which three things do you verify before touching code?

**1. Metrics Server** — HPA is completely blind without metrics. Run `kubectl top pods`. If it returns "metrics not available," metrics-server is either not installed or crashed. Check `kubectl get pods -n kube-system | grep metrics-server`. Real scenario: After a cluster upgrade, metrics-server's API certificate expired. `kubectl top` returned errors, HPA sat at 1 replica during a traffic spike, and we had a 10-minute outage.

**2. Resource requests** — This one catches most people. HPA calculates scaling as: `currentUsage / requestedResources × 100`. If your pod has no `resources.requests.cpu`, HPA has no denominator — it can't compute a percentage. Run `kubectl describe hpa <name>` and look at the targets column. If it shows `<unknown>/80%`, HPA can't read the metric. We had a service with no CPU requests defined — HPA showed "target: <unknown>" for weeks and nobody noticed until a traffic spike hit.

**3. Min/Max replicas and stabilization window** — Check `kubectl describe hpa <name>`. Is `maxReplicas` already reached? If you set `maxReplicas: 5` and you're already at 5, HPA won't scale further regardless of load. Also, HPA has a default scale-up stabilization window. If metrics fluctuate (spike → drop → spike), HPA might wait before acting. Check the `Events` section in describe output — it shows scaling decisions and reasons.

**Bonus:** If using custom metrics (requests/sec via Prometheus adapter), verify the API is responding: `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/`. If this returns 404, HPA can't read custom metrics at all — and it won't error out. It just sits there doing nothing. We had this when the Prometheus adapter pod restarted and lost its config.

---

## 5. How do you secure an AKS/Kubernetes cluster that's publicly accessible?

A publicly accessible cluster means the API server or workloads are reachable from the internet. You need to lock it down at multiple layers — network, authentication, authorization, workload, and runtime.

**1. API server access — restrict who can reach it**

On AKS, enable "Authorized IP Ranges" so only your office/VPN IPs can reach the API server:
```bash
az aks update -g myRG -n myCluster --api-server-authorized-ip-ranges "203.0.113.0/24,10.0.0.0/16"
```
On EKS, use a private endpoint + VPN. We disabled the public endpoint entirely — `kubectl` only works from within the VPC or through a bastion/VPN. This means even if someone steals a kubeconfig, they can't use it from their home network.

**2. Authentication — no static credentials**

- Integrate with Azure AD (AKS) or IAM/SSO (EKS) for user authentication. No static tokens, no shared kubeconfigs. Every user authenticates via SSO and gets time-limited credentials.
- Disable anonymous authentication: `--anonymous-auth=false`
- Disable legacy ABAC, use only RBAC

**3. RBAC — least privilege**

Every team gets a namespace with scoped RoleBindings. Developers get `view` role in prod (read-only). Only CI/CD service accounts get `edit` permissions, scoped to specific namespaces.

Real scenario: A junior developer had `cluster-admin` in production (inherited from a "quick setup" months ago). They accidentally ran `kubectl delete deployment --all -n production`. We recovered via ArgoCD (auto-synced back in 2 minutes), but after that, we audited every RoleBinding: `kubectl get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'`. Found 7 unnecessary cluster-admin bindings. Removed all except the break-glass account.

**4. Network layer**

- **NetworkPolicies** — Default deny-all in every namespace. Only allow explicitly needed pod-to-pod and egress traffic. Without this, a compromised pod can scan the entire cluster.
- **Private ingress** — Use internal LoadBalancers for internal services. Only public-facing services get an external LB, behind a WAF (Azure Front Door, AWS WAF).
- **Egress control** — Restrict which external domains pods can reach. We use Cilium DNS-aware policies: `allow egress to api.stripe.com, deny everything else`. This prevents a compromised container from calling a C2 server.

**5. Workload hardening**

- Pod Security Admission in `restricted` mode — non-root, read-only filesystem, no privilege escalation, drop all capabilities
- Image allowlisting — Kyverno policy: only images from our private ACR/ECR. Block Docker Hub, block unscanned images.
- Resource limits on every pod — prevents a single pod from consuming all node resources (DoS from inside)
- No `hostNetwork`, `hostPID`, `hostPath` — these break container isolation

**6. Secrets and data**

- Enable encryption at rest for etcd — K8s Secrets are encrypted, not just base64
- External Secrets Operator — secrets live in Azure Key Vault / AWS Secrets Manager, synced into the cluster
- Audit logging enabled — every `kubectl` command, every API call is logged. We forward audit logs to SIEM. When someone `kubectl exec`s into a prod pod, we get an alert.

**7. Runtime detection**

- Falco for syscall monitoring — alerts on shell spawns in containers, sensitive file access, unexpected network connections
- Container image scanning (Trivy) in CI + continuous scanning of running images

**Real scenario:** During a penetration test, the tester gained access to a pod via an application vulnerability (SSRF). They tried to reach the metadata API (`169.254.169.254`) to steal node IAM credentials. Our NetworkPolicy blocked egress to the metadata IP. They tried to `kubectl` from inside the pod — `automountServiceAccountToken: false` meant no token was available. They tried to scan other pods — NetworkPolicy blocked lateral movement. Five layers of defense, each one stopped a different attack vector. That's why defense in depth matters.

---

## 6. How do you debug NetworkPolicy issues step by step?

NetworkPolicies are stateless YAML — they either match or they don't. No error messages, no logs. Traffic just silently drops. That's what makes debugging them painful.

**Step 1: Confirm NetworkPolicies exist and apply to your pod**
```bash
# List all NetworkPolicies in the namespace
kubectl get networkpolicy -n <namespace>

# Check which policies select your pod (match labels)
kubectl describe networkpolicy -n <namespace>
# Look at PodSelector — does it match your pod's labels?
```

If no NetworkPolicy selects your pod, traffic is **allowed by default**. If any policy selects it, all non-matching traffic is **denied** (implicit deny).

**Step 2: Verify the pod's labels match the policy selector**
```bash
kubectl get pod <pod> --show-labels
# Compare with the podSelector in the NetworkPolicy
```

Common mistake: policy selects `app: frontend` but pod has `app: frontend-v2`. Labels must match exactly.

**Step 3: Test connectivity from inside the pod**
```bash
kubectl exec -it <source-pod> -- nc -zv <target-service> <port>
# Or use curl for HTTP
kubectl exec -it <source-pod> -- curl -v http://target-service:8080/health
```

**Step 4: Check if the CNI supports NetworkPolicies**
This catches a lot of people. **Not all CNIs enforce NetworkPolicies.** The default `kubenet` on AKS and the default AWS VPC CNI on EKS **do not** enforce NetworkPolicies unless you install a policy engine. You need Calico, Cilium, or Azure NPM.

```bash
# Check which CNI is running
kubectl get pods -n kube-system | grep -E 'calico|cilium|weave|azure-npm'
```

If nothing shows up, your NetworkPolicies are decorative — they exist in etcd but aren't enforced. This is the #1 reason NetworkPolicies "don't work."

**Step 5: Check egress rules if the issue is outbound**
People forget that NetworkPolicy controls both ingress AND egress. If you have a default-deny-all policy, your pod can't reach DNS (port 53) either. Without DNS, nothing works.

```yaml
# Must explicitly allow DNS egress
egress:
  - to:
      - namespaceSelector: {}
    ports:
      - protocol: UDP
        port: 53
```

**Real scenario:** We applied a default-deny NetworkPolicy to the `payments` namespace. All pods immediately lost connectivity — not just to external services, but to each other. We forgot to allow DNS egress. Every service lookup failed because pods couldn't reach CoreDNS on port 53. The fix was adding a DNS egress rule. Took us 20 minutes to figure out because the error was "connection timed out," not "DNS resolution failed" — the app retried DNS, timed out, and reported a generic connection error.

**Step 6: Use Cilium's policy verdict logs (if using Cilium)**
```bash
kubectl exec -n kube-system <cilium-pod> -- cilium monitor --type policy-verdict
```
This shows real-time allow/deny decisions with source, destination, and which policy matched. This is the closest thing to NetworkPolicy "logs."

---

## 7. How do you encrypt pod-to-pod traffic (mTLS) in Kubernetes?

By default, pod-to-pod traffic in Kubernetes is **unencrypted**. Any pod on the same network can sniff traffic. For compliance (PCI-DSS, HIPAA) or zero-trust architecture, you need mTLS.

**Option 1: Service Mesh (Istio/Linkerd) — most common**

Inject a sidecar proxy (Envoy) into every pod. The proxy handles TLS termination and certificate rotation automatically. Your application code doesn't change at all.

```bash
# Istio: enable strict mTLS for a namespace
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
EOF
```

With `STRICT` mode, any unencrypted traffic to pods in this namespace is rejected. Istio's Citadel handles certificate issuance and rotation (every 24 hours by default).

**Pros:** Zero code changes, automatic cert rotation, fine-grained policy (allow service A → B, deny A → C), observability (Kiali shows traffic flows).
**Cons:** Sidecar adds ~50MB memory per pod and 1-3ms latency. At 500 pods, that's 25GB of memory just for sidecars.

**Option 2: Cilium with WireGuard — no sidecars**

Cilium can encrypt all pod-to-pod traffic at the network layer using WireGuard. No sidecars, no code changes, no certificates to manage.

```bash
# Enable in Cilium Helm values
encryption:
  enabled: true
  type: wireguard
```

**Pros:** No sidecar overhead, lower latency than Istio, simpler to operate.
**Cons:** No application-layer visibility, no per-service policies (it's all-or-nothing encryption), no traffic management features (retries, circuit breaking).

**Option 3: Application-level TLS**

App manages its own TLS certificates. Most control but most work. Use cert-manager to issue certs, mount them as Secrets, configure each service to use them. Only viable if you have < 10 services.

**What I recommend:** Start with Cilium WireGuard if you just need encryption for compliance. Move to Istio if you also need traffic management, observability, and per-service authorization policies. Never roll your own TLS between services — the operational burden is not worth it.

**Real scenario:** A security audit flagged that our inter-service traffic was unencrypted. We initially planned Istio, but the sidecar memory overhead would've required 8 additional nodes. We went with Cilium WireGuard instead — enabled it cluster-wide in one Helm upgrade. Zero application changes, zero extra memory per pod, audit passed.

---

## 8. How would you simulate a DNS failure in CoreDNS to test application resilience?

This is a chaos engineering question. You want to verify that applications handle DNS failures gracefully — with retries, timeouts, and fallbacks — instead of crashing.

**Method 1: Scale CoreDNS to zero (simplest, most destructive)**
```bash
kubectl scale deployment coredns -n kube-system --replicas=0
```
All DNS resolution in the cluster stops. Every pod that tries to resolve a service name will fail. This tests the worst case. **Don't do this in production.**

**Method 2: Inject errors via CoreDNS Corefile**
Edit the CoreDNS ConfigMap to return errors for specific domains:
```bash
kubectl edit configmap coredns -n kube-system
```
Add an `erratic` plugin for targeted failure:
```
erratic {
    drop 50   # Drop 50% of DNS queries
}
```
Or block specific domains:
```
template IN A blocked-service.namespace.svc.cluster.local {
    rcode NXDOMAIN
}
```
Then restart CoreDNS: `kubectl rollout restart deployment coredns -n kube-system`

This is more surgical — you can simulate partial failures or failures for specific services only.

**Method 3: NetworkPolicy blocking DNS**
Apply a NetworkPolicy that blocks egress to CoreDNS (UDP port 53) for a specific pod:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-dns
  namespace: test
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 8080
    # DNS (port 53) is NOT listed, so it's blocked
```

This isolates the failure to one pod — safest for testing in shared environments.

**Method 4: Use Chaos Mesh or LitmusChaos**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: DNSChaos
metadata:
  name: dns-failure-test
spec:
  action: error
  mode: all
  selector:
    namespaces:
      - test
    labelSelectors:
      app: my-app
  duration: "60s"
```

This is the production-safe approach. It's time-bounded (auto-reverts after 60s), scoped to specific pods, and integrates with your chaos engineering pipeline.

**What to observe during the test:**
- Does the app crash or gracefully degrade?
- Are there retries with backoff?
- Do health checks fail (causing unnecessary restarts)?
- Does the app cache DNS results (it should — `ndots:5` causes 5 DNS lookups per external domain without caching)
- How long until recovery after DNS is restored?

**Real scenario:** We ran a DNS chaos test and discovered our Go services had no DNS caching. Every HTTP request triggered a DNS lookup. When CoreDNS was slow (not even down — just 200ms latency), request latencies spiked 10x. We added a local DNS cache (`dnsmasq` sidecar) and response times stabilized. Without chaos testing, we'd have found this during a real CoreDNS incident.

---

## 9. How do you make a Kubernetes cluster highly available?

HA means no single point of failure — any component can die and the cluster keeps running. You need HA at three levels: control plane, worker nodes, and application.

**1. Control Plane HA**

**API Server** — Run 3+ replicas behind a load balancer. Each instance is stateless — they all read/write to etcd. If one dies, the LB routes to the others. On managed K8s (EKS/AKS/GKE), this is done for you — the provider runs multi-AZ API servers.

**etcd** — Run 3 or 5 nodes (always odd for quorum). etcd uses Raft consensus — it needs a majority (2 of 3 or 3 of 5) to accept writes. Spread etcd nodes across availability zones. If one AZ goes down, the remaining nodes maintain quorum.

```
3 nodes → tolerates 1 failure
5 nodes → tolerates 2 failures
```

**Scheduler & Controller Manager** — Run multiple replicas with leader election. Only one is active at a time — the others are standby. If the leader dies, a new one is elected within seconds. `--leader-elect=true` is the default.

**2. Worker Node HA**

**Multi-AZ node groups** — Spread nodes across 3 availability zones. If AZ-a goes down, pods reschedule to AZ-b and AZ-c.

```bash
# EKS managed node group across AZs
aws eks create-nodegroup --subnets subnet-az-a subnet-az-b subnet-az-c
```

**Cluster Autoscaler / Karpenter** — Automatically add nodes when demand increases or when a node fails and pods need somewhere to go.

**Over-provision slightly** — Don't run nodes at 95% capacity. If a node dies, the remaining nodes need headroom to absorb the evicted pods. We keep 20-30% spare capacity.

**3. Application-Level HA**

**Multiple replicas with anti-affinity** — Don't put all replicas on the same node or AZ:
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: my-app
        topologyKey: topology.kubernetes.io/zone
```
This forces pods to spread across AZs. If one AZ dies, replicas in other AZs keep serving.

**PodDisruptionBudgets** — Prevent K8s from evicting too many pods at once during maintenance:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

**Readiness probes** — Only route traffic to healthy pods. Without probes, traffic hits pods that are starting up or failing.

**4. Storage HA**

Use network-attached storage (EBS, Azure Disk) with cross-AZ replication, not local storage. For databases, use managed services (RDS, Cloud SQL) or run StatefulSets with `podAntiAffinity` and application-level replication (e.g., PostgreSQL streaming replication).

**5. Networking HA**

**Ingress Controller** — Run 3+ replicas across AZs with a multi-AZ LoadBalancer in front. Single ingress controller = single point of failure for all external traffic.

**CoreDNS** — Run at least 2 replicas (default). If CoreDNS goes down, no pod can resolve service names — the entire cluster's internal communication breaks.

**Real scenario:** We ran a single-AZ EKS cluster to save costs. AWS had an AZ outage — our entire cluster went down for 47 minutes. After that, we moved to multi-AZ with nodes in 3 AZs, pod anti-affinity rules, and PDBs on every critical service. The next AZ outage (6 months later) — zero impact. Pods in the affected AZ rescheduled to healthy AZs within 2 minutes. The slight cost increase for multi-AZ was nothing compared to the 47-minute outage.

---

## 10. What is Helm chart signing? How does it work and which tools are used?

Helm chart signing lets you **cryptographically verify** that a chart was published by a trusted source and hasn't been tampered with since. Without signing, anyone who gains write access to your chart registry can push a malicious chart, and consumers would have no way to detect it.

**How it works — two artifacts per chart:**

When you sign a chart, Helm produces two files:
- `my-app-1.4.0.tgz` — the chart package
- `my-app-1.4.0.tgz.prov` — the provenance file (contains a cryptographic hash of the chart + a PGP signature)

The provenance file lets anyone verify:
1. The chart content matches the hash (integrity — not tampered with)
2. The signature was made by a known key (authenticity — from a trusted publisher)

**Tools used:**

| Tool | Role |
|---|---|
| **GPG (GnuPG)** | Key generation and PGP signing — the core signing mechanism |
| **Helm** | Built-in `helm package --sign` and `helm verify` commands |
| **Cosign (Sigstore)** | Modern alternative to GPG, keyless signing via OIDC — growing adoption |
| **OCI registries (ECR, GHCR)** | Store signed charts as OCI artifacts alongside signatures |
| **Notation** | CNCF signing tool for OCI artifacts, used with OCI-based Helm chart registries |

**Step-by-step: Signing with GPG + Helm**

**Step 1: Generate a GPG key pair**
```bash
gpg --gen-key
# Enter name, email, passphrase
# This creates a public/private key pair in your keyring
```

**Step 2: Export public key for distribution**
```bash
gpg --export -a "Your Name" > helm-signing.pub
# Publish this to a keyserver or share with your team
```

**Step 3: Package and sign the chart**
```bash
helm package my-app/ \
  --sign \
  --key "Your Name" \
  --keyring ~/.gnupg/secring.gpg \
  --passphrase-file ./passphrase.txt

# Produces:
# my-app-1.4.0.tgz
# my-app-1.4.0.tgz.prov
```

**Step 4: Push both files to chart registry**
```bash
helm push my-app-1.4.0.tgz oci://your-registry/charts
helm push my-app-1.4.0.tgz.prov oci://your-registry/charts
```

**Step 5: Verify before installing**
```bash
# Import the publisher's public key first
gpg --import helm-signing.pub

# Verify the chart on download
helm verify my-app-1.4.0.tgz

# Or verify during install directly
helm install my-app oci://your-registry/charts/my-app \
  --verify \
  --keyring ./trusted-keys.gpg
```

If verification fails — tampered chart, wrong key, or missing provenance file — Helm refuses to install and exits with an error.

**Modern approach: Cosign (keyless signing)**

GPG key management is operationally heavy — key rotation, distribution, revocation. Cosign with Sigstore offers **keyless signing** where identity is tied to OIDC (GitHub Actions identity, GCP service account, etc.) instead of a manually managed key:

```bash
# In CI (GitHub Actions) — signs using the workflow's OIDC identity
cosign sign --yes oci://your-registry/charts/my-app:1.4.0

# Verify — no key file needed, identity is verified against Sigstore's transparency log
cosign verify oci://your-registry/charts/my-app:1.4.0 \
  --certificate-identity "https://github.com/your-org/your-repo/.github/workflows/release.yaml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com"
```

The signature is recorded in Sigstore's **Rekor** transparency log — a public, immutable ledger. Anyone can audit who signed what and when.

**Enforcing signature verification in the cluster:**

Signing only helps if you enforce verification at deploy time. Use **Kyverno** or **Connaisseur** to block unsigned or unverified charts:

```yaml
# Kyverno policy — block pods using images not signed by our key
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  rules:
    - name: check-image-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "123456789.dkr.ecr.us-east-1.amazonaws.com/*"
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "https://github.com/your-org/*"
```

This means even if someone pushes a chart to your registry manually, pods using it will be blocked by Kyverno unless the image was signed by your CI pipeline.

**Real scenario:** We passed a SOC2 Type II audit that required us to demonstrate software supply chain controls. The auditor specifically asked: "How do you ensure no unauthorized code runs in production?" We demonstrated:
1. Every chart is signed by the CI pipeline using Cosign (keyless, tied to GitHub Actions identity)
2. Kyverno blocks any pod using an image not in our ECR or without a valid Cosign signature
3. Sigstore's Rekor log provides an immutable audit trail of every image ever signed

Before implementing this, a developer had manually pushed a patched image to ECR during an incident to bypass the CI pipeline. It worked — no controls caught it. After signing enforcement, that same action would be blocked at the pod admission level, even for images that exist in ECR.

**When to implement chart signing:**
- If you distribute Helm charts publicly or to external consumers — mandatory
- If you operate in regulated environments (PCI, HIPAA, SOC2) — mandatory
- For internal-only charts in a small team — start with Cosign in CI (low effort), skip GPG key management complexity

---

## 11. How do you define SLI, SLO, and SLA in production? How do you calculate error budget?

These three concepts form the backbone of SRE. They answer: "How reliable is the system, what have we committed to, and how much risk can we take?"

**Definitions:**

**SLI (Service Level Indicator)** — a *measurement* of service behavior. A specific metric that tells you how the service is performing right now.

**SLO (Service Level Objective)** — an *internal target* for an SLI. "We want this SLI to meet X% of the time." This is what your team commits to internally.

**SLA (Service Level Agreement)** — an *external contract* with customers. Breach it and there are financial or legal consequences (credits, refunds, penalties). SLA is always looser than SLO — you need a buffer.

```
SLI: measurement  →  SLO: internal target  →  SLA: customer contract
     (what it is)         (what we aim for)        (what we promise)
```

**Defining SLIs — choose what users actually care about:**

Not all metrics make good SLIs. CPU usage is not an SLI — users don't experience CPU. They experience latency, errors, and availability.

The **RED method** maps directly to SLIs:
- **Rate** → requests per second (volume, not an SLI by itself)
- **Errors** → error rate (% of requests returning 5xx)
- **Duration** → latency (p50, p95, p99 response time)

```yaml
# Good SLIs for an HTTP API
SLI-1: Availability     = successful requests / total requests × 100
SLI-2: Latency          = % of requests completed in < 200ms
SLI-3: Error rate       = 5xx responses / total responses × 100

# Bad SLIs (don't reflect user experience)
❌ CPU utilization
❌ Memory usage
❌ Pod restart count
```

**Defining SLOs — set realistic targets based on data:**

```yaml
# Example SLOs for an e-commerce checkout API
SLO-1: Availability     ≥ 99.9%    over a 30-day rolling window
SLO-2: Latency p99      ≤ 500ms    over a 30-day rolling window
SLO-3: Error rate       ≤ 0.1%     over a 30-day rolling window
```

99.9% availability means you can afford **43.8 minutes of downtime per month**. 99.99% means only **4.38 minutes**. Choose your SLO based on what users need and what you can realistically maintain — not aspirationally.

**Calculating Error Budget:**

Error budget = the amount of unreliability you're *allowed* to have before breaching the SLO.

```
Error Budget = 100% - SLO target

For SLO of 99.9% availability over 30 days:
  Total minutes in 30 days = 30 × 24 × 60 = 43,200 minutes
  Error budget = 0.1% × 43,200 = 43.2 minutes of allowed downtime
```

```promql
# Current error budget consumption in Prometheus
# How much of the 30-day budget has been burned?

# Error rate over 30 days
sum(rate(http_requests_total{status=~"5.."}[30d]))
/
sum(rate(http_requests_total[30d]))

# Budget remaining (as %)
1 - (
  sum(rate(http_requests_total{status=~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
  /
  0.001   # SLO error budget = 0.1%
)
```

**How error budget drives decisions:**

| Error Budget Remaining | Action |
|---|---|
| > 50% | Move fast — ship features, take risks, run chaos experiments |
| 25–50% | Normal pace — review risky changes before merging |
| < 25% | Slow down — freeze non-critical deployments, focus on reliability |
| 0% (budget exhausted) | Feature freeze — all effort on reliability until budget recovers |

This turns reliability into a data-driven conversation. Instead of "we need more stability" (vague), it becomes "we've burned 80% of our error budget in 10 days — no new feature deploys until we fix the root cause."

**Implementing SLOs in Prometheus + Grafana:**

```yaml
# Prometheus recording rules for SLO tracking
groups:
  - name: slo.rules
    rules:
      # 5-minute error rate
      - record: job:http_errors:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

      # 30-day availability
      - record: job:availability:30d
        expr: |
          1 - (
            sum(rate(http_requests_total{status=~"5.."}[30d])) by (job)
            /
            sum(rate(http_requests_total[30d])) by (job)
          )
```

```yaml
# Alert when burning error budget too fast (burn rate alert)
- alert: HighErrorBudgetBurn
  expr: |
    job:http_errors:rate5m > (14.4 * 0.001)
  for: 5m
  annotations:
    summary: "Burning error budget 14x faster than sustainable rate"
```

The `14.4x` multiplier means: at this rate, you'll exhaust the entire monthly budget in 2 days (30 days / 14.4 ≈ 2 days). This is a page-worthy alert.

**SLA — the external contract:**

SLA is always set below your SLO to create a buffer:

```
SLO: 99.9% availability (internal target)
SLA: 99.5% availability (customer commitment)

Buffer: 0.4% = ~173 minutes/month to absorb unexpected incidents
        before you owe customers credits
```

Most companies set SLAs at 99.5% or 99% even when their SLOs are 99.9%, specifically so that a bad incident doesn't immediately trigger SLA penalties while the team is still recovering.

**Real scenario:** Our payments team had no SLOs — just a vague "keep it up." During a 90-minute incident, the product team kept pushing new features mid-incident because they didn't know reliability was at risk. After defining SLOs (99.95% availability, p99 < 300ms), we set an error budget of 21.6 minutes/month. The first month, we burned 18 minutes in a single incident — 83% of our budget in one event. That data made the case for a feature freeze and a two-sprint reliability investment. No SLO → nobody knows the cost of incidents. SLO → the cost is visible, measurable, and actionable.

---

## 12. What happens to a Pod when its initContainer fails with restartPolicy: Never?

The Pod enters `Failed` phase permanently. No retries. No restarts. The main container never starts.

`restartPolicy: Never` applies to the **Pod**, not just the main container. When an initContainer fails under this policy, kubelet marks the Pod as `Failed` and stops all further action on it. The Pod object stays in the cluster in `Failed` state until you delete it.

```bash
kubectl get pod <pod-name>
# STATUS: Error  (not CrashLoopBackOff — no retries happen)

kubectl describe pod <pod-name>
# Init Containers:
#   init-db-check:
#     State: Terminated
#     Reason: Error
#     Exit Code: 1
# Events:
#   Warning  Failed  pod failed to start because init container failed
```

**Contrast with other restartPolicy values:**

| restartPolicy | initContainer fails | Behaviour |
|---|---|---|
| `Always` | Retries the initContainer with backoff (10s→20s→40s...) → CrashLoopBackOff if keeps failing |
| `OnFailure` | Same as Always for failed exit codes — retries with backoff |
| `Never` | Pod immediately enters `Failed` state. No retries. Ever. |

**Why this matters in practice:**

`restartPolicy: Never` is common on Jobs and batch Pods. If your init container checks a database migration status or waits for a dependency, and it fails — the Job Pod is done. The Job controller (not kubelet) decides whether to create a new Pod based on `backoffLimit`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  backoffLimit: 3       # Job controller creates up to 3 new Pods on failure
  template:
    spec:
      restartPolicy: Never   # Each Pod attempt: no retries, fail fast
      initContainers:
        - name: wait-for-db
          image: busybox
          command: ['sh', '-c', 'nc -z postgres 5432 || exit 1']
      containers:
        - name: migrate
          image: my-app:v1
          command: ['./run-migrations.sh']
```

If `wait-for-db` fails → Pod goes `Failed` → Job controller creates a new Pod (up to `backoffLimit: 3`).

**Real scenario:** A migration Job kept creating new Pods and failing. On-call engineer saw 4 Pods in `Error` state, assumed the app was broken. Root cause: the initContainer was checking `nc -z postgres 5432` — the database was up, but the port check used the wrong service name (`postgres` instead of `postgres-service`). With `restartPolicy: Never`, each attempt failed immediately and the Job exhausted its `backoffLimit` in under a minute. The fix was correcting the service name. Understanding that `Never` = zero retries per Pod helped us focus on the init container immediately instead of debugging the main app.

---

## 13. If you delete pod-0 from a StatefulSet, do the other pods get renamed?

**No.** The other pods keep their exact names. Only the deleted pod is recreated — with the same name it had before.

StatefulSet pod names follow the pattern `<statefulset-name>-<ordinal>`. Ordinals are fixed identities, not positions. Deleting `pod-0` does not shift `pod-1` to `pod-0`. Instead, the StatefulSet controller recreates a new `pod-0` from scratch — same name, same PVC, same DNS hostname.

```bash
# StatefulSet with 3 pods: kafka-0, kafka-1, kafka-2
kubectl delete pod kafka-0

kubectl get pods -w
# kafka-0   0/1   Terminating   0   2m
# kafka-0   0/1   Pending       0   0s    ← new pod-0 being created
# kafka-0   0/1   Init:0/1      0   1s
# kafka-0   1/1   Running       0   15s   ← same name, same PVC

# kafka-1 and kafka-2 are NEVER touched
```

**Why this design exists:**

StatefulSet pods have stable identity by design. Each pod has:
- A stable hostname: `kafka-0.kafka-headless.default.svc.cluster.local`
- A bound PVC: `data-kafka-0` (follows the pod, not the node)
- A fixed ordinal that other cluster members know about (e.g., Kafka broker ID = 0)

If renaming happened on delete, the Kafka cluster would lose broker-0's identity, the PVC would become orphaned, and other brokers would have no way to re-establish replication. The whole point of StatefulSet is that identity is permanent.

**What does change:**
- The Pod gets a new IP address (it's a new Pod object)
- It starts fresh (container restarts, no in-memory state)
- But it reconnects to the same PVC with all its data intact

**Common confusion:** People expect StatefulSets to behave like arrays where deleting index 0 shifts everything down. They don't. Think of StatefulSet pods as named servers, not numbered slots.

**Real scenario:** A junior engineer deleted `elasticsearch-0` to "force a fresh start" on a misbehaving pod. They expected `elasticsearch-1` to become the new primary. Instead, `elasticsearch-0` came back as itself — and since it had the same PVC with potentially corrupted index data, it was still misbehaving. The right approach was to identify the actual problem (corrupted shard), delete the PVC along with the pod (`kubectl delete pod elasticsearch-0 && kubectl delete pvc data-elasticsearch-0`), and let the StatefulSet create a fresh `elasticsearch-0` with a new empty PVC that would re-sync from the other nodes.

---

## 14. Does a DaemonSet automatically tolerate NoSchedule taints on master/control-plane nodes?

**Yes — but only for specific system taints, and only for DaemonSets created by Kubernetes itself or with the right tolerations.**

Kubernetes automatically adds tolerations to DaemonSet pods for these taints:

```yaml
# K8s automatically adds these tolerations to ALL DaemonSet pods:
tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
  - key: node.kubernetes.io/disk-pressure
    operator: Exists
    effect: NoSchedule
  - key: node.kubernetes.io/memory-pressure
    operator: Exists
    effect: NoSchedule
  - key: node.kubernetes.io/pid-pressure
    operator: Exists
    effect: NoSchedule
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```

**For control-plane nodes specifically:**

Control-plane nodes have this taint by default:
```
node-role.kubernetes.io/control-plane:NoSchedule
```

DaemonSets do **NOT** automatically tolerate this. If you want your DaemonSet to run on control-plane nodes, you must add the toleration explicitly:

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
```

**Why the distinction matters:**

System DaemonSets like `kube-proxy` and `aws-node` (VPC CNI) need to run on every node including control-plane — so they include this toleration explicitly. Your custom DaemonSets (Promtail, Datadog, Falco) will NOT run on control-plane nodes unless you add it.

**Real scenario:** We deployed Falco (security monitoring) as a DaemonSet to watch syscalls on every node. After deployment, `kubectl get pods -n falco -o wide` showed Falco running on all 8 worker nodes but not on the 3 control-plane nodes. An attacker who gained access to a control-plane node would be invisible to Falco. We added the control-plane toleration to the Falco DaemonSet. After that, Falco ran on all 11 nodes — worker and control-plane — giving us complete cluster coverage.

---

## 15. You push a new image tag mid-rollout. What does Kubernetes do — wait or start fresh?

**Kubernetes starts a fresh rollout immediately.** The in-progress rollout is abandoned and a new one begins from wherever the current state is.

Here's exactly what happens internally:

1. You have a Deployment rolling from `v1` → `v2`. Halfway through: 3 `v2` pods running, 3 `v1` pods still running.
2. You run `kubectl set image deployment/my-app app=my-app:v3`
3. The Deployment controller computes a new pod template hash for `v3`
4. A **new ReplicaSet** is created for `v3`
5. The `v2` ReplicaSet (partially scaled up) starts scaling **down**
6. The `v3` ReplicaSet starts scaling **up**
7. The `v1` ReplicaSet (if not yet fully scaled down) also continues scaling down

```bash
kubectl get rs -w
# NAME                    DESIRED   CURRENT   READY
# my-app-v1-hash          0         2         2     ← still scaling down
# my-app-v2-hash          0         3         3     ← abandoned, now scaling down
# my-app-v3-hash          6         4         2     ← new rollout taking over
```

**The old rollout is NOT waited for or completed — it is superseded.**

```bash
kubectl rollout history deployment/my-app
# REVISION  CHANGE-CAUSE
# 1         my-app:v1
# 2         my-app:v2   ← this revision exists but was never fully deployed
# 3         my-app:v3   ← current target
```

**What this means for rollback:**

`kubectl rollout undo` by default goes to revision 2 — the partially deployed `v2`. If `v2` was the broken image, rolling back to it is wrong. Always check `kubectl rollout history` before undoing and specify the revision explicitly:

```bash
kubectl rollout undo deployment/my-app --to-revision=1   # Go back to v1
```

**Real scenario:** During an incident, an engineer pushed a hotfix image (`v2.1-hotfix`) while the initial `v2.1` rollout was still in progress. The Deployment started a new rollout for the hotfix immediately. Three ReplicaSets existed simultaneously — the old `v2.0`, the incomplete `v2.1`, and the new `v2.1-hotfix`. During the chaos, `kubectl rollout undo` was run and accidentally landed on the incomplete `v2.1` (which had the bug) instead of `v2.0`. After this incident, we added a policy: mid-rollout image pushes are prohibited. All rollouts must complete or be fully rolled back before a new image is pushed.

---

## 16. Can two containers in the same Pod bind to the same port?

**No.** They share the same network namespace, which means they share the same IP address and the same port space. Two processes cannot bind to the same port on the same IP — this is a fundamental OS networking constraint, not a Kubernetes limitation.

```yaml
# This will FAIL at runtime — port 8080 conflict
spec:
  containers:
    - name: app
      image: my-app:v1
      ports:
        - containerPort: 8080   # app binds to :8080
    - name: sidecar
      image: my-sidecar:v1
      ports:
        - containerPort: 8080   # also tries to bind :8080 → FAILS
```

The second container to start will get `address already in use` and crash. The `ports` field in a pod spec is purely informational — Kubernetes does not enforce or validate it. The OS enforces it at bind time.

**What containers in the same Pod CAN do:**

- Communicate via `localhost` — container A calls `localhost:8080` to reach container B
- Share volumes mounted at the same path
- See each other's processes (if `shareProcessNamespace: true`)

```yaml
# Correct pattern — different ports
spec:
  containers:
    - name: app
      image: my-app:v1      # listens on :8080
    - name: envoy-proxy
      image: envoy:v1       # listens on :9901 (admin), :15001 (traffic)
```

**Real scenario:** A team added an Nginx sidecar to an existing pod for request buffering. Both the main app and Nginx were configured to listen on port 80. The main app started first, bound to `:80` successfully. Nginx started second, failed with `bind: address already in use`, and the pod entered CrashLoopBackOff. The fix: changed Nginx to listen on `:8080` and updated the Service `targetPort` accordingly. The `ports` declaration in the pod spec showed both containers listing port 80 with no warning from Kubernetes — the conflict only appeared at runtime.

---

## 17. HPA metrics server goes down during a traffic spike. What happens to scaling?

**HPA stops scaling — in either direction.** It does not scale up to handle the spike, and it does not scale down existing replicas. The current replica count is frozen until metrics become available again.

Specifically: when the metrics server is unavailable, HPA enters a state where it cannot compute the desired replica count. The HPA controller uses a **last-known-good** approach — it keeps the current replica count unchanged rather than making a scaling decision with no data.

```bash
kubectl describe hpa my-app-hpa
# Conditions:
#   AbleToScale     True    ReadyForNewScale
#   ScalingActive   False   FailedGetScale
#     message: the HPA was unable to compute the replica count:
#              unable to get metrics for resource cpu: unable to fetch metrics from resource metrics API
```

**What this means in practice:**

- If you had 5 replicas when metrics server went down → you stay at 5 replicas
- Traffic spikes → no new pods are added → existing 5 pods get overloaded
- Traffic drops → no pods are removed → you keep paying for 5 pods

**The dangerous scenario:**

If metrics server goes down *after* a scale-up event (say you're at 20 replicas during a spike), and then traffic drops, HPA cannot scale back down. You stay at 20 replicas — paying for 20 pods — until metrics are restored.

**How to protect against this:**

```yaml
# Set a reasonable default replica count — not 1
spec:
  minReplicas: 3    # Even with no metrics, you always have 3 pods
  maxReplicas: 30
```

```yaml
# Alert on metrics server unavailability
- alert: MetricsServerDown
  expr: absent(up{job="metrics-server"}) == 1
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Metrics server is down — HPA is blind"
```

**Also alert on HPA scaling failures:**
```promql
kube_horizontalpodautoscaler_status_condition{
  condition="ScalingActive",
  status="false"
} == 1
```

**Real scenario:** Our metrics-server pod was evicted during a node pressure event at peak traffic. HPA froze at 8 replicas. Traffic continued growing — p99 latency went from 120ms to 4 seconds. Nobody noticed metrics-server was down because we had no alert for it. We only found out when manually checking HPA status 20 minutes into the incident. After restoring metrics-server, HPA scaled to 22 replicas within 3 minutes and latency recovered. After the incident: added metrics-server to a dedicated node with `priorityClass: system-cluster-critical` so it never gets evicted, and added an alert for HPA scaling failures.

---

## 18. What actually happens to mounted tokens when you delete a ServiceAccount?

**Pods that already have the token mounted continue running — the token file in the pod is not deleted.** But the token becomes invalid immediately for new API calls, because the ServiceAccount no longer exists in the API server.

**Two token types — different behaviour:**

**Legacy auto-mounted tokens (pre-K8s 1.24):**
- Stored as a Secret of type `kubernetes.io/service-account-token`
- Mounted into the pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`
- When you delete the ServiceAccount, the Secret is garbage-collected
- The file in the running pod's filesystem remains (it was already written to the container's volume)
- But any API call using that token gets `401 Unauthorized` — the token references a deleted SA

**Bound service account tokens (K8s 1.24+, default now):**
- Generated per-pod with an expiry (default 1 hour)
- Not stored as a Secret — issued directly by the token API
- Deleting the ServiceAccount immediately invalidates all bound tokens for that SA
- Running pods lose API access on their next token use

```bash
# Check what token a pod is using
kubectl exec <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d. -f2 | base64 -d 2>/dev/null | jq .

# Test if the token still works
kubectl exec <pod> -- \
  curl -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://kubernetes.default.svc/api --insecure
# Returns 401 if SA is deleted
```

**What breaks when the SA is deleted:**

- Pod cannot list/watch/create K8s resources (if it was using the SA for API access)
- Operators and controllers that use SA tokens to manage resources fail
- Anything calling the K8s API from inside the pod gets 401

**What does NOT break:**

- The pod itself keeps running — kubelet does not kill it
- Non-K8s API calls (database, external APIs) are unaffected
- The pod's network and filesystem remain intact

**Real scenario:** A developer deleted a ServiceAccount named `monitoring-sa` while cleaning up "unused" resources. They checked — no pods were using it directly in their namespace. What they missed: Prometheus was running in the `monitoring` namespace using that exact SA to scrape metrics from pods across all namespaces via ClusterRoleBinding. Within 60 seconds, Prometheus started getting 401s on all cross-namespace scrape targets. All metrics went flat. Alerts stopped firing (Alertmanager pulls from Prometheus). We had no monitoring for 25 minutes until we recreated the SA and re-bound the ClusterRoleBinding. Added a policy after: ServiceAccounts referenced in ClusterRoleBindings cannot be deleted without peer review.

---

## 19. Can strict anti-affinity rules cause a permanent scheduling deadlock?

**Yes.** If every pod in a Deployment requires `requiredDuringSchedulingIgnoredDuringExecution` anti-affinity and there are not enough nodes to satisfy the constraint, new pods will never schedule — they stay `Pending` permanently.

**Classic deadlock scenario:**

```yaml
# 3 replicas, each must be on a different node — fine with 3+ nodes
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: my-app
        topologyKey: kubernetes.io/hostname   # different physical node
```

This works when you have 3+ nodes. But if a node goes down and you only have 2 nodes left:
- 2 existing pods are on node-A and node-B ✅
- The replacement pod for the failed node tries to schedule
- node-A: already has `app=my-app` → anti-affinity blocks it
- node-B: already has `app=my-app` → anti-affinity blocks it
- **Result: pod stuck Pending indefinitely**

```bash
kubectl describe pod <pending-pod>
# Events:
#   Warning  FailedScheduling  0/2 nodes available:
#   2 node(s) didn't match pod anti-affinity rules
```

**Another deadlock: circular dependency**

Pod A has anti-affinity against Pod B. Pod B has anti-affinity against Pod A. Both are `Pending` — neither can schedule because the other isn't running yet but the constraint checks existing pods.

Actually this specific case resolves (if no pods exist yet, the constraint is satisfied). The real deadlock is node-count-based.

**How to avoid:**

```yaml
# Use preferred instead of required — soft constraint, won't deadlock
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname
```

Or combine required with a higher `minReplicas` buffer:
```yaml
# If you have 5 nodes, keep maxReplicas ≤ 5 for required anti-affinity
spec:
  replicas: 3
  # maxReplicas in HPA: 5 — never exceed node count
```

**Real scenario:** We ran Elasticsearch with `required` anti-affinity across 3 nodes. During a node upgrade, we cordoned node-3 (making it unschedulable) and drained it. The ES pod from node-3 tried to reschedule — but node-1 and node-2 already had ES pods. It sat `Pending` for 40 minutes blocking the entire cluster upgrade process. Fix: uncordon node-3, let the ES pod come back, finish the upgrade on node-3 last. Long-term fix: switched to `preferred` anti-affinity and added a fourth node so the constraint would always be satisfiable.

---

## 20. With ReadWriteOnce access mode, can multiple Pods on the same node use the same PVC simultaneously?

**Yes — multiple pods on the same node can mount a ReadWriteOnce PVC simultaneously.** The "Once" in ReadWriteOnce means once per node, not once per pod.

`ReadWriteOnce` (RWO) means: the volume can be mounted as read-write by a **single node**. Multiple pods on that same node can all mount it concurrently.

```
Node-A:
  pod-1  ┐
  pod-2  ├── all mount pvc-data (RWO) → ✅ works
  pod-3  ┘

Node-B:
  pod-4 → tries to mount pvc-data (RWO) → ❌ blocked (already bound to Node-A)
```

**Access modes summary:**

| Mode | Abbreviation | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Read-write by **one node** (multiple pods on same node: OK) |
| `ReadOnlyMany` | ROX | Read-only by **many nodes** simultaneously |
| `ReadWriteMany` | RWX | Read-write by **many nodes** simultaneously (NFS, EFS) |
| `ReadWriteOncePod` | RWOP | Read-write by **one pod only** (K8s 1.22+) |

**If you want truly single-pod access — use `ReadWriteOncePod`:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: exclusive-storage
spec:
  accessModes:
    - ReadWriteOncePod    # Only one pod in the entire cluster can mount this
  resources:
    requests:
      storage: 10Gi
```

With `ReadWriteOncePod`, if a second pod tries to mount the PVC while the first pod is running, the second pod stays `Pending` with a clear error.

**Real scenario:** A team ran a StatefulSet with `ReadWriteOnce` PVCs, assuming each pod exclusively owned its volume. During a rolling update, the new pod was scheduled on the same node as the old (terminating) pod. For a brief window, both the old and new pod had the same PVC mounted. The old pod was still writing while the new pod started reading — causing data corruption in an append-only log file. The fix: switched to `ReadWriteOncePod` so the new pod stays `Pending` until the old pod fully terminates and releases the volume.

---

## 21. A NetworkPolicy only has ingress rules defined. Is egress still open?

**Yes — egress is completely open** if no egress rules are defined in any NetworkPolicy that selects the pod.

NetworkPolicy uses an **explicit opt-in model**. A NetworkPolicy only restricts the traffic types it explicitly declares. If a policy only specifies `policyTypes: [Ingress]`, egress is untouched. If no NetworkPolicy at all selects the pod, both ingress and egress are fully open.

```yaml
# This policy ONLY restricts ingress — egress is completely open
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress       # Only ingress is controlled
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
```

**The pod selected by this policy:**
- Ingress: only `app=frontend` pods can reach it ✅ (restricted)
- Egress: can connect to anything, anywhere ✅ (unrestricted)

**How default-deny works for both directions:**

```yaml
# Default deny ALL ingress AND egress in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}       # Selects ALL pods in the namespace
  policyTypes:
    - Ingress
    - Egress
  # No ingress or egress rules = deny everything
```

**The trap — forgetting to include `policyTypes`:**

```yaml
# BAD — policyTypes not specified
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
    - from: [...]
# Result: only ingress is restricted, egress is open
# This is the DEFAULT behaviour when policyTypes is omitted and only ingress rules exist
```

**Real scenario:** We applied a NetworkPolicy to restrict ingress to our payment service — only the API gateway could reach it. Security audit passed. Three months later, a penetration tester pointed out that any compromised pod in the namespace could make outbound calls to external IPs (C2 servers, data exfiltration endpoints) — because we never restricted egress. Our NetworkPolicy had locked the front door but left the back door wide open. Fix: added explicit egress rules allowing only necessary outbound traffic (DNS on 53, PostgreSQL on 5432, Stripe API) and a default-deny-egress policy for the namespace.

---

## 22. etcd performance is degrading. What metric tells you the real root cause?

The single most important metric is **`etcd_disk_wal_fsync_duration_seconds`** — the time it takes etcd to fsync its write-ahead log to disk.

etcd is a consensus database. Every write (pod creation, configmap update, secret change) must be committed to disk via fsync before etcd acknowledges it. If fsync is slow, everything is slow — the API server queues up, kubectl commands hang, controllers can't reconcile.

```promql
# P99 WAL fsync duration — should be < 10ms
histogram_quantile(0.99,
  rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])
)

# P99 backend commit duration (BoltDB writes) — should be < 25ms
histogram_quantile(0.99,
  rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])
)
```

| Metric value | Meaning |
|---|---|
| WAL fsync p99 < 10ms | Healthy — disk is fast enough |
| WAL fsync p99 10–50ms | Warning — disk under pressure, API latency increasing |
| WAL fsync p99 > 100ms | Critical — API server will time out, cluster feels unresponsive |

**Other key etcd metrics to check:**

```promql
# Leader elections — should be 0 in a stable cluster
etcd_server_leader_changes_seen_total

# Database size — default quota is 8GB, approaching it causes "mvcc: database space exceeded"
etcd_mvcc_db_total_size_in_bytes

# Active clients — sudden spike means controllers are reconnecting (instability signal)
etcd_server_client_requests_total

# Proposal failures — should be near 0
etcd_server_proposals_failed_total
```

**Root cause diagnosis by metric:**

| Symptom | Metric | Root cause |
|---|---|---|
| Slow API calls | High WAL fsync duration | Disk IOPS too low (gp2 vs gp3) |
| Cluster instability | Leader changes > 0 | Network latency between etcd nodes or disk pressure causing missed heartbeats |
| "database space exceeded" errors | DB size near 8GB | Too many revisions, need compaction |
| Controllers stuck | High proposal failures | etcd quorum lost (node down) |

**The most common fix — disk upgrade:**

```bash
# On AWS: migrate etcd EBS volume from gp2 to gp3 with higher IOPS
aws ec2 modify-volume \
  --volume-id vol-xxx \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125
```

etcd docs recommend dedicated SSDs with at least 2000 IOPS. A gp2 volume on a small disk (default 100 IOPS) will cause API server degradation under any meaningful cluster activity.

**etcd compaction — fix database size:**
```bash
# Compact old revisions (keeps only the last 1000)
etcdctl compact $(etcdctl endpoint status --write-out="json" | jq '.[0].Status.header.revision')

# Defragment to reclaim space
etcdctl defrag --endpoints=https://127.0.0.1:2379
```

**Real scenario:** Our cluster API latency jumped from 50ms to 8 seconds overnight. `kubectl` commands took 10-15 seconds. `kube_apiserver_request_duration_seconds` was high but that's a symptom, not a cause. Checking `etcd_disk_wal_fsync_duration_seconds` showed p99 at 320ms — 30x above the healthy threshold. The etcd nodes were on gp2 EBS volumes that had grown to 200GB (baseline IOPS scales with size on gp2 but only to a point). We migrated to gp3 with 3000 provisioned IOPS. WAL fsync dropped to 4ms within minutes. API latency returned to 50ms. The WAL fsync metric pointed directly at the root cause — without it we would have kept tuning API server flags and getting nowhere.

---

## 23. How do you enforce that all images come only from your internal registry?

Use **Kyverno** (or OPA/Gatekeeper) to validate every Pod at admission time. Any pod using an image not from your approved registry is rejected before it even reaches the scheduler.

**Kyverno policy — block non-approved registries:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce    # Reject — not just warn
  background: true                    # Also audit existing resources
  rules:
    - name: validate-registries
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Images must come from 123456789.dkr.ecr.us-east-1.amazonaws.com only."
        pattern:
          spec:
            containers:
              - image: "123456789.dkr.ecr.us-east-1.amazonaws.com/*"
            initContainers:
              - image: "123456789.dkr.ecr.us-east-1.amazonaws.com/*"
            ephemeralContainers:
              - image: "123456789.dkr.ecr.us-east-1.amazonaws.com/*"
```

**Also block `latest` tag:**

```yaml
    - name: disallow-latest-tag
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "The 'latest' image tag is not allowed. Use a specific version tag."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

**Test the policy before enforcing:**

```bash
# Start in Audit mode — logs violations but doesn't block
validationFailureAction: Audit

# Check what would have been blocked
kubectl get policyreport -A

# Once violations are resolved, switch to Enforce
validationFailureAction: Enforce
```

**Add image signing verification (next layer):**

Even if the image is from your registry, it might have been pushed manually (bypassing CI). Combine registry restriction with Cosign signature verification:

```yaml
    - name: verify-image-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "123456789.dkr.ecr.us-east-1.amazonaws.com/*"
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "https://github.com/your-org/*"
```

This enforces: image must be from our ECR AND signed by our GitHub Actions pipeline. Manual pushes — even to our own registry — are blocked.

**Verify the policy works:**

```bash
# Try to deploy a Docker Hub image — should be rejected
kubectl run test --image=nginx:latest
# Error from server: admission webhook "kyverno-resource-validating-webhook.kyverno.svc" denied:
# Images must come from 123456789.dkr.ecr.us-east-1.amazonaws.com only.
```

**Real scenario:** A developer was debugging a production issue at midnight and ran `kubectl run debug --image=ubuntu:latest` in the prod namespace to get a shell. The container started successfully (we had no policy). From inside `ubuntu`, they installed tools and made network calls to diagnose the issue — but ubuntu has known vulnerabilities and was never scanned. After implementing this Kyverno policy, the same command returns an immediate rejection. Debugging is now done using a hardened debug image in our ECR: `123456789.dkr.ecr.us-east-1.amazonaws.com/debug-tools:latest` — scanned, minimal, and logged.

---

## 24. How do you enforce tenant isolation in a multi-tenant Kubernetes setup?

Multi-tenant K8s means multiple teams or customers share one cluster. Isolation has four layers: **namespace boundary**, **RBAC**, **network**, and **resource quotas**. Without all four, one tenant can affect another.

**Layer 1: Namespace per tenant**

Each tenant gets their own namespace. This is the logical boundary everything else attaches to.

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
```

**Layer 2: RBAC — scoped to namespace**

Each tenant's service account and users can only operate within their namespace. No cluster-wide permissions.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: tenant-a
  name: tenant-a-developer
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services", "configmaps"]
    verbs: ["get", "list", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: tenant-a
  name: tenant-a-developer-binding
subjects:
  - kind: User
    name: alice@company.com
roleRef:
  kind: Role
  name: tenant-a-developer
  apiGroup: rbac.authorization.k8s.io
```

**Layer 3: NetworkPolicy — block cross-tenant traffic**

By default pods in any namespace can talk to pods in any other namespace. Lock this down:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-tenant
  namespace: tenant-a
spec:
  podSelector: {}      # applies to all pods in tenant-a
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: tenant-a   # only allow from same namespace
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: tenant-a
    - to: {}           # allow egress to external (internet, databases)
      ports:
        - port: 443
        - port: 53
          protocol: UDP
```

**Layer 4: ResourceQuota + LimitRange — prevent noisy neighbour**

One tenant doing a memory leak or running batch jobs shouldn't starve others.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-a-limits
  namespace: tenant-a
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 8Gi
```

**Layer 5: Pod Security Admission — prevent privilege escalation**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

This blocks: privileged containers, hostNetwork, hostPID, running as root, capability escalation.

**Layer 6: Dedicated Node Groups (hard isolation)**

For regulated tenants (finance, healthcare), use node taints to pin workloads to dedicated nodes:

```yaml
# Taint dedicated nodes
kubectl taint nodes node-group-tenant-a dedicated=tenant-a:NoSchedule

# Tenant-a pods must tolerate this
tolerations:
  - key: dedicated
    operator: Equal
    value: tenant-a
    effect: NoSchedule
nodeSelector:
  tenant: tenant-a
```

**Comparison: Soft vs Hard isolation**

| Approach | Use case | Blast radius |
|---|---|---|
| Namespace + RBAC + NetworkPolicy | Internal teams, cost sharing | Low |
| + ResourceQuota + LimitRange | Multi-product on shared cluster | Medium |
| + Dedicated node groups | External customers, compliance | Minimal |
| Separate clusters | PCI-DSS, HIPAA, zero-trust | Zero cross-tenant |

**Real scenario:** We ran three product teams on one EKS cluster with just namespaces and RBAC. One team ran a batch ML training job that consumed all available CPU on shared nodes — the payment service pods were throttled and latency went from 120ms to 4.2s. After adding ResourceQuota (capping each namespace at 10 CPU requests) and dedicated node groups for the payment service (`taint: team=payments:NoSchedule`), batch jobs could never touch payment nodes. Payment latency stayed flat even during peak batch processing.

**When to use hard node isolation:** any workload touching PII, financial data, or with SLA commitments where another team's noisy workload is an unacceptable risk.

---

## 25. How do you enforce that only validated Kubernetes configs reach production in a CI/CD pipeline?

> **Also asked as:** "How do you prevent bad configs from reaching production in a CI/CD pipeline?"

Bad configs in production — wrong resource limits, missing labels, broken Ingress rules — cause outages that are entirely preventable. I enforce validation at three gates: **static analysis in CI**, **admission control in the cluster**, and **diff review before apply**.

**Gate 1: Static validation in CI (before merge)**

```yaml
# GitHub Actions step — runs on every PR
- name: Lint with kubeval / kubeconform
  run: |
    kubeconform -strict -kubernetes-version 1.29.0 ./manifests/

- name: Lint Helm chart
  run: |
    helm lint ./charts/my-app --strict

- name: Dry-run against real cluster (staging)
  run: |
    kubectl apply --dry-run=server -f ./manifests/
    # Server-side dry-run catches: missing CRDs, quota violations,
    # admission webhook rejections — things static tools miss
```

`kubeconform` catches: wrong apiVersion, missing required fields, type mismatches.
`helm lint --strict` catches: missing values, invalid templates, undefined references.
`--dry-run=server` catches: quota exceeded, policy violations, missing secrets.

**Gate 2: OPA/Kyverno admission control in cluster**

Kyverno policy that blocks deployment without resource limits:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-resources
      match:
        any:
          - resources:
              kinds: ["Deployment", "StatefulSet", "DaemonSet"]
      validate:
        message: "CPU and memory limits are required on all containers."
        pattern:
          spec:
            template:
              spec:
                containers:
                  - resources:
                      limits:
                        cpu: "?*"
                        memory: "?*"
```

Every `kubectl apply` or Helm deploy must pass this — even from automated pipelines.

**Gate 3: GitOps diff review (ArgoCD)**

With ArgoCD, no one applies directly to production. Changes go through Git:

```
PR opened → CI runs kubeconform + helm lint + dry-run
         → PR review (human)
         → Merge to main
         → ArgoCD detects diff → shows sync preview
         → Manual sync approval for production
```

ArgoCD's sync preview shows exactly what will change before apply — like `terraform plan` but for K8s.

**Common bad configs caught by each gate:**

| Bad config | Caught by |
|---|---|
| Wrong `apiVersion` | kubeconform |
| Missing `limits` | Kyverno ClusterPolicy |
| Image from unverified registry | Kyverno + Cosign |
| ResourceQuota exceeded | `--dry-run=server` |
| Broken Ingress path | `helm lint` + dry-run |
| Drift from Git | ArgoCD |

**Real scenario:** A developer updated a Helm values file and accidentally set `resources.limits.memory: 128` (integer, not string). Helm rendered it as an invalid manifest. Without CI validation, this deployed fine on staging (which had no LimitRange enforcement) but failed on production during the next deployment. After adding `kubeconform` to the PR pipeline, this class of error gets caught in under 30 seconds — before code review even begins.

---
