# Persist Data with a Volume — Study Notes

Same MongoDB + Node.js web app as lesson 2 (Ingress on `myapp.com`), but MongoDB now writes
to a **PersistentVolume** backed by a directory on the Minikube node instead of the pod's
ephemeral container filesystem. Delete the mongo pod, the data survives.

Deliberately **not** a StatefulSet — a plain Deployment with one replica and a
PersistentVolumeClaim. StatefulSets come later.

## The problem this solves

A container's writable layer dies with the container. MongoDB stores its data in `/data/db`
inside the container, so without a volume:

- pod restart / crash loop → data gone
- `kubectl delete pod` → data gone
- rolling update to a new image → data gone

## Architecture

```
  browser ──► nginx ingress controller ──► webapp-ingress (host: myapp.com)
                                                │
                                                ▼
                                     webapp-service (ClusterIP :3000)
                                                │
                                                ▼
                                         webapp pod (:3000)
                                                │  DB_URL = mongo-service
                                                ▼
                              mongo-service (:27017) ──► mongo pod (:27017)
                                                             │
                                              volumeMount /data/db
                                                             ▼
                                                   mongo-pvc  (claim, 1Gi, RWO)
                                                             │  bound to
                                                             ▼
                                                   mongo-pv   (hostPath /data/mongo)
                                                             │
                                                             ▼
                                            minikube node filesystem
```

## The three storage objects

| Object                          | Who creates it    | What it is                                                     |
| ------------------------------- | ----------------- | -------------------------------------------------------------- |
| **PersistentVolume** (PV)       | admin / this repo | The actual storage — a piece of the cluster. Not namespaced.   |
| **PersistentVolumeClaim** (PVC) | app author        | A _request_ for storage (size + access mode). Namespaced.      |
| **volume + volumeMount**        | pod spec          | Attaches the claim into a container at a path.                 |

The pod never names the PV. It names the **PVC**; Kubernetes binds PVC → PV. That
indirection is the whole point — the app asks for "1Gi RWO", the cluster decides what
backs it.

## Files

| File                     | Kind                           | Purpose                                                      |
| ------------------------ | ------------------------------ | ------------------------------------------------------------ |
| `mongo-config-map.yaml`  | ConfigMap                      | Holds `mongo-url` = `mongo-service` (DB host name)           |
| `mongo-secrets.yaml`     | Secret                         | base64 DB user + password                                    |
| `mongo-volume.yaml`      | PersistentVolume + Claim       | 1Gi hostPath volume at `/data/mongo` + the claim for it      |
| `mongo-deployment.yaml`  | Deployment + Service           | MongoDB pod mounting the PVC at `/data/db`, Service on 27017 |
| `webapp-deployment.yaml` | Deployment + Service + Ingress | Web app pod, ClusterIP Service on 3000, Ingress `myapp.com`  |

## Manifest anatomy

### PersistentVolume

```yaml
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/mongo
    type: DirectoryOrCreate
```

- `hostPath` = a directory on **the node**. Learning-only: it ties data to one node and is
  a security hole on real clusters (a pod can mount any host path). Real clusters use
  cloud disks, NFS, Ceph, etc. through a CSI driver.
- `type: DirectoryOrCreate` creates the directory if missing instead of failing the mount.
- `storageClassName: manual` — an invented name, matched by the PVC. It **disables dynamic
  provisioning** for this pair: without it, Minikube's default `standard` StorageClass would
  provision a brand-new volume for the claim and this hand-written PV would sit `Available`,
  unused.
- Reclaim policy on PVC delete: `Retain` keeps the PV and the data (manual cleanup),
  `Delete` wipes the backing storage.

### Access modes

| Mode                  | Meaning                                                  |
| --------------------- | -------------------------------------------------------- |
| `ReadWriteOnce` (RWO) | mounted read-write by **one node**                       |
| `ReadOnlyMany` (ROX)  | mounted read-only by many nodes                          |
| `ReadWriteMany` (RWX) | mounted read-write by many nodes (needs an NFS-ish backend) |

Databases want RWO. RWO is per **node**, not per pod — two pods on the same node can both
mount it, which is exactly why `strategy: Recreate` matters below.

### PersistentVolumeClaim

```yaml
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```

A PV matches a claim only if class, access mode, and capacity all satisfy it. The bind is
**one PVC to one PV, exclusively** — a 1Gi claim against a 5Gi PV consumes the whole PV.

### Mounting it in the pod

```yaml
spec:
  containers:
    - name: mongodb
      volumeMounts:
        - name: mongo-data # must match a name under spec.volumes
          mountPath: /data/db # where mongod writes
  volumes:
    - name: mongo-data
      persistentVolumeClaim:
        claimName: mongo-pvc
```

`volumes` is on the **pod spec** (sibling of `containers`); `volumeMounts` is on the
**container**. The `name` is the join key between them.

### Recreate strategy

```yaml
strategy:
  type: Recreate
```

Default `RollingUpdate` starts the new pod before terminating the old one. Both land on the
same node, both mount the same directory, and the second `mongod` fails on the lock file.
`Recreate` = brief downtime, no corruption.

## Apply order

```bash
kubectl apply -f mongo-config-map.yaml
kubectl apply -f mongo-secrets.yaml
kubectl apply -f mongo-volume.yaml       # PV + PVC first
kubectl apply -f mongo-deployment.yaml
kubectl apply -f webapp-deployment.yaml
```

Or the whole directory: `kubectl apply -f .`

A pod referencing a missing PVC stays `Pending` with a `FailedScheduling` event, so create
storage before workloads.

Ingress still needs the controller and the hosts entry from lesson 2:

```bash
minikube addons enable ingress
minikube ip                                     # e.g. 192.168.49.2
echo "192.168.49.2 myapp.com" | sudo tee -a /etc/hosts
```

## Inspect

```bash
kubectl get pv                      # STATUS Bound, CLAIM = default/mongo-pvc
kubectl get pvc                     # STATUS Bound, VOLUME = mongo-pv
kubectl describe pvc mongo-pvc      # binding events / why it's still Pending
kubectl describe pod <mongo-pod>    # Volumes + Mounts sections
kubectl exec -it <mongo-pod> -- ls -l /data/db
```

Look at the data on the node itself:

```bash
minikube ssh -- sudo ls -l /data/mongo
```

## Prove it persists

```bash
# 1. add data through the web app UI at http://myapp.com
# 2. kill the pod
kubectl delete pod -l app=mongo
kubectl get pods -w                 # a new mongo pod comes up
# 3. reload the app — the data is still there
```

Stronger test — delete the whole Deployment, re-apply it, data still there. The PVC and PV
are separate objects with their own lifecycle.

## Tear down

```bash
kubectl delete -f .                        # deletes the PVC and PV objects
minikube ssh -- sudo rm -rf /data/mongo    # Retain leaves the hostPath data behind
```

To keep the data across a redeploy, delete only the workloads:

```bash
kubectl delete -f mongo-deployment.yaml -f webapp-deployment.yaml
```

## Gotchas

- **PVC stuck `Pending`** — usually a `storageClassName` mismatch, a requested size larger
  than the PV, or an access-mode mismatch. `kubectl describe pvc mongo-pvc` names the reason.
- **Omitting `storageClassName`** ≠ "no class". An empty field gets the cluster's **default**
  StorageClass, so Minikube dynamically provisions a fresh volume and ignores the PV here.
- **PV is cluster-scoped, PVC is namespaced.** A PVC binds only a PV no other claim holds,
  but the PV itself has no namespace.
- **Binding is sticky.** Once bound, a PV stays tied to that PVC's UID. After the PVC is
  deleted, a `Retain` PV goes to `Released` — not `Available` — and won't rebind until
  `spec.claimRef` is cleared or the PV is recreated.
- **`hostPath` doesn't work on multi-node clusters.** The data lives on whichever node the
  pod ran on; reschedule elsewhere and the volume looks empty. Single-node Minikube only.
- **`minikube delete` destroys `/data/mongo`** along with the node. `minikube stop` /
  `start` keeps it.
- **Mongo creates the root user only on an empty `/data/db`.** Change the Secret after data
  exists and auth still uses the old credentials — the init script never re-runs.
- **A mount hides whatever the image shipped at that path.** `/data/db` is meant to be
  empty, so it's fine here; not true for every image.
- **Secrets are base64, not encrypted.**
