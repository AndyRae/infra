# infra

GitOps fleet: one k3s cluster per machine (`ambp`, `amm`, `ama`), each running its
own ArgoCD instance polling this repo. Add an app once under `apps/`, declare it
per-cluster under `clusters-manifest/<cluster>/`, and it syncs automatically.

Full design rationale, current cluster status, and known gotchas live in `CLAUDE.md`.

## Rebuild a machine from zero

```bash
brew install colima kubectl helm argocd kubeseal

# Fleet cluster lives in its own Colima profile, isolated from everyday docker use.
# Pass config as explicit flags — a pre-written colima.yaml is not reliably picked up.
colima start --profile fleet --cpu 4 --memory 8 --disk 60 \
  --kubernetes --kubernetes-version v1.35.0+k3s1 \
  --k3s-arg='--write-kubeconfig-mode=644'
kubectl config rename-context colima-fleet <cluster-name>   # ambp | amm | ama

# ArgoCD install — server-side apply is required, some CRDs are too large for
# client-side apply's annotation size limit. Check for current stable before installing.
kubectl --context <cluster-name> create namespace argocd
kubectl --context <cluster-name> apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.5/manifests/install.yaml

# Sealed Secrets controller — each cluster generates its own keypair, nothing to copy around.
helm repo add sealed-secrets https://bitnami.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --kube-context <cluster-name> -n kube-system \
  --set-string fullnameOverride=sealed-secrets-controller

# Bootstrap this cluster's app-of-apps root (the only manual kubectl apply you'll ever run again)
kubectl --context <cluster-name> apply -f clusters/<cluster-name>/root-app.yaml
```

## Layout

- `apps/<name>/` — what an app is and how to run it (Helm umbrella chart + per-cluster
  `values/<cluster>.yaml` where a solid upstream chart exists; Kustomize base+overlays otherwise).
- `clusters-manifest/<cluster>/` — which apps run on which cluster (one ArgoCD `Application` per app).
- `clusters/<cluster>/root-app.yaml` — the app-of-apps root for that cluster.

Secrets are per-app `SealedSecret` manifests committed as ciphertext inside each
app's `values/<cluster>.yaml` (see `CLAUDE.md` → Adding an app) — no separate
secrets directory or key file to manage.
