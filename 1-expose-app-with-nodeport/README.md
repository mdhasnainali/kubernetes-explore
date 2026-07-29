# Expose an App with NodePort — Study Notes

A minimal Kubernetes demo: a MongoDB database plus a Node.js web app that talks to it,
running locally on Minikube. Config split into ConfigMap, Secret, two Deployments, two Services.
The web app is reached from the browser via a **NodePort** Service on port `30100`.

> Next lesson swaps NodePort for an Ingress: [`../2-expose-app-with-ingress/`](../2-expose-app-with-ingress/)

## Architecture

```
                 NodePort 30100
  browser ────────────────────────► webapp-service ──► webapp pod (:3000)
                                                             │
                                                             │ DB_URL = mongo-service
                                                             ▼
                                          mongo-service (:27017) ──► mongo pod (:27017)
```

- **webapp** reads DB user/password from the Secret and the DB host from the ConfigMap.
- **mongo** reads its root user/password from the same Secret.
- webapp reaches mongo by the internal DNS name `mongo-service` (Kubernetes Service discovery).

## Files

| File                     | Kind                 | Purpose                                            |
| ------------------------ | -------------------- | -------------------------------------------------- |
| `mongo-config-map.yaml`  | ConfigMap            | Holds `mongo-url` = `mongo-service` (DB host name) |
| `mongo-secrets.yaml`     | Secret               | base64 DB user + password                          |
| `mongo-deployment.yaml`  | Deployment + Service | MongoDB pod, internal Service on 27017             |
| `webapp-deployment.yaml` | Deployment + Service | Web app pod, external NodePort Service on 30100    |

## Concepts

- **ConfigMap** — non-secret config as key/value. Injected as env var via `configMapKeyRef`.
- **Secret** — sensitive data, base64-encoded (NOT encrypted by default). Injected via `secretKeyRef`.
- **Deployment** — manages pods + replicas. `selector.matchLabels` must match pod template `labels`.
- **Service** — stable network endpoint for a set of pods (matched by `selector`).
  - `ClusterIP` (default) — internal only.
  - `NodePort` — exposes externally on every node at `nodePort` (valid range **30000–32767**).
  - `port` = Service port, `targetPort` = container port.

## Apply order

Config/Secret first, then the workloads that consume them.

```bash
kubectl apply -f mongo-config-map.yaml    # -f = file
kubectl apply -f mongo-secrets.yaml
kubectl apply -f mongo-deployment.yaml
kubectl apply -f webapp-deployment.yaml
```

Or apply the whole directory at once:

```bash
kubectl apply -f .
```

## Start Minikube

```bash
minikube start                # start local cluster
minikube status               # check cluster state
kubectl get nodes             # confirm node is Ready
```

## Inspect

```bash
kubectl get all               # everything in current namespace
kubectl get pod               # list pods
kubectl get svc               # list services
kubectl get configmap
kubectl get secret

kubectl get pod -o wide       # + node/IP details
kubectl get pod --watch       # live updates
```

## Debug

```bash
kubectl describe pod <pod-name>    # events, why pod not starting
kubectl logs <pod-name>            # container stdout/stderr
kubectl logs <pod-name> -f         # follow logs
kubectl exec -it <pod-name> -- sh  # shell inside container
```

## Access the web app

NodePort Services on Minikube are not on `localhost` directly — use `minikube service`:

```bash
minikube service webapp-service          # opens the URL in browser
minikube service webapp-service --url    # just print the URL
```

## Secrets: encode / decode

```bash
echo -n 'mongouser' | base64            # encode  -> bW9uZ291c2Vy
echo 'bW9uZ291c2Vy' | base64 --decode   # decode  -> mongouser
```

## Update / restart

```bash
kubectl apply -f mongo-deployment.yaml       # re-apply after edit
kubectl rollout restart deployment mongo-deployment
kubectl rollout status deployment webapp-deployment
```

## Tear down

```bash
kubectl delete -f .          # delete everything defined here
# or individually:
kubectl delete -f webapp-deployment.yaml
kubectl delete -f mongo-deployment.yaml
kubectl delete -f mongo-secrets.yaml
kubectl delete -f mongo-config-map.yaml

minikube stop                # stop cluster (keeps state)
minikube delete              # destroy cluster
```

## Gotchas

- **Service `port` must match what the client expects.** webapp connects to mongo on **27017**, so `mongo-service` exposes `port: 27017`.
- **NodePort range** is 30000–32767 — `30100` is valid.
- **Secrets are base64, not encrypted.** Anyone with read access can decode them.
- **`image: latest`** is not reproducible — pin a version (`mongo:7.0`).
- Apply **ConfigMap/Secret before** the Deployments that reference them.
