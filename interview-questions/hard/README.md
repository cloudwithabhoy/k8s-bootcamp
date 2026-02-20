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
