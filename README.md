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

| #   | Directory                                    | Focus                                                                                  | Key objects                                                   |
| --- | -------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| 1   | [`mongo-webapp-demo/`](./mongo-webapp-demo/) | First full app: DB + web app, wiring config and secrets, exposing a service externally | ConfigMap, Secret, Deployment, Service (ClusterIP + NodePort) |

> More lessons get added here as the exploration continues.

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
- **ConfigMap** — non-secret key/value config.
- **Secret** — sensitive data, base64-encoded (⚠️ not encrypted at rest by default).

## Cluster lifecycle

```bash
minikube start          # start
minikube status         # state
minikube dashboard      # web UI
minikube stop           # stop (keeps state)
minikube delete         # destroy
```

## Notes

- Demo secrets in these lessons are **intentional learning fixtures**, not real credentials.
- Container images are pinned to explicit versions for reproducibility.
