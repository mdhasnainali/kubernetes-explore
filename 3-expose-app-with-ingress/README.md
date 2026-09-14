# Expose an App with Ingress — Study Notes

Same MongoDB + Node.js web app as lesson 1, but the browser reaches it through an
**Ingress** on a hostname (`myapp.com`) instead of a NodePort. The webapp Service goes
back to plain `ClusterIP` — Ingress is the only external entry point.

## Architecture

```
                     host: myapp.com
  browser ──► nginx ingress controller ──► webapp-ingress
              (minikube ingress addon)          │
                                                ▼
                                     webapp-service (ClusterIP :3000)
                                                │
                                                ▼
                                         webapp pod (:3000)
                                                │
                                                │ DB_URL = mongo-service
                                                ▼
                              mongo-service (:27017) ──► mongo pod (:27017)
```

- **Ingress** is only a routing rule set. It does nothing without an **Ingress controller** running in the cluster.
- On Minikube the controller comes from the `ingress` addon (nginx), running in namespace `ingress-nginx`.
- `webapp-service` is `ClusterIP` — no `nodePort`, no `type:`. The controller runs inside the cluster, so ClusterIP is reachable to it.

## Files

| File                     | Kind                           | Purpose                                                     |
| ------------------------ | ------------------------------ | ----------------------------------------------------------- |
| `mongo-config-map.yaml`  | ConfigMap                      | Holds `mongo-url` = `mongo-service` (DB host name)          |
| `mongo-secrets.yaml`     | Secret                         | base64 DB user + password                                   |
| `mongo-deployment.yaml`  | Deployment + Service           | MongoDB pod, internal Service on 27017                      |
| `webapp-deployment.yaml` | Deployment + Service + Ingress | Web app pod, ClusterIP Service on 3000, Ingress `myapp.com` |
| `logs.txt`               | —                              | Captured `minikube logs` dump from a debugging session      |

## Enable the ingress controller

```bash
minikube addons list                  # check status
minikube addons enable ingress        # installs nginx ingress controller
kubectl get pods -n ingress-nginx     # controller pod should be Running
```

## Apply order

```bash
kubectl apply -f mongo-config-map.yaml
kubectl apply -f mongo-secrets.yaml
kubectl apply -f mongo-deployment.yaml
kubectl apply -f webapp-deployment.yaml    # Deployment + Service + Ingress
```

Or the whole directory: `kubectl apply -f .`

## Ingress manifest anatomy (`networking.k8s.io/v1`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
spec:
  rules:
    - host: myapp.com # requests carrying this Host header
      http:
        paths:
          - path: / # match all paths
            pathType: Prefix # required in v1
            backend:
              service:
                name: webapp-service
                port:
                  number: 3000
```

- `host` matches the HTTP **Host header** — the name still has to resolve to the cluster separately.
- `pathType` is **required** in `networking.k8s.io/v1`: `Prefix`, `Exact`, or `ImplementationSpecific`.
- Multiple `rules` = host-based routing (virtual hosts). Multiple `paths` under one host = path-based routing.

## Point the hostname at the cluster

`myapp.com` is not real DNS — map it locally.

```bash
minikube ip                                     # e.g. 192.168.49.2
echo "192.168.49.2 myapp.com" | sudo tee -a /etc/hosts
```

Then open <http://myapp.com>.

## Inspect

```bash
kubectl get ingress                          # HOSTS + ADDRESS columns
kubectl describe ingress webapp-ingress      # rules, backends, events
kubectl get endpoints webapp-service         # backend must have ready endpoints
kubectl get pods -n ingress-nginx            # controller health
kubectl logs -n ingress-nginx <controller-pod>
```

`ADDRESS` is blank right after creation — the controller fills it in once it programs the
rule. Blank for a long time usually means no controller is running.

## Tear down

```bash
kubectl delete -f .
minikube addons disable ingress   # optional
```

## Gotchas

- **v1 backend syntax changed.** Old `extensions/v1beta1` style (`backend.serviceName` +
  `backend.servicePort`) is rejected under `networking.k8s.io/v1` with
  `strict decoding error: unknown field ...`. Use `backend.service.name` and
  `backend.service.port.number`.
- **Field names are case-sensitive.** `ServicePort` (capital S) fails as an unknown field.
- **`path` + `pathType` are mandatory** in v1 — omitting `pathType` is a validation error.
- **Ingress without a controller does nothing.** The object gets created, `ADDRESS` stays blank, nothing routes.
- **Backend Service needs ready endpoints**, else nginx returns 503.
- **NodePort is not needed here.** Adding one alongside Ingress is redundant for this use case.
- **Secrets are base64, not encrypted.**
