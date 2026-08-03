# ConfigMap from a .env File — Study Notes

Same MongoDB + Node.js web app as lesson 1, but the ConfigMap is generated from a
`.env` file via **kustomize** instead of being hand-written as a `ConfigMap` manifest.

> Previous lesson: [`../1-expose-app-with-nodeport/`](../1-expose-app-with-nodeport/)

## Architecture

```
                 NodePort 30100
  browser ────────────────────────► webapp-service ──► webapp pod (:3000)
                                                             │
                                                             │ DB_URL = mongo-service
                                                             ▼
                                          mongo-service (:27017) ──► mongo pod (:27017)
```

- **webapp** reads DB user/password from the Secret and the DB host from the generated ConfigMap.
- **mongo** reads its root user/password from the same Secret.
- webapp reaches mongo by the internal DNS name `mongo-service`.

## Files

| File                     | Kind                 | Purpose                                                  |
| ------------------------ | -------------------- | --------------------------------------------------------- |
| `.env`                  | env file             | `MONGO_URL=mongo-service` — source for the generated ConfigMap |
| `kustomization.yaml`     | Kustomize            | `configMapGenerator` that turns `.env` into a ConfigMap    |
| `mongo-config-map.yaml`  | ConfigMap            | Hand-written fallback if applying with plain `kubectl apply -f .` |
| `mongo-secrets.yaml`     | Secret               | base64 DB user + password                                  |
| `mongo-deployment.yaml`  | Deployment + Service | MongoDB pod, internal Service on 27017                     |
| `webapp-deployment.yaml` | Deployment + Service | Web app pod, external NodePort Service on 30100             |

## configMapGenerator

```yaml
# kustomization.yaml
configMapGenerator:
  - name: mongo-config-map
    envs:
      - .env
```

- Each line in `.env` becomes one ConfigMap key, **name unchanged** — `MONGO_URL=mongo-service`
  becomes key `MONGO_URL` (uppercase, exact match to the file). It is **not** lowercased.
- kustomize appends a **content hash** to the generated ConfigMap name (e.g.
  `mongo-config-map-7h5t29cmcf`) and rewrites every `configMapKeyRef.name` /
  `envFrom.configMapRef.name` in the same kustomization to point at the hashed name. Editing
  `.env` and re-applying rolls out a new ConfigMap + triggers a pod restart automatically,
  since the Deployment's pod template changes.
- This rewrite only happens when you apply **through kustomize**
  (`kubectl apply -k .`). Applying the raw files with `kubectl apply -f .` uses
  `mongo-config-map.yaml` as-is (key `mongo-url`, no hash) and ignores `kustomization.yaml`
  entirely — pick one apply method, don't mix them.

## Two ways to consume a ConfigMap/Secret

### Explicit key mapping — `valueFrom` (used here)

Pick individual keys, rename them to whatever env var the app expects:

```yaml
env:
  - name: USER_NAME
    valueFrom:
      secretKeyRef:
        name: mongo-secret
        key: mongo-user
  - name: DB_URL
    valueFrom:
      configMapKeyRef:
        name: mongo-config-map
        key: MONGO_URL
```

### Bulk load — `envFrom` (no per-key mapping)

Loads **every** key as an env var, named exactly as the key:

```yaml
containers:
  - name: webapp
    envFrom:
      - configMapRef:
          name: mongo-config-map
      - secretRef:
          name: mongo-secret
```

`envFrom` only works when the app's expected env var names match the ConfigMap/Secret key
names exactly (no renaming). This app expects `DB_URL`/`USER_NAME`/`USER_PWD`, but the keys
are `MONGO_URL`/`mongo-user`/`mongo-password` — different names, so `envFrom` doesn't apply
without first renaming the keys. Use `valueFrom` whenever env var name ≠ key name.

## Apply order

```bash
kubectl apply -k .          # kustomize: generates ConfigMap from .env, applies everything
```

Or without kustomize (uses the static `mongo-config-map.yaml` instead of `.env`):

```bash
kubectl apply -f mongo-config-map.yaml
kubectl apply -f mongo-secrets.yaml
kubectl apply -f mongo-deployment.yaml
kubectl apply -f webapp-deployment.yaml
```

## Inspect

```bash
kubectl get configmap                          # note the hash suffix from kustomize
kubectl get configmap <name> -o yaml           # confirm the generated key name
kubectl get pod -o wide
kubectl describe pod <pod-name>                # events, why pod not starting
```

## Tear down

```bash
kubectl delete -k .
# or, if applied with plain -f:
kubectl delete -f .
```

## Gotchas

- **Key name mismatch is the #1 failure here.** `configMapGenerator` from `.env` keeps the
  var name verbatim (`MONGO_URL`), but the hand-written `mongo-config-map.yaml` uses a
  different key (`mongo-url`). A `configMapKeyRef.key` that doesn't exist in the actual
  ConfigMap → pod stuck `CreateContainerConfigError`.
- **Don't mix apply methods.** `kubectl apply -k .` and `kubectl apply -f .` in the same
  directory can create two different ConfigMaps (hashed vs. unhashed name) and only one of
  them is what the Deployment actually references.
- `kubectl describe pod <pod-name>` names the missing key/ConfigMap directly under Events —
  check there first when a pod won't start.
- **Secrets are still base64, not encrypted.**
