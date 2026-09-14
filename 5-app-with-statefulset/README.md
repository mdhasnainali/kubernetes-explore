# StatefulSet with Local Storage — Study Notes

Same MongoDB + Node.js web app as lesson 3, but MongoDB now runs as a **StatefulSet with 3
replicas**, each pod getting its **own** PersistentVolumeClaim carved out of **local
(node-attached) storage**. The web app stays a Deployment — now with 2 replicas — so the
stateless/stateful contrast is visible side by side.

## What changed from lesson 3

| Lesson 3                                | Lesson 4                                                       |
| --------------------------------------- | -------------------------------------------------------------- |
| `Deployment`, 1 replica                 | `StatefulSet`, 3 replicas                                      |
| Random pod name `mongo-deployment-x7f2` | Stable ordinal names `mongo-0`, `mongo-1`, `mongo-2`            |
| ClusterIP Service (one VIP)             | **Headless** Service (`clusterIP: None`) + per-pod DNS          |
| One hand-written PV + one PVC           | `volumeClaimTemplates` → one PVC per pod                        |
| `storageClassName: manual`, Immediate   | `local-storage` class, `WaitForFirstConsumer`, node-pinned PVs  |
| `strategy: Recreate`                    | Ordered rollout is built in — no strategy hack needed           |

## Honest caveat: 3 mongods ≠ a Mongo replica set

This lesson teaches **StatefulSet mechanics**, not MongoDB clustering. The 3 pods are three
**independent, unrelated** mongod processes, each with its own empty database on its own
volume. Nothing replicates between them. The webapp is pinned to `mongo-0` via the ConfigMap
so writes always land in the same database.

A real Mongo replica set additionally needs: `--replSet rs0` on every mongod, a one-time
`rs.initiate()` naming all three pod DNS names, a shared keyfile Secret for internal auth, and
a client connection string listing all members. That's a MongoDB lesson, not a Kubernetes one —
see the last section for the sketch.

## Architecture

```
  browser ──► nginx ingress controller ──► webapp-ingress (host: myapp.com)
                                                │
                                                ▼
                                     webapp-service (ClusterIP :3000)
                                          │            │
                                          ▼            ▼
                                   webapp pod     webapp pod     (Deployment, 2 replicas)
                                          │            │
                                          └─────┬──────┘   DB_URL = mongo-0.mongo-service
                                                ▼
                            mongo-service (headless, clusterIP: None)
                                                │  DNS → pod IPs, plus per-pod names
                ┌───────────────────────────────┼───────────────────────────────┐
                ▼                               ▼                               ▼
        mongo-0 (:27017)                mongo-1 (:27017)                mongo-2 (:27017)
                │                               │                               │
        mongo-data-mongo-0              mongo-data-mongo-1              mongo-data-mongo-2   (PVCs)
                │                               │                               │
          mongo-pv-0                      mongo-pv-1                      mongo-pv-2         (PVs)
                │                               │                               │
        node: minikube                node: minikube-m02              node: minikube-m03
        /data/mongo                     /data/mongo                     /data/mongo
```

Each PV is pinned to one node by `nodeAffinity`, so pod ↔ node ↔ disk is a fixed triple.

## Files

| File                     | Kind                               | Purpose                                                   |
| ------------------------ | ---------------------------------- | --------------------------------------------------------- |
| `mongo-config-map.yaml`  | ConfigMap                          | `mongo-url` = `mongo-0.mongo-service` (per-pod DNS name)   |
| `mongo-secrets.yaml`     | Secret                             | base64 DB user + password                                 |
| `mongo-storage.yaml`     | StorageClass + 3 PersistentVolumes | `local-storage` class + one node-pinned 1Gi PV per node    |
| `mongo-statefulset.yaml` | Service (headless) + StatefulSet   | Mongo StatefulSet, 3 replicas, `volumeClaimTemplates`      |
| `webapp-deployment.yaml` | Deployment + Service + Ingress     | Web app, 2 replicas, ClusterIP :3000, Ingress `myapp.com`  |

## Why a StatefulSet

A Deployment treats pods as interchangeable cattle: random names, random start order, all
sharing whatever volumes the template names. Stateful workloads need the opposite.

**Stable identity.** Pod names are `<statefulset>-<ordinal>`: `mongo-0`, `mongo-1`, `mongo-2`.
Delete `mongo-1` and the replacement is *also* called `mongo-1` and reattaches to the *same*
PVC. A Deployment's replacement gets a fresh random name.

**Stable storage.** `volumeClaimTemplates` creates one PVC per pod, named
`<template>-<statefulset>-<ordinal>`. That binding survives pod deletion, node reboot, and even
StatefulSet deletion.

**Stable network name.** With the headless Service, each pod is addressable at
`<pod>.<service>.<namespace>.svc.cluster.local`, e.g.
`mongo-0.mongo-service.default.svc.cluster.local` (short form `mongo-0.mongo-service` works
inside the cluster). Peers can name each other — required by every clustered database.

**Ordered lifecycle.** Default `podManagementPolicy: OrderedReady`:

- scale up / create: `mongo-0` must be Running **and Ready** before `mongo-1` starts
- scale down / delete: reverse order, `mongo-2` first
- rolling update: highest ordinal first, one pod at a time

That's why no `strategy: Recreate` workaround is needed — two mongods never share a data dir,
because each has its own, and the rollout never doubles up a pod.

## Manifest anatomy

### Headless Service

```yaml
spec:
  clusterIP: None # ← the whole difference
  selector:
    app: mongo
```

`clusterIP: None` means no virtual IP and no kube-proxy load balancing. A DNS lookup of
`mongo-service` returns the A records of all ready pods; a lookup of `mongo-0.mongo-service`
returns exactly that one pod. The StatefulSet's `serviceName` field must name this Service —
that field is what creates the per-pod DNS records.

### volumeClaimTemplates

```yaml
spec:
  template: ...
  volumeClaimTemplates: # sibling of `template`, NOT inside it
    - metadata:
        name: mongo-data # the container's volumeMount references this name
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: local-storage
        resources:
          requests:
            storage: 1Gi
```

No `spec.template.spec.volumes` entry is needed — the StatefulSet controller injects the
generated PVC into each pod. Resulting objects:

```
PVC mongo-data-mongo-0  →  pod mongo-0
PVC mongo-data-mongo-1  →  pod mongo-1
PVC mongo-data-mongo-2  →  pod mongo-2
```

`volumeClaimTemplates` is **immutable** apart from `resources.requests` (and even that only
with an expandable StorageClass). Changing the class or access mode means deleting and
recreating the StatefulSet.

### Local storage: StorageClass + node-pinned PVs

```yaml
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner # nobody provisions; the PVs are hand-written
volumeBindingMode: WaitForFirstConsumer
```

```yaml
kind: PersistentVolume
spec:
  storageClassName: local-storage
  hostPath:
    path: /data/mongo
    type: DirectoryOrCreate
  nodeAffinity: # the scheduler will only place the consuming pod on this node
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: [minikube]
```

Two pieces carry the weight:

- **`nodeAffinity` on the PV** — the data physically lives on one node's disk, so the pod that
  claims it must be scheduled there. Without this, the scheduler could put `mongo-1` on a node
  where `/data/mongo` is a different (empty) directory.
- **`volumeBindingMode: WaitForFirstConsumer`** — delay PVC→PV binding until a pod is being
  scheduled, so the scheduler weighs pod constraints and volume topology **together**. With the
  default `Immediate`, the PVC binds to a random matching PV the moment it is created, and that
  arbitrary choice can pin the pod to a node it cannot otherwise run on → permanent `Pending`.

Three PVs exist because there are three replicas. **One PVC binds one PV, exclusively.** Scale
to 4 and `mongo-3` sits `Pending` until a 4th PV is added. Hand-written PVs don't autoscale —
that's what a dynamic CSI provisioner is for.

### Why not minikube's default `standard` class

This cluster has 3 nodes. `standard` (`k8s.io/minikube-hostpath`) is `Immediate`-binding and
creates a directory through the provisioner pod on the control-plane node, carrying no topology
information. A pod scheduled onto `minikube-m02` then mounts an empty
`/tmp/hostpath-provisioner/...` on *that* node's disk. It appears to work, then silently loses
data. `standard` is single-node-only.

## Apply order

```bash
kubectl apply -f mongo-config-map.yaml
kubectl apply -f mongo-secrets.yaml
kubectl apply -f mongo-storage.yaml        # StorageClass + PVs first
kubectl apply -f mongo-statefulset.yaml
kubectl apply -f webapp-deployment.yaml
```

Or the whole directory: `kubectl apply -f .`

Ingress still needs the controller and the hosts entry from lesson 2:

```bash
minikube addons enable ingress
echo "$(minikube ip) myapp.com" | sudo tee -a /etc/hosts
```

## Inspect

```bash
kubectl get pods -l app=mongo -o wide      # mongo-0/1/2, one per node
kubectl get pvc                            # mongo-data-mongo-{0,1,2}, all Bound
kubectl get pv                             # each Bound to its matching claim
kubectl get statefulset mongo
kubectl describe statefulset mongo         # ordinal create/scale events
kubectl get svc mongo-service              # CLUSTER-IP column shows None
```

Watch the ordered startup live:

```bash
kubectl delete statefulset mongo && kubectl apply -f mongo-statefulset.yaml
kubectl get pods -l app=mongo -w           # 0 Ready → then 1 starts → then 2
```

DNS from inside the cluster:

```bash
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -c 'nslookup mongo-service; nslookup mongo-0.mongo-service'
```

First lookup returns 3 IPs, second returns 1.

Data on the nodes:

```bash
minikube ssh -n minikube     -- sudo ls -l /data/mongo
minikube ssh -n minikube-m02 -- sudo ls -l /data/mongo
minikube ssh -n minikube-m03 -- sudo ls -l /data/mongo
```

## Prove identity + storage are stable

```bash
# write a marker into mongo-0's database
kubectl exec -it mongo-0 -- mongosh -u mongouser -p mongopassword \
  --eval 'db.getSiblingDB("test").marker.insertOne({hello: "lesson4"})'

# nuke the pod
kubectl delete pod mongo-0
kubectl get pods -l app=mongo -w        # the replacement is ALSO named mongo-0

# same name, same PVC, same node, data intact
kubectl exec -it mongo-0 -- mongosh -u mongouser -p mongopassword \
  --eval 'db.getSiblingDB("test").marker.find()'
```

Stronger: delete the StatefulSet itself, re-apply, data still there — PVCs from
`volumeClaimTemplates` are **not** garbage-collected with the StatefulSet.

Confirm the pods really are separate databases (the caveat above) — `mongo-1` has no marker:

```bash
kubectl exec -it mongo-1 -- mongosh -u mongouser -p mongopassword \
  --eval 'db.getSiblingDB("test").marker.countDocuments()'   # 0
```

## Scaling

```bash
kubectl scale statefulset mongo --replicas=2   # deletes mongo-2 only; its PVC survives
kubectl scale statefulset mongo --replicas=3   # mongo-2 returns and reattaches its old PVC
kubectl scale statefulset mongo --replicas=4   # mongo-3 stays Pending: no 4th PV exists
```

Scale-down deleting the pod but keeping the PVC is deliberate — the data may still be wanted.
Cleanup is manual, or automatic via `persistentVolumeClaimRetentionPolicy`.

## Tear down

```bash
kubectl delete -f .
kubectl delete pvc -l app=mongo
for n in minikube minikube-m02 minikube-m03; do
  minikube ssh -n $n -- sudo rm -rf /data/mongo
done
```

The PVCs need an explicit delete: `kubectl delete -f .` removes the StatefulSet, not the PVCs
it generated.

## Gotchas

- **PVC `Pending` with `waiting for first consumer`** — normal for a few seconds under
  `WaitForFirstConsumer`; it clears once a pod schedules. If it persists, no PV matches a node
  the pod can run on.
- **Fewer PVs than replicas** → the highest ordinals stay `Pending` forever. Hand-written PVs
  do not autoscale.
- **`volumeClaimTemplates` is immutable.** Editing the class, access mode, or name is rejected;
  delete the StatefulSet (`--cascade=orphan` to keep pods running) and recreate.
- **Deleting a StatefulSet leaves the PVCs.** Intentional — but it also means a "clean"
  reinstall can silently pick up old data.
- **`serviceName` must match an existing headless Service**, or there are no per-pod DNS
  records and peers can't find each other. Nothing errors loudly — DNS just fails.
- **A headless Service does not load balance.** Clients get all pod IPs and choose for
  themselves. Pointing the webapp at plain `mongo-service` would spread requests across three
  unrelated databases; hence the `mongo-0.` prefix in the ConfigMap.
- **`OrderedReady` blocks on readiness.** With no readiness probe, "Ready" means "container
  started", which for a database is optimistic. A wedged `mongo-0` stalls the whole rollout —
  use `podManagementPolicy: Parallel` when pods genuinely don't depend on each other.
- **`hostPath` + `nodeAffinity` is the learning-grade version of local storage.** The real thing
  is `spec.local.path` instead of `hostPath`: it refuses to mount unless the directory already
  exists on the node (safer — a typo can't silently create a fresh empty dir), and it's normally
  managed by the local-static-provisioner. `hostPath` is used here only so
  `type: DirectoryOrCreate` saves a manual `mkdir` on all three nodes.
- **`hostPath` is a security hole on real clusters** — a pod can mount any path on the host.
- **`minikube delete` destroys all three `/data/mongo` directories** along with the nodes.
  `minikube stop` / `start` keeps them.
- **Mongo creates the root user only on an empty `/data/db`.** Changing the Secret after data
  exists does nothing — the init script never re-runs.
- **Secrets are base64, not encrypted.**

## Sketch: turning this into a real Mongo replica set

Not implemented here — reference for what the *database* layer additionally requires:

1. `command: ["mongod", "--replSet", "rs0", "--keyFile", "/etc/mongo-keyfile/key", "--bind_ip_all"]`
2. A Secret holding a shared keyfile (internal member auth), mounted mode `0400` and owned by
   the `mongodb` user — usually via an initContainer that copies and chowns it.
3. Run once, against `mongo-0`:
   ```js
   rs.initiate({
     _id: "rs0",
     members: [
       { _id: 0, host: "mongo-0.mongo-service:27017" },
       { _id: 1, host: "mongo-1.mongo-service:27017" },
       { _id: 2, host: "mongo-2.mongo-service:27017" },
     ],
   })
   ```
   Stable per-pod DNS is exactly why this is a StatefulSet — that member list has to stay
   permanently valid.
4. Give the client a replica-set connection string listing all three hosts plus
   `?replicaSet=rs0`, so it follows primary elections instead of hard-coding `mongo-0`.

In practice: use an operator (MongoDB Community Operator) or a maintained Helm chart — which is
a StatefulSet underneath, doing exactly this.
