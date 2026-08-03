# kubernetes-explore

Hands-on notes and manifests for learning Kubernetes locally with **Minikube** + **kubectl**.
Each subdirectory is one self-contained lesson with its own `README.md` study notes.

## Prerequisites

```bash
minikube version
kubectl version --client
minikube start          # spin up local single-node cluster
kubectl get nodes       # confirm node is Ready
```

## Learning sequence

Follow in order — each builds on the concepts of the previous.

| #   | Directory                                                              | Focus                                                                                  | Key objects                                                   |
| --- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| 1   | [`1-expose-app-with-nodeport/`](./1-expose-app-with-nodeport/)         | First full app: DB + web app, wiring config and secrets, exposing a service externally | ConfigMap, Secret, Deployment, Service (ClusterIP + NodePort) |
| 2   | [`2-expose-app-with-ingress/`](./2-expose-app-with-ingress/)           | Same app, external access via hostname routing instead of a NodePort                   | Ingress (`networking.k8s.io/v1`), nginx ingress controller    |
| 3   | [`3-app-with-persist-data/`](./3-app-with-persist-data/)               | Same app, MongoDB data survives pod deletion via node-local storage                    | PersistentVolume (`hostPath`), PersistentVolumeClaim, volumeMounts |
| 4   | [`4-app-with-statefulset/`](./4-app-with-statefulset/)                 | MongoDB as 3 pods with stable identity and one volume each, on local storage           | StatefulSet, headless Service, `volumeClaimTemplates`, StorageClass (`WaitForFirstConsumer`) |
| 5   | [`5-app-with-hpa/`](./5-app-with-hpa/)                                 | Web app replica count driven by measured CPU/memory instead of a fixed number          | HorizontalPodAutoscaler (`autoscaling/v2`), metrics-server, resource requests/limits, probes |

> More lessons get added here as the exploration continues.

**Roadmap:** [`TASKS.md`](./TASKS.md) — phased task list from here to production readiness, plus capstone projects.

## How each lesson works

1. `cd` into the lesson directory.
2. Read its `README.md` — architecture, concepts, gotchas.
3. Apply the manifests (order matters — config/secrets before workloads):

   ```bash
   kubectl apply -f .
   ```

4. Inspect and debug:

   ```bash
   kubectl get all
   kubectl describe pod <pod-name>
   kubectl logs <pod-name>
   ```

5. Tear down when done:

   ```bash
   kubectl delete -f .
   ```

## Core concepts reference

- **Pod** — smallest deployable unit; one or more containers sharing network/storage.
- **Deployment** — declarative manager for pods + replicas + rolling updates.
- **Service** — stable endpoint for a set of pods.
  - `ClusterIP` (default) — internal-only.
  - `NodePort` — external access on each node, port range **30000–32767**.
  - `LoadBalancer` — cloud external LB (`minikube tunnel` locally).
- **Ingress** — HTTP host/path routing rules into cluster Services. Inert without an **Ingress controller** (`minikube addons enable ingress`).
- **PersistentVolume / PersistentVolumeClaim** — storage decoupled from pod lifetime. The PV is the storage, the PVC is the request; pods mount the claim.
- **StatefulSet** — pods with stable ordinal names, ordered lifecycle, and one PVC each.
- **HorizontalPodAutoscaler** — rewrites a workload's `spec.replicas` from observed metrics. Needs resource `requests` and **metrics-server** (`minikube addons enable metrics-server`).
- **ConfigMap** — non-secret key/value config.
- **Secret** — sensitive data, base64-encoded (⚠️ not encrypted at rest by default).

## Cluster lifecycle

```bash
minikube start          # start
minikube status         # state
minikube ip             # cluster IP (for /etc/hosts entries)
minikube dashboard      # web UI
minikube addons list    # optional components (ingress, metrics-server, ...)
minikube stop           # stop (keeps state)
minikube delete         # destroy
```

## Notes

- Demo secrets in these lessons are **intentional learning fixtures**, not real credentials.
- Container images are pinned to explicit versions for reproducibility.
