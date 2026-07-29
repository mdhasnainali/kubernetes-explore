# Kubernetes — Path to Production Readiness

A task list that continues from where this repo already is (lessons 1–2: NodePort, Ingress)
and ends with 5–7 portfolio projects you could defend in a production discussion.

## How to use this list

- Work **top to bottom**. Phases build on each other; skipping storage before StatefulSets, or RBAC before GitOps, leaves holes.
- Each task becomes a **new numbered lesson directory** matching the existing convention:
  `3-probes-and-resources/`, `4-storage-with-pvc/`, … each with its own `README.md` study notes (architecture diagram, files table, apply order, inspect commands, gotchas).
- For every task, do the loop: **write manifest → apply → break it on purpose → read the error → fix → write the gotcha down.** The gotchas are the actual learning; the YAML is not.
- Mark done only when you can explain the concept without notes *and* debug it when it breaks.
- Reference: `kubectl explain <kind>.spec --recursive` beats searching the web for field names.

Legend: 🟢 local Minikube is enough · 🔵 needs a real/multi-node cluster (kind multi-node, k3d, or cloud) · ⏱ rough time

---

## Phase 0 — Close the gaps in what you already built

Before new topics, harden lessons 1–2. Everything here is a real production defect in the current manifests.

- [ ] **0.1 — Add liveness + readiness probes to both Deployments** 🟢 ⏱1h
  - Why: without a readiness probe, a Service sends traffic to a pod that is not ready yet → 503s during rollout. Without liveness, a hung process never restarts.
  - Do: `readinessProbe` (httpGet on the webapp, `exec` mongosh ping on Mongo), `livenessProbe`, and a `startupProbe` on Mongo.
  - Verify: `kubectl get pod -w` during a rollout — new pod must be `Ready 1/1` before the old one terminates.
  - Gotcha: liveness probe with too short `initialDelaySeconds`/`failureThreshold` = restart loop on slow start. That is what `startupProbe` exists for.

- [ ] **0.2 — Add resource `requests` and `limits`** 🟢 ⏱1h
  - Why: no requests = scheduler is blind and the pod lands in QoS class `BestEffort`, first to be evicted under node pressure.
  - Do: set CPU/memory requests+limits on every container. Then read `kubectl describe pod | grep QoS`.
  - Learn: QoS classes `Guaranteed` (requests == limits) / `Burstable` / `BestEffort`, and eviction order.
  - Gotcha: **CPU limit throttles, memory limit kills** (OOMKilled, exit code 137). Never set a memory limit you have not measured.

- [ ] **0.3 — Stop committing plaintext-ish Secrets; understand why base64 is not encryption** 🟢 ⏱30m
  - Do: `kubectl get secret mongo-secret -o jsonpath='{.data.mongo-user}' | base64 -d` — prove to yourself it is reversible.
  - Note in the README: encryption at rest is a **cluster/etcd** setting (`EncryptionConfiguration`), not a Secret field. Real fix comes in Phase 6.

- [ ] **0.4 — Pin images by digest, not tag** 🟢 ⏱30m
  - Why: `mongo:7.0` is mutable. `mongo@sha256:…` is not. Reproducible deploys need immutability.
  - Do: `docker buildx imagetools inspect mongo:7.0` (or `crane digest`), record digest, set `imagePullPolicy: IfNotPresent`.

- [ ] **0.5 — Move everything out of `default` namespace** 🟢 ⏱30m
  - Do: create a namespace manifest, add `namespace:` to metadata, apply, and set context: `kubectl config set-context --current --namespace=<ns>`.
  - Learn: which objects are namespaced vs cluster-scoped — `kubectl api-resources --namespaced=true|false`.

**Phase 0 exit test:** delete a pod mid-request-load and see zero failed requests.

---

## Phase 1 — Workload primitives beyond Deployment

- [ ] **1.1 — Pod lifecycle deep dive** 🟢 ⏱2h
  - Phases (`Pending`/`Running`/`Succeeded`/`Failed`), container states (`Waiting`/`Running`/`Terminated`), restart backoff.
  - Debug drill: intentionally cause each of `ImagePullBackOff`, `CrashLoopBackOff`, `CreateContainerConfigError`, `Pending/Unschedulable`, `OOMKilled`. Write the diagnosing command for each.
  - Tools: `kubectl describe pod`, `kubectl get events --sort-by=.lastTimestamp`, `kubectl logs --previous`, `kubectl debug -it <pod> --image=busybox --target=<container>` (ephemeral containers).

- [ ] **1.2 — Graceful shutdown** 🟢 ⏱1h
  - `terminationGracePeriodSeconds`, `preStop` hook, SIGTERM handling, and why a `preStop: sleep 5` fixes the endpoint-propagation race during rollout.
  - Verify: run `hey`/`ab` against the app while rolling; count non-200s before and after adding the hook.

- [ ] **1.3 — Init containers and sidecars** 🟢 ⏱2h
  - Init container: wait-for-Mongo before webapp starts (replaces app-side retry hacks).
  - Native sidecar: an `initContainers` entry with `restartPolicy: Always` (beta and on by default since v1.29) — starts before app containers, keeps running, and terminates *after* them. This is the correct modern pattern; a plain extra container in `containers[]` is the old workaround.
  - Add a log-shipper or config-reloader sidecar sharing an `emptyDir`.

- [ ] **1.4 — Jobs and CronJobs** 🟢 ⏱2h
  - Job: a one-shot Mongo seed/migration. Fields that matter: `backoffLimit`, `completions`, `parallelism`, `activeDeadlineSeconds`, `ttlSecondsAfterFinished`.
  - CronJob: nightly `mongodump` to a volume. Fields: `schedule`, `concurrencyPolicy`, `startingDeadlineSeconds`, `successfulJobsHistoryLimit`.
  - Gotcha: CronJob without `concurrencyPolicy: Forbid` can stack overlapping runs; without `ttlSecondsAfterFinished` you accumulate thousands of completed pods.

- [ ] **1.5 — DaemonSet** 🔵 ⏱1h
  - Run a node-level agent (log collector / node-exporter) on every node. Needs a multi-node cluster to be meaningful: `minikube start --nodes=3` or `kind create cluster --config multi-node.yaml`.
  - Learn: why DaemonSets need `tolerations` to land on control-plane nodes.

- [ ] **1.6 — StatefulSet (do after Phase 2 storage)** 🔵 ⏱4h
  - Convert Mongo from Deployment to a 3-replica StatefulSet with a replica set.
  - Learn: stable network identity (`mongo-0.mongo-headless`), ordinal ordering, headless Service requirement, `volumeClaimTemplates`, `podManagementPolicy`, and that **PVCs are not deleted** when the StatefulSet is.
  - Gotcha: scaling down a StatefulSet leaves orphan PVCs on purpose. Know how to reclaim them.

**Phase 1 exit test:** given a broken pod you did not create, find the root cause in under 3 commands.

---

## Phase 2 — Storage and state

- [ ] **2.1 — Volume types** 🟢 ⏱1h — `emptyDir` (and `emptyDir.medium: Memory`), `hostPath` (and why it is a security problem), `configMap`/`secret`/`projected`/`downwardAPI` as volumes.
- [ ] **2.2 — PV / PVC / StorageClass** 🟢 ⏱3h
  - Static provisioning (hand-written PV) *then* dynamic (StorageClass). Give Mongo a real PVC so data survives pod deletion.
  - Learn: access modes (`RWO`/`ROX`/`RWX`/`RWOP`), `reclaimPolicy` `Delete` vs `Retain`, `volumeBindingMode: WaitForFirstConsumer` and why it prevents cross-zone scheduling failures.
  - Verify: `kubectl delete pod mongo-…`, pod returns, data still there.
  - Gotcha: `RWO` = one **node**, not one pod. A Deployment with `replicas: 2` on an RWO volume will hang the second pod in `Pending` on a multi-node cluster.
- [ ] **2.3 — Resize a PVC** 🟢 ⏱1h — `allowVolumeExpansion: true`, edit the PVC, watch the `FileSystemResizePending` condition.
- [ ] **2.4 — Backup and restore drill** 🟢 ⏱2h — CronJob `mongodump` → PVC; then **actually restore into a fresh cluster**. A backup you have not restored is not a backup.

---

## Phase 3 — Networking for real

- [ ] **3.1 — Cluster DNS** 🟢 ⏱1h — `svc.<ns>.svc.cluster.local` resolution rules, `kubectl run -it --rm dns --image=busybox:1.36 -- nslookup mongo-service`, headless Services returning pod IPs, `dnsPolicy`, `hostNetwork` side effects.
- [ ] **3.2 — Service internals** 🟢 ⏱2h — Endpoints/EndpointSlice, selector→label matching, `targetPort` vs `port` vs `nodePort`, `sessionAffinity`, `externalTrafficPolicy: Local|Cluster`, ExternalName Services.
  - Drill: break a Service by typoing a selector label; diagnose with `kubectl get endpoints`.
- [ ] **3.3 — Ingress, level 2** 🟢 ⏱3h — path-based + host-based routing to two apps, `rewrite-target`, `nginx.ingress.kubernetes.io/*` annotations, `IngressClass`, default backend, `Prefix` vs `Exact` vs `ImplementationSpecific`.
- [ ] **3.4 — TLS** 🟢 ⏱3h — `spec.tls` with a `kubernetes.io/tls` Secret; self-signed first, then **cert-manager** with a `ClusterIssuer` + `Certificate` (staging Let's Encrypt ACME). This is the production pattern.
- [ ] **3.5 — NetworkPolicy** 🔵 ⏱3h
  - Default-deny ingress in the namespace, then allow only webapp→mongo on 27017 and ingress-controller→webapp.
  - **Gotcha:** Minikube's default CNI ignores NetworkPolicy — the policy applies and enforces nothing. Use `minikube start --cni=calico` or kind+Calico, then verify enforcement by curling from a pod that should be blocked.
- [ ] **3.6 — Gateway API** 🟢 ⏱3h — the successor to Ingress: `GatewayClass`/`Gateway`/`HTTPRoute`, role separation between platform and app teams, header/weight-based routing that Ingress annotations hacked around. Reimplement lesson 2 with it.

---

## Phase 4 — Scheduling, availability, autoscaling

- [ ] **4.1 — Scheduling controls** 🔵 ⏱3h — `nodeSelector`, node/pod affinity + anti-affinity (`requiredDuringScheduling…` vs `preferred…`), taints and tolerations, `topologySpreadConstraints`.
  - Drill: force the 3 webapp replicas onto 3 different nodes with `topologySpreadConstraints` (`maxSkew: 1`, `whenUnsatisfiable: DoNotSchedule`).
- [ ] **4.2 — Rollout strategies** 🟢 ⏱2h — `RollingUpdate` `maxSurge`/`maxUnavailable`, `minReadySeconds`, `progressDeadlineSeconds`, `revisionHistoryLimit`, `kubectl rollout status|history|undo|pause|resume`, and `Recreate`.
- [ ] **4.3 — PodDisruptionBudget + PriorityClass** 🔵 ⏱2h — set a PDB, then `kubectl drain <node>` and watch it block. Learn voluntary vs involuntary disruption, and preemption via PriorityClass.
- [ ] **4.4 — HPA** 🟢 ⏱3h — `minikube addons enable metrics-server`, then `autoscaling/v2` HPA on CPU; load-test to trigger scale-up; tune `behavior.scaleDown.stabilizationWindowSeconds` to stop flapping.
  - Gotcha: HPA on CPU **requires resource `requests`** — no requests, no target percentage, no scaling.
- [ ] **4.5 — Scale on custom metrics** 🟢 ⏱4h — Prometheus Adapter or **KEDA** scaling on queue depth / RPS. This is what most real workloads need instead of CPU.
- [ ] **4.6 — VPA and in-place resize** 🔵 ⏱2h — VPA recommendation mode (`updateMode: "Off"`) to right-size requests; and the newer **in-place pod resize** (`resize` subresource + `resizePolicy`) that changes CPU/memory without a restart. Know which one your cluster version supports.
- [ ] **4.7 — Cluster autoscaler concepts** 🔵 ⏱1h — node groups, scale-from-zero, why unschedulable pods (not CPU load) drive it. Read-only unless you have a cloud cluster.

---

## Phase 5 — Security (do not skip; this is the most common interview filter)

- [ ] **5.1 — ServiceAccounts and RBAC** 🟢 ⏱4h
  - Make a dedicated SA per workload, drop `automountServiceAccountToken` where unused, write `Role`/`RoleBinding` and `ClusterRole`/`ClusterRoleBinding`.
  - Drill: `kubectl auth can-i --as=system:serviceaccount:<ns>:<sa> get pods` — must return `no` for everything you did not grant.
  - Learn: default SA token is a projected, short-lived, audience-bound token now — not a permanent Secret.
- [ ] **5.2 — securityContext hardening** 🟢 ⏱2h — `runAsNonRoot`, `runAsUser/Group`, `fsGroup`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]`, `seccompProfile: RuntimeDefault`.
  - Expect breakage: read-only rootfs forces you to mount `emptyDir` on `/tmp`. That is the lesson.
- [ ] **5.3 — Pod Security Admission** 🟢 ⏱2h — label the namespace `pod-security.kubernetes.io/enforce: restricted`, watch non-compliant pods get rejected at admission. (PodSecurityPolicy is **removed** — do not learn it.)
- [ ] **5.4 — Policy as code** 🟢 ⏱4h — **Kyverno** (easier) or OPA/Gatekeeper: enforce "no `:latest` tag", "requests required", "must have `team` label". Then learn what a ValidatingAdmissionWebhook / `ValidatingAdmissionPolicy` (CEL) actually is.
- [ ] **5.5 — Supply chain** 🟢 ⏱3h — scan images (`trivy image`), scan manifests (`trivy config`, `kubescape`), build a minimal/distroless image, sign with `cosign` and verify at admission.
- [ ] **5.6 — Cluster-level security reading** ⏱2h — etcd encryption at rest, API server audit logs, why `hostPath`/`privileged`/`hostNetwork`/`hostPID` are escape hatches, control-plane component roles (`kube-apiserver`, `etcd`, `kube-scheduler`, `controller-manager`, `kubelet`, `kube-proxy`).

---

## Phase 6 — Secrets done properly

- [ ] **6.1 — External Secrets Operator** 🟢 ⏱4h — sync from Vault / AWS Secrets Manager / GCP SM into a k8s Secret via `SecretStore` + `ExternalSecret`. Vault in dev mode is fine locally.
- [ ] **6.2 — GitOps-safe secrets** 🟢 ⏱2h — **Sealed Secrets** or **SOPS + age**: encrypted secrets that are safe to commit. Pick one and use it for the rest of the repo.
- [ ] **6.3 — Config rollout on change** 🟢 ⏱1h — mounted ConfigMaps update in place but the app does not reread them; env vars never update. Fix with the `checksum/config` pod-annotation trick (or a reloader) so a ConfigMap change triggers a rollout. Also try `immutable: true`.

---

## Phase 7 — Packaging and delivery

- [ ] **7.1 — Kustomize** 🟢 ⏱4h — turn one lesson into `base/` + `overlays/{dev,staging,prod}` with patches, `namePrefix`, `commonLabels`, `configMapGenerator`, `secretGenerator`, image tag overrides. `kubectl kustomize` / `kubectl apply -k`.
- [ ] **7.2 — Helm, as a consumer** 🟢 ⏱3h — `helm repo add`, `install`, `upgrade`, `rollback`, `-f values.yaml`, `--set`, `helm template` to see rendered output before applying.
- [ ] **7.3 — Helm, as an author** 🟢 ⏱6h — package the Mongo+webapp stack as a chart: `values.yaml`, `_helpers.tpl`, `NOTES.txt`, `values.schema.json`, subchart dependency, `helm lint`, `helm unittest`.
- [ ] **7.4 — GitOps with Argo CD (or Flux)** 🟢 ⏱6h — install Argo CD in Minikube, point an `Application` at this repo, prove drift correction (`kubectl edit` a Deployment → watch it revert), then enable auto-sync + self-heal + prune. Learn app-of-apps and sync waves.
- [ ] **7.5 — CI pipeline** 🟢 ⏱4h — GitHub Actions: lint (`kubeconform`, `helm lint`) → build+push image by digest → scan → bump the digest in the GitOps repo. **CI writes the manifest; Argo does the deploy.** Never `kubectl apply` from CI in a GitOps setup.

---

## Phase 8 — Observability

- [ ] **8.1 — Metrics stack** 🟢 ⏱6h — `kube-prometheus-stack` via Helm; expose app `/metrics`; a `ServiceMonitor`; a Grafana dashboard; a `PrometheusRule` alert that you deliberately fire.
- [ ] **8.2 — The four signals for k8s** ⏱2h — latency/traffic/errors/saturation, plus what to alert on: `CrashLoopBackOff`, pod restarts, PVC near-full, node `NotReady`, HPA at max, certificate expiry.
- [ ] **8.3 — Logs** 🟢 ⏱3h — Loki + Promtail (or Fluent Bit); why `kubectl logs` alone is not observability (pod dies → logs gone); structured JSON logging; log-volume cost.
- [ ] **8.4 — Tracing** 🟢 ⏱4h — OpenTelemetry Collector + Jaeger/Tempo; propagate trace context through the webapp.
- [ ] **8.5 — Debugging toolkit** 🟢 ⏱2h — `kubectl debug` (ephemeral container + node debug + pod copy), `port-forward`, `exec`, `cp`, `top`, `events -w`, `-o jsonpath`, `--dry-run=server -o yaml`, `kubectl diff`. Install `stern`, `kubectx`, `kubens`, `k9s`.

---

## Phase 9 — Real clusters and operations

- [ ] **9.1 — Multi-node local cluster** 🔵 ⏱2h — `kind` with 1 control-plane + 3 workers. Everything marked 🔵 earlier becomes testable here.
- [ ] **9.2 — Build a cluster from scratch once** 🔵 ⏱6h — `kubeadm init` on two VMs, install a CNI, join a worker. You never need this again, but it makes the control plane concrete.
- [ ] **9.3 — Managed cluster** 🔵 ⏱6h — one of EKS/GKE/AKS via Terraform or eksctl. Learn: node pools, IRSA / Workload Identity (pod→cloud auth **without** static keys), cloud LoadBalancer Services, CSI drivers, real Ingress + DNS + ACME.
  - ⚠️ Costs money. Set a budget alert and `terraform destroy` the same day.
- [ ] **9.4 — Upgrades** 🔵 ⏱3h — version skew policy, upgrade order (control plane → nodes), `cordon`/`drain`/`uncordon`, surge upgrades, and finding removed APIs before they break you (`kubent` / `pluto`).
- [ ] **9.5 — Cluster backup/DR** 🔵 ⏱3h — **Velero**: back up namespace + PVs, delete the namespace, restore it. Also: etcd snapshot/restore theory.
- [ ] **9.6 — Multi-tenancy and cost** 🟢 ⏱3h — `ResourceQuota`, `LimitRange`, namespace-per-team, `kubecost`/OpenCost, and how over-requesting silently burns money.

---

## Phase 10 — Extend Kubernetes (senior-level differentiator)

- [ ] **10.1 — CRDs** 🟢 ⏱3h — write a `CustomResourceDefinition` with an OpenAPI v3 schema, `subresources.status`, `additionalPrinterColumns`, and a short name. Apply a CR of it.
- [ ] **10.2 — A real controller/operator** 🟢 ⏱10h — `kubebuilder` (Go) or `kopf` (Python): implement a `Reconcile` loop for a small CRD (e.g. `Backup` → creates a Job). Learn the control loop, level-triggered reconciliation, idempotency, `ownerReferences` + garbage collection, finalizers, and status conditions.
- [ ] **10.3 — Service mesh** 🔵 ⏱6h — Istio (ambient mode) or Linkerd: mTLS between services, retries/timeouts, traffic splitting for canaries, mesh observability. Know the sidecar cost trade-off and when a mesh is *not* worth it.
- [ ] **10.4 — Progressive delivery** 🟢 ⏱4h — Argo Rollouts or Flagger: canary with automated metric analysis and auto-rollback.

---

## Production readiness checklist (memorize this)

Score every workload you ever ship:

| # | Requirement | Verify with |
| - | ----------- | ----------- |
| 1 | Requests **and** limits on every container | `kubectl describe pod` → QoS |
| 2 | Readiness + liveness (+ startup if slow) probes | rollout with zero 5xx |
| 3 | `replicas >= 2` + pod anti-affinity/spread + PDB | `kubectl drain` a node |
| 4 | Graceful shutdown: SIGTERM handled, `preStop`, grace period | load test during rollout |
| 5 | Non-root, read-only rootfs, all caps dropped, `restricted` PSA | admission rejects a bad pod |
| 6 | Dedicated ServiceAccount, least-privilege RBAC | `kubectl auth can-i` |
| 7 | Secrets from an external store, never in git plaintext | repo grep |
| 8 | Images pinned by digest, scanned, signed | CI gate |
| 9 | NetworkPolicy default-deny + explicit allows | curl from a blocked pod |
| 10 | Metrics + logs + alerts wired, dashboards exist | fire a test alert |
| 11 | State on PVCs with a **tested** restore | actually restore |
| 12 | Declarative + GitOps: cluster == git, no manual `kubectl apply` | drift test |
| 13 | HPA (and headroom) sized for real load | load test |
| 14 | Rollback path proven | `rollout undo` / Argo rollback |

---

## Capstone projects (5–7, build in this order)

Each is a repo of its own with a README covering architecture, trade-offs, and failure modes. **Failure-mode writeups are what make these credible** — plain "I deployed an app" is not.

- [ ] **P1 — Production-grade 3-tier app on Minikube/kind** ⏱1 week
  React/Next front end + API + Postgres (StatefulSet + PVC) + Redis. Every box in the checklist above ticked. Kustomize overlays for dev/prod. Deliverable: a "chaos log" — kill a pod, drain a node, fill a disk, OOM a container; document detection and recovery for each.

- [ ] **P2 — GitOps platform** ⏱1 week
  Argo CD app-of-apps managing ingress-nginx, cert-manager, external-secrets, kube-prometheus-stack, plus your apps. Two environments from one repo. Deliverable: proof of drift correction and a one-commit rollback.

- [ ] **P3 — Full observability stack with SLOs** ⏱1 week
  Prometheus + Grafana + Loki + Tempo + OTel on an instrumented app. Define an SLO, an error budget, and alert rules. Deliverable: a screenshotted incident — inject latency, show the alert firing, trace to root cause.

- [ ] **P4 — Secure multi-tenant cluster** ⏱1 week
  Namespace-per-team, ResourceQuota/LimitRange, `restricted` PSA, default-deny NetworkPolicies, Kyverno policy pack, per-team RBAC, image signing verified at admission. Deliverable: an attack/misconfig log — 10 things you tried to do wrong and the exact control that blocked each.

- [ ] **P5 — CI/CD with progressive delivery** ⏱1 week
  GitHub Actions → build/scan/sign → digest bump → Argo CD → Argo Rollouts canary with metric analysis and automatic rollback. Deliverable: a video/log of a bad deploy auto-rolling back.

- [ ] **P6 — Managed cloud cluster via IaC** ⏱1–2 weeks ⚠️ costs money
  Terraform: VPC + EKS/GKE, node pools (incl. spot), IRSA/Workload Identity, cloud LB + real DNS + Let's Encrypt cert, cluster autoscaler, Velero backups to object storage. Deliverable: `terraform apply` from zero to a working HTTPS app, then a documented cluster upgrade, then `destroy`. Include the cost breakdown.

- [ ] **P7 — Custom operator** ⏱1–2 weeks
  Kubebuilder operator for something you actually want: e.g. a `ManagedDatabase` CRD that provisions a StatefulSet + Secret + backup CronJob, with status conditions, finalizers, and owner-reference GC. Deliverable: unit + envtest coverage and a design doc on reconciliation idempotency.

**Minimum credible portfolio: P1 + P2 + P3 + P4.** Add P5 for a DevOps/platform role, P6 if the job is cloud-heavy, P7 for senior/platform-engineer positioning.

---

## Certification alignment (optional)

- **CKA** ≈ Phases 0–5 + 9 (cluster ops, troubleshooting, heavy `kubectl` speed).
- **CKAD** ≈ Phases 0–4 + 6–7 (workloads, config, observability from the app side).
- **CKS** ≈ Phase 5 + 6 (+ requires a valid CKA).
All three are hands-on, time-pressured terminals. Practice with `--dry-run=client -o yaml` scaffolding and `kubectl explain`; do not memorize YAML by hand.
