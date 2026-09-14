# Horizontal Pod Autoscaler — Study Notes

Same MongoDB + Node.js web app as lesson 4, but the webapp Deployment no longer has a fixed
replica count. A **HorizontalPodAutoscaler** watches CPU and memory usage through
**metrics-server** and rewrites `spec.replicas` between 2 and 10 as load moves. Mongo stays a
3-replica StatefulSet — deliberately *not* autoscaled, which is half the lesson.

## What changed from lesson 4

| Lesson 4                                  | Lesson 5                                                       |
| ----------------------------------------- | -------------------------------------------------------------- |
| `webapp-deployment` pinned to 2 replicas  | No `replicas:` field — the HPA owns it                          |
| No `resources` on any container           | `requests` + `limits` on webapp *and* mongo                     |
| No probes                                 | Readiness + liveness probes on the webapp                       |
| —                                         | `webapp-hpa.yaml`: `autoscaling/v2`, CPU 50% + memory 80%       |
| —                                         | `behavior` block tuning scale-up speed and scale-down damping   |
| —                                         | `load/load-generator.yaml` to actually trigger a scale event    |
| Cluster needs the ingress addon           | Also needs the **metrics-server** addon                         |

## Architecture

```
        ┌──────────────┐   reads pod CPU/mem   ┌─────────────────┐
        │ HPA          │◄──────────────────────│ metrics-server  │◄── kubelet /metrics/resource
        │ webapp-hpa   │   (metrics.k8s.io)    └─────────────────┘        on every node
        └──────┬───────┘
               │ writes spec.replicas via the /scale subresource
               ▼
     webapp-deployment  ──►  ReplicaSet  ──►  pod pod pod …  (2 … 10)
               ▲                                   │
               │                                   ▼
   webapp-service (ClusterIP :3000) ◄── load-generator pods (in-cluster traffic)
               ▲
               │
   webapp-ingress (host: myapp.com) ◄── browser

     webapp pods ──► mongo-0.mongo-service ──► mongo StatefulSet (3 replicas, FIXED)
                                                    each with its own PVC/PV
```

The HPA is a control loop in kube-controller-manager, running every **15s** by default
(`--horizontal-pod-autoscaler-sync-period`). It is not a webhook and not event-driven — a load
spike takes up to one sync period plus a metrics-scrape interval (~15s more) to even be *seen*.

## Files

| File                       | Kind                               | Purpose                                                    |
| -------------------------- | ---------------------------------- | ---------------------------------------------------------- |
| `mongo-config-map.yaml`    | ConfigMap                          | `mongo-url` = `mongo-0.mongo-service`                       |
| `mongo-secrets.yaml`       | Secret                             | base64 DB user + password                                   |
| `mongo-storage.yaml`       | StorageClass + 3 PersistentVolumes | `local-storage` class + one node-pinned 1Gi PV per node      |
| `mongo-statefulset.yaml`   | Service (headless) + StatefulSet   | Mongo, 3 replicas, now with resource requests/limits         |
| `webapp-deployment.yaml`   | Deployment + Service + Ingress     | Web app: **no replica count**, requests/limits, probes       |
| `webapp-hpa.yaml`          | HorizontalPodAutoscaler            | `autoscaling/v2`, 2–10 replicas, CPU 50% + memory 80%        |
| `load/load-generator.yaml` | Deployment                         | busybox request loop — applied **separately**, on purpose    |

## Prerequisite: metrics-server

The HPA reads `metrics.k8s.io`, an API served by metrics-server. Without it the HPA reports
`<unknown>` forever and never scales.

```bash
minikube addons enable metrics-server
kubectl -n kube-system rollout status deployment/metrics-server
kubectl top nodes          # both commands stay empty for ~60s after enabling
kubectl top pods
```

metrics-server holds a short in-memory window only (~last 15–30s per pod). It is **not**
monitoring — no history, no queries, no alerting. That's Prometheus' job.

## How the HPA decides

```
desiredReplicas = ceil( currentReplicas × ( currentMetricValue / desiredMetricValue ) )
```

Averaged across all **Ready** pods. Worked example with `averageUtilization: 50` and a `100m`
request (so the target is 50m per pod):

| Current pods | Avg usage | Ratio       | Raw       | Desired      |
| ------------ | --------- | ----------- | --------- | ------------ |
| 2            | 90m       | 90/50 = 1.8 | 3.6       | **4**        |
| 4            | 55m       | 1.1         | 4.4       | 4 (tolerance) |
| 4            | 10m       | 0.2         | 0.8       | **2** (minReplicas) |

Rules that fall out of that formula:

- **Utilization is a percentage of `requests`, not of `limits` and not of node capacity.** A pod
  requesting `100m` and burning `200m` reports **200%**. Values above 100% are normal and are
  exactly what triggers scaling.
- **±10% tolerance.** A ratio inside `[0.9, 1.1]` is treated as "on target" and nothing happens.
  This is a cluster-wide flag (`--horizontal-pod-autoscaler-tolerance`); per-HPA `behavior.*.tolerance`
  exists but is alpha as of v1.33 (`HPAConfigurableTolerance` gate) and is off on minikube.
- **Pods that are not Ready, or still starting, are excluded** from the average, so a cold-start
  CPU burst can't stampede the count upward.
- **Multiple metrics → max wins.** Each metric produces its own desired count; the largest is
  used. Scale-*down* therefore requires *every* metric to agree.
- **If any metric is unavailable, scale-down is blocked** (scale-up is still allowed). Safe
  default: unknown load means don't shed capacity.

## The `behavior` block

Defaults, when `behavior` is omitted:

| Direction  | Stabilization | Policy                                   |
| ---------- | ------------- | ---------------------------------------- |
| scale up   | 0s            | max(+4 pods, +100%) per 60s              |
| scale down | **300s**      | 100% (down to `minReplicas`) per 60s     |

That 300s scale-down stabilization window is the single most surprising default: after load
drops, pods linger for five minutes. The window works by keeping the **highest recommendation
seen in the window** and scaling only to that — so a dip has to persist for the whole window
before anything shrinks.

This lesson's tuning, and why:

```yaml
scaleUp:
  stabilizationWindowSeconds: 0 # spikes hurt now; react now
  policies:
    - { type: Percent, value: 100, periodSeconds: 30 } # double, or...
    - { type: Pods, value: 4, periodSeconds: 30 } # ...+4, whichever is more
  selectPolicy: Max
scaleDown:
  stabilizationWindowSeconds: 120 # shortened from 300 so the demo finishes
  policies:
    - { type: Percent, value: 50, periodSeconds: 60 } # halve at most, per minute
  selectPolicy: Max
```

`selectPolicy: Max` picks the most permissive policy, `Min` the most restrictive.
`selectPolicy: Disabled` on a direction **forbids scaling that way entirely** — the usual way to
build an up-only autoscaler for workloads where terminating a pod is expensive.

## Apply order

```bash
minikube start --nodes 3          # lesson 4 built this 3-node cluster
minikube addons enable ingress
minikube addons enable metrics-server

kubectl apply -f mongo-config-map.yaml
kubectl apply -f mongo-secrets.yaml
kubectl apply -f mongo-storage.yaml
kubectl apply -f mongo-statefulset.yaml
kubectl apply -f webapp-deployment.yaml
kubectl apply -f webapp-hpa.yaml
```

Or the whole directory: `kubectl apply -f .` — non-recursive, so `load/` is skipped. That is the
point of the subdirectory.

Hosts entry, as in lesson 2:

```bash
echo "$(minikube ip) myapp.com" | sudo tee -a /etc/hosts
```

## Inspect

```bash
kubectl get hpa webapp-hpa
# NAME         REFERENCE                      TARGETS                        MINPODS  MAXPODS  REPLICAS
# webapp-hpa   Deployment/webapp-deployment   cpu: 3%/50%, memory: 41%/80%   2        10       2

kubectl describe hpa webapp-hpa      # Conditions + the scaling Events log — read this first
kubectl top pods -l app=webapp       # the raw numbers the HPA is dividing
kubectl get hpa webapp-hpa -o yaml   # status.currentMetrics, status.lastScaleTime
```

Healthy `Conditions` in `describe`:

- `AbleToScale: True`
- `ScalingActive: True` — metrics are arriving and being used
- `ScalingLimited: False` — or `True` with reason `TooManyReplicas` when pinned at `maxReplicas`

`TARGETS` showing `<unknown>/50%` means metrics-server is missing/not ready, or the pod has no
CPU request.

## Load test

Three terminals.

```bash
# 1 — watch the autoscaler
kubectl get hpa webapp-hpa -w

# 2 — watch pods appear and disappear
kubectl get pods -l app=webapp -w
```

```bash
# 3 — drive load
kubectl apply -f load/load-generator.yaml
kubectl scale deployment load-generator --replicas=8    # if CPU stays under target
```

Expected timeline: ~15–45s until `TARGETS` climbs past `50%`, then a first scale-up event
(2 → 4, doubling policy), further steps every 30s while the target stays exceeded, settling
somewhere below 10.

```bash
kubectl delete -f load/load-generator.yaml
```

CPU falls at once, replicas do **not** — the 120s scale-down stabilization window holds them,
then the 50%-per-minute policy steps down: 8 → 4 → 2. Total ~3 minutes. Watch it as events:

```bash
kubectl describe hpa webapp-hpa | tail -20
kubectl get events --field-selector involvedObject.name=webapp-hpa --sort-by=.lastTimestamp
```

Single-shot manual scale, for comparison — and to see the HPA overrule it within ~15s:

```bash
kubectl scale deployment webapp-deployment --replicas=7
kubectl get hpa webapp-hpa -w        # returns to 2 once the window elapses
```

## Why mongo is not autoscaled

An HPA *can* target a StatefulSet — `scaleTargetRef` accepts anything with a `/scale`
subresource. It would be wrong here on three counts:

1. **Storage doesn't autoscale.** Lesson 4 hand-wrote three PVs. Replica 4 has nothing to bind
   and sits `Pending` forever. The HPA keeps counting it as a pod it asked for.
2. **A new mongod is not more capacity.** These are three unrelated databases; adding a fourth
   adds an empty one. Even in a real replica set, extra members are read capacity at best —
   writes still funnel through one primary. Databases scale by sharding, not by replica count.
3. **Scale-down destroys the wrong thing.** Removing the highest ordinal takes a data-holding
   member offline; done under load, that *raises* pressure on the rest.

The general rule: HPA is for **stateless, horizontally-shardable request handlers**. Stateful
tiers scale by vertical sizing, read replicas with a routing layer, or sharding.

## Gotchas

- **No `requests` → no HPA.** Utilization has no denominator; `TARGETS` reads `<unknown>` and the
  HPA never acts. The most common failure by a wide margin.
- **Keeping `replicas:` in the Deployment fights the HPA.** Every `kubectl apply` (or Argo/Flux
  sync) resets the count, the HPA re-scales, and the loop repeats. Omit the field, or make the
  GitOps tool ignore it. Note the flip side: on first create, a Deployment with no `replicas`
  starts at **1** until the HPA raises it to `minReplicas`.
- **The HPA never scales to 0.** `minReplicas: 0` requires the `HPAScaleToZero` feature gate; for
  real scale-to-zero use KEDA or Knative.
- **`minReplicas == maxReplicas`** is a valid way to freeze scaling temporarily without deleting
  the HPA.
- **Memory is a weak autoscaling signal.** Most runtimes (JVM, Node, Go) grow the heap and never
  return it, so memory utilization ratchets up and stays there — the HPA adds pods that don't
  help, and scale-down never triggers. It is included here to show multi-metric arithmetic. Real
  systems scale on RPS, queue depth, or latency (lesson: custom metrics / KEDA).
- **CPU limits distort the signal.** Once a pod is throttled at its limit, measured CPU flattens
  at the ceiling — the pod is starving but utilization no longer climbs, so the HPA under-reacts.
  Setting the CPU limit well above the request (300m vs 100m here) leaves room to observe real
  demand.
- **HPA + VPA on the same workload conflict** unless the VPA is in recommendation mode
  (`updateMode: "Off"`) or they target different resources. Both edit the same feedback loop.
- **Scaling is only as fast as pod startup.** The HPA can request 10 pods in a second; a 40s image
  pull plus a 15s readiness delay means the capacity arrives a minute later. Autoscaling does not
  absorb instant spikes — that needs headroom (a lower target %) or pre-warming.
- **`maxReplicas` must actually fit the cluster.** Beyond node capacity the extra pods are
  `Pending`, `ScalingLimited` stays quiet about it, and the HPA thinks it succeeded.
- **Missing probes make scale-up look worse than no scaling.** Without a readiness probe a pod
  joins the Service the moment the container starts and drops requests until it's warm.
- **`kubectl top` failing means metrics-server is broken, not the HPA.** Debug from the bottom:
  `kubectl top pods` → `kubectl -n kube-system logs deploy/metrics-server` →
  `kubectl get apiservice v1beta1.metrics.k8s.io`.
- **On non-minikube clusters metrics-server often needs `--kubelet-insecure-tls`** because kubelet
  serving certs aren't signed by the cluster CA. The minikube addon handles this already.
- **`autoscaling/v1` silently drops what it can't express.** `kubectl get hpa -o yaml` may return
  the v1 shape, hiding memory metrics and `behavior` in an annotation. Ask for the version:
  `kubectl get hpa webapp-hpa -o yaml --output-version=autoscaling/v2`.

## Tear down

```bash
kubectl delete -f load/load-generator.yaml --ignore-not-found
kubectl delete -f .
kubectl delete pvc -l app=mongo
minikube addons disable metrics-server
```

PVCs from `volumeClaimTemplates` still need the explicit delete — see lesson 4.

## Next

- **Custom metrics** (RPS, queue depth) via Prometheus Adapter or KEDA — what production actually
  scales on.
- **VPA** for right-sizing the `requests` this lesson depends on.
- **Cluster autoscaler** — HPA adds pods, but something has to add *nodes* when they don't fit.
