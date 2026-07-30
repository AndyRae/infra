# infra

GitOps fleet: one k3s cluster per machine (`ambp`, `amm`, `ama`), each running its
own ArgoCD instance polling this repo. Add an app once under `apps/`, declare it
per-cluster under `clusters-manifest/<cluster>/`, and it syncs automatically.

Full design rationale and milestone plan live in `CLAUDE.md` (gitignored, local-only).

## Rebuild a machine from zero

```bash
brew install colima kubectl helm argocd sops age

# Fleet cluster lives in its own Colima profile, isolated from everyday docker use.
colima start --profile fleet
kubectl config rename-context colima-fleet <cluster-name>   # ambp | amm | ama

# ArgoCD install (pin version — check for current stable before installing)
kubectl --context <cluster-name> create namespace argocd
kubectl --context <cluster-name> apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.13.2/manifests/install.yaml

# Age key for SOPS-encrypted secrets — generate locally on each machine, never copy the private key
age-keygen -o ~/.config/sops/age/keys.txt
# add the printed public key to secrets/.sops.yaml as a recipient, commit, push

# Bootstrap this cluster's app-of-apps root (the only manual kubectl apply you'll ever run again)
kubectl --context <cluster-name> apply -f clusters/<cluster-name>/root-app.yaml
```

## Layout

- `apps/<name>/` — what an app is and how to run it (Helm umbrella chart + per-cluster
  `values/<cluster>.yaml` where a solid upstream chart exists; Kustomize base+overlays otherwise).
- `clusters-manifest/<cluster>/` — which apps run on which cluster (one ArgoCD `Application` per app).
- `clusters/<cluster>/root-app.yaml` — the app-of-apps root for that cluster.
- `secrets/.sops.yaml` — SOPS config; age public keys, one recipient per machine.
